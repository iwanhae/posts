---
title: Project: Minimal
description: 
published: 1
date: 2023-08-13T01:56:45.557Z
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

