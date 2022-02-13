
# A three  node microk8s using metallb for ingress load balancing with BGP


```yaml
k8s: 
 - node-1 172.16.1.16/24
 - node-2 172.16.1.17/24
 - node-3 172.16.1.18/24

Router:
 eth1: 172.16.0.1/24    # my laptop default-gateway
 eth0: 172.16.1.1/24    # mk8s cluster default-gateway
 ASN: 65000
```


## 1 - setup a multinode k8s cluster 


In my lab I use [microk8s](https://microk8s.io) 
just follow this [guide](https://microk8s.io/docs/clustering) to create a multinode microk8s cluster

## 2 - install and enable metallb

microk8s comes with pre-installed metallb package, just enable it


```bash
$ microk8s enable metallb
```

and check if it's running

```bash
$ microk8s status

microk8s is running
high-availability: yes
  datastore master nodes: 172.16.1.18:19001 172.16.1.16:19001 172.16.1.17:19001
  datastore standby nodes: none
addons:
  enabled:
...
      metallb              # Loadbalancer for your Kubernetes cluster
```

## 3 - Add configMap

get the stock ConfigMap name

```bash
$ kubectl get configmaps -n metallb-system
NAME               DATA   AGE
kube-root-ca.crt   1      46h
config             1      46h
```

and get the stock configmap as a base

```bash
$ kubectl -n metallb-system get configmaps config -o yaml > metallb.yaml
```

adjust the configmap to reflect your environment, this is my setup

```yaml
apiVersion: v1
kind: ConfigMap
data:
  config: |
    peers:
    - my-asn: 64512
      peer-asn: 65000
      peer-address: 172.16.1.20
      peer-port: 179
    address-pools:
    - name: my-ip-space
      protocol: bgp
      addresses:
      - 172.16.24.0/24
metadata:
  name: config
  namespace: metallb-system
```

my router that act also as default-gateway for the lan segment has 172.16.1.1/24 as lan ip and BGP ASN 65000 
I reserve the pool "my-ip-space" 172.16.24.0/24 resulting in 256 services (including 172.16.24.0/32 and 172.16.24.255/32) 

apply the configuration with

```bash
$ kubectl apply -f metallb.yaml
```

controllers and speakers requires to reload the configuration

```bash
$ kubectl -n metallb-system rollout restart deployment controller
$ kubectl -n metallb-system rollout restart daemonset speaker
```

and then check the full deploy

```zsh
$ kubectl get pod -n metallb-system -o wide
NAME                          READY   STATUS    RESTARTS   AGE   IP            NODE            NOMINATED NODE   READINESS GATES
speaker-mwcd4                 1/1     Running   0          10m   172.16.1.16   devops-k8s-02   <none>           <none>
speaker-bc4jq                 1/1     Running   0          10m   172.16.1.17   devops-k8s-03   <none>           <none>
speaker-dd8rc                 1/1     Running   0          10m   172.16.1.18   devops-k8s-01   <none>           <none>
controller-559b68bfd8-w47dd   1/1     Running   0          10m   10.1.63.141   devops-k8s-01   <none>           <none>
```


## 4 - router config

I use a [vyos](https://vyos.io) router in my lab, it's just a VM and this is an excerpt from the BGP configuration

```junos
 protocols {
     bgp 65000 {
         maximum-paths {
             ebgp 16
         }
         neighbor 172.16.1.16 {
             peer-group K8S
         }
         neighbor 172.16.1.17 {
             peer-group K8S
         }
         neighbor 172.16.1.18 {
             peer-group K8S
         }
         peer-group K8S {
             passive
             remote-as 64512
             update-source 172.16.1.1
         }
     }
 }
```

it's a vanilla bgp configuration, with a peer-group to group the common parameters of the three k8s nodes.
and obviously enable multipath

commit the config and check the bgp status


```junos
vyos@vy-01:~$ sh ip bgp summary

IPv4 Unicast Summary:
BGP router identifier 172.16.19.1, local AS number 65000 vrf-id 0
BGP table version 29
RIB entries 2, using 368 bytes of memory
Peers 3, using 102 KiB of memory
Peer groups 1, using 64 bytes of memory

Neighbor        V         AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd
172.16.1.16     4      64512   65527   65532        0    0    0 00:00:30            1
172.16.1.17     4      64512   65527   65532        0    0    0 00:00:30            1
172.16.1.18     4      64512   65529   65530        0    0    0 00:00:30            1
```

bgp sessions are estabilished with all the k8s nodes 


## 5 - create a deploy that require a LoadBalancer service

let's create a web server that use nginx on port 80

```bash
$ kubectl run web-server --image=nginx --port=80
pod/web-server created

$ kubectl get pod -o wide
NAME         READY   STATUS    RESTARTS   AGE   IP            NODE            NOMINATED NODE   READINESS GATES
web-server   1/1     Running   0          13s   10.1.227.23   devops-k8s-02   <none>           <none>
```

and expose the service 

```bash
kubectl expose pod web-server --type=LoadBalancer
service/web-server exposed

$ kubectl get service
NAME         TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
web-server   LoadBalancer   10.152.183.250   172.16.24.0   80:32130/TCP   15s
```

and the external-IP assigned it's the first of metallb pool


## 6 - check BGP advertisement and service reachability


it's time to check the bgp advertisement on vyos router, that now act as a load-balancer

```text
vyos@vy-01:~$ sh ip route bgp
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR, f - OpenFabric,
       > - selected route, * - FIB route, q - queued route, r - rejected route

B>* 172.16.24.0/32 [20/0] via 172.16.1.16, eth1, 00:11:46
  *                       via 172.16.1.17, eth1, 00:11:46
  *                       via 172.16.1.18, eth1, 00:11:46
```

and perform a quick&dirty check directly from the router

```html
vyos@vy-01:~$ telnet to 172.16.24.0 port 80
Trying 172.16.24.0...
Connected to 172.16.24.0.
Escape character is '^]'.
GET /
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
Connection closed by foreign host.

```

and now create a deploy and play with replicas...

Enjoy!



