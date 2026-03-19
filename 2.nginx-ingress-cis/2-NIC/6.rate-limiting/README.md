# Ingress Controller Lab #6 - Rate-Limiting

## Lab 개요 
* NGINX Ingress Controller를 통해 노출된 애플리케이션에 대한 속도 제한(rate-limit)을 적용합니다.

* 랩의 결과 
    - 기준치 이상의 트래픽 인입시 429 Return 처리 

<br>

---

### 샘플 서비스 배포
```code
kubectl apply -f 0.webapp.yaml
```

### 샘플 정책 배포 - Deny 
```code
kubectl apply -f 1.rate-limit.yaml
```

### NGINX Virtual Server 배포 
```code
kubectl apply -f 2.virtual-server.yaml
```

### 테스트 접속 수행
* 배포된 Virtual Server 접속 - 429 Reponse 여부 확인 
    ```code
    curl webapp.vs.example.com

    <html>
    <head><title>429 Too Many Requests</title></head>
    <body>
    <center><h1>429 Too Many Requests</h1></center>
    <hr><center>nginx/1.27.2</center>
    </body>
    </html>
    ```

### 샘플 정책 배포 - Allow
```code
kubectl apply -f 3.rate-limit-mitigate.yaml
```

### 테스트 접속 수행
* 완화 정책 배포된 Virtual Server 접속 - 지속적 정상 접속 가능 
    ```code
    curl webapp.vs.example.com

    Server address: 10.221.0.59:8080
    Server name: webapp-8598df94db-dmhrt
    Date: 19/Mar/2026:09:13:16 +0000
    URI: /
    Request ID: f9e4fc1ac1dde39b238236d944476aa6
    ```