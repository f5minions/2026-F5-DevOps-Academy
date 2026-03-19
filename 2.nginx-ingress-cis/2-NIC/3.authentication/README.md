# Ingress Controller Lab #3 - Authentication  

## Lab 개요 
* NGINX Ingress Controller 수준에서 JWT 인증을 적용하는 방법을 실습 

* 랩의 결과 
    - JWT Token 이 없는 접속은 401 Authorization Required Return
    - 올바른 JWT Token 이 있는 접속은 정상 접속 
    - 비정상 JWT Token 의 경우 401 Authorization Required Return

<br>

---

### 샘플 서비스 배포
```code
kubectl apply -f 0.webapp.yaml
```

### JWK Secret 배포
```code
kubectl apply -f 1.jwk-secret.yaml
```

### JWT Policy 배포 
```code
kubectl apply -f 2.jwt-policy.yaml
```

### NGINX Virtual Server 배포 
```code
kubectl apply -f 3.virtual-server.yaml
```

### 테스트 접속 수행
* 초도 접속 수행 -> 401 CODE 
```code
curl http://webapp.vs.example.com

<html>
<head><title>401 Authorization Required</title></head>
<body>
<center><h1>401 Authorization Required</h1></center>
<hr><center>nginx/1.27.2</center>
</body>
</html>
```
* 2차 접속 수행 (with JWT Token) 
```code
curl http://webapp.vs.example.com -H "token: `cat token.jwt`"

Server address: 10.221.0.59:8080
Server name: webapp-8598df94db-dmhrt
Date: 19/Mar/2026:08:59:23 +0000
URI: /
Request ID: 34e2d9643613ed57f5149605cf3a7c2b
```
* 3차 접속 수행 (with Abnormal JWT Token) 
```code
curl http://webapp.vs.example.com -H "token: `cat wrong.jwt`"

<html>
<head><title>401 Authorization Required</title></head>
<body>
<center><h1>401 Authorization Required</h1></center>
<hr><center>nginx/1.27.2</center>
</body>
</html>
```