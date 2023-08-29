---
title: usearch with actix
description: 
published: 1
date: 2023-08-29T11:31:07.630Z
tags: 
editor: markdown
dateCreated: 2023-08-29T11:31:07.630Z
---

# Rust 환경 usearch 적용

## 배경

- embedding 한 벡터더미들에서 cos similarity 가 가장 유사한 벡터를 찾아야함
- 그냥 naive 하게 전부 순회하는것도 사실 나쁘진 않음. 100만개라고 해도 200ms 수준으로 처리 가능
- 하지만 돈아낄려면 성능이 더 좋아야함 -> indexing
- 시중에 나온건 대부분 stateful 한 별도의 서버를 띄워야함. (맘에 안듬)
- usearch -> sqlite 마냥 파일 하나만 들고 여기저기서 사용 가능

## 문제

- API (C ABI) 가 thread safe 하게 설계되어있지 않음
- Rust 는 thread safe 하지 않으면 컴파일단계에서 거부함 ㅡ,ㅡ

## 삽질

### 1. 멀티스레드로 만들기

