# hybridCloud

# 1. Git clone
```
$ git clone https://github.com/developer-onizuka/hybridCloud
$ cd hybridCloud
```


# 2. Using On-premises DB
First of all, Create and configure the Ops Manager itself.
> https://github.com/developer-onizuka/mongoDB_opsManager

```
$ kubectl apply -f onprem/mongo-vm-svc.yaml,onprem/mongo-vm-wkle.yaml
$ kubectl apply -f onprem/employee-onprem-mongodb.yaml
$ kubectl create configmap nginx-onprem-config --from-file=onprem/default.conf
$ kubectl apply -f onprem/nginx-onprem.yaml
```
![hybridcloud2.png](https://github.com/developer-onizuka/hybridCloud/blob/main/hybridcloud2.png)

![hybridcloud4.png](https://github.com/developer-onizuka/hybridCloud/blob/main/hybridcloud4.png)


# 3. Using Azure DB
First of all, Create Azure Cosmos DB API for MongoDB and get connection string.
> https://github.com/developer-onizuka/mvc_CosmosDB#1-create-azure-cosmos-db-api-for-mongodb

```
$ vi connection-string.txt 
mongodb://myfirstcosmosdb:XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX==@myfirstcosmosdb.mongo.cosmos.azure.com:10255/?ssl=true&retrywrites=false&replicaSet=globaldb&maxIdleTimeMS=120000&appName=@myfirstcosmosdb@
$ kubectl create secret generic connectionstring --from-file=secretenv=./connection-string.txt
$ rm -f connection-string.txt
```
```
$ kubectl apply -f azure/employee-azure-cosmosdb.yaml
$ kubectl create configmap nginx-onprem-config --from-file=onprem/default.conf
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
