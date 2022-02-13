---
title: 리액트에서 폴더 구조대로 라우팅하기 (feat. Vite)
author: keencho
date: 2021-08-28 20:12:00 +0900
categories: [React]
tags: [React, Vite]
---

# **Next js 처럼 라우팅하기**
Next js는 리액트에서 SSR을 쉽게 구현할수 있게 도와주는 라이브러리 입니다. Next의 `파일 시스템을 기반으로한 라우팅` 을 기본으로 하고 있습니다. ([참조](https://nextjs.org/docs/routing/introduction))
오늘은 Next 없이 Next와 같은 라우팅을 구현해보고자 합니다.

## **CSR로 구현하기?**
리액트는 lazy 와 Suspense API로 동적 로딩을 지원하고 있기는 합니다. 이를 이용해 쉽게 구현할수도 있지요. 또 [이런](https://www.npmjs.com/package/react-dynamic-route) 라이브러리도 있습니다.
하지만 결국 이는 클라이언트 사이드에서 앱이 시작될때 라우팅 규칙이 생성되기 때문에 뭔가 제 마음에 썩 들지는 않았습니다.

## **Vite**
사실 이 포스팅의 목적은 폴더 구조대로 라우팅이 아닌 최근에 관심있게 지켜보고 있는 `Vite` 라는 빌드 툴에 대해 정리하고자 하는것이었습니다. 하지만 Vite의 docs를 읽어보는 도중 폴더 구조대로 라우팅하는 방법을 찾았고, 포스팅의 목적을 바꾸게 되었습니다.

### **Vite란?**
일단 Vite에 대해 조금만 알아보고 넘어가도록 하겠습니다. Vite는 Vue js의 개발자인 Evan You가 개발한 빌드 툴입니다. Webpack 이나 Snowpack같은 툴이지요. (Snowpack에 더 가깝긴 합니다.) 2021년 8월 기준 깃헙 스타도 31.3k 정도로 핫한 빌드툴입니다.

Vite는 모던 브라우저에서 지원하는 `<script module>` 을 이용해 개발환경 구동시 필요한 모듈만 불러와서 실행하고 실제 서비스 빌드시에는 Rollup으로 번들링해주는 툴입니다. 직접 적용해서 시작해보면 시작속도가 굉장히 빠른것을 확인할 수 있습니다.
뿐만 아니라 수정시 수정한곳만 다시 리로드해서 보여주므로 리로딩 속도도 굉장히 빠른편입니다. 또한 SSR도 지원합니다.

Webpack을 대체할수 있어? 라고 물으신다면 이미 Webpack을 기반으로한 여러 라이브러리들이 많고 Vite는 IE11을 지원하지 않는등 여러 문제가 있어 아직은 시기상조라고 생각되나 추후 이 라이브러리가 더 개발된다면
충분히 대체할수 있다고 생각합니다.

## **파일 시스템 기반 라우팅**
Vite에 대해 대충 훑어봤으니 실제로 Vite를 이욯애 리액트앱을 만들고 파일 시스템 기반 라우팅을 적용해보도록 하겠습니다. 최종적인 라우팅 목표는 다음과 같습니다.

- 인덱스 라우팅
  - src/pages/index.tsx -> '', '/'
  - src/pages/user/index.tsx -> '/user'
- 중첩 라우팅
  - src/pages/user/list.tsx -> 'user/list'
- 동적 라우팅
  - src/pages/user/[id].tsx -> '/user/12', 'user/34'

### **시작하기**
다음 명령어로 Vite를 이용해 리액트 - 타입스크립트 앱을 생성합니다.
```
npm init vite@latest react-app --template react-ts
```
그리고 `react-router-dom`을 설치합니다.
```
npm install react-router-dom @types/react-router-dom
```

### **라우팅 규칙 정의하기**
Vite의 [glob import API](https://vitejs.dev/guide/features.html#glob-import)를 사용하여 glob 패턴으로 라우팅 규칙을 정의해보겠습니다. Vite의 glob은 fast-glob 과 일치하며
glob에서 지원하는 규칙을 알고 싶으시면 [여기](https://github.com/mrmlnc/fast-glob#pattern-syntax) 를 확인해보세요.

`import.meta.globEager` 를 사용하여 동기방식으로 `src/pages` 폴더 내부에 있고 확장자가 `.tsx`인 컴포넌트들을 가져와 변수에 저장하겠습니다.
```typescript
const COMPONENTS = import.meta.globEager('/src/pages/**/[a-z[]*.tsx')
```
위 패턴은 파일 이름이 소문자 a-z 로 시작하거나 중괄호 [ 로 시작하며 확장자는 tsx인 파일들을 모두 찾아옵니다.

`.map` 메소드를 사용해 `COMPONENTS`를 순회하며 경로(path) 와 컴포넌트를 값으로 가지는 `components`변수를 정의하겠습니다.
```typescript
const components = Object.keys(COMPONENTS).map((component) => {
  const path = component
    .replace(/\/src\/pages|index|\.tsx$/g, '')
    .replace(/\[\.{3}.+\]/, '*')
    .replace(/\[(.+)\]/, ':$1')

  return { path, component: COMPONENTS[component].default }
})
```
위 규칙으로 정의한 `components` 변수의 값은 다음과 같이 정의될 수 있습니다.
```
components = [
  { path: "/", component: f App() },
  { path: "/user", component: f UserList() },
  { path: "/user/:id }, component: f UserDetail() }
]
```
그리고 마지막으로 `components` 변수를 순회하며 리액터 라우트를 정의하여 리턴하는 `Routes()` 컴포넌트를 만들겠습니다.

```typescript
export const Routes = () => {
  return (
    <App>
      <Switch>
        {components.map(({ path, component: Component = Fragment }) => (
          <Route key={path} path={path} component={Component} exact={true} />
        ))}
      </Switch>
    </App>
  )
}
```

### **404 정의하기**
추가적으로 라우터에 등록된 컴포넌트가 아닐 경우 404 페이지를 리턴하도록 해보겠습니다. 위 `components` 변수와 같이 `_app.tsx`와 `404.tsx`의 경로를 저장하는 변수를 정의하겠습니다.

```typescript
const BASIC = import.meta.globEager('/src/pages/(_app|404).tsx')
```
`_app.tsx`와 `404.tsx`의 컴포넌트를 저장하는 `basics` 변수를 정의하겠습니다.

```typescript
const basics = Object.keys(BASIC).reduce((basic, file) => {
  const key = file.replace(/\/src\/pages\/|\.tsx$/g, '')
  return { ...basic, [key]: BASIC[file].default }
}, {})
```

자, 이제 위에서만든 `Routes()` 컴포넌트를 살짝 변경하겠습니다.
```typescript
export const Routes = () => {
  const App = basics?.['_app'] || Fragment
  const NotFound = basics?.['404'] || Fragment

  return (
    <App>
      <Switch>
        {components.map(({ path, component: Component = Fragment }) => (
          <Route key={path} path={path} component={Component} exact={true} />
        ))}
        <Route path="*" component={NotFound} />
      </Switch>
    </App>
  )
}
```
만약 basics 변수에 해당 컴포넌트가 없다면 리액트의 `Fragment`가 쓰이게 됩니다.

## **최종 코드**
모두 완성되었습니다. 최종 코드는 다음과 같습니다.

```typescript
// src/routes.tsx

import React, { Fragment } from 'react'
import { Switch, Route } from 'react-router-dom'

const BASIC: Record<string, { [key: string]: any }> = import.meta.globEager('/src/pages/(_app|404).tsx')
const COMPONENTS: Record<string, { [key: string]: any }> = import.meta.globEager('/src/pages/**/[a-z[]*.tsx')

const basics = Object.keys(BASIC).reduce((basic, file) => {
  const key = file.replace(/\/src\/pages\/|\.tsx$/g, '')
  return { ...basic, [key]: BASIC[file].default }
}, {})

const components = Object.keys(COMPONENTS).map((component) => {
  const path = component
    .replace(/\/src\/pages|index|\.tsx$/g, '')
    .replace(/\[\.{3}.+\]/, '*')
    .replace(/\[(.+)\]/, ':$1')

  return { path, component: COMPONENTS[component].default }
})

export const Routes = () => {
  const App = basics?.['_app'] || Fragment
  const NotFound = basics?.['404'] || Fragment

  return (
    <App>
      <Switch>
        {components.map(({ path, component: Component = Fragment }) => (
          <Route key={path} path={path} component={Component} exact={true} />
        ))}
        <Route path="*" component={NotFound} />
      </Switch>
    </App>
  )
}
```

```typescript
// main.tsx

import React, { StrictMode } from 'react'
import ReactDOM from 'react-dom'
import { BrowserRouter } from 'react-router-dom'

import { Routes } from './routes'

ReactDOM.render(
  <StrictMode>
    <BrowserRouter basename='/finance'>
      <Routes />
    </BrowserRouter>
  </StrictMode>,
  document.getElementById('root')
)
```
파일 구조는 다음과 같습니다.
```
├── index.html
├── index.css
├── package.json
├── package-lock.json
├── tsconfig.json
└── src
    ├── pages
    │   ├── 404.tsx
    │   ├── _app.tsx
    │   └── user
    │       ├── index.tsx
    │       └── [id].tsx
    ├── main.tsx
    ├── routes.tsx
    └── vite-env.d.tsx
```

## **결론**
Vite 와 globEager 를 사용해 파일 시스템을 기반으로한 라우팅을 적용해보았습니다. 적용하고 나니 참 편리한것 같습니다. 이곳에서 Vite에 대해 자세히 다루지는 않았지만
개인적으로 앞으로의 토이 프로젝트에는 Vite를 주로 사용할것 같습니다. 토이 프로젝트를 React나 Vue (Angular는 아직입니다...) 로 구성할 계획이 있으신 분들이면 한번 사용해 보시기 바랍니다.


