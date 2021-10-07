---
layout: post
title: "ESM vs commonjs"
subtitle: "프론트 면접 질문 정리"
author: jopemachine
tags: 
 - Frontend
header-img: "img/header-img/frontend.jpg"
header-mask: 0.3
---

# ESM vs commonjs

## Common Javascript (CJS)

- 동기적으로 모듈을 import 하는 require 구문 사용

- CJS는 top level await 구문 때문에 ESM 모듈을 require 할 수 없다.

- CJS에서 ESM 모듈을 사용하려면 dynamic import 구문을 사용해 mjs 파일에 직접 접근해 사용해야 한다.

## ESM

- ES6에서 도입된 새로운 스타일의 ESM Scripts.

- ESM은 CJS와 많은 부분에서 다르게 작동하기 때문에 (this가 global object를 참조하지 않고 use strict가 기본 값이고 json을 import 해 올 수 없고 등등...) 여전히 CJS가 노드의 디폴트이다.

- CJS와 다르게 모듈 로더를 비동기 환경에서 실행한다. 스크립트를 병렬적으로 다운로드, 로드 하는 것이고 실행은 정의된 순서대로 실행된다.

- `deno`의 경우 ESM을 디폴트로 사용한다.

- Node 12.17.0 이후 부터 experimental 플래그 없이도 사용 가능하게 되었다.

- `mjs` 확장자를 사용하거나 `type: "module"`를 사용

- require를 만들어 CJS를 사용할 수 있다. **다만 default import의 케이스에만 가능**하다! (트리 쉐이킹 불가능) CJS named exports를 제공해야 한다면 ESM Wrapper로 감싸서 제공할 수 있다.

## CJS vs ESM

- CJS와 ESM을 dual로 구성하는 방법도 있다. 하지만 해당 패키지의 다른 두 버전이 함께 로드되었을 때 충돌 가능성이 있고 일반적으로 사용하기엔 무리가 있는 방법이다.

- 좋은 Dual package를 만드는 방법은 CJS 버전을 만들고 ESM wrapper로 CJS named exports를 감싼 후, package.json에 `exports`를 추가하는 것이다.

## Related links

- [충돌 가능성을 최소화 하면서 CJS, ESM dual로 구성하는 법](https://nodejs.org/api/packages.html#packages_dual_commonjs_es_module_packages)

- [Node Modules at War: Why CommonJS and ES Modules Can’t Get Along](https://redfin.engineering/node-modules-at-war-why-commonjs-and-es-modules-cant-get-along-9617135eeca1)