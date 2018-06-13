---
layout: post
title: "Garbage Collection 3 - Details (Mark and sweep)"
description: ""
date: 2015-11-05
categories: language
tags: ["software engineering", "garbage collection", "programming language"]
---

{% include mathjax_support.html %}

## Improvements of Mark and Sweep

 이전 포스트에서, Mark and sweep 알고리즘에 대해 간략하게 소개하였다. 여기서는 mark and sweep 알고리즘을 구현할때의 strategy와, 많은 variants들, 그리고 improvements를 소개한다.
 먼저, Mark and sweep 알고리즘의 cost를 따져보자.

```cpp
function DFS(x) {
  if (x is a heap pointer)
    if (x is not marked) {
      mark x;
      for (i=1; i<=|x|; i++)
        DFS(x.fi)
    }
}

function Mark() {
  foreach (v in a stack frame)
  DFS(v);
}

function Sweep() {
  p = first address in heap;
  while (p < last address in heap) {
    if (p is marked)
      unmark p;
    else {
      p.f1 = freelist;
      freelist = p;
    }
    p = next object pointer after p
  }
}
```

 Mark phase에서의 cost는, O(R) 이다. (R: 접근 가능한 객체의 수)
 Sweep phase에서의 cost는, O(H) 이다. (H: heap에 있는 모든 객체의 수)
 각각의 cost를 c1*R, c2*H 라 하자. (c1은 대략 10 instruction, c2는 대략 3 instruction 정도 이다)
 이 때 Amortized cost는, 각 GC collection에서 H-R 개의 object가 반환되므로, $${(c1 * R) + (c2 * H)}\over{H - R}$$
 가 된다. 즉 GC cost를 줄이기 위해서는, H - R이 커야 하므로, R / H가 작아야 한다. 

 Mark and sweep algorithm은 mark과정을 위해 DFS를 사용한다. DFS는 보통 recursive 로 구현하는데, 문제는 path가 길어지면 길어질 수록 stack space를 많이 사용한다는 점이다. 최악의 경우 heap이 ㅎ나의 linked list로 연결되어 있다면, stack을 heap size보다 많이 사용하게 될 수도 있다. 이런 단점을 극복하기 위해, DSW Algorithm을 사용한다.
 
### DSW Algorithm

 DSW은 Deutsch-Schorr-Waite pointer reversal의 준말이며, recursive를 사용하지 않고, 이미 있는 graph의 component를 재활용하여 추가 stack memory를 사용하지 않는 알고리즘이다. 이미 mark를 위해 각 object에 추가로 1비트를 할당했듯이, 몇 bits을 추가하여 해결 가능하다.

{% include image.html path="documentation/dsw_algorithm.png" %}

*Figure 3. DSW Algorithm*

 이 알고리즘에서는 back, cur 두개의 pointer를 관리한다. back은 그동안 지나온 pointer에 대한 node를 저장하고, cur는 다음에 검사할 node를 저장한다. root pointer로 부터 돌면서, node를 mark하고 back에 해당 node를 저장한다. 그 다음, 그 node의 child에 대해 돌며 back pointer에 child를 저장하고, child가 부모를 바라보도록 한다. 이런 방식으로 recursive하지 않게 모든 node를 검사할 수 있다.

 아래는 이 알고리즘의 대략적인 C++ 구현이다.

```cpp
class Block {
  int n; // number of pointers
  int i; // index of currently visited pointer
  Block[] son; // pointer array
  ... // data
}

void mark (Block cur) {
  // assert cur != null && cur.i < 0
  Block prev = null;
  for (;;) {
    cur.i++; // mark
    if (cur.i < cur.n) { // advance
      Block p = cur.son[cur.i];
      if (p != null && p.i < 0) {
        cur.son[cur.i] = prev;
        prev = cur;
        cur = p;
      }
    } else { // retreat
      if (prev == null) return;
      Block p = cur;
      cur = prev;
      prev = cur.son[cur.i];
      cur.son[cur.i] = p;
    }
  }
}
```

### Mark and Compact

 Mark and compact는 mark and sweep 알고리즘과, cheoney's copying algorithm의 조합이라고도 할 수 있다. mark and sweep 알고리즘과 동일하되, sweep을 진행하지 않고 살아남은 object들을 memory space의 앞쪽으로 compact 시키는 방법이다. 나머지 sweep이 될 object들은 재사용을 위해 남겨둔다. 이와 같이 알고리즘을 구성했을때의 장점은,

 - fragmentation problem: Mark and sweep 알고리즘에서는 살아남은 object들이 메모리 여기저기에 흩어져 있어, 효율적으로 메모리를 사용할 수 없었다.  
 - object들의 memory상 ordering이 보장된다. 예를 들어 array의 경우, mark and sweep을 하게 되면 새로 할당하는 object에 대해 ordering이 보장되지 않을 수 있지만 mark and compact의 경우 garbage collection 이전에 memory ordering이 이후에도 보장되므로 이득이 있다.

 하지만 단점도 있다. mark and sweep에 비해 object copying이 일어나야 하므로 performance가 더 좋지 않다. 

 Mark and compact 를 구현하는 2가지 방법을 소개한다.

#### Table-based compaction

 Table-based compaction은 Haddon and Waite에 의해 1967년 처음 발표되었다. heap의 object간의 상대적 위치를 보존하면서, 적은 overhead로 compaction을 진행할 수 있다.

 low addresses부터 top addresses까지 돌면서, 처음 mark된 object를 맞닥뜨리면, 첫 available low address로 옮기고, break table에 저장한다. 모든 live object에 대해, table에는 처음 위치와 compaction 이후의 위치의 차이를 저장한다. 그림을 보면 쉽게 이해할 수 있다.

![Table-based compaction](https://upload.wikimedia.org/wikipedia/en/thumb/0/03/Table-based_heap_compaction_flattened.svg/280px-Table-based_heap_compaction_flattened.svg.png)

*Figure 4: Table-based compaction, Wikipedia*

 이 break table 역시 heap에 저장되어 있다. object가 relocate가 될 때 heap의 바닥부터 copy해 나가는데, break table이 저장되어 있는 공간을 침범하게 되면 여기에 기록된 record를 옮기게 되고, 이 때문에 break table의 order가 바뀔 수 있다. 이를 rolling the table이라고 저자가 칭했다. compaction이 끝난 후, break table의 sorting이 필요해지기 때문에 O(n log n)이 필요해 진다. 그 후 pointer들은 break table을 보고 자신의 reference를 다시 찾아가게 된다. TODO: 내용 보충

#### LISP2 Algorithm

 1. live object에 대해 location을 계산한다.
   - heap의 첫 시작점에 live pointer와 free pointer를 위치시킨다.
   - live pointer가 live object를 만나면, object를 가르키는 pointer를 free pointer로 업데이트하고, free pointer를 object size만큼 increment한다.
   - live pointer를 다음 object로 옮긴다.
   - 이런 식으로 live object가 heap의 끝까지 갈때까지 반복한다. 
 2. 모든 pointer를 업데이트 한다.
 3. object를 실제로 옮긴다.

 Tables-based compaction이 O(n log n)인데 반해, LISP2 algorithm은 더 심플하며 O(n) 알고리즘이다. 단, table-based는 n이 used space이지만, LISP2는 n이 전체 heap size이다. 따라서 경우에 따라 table-based algorithm이 더 성능이 좋을 수도 있다. LISP2 알고리즘이 구현하기에는 더 심플하다.

 아래는 간단한 pseudocode 이다.

```go
compact():
  computeLocations()
  updateReferences()
  relocate()

computeLocations():
  live := start
  free := start
  while live < end
    if marked(live):
      live->header.fwd = free
      free += live->header.size
      live += live->header.size

updateReferences(): 
  foreach obj in heap:
    if (!marked(obj)) continue
    foreach loc in obj->header.typeInfo->ptrs:
      *(obj+loc) = (*(obj+loc))->header.fwd

relocate(): 
  scan := start
  while scan < end:
    if isMarked(scan):
      memcpy(scan->header.fwd, scan, scan->header.size)
      unmark(scan->header.fwd)
    scan += scan->header.size
```

### Mark and Non-sweep (lazy sweep)

 Mark and sweep 알고리즘의 문제점은 항상 모든 block을 돌면서 sweep을 해야한다는 점이다. 또한, 한번 사용되었던 memory가 나중에 다시 재사용 되는 경우에는 free하지 않고 남겨두는 것이 유리하다. 예를 들어, virtual memory system에서는 한번 swap-out 된 page는 다시 swap-in 되기 마련이다. 그래서 Mark and sweep에서, sweep을 하지 않고 남겨두었다가 나중에 필요시 재활용하거나, sweep하는 알고리즘이다.

 1. Mark를 하고 나서, 더이상 reachable하지 않은 object라면 unused로 마크하고 추후를 위해 남겨둔다.
 2. Free list에 더이상 남은 공간이 없다면, unused인 object 일부를 sweep한다.
 3. 나중에 해당 object가 다시 사용된다면, unused인 마크만 다시 used로 바꾸면 된다.

 Lazy sweep을 구현한 크게 3가지 알고리즘을 소개한다.

 - Hughes’s lazy sweeping (1982)
 - Boehm-Demers-Weiser sweeper (1988)
 - Zorn's lazy sweeper (1989)


## References

 1. [Wikipedia - Garbage collection](https://en.wikipedia.org/wiki/Garbage_collection_(computer_science))
 2. [Wikipedia - Tracing garbage collection](https://en.wikipedia.org/wiki/Tracing_garbage_collection)
 3. [An Efficient On-the-Fly Cycle Collection](www.cs.technion.ac.il/~erez/presentations/cycle-collection.ppt)
 4. [Wikipedia - Reference counting](https://en.wikipedia.org/wiki/Reference_counting)
 5. [Cornell CS412](http://www.cs.cornell.edu/courses/cs412/2008sp/lectures/lec34.pdf)
 6. [Garbage collection lecture - jku](http://ssw.jku.at/Misc/SSW/02.GarbageCollection.pdf)
 7. [Concurrent Cycle Collection in Reference Counted Systems](http://researcher.watson.ibm.com/researcher/files/us-bacon/Bacon01Concurrent.pdf)
 8. [Wikipedia - Mark and Compact](https://en.wikipedia.org/wiki/Mark-compact_algorithm)
 9. [Gregor Institute CS842](http://the.gregor.institute/t/5n/842/slides/9.pdf)
 10. [Princeton - CS320](www.cs.princeton.edu/courses/archive/spr03/cs320/notes/gc2.ppt)
