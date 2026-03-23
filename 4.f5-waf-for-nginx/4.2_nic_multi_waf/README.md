# F5 WAF for NGINX — LAB 4.2: 멀티 WAF (경로별 정책 분리)

> **실습 환경:** Client-vscode 터미널  
> **목표:** VirtualServer의 URL 경로별로 서로 다른 WAF 정책을 적용하여, 애플리케이션 단위의 독립적인 보안 제어를 구현한다.

---

## 목차

1. [실습 개요](#1-실습-개요)
2. [환경 구성](#2-환경-구성)
3. [WAF 정책 배포](#3-waf-정책-배포)
4. [VirtualServer 구성 및 배포](#4-virtualserver-구성-및-배포)
5. [동작 확인](#5-동작-확인)

---

## 1. 실습 개요

하나의 VirtualServer에서 URL 경로에 따라 서로 다른 WAF 정책을 적용합니다.

| 경로                        | 서비스     | WAF 정책       |
| --------------------------- | ---------- | -------------- |
| `/tea`                      | tea-svc    | **nap-tea**    |
| `/coffee`                   | coffee-svc | **nap-coffee** |
| 그 외 모든 경로 (Catch-all) | cocoa-svc  | **nap-cocoa**  |

각 정책은 차단 시 **정책 이름이 포함된 고유한 메시지**를 반환하므로, 어느 정책이 적용되었는지 응답만으로 바로 구분할 수 있습니다.

---

## 2. 환경 구성

### Step 1 · 작업 디렉토리 이동

> **실행 환경:** Client-vscode 터미널

```bash
cd /home/ubuntu/2026-F5-DevOps-Academy/4.f5-waf-for-nginx/4.2_nic_multi_waf
```

### Step 2 · 애플리케이션 배포 (`apps.yml`)

3개의 Deployment와 3개의 Service를 한 번에 생성합니다.

```bash
kubectl apply -f apps.yml

# 배포 상태 확인
kubectl get pod -n nap && kubectl get service -n nap
```

---

## 3. WAF 정책 배포

### Step 3 · APPolicy 생성 (경로별 3개)

각 경로에 독립적인 보안 정책을 적용하기 위해 APPolicy를 3개 생성합니다.  
정책마다 고유한 차단 메시지가 설정되어 있어, 응답으로 어느 정책이 동작했는지 바로 확인할 수 있습니다.

```bash
kubectl apply -f appolicy-coffee.yml
kubectl apply -f appolicy-tea.yml
kubectl apply -f appolicy-cocoa.yml

# 생성 확인
kubectl get APPolicy -n nap
```

### Step 4 · 로그 설정 (`log.yml`)

3개 정책이 공통으로 사용할 로그 설정입니다.  
LAB 4.1에서 이미 적용된 설정과 동일하므로 `unchanged` 출력은 정상입니다.

```bash
kubectl apply -f log.yml
kubectl get APLogConf -n nap
```

### Step 5 · NGINX Policy 생성 (경로별 3개)

APPolicy + APLogConf를 묶어 NIC에 적용할 최종 Policy를 생성합니다.  
로그는 Elasticsearch(Fluentd 경유)로 전송됩니다.

```bash
kubectl apply -f policy-coffee.yml
kubectl apply -f policy-tea.yml
kubectl apply -f policy-cocoa.yml

# 생성 확인
kubectl get Policy.k8s.nginx.org -n nap
```

---

## 4. VirtualServer 구성 및 배포

### Step 6 · `virtual-server.yml` 구성 확인

경로별 정책 분기가 핵심입니다. YAML의 주석을 참고하세요.

```bash
cat virtual-server.yml
```

```yaml
apiVersion: k8s.nginx.org/v1
kind: VirtualServer
metadata:
  name: nap-cafe
  namespace: nap-vs
spec:
  host: nap-cafe.f5k8s.net
  policies:
  - name: cocoa-policy        # Catch-all: 매칭되는 경로가 없을 때 적용
  upstreams:
  - name: cocoa
    service: cocoa-svc
    port: 80
  - name: coffee
    service: coffee-svc
    port: 80
  - name: tea
    service: tea-svc
    port: 80
  routes:
  - path: /                   # 기본 경로 → cocoa 정책 적용
    action:
      pass: webapp
  - path: /tea                # /tea 경로 전용 정책
    policies:
    - name: tea-policy
    action:
      pass: webapp
  - path: /coffee             # /coffee 경로 전용 정책
    policies:
    - name: coffee-policy
    action:
      pass: webapp
```

> **포인트:** 최상위 `policies`는 Catch-all 역할을 하며, 각 `routes` 안의 `policies`가 해당 경로에 우선 적용됩니다.

### Step 7 · VirtualServer 배포

```bash
kubectl apply -f virtual-server.yml

# 배포 상태 확인
kubectl get virtualserver.k8s.nginx.org -n nap
```

---

## 5. 동작 확인

XSS 공격 요청을 각 경로로 전송하여 정책이 올바르게 분리 적용되는지 확인합니다.

### `/tea` 경로 → `nap-tea` 정책 차단

```bash
curl "http://nap-cafe.f5k8s.net/tea/?user=<script>"
```

```html
<html><head><title>Custom Reject Page</title></head>
<body>Blocked from NAP policy: NAP-TEA
<br><br>Your support ID is: 13956862429059552813</body></html>
```

### `/coffee` 경로 → `nap-coffee` 정책 차단

```bash
curl "http://nap-cafe.f5k8s.net/coffee/?user=<script>"
```

```html
<html><head><title>Custom Reject Page</title></head>
<body>Blocked from NAP policy: NAP-COFFEE
<br><br>Your support ID is: 9293616283769441734</body></html>
```

### 그 외 경로 (`/unknown`) → `nap-cocoa` 정책 차단 (Catch-all)

```bash
curl "http://nap-cafe.f5k8s.net/unknown/?user=<script>"
```

```html
<html><head><title>Custom Reject Page</title></head>
<body>Blocked from NAP policy: NAP-COCOA
<br><br>Your support ID is: 13980750205359556035</body></html>
```

### 결과 요약

| 요청 경로                 | 적용된 정책           | 차단 메시지  |
| ------------------------- | --------------------- | ------------ |
| `/tea/?user=<script>`     | nap-tea               | `NAP-TEA`    |
| `/coffee/?user=<script>`  | nap-coffee            | `NAP-COFFEE` |
| `/unknown/?user=<script>` | nap-cocoa (Catch-all) | `NAP-COCOA`  |

경로별로 독립적인 WAF 정책이 올바르게 적용된 것을 확인했습니다.

---

*LAB 4.2 End*