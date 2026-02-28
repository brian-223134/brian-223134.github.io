---
title: "[데이터 중심 애플리케이션 설계] 03장"
excerpt: "03장: 저장소와 검색"

categories:
  - Designing-Data-Intensive-Applications
tags:
  - [서적, 데이터, 중심, 애플리케이션, 설계, 03장]

permalink: /book/designing-data-intensive-applications/03/

toc: true
toc_sticky: true

date: 2026-02-28
last_modified_at: 2026-02-28
---

![book-cover](/assets/images/posts_img/book/designing-data-intensive-applications/designing-data-intensive-applications-cover.png)

# [DDIA] 3장. 저장소와 검색 (Storage and Retrieval)

> 목표: 데이터베이스가 “저장/조회”를 실제로 어떻게 처리하는지 큰 그림 잡기  
> 키워드: 로그 구조 저장, 인덱스(B-Tree/LSM), 컴팩션, OLTP vs OLAP, 컬럼 지향 저장

---

## 0. 이 장이 던지는 질문

“DB는 그냥 SQL 날리면 알아서 해주지 않나?” 싶지만,
실제로는 **저장 엔진(storage engine)** 설계에 따라 다음이 크게 달라진다.

- 쓰기 성능(append vs overwrite)
- 읽기 성능(인덱스 구조, 캐시 히트율)
- 장애 복구(WAL, fsync, crash recovery)
- 저장 비용/증폭(write amplification)
- 확장 방향(트랜잭션 처리 vs 분석 처리)

> 결론: 데이터베이스의 성능/특성은 “자료구조 + 디스크 접근 패턴”에서 나온다.

---

## 1. DB는 내부적으로 무엇을 저장하나?

가장 단순한 형태를 상상해보면:

- 파일에 `key=value`를 줄줄이 추가(append)하는 로그 파일

이 방식만으로도 “최신 값을 찾기”는 가능하다.
하지만 문제는:

- 특정 key를 찾으려면 파일을 처음부터 끝까지 스캔해야 함 → 읽기 비효율
- 같은 key가 여러 번 등장 → 파일이 계속 커짐(중복)

그래서 등장하는 것이 **인덱스(index)** 와 **컴팩션(compaction)** 이다.

---

## 2. 인덱스(Index): “읽기를 빠르게, 대신 쓰기/공간 비용을 지불”

인덱스는 기본적으로 “추가 데이터 구조”다.
즉, 인덱스를 만들면 보통:

- 읽기 성능 ↑
- 쓰기 비용/디스크 사용량 ↑ (인덱스 갱신 + 공간)

> 인덱스는 공짜가 아니다. “주요 질의 패턴”에 맞춰 최소한으로 설계하는 게 중요.

---

## 3. 해시 인덱스(Hash Index): 단순하지만 범위 질의에 약함

### 3.1 개념

- `key -> 파일 오프셋` 매핑을 해시 맵으로 유지
- 쓰기는 로그 파일에 append, 해시 맵은 최신 위치로 업데이트

### 3.2 장점

- key 정확 조회(point lookup)가 빠름

### 3.3 한계

- **범위 질의(range query)** 에 약함  
  (예: `WHERE key BETWEEN 100 AND 200`, 정렬/프리픽스 검색 등)

### 3.4 컴팩션(Compaction)

로그에 중복 key가 쌓이므로, 주기적으로:

- 최신 값만 남기고 오래된 레코드를 버려 파일 크기를 줄인다.
- 세그먼트(segment) 단위로 병합(merge)하는 형태가 흔하다.

---

## 4. B-Tree 인덱스: 가장 전통적인 “페이지 기반” 구조

### 4.1 핵심 아이디어

- 디스크를 “페이지(page)” 단위로 관리
- 트리 구조로 key 범위를 나눠서 저장
- 조회는 루트→리프 탐색으로 O(log n)

### 4.2 강점

- point lookup, range query 모두 강함
- 정렬 순서 기반 처리(ORDER BY, BETWEEN)가 자연스럽다.
- 많은 RDB에서 핵심 인덱스 구조로 사용

### 4.3 쓰기 방식: overwrite 중심

- B-Tree는 보통 페이지를 **제자리 수정(overwrite)** 한다.
- 장애 대비를 위해 보통 **WAL(Write-Ahead Log)** 을 쓴다.
  - “변경 내용을 먼저 로그에 기록하고” 그 다음 실제 페이지 반영
  - 크래시 시 WAL로 복구

### 4.4 트레이드오프

- 랜덤 쓰기(페이지 수정)가 발생하기 쉬움 → 디스크 특성에 따라 비용
- 페이지 분할/병합 등 유지 비용

---

## 5. LSM-Tree 인덱스: “로그 구조(append) + 정렬 병합” 철학

LSM(Log-Structured Merge-Tree) 계열은
“디스크에 제자리 수정하지 말고, 계속 append로 쌓고, 나중에 정리하자”에 가깝다.

### 5.1 기본 구성

- **Memtable(메모리 정렬 구조)**: 쓰기 먼저 메모리에 정렬해서 쌓음
- **SSTable(디스크 정렬 파일)**: 일정 크기 되면 디스크에 정렬된 파일로 flush
- **Compaction/Merge**: 여러 SSTable을 병합하며 정리(중복 제거)

### 5.2 장점

- 쓰기가 매우 빠른 편(대부분 append + 순차 I/O)
- 대규모 데이터에서도 쓰기 처리량이 잘 나오는 경우가 많음

### 5.3 단점/주의점

- 컴팩션이 “백그라운드 비용”으로 존재
  - 설정이 나쁘면 컴팩션이 밀려서 성능 급락 가능
- 읽을 때 여러 레벨/파일을 확인해야 할 수 있음 → 읽기 증폭(read amplification)
- 삭제도 즉시 삭제가 아니라 **tombstone(삭제 표식)** 으로 처리 후 컴팩션 때 정리

### 5.4 Bloom Filter, Sparse Index

읽기 비용을 줄이기 위해 흔히:

- Bloom filter로 “이 파일에 key가 없는지” 빠르게 거름
- SSTable 내부에 희소 인덱스(sparse index)를 둬서 탐색

---

## 6. B-Tree vs LSM: 언제 뭐가 유리한가?

### 6.1 워크로드 관점(경향)

- **쓰기 중심(append, 대량 삽입)** → LSM 계열이 유리한 경우 많음
- **읽기 지연이 매우 민감 / 범위 질의 빈번** → B-Tree가 안정적인 경우 많음

### 6.2 “증폭(Amplification)” 관점

- write amplification: 한 번의 논리적 쓰기가 디스크에 여러 번 쓰이게 되는 현상
- read amplification: 한 번의 논리적 읽기가 여러 파일/레벨을 보게 되는 현상

LSM은 컴팩션 때문에 write amplification이 커질 수 있고,  
B-Tree는 페이지 수정/프래그먼트로 랜덤 I/O 비용이 커질 수 있다.

> 결론: “정답 구조”가 아니라, 내 시스템의 **주요 질의 패턴/지연 허용치/운영 난이도**로 선택한다.

---

## 7. OLTP vs OLAP: 트랜잭션 처리와 분석 처리는 요구가 다르다

### 7.1 OLTP (Online Transaction Processing)

- 사용자 요청 기반, 짧고 잦은 쿼리
- 작은 수의 레코드에 대한 읽기/쓰기
- 지연시간(latency) 중요

예: 로그인, 주문 생성, 게시글 작성

### 7.2 OLAP (Online Analytical Processing)

- 리포트/분석, 긴 쿼리(스캔/집계)
- 대량 데이터 읽기, 컬럼 일부만 필요할 때가 많음
- 처리량(throughput), 스캔 효율이 중요

예: “지난 1년 매출을 지역별로 집계”, “퍼널 분석”

### 7.3 데이터 웨어하우스(warehouse)가 따로 존재하는 이유

OLTP DB에 분석 쿼리를 때리면:

- 대규모 스캔으로 캐시/IO가 잠식되어
- 사용자 트랜잭션 지연이 튀기 쉬움

그래서 보통:

- OLTP(서비스) + OLAP(분석) 분리
- ETL/ELT로 분석용 저장소에 적재

---

## 8. 컬럼 지향 저장(Column-oriented storage): OLAP를 위한 최적화

OLAP는 “전체 행(row)”이 아니라 “일부 컬럼(column)”만 읽는 일이 많다.
그래서 컬럼 단위로 저장하면:

- 필요한 컬럼만 읽어서 IO 절약
- 같은 타입/분포의 값이 몰려서 압축 효율 ↑
- 집계/스캔 성능 ↑

대표적으로:

- Parquet/ORC 같은 컬럼 포맷
- 컬럼 지향 DB, 웨어하우스 시스템

> 요점: OLTP는 보통 row 지향 + 인덱스, OLAP는 column 지향 + 스캔/압축이 강하다.

---

## 9. 내가 적용해볼 체크리스트

- [ ] 내 시스템은 쓰기 중심인가, 읽기 중심인가?
- [ ] 주요 쿼리는 point lookup인가, range scan/정렬인가?
- [ ] 지연시간(p99) 민감도가 큰가, 처리량이 더 중요한가?
- [ ] 컴팩션/백그라운드 작업 같은 “운영 복잡도”를 감당 가능한가?
- [ ] 분석 쿼리가 서비스 DB에 영향을 주고 있진 않은가? (OLTP/OLAP 분리 필요?)

---

## 10. 5줄 요약

- 저장 엔진은 자료구조와 디스크 접근 패턴으로 성능이 결정된다.
- 인덱스는 읽기를 빠르게 하지만 쓰기/공간 비용을 추가로 낸다.
- B-Tree는 페이지 기반 overwrite + WAL로 범용성과 range query에 강하다.
- LSM은 append + 정렬 파일(SSTable) + 컴팩션으로 쓰기에 강하지만 운영/증폭 이슈가 있다.
- OLTP와 OLAP는 요구가 달라서 row 지향과 column 지향, 그리고 저장소 분리가 자주 필요하다.
