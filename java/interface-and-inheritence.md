# 인터페이스 vs. 상속
인터페이스와 상속은 '서브클래스의 재정의'가 가능하다는 측면에서 유사성을 가지고 있다. 특히, 자바의 `interface`와 `abstract class`의 공통된 특징인 '서브클래스로의 메서드 구현 책임 일임'이라는 점을 갖기 때문에 혼동하기 쉽다. 하지만 이 둘은 엄연히 다른 철학을 바탕으로 만들어졌기 때문에, 상황과 의미에 맞게 적절히 구분되어야 한다.

먼저, 이 두 키워드의 배경적 차이를 살펴보자. 이에 집중하기 위해, Java의 문법적 키워드는 잠시 무시한다. 

## 인터페이스와 상속의 배경적 차이
### 인터페이스
인터페이스는 소프트웨어가 상호 작용하는 방식에 대한 '계약 사항'을 구체적으로 명시한 것이다. 현실 세계에는 다양한 소프트웨어 그룹이 있으며, 한 그룹은 다른 그룹의 기능을 필요로 하는 경우가 있다. 이때, 특정 그룹의 기술이 구현 완료 되었는지 혹은 내부적으로 어떻게 구현되었는지 모르더라도, '계약'을 바탕으로 소프트웨어 개발을 진행할 수 있다. 즉, 인터페이스는 다른 소프트웨어 그룹에게 특정 기술의 표준 명세와, 특정 기능을 호출하기 위한 방법을 설명하기 위한 목적을 가진다. 따라서 인터페이스의 구현체는 명세를 빠짐없이 구현해야 한다.

쉬운 예로 JPA를 생각해보자. javax.persistence.EntityManager는 JPA의 엔티티매니저를 호출하기 위해 제공하는 인터페이스이다. JPA의 벤더 중 하나인 Hibernate는 EntityManager를 실제로 구현한 SessionImpl을 제공한다. JPA를 사용하는 클라이언트는 Hibernate의 내부 구현이 어떻게 이루어졌는지 몰라도, EntityManager의 명세를 통해 JPA 기능을 사용할 수 있다.

### 상속
상속은 중복된 기능을 공통화하고 재사용성을 증대시키기 위한 목적으로 사용된다. 새로운 클래스를 정의할 때, 필요한 특정 필드나 메서드가 다른 클래스에 이미 정의되어있다면, 이를 새로 정의하는 대신 두 클래스를 부모-자식 관계로 설정해 공통화하는 것이다. 

서브클래스는 수퍼클래스의 필드와 메서드를 상속하여 자신의 특성으로 가질 수 있으며, 서브클래스만의 고유한 필드나 메서드를 정의할 수 있다. 뿐만 아니라, 수퍼클래스의 인스턴스 메서드 시그니처를 유지하면서 새로운 기능으로 재정의하는 '다형성'을 꾀할 수 있다. 다형성을 활용한다면 상속을 단순히 클래스 속성 측면의 공통화를 넘어, [Template method](https://www.geeksforgeeks.org/template-method-design-pattern/)의 용례처럼 control flow를 공통화할 수 있다.

## 인터페이스와 상속의 문법적 차이
인터페이스와 상속을 자바의 문법적인 관점에서 살펴보자. 먼저 `abstract`라는 키워드에 대해 알아야 한다. `interface`가 `abstract`와 관련된 implicit한 문법적 특성을 가지고 있기 때문이다. `abstract`는 클래스와 메서드 선언 시 붙을 수 있는 키워드이다. 
- abstract 메서드는 해당 메서드의 구현 시점을 서브클래스의 정의로 미루겠다는 것을 의미한다. 따라서, abstract 메서드는 구현의 내용을 포함하고 있어선 안된다. 또한, **abstract method를 포함하는 클래스 혹은 인터페이스를 확장/구현하는 concrete class는 반드시 abstract 메서드를 재정의해야 한다.** 
- abstract 클래스는 직접 instantiate할 수 없는 클래스로, 해당 클래스를 확장하는 서브클래스를 통해 인스턴스를 만들어야 한다. abstract 메서드가 선언된 클래스는 반드시 abstract 클래스로 선언되어야 한다. 하지만, abstract 메서드가 없더라도 abstract 클래스로 선언하는 것은 허용된다.

### interface의 특징
1. 선언된 메서드는 대체로 `public abstract`의 성격을 갖는다. 아래 1.i ~ 1.iii는 메서드 구현을 제공할 수 있는 케이스이다.
    1. Java8 이후로 static 메서드에서 메서드 구현을 제공할 수 있다.
    1. Java8 이후로 default 메서드를 통해 기본 구현을 제공할 수 있다. 
    1. Java9 이후로 private static/instance method의 구현을 허용한다. 이때 private method는 반드시 구현되어야 한다.
1. 선언된 변수는 `public static final`의 성격을 갖는다. 
1. `interface`를 상속하는 class는 `implements` 키워드를 사용한다.
1. `interface`를 상속하는 인터페이스는 `extends`를 사용한다.
1. 클래스와 인터페이스는 인터페이스을 다중상속할 수 있다.
1. `Functional Interface`로 함수형 프로그래밍을 적용할 수 있다.

특징 1로부터, 인터페이스는 직접 instantiate할 수 없다. 따라서, 모든 변수는 특징 2를 따르게 된다. 이러한 특징은 인터페이스 내부의 변수와 메서드에 명시적으로 선언해주지 않더라도 적용된다.

Java8 이후로, 인터페이스 메서드에서 구현을 할 수 있도록 변경되었다. 이전의 인터페이스는 이전의 인터페이스는 concrete class에게 구현 책임을 강제하였고, 이에 따른 몇가지 문제가 발생하게 되었다. 
- 해당 인터페이스에서 static method로 구현하기 적합한 유틸리티 메서드임에도, 별도의 companion class를 만들어 정의해야 했다. (e.g., java.util.Collections) static method의 지원은 개발 편의성 측면에서 제약을 완화하기 위한 목적이다. 
- 인터페이스의 수정 시, 해당 인터페이스를 구현하는 모든 클라이언트 코드가 수정되어야 한다는 점이다. default method로 인터페이스 자체에서 hook을 제공함으로써, 클라이언트는 선택적으로 오버라이드할 수 있도록 (혹은 다시 abstract로 만들도록) 한다. 
```java
// sub-interface가 default method를 abstract로 만드는 방법
public interface SuperInterface {
    default void simpleMethod() {
        System.out.println("hi");
    }
}

public interface SubInterface extends SuperInterface {
    void simpleMethod();
}
```

6번의 `Functional Interface`는 '추상 메서드가 단 하나인 인터페이스'를 의미한다. @FunctionalInterface는 이 조건을 컴파일 타임에 검사해준다.
이는 자바 8의 또 다른 기능인 'Lambda expression'가 Functional Interface를 전달받기 때문이다. Lambda expression은 함수의 인자로 데이터를 넘기던 기존의 패러다임에서, 아래의 코드처럼 함수 자체를 넘길 수 있도록 지원하는 기능이다.
```java
@FunctionalInterface
public interface SimpleFunctionalInterface {
    int binaryOperation(int a, int b);

    default void hello() {
        System.out.println("hello from SimpleFunctionalInterface");
    }
}

public static void main(String[] args) {
    int addResult = op(1, 2, (a, b) -> a + b);
    int mulResult = op(1, 2, (a, b) -> a * b);
    int subResult = op(1, 2, (a, b) -> a - b);
    int divResult = op(1, 2, (a, b) -> a / b);
}

static int op(int a, int b, SimpleFunctionalInterface fi) {
    return fi.binaryOperation(a, b);
}
```

### 상속의 특징
1. 클래스를 상속하기 위해 `extends` 키워드를 사용한다. 
2. 클래스의 다중상속은 불가능하다. 
3. abstract method의 접근 제한자로 private이 올 수 없다. 서브클래스에서 접근할 수 있어야 하기 때문.
4. 서브클래스가 볼 수 있는 수퍼클래스의 인스턴스 메서드는 오버라이드할 수 있다. (static method는 불가능)
5. 서브클래스의 생성자 내부에서는 Superclass의 생성자를 호출해야 한다. 

인터페이스와 상속의 기능적 차이를 보면, 1) 다중상속 가능 여부, 2) 필드 제한을 들 수 있다.

### 다중상속
다이아몬드 문제로 인해, 한 클래스는 하나의 클래스만을 직접적으로 확장할 수 있다. SubClass1, SubClass2가 SuperClass.foo()를 오버라이드하고 있는 경우를 생각해보자. SubClass3.foo()는 SubClass1.foo(), SubClass2.foo() 중 어떤 메서드를 수행해야 하는지 알 수 없다. 
![diamond-prob](/img/interface-and-inheritence/diamond-prob.jpg)

반면, 인터페이스의 경우는 기능의 최종 구현이 concrete subclass에서 이뤄진다. 따라서 다중상속 문제에서 조금 더 자유롭다. 하지만 수퍼클래스에서 default method를 정의하고 있는 경우에는 다이아몬드 문제에 직면할 수 있다.

아래와 같은 클래스 다이어그램과 코드 구현에서, Client.foo()를 구현하지 않으면 어떤 인터페이스의 메서드를 호출해야 할 지 구분할 수 없게 된다.
따라서, Client 클래스는 foo()를 오버라이드해야 한다.
![interface-diamond-prob](/img/interface-and-inheritence/interface-diamond-prob.jpg)

```java
public interface SuperInterface {
    void foo();
}

public interface SubInterface1 extends SuperInterface {
    @Override
    default void foo() {
        System.out.println("hello from SubInterface1");
    }
}

public interface SubInterface2 extends SuperInterface {
    @Override
    default void foo() {
        System.out.println("hello from SubInterface2");
    }
}

public class Client implements SubInterface1, SubInterface2 {
//    @Override
//    public void foo() {
//        System.out.println("hello from Client");
//    }
}
```

### 필드 제한
클래스와 달리 인터페이스의 필드는 `public static final`로 접근 제한이 설정된다. 앞서 설명했듯, 인터페이스 자체로는 인스턴스를 생성할 수 없기 때문에, 필드 변수는 정적이고 재할당 불가능한 값을 가질 수밖에 없다. 인터페이스가 해결하고자 하는 미션은 객체의 상태가 아닌 객체가 가져야 하는 행동 양식을 표현하는 것이며, 인터페이스의 필드는 객체의 상태를 가지기보다, 인터페이스의 유틸리티적 성격이 강하다. 

Java8 이후로 인터페이스에서 default method가 추가되면서, abstract class와의 차이가 옅어졌다. 그럼에도, '상태의 표현'이라는 관점에서 interface와 abstract class는 뚜렷히 구분된다.

## When to use: Interface vs. Abstract class
### Interface
- 설계 명세만을 선언하고 실제 구현 내용에 대해서는 관심이 없는 경우
- 객체가 구현해야 할 행위를 제시하지만, 이를 구현하는 객체 간 큰 연관성이 없을 경우 (Cloneable, Comparable ...)
- 다중 상속이 필요한 경우
- 기능의 변경이 클 것이라 예상되지 않는 경우 (default는 하위 호환성을 위한 것이며 남발은 좋지 않다는 견해에 따르면)

### Abstract
- field로 표현되는 객체의 상태가 있고, 이를 추상화할 경우
- 상속 구조 간 유사성이 있는 경우 
- 공통화할 field, method에 public 이외의 접근 제한자를 설정할 경우