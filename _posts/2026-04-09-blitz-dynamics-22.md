---
title: "[인턴] 카나리 배포 시 DDL Migration 실행 시점 전략"
excerpt: "카나리 배포 중 Blue/Green 서버가 공존할 때, 컬럼 추가·삭제·변경·제약 조건별로 Migration을 언제 실행해야 하는지 정리한 기록"

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, 인프라, 카나리배포, BlueGreen, DB마이그레이션, DDL, ECS]

permalink: /intern/company/blitz-dynamics/22/

toc: true
toc_sticky: true

date: 2026-04-09
last_modified_at: 2026-04-09
---

## 들어가며

[이전 글](/intern/company/blitz-dynamics/21/)에서는 ECS Fargate로 전환하면서 설계한 카나리 배포 전략 전반을 정리했습니다.  
이번 글에서는 그 중에서도 **DDL Migration 실행 시점**을 별도로 깊게 파고든 내용을 기록합니다.

카나리 배포를 구체적으로 설계하다 보니 한 가지 문제가 눈에 들어왔습니다.  
코드 배포는 Blue/Green 전환으로 제어할 수 있어도, **DB 스키마 변경은 그 순간 전체 DB에 즉시 반영**됩니다.  
Blue(구버전)와 Green(신버전)이 동시에 트래픽을 처리하는 구간에서, Migration 실행 시점을 잘못 잡으면 한쪽 서버가 의도치 않게 오류를 냅니다.

이 문제를 정리하기 위해 DDL 유형별로 호환성을 분석했습니다.

---

## 핵심 원칙: 호환성 방향

```
Blue (구버전)  ←──── 트래픽 전환 중 ────→  Green (신버전)
     ↑                                          ↑
  구 스키마 기대                            신 스키마 기대
```

Migration 실행 시점은 **"어느 서버가 해당 스키마에 의존하는가"** 로 결정됩니다.  
Blue가 아직 트래픽을 받는 동안 Blue가 모르는 스키마 변경을 가하면 오류가 납니다.  
반대로 Green이 필요한 스키마가 아직 없으면 Green이 기동 자체를 못합니다.

---

## Case 1: 스키마 추가 (Additive)

컬럼·테이블·인덱스가 **새로 생기는** 경우입니다.

### 실행 시점

```
[Migrate 실행] → [Green 배포] → [트래픽 전환] → [Blue 제거]
```

Green이 뜨기 전에 반드시 선행해야 합니다.  
Green은 새 컬럼을 기대하는 코드를 포함하고 있으므로, 스키마가 없으면 기동하지 못합니다.

### Blue/Green 호환성

| 상황 | Blue 반응 | Green 반응 |
|------|-----------|------------|
| Migration 전 | 정상 (컬럼 모름) | 기동 불가 (컬럼 없음) |
| Migration 후 | ✅ 정상 (컬럼 무시) | ✅ 정상 (컬럼 사용) |

Blue는 새 컬럼의 존재를 모르지만, 그것을 읽거나 쓰지 않으므로 문제가 없습니다.

### 주의사항

- **`NOT NULL` 제약 금지**: Blue가 해당 컬럼에 값을 넣지 않으므로 insert 실패
- **DEFAULT 값 설정 권장**: Blue의 insert 시 자동으로 기본값이 채워짐
- `SELECT *` 패턴이더라도 신규 컬럼이 ORM에 매핑 안 되면 무시되므로 대부분 안전

---

## Case 2: 스키마 삭제 (Destructive)

컬럼·테이블이 **제거되는** 경우입니다.

### 실행 시점

```
[Green 배포] → [트래픽 100% 전환] → [Migrate 실행] → [Blue 제거]
```

Blue 트래픽이 완전히 빠진 후에 실행해야 합니다.

### Blue/Green 호환성

| 상황 | Blue 반응 | Green 반응 |
|------|-----------|------------|
| Migration 전 | ✅ 정상 (컬럼 사용) | ✅ 정상 (컬럼 무시) |
| Migration 후 | ❌ 오류 (컬럼 없음) | ✅ 정상 |

### 주의사항

- **Green은 삭제될 컬럼을 코드에서 먼저 제거한 채 배포**되어야 합니다
- Blue가 여전히 해당 컬럼을 읽거나 쓰는 코드가 있다면, Migration 후 즉시 오류 발생
- 트래픽 전환이 100%가 된 이후에도 Blue로 롤백 가능성이 있다면 Migration을 더 늦춰야 합니다

---

## Case 3: 스키마 변경 (Rename / Type Change)

**가장 위험한 케이스**입니다.  
Blue/Green이 동시에 만족하는 스키마가 존재하지 않기 때문입니다.

### 문제

```
Blue는 user_name 컬럼을 기대
Green은 username 컬럼을 기대
→ 동시에 두 서버를 만족하는 스키마가 없음
```

### 해결: Expand-Contract 패턴 (3단계 배포)

이 문제는 단일 배포로는 해결이 불가능합니다.  
배포를 세 단계로 쪼개야 합니다.

**1단계 — Expand: 컬럼 추가 Migration + 양쪽 동기화 코드 배포**

```
[username 컬럼 추가 Migration 실행]
→ [Green 1차 배포: user_name & username 양쪽 모두 동기화하는 코드]
→ [트래픽 전환 시작]
```

- Blue: `user_name` 사용 ✅
- Green (1차): `user_name`에 쓰면 `username`에도 동기화, 반대도 동일 ✅

**2단계 — 정리: Blue 제거 + 데이터 동기화 완료**

```
[트래픽 100% Green으로 전환] → [Blue 제거]
→ 기존 user_name 데이터를 username으로 마이그레이션
```

**3단계 — Contract: 구 컬럼 삭제 Migration + 참조 코드 제거 배포**

```
[Green 2차 배포: user_name 참조 코드 완전 제거]
→ [user_name 컬럼 삭제 Migration 실행]
```

---

## Case 4: 제약 조건 변경 (Constraint)

### NOT NULL 추가

| 실행 시점 | 문제 |
|-----------|------|
| Green 배포 전 | Blue가 NULL insert → DB 오류 |
| Green 배포 후, 전환 전 | Blue가 여전히 NULL insert → DB 오류 |

**반드시 트래픽 100% 전환 후 + 기존 데이터 NULL 없음 확인 후 실행**해야 합니다.  
Case 2와 동일한 시점입니다.

### Unique 제약 추가

- 기존 데이터의 중복 여부를 반드시 사전 검증
- Blue가 중복 insert하는 코드가 있다면 전환 후 실행 (Case 2와 동일)

### Foreign Key 추가

- 참조 대상 테이블·컬럼이 먼저 존재해야 하므로 Green 배포 전 선행 실행 (Case 1과 동일)
- 단, 기존 데이터 정합성 검증 필수

---

## 전략 요약표

| DDL 유형 | Migration 시점 | 핵심 조건 |
|----------|---------------|-----------|
| 컬럼/테이블 **추가** | Green 배포 **전** | Nullable 또는 DEFAULT 필수 |
| 컬럼/테이블 **삭제** | 트래픽 전환 **100% 후** | Green 코드에서 먼저 참조 제거 |
| 컬럼 **Rename/Type 변경** | 3단계 분리 배포 | Expand-Contract 패턴 필수 |
| NOT NULL / Unique **추가** | 트래픽 전환 **100% 후** | 기존 데이터 정합성 선검증 |
| FK **추가** | Green 배포 **전** | 참조 대상 선행 존재 확인 |

---

## 실무 팁

- Migration을 코드 배포와 분리하여 **별도 파이프라인으로 관리**하면 실행 시점 제어가 명확해진다.
- **Flyway / Liquibase의 `baseline` 기능**을 활용해 Migration 이력과 실제 스키마 상태를 추적하자.
- 롤백 시나리오를 항상 가정하고, **down migration script도 함께 작성**하자. 단, Destructive DDL의 down script는 데이터 복구가 불가능할 수 있으므로 주의가 필요하다.

---

## 마치며

카나리 배포를 설계하면서 코드 배포보다 DB 스키마 변경이 훨씬 까다롭다는 것을 체감했습니다.  
코드는 버전별로 격리해서 배포할 수 있지만, DB는 공유 자원이라 변경이 즉시 모든 서버에 영향을 미칩니다.

DDL 유형별로 실행 시점을 구분하는 것이 핵심이며, 특히 Rename이나 Type Change처럼 구/신 버전을 동시에 만족하는 스키마가 없는 경우에는 Expand-Contract 패턴으로 배포 단계를 쪼개는 것 외에 다른 안전한 방법이 없습니다.
