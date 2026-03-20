# 2026 F5 DevOps Academy Lab

F5 DevOps Academy Lab으로 모던 아키텍처의 강력한 기능을 활용해 보세요! 이 저장소는 NGINX, BIG-IP, CIS를 포함한 F5 솔루션의 잠재력을 최대한 활용할 수 있도록 제공하는 통합 가이드입니다. 숙련된 DevOps 엔지니어이든 애플리케이션 전송 및 보안의 광범위한 세계를 탐색하는 초보자이든, 이 저장소는 예제, 사용 사례, 데모를 위한 핵심 참고 자료입니다.

F5 기술이 모던 아키텍처의 요구 사항을 충족하도록 인프라를 어떻게 향상시킬 수 있는지 보여주는 다양한 예제와 데모 컬렉션을 살펴보세요.

마이크로서비스 구축, 컨테이너 배포, 쿠버네티스 클러스터 오케스트레이션 등 어떤 작업을 하든 F5 기술은 프로젝트를 강화할 수 있는 강력한 솔루션을 제공합니다. 현대 IT 환경의 복잡성을 헤쳐나가는 여정에서 이 저장소를 길잡이로 삼아, 탄력적이고 확장 가능하며 안전한 솔루션을 설계하는 데 필요한 지식과 도구를 얻어가시기 바랍니다.

**F5 DevOps Academy Lab**은 F5의 [UDF 환경](https://udf.f5.com/b/a0fae003-69b4-4cb4-84b0-c68633f5b2c7)에서만 배포할 수 있습니다. F5 DevOps Academy Lab은 F5의 EMEA 솔루션 아키텍트 팀이 생성하고 관리하는 저장소에 기반하여 수정되었습니다.

F5 DevOps Academy Lab 환경의 상위 수준 다이어그램과 이 랩에서 사용되는 기술들은 아래에서 확인하실 수 있습니다.

<p align="center">
  <img src="images/udf-lab.png" 
</p>

## 기술 스택

이 환경에서 사용되는 기술들과 접속 자격증명은 아래 표에 정리되어 있습니다.

| 이름                    | 설명                                                                                                                                                                                                                                                                                                                                             | 자격증명              |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------- |
| **BIGIP**         | LTM/ASM/DNS 모듈이 프로비저닝된 독립형 BIGIP입니다.`<br>` *UI 접속: UDF의 `F5 products`에서 `bigip-01` -> `TMUI`로 이동*                                                                                                                                                                                                               | admin / Ingresslab123 |
| **NGINX IC**      | 두 K8s 클러스터 내에서 실행되며 최신 버전으로 자주 업데이트됩니다. Helm Chart를 통해 설치됩니다.                                                                                                                                                                                                                                                 | -                     |
| **NGINX Plus**    | NGINX의 상용 버전으로 가상 머신에 배포되었으며, Nginx-Agent를 통해 NIM에서 관리됩니다.                                                                                                                                                                                                                                                           | -                     |
| **NIM**           | Nginx Instance Manager가 가상 머신에 설치되어 있습니다.`<br>` *UI 접속: UDF의 `Systems`에서 `NIM` -> `NIM UI`로 이동*                                                                                                                                                                                                                  | admin / Ingresslab123 |
| **CIS**           | K8s 클러스터 내에서 실행됩니다. K3s-1에 CIS 인스턴스 2개, K3s-2에 1개가 실행 중입니다. K3s-1에는 VirtualServer/TransportServer CRD 및 ServiceType LB 기반 서비스 배포에 사용되는 `cis-crd` 인스턴스와, Ingress 리소스 및 ConfigMap에 사용되는 `cis-ingress` 인스턴스가 있습니다. K3s-2에는 cis-crd만 있습니다. 매니페스트를 통해 설치됩니다. | -                     |
| **K8s 클러스터**  | VSCode에서 접근 가능한 단일 노드 쿠버네티스 클러스터 2개(K3s-1, K3s-2)를 운영합니다. 두 클러스터 모두 버전 1.31을 실행 중입니다.                                                                                                                                                                                                                 | -                     |
| **VSCode**        | "Client" 시스템에서 웹 인터페이스를 통해 VSCode를 실행합니다.`<br>` *UI 접속: UDF의 `Systems`에서 `Client` -> `VSCode`로 이동*                                                                                                                                                                                                         | -                     |
| **Kubectl**       | 두 클러스터에 편리하게 접근하기 위해 `Client` 가상 머신에 `Kubectl`을 설치하고 `.kube/config`로 두 클러스터의 접근을 설정했습니다. `<br>` *UI 접속: UDF의 `Systems`에서 `Client` -> `VSCode`로 이동*                                                                                                                             | -                     |
| **GitLab**        | 전용 서버에서 실행되며 세 가지 주요 기능을 제공합니다:`<br>` - 소스 코드 관리 (git.f5k8s.net) `<br>` - GitLab Runner (CI/CD) `<br>` - 컨테이너 레지스트리 (registry.f5k8s.net) `<br>` *UI 접속: UDF의 `Systems`에서 `Docker` -> `Gitlab`으로 이동*                                                                               | root / Ingresslab123  |
| **ArgoCD**        | Argo CD는 쿠버네티스 컨트롤러로 구현되어 실행 중인 애플리케이션을 지속적으로 모니터링하고, 현재 실제 상태와 Git 저장소에 정의된 목표 상태를 비교합니다.`<br>` *UI 접속: F5 products에서 `bigip-01` -> `ArgoCD`로 이동*                                                                                                                   | admin / Ingresslab123 |
| **Elasticsearch** | "Docker" 시스템의 인스턴스로 실행되며, NAP, NGINX, BIGIP의 접속·에러·보안 로그를 저장하는 것이 주목적입니다.                                                                                                                                                                                                                                   | -                     |
| **Logstash**      | "Docker" 시스템의 인스턴스로 실행되며, 로그를 처리·파싱한 후 Elasticsearch로 전달하는 것이 주목적입니다.                                                                                                                                                                                                                                        | -                     |
| **Kibana**        | "Docker" 시스템의 인스턴스로 실행되며, Elasticsearch 로그를 시각화하는 역할을 합니다.`<br>` *UI 접속: UDF의 `Systems`에서 `Docker` -> `Kibana`로 이동*                                                                                                                                                                                 | -                     |
| **Prometheus**    | K3s-1에서 실행되며 BIGIP와 NGINX+를 모니터링하기 위한 시계열 데이터 저장소를 제공합니다. Helm을 통해 설치되며 CIS TransportServer를 통해 노출됩니다.`<br>` *UI 접속: UDF의 `Systems`에서 `k3s-1` -> `Prometheus`로 이동*                                                                                                               | -                     |
| **Grafana**       | Prometheus와 Elasticsearch의 메트릭/이벤트를 표시하는 다양한 대시보드가 구성되어 있습니다. K3s-1에서 실행되며 Helm을 통해 설치되고 CIS TransportServer를 통해 노출됩니다.`<br>` *UI 접속: UDF의 `Systems`에서 `k3s-1` -> `Grafana`로 이동*                                                                                             | admin / Ingresslab123 |
| **AWX**           | Ansible Tower의 오픈소스 버전으로 K3s-2에 설치되었으며 CIS TransportServer를 통해 노출됩니다.`<br>` *UI 접속: F5 products에서 `bigip-01` -> `AWX`로 이동*                                                                                                                                                                                | admin / Ingresslab123 |
| **RancherUI**     | Rancher UI가 `Client` VM에 도커로 배포되었으며 두 K8s 클러스터와 연결되어 있습니다. `<br>` *UI 접속: UDF의 `Systems`에서 `Client` -> `Rancher`로 이동*                                                                                                                                                                               | admin / Ingresslab123 |

## 2026 DevOps Academy Hands-on Lab

F5 DevOps Academy Lab을 위해 구성된 활용사례들은 아래에서 확인하실 수 있습니다:

- [**NGINX Basic**](1.nginx-basic/)
- [**NGINX Ingress Controller with BIG-IP CIS**](2.nginx-ingress-cis/)
- [**NGINX Gateway Fabric**](3.nginx-gateway-fabric/)
- [**F5 WAF for NGINX**](4.f5-waf-for-nginx/)

<br>
---

## 지원

지원이 필요하신 경우 GitHub 이슈를 등록해 주세요. 이 저장소의 코드는 커뮤니티에서 지원하며, F5 Networks의 공식 지원 대상이 아닙니다. 지원되는 프로젝트의 전체 목록은 [SUPPORT.md](SUPPORT.md)를 참고하세요.

## 커뮤니티 행동 강령

[F5 DevCentral 커뮤니티 행동 강령](code_of_conduct.md)을 참고하세요.

## 라이선스

[Apache License 2.0](LICENSE)

## 저작권

Copyright 2014-2020 F5 Networks Inc.

### F5 Networks 기여자 라이선스 계약

GitHub에서 F5 Networks, Inc.(F5)가 후원하는 프로젝트에 기여를 시작하기 전에, 기여자 라이선스 계약(CLA)에 서명해야 합니다.

개인 자격으로 서명하는 경우, 일부 고용 계약에는 다른 프로젝트에 대한 기여를 제한하는 조항이 있을 수 있으므로, CLA에 서명하기 전에 고용주(해당하는 경우)와 먼저 상의하시기 바랍니다.
CLA를 제출함으로써 귀하는 해당 계약에 명시된 라이선스를 부여할 법적 권한이 있음을 확인하는 것입니다.

귀하의 기여물을 포함하여 귀하가 창작한 지식재산에 대해 고용주가 권리를 보유하는 경우, 귀하는 고용주를 대신하여 기여할 권한을 부여받았거나, 고용주가 해당 기여물에 대한 권리를 포기했거나, 또는 고용주가 F5와 별도의 CLA를 체결했음을 확인하는 것입니다.

회사를 대표하여 서명하는 경우, 귀하는 해당 계약에 명시된 라이선스를 부여할 법적 권한이 있음을 확인하는 것입니다.

또한 귀하는 기여물을 제출하는 단체의 각 직원이 CLA에 따라 해당 단체를 대신하여 기여물을 제출할 권한이 있음을 확인하는 것입니다.
