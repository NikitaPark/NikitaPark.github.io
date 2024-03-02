---
title: '[JavaScript Trick] Value & Casting'
date: 2024-03-01
categories: [Web, Develope, Cyber Security]
tags: [JavaScript, Web Hacking, Logic]
---
## **✨ Introduction**

이번 포스트에서는 얼마 전 참여한 CTF 웹 해킹 문제 중 SQL Injection과 자바스크립트의 조건문 평가 방식을 이용한 사례가 있어, 이론적인 내용을 기록할 겸 소개하고자 합니다.

해당 문제는 SQL injection을 통한 인증우회를 통해 Flag를 가져오는 문제였습니다. 허나, 웹 서버 측 코드에는 제출한 사용자 이름이 코드 내부의 관리자 권한 유저 리스트에 포함되어 있는 지에 대한 여부를 검사하여, 관리자 권한이 있는 사용자에게만 Flag를 노출 시키도록 설정되어있었습니다.

결론적으로, 자바스크립트의 조건문과 값에 대한 평가 방식을 이용하여 해당 로직을 우회할 수 있었으며, Flag에 접근할 수 있었습니다.

이번 포스트에서 자세한 Write up에 대해선 다루지 않겠습니다. 다만, 사전에 이론적인 내용들과 함께 보안적인 관점에서 이러한 내용들이 어떻게 작용할 수 있는 지에 대해 소개해보도록 하겠습니다. 

## **🔢 Primitive vs Reference value type**

자바스크립트에서 변수는 **원시와 참조 자료형**, 이렇게 크게 두 가지 종류의 자료형으로 저장할 수 있습니다. 다음은 각 자료형 별 특징과 예시입니다.

### Primitive value type (원시값, 원시 자료형)

원시값 또는 원시 자료형이란 가장 기초적(저수준)인 데이터 유형으로, 불변하다는 특징을 가지고 있습니다. 

아래는 원시값의 종류입니다 :

- String (문자열)
- Number (숫자)
- Boolean (불대수)
- undefined 또는 null
- Symbol

위와 같은 형태의 값들은 선언 및 초기화 시 변하지 않는다는 불변성을 가지며, 변수에 새로운 값을 재할당 하면 새로운 메모리 공간에 값을 저장합니다.

```jsx
let num = 10;  // 특정 메모리 주소(공간)에 10을 할당, num은 해당 주소를 참조
num = 5;       // 다른 메모리 주소에 5를 할당, num은 새로운 주소를 참조
```

### Reference value type (참조값, 참조 자료형)

참조값 또는 참조 자료형이란 원시값을 제외한 데이터 유형으로, 특정 메모리 구간에 저장된 비원시 데이터의 주소를 참조하여 저장하여, 데이터의 값이 변경될 수 있습니다. 

아래는 참조값의 종류입니다 :

- Object (객체)
- Array (배열)
- Function (함수)

위와 같은 형태의 값들은 선언 및 초기화 시, 변수가 참조하는 메모리 주소에 위치한 데이터 변경됩니다.

```jsx
let obj = {num: 1};  // 특정 메모리 주소(공간)에 {num: 1}을 할당, obj는 해당 주소를 저장
obj['str'] = 'string';  // obj에 저장된 주소에 위치한 값을 수정
```

## **🤔 Truthy & Falsy value**

Truthy와 Falsy, 참 또는 거짓 같은 값이란 어떠한 조건문에서 참 또는 거짓으로 간주되는 값을 의미합니다. 즉, 직접적인 Boolean 값인 `True 또는 False` 가 아님에도, 참 또는 거짓으로 간주될 수 있다는 것을 의미합니다.

다음은 참 또는 거짓 같은 값의 예시입니다:

```jsx
// [Truthly value]
// - 어떠한 형태로든 존재하는 참조 값
if ({})
if ([])

// - 0을 제외한 숫자 또는 문자열, 
if ("pwnpwn")
if (100)
if (12n)
if (-0.1)
if (Infinity)
...

// [Falsy value]
// - 결과가 숫자 0인 값
if (0)
if (-0)
if (0n)

// - 빈 문자열, null, undefined, NaN, document.all
if ("")
if (null)
if (undefined)
if (NaN)
if (document.all)
```

위와 같은 값들이 연산의 결과로서 제공될 때 각각 참 또는 거짓으로 평가될 수 있습니다. 

<div class="card text-bg-dark">
    <div class="card-body">
        <p class="card-title fw-bold">💡 document.all 이 falsy 인 이유</p>
        <p>
            document.all 은 원래 IE(Internet Explorer) 에서 제공되었던 속성으로, 모든 HTML 요소를 담는 컬렉션을 제공하는, 현재는 비표준 속성입니다.  
            때문에 현재의 브라우저에선 명시적으론 지원하지 않으며, 하위 호환성을 위해서만 제공하고 있습니다.  
            따라서, 웹 표준에 더 알맞는 속성 사용을 유도하기 위해 document.all 이 논리 연산에선 undefined 또는 null로 변환되어 평가되도록 한 것입니다.
        </p>
    </div>
</div>

### Boolean coercion

자바스크립트에서 연산 결과의 기대값이 boolean 값일 경우, 다시 말해 조건식인 경우에,  피연산자를 boolean 값으로 강제 변환됩니다.

다음은 각 boolean 값으로 변환되는 값들의 예시입니다.

- False로 변환
    - null, undefined, NaN
    - 0, -0, 0n
    - 빈 문자열
- True로 변환
    - 0이 아닌 숫자
    - 문자열
    - 객체와 메서드

값의 변환 규칙을 보면 앞서 다룬 Truthy & Falsy 값들과 비슷하게 처리된다는 것을 알 수 있습니다. 이는 해당 값들이 조건문 연산식의 피연산자로써 사용될 때, 위와 같이 boolean 값으로 변환되어, 참 또는 거짓 같은 값으로 분류할 수 있는 것입니다.

## **⚔️ 보안의 관점에서**

앞서 다룬 원시값과 참조값, 참 또는 거짓 같은 값이 논리 구조에서 평가될 때, 때때로 개발자가 의도하지 않은 결과를 반환하기도 합니다. 
이해를 돕기 위해 아래 웹 어플리케이션을 예시로 들어 설명하겠습니다.

```jsx
const express = require('express');
const app = express();
const PORT = 3000;

const data = {};

let secret_key = Math.floor(Math.random() * 0xFFFFFFFF).toString(16).padStart(8, '0');
data[secret_key] = 'pwnpwn';

app.get('/try', (req, res) => {
    const key = req.query.key;

    try {
        if (data[key]) {
            res.status(200).json({
                message: `Success! The key '${key}' exists.`,
                value: data[key]
            });
        } else {
            res.status(404).json({
                message: `Error: The key '${key}' does not exist in the object.`
            });
        }
    } catch (error) {
        res.status(500).json({
            message: `An error occurred: ${error.message}`
        });
    }
});

app.listen(PORT, () => {
    console.log(`Server is running on port ${PORT}`);
});
```

위 어플리케이션은 `data`  객체에 무작위 8자리의 16진수를 키값으로 하는 데이터를 저장하고, `/try` 엔드포인트에서 `data` 객체에 사용자가 제공한 `key` 파라미터 값을 이용하여 값을 가져와 실제 값이 존재하는지 유무를 판단하여 사용자에게 결과를 다시 반환합니다.

위와 같은 구조에서 객체가 논리 연산에서 참으로 평가된다는 특성과 자바스크립트 객체 자체의 특성을 이용하면, 굳이 무작위 key 값을 알 필요 없이 성공 메세지를 받을 수 있습니다.

![prototype 객체 호출을 통한 논리 연산 우회](/assets/img/post/2024-03-01_1.png)
_prototype 객체 호출을 통한 논리 연산 우회_

위 실행 결과에서 `key`는 무작위로 지정됨에도 불구하고, 조건문에서 다른 `key`를 호출하여 결과를 참으로 만든 것을 알 수 있습니다.
이는 `data` 객체의 프로토타입 객체를 호출하여, 조건식의 결과가 `undefined`가 아닌 객체 즉, 참으로 평가되기에 가능한 것입니다.

위 예시에서도 나타나듯, 이러한 특성을 이용하여 공격자는 조건식에서 참으로 평가될 수 있는 값을 삽입하여 특정 구간 우회나 경우에 따라선 권한 검증 우회 (Bypass ACL) 등의 공격도 시도할 수 있습니다.

<div class="card text-bg-dark">
    <div class="card-body">
        <p class="card-title fw-bold">💡 Prototype 객체란?</p>
        <p>
            자바스크립트는 모든 객체들의 기본 형태 격인 프로토타입 객체를 상속 받습니다. 이를 통해 각 객체는 부모 객체의 메서드와 속성을 상속받아 이용할 수 있으며, 클래스 없이도 객체를 생성할 수 있습니다.
            프로토타입 객체는 <code class="language-plaintext highlighter-rouge">‘__proto__’</code> 속성 또는 <code class="language-plaintext highlighter-rouge">Object.getPrototypeOf</code> 로 호출할 수 있습니다.
        </p>
    </div>
</div>

### 조치/예방 방안

사용자 입력값을 객체의 속성 또는 접근자로써 사용할 때 논리 결함을 방지하기 위해선 적절한 조치 방안 적용과 로직 일관성을 유지 해야 할 것 입니다.

다음은 자바스크립트 값의 특성으로 인한 논리적 결함을 조치 및 방지하기 위한 방안입니다:

#### Strict equal operator (===)

Strict equal operator, 엄격한 비교 연산자는 비교연산에서 피연산자의 값과 자료형이 모두 동일한지를 비교합니다. 일반적인 equal operater (==) 는 각 피연산자의 자료형이 다를 시, 경우에 따라서 피연산자의 값을 형변환하여 비교합니다.

따라서 비즈니스 로직에서 어떠한 데이터를 서로 동일한지 비교해야 할 경우, 엄격한 비교 연산자를 사용하여, 로직 일관성을 높일 수 있습니다.

![일반 비교 연산자와 엄격한 비교 연산자의 실행 결과 ](/assets/img/post/2024-03-01_2.png)
_일반 비교 연산자와 엄격한 비교 연산자의 실행 결과_

#### Validation & Input filtering

사용자 입력값이 서버 측 코드 내부에 존재하는 객체의 접근 속성으로써 사용되는 경우, 로직 우회를 방지하기 위해 특정 입력값에 대한 필터링과 유효성 검증이 필요할 것입니다.

아래는 자바스크립트에서의 필터링 항목 예시입니다:

- Internal slot/method property (내부 슬롯 또는 메서드 속성):
    
    `__proto__, __defineGetter__, __defineSetter__, __lookupGetter__, __lookupSetter__` 
    
- window 객체의 속성
    
    `alert, location, document, eval, fetch …`
    

## **📑 Reference**

- [MDN Web Docs - Truthy](https://developer.mozilla.org/en-US/docs/Glossary/Truthy)
- [MDN Web Docs - Falsy](https://developer.mozilla.org/en-US/docs/Glossary/Falsy)
- [MDN Web Docs - Primitive](https://developer.mozilla.org/en-US/docs/Glossary/Primitive)
- [MDN Web Docs - Boolean coercion](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Boolean#boolean_coercion)
- … and the help of ChatGPT**🤖**