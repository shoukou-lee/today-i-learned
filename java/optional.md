## Optional을 왜 사용할까?
`NullPointerException`은 `null`에 접근해 사용하려는 시도에서 발생하는 예외이다. 만약 Nullable한 값을 리턴하는 메서드가 있다면, 클라이언트는 항상 null check와 예외 처리에 유념해야 한다. `NullPointerException`이 발생할 수 있는 구문을 일일히 확인하거나, 예외 처리 로직을 작성하는 것은 클라이언트 입장에서 매우 어려운 일이다.
Optional\<T\>는 'null이 리턴될 수 있음'을 클라이언트에게 명확하게 알려주는 효과가 있다. 또한, Optional이 담고 있는 값이 null일 경우에 대한 처리 로직을 간결하게 작성할 수 있게 해주는 메서드를 제공한다.

## Optional\<T\>을 살펴보자
Optional\<T\>의 생성과 관련된 코드이다. 내부적으로 `T value`를 홀드하고 있다. Optional\<T\>는 value의 상태를 조작하는 메서드를 제공하지 않는 불변적 성격을 갖는다. value를 defensive copy하면서까지 strict하게 immutability를 보장하지는 않는다. 클라이언트의 책임이다.
```java
public final class Optional<T> {
    
    // Optional[null]을 의미하는 공유 인스턴스
    private static final Optional<?> EMPTY = new Optional<>(null);

    // 패키징될 값
    private final T value;

    // EMPTY를 리턴하는 static factory
    public static<T> Optional<T> empty() {
        @SuppressWarnings("unchecked")
        Optional<T> t = (Optional<T>) EMPTY;
        return t;
    }

    // private 생성자
    private Optional(T value) {
        this.value = value;
    }

    // null을 허용하지 않는 static factory (throw NPE)
    public static <T> Optional<T> of(T value) {
        return new Optional<>(Objects.requireNonNull(value));
    }

    // null을 허용하는 static factory. 
    public static <T> Optional<T> ofNullable(T value) {
        return value == null ? (Optional<T>) EMPTY
                             : new Optional<>(value);
    }
```

잡고 있는 value의 nullity를 검사하는 메서드이다. 
```java
    public boolean isPresent() {
        return value != null;
    }

    public boolean isEmpty() {
        return value == null;
    }
```

내부 value의 값을 리턴하는 메서드이다. `Supplier`는 `@FunctionalInterface`이다.
```java
    // value가 null이면 인자로 받은 함수를 실행한 결과를 리턴한다. 인자로 받은 함수는 Optional<T>를 리턴해야 한다. 
    // 함수가 null이라면 NullPointerException을 던진다.
    public Optional<T> or(Supplier<? extends Optional<? extends T>> supplier) {
        Objects.requireNonNull(supplier);
        if (isPresent()) {
            return this;
        } else {
            Optional<T> r = (Optional<T>) supplier.get();
            return Objects.requireNonNull(r);
        }
    }

    // value가 null이면 인자로 받은 default value를 리턴한다.
    public T orElse(T other) {
        return value != null ? value : other;
    }

    // value가 null이면 인자로 받은 함수의 결과(nullable)를 리턴한다. 함수가 null이라면 NullPointerException을 던진다.
    public T orElseGet(Supplier<? extends T> supplier) {
        return value != null ? value : supplier.get();
    }

    // value가 null이면 NoSuchElementException를 던진다. get()과 구현이 같지만, get()은 더이상 사용하지 않는다.
    public T orElseThrow() {
        if (value == null) {
            throw new NoSuchElementException("No value present");
        }
        return value;
    }

    // value가 null이면 인자로 받은 Exception 생성 함수를 실행하고 던진다. 함수가 없다면 NullPointerException을 던진다.
    public <X extends Throwable> T orElseThrow(Supplier<? extends X> exceptionSupplier) throws X {
        if (value != null) {
            return value;
        } else {
            throw exceptionSupplier.get();
        }
    }
```
\* [get()을 더이상 사용하지 않는 이유](https://stackoverflow.com/questions/26327957/should-java-8-getters-return-optional-type/26328555)
 
내부 value의 nullity에 따라 함수를 호출하는 메서드이다. `Consumer`와 `Runnable`도 마찬가지로 `@FunctionalInterface`이다.
```java
    // value가 null이 아니라면, value를 인자로 받는 action 함수를 수행한다.
    public void ifPresent(Consumer<? super T> action) {
        if (value != null) {
            action.accept(value);
        }
    }

    // 두개의 함수를 인자로 받는다. 각각 null이 아닐 경우, null일 경우 호출한다.
    public void ifPresentOrElse(Consumer<? super T> action, Runnable emptyAction) {
        if (value != null) {
            action.accept(value);
        } else {
            emptyAction.run();
        }
    }
```

[Java8에 도입된 `@FunctionalInterface`인  `Supplier`, `Consumer`, `Predicate`, `Function`에 대한 글은 여기로](/java/java8-functional-interface.md)

## orElse() vs orElseGet()
orElse()와 orElseGet()은 default value을 얻기 위해 미리 정해진 값을 사용할 것인지, `Supplier`의 결과를 사용할 것인지에 대한 차이가 있다. 하지만 이 차이를 이해하지 못하면 원치 않는 메서드 호출이 발생할 수 있다. 예를 들어, 아래와 같이 사용자 이름으로 User 객체를 찾는 `Storage` 클래스가 있다. `Storage`의 목적은 캐시를 사용해 조회 성능을 향상시키고, 클라이언트로부터 캐시 존재를 숨길 목적으로 만들어졌다. `public findUserByName()` 메서드는 캐시를 먼저 읽고(`readFromCache`) cache-miss가 발생하면 DB에 접근(`readFromDB`)해 결과를 리턴할 의도를 구현한 것이다.

하지만 `findUserByName()`을 실행하면, cache-hit이 발생하더라도 DB 조회를 시도한다. orElse()는 value=null일 경우에 리턴을 위한 기본"값"을 되돌려준다. 파라미터로 `readFromDB(name)` 메서드의 실행 결과를 넘겨줬기 때문에, 이 값을 평가하기 위해 메서드가 실행된 것이다.

이를 해결하기 위해서는 `readFromDB` 메서드의 실행을 지연시켜야 한다. orElseGet()에 메서드를 `Supplier` 타입의 인자로 넘겨줌으로써 달성할 수 있다.
```java
public class Storage {

    private final Cache cache;

    private final Database db;

    public Storage(Cache cache, Database db) {
        this.cache = cache;
        this.db = db;
    }

    public User findUserByName(String name) {
        return readFromCache(name)
                // .orElse(readFromDB(name)); -> wrong
                .orElseGet(() -> readFromDB(name)); // readFromDB 호출 지연
    }

    private Optional<User> readFromCache(String name) {
        return cache.find(name);
    }

    private User readFromDB(String name) {
        return db.find(name);
    }
}
```

## Optional\<T\>의 편의성
앞서 설명했듯, Optional\<T\>은 클라이언트에게 명시적으로 nullable함을 알려줌으로써, isPresent()/isEmpty()를 사용한 nullity check 필요성을 리마인드해줄 수 있다. 더욱 중요한 점은, Lambda expression을 활용해 null 조건 처리를 간결하게 만들 수 있다. `@FunctionalInterface`를 인자로 받아 null 조건을 처리하는 메서드를 제공하기 때문이다. 다음은 앞서 설명한 메서드의 용례를 보여주는 예제 코드이다.
```java
public class Person {

    private String name;

    public Person() {

    }
    public Person(String name) {
        this.name = name;
    }

    public Optional<String> getName() {
        return Optional.ofNullable(name);
    }
}
```

```java
    @Test
    void or() {
        Person p = new Person();
        assertThat(doOr(p)).isEqualTo(Optional.of("감자"));
    }

    Optional<String> doOr(Person p) {
        return p.getName()
                .or(() -> Optional.of("감자"));
    }
```
```java
    @Test
    void orElse() {
        Person p = new Person();
        String name = doOrElse(p);
        assertThat(name).isEqualTo("감자");
    }

    String doOrElse(Person p) {
        return p.getName()
                .orElse("감자");
    }
```
```java
    @Test
    void orElseThrow() {
        Person p = new Person();
        assertThatThrownBy(() -> doOrElseThrow(p))
                .isInstanceOf(RuntimeException.class)
                .hasMessage("이름이 없어요");
    }

    String doOrElseThrow(Person p) {
        return p.getName()
                .orElseThrow(() -> new RuntimeException("이름이 없어요"));
    }
```
```java
    @Test
    void ifPresent() {
        Person p = new Person("감자");
        p.getName()
                .ifPresent(name -> System.out.println("name = " + name));
    }

    @Test
    void ifPresentOrElse() {
        Person p = new Person();
        p.getName()
                .ifPresentOrElse(
                        name -> System.out.println("name = " + name),
                        () -> System.out.println("이름이 없어요"));
    }
```

앞서 설명했듯, Optional\<T\>은 클라이언트에게 명시적으로 nullable함을 알려줌으로써, isPresent()/isEmpty()를 사용한 nullity check 필요성을 리마인드해줄 수 있다. 더욱 중요한 점은, Lambda expression을 활용해 null 조건 처리를 간결하게 만들 수 있다. `@FunctionalInterface`를 인자로 받아 null 조건을 처리하는 메서드를 제공하기 때문이다. 다음은 앞서 설명한 메서드의 용례를 보여주는 예제 코드이다.

## Optional\<T\>을 잘 활용하기 위해
Optional은 nullable한 리턴을 대응하기 위한 목적으로 설계되었다. Optional을 올바르게 사용하기 위해 [지켜야 할 점](https://dzone.com/articles/using-optional-correctly-is-not-optional)이 많고, 지키지 못해 생기는 심각한 부작용이 있다. 또한, 잠재적인 [성능 문제](https://pkolaczk.github.io/overhead-of-optional/)가 존재한다.

- Optional에 null을 할당하지 말 것
    - Optional 또한 reference type의 box이다. null이 대입될 수 있다.
- 리턴 타입으로만 사용할 것
    - 필드에 Optional을 사용하지 말 것. Optional은 Serializable을 구현하지 않는다.
    - 파라미터에 Optional을 넘겨주지 말 것. Optional parameter가 null일 수 있다.
- nullity check 대신 orElseXXX로 보일러플레이트를 줄일 것
- 과용하지 말 것. Optional.orElse()을 getter처럼 쓰는 것이 대표적인 예이다.
- 컬렉션 사용에 유의할 것
    - 컬렉션을 Optional로 감싸지 말 것. 빈 컬렉션을 반환하는 더 좋은 대안이 있다.
    - 컬렉션에 Optional을 집어넣지 말 것. getOrDefault()라는 더 좋은 대안이 있다.
- Equality에 유의할 것
    - Equality 검사를 위해 Optional unboxing 하지 말 것. `equals()` 구현을 확인해보면 알 수 있다.
    - `==`보다 `equals()`를 사용할 것.

