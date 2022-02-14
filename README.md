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

# 1. Git clone
```
$ git clone https://github.com/developer-onizuka/hybridCloud
$ cd hybridCloud
```

# 2. Path to the On-premises DB

```
$ kubectl create configmap nginx-onprem-config --from-file=onprem/default.conf
$ kubectl apply -f onprem/mongo-vm-svc.yaml,onprem/mongo-vm-wkle.yaml
$ kubectl apply -f onprem/employee-onprem-mongodb.yaml
$ kubectl apply -f onprem/nginx-onprem.yaml
```
![hybridcloud2.png](https://github.com/developer-onizuka/hybridCloud/blob/main/hybridcloud2.png)

![hybridcloud4.png](https://github.com/developer-onizuka/hybridCloud/blob/main/hybridcloud4.png)


# 3. Path to the Azure DB
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

# 4. Kiali's Graph View
![hybridcloud1.png](https://github.com/developer-onizuka/hybridCloud/blob/main/hybridcloud1.png)











# X. Remainder
```
$ cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: nginx-azure-vsvc
spec:
  hosts:
  - "*"
  gateways:
  - employee-gateway
  http:
  - match:
    #- uri:
    #    prefix: "/"
    - headers:
       user:
         exact: azure
    #- name: v1
    #  queryParams:
    #    v:
    #      exact: "1"
    route:
    - destination:
        port:
          number: 8080
        host: nginx-azure-svc
  - match:
    - headers:
       user:
         exact: onprem
    route:
    - destination:
        port:
          number: 8080
        host: nginx-svc
EOF
```
