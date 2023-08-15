---
title: Intel MKL 과 함께 HF candle 빠르게 만들기
description: 
published: 1
date: 2023-08-15T09:21:31.779Z
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

# 삽질기

## 1차: intel-mkl 자체가 설치가 안되고 있다고 생각

- [intel-mkl-src](https://github.com/rust-math/intel-mkl-src) 에 써져있기로는 자동으로 다운로드 된다고 써놨지만, 이게 제대로 작동 안하고 있다고 생각 코드리뷰를 진행
- 재미있게도 OCI 표준을 사용해서 ghcr 을 개인 저장소처럼 사용함
- 로직상 build.rs 에서 제대로 관련 로직이 들어있음을 확인
- 실제로 crate 캐시 폴더에 `libiomp5.a` 를 포함한 각종 static library 들이 포함되어 있음을 확인
- `cargo build` 과정에서 위 경료가 제대로 `/bin/ld` 에게 제공되고 있음을 확인
- 설치는 제대로 되고있었다.

## 2차: hgemm 이 없다는것을 발견

[배경지식](https://lappweb.in2p3.fr/~paubert/ASTERICS_HPC/10-1-3605.html)
- 오류메세지를 자세히 읽어보니 [hgemm 함수](https://github.com/huggingface/candle/blob/495e0b758035039e902819f3ed059725d0593be1/candle-core/src/mkl.rs#L147)를 못찾았다고 한다.
- hgemm 은 Half precision GEneral Matrix Multiply 를 뜻하는 함수로, 메트릭스 최적화 쪽에서는 일반적으로 사용되는 용어처럼 보인다.
- float 의 정밀도에 따라서 hgemm (f16), sgemm (f32), dgemm (f64) 등이 존재한다.
- 헌데 [intel-mkl-src](https://github.com/rust-math/intel-mkl-src) 가 자동다운로드 하는 `intel-mkl` 의 버전은 2020.01
- 해당버전의 헤더파일을 까보니 hgemm 의 선언이 안되어 있는것을 발견했다.

## 3차: 최신버전 활용의 실패

- intel-mkl 을 [수동으로 설치](https://www.intel.com/content/www/us/en/developer/tools/oneapi/onemkl-download.html?operatingsystem=linux&distributions=aptpackagemanager) 했는데 지속적으로 이전버전 (2020.01) 버전만 활용하는 이슈를 발견
```bash
# 설치과정
> apt install intel-oneapi-mkl
```
- 좀 자세히 까보니 위 패키지는 동적 라이브러리인 `*.so` 만 존재하고 정적 라이브러리 생성에 필요한 `*.a` 파일이 존재하지 않음을 확인
- `apt search` 로 까본결과 `intel-oneapi-mkl-dev` 패키지가 별도로 존재하고 정적 라이브러리들 또한 이쪽에만 존재하는것을 확인

# 결론

아래와 같은 devcontainer 환경을 구축함 [커밋](https://github.com/iwanhae/Kuberian/commit/6f083b3bf90750189a4df690a6da733a814319b9#diff-3f5f9370fc4be17b456856e3aeef04db05c0ccb4b74062b626334cd28ecb9dadR1-R13)

```Dockerfile
ARG VARIANT="bullseye"
FROM mcr.microsoft.com/vscode/devcontainers/rust:1-${VARIANT}

RUN wget -O- https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB \
| gpg --dearmor | sudo tee /usr/share/keyrings/oneapi-archive-keyring.gpg > /dev/null && \
  echo "deb [signed-by=/usr/share/keyrings/oneapi-archive-keyring.gpg] https://apt.repos.intel.com/oneapi all main" | \
  sudo tee /etc/apt/sources.list.d/oneAPI.list

RUN apt update

# for builder
RUN apt install -y intel-oneapi-mkl-devel
# for runtime environment
RUN apt install -y libomp-dev
```
