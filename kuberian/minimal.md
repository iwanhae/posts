---
title: Project: Minimal
description: 
published: 1
date: 2023-08-13T03:08:16.581Z
tags: 
editor: markdown
dateCreated: 2023-08-13T01:56:45.557Z
---

# 배경설명

- Kuberian 의 Backend API 는 Cloud Run 환경에서 구동된다
- Kuberian 의 컨테이너 크기는 약 800MB 정도이다.
- [rust-bert](https://github.com/guillaume-be/rust-bert) 프로젝트를 사용하고 있는데, libtorch 에 대한 의존성이 있으며 x86 환경에서 이 .so 파일의 크기가 700MB 정도 한다.
- 하는거라고는 `all-minilm-l6-v2` 모델 한번 forward 하는것 밖에 없는데 700MB 는 과하다. 
- Registry 저장소 공간 먹어서 돈 많이나가고, cold start 타임이 30초 정도 걸리는건 둘째치고, 빌드타임도 오래걸린다.
- 그리고 이 프로젝트는 별도의 DB 가 없는게 미덕이다. 앞으로 1GB 정도는 DB 공간으로 활용하게 될텐데 잘 사용하지도 않을 lib 따위에 700MB 씩이나 쓰는것은 아깝다.

# 분석

- libtorch 가 용량이 큰 이유는 어떤 CPU 환경에서도 최적의 성능을 보장하기 위해 모든 CPU Instruction 케이스에 대한 코드가 다 들어갔기 때문
- 그 중 Intel MKL 이 가장 포션이 크다.
- 실제로 aarch64 용으로 컴파일 된 libtorch 의 경우 200MB 내외로 거의 1/4 크기이다.

## 고민 1 - libtorch 커스텀 컴파일

- 해보려고 했으나 두가지 문제점 발견
  1. 16GB 메모리로는 OOM 발생. 로컬 환경에서는 어찌되었던 CI 환경에서 이정도로 고스펙 머신을 사용하고 싶지는 않음
  2. Intel MKL 을 제외하고는 어떻게 해도 컴파일 실패
- 빠른 포기

## 고민 2 - 직접 구현

- 흠...
- 공수가 너무 많이 든다.

## 결정 3 - huggingface candle 사용

https://github.com/huggingface/candle

서버리스 환경에서 쓰라고 huggingface 에서 친절하게도 candle 이란 프로젝트를 내놓았다.

감사하게도 예제코드부터가 [sentence-transformers/all-MiniLM-L6-v2](https://github.com/huggingface/candle/blob/60cd1551ca29b2e3049f18ec8e60b6f165cfe941/candle-examples/examples/bert/main.rs#L51C30-L51C68)

# 삽질 1차 - 컴파일 안됨

```bash
error: instruction requires: fullfp16
   --> /usr/local/cargo/registry/src/index.crates.io-6f17d22bba15001f/candle-gemm-f16-0.15.6/src/microkernel.rs:364:18
    |
364 |                 "fmul {0:v}.8h, {1:v}.8h, {2:v}.8h",
    |                  ^
    |
```

[gemm](https://github.com/sarah-ek/gemm) 을 HF 측에서 커스터마이징 한 라이브러리 같은데 `fullfp16` 명령어 셋이 없다고 한다.

구글링 결과 흔히 발생하는 이슈는 아닌듯

[rust 코드상으로는](https://github.com/rust-lang/rust/blob/49af618ef94182deafa1d7ee4f2f539869d1d2f2/compiler/rustc_codegen_gcc/src/attributes.rs#L62) 이 과정에 대해서 이슈가 특별히 날 이유가 없어보임

## 원인

M2 MBA, devcontainer 환경에서 개발을 진행했는데 orbStack 에서 VM 을 돌리면서 NEON Instruction set 지원이 빠진것으로 보임

본래 호스트 환경에서는 컴파일 잘됨

# PoC

```bash
./kuberian --prompt "Today is a good day to die, and here we are, with a brand new macbook vscode pro and behold!"
Running on CPU, to run on GPU, build this example with `--features cuda`
Loaded and encoded 139.136139ms
[[[-0.0298, -0.2022, -0.2228, ...,  0.1290, -0.1118, -0.0714],
  [ 0.0190,  0.3009,  0.4286, ..., -0.4848, -0.5861,  0.4919],
  [-0.1395, -0.1271,  0.1739, ..., -0.0540,  0.0456,  0.1576],
  ...
  [-0.3650, -0.1400,  0.0342, ..., -0.2788, -0.0586,  0.1826],
  [-0.0269,  0.1758,  0.2966, ..., -0.6676, -0.4124,  0.9807],
  [ 0.0298, -0.2225,  0.1597, ..., -0.0751, -0.2911,  0.2272]]]
Tensor[[1, 27, 384], f32]
Took 66.519046ms
```
쿼리 하나 임베딩 하는데 대략 65ms 내외 소모. libtorch 대비 15% 정도 빠름.
바이너리 크기 16MB, 흡족
