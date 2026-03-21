# HTTP 요청 및 응답 해더의 수정

이번 실습에서는 Gateway에서 애플리케이션의 HTTP headers를 어떻게 수정할 수 있는가를 확인합니다.

Lab 실습을 위해 해당 디렉토리로 이동합니다.

```code
이전 Lab 디렉토리에 있다면,
cd ../3.nginx-gateway-fabric/labs/3.http-headers

실습 메인 디렉토리(2026-F5-DevOps-Academy)에 있다면,
cd 3.nginx-gateway-fabric/labs/2.3.http-headers
```

실습을 위한 애플리케이션을 먼저 배포합니다.

```code
kubectl apply -f 0.app.yaml
```

배포한 애플리케이션의 `running` 상태를 확인합니다 

```code
kubectl get all
```

아래와 같은 결과를 확인할 수 있습니다.

```
NAME                           READY   STATUS    RESTARTS   AGE
pod/headers-67f468496f-ncf8s   1/1     Running   0          18s

NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/headers      ClusterIP   10.105.244.169   <none>        80/TCP    18s
service/kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP   268d

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/headers   1/1     1            1           18s

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/headers-67f468496f   1         1         1       18s
```

`Gateway` 오브젝트를 배포합니다. NGINX Gateway Fabric의 데이터플레인을 현재 `namespace`에 배포합니다.

```code
kubectl apply -f 1.gateway.yaml
```

NGINX Gateway Fabric 데이터플레인 pod의 상태를 확인합니다.

```
kubectl get pods
```

`gateway-nginx-c9bcdf4d4-j9pw5`는 NGINX Gateway Fabric 데이터플레인 pod 입니다.

```code
NAME                            READY   STATUS    RESTARTS   AGE
gateway-nginx-c9bcdf4d4-j9pw5   1/1     Running   0          49s
headers-67f468496f-ncf8s        1/1     Running   0          92s
```

Gateway 배포 정보를 확인합니다.

```code
kubectl get gateway
```

아래와 같은 결과를 확인할 수 있습니다.

```code
NAME      CLASS   ADDRESS      PROGRAMMED   AGE
gateway   nginx   10.99.25.2   True         4s
```

그리고 NGINX Gateway Fabric 데이터플레인의 서비스 설정 정보도 함께 확인합니다.

```code
kubectl get service
```

`gateway-nginx`는 NGINX Gateway Fabric 데이터플레인의 서비스입니다.

```code
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
gateway-nginx   NodePort    10.99.25.2       <none>        80:30344/TCP   4m19s
headers         ClusterIP   10.105.244.169   <none>        80/TCP         5m2s
kubernetes      ClusterIP   10.96.0.1        <none>        443/TCP        268d
```

이제 Gateway 테스트를 위한 HTTP routes를 배포합니다. 배포 전에 HTTP Route의 설정이 어떻게 되어 있는지를 코드(`2.httproute.yaml`)를 통해서 먼저 확인을 하시기 바랍니다.

```code
kubectl apply -f 2.httproute.yaml
```

HTTP routes 설정을 확인합니다.

```code
kubectl get httproute
```

아래와 같은 결과를 확인할 수 있습니다.

```code
NAME      HOSTNAMES              AGE
headers   ["echo.example.com"]   3s
```

NGINX Gateway Fabric의 데이터플레인 인스턴스의 IP와 HTTP PORT정보를 가져와서 변수로 저장합니다.

```code
export NGF_IP=`kubectl get pod -l app.kubernetes.io/instance=ngf -o json|jq '.items[0].status.hostIP' -r`
export HTTP_PORT=`kubectl get svc gateway-nginx -o jsonpath='{.spec.ports[0].nodePort}'`
```

NGINX Gateway Fabric 데이터플레인 인스턴스의 IP와 HTTP PORT 정보를 확인합니다.

```code
echo -e "NGF address: $NGF_IP\nHTTP port  : $HTTP_PORT"
```

아래와 같이 애플리케이션 2개의 HTTP headers 정보를 삽입하여 요청을 시도합니다.

```code
curl -i --resolve echo.example.com:$HTTP_PORT:$NGF_IP http://echo.example.com:$HTTP_PORT/nofilter -H "My-Cool-Header:my-client-value" -H "My-Overwrite-Header:dont-see-this" 
```

아래와 같은 결과를 확인할 수 있습니다.

```code
HTTP/1.1 200 OK
Server: nginx
Date: Thu, 18 Sep 2025 21:42:45 GMT
Content-Type: text/plain
Content-Length: 450
Connection: keep-alive

Headers:
  header 'Host' is 'echo.example.com:30177'
  header 'X-Forwarded-For' is '10.1.1.8'
  header 'X-Real-IP' is '10.1.1.8'
  header 'X-Forwarded-Proto' is 'http'
  header 'X-Forwarded-Host' is 'echo.example.com'
  header 'X-Forwarded-Port' is '80'
  header 'Connection' is 'close'
  header 'User-Agent' is 'curl/7.81.0'
  header 'Accept' is '*/*'
  header 'My-Cool-Header' is 'my-client-value'
  header 'My-Overwrite-Header' is 'dont-see-this'
```

요청 해더를 정리하면:

- User-Agent가 존재.
- My-Cool-header 해더는 "my-client-value"라는 단일 값을 가지고 있음.
- My-Overwrite-Header 해더는 "dont-see-this"라는 값을 가지고 있음.
- Accept-encoding 해더는 존재하지 않음.

그리고 응답 해더에서  `X-Header-Set`과 `X-Header-Add` 해더 정보가 없음을 미리 확인하고 다음 테스트를 진행합니다.

이번엔 테스트 애플리케이션에 필터 라우터가 적용되는 경로로 요청을 시도합니다. 

```code
curl -i --resolve echo.example.com:$HTTP_PORT:$NGF_IP http://echo.example.com:$HTTP_PORT/headers -H "My-Cool-Header:my-client-value" -H "My-Overwrite-Header:dont-see-this" 
```

결과는 아래와 같습니다. 

```code
HTTP/1.1 200 OK
Server: nginx
Date: Thu, 12 Jun 2025 11:09:02 GMT
Content-Type: text/plain
Content-Length: 495
Connection: keep-alive
X-Header-Add: this-is-the-appended-value
X-Header-Set: overwritten-value

Headers:
  header 'Accept-Encoding' is 'compress'
  header 'My-cool-header' is 'my-client-value,this-is-an-appended-value'
  header 'My-Overwrite-Header' is 'this-is-the-only-value'
  header 'Host' is 'echo.example.com:30344'
  header 'X-Forwarded-For' is '192.168.2.26'
  header 'X-Real-IP' is '192.168.2.26'
  header 'X-Forwarded-Proto' is 'http'
  header 'X-Forwarded-Host' is 'echo.example.com'
  header 'X-Forwarded-Port' is '80'
  header 'Connection' is 'close'
  header 'Accept' is '*/*'
```

위 결과를 보면 특정 요청에 대해서 필터가 적용되어 요청 해더가 신규로 삽입되거, 수정 또는 삭제를 할 수 있습니다.

- User-Agent가 삭제됨.
- My-Cool-header는 새로운 "my-client-vlaue"라는 값이 추가됨.
- My-Overwrite-Header는 기존 "dont-see-this" 값이 "this-is-the-only-value"라는 값으로 수정됨.
- Accept-encoding전송된 curl request요청에서 수정하지 않았으므로 변경되지 않음.

그리고 응답 해더는 다음과 같이 수정됨

- `X-Header-Set`해더는  `overwritten-value`라는 이름으로 설정됨
- `X-Header-Add` 해더에 `this-is-the-appended-value`라는 값이 추가됨
- `X-Header-Remove` 삭제됨

설정한 2.httproute.yaml 파일의 내용과 실제 테스트 결과를 유심히 확인하고 어떻게 동작을 한 것인가에 대해서 다시 한번 더 숙지 하시기 바랍니다. 여기까이 확인을 했다면 다음 랩을 위해 적용된 모든 설정을 삭제합니다.

```code
kubectl delete -f .
```
