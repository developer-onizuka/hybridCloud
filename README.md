# hybridCloud

# 1. Apply virtualservice
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

# 2. Access the gateway with the Header for azure
```
$ curl -H 'User: azure' 192.168.33.223
...
    <tr>
        <td>1</td>
        <td>Yukichi</td>
        <td>Fukuzawa</td>
    </tr>
...
```

# 3. Access the gateway with the Header for onprem
```
$ curl -H 'User: onprem' 192.168.33.223
...
    <tr>
        <td>1</td>
        <td>Yukichi</td>
        <td>Fukuzawa</td>
    <tr>
        <td>2</td>
        <td>Shoin</td>
        <td>Yoshida</td>
    </tr>
...
```

![hybrid-cloud.png](https://github.com/developer-onizuka/hybridCloud/blob/main/hybrid-cloud.png)
