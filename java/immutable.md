### Immutable Object를 사용하는 이유

Immutable object의 정의는 단순하다. ‘한번 생성된 후에는 상태를 바꿀 수 없는 객체’를 의미한다. 상태가 변하지 않기 때문에 사용하기 간단하고, 스레드 경합으로 인한 부작용을 걱정하지 않아도 된다. 이로써 얻을 수 있는 장점은 ‘공유하기 쉽다’는 것이다. 얼핏 들으면 쉽게 이해가 가지 않지만, BigInteger 클래스는 ‘공유’라는 측면을 잘 사용하는 예 중 하나이다.

BigInteger는 자주 사용되는 일부 값들을 미리 static으로 정의해 공유한다. BigInteger는 immutable하므로, 여러 스레드가 동시에 ZERO, ONE, …에 접근하더라도 상태가 바뀌지 않음이 보장된다. 

```java
public static final BigInteger ZERO = new BigInteger(new int[0], 0);
public static final BigInteger ONE = valueOf(1);
public static final BigInteger TWO = valueOf(2);
...
```

Immutable object의 단점은 같은 값을 같는 객체가 중복으로 생성될 수 있다는 점이다. 이 점은 캐싱을 활용해 어느정도 보완할 수 있다. (flyweight pattern, interning 등)
예를 들면, Integer.valueOf()는 -128~127 범위의 객체 생성 요청이 들어오면, 내부 캐시에서 해당 값을 찾아 리턴한다.
하지만, 1M-bit 데이터 중 일부 비트만 바꾸고 싶은 경우 등의 상황에는, mutable object를 다루는 데 비해 시간/공간 비용이 커지게 된다.

### Immutable Object를 위한 가이드라인
1. setter를 제공하지 않음으로써 임의 수정을 막는다.
2. 필드를 private final로 선언해 재할당을 금지시킨다.
3. 서브클래싱을 못하도록 final class로 선언 혹은 생성자를 private으로 막고 별도의 factory를 제공한다.
    - 단, final 대신 private 생성자의 경우, inner class modifier로 상태를 변경시킬 수 있음.
4. Mutable obect를 필드로 가지고 있다면, mutable object에 대한 레퍼런스를 공유하지 말고 defensive copy를 활용한다.
    - 생성자에서 mutable object의 레퍼런스를 받을 때, 그 레퍼런스의 복사본을 만들어 참조하게끔 만든다.
    - 메서드에서 mutable object를 리턴할 때 복사본을 만들어 넘겨주도록 한다.

[위의 체크리스트를 모두 만족해야만 Immutable object라고 할 수 있는 것은 아니다. 인스턴스가 생성된 후 불변으로 유지될 수 있다면 immutable이라 할 수 있다.](<https://docs.oracle.com/javase/tutorial/essential/concurrency/imstrat.html>)

> Not all classes documented as "immutable" follow these rules. This does not necessarily mean the creators of these classes were sloppy — they may have good reason for believing that instances of their classes never change after construction.

### Immutable한 클래스를 만들어보자

위의 원칙대로 Immutable class Money를 만들어보자. 

필드 int는 primitive type이고, String은 Immutable한 클래스이다. IssuedDate는 Mutable한 클래스라고 가정하자.

private final을 필드에 선언함으로써, 객체의 필드 재할당과 상태 변경이 불가능해지므로, setter를 제공할 수 없다. 또한, 생성자 내부에서 IssuedDate의 defensive copy로 필드를 설정해준다. 생성자 액세스레벨은 private으로 확장할 수 없도록 만든다.

```java
public class Money {

    // 1), 2) : private로 접근 제한, final로 재할당 제한 (setter 금지)
    private final int amount;
    private final String currency;
    private final IssuedDate issuedDate;

    // 3) private 생성자로 서브클래싱 금지
    // 4) passed reference의 defensive copy
    private Money(int amount, String currency,IssuedDate issuedDate) {
        this.amount = amount;
        this.currency = currency;
        this.issuedDate = new IssuedDate(issuedDate.getYear(), issuedDate.getMonth(), issuedDate.getDay());
    }

    // 3) 대신, static factory로 객체 생성 방법 제공
    public static Money of(int amount, String currency, IssuedDate issuedDate) {
        return new Money(amount, currency, issuedDate);
    }

    // 연산의 결과는 객체 상태를 변경하는 대신 새 객체를 제공
    public Money plus(Money money) {
        if (!currency.equals(money.getCurrency())) {
            throw new RuntimeException();
        }
        return of(amount + money.getAmount(), currency, issuedDate);
    }

    public int getAmount() {
        return amount;
    }

    public String getCurrency() {
        return currency;
    }

    // 4) returns copy of this.issuedDate
    public IssuedDate getIssuedDate() {
        return new IssuedDate(issuedDate.getYear(), issuedDate.getMonth(), issuedDate.getDay());
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;

        Money that = (Money) o;

        if (amount != that.amount) return false;
        return currency.equals(that.currency);
    }

    @Override
    public int hashCode() {
        int result = amount;
        result = 31 * result + currency.hashCode();
        return result;
    }
}
```

### 멀티스레드 상황에서 Immutable object의 상태 불변성

아래 테스트 코드를 실행하면, 여러 스레드가 동시에 oneDollar 객체에 접근해도, plus 메서드의 결과는 2를 가지게 된다. plus()는 객체의 상태를 변화시키는 대신 새로운 객체를 만들어 리턴하기 때문이다.

```java
@Test
@DisplayName("thread-safe though threads access to same 'oneDollar'")
void addMoney() throws InterruptedException {
    int numThreads = 100000;
    List<Thread> threads = new ArrayList<>();

    IssuedDate issuedDate = new IssuedDate(2023, 1, 1);

    Money oneDollar = Money.of(1, "USD", issuedDate);
    for (int i = 0; i < numThreads; i++) {
        threads.add(new Thread(() -> plusAndValidate(oneDollar)));
    }
    for (Thread thread : threads) {
        thread.start();
    }

    for (Thread thread : threads) {
        thread.join();
    }
}

void plusAndValidate(Money oneDollar) {
    IssuedDate issuedDate = new IssuedDate(2023, 1, 1);
    Money usd = oneDollar.plus(Money.of(1, "USD", issuedDate));
    assertThat(usd).isEqualTo(Money.of(2, "USD", issuedDate));
}
```

### Mutable reference field를 가지고 있는 경우

Money의 필드로 가지고 있는 IssuedDate를 final 키워드로 재할당을 금지했다. 이는 isseudDate 필드가 다른 레퍼런스를 가리키지 않도록 한다.

하지만 IssuedDate가 Mutable하다면 문제가 생길 수 있다. getIssuedDate()의 구현이 `return this.issuedDate;`라면 필드로 잡는 레퍼런스를 직접 넘겨주게 된다. 클라이언트는 이 레퍼런스에 접근해 값을 조작할 수 있게 되므로, immutability 보장이 되지 않는다. 예를 들면, 아래의 테스트 코드가 실패한다.

```java
public class IssuedDate {
    private int year;
    private int month;
    private int day;

    public IssuedDate(int year, int month, int day) {
        this.year = year;
        this.month = month;
        this.day = day;
    }
    
    // getters and setters below
    ...
}

@Test
void getIssuedAt() {
    IssuedDate issuedDate = new IssuedDate(2023, 1, 1);
    Money oneDollar = Money.of(1, "USD", issuedDate);
    oneDollar.getIssuedDate().setYear(2099);

    assertThat(oneDollar.getIssuedDate().getYear()).isEqualTo(2023);
}
```

필드로 잡고 있는 Mutable object를 defensive-copy하여 전달해준 이유이다.. 즉, Money가 immutable하려면, String처럼 IssuedDate가 불변성을 보장하던지, Money는 IssuedDate의 레퍼런스가 아닌 사본을 만들어 리턴해야 한다.