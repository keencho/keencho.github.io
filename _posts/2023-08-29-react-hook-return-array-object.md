---
title: React custom hook return array vs object
author: keencho
date: 2023-08-29 08:12:00 +0900
categories: [React]
tags: [React]
---

# **React custom hook return array vs object**

## **1. 개요**
리액트 자체의 훅, 또는 리액트 관련 라이브러리들의 훅을 사용하다 보면 드는 의문점이 하나 있다.

```typescript
const [state, setState] = useState<string>('');
```

```typescript
const { table, selection } = useTable();
```

어떤 훅은 리턴타입이 배열이고, 어떤 훅은 리턴타입이 객체이다. (엄밀히 따지면 배열도 객체이긴 하지만 편의상 객체라고 하겠다.)

이 둘의 차이점을 살펴보자.

## **2. 차이점?**
어떤것을 선택하던 훅이 수행하는 행위의 차이점은 없다.

리턴타입으로 배열을 사용하는 경우, 동일 훅을 같은 컴포넌트 함수 내에서 N번 사용하는 경우 배열은 순서에 기반하기 때문에 객체 형태에 비해 코드량이 줄어드는 장점(?) 이 있다.

만약 `useState()` 훅의 리턴타입이 객체였다면 어땠을까? 아래 예시를 보자
```typescript
const { state: state1, setState: setState1 } = useState<string>('');
const { state: state2, setState: setState2 } = useState<string>('');
```

배열 방식에 비해 덜 깔끔하지 않은가? 훅이 N번 사용될 경우 객체 비구조화 할당시 위와같이 별칭을 사용해야 하기 때문이다. 이것이 리액트 저자들이 `useState()` 훅의 리턴 타입을 배열로 지정한 이유이다.

물론 이것이 항상 훅이 리턴타입으로 배열을 사용해야 함을 의미하는 것은 아니다. 훅이 리턴하는 값의 갯수가 많을수록 배열은 적합하지 않다.

어떤 훅의 리턴타입이 배열이고 그 배열이 포함하고 있는 값이 5개라고 가정하고 개발자는 배열의 첫번째 속성과 마지막 속성만 사용한다고 가정하자. 우리는 훅의 모든 속성을 사용하는 것이 아님에도 배열의 구조적 특성상 변수를 할당해야 한다.

```typescript
const [v1, _1, _2, _3, v2] = useCustomHook();
```

위와같이 실제 사용하는 속성은 v1, v2 임에도 순서를 유지하기 위해 불필요한 변수를 할당해야 하는 것이다.

## **3. 그래서?**
[공식 문서](https://legacy.reactjs.org/docs/hooks-custom.html) 에서는 다음과 같이 설명하고 있다.

> Unlike a React component, a custom Hook doesn’t need to have a specific signature. We can decide what it takes as arguments, and what, if anything, it should return. In other words, it’s just like a normal function. Its name should always start with use so that you can tell at a glance that the rules of Hooks apply to it.

그냥 사용하는 용도에 따라 선택하라고 설명되어 있다.

따라서 본인은 다음과 같은 룰을 정하여 개발하고 있다.
- 같은 훅이 동일한 컴포넌트 내에서 N번 사용된다면 리턴타입으로 배열을 사용한다.
- 훅이 리턴하는 값의 갯수가 3개를 넘어가면 리턴타입으로 객체를 사용한다.

어떤 글을 봐도 위 주제에는 딱히 정답이랄게 없어 보인다. 개발자가 확실한 규칙을 갖고 훅을 개발한다면, 곧 그것이 정답이 아닐까?
