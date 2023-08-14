---
title: Intel MKL 과 함께 HF candle 빠르게 만들기
description: 
published: 1
date: 2023-08-14T14:09:50.613Z
tags: 
editor: markdown
dateCreated: 2023-08-14T14:09:50.613Z
---

# 배경설명
- [이전](./minimal) 프로젝트 이후 바이너리 사이즈는 총 150 MB 내외로 흡족하게 줄였다
- 콜드부트 타임이 기존 30초 에서 2초로 확 줄었다.
- 하지만 연산속도는 의외로 느린것으로 판별되었다.
  - BERT, 150 토큰으로 고정시
    - libtorch > 20ms
    - candle > 150ms
    (싱글코어 기준)
- 좀 더 알아본 결과 libtorch 는 온갖 BLAS 를 써가면서 최적화를 해뒀고
- candle 은 별도의 최적화는 Feature 를 통해서 활성화가 가능하게 해두었다.
- 현재 시점 기준 CPU 에서 사용가능한 기능은 `intel-mkl` 
- 다른 벤치마크 결과만 봤을때 AVX2 기준 대략 10배정도 빨라지던데 이는 pytorch 와 유사한 수준이다.

# 문제점

- 컴파일이 안된다.

`cargo install --git https://github.com/huggingface/candle.git candle-examples --examples bert -F mkl`
```
  = note: /usr/bin/ld: /workspaces/Kuberian/searcher/target/debug/deps/libcandle_core-0afc8671b4dae8af.rlib(candle_core-0afc8671b4dae8af.candle_core.b11884625c01537d-cgu.13.rcgu.o): in function `candle_core::mkl::hgemm':
          /usr/local/cargo/git/checkouts/candle-0c2b4fa9e5801351/60cd155/candle-core/src/mkl.rs:162: undefined reference to `hgemm_'
          collect2: error: ld returned 1 exit status
          
  = note: some `extern` functions couldn't be found; some native libraries may need to be installed or have their path specified
  = note: use the `-l` flag to specify native libraries to link
  = note: use the `cargo:rustc-link-lib` directive to specify the native libraries to link with Cargo (see https://doc.rust-lang.org/cargo/reference/build-scripts.html#cargorustc-link-libkindname)
```
- 일단 이슈라이징: https://github.com/huggingface/candle/issues/443