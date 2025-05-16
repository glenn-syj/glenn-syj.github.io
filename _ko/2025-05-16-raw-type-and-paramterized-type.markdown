---
title: Raw Type과 ParameterizedTypeReference
lang: ko
layout: post
---

## 들어가며

- 타입 안전성을 보장하지 않는 Raw Type과 이를 해결하기 위한 ParameterizedTypeReference에 대해 알아보는 글입니다. 특히, Spring Boot에서 `WebClient`를 이용해 배열 형태로 response body를 받는 경우에 대하여 살펴보고, 이에서 단순히 `List.class`를 이용했을 때의 문제 상황에서 시작합니다.

## Raw Type과 ParameterizedTypeReference
