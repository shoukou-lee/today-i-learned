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

String.equals(...)의 구현은 문자열을 구성하는 개별 문자의 byte 값을 비교하도록 구현되어있다.
```java
// String.java
public boolean equals(Object anObject) {
	if (this == anObject) {
		return true; // 레퍼런스 비교
	} 
	return (anObject instanceof String aString) // 타입 비교
			&& (!COMPACT_STRINGS || this.coder == aString.coder) // COMPACT_STRINGS enable이라면, coder가 같은지 비교
			&& StringLatin1.equals(value, aString.value); // 두 byte[]를 선형 탐색 비교
}

// StringLatin1.java
@IntrinsicCandidate
public static boolean equals(byte[] value, byte[] other) {
	if (value.length == other.length) {
		for (int i = 0; i < value.length; i++) {
			if (value[i] != other[i]) {
				return false; 
			}
		}
		return true;
	}
	return false;
}
```
> NOTE: COMPACT_STRINGS는 Java9에서 생긴 개념으로, 모든 문자를 2-byte char로 표현하는 대신, (각 문자를 1-byte로 표현 가능하다면) 1-byte로 표현한다. 이때 사용되는 인코딩 형식이 LATIN1이며, 한 문자에 2-byte를 매핑할 때 사용되는 형식은 UTF-16이다.

만일, equals()를 오버라이드할 경우, (정말 특별한 경우가 아니라면) hashCode()를 같이 오버라이드해야 한다. 

## **hashCode() 메서드**
개별 객체가 가진 주소를 32-bit 정수로 변환한 값을 리턴한다. 해시 테이블의 성능 향상이 주 목적이다. 다음은 javadoc에서 설명하는 hashCode()’s contract이다.

- 같은 자바 프로세스 내에서 한 객체에 대해 hashCode()가 여러번 호출되더라도 동일한 정수를 리턴한다. JVM이 GC를 수행하는 과정에서 객체의 메모리 주소가 이동하더라도 해시코드는 변하지 않아야 한다. 단, equals() 판단에 사용되는 정보가 수정되는 경우는 해시코드가 변경될 수 있다. 
- 두 객체의 equals() 결과가 같으면, hashCode() 결과도 같아야 한다.
- 두 객체의 equals()가 다르더라도 꼭 hashCode()의 결과가 달라야하는 것은 아니다. 다만, 다른 해시코드를 갖는 것이 해시 테이블의 성능을 향상시킬 수 있다. (해시 충돌)

이 중 2nd contract이 바로 해시 기반 자료구조 (e.g., HashMap)의 최적화를 위한 조건이다. 가령, HashMap.putVal(…) 메서드에서는 Key의 해시코드 정보를 바탕으로, 어떤 버킷에 키/값 쌍을 저장할 지 결정한다. 

논리적으로 같은 두 객체가 같은 해시코드를 갖는다면, 하나의 버킷에 대해 키/값 쌍이 덮어써진다. 하지만 해시코드를 오버라이드하지 않아 다른 해시코드를 가지게 되면, 해시맵 내부의 테이블 중 서로 다른 버킷에 데이터가 쓰여지므로, 중복 키가 저장될 수 있다.

따라서, 다음의 경우 항상 hashCode() 오버라이드를 구현해야 한다.
- equals()의 오버라이딩 시
- 해시 기반 자료구조에서 키로 활용 시

String의 hashCode()는 문자열 내용을 토대로 해시코드를 도출한다. 
```java
/**
 * 스트링에 대한 해시코드를 반환합니다.
 * String 객체의 해시코드는 아래의 식에 의해 계산됩니다 : 
 * s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]
 * 이때, s[i]는 i-th 문자, n은 스트링의 길이를 의미합니다.
 * 이 규칙에 의해, empty string의 해시코드는 0을 리턴합니다.
 * @return  a hash code value for this object.
 */
public int hashCode() {
    int h = hash; // 초기값 0
    // 이전에 해시코드를 계산하지 않은 경우 계산
    if (h == 0 && !hashIsZero) {
        h = isLatin1() ? StringLatin1.hashCode(value) // s[i]에 대해(0~255) 위 수식 적용
                       : StringUTF16.hashCode(value); // {s[i], s[i + 1]}을 bit-concat한 결과에(0~65535) 위 수식 적용
        if (h == 0) {
            hashIsZero = true; // hashIsZero flag를 set해 추후 비교 없이 리턴함
        } else {
            hash = h; // 0이 아닌 경우 업데이트
        }
    }
    return h;
}
```
가령, “abcd”의 해시코드는 다음과 같이 계산된다. (coder=Latin1인 경우)
- byte[] = {a(_97_), b(_98_), c(_99_), d(_100_)}
- (_97_ * 31^3) + (_98_ * 31^2) + (_99_ * 31^1) + (_100_ * 31^0) = 2987074

String 객체의 해시코드를 결정하는 것은 문자열의 내용이다. 서로 다른 레퍼런스를 가진 String 객체더라도, 내부적으로 byte[] value content가 같다면 항상 같은 해시코드를 얻게 된다. 어떤 경우에는 다른 문자열로 같은 해시코드를 얻는 충돌 상황이 생길 수도 있다. 대표적인 예는 "Siblings"와 "Teheran"이다. 두 스트링 모두 231609873의 해시코드를 가진다. 
![image](/img/string-hashcode-collision.png) 

## **References**
- [Java Language Specification](http://docs.oracle.com/javase/specs/jls/se19/html/jls-15.html#jls-15.21)
- [javadoc 19](https://docs.oracle.com/en/java/javase/19/docs/api/java.base/java/lang/Object.html)