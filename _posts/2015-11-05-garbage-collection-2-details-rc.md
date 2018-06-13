---
layout: post
title: "Garbage Collection 2 - Details (Reference counting)"
description: ""
date: 2015-11-05
categories: language
tags: ["software engineering", "garbage collection", "programming language"]
---

{% include mathjax_support.html %}

 앞선 포스트에서 Garbage collection 알고리즘이 어떤 종류가 있는지 기본적으로 설명하였다. 이 포스트와 다음 포스트에서는 이전 포스트에서 다뤘던 Garbage collection중 Reference counting, Mark and sweep 의 좀더 자세한 알고리즘에 대해 포스팅 하겠다.

## Improvements of reference counting

 Reference counting 기법에는 크게 세가지 단점이 있다.

 1. reference / dereference 시에 항상 reference counting을 해야 함으로써 생기는 performance 문제
 2. parallelism 에서의 atomicity 문제
 3. cycle detection 문제

 이러한 문제들에 대해 어떻게 알고리즘이 improve되었는지를 알아보자.

### Dealing with inefficiency of updates

 reference / dereference 시마다 항상 counting을 해야 하므로 시간이 걸릴 뿐더러, count가 업데이트 됨으로써 cache가 작동하지 않을 수 있다. 가장 간단한 방법으로는 compiler가 비슷한 시기에 reference가 일어나는 object들을 모아 한번에 해주는 것이다. Deutsch-Bobrow는 대부분의 reference가 local variable에서 온다는 점을 착안하여, 이 때에는 counting을 하지 않고, heap에서 참조될 때에만 counting을 하도록 하였다. ^[4]

#### Deutsch-Bobrow deferred reference counting ^[5] 
 이 알고리즘의 키 아이디어는 다음과 같다.

 1. zero count인 object를 모아뒀다가 한번에 해제한다.
 2. 대부분의 reference는 local variable로 부터 생기니 stack이나 register로부터 reference 된 object들은 무시하자.

 다음의 과정으로 수행된다.

 - stack에서 reference된 object들은 count하지 않는다.
 - reference count가 0이 되면, zero count set에 저장한다.
 - zero count set이 꽉 차면, 
   - stack 을 돌면서 참조된 object의 count를 증가시킨다.
   - count가 0인 object들을 해제한다.
   - 다시 stack에 의해 참조된 object의 count를 감소시킨 후, 0이 된 object를 zero count set에 넣는다.
   - 그럼에도 불구하고 zero count set이 꽉 찼다면 zero count set을 증가시킨다.

 가장 큰 performance 향상을 보여준, 대표적인 improvement는 Levanoni and Petrank가 2001년 발표한 update coalescing method라는 알고리즘이다. 이 알고리즘의 컨셉은 많은 필요없는 reference count update를 줄이는 것인데, 예를 들어 어떤 pointer가 $O_1$, $O_2$, ..., $O_n$을 차례로 가르키게 될때, reference count를 $rc(O_1)$--, $rc(O_2)$++, $rc(O_2)$--, $rc(O_3)$++, ..., $rc(O_n)$++을 하는 것이 아니라 $rc(O_1)$--, $rc(O_n)$++을 하는 것이다. 자세한 내용은 [논문](http://www.cs.technion.ac.il/~erez/Papers/refcount.ps)을 참고하자.

 논문에서는 Levanoni and Petrank가 Java에서 해당 알고리즘을 구현하여 99%의 counter update를 줄였으며, pointer update에서의 atomic operation의 필요를 없앴다고 주장한다. 

 또한, 이후로 object의 age에 따라 (object가 언제 만들어 졌는지) 알고리즘을 다르게 구성함으로써 performance를 향상시키는 방향으로 최근의 연구가 진행되고 있다. 2003년에 발표된 Blackburn and Mckinley's ulterior reference counting method [논문](https://dx.doi.org/10.1145%2F949305.949336) 이 대표적이다. 간단하게 소개하자면, young object에 대해서는 generational collection을 구성하고 old object에 대해서는 reference counting을 수행하는 방식이다.
 
### Dealing with reference cycles 

 일단 가장 쉬운 방법은 cycle을 만들지 않는 방법이다. 이를 위해 reference count를 건드리지 않는 weak reference가 존재한다. 예를 들어 Cocoa framework에서는, parent가 child에 대한 reference는 strong reference이지만, child가 parent에 대한 reference는 weak reference로 존재하고 있다.

 reference cycle을 자동으로 제거하는 알고리즘도 다수 존재한다. 일반적으로 Graph에서 Cycle 존재 여부를 찾는 문제 역시 graph theory에서 큰 부분을 차지 하고 있는 문제인데, 여기서는 간단하게 2가지 알고리즘: 1) **The Martinez-Wachenchauzer-Lins algorithm** 2) **Bacon's method** 만 소개하도록 하겠다. (더 많은 cycle detection algorithm은 [Wikipedia - cycle detection](https://en.wikipedia.org/wiki/Cycle_detection) 항목을 참조하길 바란다.

#### Lins algorithm

 The Martinez-Wachenchauzer-Lins algorithm (Lins algorithm)은 각 node (object)에 coloring을 하면서 cycle을 파악하는 방법이다. node는 다음의 4가지 색깔로 구분한다. [6]

 - black: reference 되고 있는 object
 - white: reference 되고 있지 않은 object (메모리 해제될 대상) 
 - grey: GC에 의해 검토중인 object 
 - red: GC에 의해 검토되야 하는 object 

 reference를 증가시킬때와 감소시킬때, 각 node (object) 에 coloring을 한다. reference를 증가시킬때는 간단히 count를 증가시키로 black으로 마킹한다. reference를 감소시킬 때 count가 0이 되면, color를 white로 채우고 deallocation을 한다. 만약에 count가 0이 아니라면, red로 마킹하고 garbage collection 후보에 등록한다. 후에 garbage collection이 이루어 질때, 색깔이 red인 node에 대해 mark, sweep, collectWhite의 과정을 거친다.

```cpp
void decRef(Obj p) {
  p.count--;
  if (p.count == 0) {
    p.color = white;
    for (all sons q) decRef(q);
    dealloc(p);
  } else if (p.color != red) {
    p.color = red;
    gcList.add(p);
  }
}

void incRef(Obj p) {
  p.count++;
  p.color = black;
}

void gc() {
  do
    p = gcList.remove()
  until (p == null || p.color == red);
  if (p != null) {
    mark(p);
    sweep(p);
    collectWhite(p);
  }
}
```

 mark, sweep, collectWhite은 아래와 같이 이루어진다.
 * mark 단계
   - red node부터 시작한다.
   - 이 node를 grey로 칠하고, 참조하는 node 수만큼 count를 감소시킨다. 
   - 참조하는 node들로 따라가면서 이전 단계를 반복한다.

 이 과정을 반복하면, 각 node에서 참조하는 node 수 만큼 count를 줄일 수 있다. 즉, 만약 내가 참조되는 node와 내가 참조하는 node의 수가 같다면, count가 0이 될 것이다. 

```cpp
void mark(Obj p) {
  if (p.color != grey) {
    p.color = grey;
    for (all sons q) { q.count--; mark(q); }
  }
}
```

 sweep 과정에서는 count가 0이 된 node들을 white로 바꾼다. 이 때, count가 1 이상이라면 restore과정을 커진다. restore 과정에서는 나 자신을 black으로 바꾸고, 내가 참조하는 다른 black이 아닌 node들을 다시 restore (count 증가, black 마킹) 과정을 거친다.

```cpp
void sweep(Obj p) {
  if (p.color == grey) {
    if (p.count == 0) {
      p.color = white;
      for (all sons q) sweep(q);
    } else restore(p);
  }
}

void restore(Obj p) {
  p.color = black;
  for (all sons q) {
    q.count++;
    if (q.color != black) restore(q);
  }
}
```

 collectWhite 과정에서는 white인 node를 dealloc 한다.

```cpp
void collectWhite(Obj p) {
  if (p.color == white) {
    p.color = grey;
    for (all sons q) collectWhite(q);
    dealloc(p);
  }
}
```

#### Bacon's algorithm

 Bacon's algorithm은 이 [논문](http://researcher.watson.ibm.com/researcher/files/us-bacon/Bacon01Concurrent.pdf) 에서 자세한 내용을 볼 수 있다.
 기본적으로 좀더 빠르고 cache가 잘 작동하도록 만든 알고리즘이며, object의 reference count가 0이 아닌 어떤 value로 decrement하기 전까지 cycle이 만들어지지는 않는다는 사실에 착안하였다. root list에 있는 모든 object를 주기적으로 돌면서, 도달 가능한 cycle을 찾는다. [좀 더 발전한 알고리즘](researcher.watson.ibm.com/researcher/files/us-bacon/Paz05Efficient.pdf)을 Paz et al이 발표하였는데, 다른 operation들과 concurrent하게 작동가능하며 Lavanoni and Petrank가 발표한 update coalescing method를 사용하여 효율성을 증대시키는 알고리즘이다.

### Variants of reference counting

 Reference counting의 문제점과 performance를 해결하기 위해 좀더 근본적으로 변경시킨 다른 variants도 있다.

#### Weighted reference counting

 weighted reference counting기법은, 각 reference에 weight를 매기고, object의 referring을 count 하는게 아니라, reference의 총 weight를 count하는 방법이다. 이 방법은 reference count를 바꿀 필요가 없으므로 reference count에 접근하는 비용이 매우 클때 사용하기 용이하다. 대부분 이런 방법의 garbage collection을 사용하는 곳은 network상에 분산시스템에서 garbage collection을 구현하는 경우이다. 이를 distributed reference counting이라 하는데, 분산 시스템에서 기존의 reference count를 사용하면, 아래와 같은 문제가 발생할 수 있다.

{% include image.html path="documentation/distributed_reference_count1.png" %}

*Figure 1. Problem of distributed reference counting*

 그림에서 보면, 1에서 reference count를 요청하고 성공하여 +1이 되었지만, 2에서 ACK가 전달되지 못해 Proxy P가 3에서 재전송하였고, 4에서 ACK를 받았다. 이 과정에서 실제로 object는 한번 접근했지만 reference count를 2번 증가시키는 참사가 발생하였다. 또한 아래와 같은 경우도 발생할 수 있다.

{% include image.html path="documentation/distributed_reference_count2.png" %}

*Figure 2. Problem of distributed reference counting*

 이 경우에서도, network의 딜레이로 인해 garbage collection 타이밍을 잡기 난감해 졌다. 분산시스템에서 reference count의 가장 큰 문제점은 reference를 관리하는 곳과 사용하는 곳 사이의 네트워크 딜레이/타이밍 문제로 인해서 생긴다. 이 때 weighted reference counting 기법이 용이하게 쓰인다.

 1. Object O가 만들어지면, total weight를 매우 크게 (예를들어, 2^16) 잡는다. Object는 total weight와 partial weight를 가지고 있다.
 2. 새로운 reference가 copy되면, partial weight의 절반을 가져간다. 즉, 원래의 reference의 partial weight는 2^15가 되고, 새로 만들어지는 reference의 partial weight는 2^15가 된다.
 3. 만약 reference가 move되면, partial weight를 가져간다. 이 과정을 통해 partial weight의 합을 total weight와 같게 보존한다.
 4. reference가 사라지면, total weight에서 해당 reference가 가지고 있던 partial weight만큼을 줄인다.
 5. 원본 Object O의 Total weight가 partial weight와 같아지면, 다른 곳에서 참조하고 있지 않으므로, GC가 된다.

 이 weighted reference counting의 문제점은, reference를 destroy할 때 여전히 reference count를 접근해야한다는 사실이다. 만약 여러 reference가 한번에 지워진다면, 이 부분에서 병목이 발생할 수 있다. 그래서 몇몇 weighted reference counting 알고리즘에서는 total weight를 건드리지 않고 다른 살아있는 reference에게 partial weight를 넘기기도 한다.

 이 알고리즘은 1987년 Bevan에 의해 처음 발표되었으며, 같은해 Watson & watson에 의해 발전되었다.

#### Indirect reference counting

 Indirect reference counting 역시, 분산 시스템(distributed reference count)에서 많이 쓰이는 GC 기법이다. 이 알고리즘은 누가 해당 reference를 가져갔는지 track한다. 처음 만들어질 때 object에 2개의 reference가 저장되는데, 하나는 만들어진 쪽에 전달하고, 다른 하나는 diffusion tree, (Dijkstra-Scholten algorithm)에 전달되어, garbage collector로 부터 dead object인지 판별되는 곳에 쓰인다. 이를 통해 object가 아직 사용중인데 GC되는 것을 방지한다. TODO: 내용 추가 


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
