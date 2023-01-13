Java Hashmap의 put() 메서드의 내부 동작을 확인하고, 어떻게 해시코드를 활용하는지 살펴보자.

### HasgMap.putVal(...)
arguments
 * hash: key의 해시코드 (충돌 방지를 위해 일부 조작)
 * key: key
 * value: value
 * onlyIfAbsent: true라면, 같은 키에 대해 overwrite를 비허용
 * evict: if false, the table is in creation mode.
 * return: 이전 값 (없었다면 null)

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
            boolean evict) {
   Node<K,V>[] tab; Node<K,V> p; int n, i;
   
   if ((tab = table) == null || (n = tab.length) == 0) // (1)
      n = (tab = resize()).length; 
   
   if ((p = tab[i = (n - 1) & hash]) == null) // (2)
      tab[i] = newNode(hash, key, value, null);
   else {
      Node<K,V> e; K k; // (3)
      
      /* 같은 해시코드를 갖는 경우 */
      if (p.hash == hash &&
         ((k = p.key) == key || (key != null && key.equals(k)))) {
         e = p; // (4)
      }
      /* 다른 해시코드를 갖는 경우 */
      else if (p instanceof TreeNode) {
         e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value); // (5)
      }
      else {
         // collision 상황에서는 노드를 list chaining으로 관리함.
         // next node == null일때까지 순회
         for (int binCount = 0; ; ++binCount) { // (6)
            if ((e = p.next) == null) {
               // p.next == null이면 p.next에 새 노드를 생성, 이때 e == null로 유지됨
               p.next = newNode(hash, key, value, null); // (7)
               if (binCount >= TREEIFY_THRESHOLD - 1) {// -1 for 1st
                  treeifyBin(tab, hash); // (8)
               }
               break;
            }
            if (e.hash == hash &&
               ((k = e.key) == key || (key != null && key.equals(k)))) {
               // 순회 중 같은 키를 가진 노드를 만나면 e에 대한 레퍼런스를 유지하면서 escape
               // 다른 키라면 검사를 하지 않고,다음 노드 순회로 건너뜀
               break; // (9)
            }
            p = e;
         }
      }
      if (e != null) { // existing mapping for key
         // (10)
         // 이미 같은 키를 가진 노드가 있다면
         // onlyIfAbsent=false거나 이전 값이 null일 때만 값을 업데이트
         // 이후, 이전 값을 리턴
         V oldValue = e.value;
         if (!onlyIfAbsent || oldValue == null) {
            e.value = value;
         }
         afterNodeAccess(e);
         return oldValue;
      }
   }
   // (11)
   ++modCount;
   if (++size > threshold)
      resize(); // (12)
   afterNodeInsertion(evict);
   return null;
}
```

1. tab은 this.table, n은 table 배열의 크기를 의미한다. 테이블이 없다면 테이블 초기화가 수행된다. 배열 크기 초기값은 `DEFAULT_INITIAL_CAPACITY`=16로 정의되어있다. 테이블의 크기는 항상 2의 지수 크기를 가져야만 한다. 아래 2번 설명을 보면 그 이유를 알 수 있다.
2. p는 엔트리 저장을 위해 특정된 노드이고, i는 이 버킷이 테이블 배열에서 가지는 인덱스이다. i는 `(n - 1) & hash` 라는 해시함수로 해싱한 결과이다. 이 해시 함수는 n이 2의 지수일 때 `hash % n`과 동치이며, modulo 연산을 bitwise 연산으로 최적화한 표현이다. 이 해시 버킷에 할당된 노드가 없다면, 새 노드를 추가한다. 
3. else block은 버킷 p에 이미 노드가 할당된 경우 진입하게 된다. 이때 p에는 여러 버킷이 Linked list 형태로 체이닝 될 수 있다. p는 그 중 가장 첫번째를 참조하고 있다. `Node<K,V> e`는 새 엔트리를 추가하게 되는 버킷을 참조하기 위해 선언된다.
4. 이 블록은 같은 키를 가진 엔트리일 때, e = p로 참조하도록 설정한다. (1) p 노드의 해시코드와 파라미터로 받은 해시코드가 같으면서, (2) p 노드에 저장된 키와 파라미터로 받은 키의 레퍼런스가 같거나, equals() 평가값이 같을 때 진입한다. 해시코드가 같더라도 두 객체는 서로 다를 수 있기 때문에(collision), 추가로 equals()를 평가해야 한다. 이 블록에 진입하기 위해서는 두 해시코드가 같아야 한다는 점에 주목하자.
    - equals() 평가가 true라면, 두 객체는 같은 해시코드를 가져야 한다.
    - 두 객체가 다르더라도 hashCode()는 같을 수 있다.
5. p가 TreeNode 타입이라면 putTreeVal(...)을 수행하도록 한다.
6. 한 버킷에 여러 노드가 Linked list로 chaining된 상황을 처리하는 블록이다. 버킷의 모든 노드를 선형 순회하는 무한 루프를 수행한다. binCount라는 변수로 루프 횟수를 카운트하는데, 탐색한 노드 개수를 누적하기 위함이다. 
7. 루프 순회 중 마지막 노드에 도달하여 next가 null이라면, 새 노드를 생성해 next node로 연결한다. 이때, p.next에 노드를 할당하기 때문에, e는 계속 null을 참조한다.
8. 순회한 노드의 수가 `TREEIFY_THRESHOLD`를 넘어가면, 탐색 최적화를 위해 Linked list에서 red-black tree 구조로 변환한다. 기본적으로 `TREEIFY_THRESHOLD`=8로 설정되어 있다. 위의 (5)가 바로 treefied structure에서의 노드 추가를 처리하는 블록이다.
9. 루프 순회 중 동일한 키를 가진 노드에 도달하면 루프를 빠져나온다. 
10. e != null은 이미 같은 키가 버킷에 저장되어있는 경우이다. 다시말해, (4)와 (9)의 경우에서 reachable한 블록이다. 메서드 인자로 받은 `onlyIfAbsent`=false는 이미 같은 키가 저장되어 있을 때, value를 overwrite하는 옵션이다. 이후, 업데이트 되기 전의 값을 리턴한다.
11. `modCount`는 해시의 구조가 변경된 횟수를 기록한다. (10)처럼 같은 키에 대해 처리하는 경우를 제외한 모든 메서드 호출이 `modCount`를 증가시킨다.
12. 현재 HashMap에 저장된 엔트리 개수가 threshold를 넘으면, resize()를 호출한다. 이는 threshold와 capacity를 더블링한 새 HashMap을 만드는 메서드이다. 예를 들면, HashMap의 기본 threshold는 12이며, `DEFAULT_LOAD_FACTOR` = 0.75f와 `DEFAULT_INITIAL_CAPACITY`=16을 곱한 값을 갖게 된다. 12개의 데이터가 저장되면, 24의 threshold와 32의 capacity를 가진 해시맵 객체를 새로 만들어 리턴한다.  

- 더 공부해볼 것들
    - treefy가 진행되는 과정 : hashCode()가 항상 동일한 값을 리턴하도록 구현해 강제로 해시 충돌을 발생시켰을 때, HashMap doubling은 size=8일 때 진행됨. (default threshold인 12가 아닌) treefy threshold = 8과 연관이 있을 듯 함.
    - 동일 키인지 판별할 때 굳이 해시 코드를 먼저 비교하는 이유 : 
        - 다른 객체가 같은 해시코드를 가질 경우, 주소를 비교하는 것(==)으로 판별 가능. 해시 코드와 마찬가지로 정수 비교일 것 같은데, 어떤 이점이 있는지?
        - 다른 객체가 다른 해시코드를 가질 경우, 코드블록 (2)에서 처리되고, 해시코드를 비교하는 코드라인은 unreachable함.