## 쿠키
Stateless protocol인 HTTP에서 상태 관리가 필요한 경우에, 브라우저의 메모리 혹은 파일로 저장되는 작은 데이터로 상태를 기록하는 방법이다.

## 쿠키 사용 목적
- 세션 관리 : 서버가 클라이언트를 기억해야 하는 상황에서 사용
    - 장바구니, 로그인 등
- 개인화 : 사용자 별 환경 설정 저장
- 추적 : 사용자의 사용 패턴 분석, 통계 데이터 등

## 쿠키의 저장
1. 서버는 응답 헤더에 `Set-Cookie: <이름>=<값>; <속성이름>=<속성값>`를 추가한다. 속성은 optional.
1. 브라우저는 쿠키를 저장하고 이후 같은 서버로의 요청 헤더에 `Cookie: <cookie-name>=<cookie-value>`를 담아 보낸다.

## 세션 쿠키, 지속 쿠키
세션 쿠키와 지속 쿠키의 차이는 만료시간의 유무이다. 
- 세션 쿠키는 만료시간과 관련된 속성이 정의되지 않은 쿠키이다. 클라이언트의 메모리에 저장되고, 브라우저의 세션이 닫힐 때 함께 제거된다. 
- 지속 쿠키는 `Expire` 혹은 `Max-Age` 속성으로 만료시간을 명시해준 쿠키이다. 디스크에 저장되고 만료 시 삭제된다.
    - `Set-Cookie: MYCOOKIE=HELLO; Max-Age=1800;`는 1800초동안 유효한 지속쿠키이다.

## 쿠키와 보안
쿠키는 패킷 캡쳐를 통해 쿠키 값을 확인하거나 브라우저에서 조작할 수 있기 때문에 보안 상 문제가 있다.
서버로부터 인증받은 유저의 패킷에서 `JSESSIONID`를 검출해 해당 세션으로 위장해 요청을 보내는 경우 혹은 XSS 공격과 같이 사용자의 쿠키를 탈취하고 조작할 수 있다. 이러한 문제를 방지하기 위해 HTTP에서는 쿠키의 보안 방법을 제공하며, Set-Cookie 헤더의 속성으로 설정할 수 있다.

### HttpOnly
javascript의 [document.cookie](https://developer.mozilla.org/en-US/docs/Web/API/Document/cookie)는 해당 쿠키에 접근해 값을 읽거나 쓰도록 하는 API이다. `HttpOnly` 속성은 이 API를 통해 쿠키에 접근할 수 없도록 한다.
예를 들면 JSESSIONID는 서버로 보내 사용자 인증을 위한 목적이지 브라우저를 통해 접근하거나 가공할 목적이 아니다. 이러한 경우 `HttpOnly`를 통해 [XSS 공격으로 세션 쿠키를 탈취](https://owasp.org/www-community/HttpOnly)하려는 시도를 막을 수 있다.
- ex. `Set-Cookie: JSESSIONID={id}; HttpOnly`

### Secure
HttpOnly로 브라우저에서 쿠키 접근을 막더라도, Wireshark 등을 활용한 패킷 캡쳐로 쿠키의 값을 확인할 수 있다. `Secure` 속성은 클라이언트의 요청이 HTTPS인 경우에만 쿠키를 전송하도록 한다. 페이로드는 암호화되었기 때문에 중간 공격자가 이를 쉽게 해독할 수 없게 된다.
- ex. `Set-Cookie: JSESSIONID={id}; HttpOnly; Secure`

### SameSite
[CSRF 공격](https://developer.mozilla.org/en-US/docs/Glossary/CSRF)은 한 서버에 인증된 사용자가 파싱 사이트에 접근해 인증 정보를 탈취당하고, 파싱 사이트가 탈취한 인증 정보로 서버를 공격하는 것이다. 
`SameSite`는 CSRF를 방어하기 위한 용도로, 방문한 웹사이트가 아닌 다른 웹사이트에서 발행된 쿠키인 [서드파티 쿠키](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies#tracking_and_privacy) 전달을 제한한다.
`strict`, `lax`, `none`이 있다.
- `strict` : 항상 first-party cookie만 전송. 즉, 사용자가 방문한 도메인이 발행한 쿠키만 전달하도록 허용
- `lax` : 안전한 (GET) 메서드만 서드 파티 쿠키를 전송. 다른 웹사이트에서 발행한 쿠키더라도, 안전한 메서드라면 쿠키 전달 허용
- `none` : 서드 파티 쿠키 전송 허용
- ex. `Set-Cookie: JSESSIONID={id}; HttpOnly; Secure; SameSite=strict`


