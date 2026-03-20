# Ingress Controller Lab #4 - Traffic Splitting  

## Lab 개요 
* NGINX Ingress Controller 수준에서 JWT 인증을 적용하는 방법을 실습 

* 랩의 결과 
    - `coffee-v1-svc`와 `coffee-v2-svc`라는 두 개의 서비스를 가진 샘플 애플리케이션에 대한 트래픽 분할을 구성
    - `coffee` 애플리케이션 트래픽의 90%는 `coffee-v1-svc`로, 나머지 10%는 `coffee-v2-svc`로 전송됩니다.

<br>

---
## 실습 

### #1 샘플 서비스 배포 
```code
kubectl apply -f 0.cafe.yaml
```


### #2 NGINX Virtual Server 배포 
```code
kubectl apply -f 1.virtual-server.yaml
```

### #3 테스트 접속 수행
* 스크립트를 통한 접속 수행 ( 총 100회 접속 ) 
```code
bash test.sh

....(생략))
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   164  100   164    0     0  27333      0 --:--:-- --:--:-- --:--:-- 32800
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   164  100   164    0     0  32800      0 --:--:-- --:--:-- --:--:-- 32800
Summary of responses:
Coffee v1: 90 times
Coffee v2: 10 times
```
