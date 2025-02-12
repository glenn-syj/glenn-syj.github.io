---
title: Elasticsearch와 Lucene의 파일 포맷 관리
lang: ko
layout: post
---

## 들어가며

- Last Update: 2025/02/12

- Elasticsearch와 Lucene의 파일 포맷 관리 방식을 살펴보며, 검색엔진에서 파일 포맷을 어떻게 다루는지 알아봅니다.

## 검색 엔진 NoSQL에서의 파일 포맷 관리

문서 기반 NoSQL 데이터베이스, 특히 Elasticsearch는 문서를 저장하고 검색하는 것이 핵심입니다. 저는 요즘 Elasticsearch와 유사한 문서 기반 NoSQL 데이터베이스를 만드는 프로젝트(Feather)를 진행중인데요. 이러한 시스템에서 기본적인 파일 포맷 관리가 중요한 이유를 살펴보겠습니다.

### 문서 구조의 유연성 지원

elasticsearch는 인덱스 매핑을 통해 스키마를 정의합니다. 인덱스 생성 시 필드의 타입과 속성을 미리 정의하는 명시적 매핑(Explicit Mapping), 새로운 필드가 유입될 때 자동으로 타입을 추론하는 동적 매핑(Dynamic Mapping) 두 가지 방식을 지원합니다

이러한 매핑 정보는 클러스터 상태 중 하나로 관리되며, 인덱스 메타데이터에 저장되는데요. Elasticsearch는 이를 통해 스키마를 유연하게 관리하면서도, 데이터의 일관성과 타입 안정성을 보장하는 셈입니다.
