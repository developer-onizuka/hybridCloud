# hybridCloud


# 0. Prerequisites and what you must create before trying mongoDB's replicaSet

| Virtual Machine | IPaddress | Function | Vagrantfile |
| --- | --- | --- | --- |
| Master | 192.168.33.100 | k8s's master | [k8s_master/Vagrantfile](https://github.com/developer-onizuka/mongoDB_replicaSet/blob/main/k8s_master/Vagrantfile) |
| Worker1 | 192.168.33.101 | k8s's worker | ^ |
| Worker8 | 192.168.33.108 | ^ | [k8s_worker/Vagrantfile](https://github.com/developer-onizuka/mongoDB_replicaSet/blob/main/k8s_worker/Vagrantfile) |
| Worker9 | 192.168.33.109 | ^ | ^ |
| ops-manager | 192.168.33.12 | mongoDB ops Manager | [opsManager/Vagrantfile](https://github.com/developer-onizuka/mongoDB_replicaSet/blob/main/opsManager/Vagrantfile) |
| mongo-0 | 192.168.33.31 | mongoDB replicaSet | [replicaSet/Vagrantfile](https://github.com/developer-onizuka/mongoDB_replicaSet/blob/main/replicaSet/Vagrantfile) |
| mongo-1 | 192.168.33.32 | ^ | ^ |
| mongo-2 | 192.168.33.33 | ^ | ^ |

- Create kubernetes cluster attached LoadBalancer such as MetalLB. You might use the Vagrantfiles above and the link below for MetalLB system:
  > https://github.com/developer-onizuka/metalLB

- If you want to use multi-nodes cluster with open-vSwitch, then you might use the guide below:
  > https://github.com/developer-onizuka/openvswitch_install

- Download and install istio and make label on the default namespace with istio-injection=enabled.
  > https://github.com/developer-onizuka/istio

- Create and configure the Ops Manager itself. 
  > https://github.com/developer-onizuka/mongoDB_opsManager

- Create Three Virtual Machines for mongoDB's replicaSet which will be used later. 

- Istio's WorkloadEntry creates a kind of resolvor between endpoint and IP address for the pod in the Service mesh. You don't need creating entries in /etc/hosts as like below if you create WorkloadEntry resource thru mongo-vm-svc.yaml and mongo-vm-wkle.yaml attached.
```
192.168.33.30 mongo-0
192.168.33.31 mongo-1
192.168.33.32 mongo-2
```

# 1. Git clone and Create IngressGateway
Create new IngressGateway so that you could attach new Ports such as 30001 or 30002 for the access from outside.
```
$ git clone https://github.com/developer-onizuka/hybridCloud
$ cd hybridCloud
$ export PATH="$PATH:/home/vagrant/istio-1.12.2/bin"
$ istioctl install -y -f hybrid-cloud.yaml
$ kubectl apply -f ingress-gateway.yaml
```

# 2. Browser Access

Find the External-IP for istio-hybridcloud. In My case, it is "192.168.33.222".
```
kubectl get services -n istio-system 
NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)                                                                      AGE
grafana                ClusterIP      10.96.20.62      <none>           3000/TCP                                                                     5d2h
istio-hybridcloud      LoadBalancer   10.108.204.110   192.168.33.222   15021:30812/TCP,443:30262/TCP,80:32350/TCP,30001:30022/TCP,30002:32767/TCP   15h
istio-ingressgateway   LoadBalancer   10.104.215.64    192.168.33.220   15021:31534/TCP,80:32400/TCP,443:30297/TCP                                   5d2h
istiod                 ClusterIP      10.104.211.132   <none>           15010/TCP,15012/TCP,443/TCP,15014/TCP                                        5d2h
jaeger-collector       ClusterIP      10.108.234.88    <none>           14268/TCP,14250/TCP,9411/TCP                                                 5d2h
kiali                  LoadBalancer   10.109.66.129    192.168.33.221   20001:32457/TCP,9090:30661/TCP                                               5d2h
prometheus             ClusterIP      10.102.165.138   <none>           9090/TCP                                                                     5d2h
tracing                ClusterIP      10.106.81.224    <none>           80/TCP,16685/TCP                                                             5d2h
zipkin                 ClusterIP      10.104.42.193    <none>           9411/TCP                                                                     5d2h
```

# 3. Path to the On-premises DB

```
$ kubectl create configmap nginx-onprem-config --from-file=onprem/default.conf
$ kubectl apply -f onprem/mongo-vm-svc.yaml,onprem/mongo-vm-wkle.yaml
$ kubectl apply -f onprem/employee-onprem-mongodb.yaml
$ kubectl apply -f onprem/nginx-onprem.yaml
```
![hybridcloud2.png](https://github.com/developer-onizuka/hybridCloud/blob/main/hybridcloud2.png)

![hybridcloud4.png](https://github.com/developer-onizuka/hybridCloud/blob/main/hybridcloud4.png)


# 4. Path to the Azure DB
First of all, Create Azure Cosmos DB API for MongoDB and get connection string.
> https://github.com/developer-onizuka/mvc_CosmosDB#1-create-azure-cosmos-db-api-for-mongodb

```
$ vi connection-string.txt 
mongodb://myfirstcosmosdb:XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX==@myfirstcosmosdb.mongo.cosmos.azure.com:10255/?ssl=true&retrywrites=false&replicaSet=globaldb&maxIdleTimeMS=120000&appName=@myfirstcosmosdb@
$ kubectl create secret generic connectionstring --from-file=secretenv=./connection-string.txt
$ rm -f connection-string.txt
```
```
$ kubectl create configmap nginx-onprem-config --from-file=azure/default.conf
$ kubectl apply -f azure/employee-azure-cosmosdb.yaml
$ kubectl apply -f azure/nginx-azure.yaml
```

![hybridcloud3.png](https://github.com/developer-onizuka/hybridCloud/blob/main/hybridcloud3.png)

![hybridcloud5.png](https://github.com/developer-onizuka/hybridCloud/blob/main/hybridcloud5.png)

# 5. Kiali's Graph View
![hybridcloud1.png](https://github.com/developer-onizuka/hybridCloud/blob/main/hybridcloud1.png)


# 6. L7 Aware Access
- Create Gateway.<br>
This Gateway is on "istio-ingressgateway" which is Istio default IngressGateway implementation and its External-IP is "192.168.33.220".<br>
In this case, I'm not gonna create new ports on IngressGateway and I can use the host of HTTP Header to manage and separate L7 access for onprem and Azure.
```
$ kubectl apply -f ingress-gateway-L7.yaml
```

- Create Configmap for Nginx.
```
$ kubectl create configmap nginx-onprem-config --from-file=onprem-L7/default.conf
$ kubectl create configmap nginx-onprem-config --from-file=azure-L7/default.conf
```

- Create each pod.
```
$ kubectl apply -f onprem-L7/.
$ kubectl apply -f azure-L7/.
```

# 7. Browser Access
Before Access, You should resolve the 192.168.33.220 to the hostname.
```
$ vi /etc/hosts
192.168.33.220 onprem.example.com
192.168.33.220 azure.example.com
```

![hybridcloud6.png](https://github.com/developer-onizuka/hybridCloud/blob/main/hybridcloud6.png)
![hybridcloud7.png](https://github.com/developer-onizuka/hybridCloud/blob/main/hybridcloud7.png)
