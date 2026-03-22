# Lab 3 — HTTP 요청 및 응답 헤더 수정

> HTTPRoute 필터를 통해 요청/응답 헤더를 추가·수정·삭제하는 방법을 실습합니다.

---

## 실습 경로 이동

```bash
# 이전 Lab 디렉토리에 있다면
cd ../3.nginx-gateway-fabric/labs/3.http-headers

# 실습 메인 디렉토리(2026-F5-DevOps-Academy)에 있다면
cd 3.nginx-gateway-fabric/labs/3.http-headers
```

---

## Step 1 — 테스트 애플리케이션 배포

```bash
kubectl apply -f 0.app.yaml
```

배포 상태가 `Running`인지 확인합니다.

```bash
kubectl get all
```

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

---

## Step 2 — Gateway 배포

NGINX Gateway Fabric 데이터플레인을 현재 `namespace`에 배포합니다.

```bash
kubectl apply -f 1.gateway.yaml
```

### Pod 상태 확인

```bash
kubectl get pods
```

`gateway-nginx-c9bcdf4d4-j9pw5`는 NGINX Gateway Fabric 데이터플레인 Pod입니다.

```
NAME                            READY   STATUS    RESTARTS   AGE
gateway-nginx-c9bcdf4d4-j9pw5   1/1     Running   0          49s
headers-67f468496f-ncf8s        1/1     Running   0          92s
```

### Gateway 상태 확인

```bash
kubectl get gateway
```

```
NAME      CLASS   ADDRESS      PROGRAMMED   AGE
gateway   nginx   10.99.25.2   True         4s
```

### Service 상태 확인

`gateway-nginx`는 NGINX Gateway Fabric 데이터플레인의 서비스입니다.

```bash
kubectl get service
```

```
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
gateway-nginx   NodePort    10.99.25.2       <none>        80:30344/TCP   4m19s
headers         ClusterIP   10.105.244.169   <none>        80/TCP         5m2s
kubernetes      ClusterIP   10.96.0.1        <none>        443/TCP        268d
```

---

## Step 3 — HTTPRoute 배포

배포 전 `2.httproute.yaml` 파일의 헤더 필터 설정을 먼저 확인해보시기 바랍니다.

```bash
kubectl apply -f 2.httproute.yaml
```

```bash
kubectl get httproute
```

```
NAME      HOSTNAMES              AGE
headers   ["echo.example.com"]   3s
```

---

## Step 4 — 테스트

### 환경 변수 설정

```bash
export NGF_IP=`kubectl get pod -l app.kubernetes.io/instance=ngf -o json|jq '.items[0].status.hostIP' -r`
export HTTP_PORT=`kubectl get svc gateway-nginx -o jsonpath='{.spec.ports[0].nodePort}'`
```

```bash
echo -e "NGF address: $NGF_IP\nHTTP port  : $HTTP_PORT"
```

---

### 테스트 1 — 필터 미적용 경로 (`/nofilter`)

헤더 필터가 없는 경로로 요청하여 원본 헤더를 확인합니다.

```bash
curl -i --resolve echo.example.com:$HTTP_PORT:$NGF_IP http://echo.example.com:$HTTP_PORT/nofilter -H "My-Cool-Header:my-client-value" -H "My-Overwrite-Header:dont-see-this"
```

```
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

필터 미적용 상태의 요청 헤더 특징:

- `User-Agent` 존재
- `My-Cool-Header`는 `my-client-value` 단일 값
- `My-Overwrite-Header`는 `dont-see-this` 값 유지
- 응답 헤더에 `X-Header-Set`, `X-Header-Add` 없음

---

### 테스트 2 — 필터 적용 경로 (`/headers`)

헤더 필터가 적용된 경로로 동일한 요청을 전송합니다.

```bash
curl -i --resolve echo.example.com:$HTTP_PORT:$NGF_IP http://echo.example.com:$HTTP_PORT/headers -H "My-Cool-Header:my-client-value" -H "My-Overwrite-Header:dont-see-this"
```

```
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

필터 적용 후 변경 내용:

**요청 헤더:**
- `User-Agent` — 삭제됨
- `My-Cool-Header` — 기존 값에 `this-is-an-appended-value` 추가됨
- `My-Overwrite-Header` — `dont-see-this` → `this-is-the-only-value`로 덮어쓰기됨
- `Accept-Encoding` — `compress`로 새로 추가됨

**응답 헤더:**
- `X-Header-Set` — `overwritten-value`로 설정됨
- `X-Header-Add` — `this-is-the-appended-value` 값 추가됨
- `X-Header-Remove` — 삭제됨

> `2.httproute.yaml` 파일의 설정 내용과 테스트 결과를 비교하며 동작 방식을 확인해보시기 바랍니다.

---

## 실습 종료 — 리소스 삭제

```bash
kubectl delete -f .
```
