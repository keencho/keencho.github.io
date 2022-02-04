---
title: 리액트에서 쉽게 아임포트 호출하기
author: keencho  
date: 2022-01-08 20:12:00 +0900  
categories: [React]  
tags: [React]
---

# **개요**
제가 운영중인 서비스에서는 `아임포트`를 결제 모듈로 사용하고 있습니다. 현재 홈페이지 내부에서 결제할 수 있는 페이지는 1개가 아닙니다. 따라서 각각의 컨테이너 혹은 컴포넌트에서 그때그때 아임포트 모듈을 호출해야 합니다.  

코드 재사용성을 향상시키기 위해 `recoil`을 이용하여 컴포넌트를 만들었습니다.  

## **recoil**  
현재 프로젝트에 recoil 패키지가 포함되어 있다는 가정하에 진행됩니다. 혹시 설치되어있지 않다면 [여기](https://recoiljs.org/ko/docs/introduction/installation/) 를 참고하여 recoil을 설치해보세요.  

## **코드 작성하기**  

### **IamportDataAtom**  
첫번째로 아임포트에서 사용할 Atom (상태의 단위) 을 작성하겠습니다.  

IamportModel.ts
```typescript
export default interface IamportModel {
	fire: boolean,
	data: any,
	method: string,
	callBack: any
	customData: any,
	amount: number,
	mobileRedirectUrl?: string
	notNeedBuyerInfo?: boolean
}
```

iamport.atom.ts
```typescript
import {atom} from 'recoil';
import IamportModel from 'models/iamport.model';

const IamportDataAtom = atom<IamportModel>({
	key: 'iamportData',
	default: {
		fire: false,
		data: undefined,
		callBack: undefined,
		method: 'card',
		customData: {},
		amount: 10000,
		mobileRedirectUrl: undefined
	}
});

export default IamportDataAtom;
```

결제에 사용될 정보들을 담을 인터페이스와 그 인터페이스를 활용한 atom 객체 입니다. 결제는 pc, mobile로 나뉘고 결제자 정보가 필요없는 경우도 있기 때문에 `mobileRedirectUrl` 변수와 `notNeedBuyerInfo` 변수는 옵셔널로 두었습니다.  

### **Iamport**  
아임포트 Element를 작성하겠습니다. 이 Element는 최상위 index.tsx에 포함됩니다.  

```typescript
ReactDOM.render(
  <React.StrictMode>
    <RecoilRoot>
      <App />
      <Iamport />
    </RecoilRoot>
  </React.StrictMode>,
  document.getElementById('root')
);
```  

```typescript
const Iamport = (): JSX.Element => {
	const iamportData = useRecoilValue(IamportDataAtom);
	const setIamportData = useSetRecoilState(IamportDataAtom);
	
	const defaultBuyerTel = '01011112222'
	const defaultBuyerName = '홍길동'
	const defaultBuyerEmail = 'korea@korea.co.kr'
	
	const fire = async() => {
		setIamportData({ ...iamportData, fire: false } as IamportModel)
		
		const { IMP }: any = window;
        
        const impCode = 'xxx';
		
		IMP.init(impCode);

		let data: any = {
			pg : 'html5_inicis',
			pay_method : iamportData.method.toLowerCase(),
			merchant_uid : 'service_' + new Date().getTime(),
			amount : iamportData.amount,
			custom_data : iamportData.customData,
			name : '결제 테스트',
			buyer_tel : '010xxxxxxxx',
			buyer_email : 'buyer@korea.com',
			buyer_name : '김철수',
		}
		
		if (iamportData.mobileRedirectUrl !== undefined && iamportData.mobileRedirectUrl.length !== 0) {
			data['m_redirect_url'] = iamportData.mobileRedirectUrl;
		}
		
		if (iamportData.notNeedBuyerInfo === true) {
			data['buyer_tel'] = defaultBuyerTel;
			data['buyer_email'] = '';
			data['buyer_name'] = defaultBuyerName;
		}

		IMP.request_pay(data, iamportData.callBack);
	}
	
	useEffect(() => {
		const jquery = document.createElement('script');
		jquery.src = 'https://code.jquery.com/jquery-1.12.4.min.js';
		
		const iamport = document.createElement('script');
		iamport.src = 'https://cdn.iamport.kr/js/iamport.payment-1.1.7.js';
		
		document.head.appendChild(jquery);
		document.head.appendChild(iamport);
		
		return () => {
			document.head.removeChild(jquery);
			document.head.removeChild(iamport);
		}
	}, [])
	
	useEffect(() => {
		if (iamportData.fire === true) {
			fire();
		}
	}, [iamportData])
	
	return (
		<></>
	)
}

export default Iamport
```  

1. 아임포트는 jquery를 필요로 하기 때문에 마운팅시 코드를 동적으로 생성하여 jquery를 사용할 수 있게 추가합니다. 반대로 언마운트시에는 제거합니다.  
2. useEffect를 통해 IamportDataAtom 상태를 구독합니다. 
3. 만약 fire이 true가 되면, fire() 함수를 호출하여 아임포트 결제 모듈을 호출합니다.  
4. 미리 정의된 IamportDataAtom 값을 바탕으로 아임포트 결제 모듈을 호출합니다.  

### **Business Logic**  
이제 위에 만들어둔 IamportDataAtom 에 데이터를 입력하고 fire을 true로 하여 useSetRecoilState를 통해 상태를 업데이트하기만 해주면 끝입니다.  

```typescript
...
const setIamportData = useSetRecoilState(IamportDataAtom);
...

const callIamport = async (params: object, method: string) => {
  const res: any = await OrderService.encryptAndBuildData(params);
  
  const iamportData: IamportModel = {
    fire: true,
    data: null,
    customData: res.data,
    method: method,
    amount: 10000,
    notNeedBuyerInfo: true or false
    callBack: async (data: any) => {
      if (data.success === false) {
        alert('결제 실패');
        return;
      }
      
      await orderPay(data.imp_uid, data.merchant_uid)
    }
  }
  
  if (isMobile) {
    iamportData.mobileRedirectUrl = window.location.href;
  }
  
  setIamportData(iamportData)
}

const orderPay = async(impUid: string, merchantUid: string) => {
  // 데이터를 활용한 결제처리
}
```  

모바일웹 혹은 앱에서의 결제는 PC와 달리 앱 스킴을 통해 사용자가 결제할 앱을 직접 호출하여 결제하는 방식이기 때문에 아임포트는 결제가 완료되면 mobileRedirectUrl 에 미리 정해둔 url에 `imp_success`, `imp_uid`등 결제 정보를 쿼리스트링에 담아 호출하게 됩니다.  

따라서 최초 마운트시 현재 url의 쿼리스트링을 체크하는 다음과 같은 함수가 필요합니다.  

```javascript
const checkMobileCallback = async() => {

    const getParamByName: string | undefined  = (name: string) => {
        const params: { [p: string]: string } = this.getAllParams();

        return params[name];
    }
    
    const impSuccess = getParamByName('imp_success');
    const errorMsg = getParamByName('error_msg');
    const impUid = getParamByName('imp_uid');
    const merchantUid = getParamByName('merchant_uid');

    // 아무것도 없다면 early return
    if (StringUtils.hasText(impSuccess) === false || StringUtils.hasText(impUid) === false || StringUtils.hasText(merchantUid) === false) {
        return;
    }

    await orderPay(data.imp_uid, data.merchant_uid)
}

```  

# **마무리**  
아임포트 결제 모듈을 간단하게 호출할 수 있는 컴포넌트를 


