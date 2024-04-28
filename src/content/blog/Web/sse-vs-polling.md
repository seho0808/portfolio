---
slug: "/blog/sse-vs-polling"
date: "2024-04-20"
title: "SSE vs Polling (feat. Keep-Alive)"
subtitle: "브라우저 - 서버 간의 지속적인 통신 방법에 대해 알아보자"
---

## **SSE vs Polling (feat. Keep-Alive) - 미완성 포스트**

<p class="text-time">최초 업로드 2024-04-29 / 마지막 수정 2024-04-29</p>

내가 육군본부에서 구현했던 AI 모니터링 체계가 일반 폴링(regular polling) 방식이었는데, 부하를 확인했을 때 적어서 굳이 추가적으로 최적화를 해주지 않았던 기억이 있다.

그때 서버가 겪고 있던 부하를 요약해보자면:

- 부하 1: 두 개의 타 서버의 CPU 점유율 초 단위로 받기
- 부하 2: 수십 명의 피감시 체계 유저 브라우저에서 날아오는 하루 몇 천건의 유저 행동
- 부하 3: AI 모니터링 체계가 사용중이라면 거기서 요청하는 5초 단위의 감시데이터

아쉬운 점을 요약해보자면:

- 부하 1의 경우: 초 단위로 일방적으로 받기 때문에 SSE를 쓰는게 더 좋지 않았나 싶다. (비록 서버-클라이언트가 아닌 서버-서버의 SSE이지만.)
- 부하 2의 경우: 이것은 산발적으로 오는 요청들이기에 일반 http 요청이 좋은 것 같다.
- 부하 3의 경우: 일단 AI 모니터링 체계를 사용하는 유저 수가 10명 미만으로 예정되어있었기에 (유지보수하는 부서만 보고 있으면됨) 요청량이 많지는 않다. 그리고 대시보드 실시간 데이터의 주기가 5초로 주기가 꽤 길어서 SSE 보다 일반 HTTP요청을 써도 괜찮아보이고, 같은 데이터라도 무조건 다시 받아와야하기에 일반 폴링도 문제가 없어 보인다.

부하 1이 가장 아쉬운 부분인 것 같다.

내가 위 개선점을 고민하면서 SSE와 폴링 모두 Keep-Alive 옵션을 사용하는 것을 보았기에 정확하게 어떤 차이가 있는지 잘 이해하고 싶었다.
공부를 하다보니 http/1.1, http/2, http/3에서의 Keep-Alive 관련 옵션이 너무 상이하게 적용되어서 이 개념들을 먼저 정리하고 넘어가야했다. (Keep-Alive는 http/2와 3에서는 지원을 안하며, 사파리에서는 오류를 일으킬 수도 있다. 자세히 알아보자.)

<br/>

### **HTTP/1.1에서의 지속적인 통신**

HTTP/1.1의 [2022년 Keep-Alive 1.0과의 차이](https://datatracker.ietf.org/doc/html/rfc9112#name-keep-alive-connections)과 [2022년 Persistence 스펙](https://datatracker.ietf.org/doc/html/rfc9112#section-9.3)과 [1997년 Persistence 스펙](https://www.rfc-editor.org/rfc/rfc2068#section-8.1)을 참고해서 적어보았다:

- HTTP/1.0에서는 Keep-Alive를 명시적으로 적어주어야하고 실험적 구현이었기에 서버에 따라서 제대로 구현이 안되어있을 수도 있다.
- 하지만 <span class="text-skyblue">HTTP/1.1에서는 기본적으로 Keep-Alive설정이 생략되어도 적용된다.</span>
- 다만, [timeout과 max 파라미터](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Keep-Alive)의 디폴트 값은 스펙에는 적혀있지 않다.
- 그렇기에 HTTP/1.1에서 `Connection: Keep-Alive`와 함께 `Keep-Alive: timeout=5, max=1000`를 원하는 값으로 헤더에 주는 것은 유의미하다. (서버 마다 구현한 디폴트 값이 다를 수 있기에)

추가적인 디테일

- Connection 옵션이 "close"로 설정된 경우: 현재 응답 이후 연결이 지속되지 않는다.
- 지속적인 연결을 지원하지 않는 클라이언트는 모든 요청 메시지에 "close" 옵션을 포함해야 한다.
- 지속적인 연결을 지원하지 않는 서버는 1xx 코드가 아닌 모든 요청 메시지에 "close" 옵션을 포함해야 한다.
- 파이프라이닝(요청 응답 받기 전에 클라이언트에서 계속 추가 요청 보내는 것) 방식으로 클라이언트는 보내"도"(MAY) 된다. 서버는 그 응답을 받아서 병렬 처리를 해주어도 되고, 직렬로 처리해도된다. 다만, 받은 요청 순서대로 응답을 보내주어야한다. HTTP에는 TCP처럼 패킷 순서를 알려주는 방식이 표준에는 없기에 중요하다. => HTTP/2.0에서 해결하고자하는 문제 중 하나다.
- 매우 매우 놀랍게도 커넥션 끊기에는 명확한 기준이 없다. (다람쥐 책 - HTTP 완벽 가이드 참고)
- 커넥션 끊기가 서버랑 클라이언트 어느쪽에서든 발생할 수 있고 오류에 취약하기에 멱등하지 않은 POST의 경우 파이프라이닝을 사용하지 않는 것을 권장한다. (다람쥐 책 - HTTP 완벽 가이드 참고)
- [TCP Keep-Alive와 HTTP Keep-Alive는 완전 다르다](https://stackoverflow.com/questions/9334401/http-keep-alive-and-tcp-keep-alive). RFC 1122 4.2.3.6 TCP Keep-Alives 명시를 수십분 동안 열심히 읽었는데, 알고보니 이름만 같았다.

<br/>

### **HTTP/2에서의 지속적인 통신**

<span class="text-skyblue">HTTP/2는 기본적으로 지속적이다.</span>

HTTP/1.1에서 Status Line, Headear, Body를 묶어서 Message라고 부르며 이것이 하나의 요청 단위이다.
HTTP/2에서는 Frame과 Message와 Stream이 하나의 요청 단위이다:

- Frame: Header 혹은 Data (HTTP/1.1로 치면 Header 혹은 Body의 일부)
- Message: 여러 개의 Frame. 하나의 메세지 단위임. (HTTP/1.1로 치면 Status Line + Header + Body)
- Stream: 하나의 응답을 위한 하나의 요청. 여러 개의 요청이 동시에 오고 갈 수 있음. (HTTP/1.1로 치면 한 번의 응답을 위한 요청.)

------------스트림 이미지 1 - 이미지 출처 넣기-----------

------------스트림 이미지 2 - 이미지 출처 넣기----------- - 마치 CPU의 컨텍스트 스위칭 PCB 순서처럼 쪼개져서 옴.

이렇게 설계(멀티플렉싱)했기에 HTTP/1.1에서처럼 여러 개의 TCP요청으로 리소스 여러 개를 병렬로 가져오는 것이 아니라 하나의 연결 안에서 모든 리소스를 요청할 수 있고,
이러면 당연히 지속적인 통신일 수 밖에 없다. 그럼 HTTP/2는 도대체 언제 끊기냐!면 [스펙 9.1 Connection Management](https://datatracker.ietf.org/doc/html/rfc9113#name-connection-management)에 따르면 클라이언트는 그 페이지에서 나가거나 완전히 이탈할 때까지 연결을 유지해야하고, 서버는 최에에에대한 연결을 유지하다가 idle로 판단되면 GOAWAY 프레임을 클라이언트에 보내야한다고 한다. 그리고 재미있는 점은 크롬 탭이랑 윈도우끼리도 동일한 도메인에 대해서는 HTTP/2통신을 공유한다는 [카더라](https://stackoverflow.com/a/75502115/14971839)가 있다.

<br/>

### **HTTP/3에서의 지속적인 통신**

HTTP/3은 구글의 QUIC(Quick UDP Internet Connections)기반이고 QUIC은 UDP기반이다. <span class="text-skyblue">흥미롭게도 TCP에서 가지는 핸드셰이크, 연결성, 패킷 순서 보장등을 UDP위에서 구현한 것이 QUIC이다. 그렇기에 멀티플렉싱을 제공하는 지속적인 통신이 기본 옵션이라고 생각해야할 것이다.</span>

- 멀티플렉싱을 HTTP의 애플리케이션 레이어가 아닌 트랜스포트 레이어에서 구현하기 때문에, HTTP/2보다 안정적으로 멀티플렉싱이 가능하다. HTTP/2는 멀티플렉싱 도중에 하나의 스트림에서 문제가 생기면 다른 스트림들이 다 일시정지가 된다고 한다. 이런 문제를 HTTP/3에서는 근본적으로 트랜스포트 레이어에서 해결한다.
- HTTPS를 HTTP/2이하에서 구현하면 TCP 핸드셰이크 이후 TLS 핸드셰이크를 하는데, QUIC은 한 번의 핸드셰이크에 TLS를 포함해버려서 더 빠르다고 한다.
- QUIC이 일부 환경이나 컴퓨터에서 HTTP/2 보다 느리다는데, 이건 카더라라서 나도 잘 모르겠다. 나중에 확인해보자.

이외에도 수많은 디테일들이 있지만 생략했다. 나중에 더 잘 알게되면 여기에 추가하도록하자. RFC는 9000번이다.

<br/>

### **일반 폴링 (Regular Polling)**

일반 폴링은 정해진 시간(n초)에 따라 HTTP 요청을 주기적으로 보낸다.

- HTTP/1.0: Keep-Alive가 없다면 매번 TCP 핸드셰이크를 한다.
- HTTP/1.1: Keep-Alive가 디폴트이기에 자동으로 요청 사이의 TCP 연결이 유지된다. 폴링 시간 간격이 길어서 timeout되면 TCP 3-way 핸드셰이크가 매번 수행될 수도 있다.
- HTTP/2와 3: 통신이 거의 항상 유지되는 것으로 예상되기에 새로운 TCP Handshake는 거의 발생하지 않는다. 하나의 스트림이 생겼다가 응답 후 사라진다.

<br/>

### **롱 폴링 (Long Polling)**

[롱 폴링](https://ko.javascript.info/long-polling)은 "데이터가 바뀌거나 처리된 후 응답을 받고 싶은 경우" 유용하다.

- HTTP/1.0: 맨 처음에 클라이언트가 서버에 요청을 보내고나서 데이터가 돌아오는 시간이 1초가 될지, 3초가 될지, 10초가 될지 미정이다. 서버에서 응답이 올 때까지 클라이언트에서 통신을 열어두다가, 응답이 오는 즉시 TCP통신을 끝낸다. (혹은 Keep-Alive로 유지한다.) 끝낸 후에 다시 다음 요청을 보낸다. (직후가 될 수도 있고 몇 초 기다려도 되는듯하다.)
- HTTP/1.1: 1.0과 동일하지만, 원래 Keep-Alive가 디폴트로 적용된다. timeout과 max의 정확한 수치 조정이 필요하다면 적용한다.
- HTTP/2와 3: 서버에서 언젠가 응답이 돌아올 때까지 하나의 스트림을 유지하는 방식으로 구현할 수 있다.

<br/>

### **SSE (Server Side Events)**

SSE는 서버에서 클라이언트로 계속해서 데이터를 푸시하는 단방향 HTTP 연결이다. => 내일 자세히 조사

- HTTP/1.0: 지원되지 않는다.
- HTTP/1.1: `Content-Type: text/event-stream`을 보내면 SSE가 활성화된다. => 내일 자세히 조사
- HTTP/2와 3: 연결 하나에서

<br/>

### **TCP 연결 하나당 유지 비용?**

<br/>

### **Polling 외에 Keep-Alive의 사용처**

- 동영상이나 오디오 스트리밍에서 Keep-Alive를 사용한다고 함.
- 웹 소캣 연결 초기화 시 HTTP를 사용하는데 이때 Keep-Alive가 사용된다고 함.
- ...

<br/>

### **마치며**

생각보다 깊이가 매우 깊은 주제였다. 원래는 HTTP 1.1만 하려고했는데 2.0과 3.0의 점유율을 찾아보니 50%를 넘어가는 시대가 되어서, 역시 웹의 세계는 빠름을 느꼈다. HTTP/1.1 Keep-Alive 스펙까지는 탐구할만했지만, HTTP/3.0이 UDP기반으로 돌아간다는 것을 보고 조금 지치기 시작했다. HTTP/2.0과 HTTP/3.0도 다시 읽어봐야겠다. 근데 다람쥐책 분명히 1년전에 한 번 정독했는데 진짜 뇌에서 거의 다 증발해버린 것 같다. 역시 실무 경험 기반으로한 공부가 잘 안까먹는 것 같긴하다. 흥미로운 점은, 다람쥐 책을 읽으면서 동시에 RFC를 읽어보니, 다람쥐 책의 상당 부분이 곧 HTTP RFC 내용들을 그대로 풀어서 적은 것이었다는 점이다.

<br/>

### **참고 자료**

- [롱 폴링](https://ko.javascript.info/long-polling)
- [SSE 스펙](https://html.spec.whatwg.org/multipage/server-sent-events.html)
- HTTP 완벽 가이드 4.5장 지속 커넥션 (다람쥐 책)
- [RFC 1122 - 4.2.3.6 Keep-Alives - 이건 HTTP Keep-Alive가 아니라 TCP Keep-Alive이다.](https://datatracker.ietf.org/doc/html/rfc1122)
- [RFC 9112 - HTTP/1.1 개정판 스펙](https://datatracker.ietf.org/doc/html/rfc9112)
- [RFC 9113 - HTTP/2.0 개정판 스펙](https://datatracker.ietf.org/doc/html/rfc9113)
- [쉬운 QUIC 설명 영상](https://www.youtube.com/watch?v=y8xHJJWwJt4&ab_channel=PieterExplainsTech)