## NGINX Ingress Controller Lab Deploy 

사전 필수 
- F5 CIS Deploy 

### NGINX Ingress Controller Deploy 

    yum install nginx   

<br>

### #1 Sample App Deploy 

샘플 서비스로 활용할 Application 및 기본 Ingress / Virtual Server 리소스 배포 

    cd 2-NIC/1.basic-ingress
    kubectl apply -f 0.cafe.yaml
    kubectl apply -f 1.cafe-secret.yaml

<br>

### #2 기본 Ingress 및 Virtual Server 배포 


    kubectl apply -f 2.cafe-ingress.yaml
    kubectl apply -f 3.cafe-virtualserver.yaml 

서비스 검증

    kubectl get ingress 
    
<br>

### #3 Advanced Routing 서비스 배포 

v1/v2 서비스 구분을 위한 테스트 <br>

    cd 2-NIC/2.advanced-routing 
    kubectl apply -f 0.cafe.yaml 
    kubectl apply -f 1.cafe-secret.yaml

라우팅

    kubectl apply -f 2.advanced-routing.yaml

<br>

### #4 JWT Authentication 서비스 배포
JWT 에 대한 유효성 검사 

<br>

### #5Traffic Splitting 서비스 배포 
서비스에 대한 분리 9:1 설정 

<br>

### #6 Access Control 서비스 배포 <br>

특정 IP 대역에서 들어오는 서비스 차단 설정 

<br>

### #7 Ratelimiting 서비스 배포 
Rate-limit Policy 설정을 통한 트래픽에 대한 rate-limiting 설정 

