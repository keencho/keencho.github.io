---
title: 리액트로 멀티탭 구현하기 (feat. Recoil)
author: keencho
date: 2022-04-17 08:12:00 +0900
categories: [React]
tags: [React, Recoil]
---

# 리액트로 멀티탭 구현하기
탭을 클릭했을때 해당 탭의 컴포넌트를 보여주는 방식의 코드나 라이브러리는 인터넷에 검색해보면 많습니다. 당장 npm에만 봐도 조금 오래되긴 했지만 [이런](https://www.npmjs.com/package/react-router-tabs) 라이브러리도 존재합니다.  

하지만 탭을 클릭했을때 해당 탭을 새로운 컴포넌트로써 오픈하고 오픈된 컴포넌트들의 리스트를 관리하는 형태의 예제코드는 별로 없는것 같습니다. 그래서 오늘은 이러한 방식으로 멀티탭을 구현하는 방법에 대해 알아보겠습니다.  

### 사용 라이브러리  
아래는 이 예제에서 사용될 라이브러리 입니다. bootstrap, react-dnd, recoil, sass 를 사용하였습니다.  

package.json
```json
{
  "name": "react-multi-tab",
  "private": true,
  "version": "0.0.0",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "bootstrap": "^5.1.3",
    "path": "^0.12.7",
    "react": "^18.0.0",
    "react-bootstrap": "^2.3.0",
    "react-dnd": "^16.0.1",
    "react-dnd-html5-backend": "^16.0.1",
    "react-dom": "^18.0.0",
    "recoil": "^0.7.2",
    "sass": "^1.50.1"
  },
  "devDependencies": {
    "@types/react": "^18.0.0",
    "@vitejs/plugin-react": "^1.3.0",
    "typescript": "^4.6.3",
    "vite": "^2.9.5"
  }
}

```  

### recoil 코드 준비
이 예제는 recoil을 사용합니다. 현재 시점에는 모르는 분이 거의 없다고 생각됩니다만, 혹시 처음들어보았거나 잘 모르시다면 [이곳](https://recoiljs.org/ko/docs/introduction/getting-started) 을 읽으며 학습해보세요. 개인적으로 공식 문서를 읽는 것만으로도 쉽게 이해 가능하다고 생각합니다.  

먼저 RecoilNexus를 준비합니다. recoil은 일종의 훅이기 때문에 함수형 컴포넌트 안에서만 사용해야 합니다. 하지만 저는 유틸성 클래스에서 recoil을 사용하고 싶기 때문에 RecoilNexus를 만들어 일반 클래스에서도 사용 가능하게 하겠습니다.  
```typescript
import {RecoilState, RecoilValue, useRecoilCallback} from 'recoil';

interface Nexus {
  get?: <T>(atom: RecoilValue<T>) => T
  getPromise?: <T>(atom: RecoilValue<T>) => Promise<T>
  set?: <T>(atom: RecoilState<T>, valOrUpdater: T | ((currVal: T) => T)) => void
  reset?: (atom: RecoilState<any>) => void
}

const nexus: Nexus = {}

export default function RecoilNexus() {
  
  nexus.get = useRecoilCallback<[atom: RecoilValue<any>], any>(({ snapshot }) =>
    function <T>(atom: RecoilValue<T>) {
      return snapshot.getLoadable(atom).contents
    }, [])
  
  nexus.getPromise = useRecoilCallback<[atom: RecoilValue<any>], Promise<any>>(({ snapshot }) =>
    function <T>(atom: RecoilValue<T>) {
      return snapshot.getPromise(atom)
    }, [])
  
  // @ts-ignore
  nexus.set = useRecoilCallback(({ set }) => set, [])
  
  nexus.reset = useRecoilCallback(({ reset }) => reset, [])
  
  return null
}

export function getRecoil<T>(atom: RecoilValue<T>): T {
  return nexus.get!(atom)
}

export function getRecoilPromise<T>(atom: RecoilValue<T>): Promise<T> {
  return nexus.getPromise!(atom)
}

export function setRecoil<T>(atom: RecoilState<T>, valOrUpdater: T | ((currVal: T) => T)) {
  nexus.set!(atom, valOrUpdater)
}

export function resetRecoil(atom: RecoilState<any>) {
  nexus.reset!(atom)
}
``` 

이렇게 만든 RecoilNexus를 사용하려면 컴포넌트 렌더링하는 Main 코드에 전체 컴포넌트를 RecoilRoot로 감싸주고 내부에 RecoilNexus를 선언해야 합니다.
```tsx
ReactDOM.createRoot(document.getElementById('root')!).render(
  <RecoilRoot>
    <RecoilNexus />
  </RecoilRoot>
)
```
React18 부터는 ReactDOM.render 대신 ReactDOM.createRoot 를 써야 에러가 나지 않는다는 점을 잊지 말아주세요.  

다음은 탭 컴포넌트에 쓰일 recoil atom의 데이터 타입입니다.
```typescript
export default interface RouterComponentModel {
  displayName: string,
  uniqueKey: string,
  sequence: number,
  active: boolean,
  component: JSX.Element
}
```
브라우저에 보여질 이름, 시퀀스, 유니크키, 활성화 여부, 렌더링할 컴포넌트의 정보를 가지고 있습니다.  

다음은 recoil atom 입니다.
```typescript
import {atom} from 'recoil';
import RouterComponentModel from '@/model/router-component.model';

const RouterComponentAtom = atom<RouterComponentModel[]>({
  key: 'routerComponent',
  default: []
});

export default RouterComponentAtom;
```  

### RouterControlUtil 클래스  
다음으로 작성할 코드는 `RouterControlUtil` 클래스 입니다. 앞에서 말했다시피 저는 `RouterComponentAtom`을 컨트롤하는 코드를 모두 한곳에서 모아 처리할 곳이기 때문에 이 유틸 클래스가 제일 중요하다고 할 수 있습니다.  

##### 1. 컴포넌트 리스트 추가 / 업데이트
![save-or-update](/assets/img/custom/react/multi-tab/save-or-update.gif)

위 이미지처럼 탭을 클릭했을시 컴포넌트를 추가하고, 만약 중복탭을 허용하지 않는 경우 이미 열려있는 탭을 active로 만드는 코드를 작성해 보겠습니다.

```typescript
export default class RouterControlUtil {
  static MAX_TAB_SIZE = 15;
  static MAX_TAB_MESSAGE = `최대 탭 갯수에 도달하였습니다. (${this.MAX_TAB_SIZE} 개)`;
  
  static atom = RouterComponentAtom;
  
  static saveOrUpdateComponent = (component: JSX.Element, allowDuplicateTab?: boolean) => {
    const value: RouterComponentModel[] = getRecoil(this.atom)
    
    if (value.length >= this.MAX_TAB_SIZE) {
      alert(this.MAX_TAB_MESSAGE);
      return;
    }
    
    //////////////////// 중복탭을 허용하지 않는 경우 ////////////////////
    if (allowDuplicateTab !== true) {
      // 만약 이미 recoil에 저장된 컴포넌트가 있다면 해당 컴포넌트의 show를 true로 세팅하고 끝낸다.
      if (value.some(v => v.displayName === component.type.name)) {
        this.setRecoil([ ...value.map(v => {
          return { ...v, active: v.displayName === component.type.name} as RouterComponentModel;
        })]);
        return;
      }
      
      // 없다면 새로 추가한다.
      this.setRecoil([ ...value.map(v => {
        return { ...v, active: false } as RouterComponentModel
      }), this.generateComponentModel(component)]);
      
      return;
    }
    
    //////////////////// 중복탭을 허용하는 경우 ////////////////////
    const componentModel: RouterComponentModel = this.generateComponentModel(component);
    
    // 중복탭을 허용하는경우 마지막 시퀀스를 찾아서 새롭게 만들 탭의 시퀀스를 +1 시켜줘야 한다.
    const filteredList = value.filter(v => v.displayName === component.type.name);
    if (filteredList.length === 0) {
      this.setRecoil([ ...value.map(v => {
        return { ...v, active: false } as RouterComponentModel
      }), componentModel ]);
      return;
    }
    
    const lastComponent = filteredList.sort((pv, nx) => pv.sequence - nx.sequence)[filteredList.length - 1];
    componentModel.sequence = lastComponent.sequence + 1;
    
    this.setRecoil([ ...value.map(v => {
      return { ...v, active: false } as RouterComponentModel
    }), componentModel ]);
  }
  
  private static generateComponentModel = (component: JSX.Element): RouterComponentModel => {
    return {
      displayName: component.type.name,
      uniqueKey: crypto.randomUUID(),
      component: component,
      active: true,
      sequence: 1
    } as RouterComponentModel
  }
}
```
1. 먼저 현재 탭의 갯수를 파악하여 만약 최대 탭 갯수에 도달하였다면 메시지를 alert하고 종료 합니다.
2. 중복탭을 허용하지 않는 경우 이미 열린 탭이 있다면 해당 탭을 활성 상태로 만들고 종료 합니다.
3. 열린 탭이 없다면 새로운 탭을 추가합니다. 이떄 uniqueKey는 `crpyto.randomUUID()` 를 이용하였습니다.
4. 중복탭을 허용하는 경우 이미 열린 탭이 있다면 이미 열린 탭들중 마지막 탭을 찾아 새롭게 만들 탭의 시퀀스를 마지막탭의 시퀀스 + 1 처리해야 합니다.
5. 열릴탭을 제외한 나머지 탭들의 active는 false로 세팅합니다.  

##### 2. 컴포넌트 열기
![open](/assets/img/custom/react/multi-tab/open.gif)  

다음은 열린 탭중 하나를 클릭해 클릭한 탭을 활성화하는 코드입니다.  
```typescript
static openComponent = (component: RouterComponentModel) => {
  const value: RouterComponentModel[] = getRecoil(this.atom)
  
  this.setRecoil([ ...value.map((v) => {
    return { ...v, active: this.isUniqueKeyEqual(v, component) } as RouterComponentModel
  })]);
}
```  
클릭한 탭의 컴포넌트를 인자로 받아 loop를 돌며 uniqueKey가 동일하면 active를 true로 세팅하고, 그렇지 않다면 false로 세팅합니다.  

##### 3. 컴포넌트 닫기
![close](/assets/img/custom/react/multi-tab/close.gif)

다음은 열려 있는 탭중 하나를 닫는 코드입니다.
```typescript
static closeComponent = (component: RouterComponentModel) => {
  let value: RouterComponentModel[] = getRecoil(this.atom);
  const activeComponent: RouterComponentModel | undefined = value.find(v => v.active && this.isUniqueKeyEqual(component, v));
  const activeComponentIdx: number = value.findIndex(v => this.isUniqueKeyEqual(component, v));
  
  // 닫으려는 컴포넌트가 활성화된 컴포넌트라면 맨 앞에있는 컴포넌트를 활성 상태로 만든다 (0이라면 1)
  if (value.length > 1) {
    if (activeComponent !== undefined) {
      if (this.isUniqueKeyEqual(activeComponent, component)) {
        value = value.map((v, idx) => {
          return { ...v, active: (activeComponentIdx === 0 ? (idx === 1) : idx === 0) } as RouterComponentModel
        })
      }
    }
  }
  
  let filteredValue = value.filter(v => !this.isUniqueKeyEqual(v, component));
  
  // 시퀀스 재정렬
  if (filteredValue.length > 0) {
    filteredValue = filteredValue.map(v => {
      // 지워진 컴포넌트의 시퀀스보다 큰 시퀀스를 가진 컴포넌트
      if (v.displayName === component.displayName && v.sequence > component.sequence) {
        return { ...v, sequence: v.sequence - 1 } as RouterComponentModel;
      }
      return v ;
    });
  }
  
  this.setRecoil(filteredValue);
}
```  

컴포넌트를 닫는 코드에는 주의해야할점이 있습니다. 닫으려는 컴포넌트가 활성화된 컴포넌트라면 활성화될 컴포넌트를 지정해줘야 하지요. 따라서 컴포넌트를 삭제하기 전에 먼저 활성화될 컴포넌트를 지정하는 코드를 작성하였습니다.  

컴포넌트를 삭제한 후에는 시퀀스를 재정렬하는 코드가 필요합니다. 만약 지워진 컴포넌트의 시퀀스가 2라면 2보다 큰 시퀀스를 가진 같은 이름의 컴포넌트들은 그 시퀀스에서 -1을 해줘야 하지요.  

##### RouterControlUtil 클래스 마무리  
`RouterControlUtil` 클래스 작성이 완료되었습니다. 남은것은 뷰를 이쁘게 그리고 알맞은 시점에 알맞은 함수를 호출하는것 뿐입니다.  

### Drag and Drop 구현  
이대로 끝내긴 뭔가좀 아쉽습니다. Drag and drop으로 컴포넌트들의 순서를 변경하는 기능을 구현해보겠습니다. 아래와 같이 동작하는 코드를 작성하겠습니다.  
![re-order](/assets/img/custom/react/multi-tab/re-order.gif)

##### 1. RouterControlUtil 클래스 수정
일단 `RouterControlUtil` 클래스에 순서를 재지정하는 함수를 추가하겠습니다. 이 함수는 대상 컴포넌트와 대상 컴포넌트가 이동될 위치 인덱스를 인자로 받습니다.  

```typescript
static controlOrder = (currentComponent: RouterComponentModel, targetIndex: number) => {
  const componentModelValue: RouterComponentModel[] = getRecoil(this.atom)
  
  const draggedComponentIndex: number = componentModelValue.findIndex(value => value.uniqueKey === currentComponent.uniqueKey);
  
  if (targetIndex === draggedComponentIndex) return;
  
  const existingComponent = componentModelValue[targetIndex];
  const newComponentModelValue = componentModelValue.map((item, idx) => {
    if (idx === targetIndex) return { ...currentComponent, active: true } as RouterComponentModel;
    if (idx === draggedComponentIndex) return { ...existingComponent, active: false } as RouterComponentModel;
    
    return { ...item, active: false } as RouterComponentModel;
  });
  
  this.setRecoil(newComponentModelValue);
}
```  

1. 대상 컴포넌트의 인덱스를 검색합니다. 만약 이동될 위치와 대상 컴포넌트의 위치가 동일하다면 함수를 종료합니다.  
2. 이동될 위치에 이미 존재하는 컴포넌트를 `existingComponent` 에 할당합니다.
3. 이제 루프를 돌며 순서를 재지정 합니다. 이때 기존의 활성화 여부와 관계없이 대상 컴포넌트들 활성 상태로 만듭니다.  

##### 2. Drag and Drop 코드 작성
Drag and drop 코드를 작성해 보겠습니다. 위 패키지 설명에도 나와있듯이 라이브러리는 react-dnd를 사용하였습니다. 

```tsx
interface Props {
  component: RouterComponentModel,
  index: number,
}

const Tab = (props: Props): JSX.Element => {
  
  const [{isDragging}, dragRef] = useDrag(() => ({
    type: 'item',
    item: props.component,
    collect: monitor => ({
      isDragging: monitor.isDragging(),
    }),
  }))
  
  const [spec, dropRef] = useDrop({
    accept: 'item',
    hover: (item, monitor) => {
      RouterControlUtil.controlOrder(item as RouterComponentModel, props.index);
    }
  })
  
  const ref = useRef(null);
  const dragDropRef = dragRef(dropRef(ref));
  
  const opacity = isDragging  ? 0.3 : 1;
  
  return (
    <div
      // @ts-ignore
      ref={dragDropRef}
      className={`${style.item} ${RouterControlUtil.isActiveComponent(props.component) ? style.active : ''}`}
      style={{ ...style, opacity }}
    >
      <button
        className={style.btn}
        onClick={() => RouterControlUtil.openComponent(props.component)}>
        {RouterControlUtil.getTabName(props.component)}
      </button>
      <button className={style.btnClose} onClick={() => RouterControlUtil.closeComponent(props.component)} />
    </div>
  )
}

export default Tab
```

간단한 코드입니다. 핵심은 [spec, dropRef] 부분인데요, hover 이벤트가 일어나면 위에서 작성한 `controlOrder()` 메소드를 호출하여 순서가 변경될 수 있도록 하였습니다.

### 마무리
리액트 라우터를 사용하지 않고 멀티탭을 구현하는 방법을 소개해 보았습니다. 단, 위의 코드로는 실제 url이 변경되지 않기 때문에 만약 url이 변경되게 하고 싶다면 추가 코드를 작성해야할 수도 있습니다.

[이곳](https://github.com/keencho/react-multitab-example) 에서 전체 코드를 확인하실 수 있습니다.
