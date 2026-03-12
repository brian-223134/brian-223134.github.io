---
title: "[인턴] AWS/EC2 CloudWatch 메트릭 전체 정리"
excerpt: "EC2 인스턴스에 기본으로 제공되는 AWS/EC2 네임스페이스 CloudWatch 메트릭 18개를 카테고리별로 정리합니다."

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, 인프라, CloudWatch, EC2, 메트릭, 모니터링]

permalink: /intern/company/blitz-dynamics/04/

toc: true
toc_sticky: true

date: 2026-03-10
last_modified_at: 2026-03-12
---

# 들어가며

AWS EC2 인스턴스를 운영하면 CloudWatch가 `AWS/EC2` 네임스페이스 아래 다양한 메트릭을 자동으로 수집한다. 별도의 에이전트 설치 없이 기본으로 제공되는 메트릭들이다.

이 포스트에서는 실제 EC2 인스턴스에 연결되어 있는 메트릭 18개를 카테고리별로 정리한다.

```
[대상 메트릭 목록 - Namespace: AWS/EC2]
CPU:       CPUUtilization
Network:   NetworkIn, NetworkOut, NetworkPacketsIn, NetworkPacketsOut
EBS I/O:   EBSReadBytes, EBSWriteBytes, EBSReadOps, EBSWriteOps
EBS 크레딧: EBSIOBalance%, EBSByteBalance%
EBS 초과:  InstanceEBSIOPSExceededCheck, InstanceEBSThroughputExceededCheck
상태 체크:  StatusCheckFailed, StatusCheckFailed_Instance,
           StatusCheckFailed_System, StatusCheckFailed_AttachedEBS
메타데이터: MetadataNoToken
```

---

# 1. CPU 메트릭

## CPUUtilization

EC2 인스턴스가 현재 사용 중인 CPU의 비율이다.

| 항목 | 값 |
|---|---|
| **단위** | Percent (%) |
| **권장 집계** | Average |
| **수집 주기** | 기본 5분 / 세부 모니터링 활성화 시 1분 |

**동작 원리**: EC2 하이퍼바이저가 인스턴스에 할당된 vCPU 사용량을 측정한다. 100%는 해당 인스턴스에 할당된 모든 vCPU를 완전히 사용 중임을 의미한다.

**모니터링 포인트**:
- 지속적으로 80% 이상이면 인스턴스 스펙 업그레이드를 검토한다.
- T 계열(t3, t3a 등) 버스터블 인스턴스는 CPU 크레딧 소진 시 기준 성능으로 제한되므로 `CPUCreditBalance`와 함께 확인한다.

---

# 2. 네트워크 메트릭

## NetworkIn

인스턴스의 **모든 네트워크 인터페이스로 수신된 바이트 수**다.

| 항목 | 값 |
|---|---|
| **단위** | Bytes |
| **권장 집계** | Sum |
| **수집 주기** | 기본 5분 |

인터넷, VPC 내부 통신, AWS 서비스 호출 등 모든 네트워크 수신 트래픽이 포함된다. 갑작스러운 급증은 트래픽 폭증, DDoS, 또는 대규모 데이터 수신 작업의 신호일 수 있다.

---

## NetworkOut

인스턴스의 **모든 네트워크 인터페이스에서 송신된 바이트 수**다.

| 항목 | 값 |
|---|---|
| **단위** | Bytes |
| **권장 집계** | Sum |
| **수집 주기** | 기본 5분 |

응답 데이터 전송, 외부 API 호출, 데이터 복제 등의 송신 트래픽을 포함한다. `NetworkIn`과 함께 추적하면 트래픽의 방향성을 파악할 수 있다.

---

## NetworkPacketsIn

인스턴스의 **모든 네트워크 인터페이스로 수신된 패킷 수**다.

| 항목 | 값 |
|---|---|
| **단위** | Count |
| **권장 집계** | Sum |
| **수집 주기** | 기본 5분 |

`NetworkIn`(바이트)과 쌍으로 확인하면 트래픽의 특성을 구분할 수 있다.
- 패킷 수는 많지만 바이트가 적다 → 소규모 패킷 다수 (예: SYN Flood 공격 의심)
- 패킷 수는 적지만 바이트가 많다 → 대용량 파일 전송

---

## NetworkPacketsOut

인스턴스의 **모든 네트워크 인터페이스에서 송신된 패킷 수**다.

| 항목 | 값 |
|---|---|
| **단위** | Count |
| **권장 집계** | Sum |
| **수집 주기** | 기본 5분 |

`NetworkOut`과 함께 확인한다. 패킷 수 대비 바이트 비율이 정상 범위를 이탈하면 네트워크 이상 징후를 의심할 수 있다.

---

# 3. EBS 메트릭

EBS(Elastic Block Store)는 EC2 인스턴스에 연결되는 네트워크 기반 블록 스토리지다. EBS 관련 메트릭은 세 가지 범주로 나뉜다.

- **I/O 처리량**: 실제로 읽고 쓴 바이트 및 작업 수
- **버스트 크레딧**: T 계열 인스턴스의 EBS 버스트 가능성
- **초과 체크**: Nitro 기반 인스턴스의 IOPS/처리량 한계 초과 여부

## 3-1. EBS I/O 처리량 메트릭

### EBSReadBytes

지정된 기간 동안 **연결된 모든 EBS 볼륨에서 읽은 바이트 수**다.

| 항목 | 값 |
|---|---|
| **단위** | Bytes |
| **권장 집계** | Sum |
| **수집 주기** | 기본 5분 |

데이터베이스 쿼리, 로그 읽기, 파일 서빙 등 읽기 집약적인 작업의 부하를 파악하는 데 사용한다.

---

### EBSWriteBytes

지정된 기간 동안 **연결된 모든 EBS 볼륨에 쓴 바이트 수**다.

| 항목 | 값 |
|---|---|
| **단위** | Bytes |
| **권장 집계** | Sum |
| **수집 주기** | 기본 5분 |

로그 기록, DB 쓰기, 파일 업로드 처리 등 쓰기 집약적인 작업의 부하를 파악한다. 지속적으로 높은 값은 EBS 볼륨 타입 변경(예: gp2 → gp3 또는 io1)을 검토할 신호가 된다.

---

### EBSReadOps

지정된 기간 동안 **연결된 모든 EBS 볼륨에서 완료된 읽기 작업(I/O 요청) 수**다.

| 항목 | 값 |
|---|---|
| **단위** | Count |
| **권장 집계** | Sum |
| **수집 주기** | 기본 5분 |

`EBSReadBytes ÷ EBSReadOps`를 계산하면 **평균 읽기 I/O 크기**를 구할 수 있다. I/O 크기가 작고 횟수가 많다면 랜덤 I/O, 크기가 크고 횟수가 적다면 순차 I/O 패턴임을 알 수 있다.

---

### EBSWriteOps

지정된 기간 동안 **연결된 모든 EBS 볼륨에 완료된 쓰기 작업 수**다.

| 항목 | 값 |
|---|---|
| **단위** | Count |
| **권장 집계** | Sum |
| **수집 주기** | 기본 5분 |

`EBSWriteBytes ÷ EBSWriteOps`로 평균 쓰기 I/O 크기를 구할 수 있다. EBS 볼륨의 프로비저닝된 IOPS 한계와 비교하여 성능 병목 여부를 판단한다.

---

## 3-2. EBS 버스트 크레딧 메트릭 (T 계열 인스턴스)

> **적용 대상**: `t3`, `t3a` 등 버스터블 인스턴스 계열에만 해당한다. 다른 인스턴스 타입에서는 이 메트릭이 보고되지 않는다.

T 계열 인스턴스의 EBS는 **버스트 모델**로 동작한다. 평소에는 크레딧을 적립하고, 일시적으로 높은 I/O가 필요할 때 크레딧을 소모하여 기준 성능 이상을 낼 수 있다. 크레딧이 소진되면 기준 성능으로 제한된다.

### EBSIOBalance%

**남아있는 I/O 크레딧(IOPS 버스트 크레딧)의 비율**이다.

| 항목 | 값 |
|---|---|
| **단위** | Percent (%) |
| **범위** | 0 ~ 100 |
| **권장 집계** | Average |
| **수집 주기** | 기본 5분 |

값이 0에 가까워지면 추가적인 버스트 I/O가 불가능해지며, 다시 크레딧이 쌓일 때까지 기준 IOPS로 제한된다. 지속적으로 낮은 경우 인스턴스 타입 변경 또는 Provisioned IOPS(io1/io2) EBS 볼륨 도입을 검토한다.

---

### EBSByteBalance%

**남아있는 처리량 크레딧(Throughput 버스트 크레딧)의 비율**이다.

| 항목 | 값 |
|---|---|
| **단위** | Percent (%) |
| **범위** | 0 ~ 100 |
| **권장 집계** | Average |
| **수집 주기** | 기본 5분 |

`EBSIOBalance%`가 IOPS(I/O 작업 횟수) 기준의 크레딧이라면, `EBSByteBalance%`는 처리량(Byte) 기준의 크레딧이다. 두 값을 함께 확인해야 버스트 성능 한계를 정확히 파악할 수 있다.

---

## 3-3. EBS 초과 체크 메트릭 (Nitro 기반 인스턴스)

> **적용 대상**: Nitro 하이퍼바이저 기반 인스턴스에서 EBS 최적화가 활성화된 경우에 제공된다.

이 메트릭들은 수치가 아닌 **이진(binary) 상태값**을 반환한다.
- `0` = 초과 없음 (정상)
- `1` = 초과 발생

### InstanceEBSIOPSExceededCheck

인스턴스 레벨에서의 **EBS IOPS 처리량 한계 초과 여부**를 나타낸다.

| 항목 | 값 |
|---|---|
| **단위** | Count (0 또는 1) |
| **권장 집계** | Maximum |
| **수집 주기** | 기본 5분 |

EC2 인스턴스 타입마다 EBS로 처리할 수 있는 최대 IOPS 한도가 존재한다. 연결된 EBS 볼륨들의 합산 IOPS가 이 한도를 넘으면 `1`이 보고된다. 이 경우 더 높은 EBS 대역폭을 지원하는 인스턴스 타입으로 변경을 고려한다.

---

### InstanceEBSThroughputExceededCheck

인스턴스 레벨에서의 **EBS 처리량(Throughput) 한계 초과 여부**를 나타낸다.

| 항목 | 값 |
|---|---|
| **단위** | Count (0 또는 1) |
| **권장 집계** | Maximum |
| **수집 주기** | 기본 5분 |

IOPS가 아닌 **바이트 처리량** 기준의 초과 여부를 확인한다. `InstanceEBSIOPSExceededCheck`는 정상이지만 이 값이 `1`이라면 I/O 횟수는 적어도 각 I/O의 크기가 커서 처리량 한계에 도달한 경우다.

---

# 4. 상태 체크 메트릭

상태 체크(Status Check) 메트릭은 EC2 인스턴스와 그 기반 인프라의 **건강 상태**를 나타낸다. CloudWatch가 1분 간격으로 자동으로 수행하며, 문제가 없으면 `0`, 실패하면 `1`을 반환한다.

상태 체크는 계층 구조로 이루어진다.

```
StatusCheckFailed (통합)
  ├── StatusCheckFailed_System    (AWS 인프라 레벨)
  ├── StatusCheckFailed_Instance  (OS/소프트웨어 레벨)
  └── StatusCheckFailed_AttachedEBS (연결된 EBS 레벨)
```

## StatusCheckFailed

**시스템 체크와 인스턴스 체크를 통합한 결과**다.

| 항목 | 값 |
|---|---|
| **단위** | Count (0 또는 1) |
| **권장 집계** | Maximum |
| **수집 주기** | 1분 |

`StatusCheckFailed_System` 또는 `StatusCheckFailed_Instance` 중 하나라도 실패(`1`)하면 이 값도 `1`이 된다. 통합 알람용으로 가장 먼저 설정하는 메트릭이다.

---

## StatusCheckFailed_Instance

**인스턴스 레벨 상태 체크 실패 여부**다. 이 체크는 **EC2 인스턴스 내부의 소프트웨어·운영체제 문제**를 감지한다.

| 항목 | 값 |
|---|---|
| **단위** | Count (0 또는 1) |
| **권장 집계** | Maximum |
| **수집 주기** | 1분 |

다음 상황에서 `1`이 보고된다.

- OS 커널 패닉 또는 멈춤
- 파일 시스템 오류로 인한 부팅 실패
- 잘못된 네트워크 인터페이스 설정
- 메모리 부족으로 인한 응답 불가

인스턴스 체크 실패는 AWS가 자동으로 해결할 수 없는 **사용자 관할 영역의 문제**다. 직접 인스턴스에 접속하여 OS 수준에서 조치가 필요하다.

---

## StatusCheckFailed_System

**시스템 레벨 상태 체크 실패 여부**다. 이 체크는 **EC2 인스턴스가 올라가 있는 AWS의 물리 인프라 문제**를 감지한다.

| 항목 | 값 |
|---|---|
| **단위** | Count (0 또는 1) |
| **권장 집계** | Maximum |
| **수집 주기** | 1분 |

다음 상황에서 `1`이 보고된다.

- 물리 호스트 서버 하드웨어 장애
- AWS 네트워크 연결 문제
- 물리 호스트의 전원 공급 이상
- AWS 시스템이 인스턴스 연결성을 검증하지 못 할 경우

시스템 체크 실패는 **AWS가 관할하는 인프라 문제**이므로 사용자가 직접 고칠 수 없다. 인스턴스를 중지 후 재시작하면 다른 물리 호스트로 이전되어 해결되는 경우가 많다.

> `StatusCheckFailed_Instance`는 재부팅으로, `StatusCheckFailed_System`은 중지 후 재시작(Stop → Start)으로 대응하는 것이 일반적이다.

---

## StatusCheckFailed_AttachedEBS

**연결된 EBS 볼륨의 상태 체크 실패 여부**다.

| 항목 | 값 |
|---|---|
| **단위** | Count (0 또는 1) |
| **권장 집계** | Maximum |
| **수집 주기** | 1분 |

EC2 인스턴스는 정상이지만 **EBS 볼륨이 정해진 시간 내에 I/O 작업을 완료하지 못 할 경우** `1`이 보고된다.

원인은 주로 두 가지다.
- EBS 볼륨 자체의 하드웨어 또는 네트워크 문제
- I/O 요청이 지나치게 몰려 처리 지연 발생

이 메트릭이 `1`이 되면 해당 볼륨을 사용하는 애플리케이션에서 `I/O error`가 발생할 수 있다.

---

# 5. 메타데이터 메트릭

## MetadataNoToken

**세션 토큰 없이 인스턴스 메타데이터 서비스(IMDS)에 접근한 횟수**다.

| 항목 | 값 |
|---|---|
| **단위** | Count |
| **권장 집계** | Sum |
| **수집 주기** | 1분 |

**배경**: EC2 인스턴스는 자신의 인스턴스 ID, IAM 역할의 임시 자격증명 등을 `http://169.254.169.254/latest/meta-data/` 엔드포인트로 조회할 수 있다. 이 서비스를 IMDS(Instance Metadata Service)라고 한다.

IMDS에는 두 가지 버전이 있다.

| 버전 | 특징 |
|---|---|
| **IMDSv1** | 단순 GET 요청만으로 메타데이터 접근 가능. 세션 토큰 불필요. SSRF 공격에 취약 |
| **IMDSv2** | 세션 토큰을 먼저 발급받아야 메타데이터 접근 가능. 더 안전 |

`MetadataNoToken`은 **IMDSv1 방식으로 메타데이터를 조회한 횟수**다. 값이 `0`보다 크다면 인스턴스 내부의 어딘가(애플리케이션 코드, SDK, 에이전트 등)가 구형 IMDSv1을 사용 중이라는 의미다.

**모니터링 포인트**: 보안 강화를 위해 점진적으로 IMDSv2만 허용하는 방향(`HttpTokens: required`)으로 전환할 때, 이 메트릭으로 아직 IMDSv1을 사용하는 코드를 탐지할 수 있다.

---

# 6. 전체 메트릭 요약 테이블

| 메트릭 | 카테고리 | 단위 | 권장 집계 | 주요 용도 |
|---|---|---|---|---|
| `CPUUtilization` | CPU | % | Average | 서버 부하 파악 |
| `NetworkIn` | 네트워크 | Bytes | Sum | 수신 트래픽 량 |
| `NetworkOut` | 네트워크 | Bytes | Sum | 송신 트래픽 량 |
| `NetworkPacketsIn` | 네트워크 | Count | Sum | 수신 패킷 수, 이상 트래픽 감지 |
| `NetworkPacketsOut` | 네트워크 | Count | Sum | 송신 패킷 수, 이상 트래픽 감지 |
| `EBSReadBytes` | EBS I/O | Bytes | Sum | 스토리지 읽기 처리량 |
| `EBSWriteBytes` | EBS I/O | Bytes | Sum | 스토리지 쓰기 처리량 |
| `EBSReadOps` | EBS I/O | Count | Sum | 스토리지 읽기 IOPS |
| `EBSWriteOps` | EBS I/O | Count | Sum | 스토리지 쓰기 IOPS |
| `EBSIOBalance%` | EBS 버스트 | % | Average | T 계열 IOPS 크레딧 잔량 |
| `EBSByteBalance%` | EBS 버스트 | % | Average | T 계열 처리량 크레딧 잔량 |
| `InstanceEBSIOPSExceededCheck` | EBS 초과 | 0/1 | Maximum | 인스턴스 IOPS 한계 초과 여부 |
| `InstanceEBSThroughputExceededCheck` | EBS 초과 | 0/1 | Maximum | 인스턴스 처리량 한계 초과 여부 |
| `StatusCheckFailed` | 상태 체크 | 0/1 | Maximum | 통합 상태 이상 감지 |
| `StatusCheckFailed_Instance` | 상태 체크 | 0/1 | Maximum | OS/소프트웨어 문제 감지 |
| `StatusCheckFailed_System` | 상태 체크 | 0/1 | Maximum | AWS 인프라 문제 감지 |
| `StatusCheckFailed_AttachedEBS` | 상태 체크 | 0/1 | Maximum | EBS I/O 지연 감지 |
| `MetadataNoToken` | 메타데이터 | Count | Sum | IMDSv1 사용 탐지 |

---

# 마무리

18개 메트릭 중 인스턴스 타입에 무관하게 항상 수집되는 것은 다음이다.

- **CPU**: `CPUUtilization`
- **네트워크**: `NetworkIn`, `NetworkOut`, `NetworkPacketsIn`, `NetworkPacketsOut`
- **EBS I/O**: `EBSReadBytes`, `EBSWriteBytes`, `EBSReadOps`, `EBSWriteOps`
- **상태 체크**: `StatusCheckFailed`, `StatusCheckFailed_Instance`, `StatusCheckFailed_System`, `StatusCheckFailed_AttachedEBS`
- **메타데이터**: `MetadataNoToken`

나머지는 인스턴스 조건에 따라 제공된다.
- `EBSIOBalance%`, `EBSByteBalance%` → T 계열 버스터블 인스턴스만
- `InstanceEBSIOPSExceededCheck`, `InstanceEBSThroughputExceededCheck` → Nitro 기반 + EBS 최적화 활성화 인스턴스만

이 메트릭들을 기반으로 CloudWatch Alarm과 대시보드를 구성하는 내용은 [이전 포스트](/intern/company/blitz-dynamics/pulumi-cloudwatch-slack/)에서 확인할 수 있다.
