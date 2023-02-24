## Connection
애플리케이션과 DB와의 연결을 의미한다. 연결 정보를 담는 `java.sql.Connection` 객체로 표현된다. 

![get-connection](/img/connection-pool/get-connection.gif)

위의 그림은 Connection을 생성하고 해제하는 과정을 표현한다. 애플리케이션과 DB 간의 TCP 연결, DB 계정 인증, Connection 객체의 할당 등의 절차를 수행하기 때문에, Connection의 생성은 많은 비용이 드는 작업이다. 애플리케이션이 DB를 사용하기 위해 매번 Connection을 생성하게 되면 여러 문제가 있다. 프로그래머가 직접 생성/해제를 신경써주어야 하는 것은 물론, 매 요청마다 Connection 생성 비용이 발생한다. 또한, 동시 요청이 급증하게 된다면 Connection의 수도 같이 증가하게 되므로, 서버의 리소스가 빠르게 고갈될 수도 있다. 이러한 이유에서 Connection Pool을 사용하게 된다.

## Connection Pool
WAS는 초기화 과정에서 일정 개수의 Connection을 미리 할당해두는데, 이들을 저장해둔 곳을 Connection pool이라 한다. 

![connection-pool](/img/connection-pool/connection-pool.gif)

WAS의 스레드는 DB의 커넥션을 이용해야 할 때마다 Connection pool에서 Connection을 이용한 후, 다시 반납한다. 이로써 Connection 생성 비용을 매번 지불하지 않아도 되며, Connection pool size로 최대 Connection 수를 제한할 수 있다.

## 적절한 Connection pool size?
[HikariCP는 적절한 Connection pool size를 튜닝하기 위한 시작점으로, `(DB 서버의 물리 코어 수 * 2) + DB 서버의 HDD 수)`를 제시한다.](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing) 이 수식은 Connection이 너무 적거나 크면 최적화되기 어렵다는 기본 가정에서 출발해, 경험적으로 얻은 수식이다. 

DB 서버는 `DB 서버의 HDD 수` 만큼의 동시 I/O 접근을 할 수 있으며, `DB 서버의 물리 코어 수` 만큼의 쿼리를 동시에 처리할 수 있다고 가정하자. DB 서버가 가진 리소스에 비해 너무 적은 Connection을 가지면, 리소스를 충분히 활용하지 못하게 된다. 리소스를 충분히 활용하려면 디스크에 접근하느라 I/O 대기 상태에 빠진 스레드 대신 CPU 연산을 필요로 하는 다른 쿼리를 코어가 처리할 수 있도록 컨텍스트 스위칭 해주어야 한다. 이를 위해 코어 수에 `* 2`라는 term을 추가했다. 리소스를 충분히 활용하려는 의도로 너무 많은 Connection을 생성하게 되면, 쿼리를 처리하는 도중 컨텍스트 스위칭이 더 빈번하게 일어나게 되어 디스패치 오버헤드가 증가한다. 제안한 수식은 SSD를 사용하는 환경에서의 경향성을 설명해주지 않기 때문에, 성능 테스트를 통해 적절한 pool size를 도출해야 한다.

> You want a small pool of a few dozen connections at most, and you want the rest of the application threads blocked on the pool awaiting connections.

## 같이 보면 좋은 글
- [postgresql: pool size로 인한 성능 저하 이유](https://wiki.postgresql.org/wiki/Number_Of_Database_Connections)