---
title: "[회고] 고려대 CCSLAB 학부 인턴십: SBOM 분석 및 통합 프로토타입 개발을 마치며"
excerpt: "고려대학교 컴퓨터 보안 연구실 학부 인턴 회고 내용 입니다."

categories:
  - Laboratory
tags:
  - [고려대학교, ccs, 보안, 연구실, 학부인턴]

permalink: /intern/lab/ccs-review/

toc: true
toc_sticky: true

date: 2026-02-18
last_modified_at: 2026-02-26
---

![ccs](/assets/images/posts_img/intern/ccs/ccs.svg)

지난 8주간 고려대학교 **컴퓨터 및 통신 보안 연구실(CCSLAB)** 에서 학부 인턴으로 활동하며 연구한 내용을 정리해 보려 합니다.

## 1. 연구 주제: Toward a Unified SBOM
소프트웨어 공급망 보안이 중요해지면서 **SBOM (Software Bill of Materials)** 은 이제 필수적인 보안 요소가 되었습니다. 하지만 시중의 여러 도구(Syft, Trivy, cdxgen 등)를 직접 분석해 보니 결과가 제각각이라는 문제점이 있었습니다. 저는 이 파편화된 데이터를 어떻게 신뢰할 수 있는 하나의 형태로 통합할 수 있을지를 고민했습니다.

---

## 2. 주요 활동 및 성과

### CISA 11 Minimum Elements 분석
기존 NTIA 의 7개 요소를 넘어 CISA 에서 새롭게 제안한 **11개 최소 요소** 를 분석했습니다. 특히 다음 4가지 지표에 집중하여 도구들의 신뢰성을 평가했습니다.
* **Tool Name**: 데이터 생성 방법론의 신뢰성 판단 근거 제공
* **Generation Context**: 라이프사이클 단계별 데이터 가시성 확인
* **Component Hash**: 무결성 검증 및 추적의 핵심
* **License**: 법적 리스크 평가 및 집행에 필수적

### 오픈소스 기여 (Anchore Syft)
실험 도중 Syft가 특정 의존성을 누락하는 버그를 발견했습니다. `poetry.lock` 파일의 `dj-rest-auth` 패키지가 `Django` 를 의존함에도 불구하고, 대소문자 정규화 문제로 이를 인식하지 못하는 케이스였습니다. 직접 소스 코드를 분석해 **case-insensitive matching** 패치를 작성하고 공식 레포지토리에 Issue를 제안하며 오픈소스 생태계에 기여해 보는 값진 경험을 했습니다.

### Ground Truth Parser 개발 및 벤치마크
실험의 객관성을 위해 직접 **GT Parser**를 개발하여 기준점(Ground Truth)을 설정했습니다.
* **Python**: `uv.lock`, `poetry.lock`, `requirements.txt` 등 재귀적 스캔
* **Go**: 모든 `go.mod` 파일 추적

이를 바탕으로 대규모 프로젝트(Terraform, Langchain) 벤치마크를 수행했으며(이외에도 여러 가지 프로젝트를 대상으로 벤치마크를 수행하였습니다.), **Accuracy (Union)** 지표를 활용해 정확도를 측정했습니다.
$$Accuracy (Union) = \frac{2TP}{|GT \cup SBOM|}$$
- 아래의 표는 Go project에 대한 Ground Truth를 대상으로 진행한 실험의 결과 입니다.

| Project | Tool | Time (s) | Accuracy (union) | Precision | Recall | F1 |
|---|---|---:|---:|---:|---:|---:|
| gin | cdxgen | 8.060 | 1.000 | 1.000 | 1.000 | 1.000 |
| gin | syft | 4.768 | 1.000 | 1.000 | 1.000 | 1.000 |
| gin | trivy | 1.731 | 1.000 | 1.000 | 1.000 | 1.000 |
| prometheus | cdxgen | 56.799 | 0.957 | 0.984 | 0.972 | 0.978 |
| prometheus | syft | 29.746 | 0.994 | 0.997 | 0.997 | 0.997 |
| prometheus | trivy | 13.508 | 1.000 | 1.000 | 1.000 | 1.000 |
| syncthing | cdxgen | 49.987 | 0.980 | 0.980 | 1.000 | 0.990 |
| syncthing | syft | 18.275 | 0.960 | 0.979 | 0.979 | 0.979 |
| syncthing | trivy | 7.988 | 0.960 | 0.979 | 0.979 | 0.979 |
| traefik | cdxgen | 108.102 | 0.988 | 0.988 | 1.000 | 0.994 |
| traefik | syft | 37.397 | 0.973 | 0.978 | 0.995 | 0.986 |
| traefik | trivy | 21.215 | 0.988 | 0.993 | 0.995 | 0.994 |
| terraform | cdxgen | 584.974 | 0.958 | 0.958 | 1.000 | 0.979 |
| terraform | syft | 86.818 | 0.968 | 0.968 | 1.000 | 0.984 |
| terraform | trivy | 75.325 | 1.000 | 1.000 | 1.000 | 1.000 |

---

## 3. Unified SBOM 프로토타입
연구실 자체 도구인 **HatBom** 의 높은 무결성과 **Syft** 의 풍부한 탐지 범위를 결합하는 파이프라인을 구축했습니다.
1. 각 SBOM 파싱 및 임시 객체화
2. Deprecated 필드 정규화
3. 중복 제거 및 데이터 소스(Property) 추가
4. 최종 통합 SBOM 추출

---

## 4. 마치며
이번 인턴십을 통해 기술적인 분석 역량뿐만 아니라, 보안 실험에서 **Ground Truth** 를 설정하는 것이 얼마나 중요한지 깨달았습니다. 발표를 준비하며 제 연구 내용을 논리적으로 전달하는 법도 배울 수 있었습니다.

도움을 주신 CCSLAB 교수님과 연구원분들께 감사드립니다.

---
**관련 프로젝트 링크:**
* **GT Parser**: [GitHub Repository](https://github.com/brian-223134/Projects-s-SBOM-Results/tree/main/code/analyze)
* **Unified SBOM Prototype**: [GitHub Repository](https://github.com/brian-223134/Unified_SBOM)