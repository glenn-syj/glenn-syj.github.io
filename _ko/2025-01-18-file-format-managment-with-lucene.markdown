---
title: Elasticsearch와 Lucene의 파일 포맷
lang: ko
layout: post
---

## 들어가며

- Last Update: 2025/02/16

- Elasticsearch와 Lucene의 파일 포맷 관리 방식을 살펴보며, 검색엔진에서 파일 포맷을 어떻게 다루는지 알아봅니다.

## Lucene의 파일 포맷 관리

문서 기반 NoSQL 데이터베이스, 특히 Elasticsearch는 문서를 저장하고 검색하는 것이 핵심입니다. 저는 요즘 Elasticsearch와 유사하나 더욱 가벼운 문서 기반 NoSQL 데이터베이스를 만드는 프로젝트, Feather를 진행중인데요.
Elasticsearch는 검색 엔진 라이브러리 Lucene의 주요 파일 확장자를 이용해 기능을 확장하고 있습니다. 따라서 이번 글에서는 Elasticsearch에서 이용되는 Lucene의 파일 포맷 관리 방식을 살펴보고자 합니다.

### 세그먼트 파일

Lucene에서 세그먼트(Segment)는 색인된 문서들의 집합을 의미합니다. 세그먼트라는 이름은 데이터를 작은 단위로 분할(segmentation)한다는 의미에서 유래했는데요. 각 세그먼트는 독립적인 검색 가능한 단위로, 자체적인 색인 구조를 가지고 있습니다. 세그먼트 파일이 가지는 특징은 다음과 같습니다.

#### 세그먼트 파일의 특징

1. 불변성

- 한번 생성된 세그먼트 파일은 수정되지 않습니다.
- 문서의 업데이트나 삭제가 필요한 경우, 새로운 세그먼트를 생성하고 기존 세그먼트를 제거하는 방식으로 동작합니다.

2. 증분적 업데이트

- 새로운 문서가 추가될 때마다 새로운 세그먼트가 생성됩니다.
- 불변성과 함께, 기존 세그먼트에 대한 동시성 제어를 용이하게 합니다.
- 이는 곧 NRT(Near Real-Time) 검색을 가능하게 합니다.

3. 병합

- 세그먼트 파일의 수가 많아지면 검색 성능이 저하될 수 있기에 병합이 필요합니다.
- 세그먼트 수 임계값 설정을 초과하거나 크기가 불균형할 때 병합이 이루어집니다.
- 병합은 리소스를 많이 사용하는 작업이므로, 백그라운드에서 점진적으로 수행됩니다.

4. 파일 구조:

- 각 세그먼트는 각각 용어 사전, 문서 저장, 필드 정보 등의 다른 정보를 보유한 여러 파일로 구성됩니다.
- 즉, 하나의 논리적 세그먼트는 여러 물리적 파일로 구성됩니다.

## References

[^1]: Apache Lucene Documentation - [Index File Formats](https://lucene.apache.org/core/9_9_1/core/org/apache/lucene/codecs/lucene99/package-summary.html#package.description)
