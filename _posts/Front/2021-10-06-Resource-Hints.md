---
layout: post
title: Resource Hints
subtitle: 프론트 면접 질문 정리
author: jopemachine
tags:
  - Frontend
header-img: img/header-img/frontend.jpg
header-mask: 0.3
last-update: December 19, 2021
---

# Resource Hints

## preload

- **current navigation**에 사용할 리소스들을 미리 가져온다.

- `as`로 리소스 타입을 명시하면 해당 리소스를 미리 가져온다.

## prefetch

- **next navigation**에 사용할 리소스들을 미리 가져온다.

- **next navigation**에 사용할 리소스이기 때문에 `preload` 보다 우선순위가 낮다.

- 브라우저가 바쁘거나 개체 크기가 너무 크다면 `prefetch` 요청은 무시될 수 있다.

## preconnect

- 다른 도메인에 미리 `DNS lookup`, `TLS negotiation`, `TCP handshake` 등 연결 절차를 밟음.