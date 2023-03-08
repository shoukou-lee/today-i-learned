## 서블릿이란?
동적 컨텐츠를 생성하기 위한 server-side application이다. 주로 웹 상에서 클라이언트-서버 간 요청/응답 패러다임의 통신을 구현하기 위해 사용한다.
동적 컨텐츠란, 요청 정보에 따라 다른 응답을 내려줄 수 있는 컨텐츠이다. 

## 서블릿의 라이프사이클
서블릿은 '서블릿 컨테이너'가 라이프사이클을 관리한다. 서블릿의 라이프사이클을 크게 `init`, `service`, `destroy`로 나눌 수 있으며, 이는 `jakarta.servlet.Servlet` 인터페이스의 API이기도 하다. 
즉, 서블릿을 구현하는 것은 서블릿 라이프사이클에 대한 정의를 하는 것과 같다.  

### 로딩 및 인스턴스화
로딩 및 인스턴스화 시점은 서블릿 컨테이너에 의해 결정된다. 컨테이너 부트스트랩 과정에서 로딩될 수도 있고, 클라이언트 요청 시점까지 지연될 수 있다. 
서블릿 또한 자바 클래스 중 하나이기 때문에, 서블릿 로딩 또한 자바 클래스 로딩과 같은 절차를 거친다.

### 초기화
인스턴스화된 서블릿은 사용자의 요청을 처리하기 전에 초기화되어야 한다. 컨테이너는 `init()` 메서드를 호출함으로써 서블릿을 초기화한다. 
`init()`의 인자로 `ServletConfig`를 받는데, 이는 초기화 과정에서 서블릿 컨테이너로부터 주입되는 정보이다. 

`GenericServlet`의 구현을 살펴보자. 서블릿 컨테이너가 서블릿을 초기화하기 위해, `init(ServletConfig config)`을 호출하면, 파라미터를 서블릿의 config에 바인딩한다. 
이후 `init()`을 호출한다. 이 `init()`의 구현은 No-op이며, 일반적으로 사용자가 override해서 초기화 로직을 구현하게 된다.

```java
    /**
     * Called by the servlet container to indicate to a servlet that the servlet is being placed into service. See
     * {@link Servlet#init}.
     *
     * <p>
     * This implementation stores the {@link ServletConfig} object it receives from the servlet container for later use.
     * When overriding this form of the method, call <code>super.init(config)</code>.
     *
     * @param config the <code>ServletConfig</code> object that contains configuration information for this servlet
     *
     * @exception ServletException if an exception occurs that interrupts the servlet's normal operation
     * 
     * @see UnavailableException
     */
    @Override
    public void init(ServletConfig config) throws ServletException {
        this.config = config;
        this.init();
    }

    /**
     * A convenience method which can be overridden so that there's no need to call <code>super.init(config)</code>.
     *
     * <p>
     * Instead of overriding {@link #init(ServletConfig)}, simply override this method and it will be called by
     * <code>GenericServlet.init(ServletConfig config)</code>. The <code>ServletConfig</code> object can still be retrieved
     * via {@link #getServletConfig}.
     *
     * @exception ServletException if an exception occurs that interrupts the servlet's normal operation
     */
    public void init() throws ServletException {

    }
```

서블릿의 Constructor로부터 인스턴스가 생성된 상태와 서블릿의 `init()`이 호출되어 초기화 된 상태의 가장 큰 차이는 `ServletConfig`의 바인딩 유무로 볼 수 있다.
서블릿은 `ServletConfig#getServletContext`를 통해 서블릿 컨텍스트에 접근할 수 있다. 다시 말하면, 단순히 생성된 객체 상태의 서블릿과 초기화된 서블릿은 서블릿 컨텍스트에 접근이 가능한가, 그렇지 않은가의 차이를 가진다.

예를 들면, 서블릿의 `init(ServletConfig config)` 메서드 구현에서, config를 바인딩하지 않는다면 이 서블릿의 `this.config = null`이 된다. 
`getServletContext()`의 결과는 NullPointerException이 발생하므로, 서블릿 컨텍스트를 사용할 수 없게 된다.
```java
@WebServlet(name = "customServlet", value = "/custom")
public class CustomServlet extends HttpServlet {
    private String message;

    @Override
    public void init(ServletConfig config) throws ServletException {
        // super.init(config); // let config not bounded
        init();
    }

    @Override
    public void init() {
        System.out.println("do nothing");
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        ServletContext servletContext = getServletContext(); // null-pointer exception
    }
    ...
}
```

### 요청 처리
서블릿의 `service()`메서드를 실행하여 클라이언트의 요청을 처리하는 과정이다. `ServletRequest`, `ServletResponse`를 service()의 인자로 받는다.  
`ServletRequest` 객체는 클라이언트의 요청 정보를 표현하는 객체이며, 클라이언트의 주소 정보나 컨텐츠, 메타데이터 등을 담고 있다.
서블릿은 이 요청에 대해 `ServletResponse` 객체에 응답 결과를 채워 되돌려준다.

### 종료
서블릿 컨테이너는 서블릿의 `destroy()` 메서드를 호출하여 서블릿을 제거할 수 있다. 만약 서블릿이 제거된 후 다시 필요하다면 새로운 서블릿 객체를 생성해야 한다. 
`destory()` 후에는 GC를 위해 서블릿의 참조를 해제해야 한다. 서블릿의 제거 시점은 컨테이너가 결정할 수 있다. 


## 서블릿 인터페이스와 서블릿의 구현
`jakarta.servlet.Servlet` 인터페이스는 모든 서블릿이 구현해야 하는 최상위 인터페이스이다. 서블릿 라이프사이클과 관련된 메서드와 서블릿 정보에 대한 getter 메서드를 선언했다.
```java
public interface Servlet {

    public void init(ServletConfig config) throws ServletException;
    
    public ServletConfig getServletConfig();
    
    public void service(ServletRequest req, ServletResponse res) throws ServletException, IOException;
    
    public String getServletInfo();
    
    public void destroy();
}
```

일반적으로 HTTP 요청/응답을 처리하는 서블릿은 `HttpServlet` 추상 클래스를 구현한다. `HttpServlet`은 `service()` 메서드에 대한 오버라이드를 정의하고 있다.
ServletRequest/Response를 HttpServletRequest/Response로 캐스팅한 후 내부의 service() 메서드에 전달한다.
```java
public abstract class HttpServlet extends GenericServlet {
    
    ...
    
    @Override
    public void service(ServletRequest req, ServletResponse res) throws ServletException, IOException {
        HttpServletRequest request;
        HttpServletResponse response;

        if (!(req instanceof HttpServletRequest && res instanceof HttpServletResponse)) {
            throw new ServletException("non-HTTP request or response");
        }

        request = (HttpServletRequest) req;
        response = (HttpServletResponse) res;

        service(request, response);
    }
```

내부 `service()` 메서드는 클라이언트 요청 정보에서 HTTP method 정보를 찾는다. 이후, 요청 메서드에 대응하는 `doXXX` 메서드를 실행한다. 
'do-prefix'에서 예상하듯 템플릿 메서드 패턴을 적용한 예이다.
```java
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        String method = req.getMethod();

        if (method.equals(METHOD_GET)) {   
            ...
            doGet(req, resp);
            ...
        } else if (method.equals(METHOD_HEAD)) {
            ...
            doHead(req, resp);
        } else if (method.equals(METHOD_POST)) {
            doPost(req, resp);
        } ... 
    }
```

HttpServlet은 추상 클래스이며, doXXX 메서드는 에러 응답에 대한 디폴트 구현이 정의되어 있다. 서블릿 개발자는 이 `doXXX` 메서드에서 실행할 로직을 직접 구현해야 한다. 
```java
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        String protocol = req.getProtocol();
        String msg = lStrings.getString("http.method_get_not_supported");
        resp.sendError(getMethodNotSupportedCode(protocol), msg);
    }
    
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        String protocol = req.getProtocol();
        String msg = lStrings.getString("http.method_post_not_supported");
        resp.sendError(getMethodNotSupportedCode(protocol), msg);
    }
```

