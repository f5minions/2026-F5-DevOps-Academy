# SnippetsFilter를 이용한 JWT 인증적용

이번 실습에서는 SnippetsFilter를 활용하여 애플리케이션에 JWT 토큰기반 인증을 적용합니다. 

Lab 진행을 위한 실습 경로로 이동합니다.

```code
이전 Lab 디렉토리에 있다면,
cd ../8.auth-jwt/

Lab 기본경로(2026-F5-DevOps-Academy)에 있다면,
cd 3.nginx-gateway-fabric/labs/8.auth-jwt
```

테스트를 위한 샘플 애플리케이션을 먼저 배포합니다.

```code
kubectl apply -f 0.coffee.yaml
```

테스트 애플리케이션의 배포 상태가 `running`인지 확인합니다. 

```code
kubectl get all
```

아래와 유사한 결과를 확인할 수 있습니다.

```
NAME                          READY   STATUS    RESTARTS   AGE
pod/coffee-56b44d4c55-g9gtj   1/1     Running   0          3s

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/coffee       ClusterIP   10.107.232.5   <none>        80/TCP    3s
service/kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP   385d

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/coffee   1/1     1            1           3s

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/coffee-56b44d4c55   1         1         1       3s
```

이제 동일한 namespace에 NGINX Gateway Fabric 데이터플레인을 배포하여 gateway 오브젝트로 생성합니다.

```code
kubectl apply -f 1.gateway.yaml
```

NGINX Gateway Fabric 데이터플레인 pod의 배포상태를 확인합니다.

```
kubectl get pods
```

`gateway-nginx-56678b747f-rrx4d`가 NGINX Gateway Fabric의 데이터플레인 pod 입니다. (실제 pod의 이름은 다릅니다)

```
NAME                             READY   STATUS    RESTARTS   AGE
coffee-56b44d4c55-g9gtj          1/1     Running   0          37s
gateway-nginx-56678b747f-rrx4d   1/1     Running   0          13s
```

Gateway의 배포 결과를 확인합니다.

```code
kubectl get gateway
```

아래와 유사한 결과를 확인할 수 있습니다.

```code
NAME      CLASS   ADDRESS       PROGRAMMED   AGE
gateway   nginx   10.97.70.64   True         98s
```

NGINX Gateway Fabric 데이터플레인의 서비스 상태를 확인합니다.

```code
kubectl get service
```

`gateway-nginx` 가 NGINX Gateway Fabric 데이터플레인의 서비스입니다.

```code
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
coffee          ClusterIP   10.107.232.5   <none>        80/TCP         2m17s
gateway-nginx   NodePort    10.97.70.64    <none>        80:30496/TCP   114s
kubernetes      ClusterIP   10.96.0.1      <none>        443/TCP        385d
```

이제 FactCGI 설정 snippets을 배포하기 위한 SnippetsFilter를 적용합니다. 

```code
kubectl apply -f 2.snippetsfilter-jwtauth.yaml
```

SnippetsFilter의 설정을 확인합니다.

```code
kubectl describe snippetsfilter auth-jwt
```

아래와 유사한 결과를 확인할 수 있습니다.

```code
Name:         auth-jwt
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  gateway.nginx.org/v1alpha1
Kind:         SnippetsFilter
Metadata:
  Creation Timestamp:  2025-10-07T11:41:02Z
  Generation:          1
  Resource Version:    66576310
  UID:                 e7ff7827-3ea8-42e8-8166-e758ddd6bc40
Spec:
  Snippets:
    Context:  http.server
    Value:    location = /_auth/_jwks_uri { internal;return 200 '{"keys":[{"k":"ZmFudGFzdGljand0","kty":"oct","kid":"0001"}]}'; }
    Context:  http.server.location
    Value:    auth_jwt "JWT token required";auth_jwt_type signed;auth_jwt_key_request /_auth/_jwks_uri;
Status:
  Controllers:
    Conditions:
      Last Transition Time:  2025-10-07T11:41:02Z
      Message:               SnippetsFilter is accepted
      Observed Generation:   1
      Reason:                Accepted
      Status:                True
      Type:                  Accepted
    Controller Name:         gateway.nginx.org/nginx-gateway-controller
Events:                      <none>

위 설정이 실제 NGINX Gateway Fabric의 데이터플레인에는 다음과 같이 적용됩니다. 
location = /_auth/_jwks_uri {   # = : 정확한 경로 매칭 (prefix 매칭보다 우선순위 높음)
    internal;                    # 외부 직접 접근 불가
    return 200 '{ JWKS JSON }';  # JWK Set 반환
}

{
  "keys": [
    {
      "k":   "ZmFudGFzdGljand0",   ← Base64URL 인코딩된 Secret Key
      "kty": "oct",                 ← Key Type: Octet Sequence (대칭키, HMAC용)
      "kid": "0001"                 ← Key ID: JWT 헤더의 kid와 매칭
    }
  ]
}

$ echo "ZmFudGFzdGljand0" | base64 -d
fantasticjwt        # ← 실제 HMAC Secret 값

```

설정한 SnippetsFilter를 참조하는 HTTP Route를 배포합니다. 

```code
kubectl apply -f 3.httproute.yaml
```

설정한 HTTP route를 확인합니다.

```code
kubectl get httproute
```

아래와 같은 결과를 확인할 수 있습니다.

```code
NAME     HOSTNAMES              AGE
coffee   ["cafe.example.com"]   4s
```

NGINX Gateway Fabric 데이터플레인 인스턴스의 IP 주소와 HTTP PORT 정보를 확인 후 변수로 저장합니다.

```code
export NGF_IP=`kubectl get pod -l app.kubernetes.io/instance=ngf -o json|jq '.items[0].status.hostIP' -r`
export HTTP_PORT=`kubectl get svc gateway-nginx -o jsonpath='{.spec.ports[0].nodePort}'`
```

저장된 NGINX Gateway Fabric 데이터플레인 인스턴스의 IP와 HTTP PORT정보를 확인합니다.

```code
echo -e "NGF address: $NGF_IP\nHTTP port  : $HTTP_PORT"
```

첫번째 테스트는 인증 토큰이 없는 요청으로 진행합니다. 

```code
curl -i --resolve cafe.example.com:$HTTP_PORT:$NGF_IP http://cafe.example.com:$HTTP_PORT
```

인증 토큰이 없을 경우 아래와 같이 인증 에러를 확인할 수 있습니다.

```code
HTTP/1.1 401 Unauthorized
Server: nginx
Date: Tue, 07 Oct 2025 11:43:46 GMT
Content-Type: text/html
Content-Length: 172
Connection: keep-alive
WWW-Authenticate: Bearer realm="JWT token required"

<html>
<head><title>401 Authorization Required</title></head>
<body>
<center><h1>401 Authorization Required</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

두번째 테스트로 인증 토큰을 함께 애플리케이션에 요청을 전송합니다. 

```code
curl -i --resolve cafe.example.com:$HTTP_PORT:$NGF_IP http://cafe.example.com:$HTTP_PORT -H "Authorization: Bearer `cat token.jwt`"
```

정상적인 인증 토큰을 함께 전달할 경우 인증이 완료되고 아래와 같이 정상적인 애플리케이션 접속을 확인할 수 있습니다.

```code
HTTP/1.1 200 OK
Server: nginx
Date: Tue, 07 Oct 2025 11:45:36 GMT
Content-Type: text/plain
Content-Length: 156
Connection: keep-alive
Expires: Tue, 07 Oct 2025 11:45:35 GMT
Cache-Control: no-cache

Server address: 10.0.156.110:8080
Server name: coffee-56b44d4c55-g9gtj
Date: 07/Oct/2025:11:45:36 +0000
URI: /
Request ID: 5afec06d5432b68456477e806cf8a52e
```

위 모든 확인이 끝났으면 다음 랩을 위해 이번 랩에서 설정된 모든 테스트를 삭제합니다.

```code
kubectl delete -f .
```
