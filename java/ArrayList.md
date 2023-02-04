## ArrayList란?
ArrayList는 원소 관리를 내부적으로 배열을 통해 관리하는 컬렉션이다. 배열의 특징인 Random-access가 가능하며, 배열이 가득 찬 경우 더 큰 배열을 생성해주기 때문에, 클라이언트는 가변배열처럼 활용할 수 있다. 

배열 기반의 자료구조이므로, 인덱스를 통한 접근은 O(1) 시간복잡도를 갖는다. 반면 탐색을 통한 접근은 O(N) 시간복잡도를 가지게 된다. 

## ArrayList의 구현
### ArrayList의 상속 구조
`ArrayList`은 Skeletal implementation인 `AbstractList`를 상속한다. `List`, `RandomAccess`, `Cloneable`, `Serializable`을 구현한다. 이 중 `List`를 제외한 인터페이스들은 Marker Interface이다. 아무 기능도 명세하지 않지만 Random-access 등의 기능이 있다는 것을 명시해주기 위한 인터페이스이다.
### ArrayList의 field
원소를 담을 `elementData[]`, 현재 배열에 담긴 원소의 개수 `size` 외에 빈 배열에 대한 공유 인스턴스와 기본 초기값을 필드로 가지고 있다. 
ArrayList의 크기와 관련된 `capacity`와 `size`라는 용어를 잘 구분해야한다. `capacity`는 쉽게 말하면 `elementData[].length`로, 할당된 배열의 크기를 의미한다. `size`는 `elementData[]`를 차지하는 element의 개수이며, operation이 실행될때마다 증가/감소시키며 카운트한다.

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    // element가 저장되는 배열. 배열의 길이를 capacity라고 함.
    transient Object[] elementData; // non-private to simplify nested class access

    // 초기 capacity 기본값
    private static final int DEFAULT_CAPACITY = 10;

    // 빈 배열 공유 인스턴스
    private static final Object[] EMPTY_ELEMENTDATA = {};

    // 초기 capacity를 명시하지 않은 경우 사용되는 빈 배열 공유 인스턴스 
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    // 실제 element의 개수
    private int size;
    ...
```

### ArrayList 생성자
생성자는 3개가 있다.
- `elementData[]`의 초기 capacity를 설정할 수 있다. 0이라면 `EMPTY_ELEMENTDATA`를 참조한다.
- 아무 인자도 주지 않으면 `DEFAULTCAPACITY_EMPTY_ELEMENTDATA`를 참조한다.
- Collection을 인자로 주면, Collection을 배열로 변환한 뒤 복사한다. 빈 컬렉션이면 `EMPTY_ELEMENTDATA`를 참조한다.

```java
  public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
  }
  
  public ArrayList() {
      this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
  }
  
  public ArrayList(Collection<? extends E> c) {
      Object[] a = c.toArray();
      if ((size = a.length) != 0) {
          if (c.getClass() == ArrayList.class) {
              elementData = a;
          } else {
              elementData = Arrays.copyOf(a, size, Object[].class);
          }
      } else {
          // replace with empty array.
          elementData = EMPTY_ELEMENTDATA;
      }
  }
```

### 원소 읽기/쓰기 : get(), set()
- 시간복잡도 O(1)

`get()`은 특정 인덱스의 원소를 리턴한다. `set()`은 특정 인덱스의 원소를 새 원소로 덮어쓴 후, 이전 값을 리턴한다. `ArrayList`는 element를 배열로 관리하므로 Random-access가 가능하다. 따라서, `get()`, `set()`은 O(1) 시간복잡도로 동작하게 된다.

```java
  public E get(int index) {
    Objects.checkIndex(index, size);
    return elementData(index);
  }
    
  public E set(int index, E element) {
    Objects.checkIndex(index, size);
    E oldValue = elementData(index);
    elementData[index] = element;
    return oldValue;
  }
  
  E elementData(int index) {
    return (E) elementData[index];
  }
```

### 원소 탐색 : indexOf(), lastIndexOf(), contains()
- 시간복잡도 O(N)

`indexOf()`는 특정 원소와 일치하는 첫번째 element의 인덱스를 반환한다. `elementData[]`를 인덱스 [0, size - 1] 범위로 순회하면서, `equals()`를 검사하도록 구현되어있다. 

`lastIndexOf()` 또한 전체 순회로 인덱스를 찾아주지만, 순회의 방향이 반대이다. 따라서, 일치하는 가장 마지막 인덱스를 반환하게 된다.

`contains()`는 내부적으로 `indexOf()`를 호출하며, 인덱스 대신 원소의 포함 유무를 `boolean`으로 리턴한다. 

```java
  public int indexOf(Object o) {
    return indexOfRange(o, 0, size);
  }
  
  int indexOfRange(Object o, int start, int end) {
    Object[] es = elementData;
    if (o == null) {
      for (int i = start; i < end; i++) {
        if (es[i] == null) {
            return i;
        }
      }
    } else {
      for (int i = start; i < end; i++) {
        if (o.equals(es[i])) {
            return i;
        }
      }
    }
    return -1;
  }
```

### 원소 추가 : add()와 스레드 경합 

- 시간복잡도 O(1) (amortized)

`add()`는 마지막 원소의 다음 위치에 접근해 새 element를 추가하는 연산이다. 마지막 원소의 위치를 인덱스 기반으로 접근한다. 

내부적으로 private helper method를 호출한다. helper method는 capacity와 size를 비교한다. 이 둘이 같다는 것은 배열 용량이 꽉 찬 상황을 의미한다. 이때 새 아이템을 추가하기 위해 배열을 확장해야 하는데, 이를 `grow()` 메서드가 처리한다.
이후, 배열의 마지막 인덱스에 element를 추가하고 size를 증가시킨다. resizing을 하지 않는다면, 원소 추가는 O(1)의 시간복잡도를 가지게 된다. 이는 growth의 시간복잡도에서 다시 살펴보자.

> NOTE: `modCount`라는 값을 증가시키는 것을 볼 수 있다. `modCount`는 `AbstractList`의 필드 중 하나이며, `List`에 구조적인 변화가 생길때마다 증가하는 값이다. `Iterator`를 사용해 순회하는 도중 `modCount`가 변화하면 `ConcurrentModificationException`을 던지기 위한 용도이다.

```java
  public boolean add(E e) {
    modCount++;
    add(e, elementData, size);
    return true;
  }
  
  private void add(E e, Object[] elementData, int s) {
    if (s == elementData.length)
        elementData = grow();
    elementData[s] = e;
    size = s + 1;
  }
```

`add()` 메서드를 보면 `ArrayList`는 Thread-unsafe하다는 것을 알 수 있다. 현재 size를 기준으로 helper method를 호출하며, 이 과정은 atomic하지 않다. 아래 그림과 같이 두 스레드가 `ArrayList`의 마지막 위치에 서로 다른 값을 추가하려는 상황이 있다. 두 스레드 모두 현재 평가된 `ArrayList.size`를 pass-by-value로 넘겨받은 후, 원소가 추가될 인덱스로 사용한다. 두 스레드 모두 같은 위치에 데이터를 쓰기 때문에, 최종적으로 배열의 마지막에는 "b"가 들어가게 된다. multi-thread 상황에서 같은 ArrayList 객체를 조작하는 것은 클라이언트의 책임이므로 유념해 사용해야 한다.

(만일 pass-by-value로 복사한 값이 아닌 `ArrayList.size` 필드로 직접 원소가 추가될 인덱스를 계산했다면, element와 size의 불일치 문제가 발생했을지도 모른다. 가령, `elementData[size++] = e;`처럼 구현됐다면, 아래 그림에서는 실제 원소는 하나임에도 size=2가 되었을 것이다.)
![thread-race](/img/arraylist/thread-race.png)

### 배열 확장 : grow()와 growth 정책
- 시간복잡도 O(N)

`grow()` 메서드는 필요한 최소 capacity를 인자로 넘겨받는다. 배열 capacity가 0인 경우와 그렇지 않은 경우로 분기한다.  

- `capacity=0`인 경우, `DEFAULT_CAPACITY`=10인 새 배열을 생성한다. lazy-allocation을 적용해 최적화한 시도이다. 예를 들어, `new ArrayList<>();`로 `ArrayList`를 생성했다면, 실제 `elementData[]` 배열이 10의 크기로 할당되는 시점이 `grow()` 호출까지 미뤄지는 것이다.
- `capacity>0`인 경우, 확장된 크기의 새 배열을 만들고, 기존 배열의 값을 새 배열에 복사한다. 새 배열의 capacity는 기존 capacity의 150%로 설정되어있다. (`ArraysSupport.newLength()` 참고)

growth 자체의 시간복잡도는 O(N)이다. 그렇다고 해서, `add()` 또한 O(N)의 복잡도를 갖는다고 볼 수는 없다. growth가 발생하는 상황은 capacity가 부족할 경우이며, 간헐적으로 나쁜 시간복잡도를 가지게 된다. growth의 시간적 비용을 '분할 상환'하게 되면, 상수 비용으로 볼 수 있다고 알려져 있다. ([amortized O(1)](https://medium.com/@satorusasozaki/amortized-time-in-the-time-complexity-of-an-algorithm-6dd9a5d38045))

```java
  private Object[] grow() {
    return grow(size + 1);
  }
  
  private Object[] grow(int minCapacity) {
    int oldCapacity = elementData.length;
    if (oldCapacity > 0 || elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        int newCapacity = ArraysSupport.newLength(oldCapacity,
                minCapacity - oldCapacity, /* minimum growth */
                oldCapacity >> 1           /* preferred growth */);
        return elementData = Arrays.copyOf(elementData, newCapacity);
    } else {
        return elementData = new Object[Math.max(DEFAULT_CAPACITY, minCapacity)];
    }
  }
```

### 배열 축소 : trimToSize()
- 시간복잡도 O(N)

`trimToSize()`는 배열의 capacity를 현재 size로 축소하는 새 배열을 생성한다. 내부적으로 size 크기를 갖는 새 배열에 기존 배열 내용을 복사해 돌려준다.

```java
  public void trimToSize() {
    modCount++;
    if (size < elementData.length) {
        elementData = (size == 0)
          ? EMPTY_ELEMENTDATA
          : Arrays.copyOf(elementData, size);
    }
  }
```

### 원소 삭제 : ArrayList.remove(), clear()
- 시간복잡도 O(N)

원소를 삭제하는 `remove() `는 `elementData[]`를 순차적으로 탐색하면서, 파라미터로 받은 객체와 처음으로 일치하는 element를 배열에서 삭제한다. 구현은 배열 순회을 순회하고 일치하는 객체의 인덱스를 반환하며, 배열에서 삭제하는 과정은  내부적으로 `fastRemove()`를 통해 구현된다. `fastRemove()`의 구현은 in-place copy로 구현되어 있다. '삭제할 인덱스 + 1'부터 끝까지 복사해, '삭제할 인덱스' 위치에서 덮어쓴다. 이후, 마지막 원소를 삭제한다. 

```java
  public boolean remove(Object o) {
    final Object[] es = elementData;
    final int size = this.size;
    int i = 0;
    found: {
        if (o == null) {
            for (; i < size; i++)
                if (es[i] == null)
                    break found;
        } else {
            for (; i < size; i++)
                if (o.equals(es[i]))
                    break found;
        }
        return false;
    }
    fastRemove(es, i);
    return true;
  }

  private void fastRemove(Object[] es, int i) {
    modCount++;
    final int newSize;
    if ((newSize = size - 1) > i)
        System.arraycopy(es, i + 1, es, i, newSize - i);
    es[size = newSize] = null;
  }
```

clear()는 배열 전체를 순회하며 null을 대입한다. O(N)의 시간복잡도를 가지게 된다.

```java
  public void clear() {
    modCount++;
    final Object[] es = elementData;
    for (int to = size, i = size = 0; i < to; i++)
        es[i] = null;
  }
```

### 복수 원소 삭제 : removeAll(), retainAll()
- 시간복잡도 O(N) * ?

`removeAll()`은 파라미터로 받은 컬렉션에 포함된 모든 element를 삭제한다. 반대로 `retainAll()`은 컬렉션에 포함된 element를 제외한 나머지를 삭제한다. 둘 다 내부적으로 `batchRemove()`를 호출하며, `boolean complement`의 평가값은 파라미터 컬렉션 c.contains()의 기대 결과이다. 서로 반대의 complement를 전달하고 있다.

`batchRemove()`는 `elementData[]`를 순회하며, 이 원소 `e`를 `c.contains(e)`로 평가한다. 따라서 시간복잡도가 `c.contains()`에 의존하며, O(N) * O(c.contains())로 볼 수 있다. 만일 N-크기의 `elementData[]`와 M-크기의 `List c`라면, 시간복잡도는 O(NM)이 될 것이지만, c의 자료구조에 따라 변할 수 있다.

```java
    public boolean removeAll(Collection<?> c) {
      return batchRemove(c, false, 0, size);
    }

    public boolean retainAll(Collection<?> c) {
      return batchRemove(c, true, 0, size);
    }
```