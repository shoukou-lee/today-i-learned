Garbage와 Garbage Collection
---------------------------
Oracle GC 튜토리얼

### Garbage

* 스택 영역/메서드 영역에서 도달 불가능한 (`Unreachable`) 힙 영역 객체
* 더이상 참조되지 않는 힙 영역의 객체를 의미

```java
// 힙 영역에 User("garbage")가 할당되고, 스택 영역의 user는 이를 참조
User user = new User("garbage"); 
user.doSomething();

// 힙 영역에 User("new")가 할당되고, user는 이를 참조하며 User("garbage")는 garbage가 됨
user = new User("new"); 
user.doSomething();
```

### Garbage Collection

* Garbage collector에 의해 주기적으로 Garbage가 차지한 메모리 공간을 회수하는 작업
* Stop-the-world, Mark-and-sweep이라는 2단계를 거침

Memory layout - Heap / Non-heap
-------------------------------

### GC 관점에서의 Generation 구분

아래 그림은 Hotspot JVM의 메모리 레이아웃 일부이다. JVM Specification 중 Heap과 Method area의 구현이다. 3개의 세대에서 GC가 발생한다.
![image](/img/three-generation-in-javamem.png) 

* Heap은 Young/Old Generation으로 구성 
  * YoungGen 중 Eden은 새로운 객체가 처음으로 할당되는 영역, from/to는 Survivor 영역으로, 최소 1회의 GC에서 살아남은 객체들이 존재하는 영역
  * OldGen은 YoungGen에서 일정 age 이상 살아남은 객체들이 존재하는 영역
* PermGen은 Method area를 구현한 것으로, 클래스 메타데이터, 정적 변수 등이 저장되는 영역.

> NOTE: PermGen은 Java8 이후 Metaspace로 대체되며, 기존 PermGen에 저장되던 일부 요소들이 다른 영역에 저장되는 등의 수정 사항이 있음. ([JEP-122](https://openjdk.org/jeps/122))

### Weak Generational Hypothesis - 세대 별 GC의 이유

모든 객체의 참조 여부를 일일히 확인하는 것은 큰 비용이 든다. 하지만, 응용프로그램의 메모리 사용 패턴을 조사해보면 일반적으로 다음과 같은 특성이 있다.

* 오랜시간 참조되는 객체는 드물고, 대부분의 객체는 일찍 죽는다.
* Older object가 Younger object를 참조하는 경우는 드물다

이 특성을 전제로 최적화할 수 있는데, 이 전제를 `약한 세대 가설` 이라 한다. 각각의 세대 별로 GC를 수행하는 것은 죽은 객체를 조사할 탐색 범위가 줄어드는 것을 의미한다. 가령, 새로운 객체가 빈번히 생성되어 Eden 영역이 가득 차면, Eden 영역에 할당된 객체의 참조 여부를 검사하고 죽은 객체를 비운다.

GC Process
----------

### Minor GC

1. `Eden`에 새로운 객체가 할당된다.
  ![image](/img/gc-1.png) 

2. Eden이 가득 차면, 애플리케이션의 모든 스레드가 중지된다 (Stop-the-world). 이후, 죽은 객체와 살아있는 객체를 구분하는 작업을 한다 (Mark). 주황색으로 표시된 객체는 죽은 객체이므로 해제된다 (Sweep). 파란색으로 표시된 살아있는 객체는 survivor space로 이동한다. survivor 영역의 객체에 쓰인 1은 `Age`를 의미하며, GC에서 살아남은 횟수를 의미한다.

  ![image](/img/gc-2.png) 
3. 다시 Eden이 가득 차면 Minor GC가 실행된다. 이때, Eden과 From survivor space에 있는 모든 lived 객체는 `To survivor space`로 이동하며, Age가 1 증가한다. From/To survivor space는 매 GC마다 번갈아가면서 바뀐다. 두 survivor space 중 하나에만 객체가 존재해야 한다. From/To를 번갈아가며 객체를 이동시킴으로써, 살아남은 객체를 구분할 수 있다는 점, Memory compaction 효과를 얻을 수 있다.

  ![image](/img/gc-3.png) 
4. Minor GC가 반복된 상황. 마찬가지로 lived 객체의 복사가 일어난다. 이때 From survivor space에서 Age≥8이 되는 객체들은 Old Generation으로 복사된다. 객체의 Age가 특정 Threshold를 넘어서면 Old Generation으로 이동하는 과정을 `Promotion`이라 한다.

  ![image](/img/gc-4.png) 

GC를 구성하는 절차
-----------

### Stop The World

* JVM이 GC를 실행하는 스레드를 제외한 모든 스레드를 중단시키는 작업
* 즉, 애플리케이션의 실행을 멈추는 것이며, 성능에 악영향을 준다.

### Mark and Sweep

* Mark

  ![image](/img/mark.png) 

  * 스택의 변수로부터 도달할 수 있는 힙 영역의 객체를 그래프 탐색한 후, 도달한 힙 영역의 객체를 마킹
* Sweep

  ![image](/img/sweep.png) 

  * 힙 영역의 각 객체를 탐색 후, 마킹되지 않은 객체는 collect, 마킹된 객체는 다음번 알고리즘 수행을 위해 마킹을 해제
* Compact

  ![image](/img/compact.png) 

  * Sweep으로 인해 가비지가 제거되었을 경우, 메모리 파편화가 발생할 수 있음
  * 따라서, 살아있는 객체를 메모리의 시작 부분으로 복사하고, 해당 개체의 참조를 업데이트하는 일종의 조각모음
  * Stop-the-world time이 길어질 수 있음

## References
- [Java Garbage Collection Basics](https://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html)


---

* minor GC인 경우 YoungGen에 대해 관여할 텐데, 스택에서 tracing 시 old/young reference의 구분을 어떻게?