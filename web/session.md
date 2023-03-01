## Session
세션은 서버가 클라이언트의 상태를 저장하기 위한 공간이다. HTTP는 stateless한 특성이 있기 때문에, HTTP를 통한 이전 요청과 이후 요청 사이의 연관성을 가질 수 없게 된다. 따라서 서버가 개별 클라이언트를 각각 식별하고 상태를 세션 공간에 보관함으로써, stateless로 인한 한계를 보완할 수 있다. 

세션은 일반적으로 서블릿 컨테이너의 메모리 공간에 할당되는 저장 영역이다. 각각의 클라이언트에 대해 별도의 세션 공간이 할당되기 때문에, 서버의 메모리 사용량이 커질 수 있다. 따라서, 세션 공간에 대해 적절한 expiration time을 설정해줌으로써 세션의 메모리 해제를 보장해주어야 한다.

[HttpSession#setMaxInactiveInterval](https://docs.oracle.com/javaee/6/api/javax/servlet/http/HttpSession.html#setMaxInactiveInterval(int))은 세션의 만료 시간을 초 단위로 설정해주는 메서드이며, n-초 동안 접근되지 않은 세션 공간은 사라지게 된다. n <= 0일 경우 세션은 애플리케이션이 종료될 때 까지 살아있게 되며, 메모리 누수, starvation이 발생할 수 있다.

[HttpSession#invalidate](https://docs.oracle.com/javaee/6/api/javax/servlet/http/HttpSession.html#invalidate())를 통해 세션에 저장된 객체를 삭제하고 세션 공간을 해제할 수 있다. 필요하지 않은 메모리를 빠르게 회수함으로써 메모리 낭비를 막을 수 있다. 또한, 로그아웃한 사용자의 경우 세션 공간을 즉시 해제하지 않으면 세션이 만료되기 전에 해당 유저의 정보를 사용할 수 있게 되어 보안 위험이 있다.

## 세션 정보의 공유
클라이언트의 요청을 서블릿 컨테이너는`HttpServletRequest`로 가공해 서블릿에 전달한다. 이 객체 안에서 요청된 Session ID를 찾을 수 있다. 이 Session ID는 클라이언트의 요청 쿠키에 담겨있거나, 요청 URL에 담길 수 있다.

클라이언트가 Session ID를 쿠키에 담던 URL던, 서버가 요청받은 세션 공간에 접근하고자 한다면 [HttpServletRequest#getSession](https://docs.oracle.com/javaee/6/api/javax/servlet/http/HttpServletRequest.html#getSession())을 호출하면 된다. 

### 쿠키를 이용한 세션 공유
#### 새 세션의 생성
브라우저는 서버로 요청을 보낸다. 서버는 요청 정보에 Session ID가 없는 것을 확인하고, 첫 요청으로 판단해 새 세션 공간을 할당한다. 세션 공간이 생성되면 이 공간에 접근하기 위한 유니크한 세션 ID가 생성된다. 이 공간에 해당 유저의 정보를 저장해둔다. 클라이언트가 이후 요청부터는 세션 ID를 쿠키에 담아 보내도록 하기 위해, 응답 헤더에 `Set-Cookie: JSESSIONID=...`를 담아 응답한다. `Set-Cookie`를 통한 서버의 응답은 HTTP 표준 명세에 따른다.
![new-session](/img/session/new-session.jpg)

#### 세션 정보를 담은 브라우저 요청
[HTTP 표준 명세에 따라, ](https://www.rfc-editor.org/rfc/rfc6265#section-4.2) 클라이언트가 `Set-Cookie` 응답을 받게 되면, 이후 요청은 쿠키를 설정해 요청해야 한다. 따라서, 브라우저는 쿠키에 `JSESSIONID=...`를 담아 요청하게 된다. 서버는 쿠키에서 세션 ID를 확인하고, 해당 세션 공간에 접근해 저장해두었던 유저의 상태 정보를 확인할 수 있다. 
![req-with-session](/img/session/req-with-session.jpg)

#### 만료된 세션에 대한 요청
세션 공간이 만료되어 더이상 해당 세션 정보를 찾을 수 없게 된다. 클라이언트는 더이상 유효하지 않은 세션 ID를 쿠키에 담아 보내면, 서버는 해당 세션 ID는 세션 공간에 저장되지 않았다고 판단한다. 따라서, 새 세션을 만드는 과정을 반복한다. 
![renew-session](/img/session/renew-session.jpg)


### URL rewriting
쿠키를 사용할 수 없는 브라우저에서는 세션 정보를 쿠키로 교환할 수 없게 된다. 다시 말하면, 클라이언트 쿠키에서 Session ID를 찾을 수 없기 때문에, 서버는 매번 새로운 요청으로 판단하게 될 것이다. 

이러한 문제를 해결하기 위한 방법이 URL rewriting이다. 서버는 요청받은 URL에 해당 세션 ID를 추가해 redirect를 유도한다. 
`http://foo.bar.com;jsessionid=...`의 형식으로 URL을 새로 만드는 것이다. 

URL rewriting을 위해 [HttpServletResponse#encodeURL](https://docs.oracle.com/javaee/6/api/javax/servlet/http/HttpServletResponse.html#encodeURL(java.lang.String)) 메서드를 제공한다. 아래의 코드는 쿠키를 사용할 수 없는 클라이언트 환경을 위해 rewritten-url을 제공하고, URL로 Session ID를 식별해 방문 횟수를 카운트하는 예이다. 

```java
@Override
protected void doGet(HttpServletRequest req, HttpServletResponse res) throws ServletException, IOException {
    final String VISIT_COUNT = "visit_count";

    HttpSession session = req.getSession(false);
    if (session == null) {
        session = req.getSession(true);
        session.setAttribute(VISIT_COUNT, 1);
    } else {
        Integer count = (Integer) session.getAttribute(VISIT_COUNT);
        session.setAttribute(VISIT_COUNT, count + 1);
    }

    Integer count = (Integer) session.getAttribute(VISIT_COUNT);
    String encodedUrl = res.encodeURL(HttpUtils.getRequestURL(req).toString());

    res.setContentType("text/html");
    PrintWriter out = res.getWriter();
    out.println("<a> count = " + count + " </a>");
    out.println("<br>");
    out.println("<a> your own url is " + encodedUrl + " </a>");
    out.close();
}
```

## 분산 환경에서의 세션 불일치
서버가 scaled-out 된 상황 등, 분산 환경에서는 `세션 불일치` 문제가 발생할 수 있다.

Session scope는 서블릿 컨텍스트 단위로 유효하므로, 한 애플리케이션은 다른 애플리케이션의 세션 공간에 접근할 수 없다.

아래 그림은 요청을 로드 밸런서가 받아 3개의 서버 인스턴스로 중계하는 상황을 표현한다. 이전 요청은 Server 1에서 서비스되었기 때문에, Server 1의 세션 공간에는 `abc`라는 세션 ID를 발급한 후, `{"userid":"a"}`라는 사용자 정보를 저장했다. 이후 다음 요청은 로드 밸런서에 의해 Server 2로 중계되었지만, Server 2는 이 사용자에 대한 세션 정보를 가지고 있지 않다. 

![session-inconsistency](/img/session/session-inconsistency.jpg)

### Sticky sessions
스티키 세션은 서버가 세션을 생성한 후, Set-Cookie 헤더에 서버의 식별자를 넣어 응답한다. 이후 클라이언트는 서버 식별자를 쿠키에 담아 요청하게 된다. 로드밸런서는 쿠키에 서버 식별자가 있다면 이에 대응하는 서버 인스턴스로 요청을 중계한다. 

아래 그림은 스티키 세션의 동작을 보여준다. 로드밸런서는 쿠키의 `server=1`을 확인하고 Server 1로 요청을 중계한다.
![sticky-session](/img/session/sticky-session.jpg)

하지만 스티키 세션은 HA라는 scale-out의 장점을 십분 활용하기 어려운 방식이다.
- 특정 서버에 장애가 생겼을 경우, 로드밸런서는 나머지 서버로 요청을 중계해야 한다. 하지만 어떠한 서버도 장애가 발생한 서버의 세션 데이터를 가지고 있지 않다.
- 로드밸런서의 중계 알고리즘이 트래픽 분산에 효과적이지 못할 수 있다. 단지 요청 쿠키의 서버 식별자로 중계하기 때문에, 특정 서버에만 과도하게 요청이 몰리는 상황이 발생할 수 있다.

### Session clustering
다음 그림에서, Server 1의 세션 공간에는 `abc`라는 세션 ID를 발급한 후, `{"userid":"a"}`라는 사용자 정보를 저장했다. 이 정보는 Server 2, Server 3으로 복제된다. 이러한 방식을 세션 클러스터링이라 한다.

![session-clustering](/img/session/session-clustering.jpg)

세션 클러스터링은 스티키 세션과 달리, 부하 분산 측면에서 유리하다. 모든 서버가 세션 정보의 정합성을 동기화하고 있기 때문에, 로드밸런서는 트래픽 분산을 위한 중계를 할 수 있다.
하지만 세션 클러스터링에도 문제점이 있다.
- 타이밍 이슈로 인해 정합성이 100% 맞지 않을 수 있다. 예를 들면, 한 서버의 세션 정보 변경이 다른 서버로 동기화되기 전에, 같은 브라우저의 새 요청이 들어오는 경우가 있다.
- 다른 서버의 모든 세션 정보를 가지고 있어야 하므로, 세션을 위한 메모리 요구량이 증가한다.
- 세션 동기화로 인한 네트워크 트래픽이 발생한다.

### Session storage 분리
각 서버의 세션 저장소를 외부로 분리하는 방법이다. 상태를 저장소에서 관리하고, 서버는 필요 시 저장소에서 데이터를 조회하고 가져온다. 따라서 트래픽에 따른 로드밸런싱이 가능해진다. 

![session-storage](/img/session/session-storage.jpg)

세션 스토리지 분리 방법을 선택할 때는 어떤 저장소가 적합할 지 고려해야한다.
- 가령, 세션 스토리지로 RDB를 사용하는 것은 매 요청마다 세션 정보 조회를 위한  DB 접근이 발생하게 됨을 의미한다. 세션 정보의 영속화보다 빠른 접근시간을 우선한다면 인메모리 key-value store가 적합할 수 있다.
- 스토리지가 SPoF가 될 수 있다. 스토리지에 장애가 생기면 이를 바라보는 모든 서버는 세션 정보에 접근이 불가능하게 된다. 스토리지의 replication 용이성이 선택 기준이 될 수 있다.