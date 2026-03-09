# 🧪 NGINX Plus 블루-그린 배포 테스트 가이드

이 가이드는 **NGINX Plus** 환경에서 Blue/Green/Red 구성이 올바르게 동작하는지 검증합니다.  
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

## 🧭 1. 기본 연결 테스트

### 🔴 Red (기본값) — 헤더 없이 요청

```bash
curl -i http://10.1.1.11/
```

### 🔵 Blue 그룹으로 요청

```bash
curl -i -H "User: bluegroup" http://10.1.1.11/
```

### 🔵🟢 Blue-Green 혼합 요청

```bash
curl -i -H "User: bluegreen" http://10.1.1.11/
```

### 분산 비율 확인 (100회 반복 요청)

```bash
for i in {1..100}; do
  curl -s -o /dev/null -w "%{remote_ip}\n" -H "User: bluegreen" http://10.1.1.11/
done | sort | uniq -c
```

---

## 🧭 2. NGINX Plus API 테스트

### a. API 루트 확인

```bash
curl -s http://10.1.1.11:8080/api/6/ | jq .
```

### b. 전체 업스트림 목록 조회

```bash
curl -s http://10.1.1.11:8080/api/6/http/upstreams/ | jq .
```

### c. bluegreen 업스트림 상세 조회

```bash
curl -s http://10.1.1.11:8080/api/6/http/upstreams/bluegreen_bullet/servers/ | jq .
```

---

## 🚀 3. 점진적 전환 (동적 가중치 기반 프로모션)

NGINX Plus를 리로드하지 않고 API를 통해 blue → green으로 실시간 트래픽 비율을 조정할 수 있습니다!

### 예시: 50/50 분할로 전환

```bash
curl -X PATCH -d '{"weight":50}' http://10.1.1.11:8080/api/6/http/upstreams/bluegreen_bullet/servers/0
curl -X PATCH -d '{"weight":50}' http://10.1.1.11:8080/api/6/http/upstreams/bluegreen_bullet/servers/1
```

### 예시: Green 100% 전환 (완전 프로모션)

```bash
curl -X PATCH -d '{"weight":0}'   http://10.1.1.11:8080/api/6/http/upstreams/bluegreen_bullet/servers/0
curl -X PATCH -d '{"weight":100}' http://10.1.1.11:8080/api/6/http/upstreams/bluegreen_bullet/servers/1
```

### 변경된 가중치 확인

```bash
curl -s http://10.1.1.11:8080/api/6/http/upstreams/bluegreen_bullet/servers/ | jq '.[] | {server,weight}'
```

---

## 🔄 4. 롤백 절차 (즉시 복구)

```bash
curl -X PATCH -d '{"weight":95}' http://10.1.1.11:8080/api/6/http/upstreams/bluegreen_bullet/servers/0
curl -X PATCH -d '{"weight":5}'  http://10.1.1.11:8080/api/6/http/upstreams/bluegreen_bullet/servers/1
```

### 롤백 결과 확인

```bash
curl -s http://10.1.1.11:8080/api/6/http/upstreams/bluegreen_bullet/servers/ | jq '.[] | {server,weight}'
```

---

## 🩺 5. 헬스 체크 및 모니터링

### NGINX Plus 액티브 헬스 체크 및 런타임 통계 확인

```bash
curl -s http://10.1.1.11:8080/api/6/http/upstreams/bluegreen_bullet | jq .
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
