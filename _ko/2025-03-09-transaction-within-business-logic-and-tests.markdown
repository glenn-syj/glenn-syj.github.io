---
title: 비즈니스 로직과 테스트 코드로 트랜잭션 처리 방식 비교하기
lang: ko
layout: post
---

## 들어가며

- 지난 번의 [전략 패턴으로 금액 정산 비즈니스 로직 성능 비교하기](https://glenn-syj.github.io/ko/posts/2025/03/05/test-performance-with-strategy-pattern/) 포스트의 내용을 같은 팀원에게 공유하고 화상으로 설명을 진행했습니다.

- 이후 Pull Reqeust 내 코드에서 서비스 클래스 코드와 구체적 전략 코드 일부에 `@Transactional` 어노테이션이 붙어있음에 대한 질문이 들어왔습니다. 추가로, `@DataJpaTest`에 대해서도 설명하며 트랜잭션을 함께 살펴보게 되엇습니다.

- 최종 수정일: 25/03/10
