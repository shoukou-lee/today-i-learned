자바에서 자료형은 Primitive type과 Reference type으로 구분되며, 변수는 클래스 변수, 로컬 변수, 인스턴스 변수, 파라미터로 구분된다.

자료형 구분의 핵심은 변수가 표현하는 것이 값인지 레퍼런스인지에 대한 기준이다. 반면, 변수의 구분은 선언 위치에 대한 기준이며, 이에 따라 라이프사이클과 저장 공간이 달라진다.

### 변수 구분 요약
| 변수의 구분 | 클래스 변수 | 로컬 변수 | 인스턴스 변수 | 파라미터 |
|-------------|-------------|-------------|-------------|-------------|
| 선언 위치 | 필드 (static) | * 생성자/메서드 {} 내부 | 필드 | 생성자/메서드 () 내부 |
| 변수 생성 영역 | 메서드 영역 | JVM 스택 | 힙 | JVM 스택 |
| 라이프사이클 | 클래스 로딩~JVM 종료 | {} 내부 | 클래스 생성 ~ 클래스 파괴 (GC) | 메서드 호출~메서드 종료 |
> \* NOTE: 필드 영역의 {} 블록은 인스턴스 초기화 블록임

### JVM 인스트럭션으로 스택 동작 확인하기

변수와 스택의 상호작용을 직접 확인하기 위해, JVM 인스트럭션을 분석하고자 한다.

스택에 저장되는 타입을 직접 확인해보기 위한 예제 코드. 인스턴스 변수와 파라미터, 로컬 변수 각각에 primitive type, reference type을 정의하고 있다.

```java
public class SimpleClass {

    private static int staticPrimitive = 1;

    private int fieldPrimitive;

    private SimpleSubClass fieldReference;
    
    public SimpleClass(int argPrimitive, SimpleSubClass argReference) {
        this.fieldPrimitive = argPrimitive;
        this.fieldReference = argReference;
    }

    public void method(int argPrimitive, SimpleSubClass argReference) {
        int localPrimitive = argPrimitive;
        {
          int localPrimitive2 = 1;
        }
        SimpleSubClass localReference = argReference;
        SimpleSubClass newLocalReference = new SimpleSubClass();
    }
}
```

이 코드는 아래처럼 disassemble 된다. 인스트럭션 오른쪽 주석은 스택 프레임 내부의 operand stack, local vairable array의 변화 과정을 이해하기 쉽도록 표현한 pseudo-code 이다.

(스택 프레임, operand stack, local variable에 대한 설명은 [JVM Specification](https://docs.oracle.com/javase/specs/jvms/se17/html/jvms-2.html#jvms-2.6)), 혹은 [요약한 글](<https://velog.io/@shoukou-lee/runtime-data-area>)에서)

```java
public void method(int, SimpleSubClass);
    descriptor: (ILSimpleSubClass;)V
    flags: (0x0001) ACC_PUBLIC
    /* 
    to explain more easier, comments of each instruction (start with '//#') show the pseudo-high-level expression.
    note that the terms 'os' and 'lva' represent 'operand stack' and 'local variable array', respectively.
    */
    Code:
      // stack = {'empty'} with its maxSize=3
      // lva[6] = {ThisRef this, int argPrimitive, ObjectRef argReference, null, null, null}
      // args_size = number of non-null elements of initial lva[6]
      stack=3, locals=6, args_size=3
         0: iload_1         //# os.push(lva[1])
         1: istore_3        //# lva[3] = os.pop()
         2: iconst_1        //# os.push(1)
         3: istore        4 //# lva[4] = os.pop() 
         5: aload_2         //# os.push(lva[2])
         6: astore        4 //# lva[4] = os.pop() (i.e., overwrite)
         8: new           #7//# os.push(ObjectRef ref = Heap.new(SimpleSubClass).reference)
        11: dup             //# os.push(os.top())
        12: invokespecial #9//# os.pop().<init>
        14: astore        5 //# lva[5] =  os.pop()
        17: return          
```

메서드가 호출될 때, 파라미터 `int argPrimitive`와 `SimpleSubClass argReference`는 local variable array에 복사되고, 로컬변수 `int localPrimitive`와 `SimpleSubClass localReference`, `SimpleSubClass newLocalReference`는 인스트럭션을 수행하는 과정에서 (operand stack manipulation) local variable array에 저장된다. 물론 `argReference`, `localReference`, `newLocalReference가` 표현하는 것은 레퍼런스이다.

`int localParamitive2`와 같이 별도의 블록 안에 선언된 변수가 저장된 local variable array 공간은 스코프를 벗어난 후 새로운 값으로 덮어씌워질 수 있다. (6: astore 4)

8: new \#7에서 보듯, 새로운 객체는 힙 영역에 생성되고, 그 레퍼런스를 local variable array에 저장된다.

즉, 스택 영역에는 지역 변수와 파라미터가 임시적으로 저장된다.

### 힙 영역에 생성된 객체 확인하기

객체를 instantiate하고, 힙 덤프를 통해 객체가 잡고 있는 필드 상태를 확인해볼 수 있을 것 같다.

힙 영역에 생성된 객체를 확인하기 위해, SimpleClass를 instantiate한다.

```java
public class Main {

    public static void main(String[] args) {
        List<SimpleClass> classes = new ArrayList<>();

        int count = 0;
        while (true) {
            addNewInstanceAndSleep(classes, count++);
        }
    }

    static void addNewInstanceAndSleep(List<SimpleClass> list, int count) {
        list.add(new SimpleClass(count, new SimpleSubClass()));
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }
}
```

Heap 영역에 생성된 인스턴스 정보를 확인하기 위한 Heap dump. physical memory address를 보여주지는 않지만, 확실히 객체가 필드로 가진 primitive type, referenced type의 값은 확인할 수 있다.

![heapdump](/img/heapdump.png)

### Primitive type은 스택, Reference type은 힙?
이 명제가 사실이라면, 모든 primitive type 변수는 thread-safe해야 한다. 왜냐하면, 스택 영역은 개별 스레드에 할당되는 영역, 힙은 스레드가 공유할 수 있는 영역이기 때문이다. 하지만 간단한 테스트로 그렇지 않다는 것을 알 수 있다. 

<details>
<summary>예를 들면 ..</summary>
<div markdown="1">

```java
public class MultiThreadedIncrease {
    private int primitive = 0;

    private int numThread;

    private List<Thread> threads;

    public MultiThreadedIncrease(int numThread) {
        this.numThread = numThread;
        this.threads = new ArrayList<>();
        for (int i = 0; i < numThread; i++) {
            threads.add(new Thread(() -> {
                try {
                    Thread.sleep(10);
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
                // synchronized (this)
                {
                    this.primitive++;
                }
            } ));
        }
    }

    public void runAllThreads() {
        for (Thread thread : threads) {
            thread.start();
        }

        for (Thread thread : threads) {
            try {
                thread.join();
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        }
    }

    public boolean isPrimitiveReallySafe() {
        return this.primitive == this.numThread;
    }
}
```

</div>
</details>

'자료형'과 '변수'를 명확히 구분해야 한다. 자료형(types)의 분류는 해당 변수가 표현하고자 하는 것이 값 자체인지 레퍼런스인지를 구분할 뿐, 변수가 저장되는 메모리 영역과 연관짓지 말아야 한다. 이는 변수의 4가지 분류와 연결되는 개념이다.