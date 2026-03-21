# TLS offload

이 활용사례는 Kubernetes Gateway API User Guides의 [TLS Spec](https://gateway-api.sigs.k8s.io/guides/tls/)과 [Redirect Spec](https://gateway-api.sigs.k8s.io/guides/http-redirect-rewrite/)을 구현합니다. TLS offload 기능과 HTTP 요청을 HTPPS로 redirect를 직접 생성하고 테스트를 진행합니다. 

해당 랩 디렉토리로 이동합니다.

```code
cd 3.nginx-gateway-fabric/labs/4.tls-offload
```

TLS 인증에 사용할 인증서와 키를 생성하고 Gateway API의 ReferenceGrant 오브젝트를 생성합니다. 인증서를 생성 후 `ReferenceGrant`를 설정하지 않을 경우 해당 인증서를 다른 Gateway에서 사용할 수가 없기 때문에 인증서 생성 시 `ReferenceGrant` 오브젝트를 생성해서 인증서를 참조할 수 있도록 설정해야 합니다. 

```code
[ReferenceGrant 설정부분]
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: access-to-cafe-secret
  namespace: certificate
spec:
  to:
  - group: ""
    kind: Secret
    name: cafe-secret # if you omit this name, then Gateways in default namespace can access all Secrets in the certificate namespace
  from:
  - group: gateway.networking.k8s.io
    kind: Gateway
    namespace: default

kubectl apply -f 0.certificate.yaml
```

테스트를 위한 애플리케이션을 배포합니다.

```code
kubectl apply -f 1.coffee.yaml
```

애플리케이션 배포 후 모든 pods가 정상적으로 `running` 상태인지 확인합니다.

```code
kubectl get all
```

아래의 내용과 유사한 결과를 확인할 수 있습니다.

```
NAME                          READY   STATUS    RESTARTS   AGE
pod/coffee-56b44d4c55-jdst2   1/1     Running   0          3s

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/coffee       ClusterIP   10.101.48.47   <none>        80/TCP    3s
service/kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP   268d

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/coffee   1/1     1            1           3s

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/coffee-56b44d4c55   1         1         1       3s
```

`Gateway` 오브젝트를 배포합니다. NGINX Gateway Fabric의 데이터플레인을 현재 `namespace`에 배포합니다. 

```code
kubectl apply -f 2.gateway.yaml
```

NGINX Gateway Fabric 데이터플레인 pod의 상태를 확인합니다.

```
kubectl get pods
```

`cafe-nginx-7b58558b4b-87b2w` pod는 NGINX Gateway Fabric 데이터플레인입니다.

```
NAME                          READY   STATUS    RESTARTS   AGE
cafe-nginx-758ff7574c-kpbqx   1/1     Running   0          24s
coffee-56b44d4c55-jdst2       1/1     Running   0          57s
```

Gateway 설정을 확인합니다.

```code
kubectl get gateway
```

아래와 유사한 결과를 확인할 수 있습니다. 

```code
NAME   CLASS   ADDRESS        PROGRAMMED   AGE
cafe   nginx   10.43.28.152   True         84s
```

NGINX Gateway Fabric 서비스를 확인합니다.

```code
kubectl get service
```

`cafe-nginx` is the NGINX Gateway Fabric 데이터플레인 서비스입니다.

```
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
cafe-nginx   NodePort    10.110.127.86   <none>        80:32417/TCP,443:32657/TCP   75s
coffee       ClusterIP   10.101.48.47    <none>        80/TCP                       108s
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP                      268d
```

HTTP routes를 설정 및 배포합니다.

```code
kubectl apply -f 3.httproute.yaml
```

HTTP routes의 정상 배포 및 설정을 확인합니다.

```code
kubectl get httproute
```

아래와 유사한 결과를 확인할 수 있습니다.

```code
NAME                HOSTNAMES              AGE
cafe-tls-redirect   ["cafe.example.com"]   4s
coffee              ["cafe.example.com"]   4s
```

NGINX Gateway Fabric의 데이터플레인 인스턴스의 IP와 HTTP PORT정보를 가져와서 변수로 저장합니다.

```code
export NGF_IP=`kubectl get pod -l app.kubernetes.io/instance=ngf -o json|jq '.items[0].status.hostIP' -r`
export HTTP_PORT=`kubectl get svc cafe-nginx -o jsonpath='{.spec.ports[0].nodePort}'`
export HTTPS_PORT=`kubectl get svc cafe-nginx -o jsonpath='{.spec.ports[1].nodePort}'`
```

NGINX Gateway Fabric 데이터플레인 인스턴스의 IP와 HTTP PORT 정보를 확인합니다.

```code
echo -e "NGF address: $NGF_IP\nHTTP port  : $HTTP_PORT\nHTTPS port : $HTTPS_PORT"
```

`HTTP` method를 통해서 `coffee` 애플리케이션으로 엑세스를 시도합니다. 

```code
curl -i --resolve cafe.example.com:$HTTP_PORT:$NGF_IP http://cafe.example.com:$HTTP_PORT/coffee
```

아래와 유사한 결과가 출력됩니다.

```code
HTTP/1.1 302 Moved Temporarily
Server: nginx
Date: Thu, 12 Jun 2025 11:19:04 GMT
Content-Type: text/html
Content-Length: 138
Connection: keep-alive
Location: https://cafe.example.com/coffee

<html>
<head><title>302 Found</title></head>
<body>
<center><h1>302 Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

이제 `HTTPS` method를 통해서 `coffee` 애플리케이션으로 엑세스를 시도합니다. 

```code
curl -k --resolve cafe.example.com:$HTTPS_PORT:$NGF_IP https://cafe.example.com:$HTTPS_PORT/coffee
```

아래와 유사한 결과가 출력되면 정상이고, 설정한 `HTTPRoute` 및 `TLS redirect` 기반의 Offloading 설정이 완료되었습니다.

```code
Server address: 10.0.156.120:8080
Server name: coffee-56b44d4c55-jdst2
Date: 12/Jun/2025:11:19:24 +0000
URI: /coffee
Request ID: 6cb931a24c1c1bbff763d5ba7481a2f3
```

다음 랩을 위해 이번에 수행했던 랩은 모두 삭제합니다.(작업폴더 유의해서 실행하세요)

```code
kubectl delete -f .
```
