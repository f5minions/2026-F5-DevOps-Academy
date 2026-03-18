# NGINX Plus 블루-그린 배포 테스트 가이드

이 가이드는 **NGINX Plus** 환경에서 Blue-Green 배포 구성이 올바르게 동작하는지 검증합니다.  
HTTP 헤더 기반 트래픽 라우팅과 NGINX Plus API를 활용한 동적 가중치 점진적 전환(Gradual Rollout)을 포함합니다.

---

## ⚙️ 환경 개요

| 컴포넌트 | 설명 | 대상 예시 |
| --- | --- | --- |
| **NGINX Plus 게이트웨이** | `User` 헤더 값에 따라 트래픽을 라우팅 | `http://10.1.1.11` |
| **업스트림 풀** | `bluebullet`, `greenbullet`, `redbullet`, `bluegreen_bullet` | 설정 파일 참조 |
| **헤더 키** | `User` | 어느 업스트림이 요청을 처리할지 결정 |
| **API 포트** | `8080` | NGINX Plus 실시간 설정 API |

---

## NGINX 설정 살펴보기

User 헤더 기반으로 서로 다른 업스트림으로 요청을 프록시 하는 NGINX 설정을 살펴보겠습니다.

### bullets-rgb-1.conf ###

```nginx
# Define upstream pools for each environment
upstream bluebullet {
    zone bluebullet 512k;
    server 10.1.20.22:32444 max_fails=1 fail_timeout=10s;
}

upstream greenbullet {
    zone greenbullet 512k;
    server 10.1.20.22:31809 max_fails=1 fail_timeout=10s;
}

upstream redbullet {
    zone redbullet 512k;
    server 10.1.20.22:31092 max_fails=1 fail_timeout=10s;
}

upstream bluegreen_bullet {
    zone bluegreen_bullet 512k;
    server 10.1.20.22:32444 max_fails=1 fail_timeout=10s;
    server 10.1.20.22:31809 max_fails=1 fail_timeout=10s;
}

# Map header to backend selection
map $http_user $chosen_backend {
    bluegroup  bluebullet;
    greengroup greenbullet;
    bluegreen  bluegreen_bullet;
    default    redbullet;
}

server {
    listen 80;
    listen [::]:80;
    server_name bluegreen.example.com;

    location / {
        proxy_pass http://$chosen_backend;

        proxy_connect_timeout 60s;
        proxy_read_timeout 60s;
        proxy_send_timeout 60s;

        client_max_body_size 1m;
        proxy_buffering on;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_next_upstream error timeout;
    }

    location /health {
        access_log off;
        return 200 "OK\n";
        add_header Content-Type text/plain;
    }
}

```

위 설정에는 4개의 `upstream` 블록이 정의되어 있으며, `map` 블록은 HTTP `User` 헤더 값을 기반으로 요청을 프록시할 업스트림을 결정합니다. 

각 헤더 값에 따라 선택되는 업스트림 정보는 다음과 같습니다:

| `User` 헤더 값 | 라우팅 대상 (Upstream) | 비고 |
| :--- | :--- | :--- |
| `bluegroup` | `bluebullet` | |
| `greengroup` | `greenbullet` | |
| `bluegreen` | `bluegreen_bullet` | `bluebullet`, `greenbullet` 모두 포함 |
| `default` | `redbullet` | User 헤더가 없거나 기타 값일 경우 기본 upstream |

NIM UI에 접속하여 bullets-rgb-1.conf 설정을 적용합니다.

## UDF > Components > NIM > ACCESS > NIM UI

basics Lab과 동일하게, UDF 설명에 있는 사용자 이름과 비밀번호를 사용합니다.

![NIM-Access](images/NIM-UDF-sign-in.webp)

etc/nginx/conf.d/ 디렉토리에 bullets-rgb-1.conf를 생성합니다.

NIM UI > Instance Groups > "Add File" 클릭 후 파일 경로와 이름 입력: bullets-rgb-1.conf

![NIM-add-file](images/nim-bullets-rgb-conf-create.png)

bullets-rgb-1.conf 파일의 내용을 붙여넣고, Publish 버튼을 클릭하면 NGINX 인스턴스에 설정이 적용됩니다.

## 1. 기본 연결 확인

각 업스트림별 기본 연결을 확인하기 위해 UDF > Docker > FIREFOX에서 Firefox 브라우저를 사용합니다.

헤더를 추가하여 요청을 전송하기 위해 Firefox 브라우저에 확장 기능을 추가합니다.

![firefox-addon](images/firefox-addon.png)

headermod 확장 기능을 검색하고 추가합니다.

![firefox-search-addon](images/firefox-search-addon.png)

![firefox-add-headermod](images/firefox-add-headermod.png)

### 🔴 Red (기본값) — 헤더 없이 요청

브라우저에서 다음 주소로 접속합니다: http://10.1.1.11

![firefox-redgroup](images/firefox-redgroup.png)


### 🔵 Blue 그룹으로 요청

추가한 Headermod 확장 기능을 사용하여 User 헤더 값을 bluegroup으로 설정하고, http://10.1.1.11 로 접속합니다.

![firefox-header-bluegroup](images/set-user-header-bluegroup.png)

![firefox-bluegroup](images/firefox-bluegroup.png)


### 🔵🟢 Blue-Green 혼합 요청

User 헤더 값을 bluegreen으로 수정하고, http://10.1.1.11 로 접속합니다.

![firefox-header-bluegreen](images/set-user-header-bluegreen.png)  

![firefox-bluegreen](images/firefox-bluegreen.png)



브라우저를 새로고침 하여 Blue와 Green 서버로 트래픽이 분산되는 것을 확인합니다.

### 분산 비율 확인 (100회 반복 요청)

```bash
for i in {1..100}; do
  curl -s -o /dev/null -w "%{remote_ip}\n" -H "User: bluegreen" http://10.1.1.11/
done | sort
```

---

## 2. NGINX Plus API 테스트

NGINX Plus는 REST API를 통해 업스트림 설정을 조회하고 동적으로 변경할 수 있습니다.

basics Lab에서 구성한 dashboard.conf의 /api 경로로 NGINX Plus API를 사용합니다.

### dashboard.conf ###
```nginx
server {

    listen       8080;

    location /api {
        api write=on; # 동적 업데이트 허용
        allow all;
    }

    location / {
    root /usr/share/nginx/html;
    index   dashboard.html;
    }
}

```

UDF > Components > Nginx-plus-apigw > Access > WEB SHELL 접속 후 curl 명령어로 API를 호출합니다.

### a. API 루트 확인

```bash
curl -s http://10.1.1.11:8080/api/9/ | jq .
```

### b. 전체 업스트림 목록 조회

```bash
curl -s http://10.1.1.11:8080/api/9/http/upstreams/ | jq .
```

### c. bluegreen 업스트림 상세 조회

```bash
curl -s http://10.1.1.11:8080/api/9/http/upstreams/bluegreen_bullet/servers/ | jq .
```

업스트림 내부 서버에 할당된 id 값과 가중치(weight)를 확인합니다.

---

## 3. 점진적 전환 (동적 가중치 기반 프로모션)

NGINX Plus를 리로드하지 않고 API를 통해 blue → green으로 실시간 트래픽 비율을 조정할 수 있습니다!

### a: 20/80 분할로 전환 (Gradual Rollout)

NGINX Plus API를 사용하여 Blue 서버의 가중치를 낮추고 Green 서버의 가중치를 높여 결과를 확인합니다.

```bash
curl -X PATCH -d '{"weight":20}' http://10.1.1.11:8080/api/9/http/upstreams/bluegreen_bullet/servers/0
curl -X PATCH -d '{"weight":80}' http://10.1.1.11:8080/api/9/http/upstreams/bluegreen_bullet/servers/1
```

### b: Green 100% 전환 (완전 프로모션)

NGINX에서 `weight` 값은 1 이상이어야 하므로 0으로 설정할 수 없습니다. 트래픽을 완전히 차단하려면 `down` 또는 `drain` 속성을 사용합니다. **`drain`을 사용하면 기존 연결을 유지하면서 새로운 요청만 차단하여 실 서비스에 영향 없이 전환이 가능합니다.**

```bash
# Blue (servers/0) 서버로 가는 새로운 트래픽을 차단하고 기존 연결이 끝나기를 대기 (Graceful Shutdown)
curl -X PATCH -d '{"drain":true}'  http://10.1.1.11:8080/api/9/http/upstreams/bluegreen_bullet/servers/0
# Green (servers/1) 서버의 가중치를 100으로 설정
curl -X PATCH -d '{"weight":100}' http://10.1.1.11:8080/api/9/http/upstreams/bluegreen_bullet/servers/1
```

### 변경된 가중치 확인

```bash
curl -s http://10.1.1.11:8080/api/9/http/upstreams/bluegreen_bullet/servers/ | jq '.[] | {server,weight}'
```

---

## 🔄 4. 롤백 절차 (즉시 복구)

완전 프로모션 시 `drain` 처리했던 Blue 서버를 다시 활성화(`drain: false`)하면서 비중을 되돌립니다.

```bash
curl -X PATCH -d '{"drain":false, "weight":95}' http://10.1.1.11:8080/api/9/http/upstreams/bluegreen_bullet/servers/0
curl -X PATCH -d '{"weight":5}'                 http://10.1.1.11:8080/api/9/http/upstreams/bluegreen_bullet/servers/1
```

### 롤백 결과 확인

```bash
curl -s http://10.1.1.11:8080/api/9/http/upstreams/bluegreen_bullet/servers/ | jq '.[] | {server,weight}'
```

---

## 🩺 5. 헬스 체크 및 모니터링


```bash
curl -s http://10.1.1.11:8080/api/9/http/upstreams/bluegreen_bullet | jq .
```

---

## 📁 관련 파일 목록

| 파일명 | 설명 |
| --- | --- |
| `bullets-rgb-1.conf` | NGINX Plus 기본 라우팅 설정 (헤더 기반) |
| `bulls-rgb-1-w2.conf` | 가중치 적용 업스트림 설정 변형 |
| `rgb-2-2.conf` | RGB 2단계 설정 |
| `app-rgb-bulls-k8s-dep-svc.yaml` | Kubernetes Deployment 및 Service 매니페스트 |
| `apigw/crAPI/` | API 게이트웨이 crAPI 관련 설정 |

---

> 📌 **참고**: 원본 저장소 링크  
> https://github.com/f5minions/2026-devops-academy/tree/main/examples/basics2
