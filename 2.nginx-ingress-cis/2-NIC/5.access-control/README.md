# Ingress Controller Lab #5 - Access Control  

## Lab 개요 
* 특정 서브넷에서 들어오는 트래픽을 허용하거나 차단하기 위해 액세스 제어 정책을 적용합니다.

* 실습 결과 
    - `coffee-v1-svc`와 `coffee-v2-svc`라는 두 개의 서비스를 가진 샘플 애플리케이션에 대한 트래픽 분할을 구성
    - `coffee` 애플리케이션 트래픽의 90%는 `coffee-v1-svc`로, 나머지 10%는 `coffee-v2-svc`로 전송됩니다.

<br>

---

## 실습 

### #1 샘플 서비스 배포
```code
kubectl apply -f 0.webapp.yaml
```

### #2 샘플 정책 배포 - Access Deny 
```code
kubectl apply -f 1.access-control-policy-deny.yaml
```

### #3 NGINX Virtual Server 배포 
```code
kubectl apply -f 2.virtual-server.yaml
```

### #4 테스트 접속 수행
* 배포된 Virtual Server 접속 
    ```code
    curl webapp.vs.example.com

    <html>
    <head><title>403 Forbidden</title></head>
    <body>
    <center><h1>403 Forbidden</h1></center>
    <hr><center>nginx/1.27.2</center>
    </body>
    </html>
    ```

### #5 샘플 정책 배포 - Access Allow
```code
kubectl apply -f 3.access-control-policy-allow.yaml
```

### #6 테스트 접속 수행
* allow 정책 배포된 Virtual Server 접속 
    ```code
    curl webapp.vs.example.com

    Server address: 10.221.0.59:8080
    Server name: webapp-8598df94db-dmhrt
    Date: 19/Mar/2026:09:13:16 +0000
    URI: /
    Request ID: f9e4fc1ac1dde39b238236d944476aa6
    ```