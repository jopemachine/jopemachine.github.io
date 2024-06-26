---
layout: post
title: 크롬 브라우저 아키텍쳐 공부한 내용 정리
subtitle: Browser 세부사항
author: jopemachine
tags:
  - Frontend
  - Translation
header-img: img/header-img/frontend.jpg
header-mask: 0.3
last-update: September 25, 2022
---

# Chrome 브라우저 아키텍쳐

- 브라우저 아키텍쳐 구현 사항에 표준은 없음.

- 다수의 스레드끼리 통신해도 되고, IPC를 통한 많은 프로세스들의 통신을 통해 구현해도 된다. 이런 건 그냥 세부사항일 뿐임.

- 크롬의 경우 한 탭 당 하나의 프로세스를 사용한다. 그래서 탭 하나 당 한 개의 렌더러 프로세스를 가진다. 그래서 멀티 스레드 구조의 다른 브라우저들에 비해 메모리를 많이 필요하다.

- 하지만 실행 옵션을 변경해 사이트 별로 각각의 프로세스를 사용하거나 렌더러 프로세스를 하나의 프로세스에서 처리하도록 변경할 수도 있다.

![](/img/posts/Front/2021-12-18-chrome-arch/browser-arch2.png)

## 멀티 프로세스 구조의 장점

- 한 탭에 문제가 생겨도 다른 탭들은 무탈하게 작동함.

- 각 탭들이 다른 프로세스이니 각 탭 별로 권한을 분리할 수 있음. (각각의 탭들이 샌드박스 형태로 작동)

## 각 프로세스의 임무

| 프로세스 | 하는 일 |
| --------------------------------------------------------- | -------------------------------------------------------------------------------- |
| `브라우저 프로세스` | 주소 창, 뒤로 및 앞으로 이동 버튼을 포함한 chrome 어플리케이션의 제어.
| `렌더러 프로세스` | 웹사이트가 디스플레이 될 때 탭 안의 모든 것 담당.                                                    |
| `플러그인 프로세스` | 플래시와 같은 웹사이트가 사용하는 모든 플러그인 담당. |
| `GPU 프로세스` | 다른 프로세스와 분리된 GPU 작업을 제어합니다. GPU는 여러 앱의 요청을 제어하고 동일한 표면에 표시하기 때문에 다른 프로세스로 분리됩니다. |

![](/img/posts/Front/2021-12-18-chrome-arch/browserui.png)

## iframe

- 각 `iframe`들에도 각각의 렌더러 프로세스가 실행된다.

![](/img/posts/Front/2021-12-18-chrome-arch/isolation.png)

## 브라우저 url을 검색했을 때

- 검색 창 등 탭 밖에 있는 모든 요소는 브라우저 프로세스에서 렌더링된다.

- Query가 검색어인지 *url*인지 판단해서 Navigation을 시작한다.

- TCP 연결이 수립되면 네트워크 스레드가 스트림의 처음 몇 바이트를 읽어, 문서 타입을 읽어와 (`Mime sniffing`) html 파일인 경우 렌더러 프로세스로 넘겨주고, html 파싱을 시작함.

- 데이터 타입이 `zip` 등 다른 타입인 경우 다운로드 매니저에 데이터를 넘겨, 다운로드 시작한다.

- *Safe Browsing*을 수행해 접속하려는 도메인이 악성 사이트인지 체크.

- `CORB` (Cross Origin Resource Blocking) 체크한다.

- `onload` 이벤트가 일어난 후, 브라우저 프로세스를 호출해 스피너를 멈춘다.

- `onload` 이벤트 후에도 리소스 로딩과 페이지 렌더링은 지속되어야 한다.

## onload 이벤트 처리 후 렌더러 프로세스

- 렌더러 프로세스의 핵심 역할은 html, css, js로 사용자와 상호 작용하는 웹 페이지를 만드는  것.

![](/img/posts/Front/2021-12-18-chrome-arch/renderer.png)

- HTML이 *Context free grammar*가 아니기 때문에, 일반적인 상향식, 하향식 파서로 작성되는 게 아니라 좀 복잡한 파서를 사용해야 하며, 다양한 에러들을 처리해 줘야한다. (CSS는 *Context free grammar*라 일반적인 파서를 사용할 수 있다.)

- 더 빠른 요청을 위해 `img`, `link` 태그는 파싱 중이 아니라 파싱 전 *Preload scanner*에 의해 미리 요청됨. (html 토큰화 이후 *Preload scanner* 실행)

- `img`는 **non blocking**으로 처리 되는 외부 리소스이지만, css는 브라우저 마다, 속성마다 다르게 처리되므로 주의해야 한다.

- `script` 엘리먼트는 명시된 스크립트가 다운로드 되고 실행이 끝날 때 까지 html 파싱을 block 한다. `script` 태그가 DOM을 수정하지 않는다면 `async` 속성을 사용해 스크립트가 다운로드 될 때 html 파싱이 block 되는 것을 막을 수 있다. `defer` 속성을 사용할 경우, 스크립트 다운로드 뿐 아니라 해당 스크립트 실행 자체를 html 파싱 후로 미룸으로써 html 파싱 blocking을 제거할 수 있다.

- `preload` 등의 리소스 힌트를 사용해 외부 리소스 다운로드의 우선순위를 변경 가능하다.

- DOM 트리를 html에 레이아웃이 끝난 후 시각적인 트리? 형태로 변환되는데, 이 용어의 이름이 브라우저 엔진마다 다른 것 같다. 웹킷은 `attachment`, `layout`이 끝난 후 생성된 트리의 이름을 `Render tree`라고 부르는 것 같다.

- `attachment`: DOM, CSSOM에서 스타일을 결정하고 렌더러를 만들어 렌더러 트리를 만듬.

- `layout`: 렌더러가 생성될 때 크기와 위치 정보는 아직 계산되어 있지 않다. 이 값들을 계산하는 과정을 `layout` 이라고 한다. (렌더러 인스턴스들의 재귀적인 layout 호출로 구현되어 있는 듯)

- `Render tree`엔 DOM 트리에서 `display: none` 속성이 들어 있는 등 시각적이지 않은 요소들은 제외된다.

- 렌더러의 `width`는 상위 렌더러의 `width`, 해당 렌더러의 `width`, `margin`, `border를` 사용해 계산되고, 계산된 `width`, `maxWidth`, `minWidth` 중 알맞은 값으로 설정된다.

# 원문, 참고

- [Chrome Developer Inside Browser - Part 1](https://developers.google.com/web/updates/2018/09/inside-browser-part1)

- [브라우저는 어떻게 동작하는가? - Naver d2 lab](https://d2.naver.com/helloworld/59361)

- [HTML script 엘리먼트 async vs defer 속성 차이](https://stackoverflow.com/questions/10808109/script-tag-async-defer)

- [CSS default 스타일 시트](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/core/html/resources/html.css)
