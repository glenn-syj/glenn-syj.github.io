---
title: 파이프로 살펴보는 동시성 - Xv6와 CSP Thread
lang: ko
layout: post
---

## 들어가며

- 이 글은 MIT에서 제공하는 과목인 Operating System Engineering(6.S081)을 학습하며, 파이프가 동시성 개념을 파이프가 동시성 개념을 이해하는 데 어떤 역할을 하는지 정리한 내용입니다.
- 학습용으로 사용되는 xv6에서 primes 과제를 수행하며, 각 소수마다 별도의 프로세스를 만들고 파이프를 통해 숫자를 전달하는 구조를 구현했습니다.
