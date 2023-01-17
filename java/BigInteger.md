
## BigInteger
임의 정밀도 산술을 제공하는 불변 클래스이다. BigInteger에서 사용되는 표현은 모두 2의 보수를 전제하고 있다.
primitive int 연산, Java.lang.Math, modulo, GCD, 소수 생성/검사, 비트 조작 등의 기능을 제공한다.
산술 연산의 경우, JLS에 정의된 정수 산술 연산 규칙을 따름. 단, 무한 단어 크기의 작업 결과를 수용하기 위한 클래스인 만큼 오버플로우에 대한 명세는 무시한다.
 
```java
public class BigInteger extends Number implements Comparable<BigInteger> {
    final int signum; // sign(x) 결과를 담는 변수
    /**
     * magnitude를 담는 big-endian order로 저장하는 배열.
     * 가령, 128-bit로 표현되는 수는 int의 크기인 32-bit 단위로 슬라이싱되어 다음과 같이 저장된다.
     * [[0:31], [32:63], [64:95], [96:127]] 
     */
    final int[] mag; // magnitue를 담는 배열
```
BigInteger를 표현하기 위해서, 부호와 숫자를 담을 mag 배열이 필드에 선언되어있다.

## BigInteger의 생성자
가장 자주 사용될 것으로 생각되는 BigInteger(String val)의 생성자를 확인해보면, 내부적으로 BigInteger(val, int radix = 10)으로 위임하고 있다.

```java
public BigInteger(String val, int radix) {
    int cursor = 0, numDigits;
    final int len = val.length();

    if (radix < Character.MIN_RADIX || radix > Character.MAX_RADIX)
        throw new NumberFormatException("Radix out of range");
    if (len == 0)
        throw new NumberFormatException("Zero length BigInteger");
```
먼저, radix parameter가 유효한 범위인지 확인한다. 2 <= radix <= 36의 범위를 지원한다. 
이후, 주어진 스트링의 길이가 0인지 확인한다. 이 조건에 맞지 않으면 NumberFormatException을 던진다.

```java
    // Check for at most one leading sign
    int sign = 1;
    int index1 = val.lastIndexOf('-');
    int index2 = val.lastIndexOf('+');
    if (index1 >= 0) {
        if (index1 != 0 || index2 >= 0) {
            throw new NumberFormatException("Illegal embedded sign character");
        }
        sign = -1;
        cursor = 1;
    } else if (index2 >= 0) {
        if (index2 != 0) {
            throw new NumberFormatException("Illegal embedded sign character");
        }
        cursor = 1;
    }

    // 부호만 있는 경우. "+", "-" 등
    if (cursor == len)
        throw new NumberFormatException("Zero length BigInteger");
```
이후, 부호를 결정한다. 초기 부호는 int sign = 1;에서, 양수를 가정한다. 
부호 결정 알고리즘은 단순한데, '+', '-'가 주어진 스트링의 0번째 인덱스에만 존재하는지 검사하는 것이다. 이후, 부호만 있고 magnitude가 없는 스트링이라면 예외를 던진다.

```java
    // Skip leading zeros and compute number of digits in magnitude
    while (cursor < len &&
           Character.digit(val.charAt(cursor), radix) == 0) {
        cursor++;
    }

    // "000"과 같이 0으로만 이루어진 경우,
    // 부호와 magnitude를 0으로 업데이트 후 리턴
    if (cursor == len) {
        signum = 0;
        mag = ZERO.mag; 
        return;
    }

    numDigits = len - cursor; // leading-0을 뺀 유효한 digit 수
    signum = sign; // 부호 set
```
leading-0을 건너뛰면서 cursor를 움직인다. cursor는 유효 digit이 시작하는 위치를 가리키게 된다. 
이후, 유효 digit 수를 계산하고, 부호를 set 한다.

```java
    // Pre-allocate array of expected size. May be too large but can
    // never be too small. Typically exact.
    long numBits = ((numDigits * bitsPerDigit[radix]) >>> 10) + 1;
    if (numBits + 31 >= (1L << 32)) {
        reportOverflow();
    }

    int numWords = (int) (numBits + 31) >>> 5;
    int[] magnitude = new int[numWords];
```
[numDigits-자리의 10진수를 표현하기 위한 비트 수를 계산하고 numBits에 저장한다.](https://stackoverflow.com/a/36661939) 이때,(numBits + 31)이 2^32 이상이면 오버플로우로, ArithmeticException을 던진다. [1,292,782,622자리 이상의 10진수인 경우 이 범위를 넘는다.](https://www.wolframalpha.com/input?i=%28%28x*3402%29+%2F+%282%5E10%29%29+%2B+1+%2B+31+%3E%3D+2%5E32)

numWords는 magnitude의 배열 크기를 의미한다. `(numBits + 31) >>> 5`는 (numBits + 31) / 2^5로, 예상하는 비트 수를 32로 나눈 것이다. 
magnitude 배열은 32-bit 크기를 가지는 int의 배열이다. 주어진 수를 magnitude 배열에 담기 위해 몇개의 공간이 필요한가를 계산하는 과정이다.
여기서, magic number 31을 더하는 것은 어떤 수를 32 단위로 ceiling하기 위함이라는 것을 유추할 수 있다.

```java
    // Process first (potentially short) digit group
    int firstGroupLen = numDigits % digitsPerInt[radix];
    if (firstGroupLen == 0)
        firstGroupLen = digitsPerInt[radix];
    String group = val.substring(cursor, cursor += firstGroupLen);
    magnitude[numWords - 1] = Integer.parseInt(group, radix);
    if (magnitude[numWords - 1] < 0)
        throw new NumberFormatException("Illegal digit");

    // Process remaining digit groups
    int superRadix = intRadix[radix];
    int groupVal = 0;
    while (cursor < len) {
        group = val.substring(cursor, cursor += digitsPerInt[radix]);
        groupVal = Integer.parseInt(group, radix);
        if (groupVal < 0)
            throw new NumberFormatException("Illegal digit");
        destructiveMulAdd(magnitude, superRadix, groupVal); // (magnitude[i] * superRadix + groupVal) for all `i`
    }
    // Required for cases where the array was overallocated.
    mag = trustedStripLeadingZeroInts(magnitude); // magnitude 배열의 leading-0 제거
    if (mag.length >= MAX_MAG_LENGTH) {
        checkRange();
    }
```
이후, 주어진 수를 일정 길이로 끊어 처리하며, magnitude 배열을 완성한다. radix=10인 경우, 9자리로 끊어서 계산한다. 
완성된 magnitude 배열을 bit-concat하면 만들고자 한 수를 얻을 수 있다. 

가령, `BigInteger bi = new BigInteger("12345678901234567890");`에서, 디버그 모드로 bi.mag을 들여다보면 `mag = [-1420514932, -350287150]`임을 볼 수 있다.
10진수 `12345678901234567890'은 hexadecimal로 변환하면 'AB54A98C_EB1F0AD2'로 표현되며, 2의 보수 AB54A98C을 10진수로 읽으면 -1420514932가 된다. EB1F0AD2도 마찬가지.

arithematic 연산의 경우에도, 이 magnitude 배열을 이용해 32-bit 단위로 끊어서 계산한다.
아래는 add(int[] x, int[] y) 메서드 스니핏.
```java
    while (yIndex > 0) {
        sum = (x[--xIndex] & LONG_MASK) + // MASKING해서 무부호 수를 얻는다.
              (y[--yIndex] & LONG_MASK) + (sum >>> 32);
        result[xIndex] = (int)sum;
    }
```
