---
title: "Use OpenBao to Exchange Certificate && Store Prvate Key"
date: 2026-08-14 09:17:00 +0900
categories: [Network]
tags: [Network, Certificate]
---

## 1. 인증서 자동 교체를 위한 Secret Engine
OpenBao에서 인증서를 다루는 핵심 엔진은 크게 `PKI Enging`과 `KV Engine`이 있다.

### 1-1. PKI Engine
OpenBao 자체를 CA로 동작하게 만들거나, `Intermediate CA`로 등록하여, 인증서를 `On-Demand`로 즉시 생성 및 교체하는 엔진이다.

- 동적 인증서 발급 (pki/issue/:role)
    - 미리 정의한 Role에 맞춰 API 호출 한 번으로 `CSR 생성`부터 `서명, 인증서/개인키/CA 체인`을 한 번에 발급
- 짧은 유효기간 정책
    - 인증서를 짧게 발급하여 만료 전 `자동 교체`를 강제화
- 자동 폐기 및 CRL/OCSP 운영
    - 인증서 폐기 (pki/revoke) 발생 시 `CRL(Certificate Revocation List)`을 자동으로 업데이트하며, OCSP Responder 기능을 지원하여 실시간 유효성 검증 처리
- ACME Protocol
    - OpenBao PKI 내부에 ACME 엔드포인트 노출
    - Let's Encrypt처럼 **certbot**이나 **vert-manager**가 표준 ACME 프로토콜로 OpenBao에서 새 인증서를 자동으로 받아가고 갱신 가능
- EST (Enrollment over Secure Transport)
    - EST Protocol을 지원하여 네트워크 장비, IoT 등이 인증서를 자동으로 갱신받을 수 있도록 지원

### 1-2. KV Secret Enging
외부 CA에서 받아온 레거시/기존 인증서와 개인키를 안전하게 암호화 보관하고 버전 관리하는 엔진이다.

- KV v2 버전 관리 및 Rollback
    - 인증서를 갱신하여 `openbao kv put`으로 저장할 때마다 새로운 버전이 쌓이며, 문제 발생 시 이전 버전 인증서/개인키로 즉시 원복(Undelete/Rollback) 가능
- Metadata 및 만료 알림 연동
    - 저장된 인증서의 만료일, 주 도메인, 담당자 정보 등을 Metadata에 함께 기록하여 관리

### 1-3. Transit Secret Enging
- 인증서 발급에 사용되는 외부 CA의 Private Key나 중요 CSR을 OpenBao 밖으로 유출하지 않고, OpenBao 내부 메모리 상에서만 암호화 서명 연산을 수행할 때 활용한다.



## 2. 서버 및 클라이언트단 자동 교체 도구
인증서가 OpenBao에 생성/저장되었을 때, 이를 실제 타겟 서버(Nginx, Apache, K8s, Envoy 등)에 배포하고 무중단 재시작까지 연결해 주는 기능

### 2-1. OpenBao Agent (Vault Agent) & Template Engine
타겟 서버에 데몬 형태로 상주하여 인증서를 자동으로 가져와 파일로 써주고, 프로세스를 재시작하는 자동화 솔루션

- Auto-Auth
    - AppRole, AWS IAM, K8s SA 등을 통해 서버가 켜지면 자동으로 OpenBao 토큰을 발급받고 주기적으로 갱신
- Consul Template
    - `templates` 블록을 활용해 PKI나 KV 엔진에서 인증서/개인키/CA 체인을 읽어 로컬 파일 시스템에 렌더링
- Command Trigger
    - 인증서 파일이 새로 교체되면 지정한 쉘 명령어를 자동으로 실행하여 무중단 교체
- file_perms, Ownership 제어
    - 개인키 파일은 `0600`, 인증서 파일은 `0644`등 파일 권한 및 소유자 자동 맞춤

### 2-2. k8s Cert-Manager 연동
- Cert-Manager Vault Issuer
    - Ingress나 Custom Resource (CRD)의 인증서 만료일을 감시하다가, 만료 N일 전 OpenBao PKI API를 호출해 새 인증서 발급
- K8s Secret 자동 갱신
    - 발급받은 인증서와 개인킬르 `kubernetes.io/tls` 타입의 Secret으로 업데이트하고, 이를 참조하는 Pod나 Ingress Controller가 자동으로 최신 인증서를 로딩

### 2-3. OpenBao CSI Provider (Screts Store CSI Driver)
- k8s Pod의 볼륨 형태로 OpenBao의 인증서/개인키를 직접 마운트(In-Memory tmpfs)
- Auto Rotation
    - CSI Driver 옵션으로, OpenBao의 값이 변경될 때 Pod 내부 마운트 파일도 자동 업데이트


## 3. 인증서 교체 모니터링 및 이벤트 알림
인증서가 정상적으로 교체되었는지, 만료 예정인 인증서가 없는지 모니터링하는 기능이다.

### 3-1. Event Notification / Event Stream Engine
- OpenBao 내부의 KV 변경이나 PKI 인증서 발급/폐기 이벤트를 이벤트 스트림으로 발행
- Webhook, Kafka, RabbitMQ 등으로 이벤트를 전송

### 3.2 Prometheus Metrics 연동(/v1/sys/metrics)
- OpenBao의 프로메테우스 메트릭을 통해 PKI 엔진 관련 지표 추출
    - openbao.pki.cert.expiration: 발급된 인증서들의 만료 시간 지표
    - openbao.secret.kv.get: 인증서 조회 횟수 및 갱신 주기 모니터링

### 3.3 Audit Logging
- 모든 개인키/인증서 접근 및 재발급 요청 기록
    - 언제, 어떤 IP/토큰을 가진 서버가 인증서 및 개인키를 읽어갔는지 저장한 JSON 형태 로그


## 4. 인증서 교체 관련 권한 제어
보안 기밀성 유지 및 사고 방지를 위한 세부 접근 제어 기능

### 4-1. Path-based ACL Policy
- 타겟 웹서버 A는 `secret/data/certs/app-a`의 개인키만 읽을 수 있고, B는 `secret/data/certs/app-b`만 읽을 수 있도록 세분화된 경로 제어

### 4-2. CIDR Block / IP 제한
- 특정 내부 IP 대역에서 들어오는 요청에 대해서만 인증서/개인키 읽기 허용

### 4-3. TTL & Response Wrapping
- Response Wrapping(-wrap-ttl)
    - 인증서/개인키를 바로 리턴하지 않고 원타임 토큰으로 감싸서 전달
    - 전달받은 타겟 서버만 한 번 복호화하여 실제 개인키를 획득 가능
        - MITM 방지

## 5. 아키텍처별 인증서 자동 교체 구현 시나리오 요약


| 구성 요소 | 사용되는 OpenBao 주요 기능 | 동작 프로세스 |
| --- | --- | --- |
| K8s 환경 (동적 발급) | PKI Engine + Cert-Manager + ACME / Vault Issuer | Cert-Manager가 만료 감지 ➔ OpenBao PKI 호출 ➔ K8s TLS Secret 자동 갱신 ➔ Ingress 적용 |
| VM / Bare-metal (KV 기반) | KV v2 Engine + OpenBao Agent + Template | CI/CD가 OpenBao KV에 개인키/crt 저장 ➔ Agent가 변화 감지 ➔ 로컬 파일 덮어쓰기 ➔ systemctl reload |
| VM / Bare-metal (PKI 기반) | PKI Engine + OpenBao Agent + Auto-Auth | Agent가 TTL(예: 30일)에 맞춰 OpenBao PKI로 자동 재발급 요청 ➔ 로컬 파일 덮어쓰기 ➔ 서비스 Reload |
| IoT / 네트워크 장비 | PKI Engine + EST / ACME Protocol | 장비 내 클라이언트가 EST/ACME 프로토콜로 OpenBao에 주기적 접속 ➔ 인증서 갱신 받아 적용 |
