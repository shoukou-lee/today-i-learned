### Functional interface의 사용법
아래와 같이, `invoke()`라는 Single Abstract Method (SAM)을 가진 인터페이스가 있다. 인터페이스는 직접 instantiate할 수 없고, 인터페이스를 구현한 구체 클래스를 instantiate해야 한다.
```java
@FunctionalInterface
public interface SimpleFunctionalInterface {
    String invoke();
}
```
하지만 특정 메서드 내부에서 인터페이스를 직접 구현할 수 있다. 이러한 클래스를 익명 클래스라고 한다.
```java
    @Test
    void functionalInterfaceUsage() {
        String given = "hello";

        SimpleFunctionalInterface byAnonClass = new SimpleFunctionalInterface() {
            @Override
            public String invoke() {
                System.out.println(given);
                return given;
            }
        };
        String result = invoke(byAnonClass);
        assertThat(result).isEqualTo(given);
    }

    String invoke(SimpleFunctionalInterface simpleFunctionalInterface) {
        return simpleFunctionalInterface.invoke();
    }
```
이후, 자바에서 Lambda expression을 지원하면서, 이와 같은 익명 클래스를 간략히 표현할 수 있게 되었다. 아래는 위의 익명 클래스를 Lambda expression으로 대치한 코드이다. `(SAM의 인자들) -> { 메서드 구현 }` 의 형태를 가진다.
```java
    @Test
    void functionalInterfaceUsage() {
        String given = "hello";

        SimpleFunctionalInterface byLambdaExp = () -> {
            System.out.println(given);
            return given;
        };

        String result = invoke(byLambdaExp);
        assertThat(result).isEqualTo(given);
    }

    String invoke(SimpleFunctionalInterface simpleFunctionalInterface) {
        return simpleFunctionalInterface.invoke();
    }
```

Lambda expression의 장점은 간결하게 표현할 수 있다는 점이다. 이 점을 더욱 활용하기 위해 람다 구현부를 메서드 추출하면 더욱 가독성 있는 코드로 만들 수 있다.
```java
    @Test
    void functionalInterfaceUsage() {
        String given = "hello";

        SimpleFunctionalInterface byLambdaExp = () -> printAndReturn(given);
        String result = invoke(byLambdaExp);
        assertThat(result).isEqualTo(given);
    }

    String invoke(SimpleFunctionalInterface simpleFunctionalInterface) {
        return simpleFunctionalInterface.invoke();
    }

    String printAndReturn(String str) {
        System.out.println(str);
        return str;
    }
```

## Java8 Functional Interface
Java8에 추가된 Functional Interface 중, 특정 타입이나 인자 개수에 따라 다양한 Functional Interface가 추가되었지만, 가장 큰 갈래로 나누면 Supplier, Consumer, Predicate, Function로 분류할 수 있다. 이 각각에 대해 살펴보자. 

### Supplier
`Supplier`는 이 인터페이스의 구현이 결과의 공급자 역할을 한다는 의미를 가지는 Functional interface이다. default method 없이 내부적으로 get() 이라는 SAM (Single Abstract Method)를 가지고 있다. get()은 인자가 없기 때문에, method reference를 통해 함수를 전달해줄 수도 있다. 콜백 등 특정 함수의 실행을 뒤로 미루고 싶은 경우에 주로 사용한다. Supplier의 사용법은 앞서 살펴 본 예제인 SimpleFunctionalInterface와 유사하다. (Supplier가 Generic return을 사용한다는 점을 제외하면)

```java
@FunctionalInterface
public interface Supplier<T> {

    /** Gets a result. */
    T get();
}
```

### Consumer
`Consumer`는 하나의 인자를 받아 처리하고, 리턴을 만들지 않는 Functional interface이다. `andThen` 메서드는 컨슈머를 먼저 실행하고, 인자로 받은 `after` 컨슈머를 순차적으로 실행하도록 구성된 새 컨슈머를 리턴한다. 
```java
@FunctionalInterface
public interface Consumer<T> {
    // 사용자 구현 필요
    void accept(T t);

    // 해당 Consumer를 먼저 실행한 후, 인자로 받은 Consumer를 실행하는 새 Consumer를 리턴한다.,
    default Consumer<T> andThen(Consumer<? super T> after) {
        Objects.requireNonNull(after);
        return (T t) -> { accept(t); after.accept(t); };
    }
}
```
`Consumer`를 적용한 예 중 대표적으로 `forEach()` 메서드가 있다. `forEach()`는 i-th index에 위치한 원소에 대한 처리 구현을 인자로 받는다. 인덱스 i로부터 해당 원소를 찾아, 사용자가 구현한 `accept()`를 실행하도록 구현되어있다.
```java
    // ArrayList.java
    @Override
    public void forEach(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        final int expectedModCount = modCount;
        final Object[] es = elementData;
        final int size = this.size;
        for (int i = 0; modCount == expectedModCount && i < size; i++)
            action.accept(elementAt(es, i));
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
```
```java
    @Test
    void consumerUsage() {
        List<String> strings = Arrays.asList("1", "2", "3", "4");
        strings.forEach(i -> System.out.println("i = " + i));
    }
```

### Predicate
하나의 인자를 받고, 평가 결과를 boolean으로 리턴하는 함수를 지원하기 위한 인터페이스이다. 특정 조건과 일치하는 결과를 필터링하기 위한 목적으로 사용될 수 있다. 예를 들면, `Stream.filter()`는 `Stream<T> filter(Predicate<? super T> predicate)`로 선언되어 있고, 인자로 `Predicate`를 받는다.
```java
@FunctionalInterface
public interface Predicate<T> {

    // 인자 T에 대한 평가 결과를 boolean으로 리턴하는 사용자 구현 필요
    boolean test(T t);

    // and(), or(), negate(), isEquals() belows
}
```
```java
    @Test
    void predicateUsage() {
        List<String> strings = Arrays.asList("1", "2", "3", "4");
        Predicate<String> predicate = s -> s.equals("2");
        List<String> result = strings.stream()
                .filter(predicate)
                .collect(Collectors.toList());
        assertThat(result).hasSize(1);
        assertThat(result.get(0)).isEqualTo("2");
    }
```

### Function
T로 R을 리턴하는 함수를 인자로 받는 인터페이스이다. Stream에 익숙하다면, 원소에 적용할 행동을 Stream.map()의 인자로 넘겨준 적이 있을 것이다.
`<R> Stream<R> map(Function<? super T, ? extends R> mapper)`의 인자로 Function\<T, R\>을 받고 있기 때문이다.
```java
@FunctionalInterface
public interface Function<T, R> {
    // 주어진 인자 t로 리턴 R을 만드는 메서드 구현 필요
    R apply(T t);
    // ...
}
```
정수 ArrayList의 각 원소를 Stringify하는 function을 정의하고, stream()을 실행하는 코드이다. 
```java
    @Test
    void functionUsage() {
        List<Integer> integers = Arrays.asList(1, 2, 3, 4);
        Function<Integer, String> function = s -> s.toString();
        List<String> result = integers.stream()
                .map(function)
                .collect(Collectors.toList());

        for (int i = 0; i < result.size(); i++) {
            assertThat(result.get(i)).isEqualTo(integers.get(i).toString());
        }
    }
```