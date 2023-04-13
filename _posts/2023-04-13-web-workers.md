---
title: Web Worker
author: keencho
date: 2023-04-13 12:12:00 +0900
categories: [Javascript]
tags: [Javascript]
---

# **Web Worker**

## **개요**
Javascript는 싱글 스레드로 동작합니다. 브라우저 자체는 싱글 스레드로 동작하지 않으며 브라우저는 다음과 같은 스레드들로 구성되어 있습니다.

1. 메인 스레드(Main Thread): 브라우저의 주요 스레드로, HTML, CSS, JavaScript의 처리, 렌더링, 이벤트 처리, 네트워크 요청 처리 등을 담당합니다.
2. 렌더링 스레드(Rendering Thread): 브라우저 창에 내용을 그리는 데 필요한 작업을 처리하는 스레드입니다. 레이아웃 계산, 페인팅 작업 등이 여기에 포함됩니다.
3. 네트워크 스레드(Network Thread): 네트워크 요청과 응답 처리를 담당하는 스레드입니다.
4. JavaScript 엔진 스레드(JavaScript Engine Thread): JavaScript 코드를 처리하는 스레드로, V8 엔진 등의 JavaScript 엔진이 여기에 속합니다.
5. 이벤트 루프 스레드(Event Loop Thread): 비동기 처리를 위한 이벤트 루프를 실행하는 스레드입니다.
6. Web Worker 스레드: 웹 워커를 사용하여 병렬 처리를 지원하기 위해 사용되는 스레드입니다. Web Worker는 메인 스레드와 별개로 실행됩니다.

다음과 같이 팩토리얼을 계산하는 함수가 있다고 가정해 봅시다.
```html
<body>
    <button id="btn">Button</button>
</body>
<script>
    document
        .getElementById('btn')
        .addEventListener('click', function() {
            alert('You clicked the button.')
        });

    function doFactorial(number) {
        let result = BigInt(1);
        for (let i = 1; i <= number; i ++) {
            result *= BigInt(i);
        }

        return result;
    }

    doFactorial(100000)
</script>
```

버튼에 이벤트를 바인딩하고 `doFactorial()` 함수를 호출하여 팩토리얼을 계산하였습니다. 이 코드를 그대로 실행시켜보면 브라우저는 정상적으로 동작하지 않습니다.
설령 버튼 UI가 렌더링 되었다고 해도 버튼을 클릭할 수 없을 것입니다. 버튼을 클릭할 수 없는 시간은 `doFactorial()` 함수의 인자값이 커지면 커질수록 늘어납니다.

원인은 간단합니다. 렌더링, 함수 처리가 모두 메인 스레드에서 동작하기 때문입니다. `doFactorial()` 함수가 메인 스레드에서 동작하고 있기 때문에 그 기간동안은 UI와 관련된 이벤트들은 처리할 수 없게 되는 것이죠.

```javascript
async function doFactorial(number) {
    let result = BigInt(1);
    for (let i = 1; i <= number; i ++) {
        result *= BigInt(i);
    }

    return result;
}

doFactorial(1000000)
```

이번에는 `doFactorial()` 함수를 비동기 함수로 바꾸고 실행해 봅시다. 이번에는 버튼을 클릭할 수 있을까요?

안타깝자만 결과는 동일합니다. 코드 블로킹이 발생하기 때문입니다. 비동기 작업도 일반적으로 메인 스레드에서 동작하기 때문에 매우 큰 계산이나 무한 반복 루프를 실행하여 코드 블로킹이 발생하는 경우 메인 스레드가 차단되어
다른 작업을 수행할 수 없게 됩니다.

## **Web worker**
이럴때 Web worker를 사용하면 됩니다. Web worker는 독립적인 스레드를 사용하기 때문에 메인 스레드와 관계 없이 작업을 수행할 수 있습니다.

### **Web worker Scope**
Web worker는 별도의 스코프에서 실행됩니다. Web worker 스코프는 전역객체 `window` 를 갖지 않습니다. 따라서 Web worker 에서 DOM API에 접근할 수 없습니다. 만약 UI 업데이트를 수행하고자 한다면 메인 스레드와의 통신이 필요합니다.

대신 `self` 를 사용하여 Web worker만의 스코프인 `WorkerGlobalScope` 에 접근할 수 있습니다. 또한 `importScriopts()` 함수를 사용하여 외부 스크립트를 로드하고 해당 스크립트의 전역 변수에 접근할 수 있습니다.

### **Error**
Web worker 내에서 에러가 발생한 경우 메인 스레드에 전파되지 않습니다. 다만 메인 스레드에서 에러를 확인해야 하는 경우 `.onerror()` 메소드를 통해 에러를 확인할 수 있습니다.

```javascript
// Web Worker 내에서 onerror 이벤트 핸들러 등록
worker.onerror = function(event) {
  console.error(event.message);
};
```

### **작업 속도 & 자원 점유**
Web worker의 작업 속도에 영향을 주는 요소는 다음과 같습니다.

1. CPU 성능: Web Worker는 JavaScript 코드를 실행하는 데에 CPU를 사용합니다. 따라서 CPU 성능이 높을수록 Web Worker의 작업 속도가 빨라집니다.
2. 메모리 용량: Web Worker가 사용하는 메모리 용량이 많을수록 브라우저의 성능이 떨어질 수 있습니다. 메모리가 부족한 상황에서는 브라우저가 Web Worker를 중지하거나 작업 속도가 느려질 수 있습니다.
3. 네트워크 대역폭: Web Worker가 네트워크 작업을 수행하는 경우, 네트워크 대역폭이 작을수록 Web Worker의 작업 속도가 느려질 수 있습니다. 특히 대용량 파일을 다운로드하거나 업로드하는 작업을 수행하는 경우에는 이러한 영향이 큽니다.
4. 작업의 복잡도: Web Worker가 수행하는 작업의 복잡도가 높을수록 작업 속도가 느려질 수 있습니다. 예를 들어, 큰 배열을 정렬하는 작업은 더 많은 시간이 소요됩니다.
5. 웹 브라우저의 종류: Web Worker의 작업 속도는 웹 브라우저의 종류에 따라 다를 수 있습니다. 다른 브라우저에서는 작업 속도가 다르게 나타날 수 있습니다.

만약 브라우저에서 사용할수 있는 CPU의 최대치가 100이라고 가정해 봅시다. 메인 스레드에서 30의 CPU를 사용한다고 하면, Web worker의 스레드는 얼마만큼의 CPU 자원을 사용할수 있을까요?

Web worker는 별도의 스레드에서 동작하기 때문에, 메인 스레드에서의 CPU 점유율과는 독립적으로 동작합니다. 따라서 메인 스레드에서 30의 CPU를 점유하고 있더라도, Web worker 스레드는 여전히 시스템에서 사용 가능한 자원의 최대치인 100까지 사용할 수 있습니다.

하지만, 실제로 Web worker에서 사용 가능한 CPU 자원의 양은 시스템 상황에 따라 제한될 수 있습니다. 예를 들어, 운영체제나 브라우저 자체적으로 스레드별 CPU 자원의 할당량을 조정하거나, 시스템의 전체 자원이 부족할 경우 Web worker도 제한될 수 있습니다. 따라서 Web worker를 사용할 때에는, 특정 작업이 얼마나 많은 CPU 자원을 사용하는지 고려하고, 가능하면 여러 개의 Web worker를 사용하여 작업을 분산시키는 것이 좋습니다.

### **장단점**
**장점**
1. 멀티 스레드 처리: Web Worker는 메인 스레드와 별개의 스레드에서 동작하므로, 메인 스레드와 별개로 병렬 처리가 가능합니다. 따라서 복잡한 계산이나 대용량 데이터 처리 등을 빠르게 처리할 수 있습니다.
2. 안정성: Web Worker는 메인 스레드와 독립적으로 동작하기 때문에, Web Worker에서 발생하는 에러가 메인 스레드에 영향을 주지 않습니다. 또한 Web Worker는 크래시나 버그로 인해 전체 애플리케이션이 멈추는 것을 방지할 수 있습니다.
3. 사용성: Web Worker는 웹 애플리케이션에서 일반적으로 사용되는 JavaScript와 동일한 문법을 사용합니다. 따라서 개발자가 추가적인 학습 없이 쉽게 사용할 수 있습니다.

**단점**
1. UI 조작 제한: Web Worker는 메인 스레드와 독립적으로 동작하기 때문에, UI 조작이 불가능합니다. 따라서 Web Worker 내에서 UI 조작을 해야하는 경우, 메인 스레드와 메시지를 주고받아야 합니다.
2. 메모리 사용량: Web Worker는 별도의 스레드에서 동작하기 때문에, 추가적인 메모리를 사용합니다. 따라서 Web Worker를 지속적으로 사용하는 경우, 메모리 사용량이 증가할 수 있습니다.
3. 파일 로딩 제약: Web Worker는 별도의 스레드에서 동작하기 때문에, 메인 스레드와 별개의 파일 로딩을 해야합니다. 따라서 Web Worker에서 로딩할 수 있는 파일의 종류가 제한적입니다.

### **예제**
간단한 예제를 통해 Web worker의 사용법에 대해 알아보겠습니다. Web worker가 실행될 수 있는 별도의 파일이 필요합니다.

```javascript
// worker.js
self.onmessage = function(event) {
  console.log('message received');
  const number = event.data;

  let result = BigInt(1);
  for (let i = 1; i <= number; i ++) {
    result *= BigInt(i);
  }
  self.postMessage(result);
}
```

worker 파일에서 self를 이용하여 `WorkerGlobalScope` 에 접근이 가능합니다. 해당 스코프에 접근하여 메세지를 수신했을때 팩토리얼을 계산하는 코드를 작성하였습니다. 넘겨진 파라미터는 `event.data`로 접근할 수 있습니다.

계산이 끝난 경우 `self.postMessage()` 메소드를 통해 결과값을 전달합니다.

```javascript
// main.js
if (window.Worker) {
    const worker = new Worker('worker.js');
    worker.postMessage(1000000);

    worker.onmessage = function(event) {
        console.log('result: ', event.data);

        worker.terminate();
    }
}
```

1. `new Worker()` 생성자를 통해 Web worker 객체를 생성합니다.
2. `worker.postMessage()` 메소드를 통해 파라미터를 전달합니다.
3. `worker.onmessage()` 메소드를 통해 결과를 전달받고 console에 결과를 기록합니다. 이때 woker에서 파라미터에 접근하는 방식과 동일하게 `event.data`로 결과에 접근할 수 있습니다.
4. 더 이상 수행할 작업이 없을땐 `worker.terminate()` 메소드로 worker를 종료합니다.



