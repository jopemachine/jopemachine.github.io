---
layout: post
title: "React, redux, 불변성"
subtitle: "프론트 면접 질문 정리"
author: jopemachine
tags: 
 - Frontend
 - React
 - Redux
header-img: "img/header-img/frontend.jpg"
header-mask: 0.3
---

# React, Redux와 불변성

## React의 경우

- `shouldComponentUpdate`, `useEffect` 등에서 상태의 변화를 감지할 때 매번 깊은 비교를 하는 것은 성능 상 불리한 방식임. 그래서 그냥 객체 자체를 바꾸는 식으로만 업데이트 할 수 있도록 강제하고 상태의 변화엔 항상 얕은 비교만 사용한다.

## Redux의 경우

- Redux도 마찬가지임. `combineReducers`에서 상태 변화를 감지해 상태가 변한 경우에만 새 상태를 만들어 리턴하고 변하지 않은 경우 같은 객체를 리턴해야 하기 때문에 얕은 비교를 위해 불변성이 강제된다. 그렇지 않은 경우 `combineReducers`에서 상태 변화를 감지하지 못하게 되 버린다.

## 결론

React와 Redux 둘 다 마찬가지로 효율적으로 state를 비교하기 위해 불변성이 필요하다.

객체 (상태) 를 비교할 때 매번 객체의 내부를 모두 순회하면서 비교하는 것은 말이 안 되기 때문에 얕은 비교를 사용한다.

(리덕스의 경우 히스토리 관리 (타임머신 디버깅)를 위해 필요한 것이기도 하다고 함)

그래서 리듀서는 액션을 처리한 후, 늘 새로운 객체를 리턴해야 하고, React의 state는 직접 변경하지 않고 setState 함수를 써서 변경해야 한다.

둘 다 마찬가지로 매번 스프레드 연산자를 쓰는 것 보다 `immer`의 `produce` 함수를 사용하는 편이 좀 더 가독성이 좋아진다.

## Related links

- [Redux Three Principles](https://redux.js.org/understanding/thinking-in-redux/three-principles)