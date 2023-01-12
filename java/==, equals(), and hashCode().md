# ==, equals() and hashCode()

## **== (Equality Operator)**

`==` 연산자는 피연산자로 boolean, numerics, reference type을 받을 수 있다. 피연산자 타입에 따라 다른 동작 방식을 가진다. 이때, 두 피연산자가 모두 Reference type이거나 null 타입이라면 object equality를 비교한다. 즉, 두 피연산자가 모두 null이거나 동일한 레퍼런스를 가진다면 true, 그렇지 않으면 false를 리턴한다.

## **equals() 메서드**

equals()는 `Object` 클래스에 정의된 메서드이다. Object 클래스는 모든 클래스들의 root-superclass이다. 다시말해, 배열을 포함한 모든 레퍼런스 타입들은 기본적으로 Object에서 정의된 메서드를 상속한다. equals() 또한 이 중에 포함된다. 

Object.equals()의 구현을 살펴보면, this reference와 파라미터로 들어온 객체의 Object equality를 수행한다. 즉, 동일한 레퍼런스인지를 비교하게 된다.

```java
// Object.java
public boolean equals(Object obj) {
	return (this == obj);
}
```

필요에 따라 Object equality의 판단 기준을 reference equality가 아닌 객체의 상태, 즉 객체 내부의 값이 같은가를 판별하는 value equality로 두고 싶은 경우가 있다. 가령 String의 문자열 동등성 검사나, VO의 값 동등성을 확인하는 등이 그 예이다. 이러한 경우, equals() 메서드를 오버라이드함으로써, 동등성 판단 기준을 바꿀 수 있다.

만일, equals()를 오버라이드할 경우, (정말 특별한 경우가 아니라면) hashCode()를 같이 오버라이드해야 한다. 

## **hashCode() 메서드**

개별 객체가 가진 주소를 32-bit 정수로 변환한 값을 리턴한다. 해시 테이블의 성능 향상이 주 목적이다. 다음은 javadoc에서 설명하는 hashCode()’s contract이다.

- 같은 자바 프로세스 내에서 한 객체에 대해 hashCode()가 여러번 호출되더라도 동일한 정수를 리턴한다. 단, equals() 판단에 사용되는 정보가 수정되는 경우는 해시코드가 변경될 수 있다.
- 두 객체의 equals() 결과가 같으면, hashCode() 결과도 같아야 한다.
- 두 객체의 equals()가 다르더라도 꼭 hashCode()의 결과가 달라야하는 것은 아니다. 다만, 다른 해시코드를 갖는 것이 해시 테이블의 성능을 향상시킬 수 있다. (해시 충돌)

이 중 2nd contract이 바로 해시 기반 자료구조 (e.g., HashMap)의 최적화를 위한 조건이다. 가령, HashMap.putVal(…) 메서드에서는 Key의 해시코드 정보를 바탕으로, 어떤 버킷에 키/값 쌍 을저장할 지 결정한다. 

논리적으로 같은 두 객체가 같은 해시코드를 갖는다면, 하나의 버킷에 대해 키/값 쌍이 덮어써진다. 하지만 해시코드를 오버라이드하지 않아 다른 해시코드를 가지게 되면, 해시맵 내부의 테이블 중 서로 다른 버킷에 데이터가 쓰여진다. 

따라서, 다음의 경우 항상 hashCode() 오버라이드를 구현해야 한다.

- equals()의 오버라이딩 시
- 해시 기반 자료구조에서 키로 활용 시

## **References**

- [Java Language Specification](http://docs.oracle.com/javase/specs/jls/se19/html/jls-15.html#jls-15.21)
- [javadoc 19](https://docs.oracle.com/en/java/javase/19/docs/api/java.base/java/lang/Object.html)