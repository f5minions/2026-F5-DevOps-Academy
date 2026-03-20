# F5 WAF for NGINX — LAB 4.3: NIM을 통한 WAF 정책 중앙 관리

> **실습 환경:** NIM UI + Nginx-plus-apigw 터미널
> **목표:** NGINX Instance Manager(NIM)를 사용하여 기존 NGINX Plus 인스턴스에 WAF 정책을 중앙에서 관리·배포하고, 예외 처리를 위한 Custom Policy를 생성한다.

---

## 목차

1. [실습 개요](#1-실습-개요)
2. [NIM 접속](#2-nim-접속)
3. [NGINX Plus 인스턴스 NAP 구성 상태 확인](#3-nginx-plus-인스턴스-nap-구성-상태-확인)
4. [NIM UI로 NAP Policy 적용](#4-nim-ui로-nap-policy-적용)
5. [NIM 이벤트 모니터링](#5-nim-이벤트-모니터링)
6. [Custom Policy 생성 및 배포 (예외 처리)](#6-custom-policy-생성-및-배포-예외-처리)

---

## 1. 실습 개요

| 구분            | LAB 4.1 / 4.2            | LAB 4.3                           |
| --------------- | ------------------------ | --------------------------------- |
| 관리 방식       | K8S CRD (kubectl apply)  | NIM UI (중앙 관리)                |
| 대상 환경       | NGINX Ingress Controller | NGINX Plus 단독 인스턴스          |
| 정책 형식       | APPolicy YAML            | JSON (NIM이 컴파일 후 배포)       |
| 이벤트 모니터링 | Grafana                  | NIM Security Monitoring Dashboard |

> NIM은 Policy를 **JSON → 바이너리로 중앙 컴파일**한 뒤 각 인스턴스에 배포하므로, 인스턴스마다 별도 컴파일 없이 일관된 정책을 유지할 수 있습니다.

---

## 2. NIM 접속

1. 실습 환경에서 제공된 NIM 접속 정보(URL, 계정/비밀번호)를 확인합니다.

<p align="center">
  <img src="images/nim-info1.png">
</p>

<p align="center">
  <img src="images/nim-info2.png">
</p>

2. 브라우저에서 NIM UI에 로그인합니다.

<p align="center">
  <img src="images/nim-info3.png">
</p>

<p align="center">
  <img src="images/nim-info4.png">
</p>

## 3. NGINX Plus 인스턴스 NAP 구성 상태 확인

### Step 1 · NIM에서 인스턴스 및 WAF 상태 확인

NIM UI에서 등록된 NGINX Plus 인스턴스를 조회하고, 현재 WAF(NAP) 구성 상태를 확인합니다.

> NAP이 설치되어 있음은 확인되지만, 실제로 활성화되어 있는지는 Config 파일을 별도로 확인해야 합니다.

<p align="center">
  <img src="images/nim-info5.png">
</p>

<p align="center">
  <img src="images/nim-info6.png">
</p>

<p align="center">
  <img src="images/nim-info7.png">
</p>

### Step 2 · NGINX Plus Config 파일 확인

NIM UI에서 해당 인스턴스의 설정 파일을 열어 NAP 컨텍스트 존재 여부를 확인합니다.

현재 `nginx.conf`에는 아래 NAP 관련 디렉티브가 **포함되어 있지 않습니다.**

```nginx
# NAP 활성화에 필요한 컨텍스트 (현재 없는 상태)
app_protect_enable on;
app_protect_policy_file /etc/nginx/waf/nac-policies/nap_nap-coffee;
app_protect_security_log_enable on;
app_protect_security_log /etc/nginx/waf/nac-logconfs/nap_logconf syslog:server=fluentd-svc.fluentd:8515;
app_protect_security_log /etc/nginx/waf/nac-logconfs/nap_logconf syslog:server=127.0.0.1:1514;
```

<p align="center">
  <img src="images/nim-info8.png">
</p>
<p align="center">
  <img src="images/nim-info9.png">
</p>

### Step 3 · XSS 공격으로 WAF 비활성 상태 검증

> **실행 환경:** Nginx-plus-apigw 터미널

```bash
# WAF가 없으면 공격 요청이 그대로 통과됨
curl "http://localhost:9000/?test=<script>"
```

요청이 차단되지 않고 정상 응답이 반환되면, WAF가 아직 동작하지 않는 상태입니다.

## 4. NIM UI를 활용한 NAP Policy 적용

### Step 1 · 관리 중인 Policy 확인

NIM UI의 **WAF → Policies** 메뉴에서 현재 등록된 Policy 목록을 확인합니다.
이번 실습에서는 기본 제공 Policy인 **NginxDefaultPolicy**를 사용합니다.

<p align="center">
  <img src="images/nim-info10.png">
</p>

<p align="center">
  <img src="images/nim-info11.png">
</p>

<p align="center">
  <img src="images/nim-info12.png">
</p>

### Step 2 · NIM에 Log 프로파일 등록

NIM은 Policy는 UI로 관리할 수 있지만, **Log 프로파일은 API를 통해서만 등록**할 수 있습니다.
NIM API Swagger 페이지에 접속하여 아래 JSON을 사용해 Log 프로파일을 등록합니다.
API 요청 Body에 아래 JSON을 그대로 입력합니다. JSON 파일은 `~/Academy/2026-F5-DevOps-Academy/4.f5-waf-for-nginx/4.3_nim_waf_management/api_data.json` 파일에서도 확인 가능합니다.

```json
{
  "content": "ewogICJmaWx0ZXIiOiB7CiAgICAicmVxdWVzdF90eXBlIjogImlsbGVnYWwiCiAgfSwKICAiY29udGVudCI6IHsKICAgICJmb3JtYXQiOiAidXNlci1kZWZpbmVkIiwKICAgICJmb3JtYXRfc3RyaW5nIjogIiVibG9ja2luZ19leGNlcHRpb25fcmVhc29uJSwlZGVzdF9wb3J0JSwlaXBfY2xpZW50JSwlaXNfdHJ1bmNhdGVkX2Jvb2wlLCVtZXRob2QlLCVwb2xpY3lfbmFtZSUsJXByb3RvY29sJSwlcmVxdWVzdF9zdGF0dXMlLCVyZXNwb25zZV9jb2RlJSwlc2V2ZXJpdHklLCVzaWdfY3ZlcyUsJXNpZ19zZXRfbmFtZXMlLCVzcmNfcG9ydCUsJXN1Yl92aW9sYXRpb25zJSwlc3VwcG9ydF9pZCUsJXRocmVhdF9jYW1wYWlnbl9uYW1lcyUsJXZpb2xhdGlvbl9yYXRpbmclLCV2c19uYW1lJSwleF9mb3J3YXJkZWRfZm9yX2hlYWRlcl92YWx1ZSUsJW91dGNvbWUlLCVvdXRjb21lX3JlYXNvbiUsJXZpb2xhdGlvbnMlLCV2aW9sYXRpb25fZGV0YWlscyUsJWJvdF9zaWduYXR1cmVfbmFtZSUsJWJvdF9jYXRlZ29yeSUsJWJvdF9hbm9tYWxpZXMlLCVlbmZvcmNlZF9ib3RfYW5vbWFsaWVzJSwlY2xpZW50X2NsYXNzJSwlY2xpZW50X2FwcGxpY2F0aW9uJSwlY2xpZW50X2FwcGxpY2F0aW9uX3ZlcnNpb24lLCV0cmFuc3BvcnRfcHJvdG9jb2wlLCV1cmklLCVyZXF1ZXN0JSIsCiAgICAiZXNjYXBpbmdfY2hhcmFjdGVycyI6IFsKICAgICAgewogICAgICAgICJmcm9tIjogIiwiLAogICAgICAgICJ0byI6ICIlMkMiCiAgICAgIH0KICAgIF0sCiAgICAibWF4X3JlcXVlc3Rfc2l6ZSI6ICIyMDQ4IiwKICAgICJtYXhfbWVzc2FnZV9zaXplIjogIjIwayIsCiAgICAibGlzdF9kZWxpbWl0ZXIiOiAiOjoiCiAgfQp9Cg==",
  "metadata": {
    "name": "custom_log"
  }
}
```

> `content` 필드의 값은 Log 프로파일 설정이 **base64로 인코딩**된 문자열입니다.
> 실제 내용이 궁금하다면 아래 명령으로 디코딩하여 확인할 수 있습니다.
>
> ```bash
> echo "<content 값>" | base64 --decode
> ```

<p align="center">
  <img src="images/nim-swagger1.png">
</p>

<p align="center">
  <img src="images/nim-swagger2.png">
</p>

<p align="center">
  <img src="images/nim-swagger3.png">
</p>

<p align="center">
  <img src="images/nim-swagger4.png">
</p>

### Step 3 · `nginx.conf`에 NAP 디렉티브 추가

NIM UI에서 해당 인스턴스의 설정 파일을 편집하여 아래 내용을 추가합니다.

```nginx
# 최상단: NAP 모듈 로드
load_module modules/ngx_http_app_protect_module.so;

http {
    app_protect_enable on;
    app_protect_policy_file /etc/nms/NginxDefaultPolicy.tgz;   # NIM에서 배포한 Policy 참조
    app_protect_security_log_enable on;
    app_protect_security_log /etc/nms/custom_log.tgz syslog:server=localhost:514;  # 로그를 NIM으로 전송
}
```

설정 저장 후 NIM이 자동으로 NGINX Plus에 배포합니다.
배포 완료 메시지를 확인합니다.

<p align="center">
  <img src="images/nim-info13.png">
</p>

<p align="center">
  <img src="images/nim-info14.png">
</p>

<p align="center">
  <img src="images/nim-info15.png">
</p>

<p align="center">
  <img src="images/nim-info16.png">
</p>

<p align="center">
  <img src="images/nim-info17.png">
</p>

### Step 4 · XSS 공격으로 WAF 활성화 검증

> **실행 환경:** Nginx-plus-apigw 터미널

```bash
curl "http://localhost:9000/?test=<script>"
```

```html
<!-- WAF가 정상 동작하여 차단됨 -->
<html><head><title>Request Rejected</title></head>
<body>The requested URL was rejected. Please consult with your administrator.
<br><br>Your support ID is: 5627465829415946071</body></html>
```

---

## 5. NIM 이벤트 모니터링

NIM UI의 **Security Monitoring Dashboard**에서 NAP이 탐지한 보안 이벤트를 확인합니다.

### 이벤트 조회 방법

1. NIM UI → **WAF → Security Dashboard** 진입
2. **Status** 필터를 `Blocked`로 설정하여 차단 이벤트만 표시
3. **Event logs** → 상세 정보 확인

> NIM에 이벤트 로그가 업데이트되기 까지 다소 시간이 소요될 수 있습니다.

이벤트 상세 화면에서 **Request Body**, **Attack Details** 등 다양한 정보를 확인할 수 있습니다.

> Grafana와 달리 NIM Dashboard는 별도 설정 없이 WAF 이벤트를 바로 조회할 수 있어 운영 편의성이 높습니다.

---

<p align="center">
  <img src="images/nim-info18.png">
</p>
<p align="center">
  <img src="images/nim-info19.png">
</p>
<p align="center">
  <img src="images/nim-info20.png">
</p>

<p align="center">
  <img src="images/nim-info21.png">
</p>
<p align="center">
  <img src="images/nim-info22.png">
</p>

## 6. Custom Policy 생성 및 배포 (예외 처리)

시스템 운영에 필요한 정상 요청이 WAF에 차단될 경우, 해당 시그니처를 예외 처리한 **Custom Policy**를 생성합니다.

### Step 1 · 차단 요청 전송 및 Support ID 확인

> **실행 환경:** Nginx-plus-apigw 터미널

```bash
# 시스템 필수 요청으로 가정 (OS 커맨드 인젝션 패턴)
curl "http://localhost:9000?id=0;%20ls%20-l"
```

```html
<!-- 차단됨 — Support ID를 복사해둠 -->
<html><head><title>Request Rejected</title></head>
<body>...Your support ID is: 17707074359114218079...</body></html>
```

### Step 2 · NIM에서 Support ID로 위반 시그니처 조회

1. NIM UI → **WAF → Support ID Details** 진입
2. 복사한 `support id` 입력 후 검색
3. 상세 페이지에서 **위반된 Attack Signature ID** 확인 및 복사

> NIM에 이벤트 로그가 업데이트되기 까지 다소 시간이 소요될 수 있습니다.

<p align="center">
  <img src="images/nim-info23.png">
</p>
<p align="center">
  <img src="images/nim-info24.png">
</p>
<p align="center">
  <img src="images/nim-info25.png">
</p>
<p align="center">
  <img src="images/nim-info26.png">
</p>

### Step 3 · Custom Policy 생성

1. NIM UI → **WAF → Policies** 에서 **NginxDefaultPolicy** 선택 후 **복사(Save As)**
2. 새 Policy 이름을 `custom_policy`로 지정하여 저장

<p align="center">
  <img src="images/nim-info27.png">
</p>
<p align="center">
  <img src="images/nim-info28.png">
</p>

<p align="center">
  <img src="images/nim-info29.png">
</p>

### Step 4 · Custom Policy에 시그니처 예외 추가

1. `custom_policy` 선택 후 **편집(Edit)** 진입
2. **Attack Signature Exceptions** 탭 확장
3. **Add Item** 클릭 → 복사한 Signature ID 검색 후 선택
4. 총 3개의 Signature ID 추가 확인 후 저장

> NIM UI에서 최종 적용된 Policy 내용을 **JSON 형식**으로 미리 볼 수 있습니다.
> JSON 파일을 직접 편집하지 않고 UI만으로 Policy를 수정·관리할 수 있는 것이 NIM의 핵심 장점입니다.

<p align="center">
  <img src="images/nim-info30.png">
</p>
<p align="center">
  <img src="images/nim-info31.png">
</p>
<p align="center">
  <img src="images/nim-info32.png">
</p>
<p align="center">
  <img src="images/nim-info33.png">
</p>

<p align="center">
  <img src="images/nim-info34.png">
</p>

<p align="center">
  <img src="images/nim-info35.png">
</p>
<p align="center">
  <img src="images/nim-info36.png">
</p>
<p align="center">
  <img src="images/nim-info37.png">
</p>

### Step 5 · Policy 컴파일

저장한 `custom_policy`를 **Compile**합니다.

> **컴파일이란?** JSON으로 작성된 Policy를 NAP 엔진이 실제로 적용할 수 있는 바이너리 형식으로 변환하는 과정입니다.
> NIM이 중앙에서 컴파일 후 각 인스턴스에 배포하므로, 인스턴스마다 별도 컴파일이 필요 없습니다.

<p align="center">
  <img src="images/nim-info38.png">
</p>

<p align="center">
  <img src="images/nim-info39.png">
</p>

### Step 6 · NGINX Plus에 Custom Policy 적용

NIM UI에서 해당 인스턴스의 `nginx.conf`를 수정합니다.
`app_protect_policy_file` 경로를 `custom_policy.tgz`로 변경합니다.

```nginx
http {
    app_protect_policy_file /etc/nms/custom_policy.tgz;   # NginxDefaultPolicy → custom_policy로 교체
}
```

설정 저장 후 배포 완료를 확인합니다.

<p align="center">
  <img src="images/nim-info40.png">
</p>

<p align="center">
  <img src="images/nim-info41.png">
</p>
<p align="center">
  <img src="images/nim-info42.png">
</p>

<p align="center">
  <img src="images/nim-info43.png">
</p>

<p align="center">
  <img src="images/nim-info44.png">
</p>

### Step 7 · 예외 처리 확인

> **실행 환경:** Nginx-plus-apigw 터미널

```bash
curl "http://localhost:9000?id=0;%20ls%20-l"
```

차단 없이 정상 응답이 반환되면 예외 처리가 성공적으로 적용된 것입니다.

---

*LAB 4.3 End*
