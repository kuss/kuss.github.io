---
layout: post
title: "Garbage Collection 3 - Details (Train algorithm, etc)"
description: ""
date: 2015-11-12
categories: 
tags: []
---

## Train algorithm

 이전 포스트에서 설명했던 Stop & Copy algorithm의 진화판 알고리즘이다. 기본 아이디어는, 
  - 메모리를 작은 block으로 나눈다.
  - stop-and-copy garbage collection을 각 block에 대해 개별적으로 진행한다.
  - block이 작으므로 더 짧게 pause를 수행할 수 있다.

 먼저, 각 heap을 segment ("car") 단위로 고정된 크기(64KBytes) 로 나눈다.

 ![Train algorithm]({{site.url}}/assets/train_algorithm1.png)

 *Figure 1: Train algorithm (jku)*

 그런 다음, GC때 하나의 semgent를 보고 live object를 다른 car로 옮긴 다음, 해당 car를 release한다. 이 알고리즘의 장점은, 하나의 car만 GC하기 때문에 굉장히 빠르다. 다만, Object가 한 car 안에 들어가지 못하는 경우는 여러 car에 나뉘어 관리되어야 한다.

 하지만 이 알고리즘에도 커버하지 못하는 영역이 있는데, car간에 죽은 cycle이 생긴 경우이다. 그래서 이 car를 묶어줄 train이라는 개념을 도입한다.

 ![Train algorithm]({{site.url}}/assets/train_algorithm2.png)

 *Figure 2: Train algorithm (jku)* 

 우리의 목표는 이 한 train안에 죽은 cycle을 다 넣는 것이다. 그 이후 만약 다른 train에서의 참조가 없으면, 이 train을 통째로 dealloc하면 된다.

 이를 위해 어떤 car에 다른 car에서 참조하는 object를 기억하는 remembered set를 도입한다. 이 한 car의 remembered set에는, 다른 pointer의 address를 저장한다.

 ![Train algorithm]({{site.url}}/assets/train_algorithm3.png)

 *Figure 3: Train algorithm (jku)* 

 이제, 어떤 한 train을 GC하려고 하자. 그러면 다음의 과정을 거친다.

 1. train의 첫번째 car를 잡고, car를 dealloc하려고 시도한다.
 2. 다른 곳에서 해당 car를 참조하지 않으면, 그 car를 dealloc한다.
 3. 다른 car에서 해당 car를 참조하고 있다면, 해당 remebered set을 보고, 참조하고 있는 train의 가장 마지막 car로 copy한다.
 4. root에서 해당 car를 참조하고 있다면, 가장 마지막 train의 가장 마지막 car로 object를 copy한다.
 5. 이 과정을 반복하여, 전체 object가 모두 옮겨지거나 dealloc될때까지 반복한다. 또는 train내부의 object를 외부에서 참조하지 않을때까지 반복한다.

 이 과정을 pseudo-code로 옮기면 다음과 같다.

```cpp
if (there are no pointers to the first train from outside this train)
  release the whole first train;
else {
  car = first car of first train;
  for (all p in rememberedSet(car)) {
    copy p obj to the last car of train(p);
    if full, start a new car in train(p);
  }
  for (all roots p that point to car) {
    copy p obj to the last car of the last train (not to the first train!);
    if full, start a new train;
  }
  for (all p in copied objects)
    if (p points to car) {
      copy pobj to last car of train(p);
      if full, start a new car in train(p);
    } else if (p points to a car m in front of car(p))
      add p to rememberedSet(m);
  release car;
}
```

#### 예제

 예제를 통해 이 과정을 좀 더 쉽게 이해해 보자. 
 <a href = "">[접기]</a>
 이 예제에서는, 한 car에는 오직 3개의 object만 들어갈 수 있다 하자.

 ![Train algorithm]({{site.url}}/assets/train_algorithm_example1.png)

 *Figure 4: Train algorithm example (jku)* 

 먼저, 첫 car에서 GC를 시작하려 한다. R은 root에서 reference 되어 있으므로 마지막 train의 마지막 car로 간다. A는 train(B)의 가장 마지막 car로 간다. C는 train(F)의 가장 마지막 car로 간다.

 ![Train algorithm]({{site.url}}/assets/train_algorithm_example2.png)

 *Figure 5: Train algorithm example (jku)* 

 그러면 해당 car는 비어있고 deallocate되게 된다.

 ![Train algorithm]({{site.url}}/assets/train_algorithm_example3.png)

 *Figure 6: Train algorithm example (jku)* 

 ![Train algorithm]({{site.url}}/assets/train_algorithm_example4.png)

 *Figure 7: Train algorithm example (jku)* 

 다음 car에서, S는 train(R)의 가장 마지막 car로 가고, 이 때 남은 공간이 없으니 새로운 car를 만든다. D는 train(C)의 마지막 car로 간다. E는 train(D)의 마지막 car로 간다.

 ![Train algorithm]({{site.url}}/assets/train_algorithm_example5.png)

 *Figure 8: Train algorithm example (jku)* 

## Generational GC

 최근에 가장 많이 쓰이는, 가장 넓은 범위를 가지는 GC 방법이다.


## GC in multi-thread system

 Multi-thread system에서 GC를 할때 문제는,

  - mutator들이 여러 thread에 있기 때문에 GC를 하기 위해서는 항상 stop-the-world가 필요하다.
  - GC를 할 수 없는 위치도 존재할 수 있다. (아래 코드에서 2번째와 3번째)

```asm
MOV EAX, objAdr
ADD EAX, fieldOffset
# GC is not allowed here, because the object may be moved
MOV [EAX], value
```

 그래서 GC를 하기 위해서 Safepoint를 판별해야 하는데, Safepoint라 함은 GC를 할 수 있는 곳이며, thread들이 멈출 수 있는 지점을 의미한다. Safepoint의 조건은 다음과 같다.

  - For every safepoint there is
    - a table of pointer offsets in the current stack frame
    - a list of registers containing pointers

 safepoint에서 thread를 멈추고 GC를 하기 위해 1. Safepoint polling 2. Safepoint patching 두가지 중 한가지 방법으로 implement를 한다.

### Safepoint polling
 
 GC가 pending중일때, mutator가 safepoint를 체크한다. 

```cpp
gcPending = true;
suspend all threads;
for (each thread t)
  if (not at a safepoint) t.resume(); // let it run to the next safepoint
// all threads are at safepoints
collect();
gcPending = false;
resume all threads;
```

 이를 위해 아래의 모든 포인트에 Compiler가 아래의 코드를 집어넣는다.

 - method 시작
 - method 끝
 - system call
 - backward jump

```cpp
if (gcPending) suspend();
```

 혹은,

```asm
MOV dummyAdr, 0 # if gc pending, dummyAddr -> readonly -> trap!
```

### Safepoint patching

 Safepoints에 thread suspend를 위한 instruction을 patch 하는 방식이다.

```cpp
suspend all threads;
for (each thread t) {
  patch next safepoint with a trap instruction;
  t.resume(); // let it run until the next safepoint
}
// all threads are at safepoints
collect();
restore all patched instructions;
resume all threads;
```

## Finalization

 GC에서 또 한가지 고려해야할 사항은 finalization이다. 일반적으로 Java에서 object가 dealloc되기 전에 해야할 행동들을 아래와 같이 finalize를 override하여 구현한다.

```java
class T {
  ...
  protected void finalize() throws Throwable {
    ... // cleanup work
     super.finalize();
  }
}
```

 이제 이와 같은 finalization을 GC과정 중에 어떻게 구현할 수 있는지 보자.

 finalize함수가 구현되면, finalization list에 추가된다. (이 때, finalization list에 추가되는 object는 pointer로 취급되지 않는다; garbage collection을 막지 않기 위함) finalize를 GC 중간에 하게 되면, stop-the-world시간이 엄청나게 길어질 수 있다. 그래서 Java나 .NET에서는, finalize를 background에서 수행한다.

 첫번째 GC에서는 finalization list에 있는 unmarked object를 돌면서 worklist에 추가한다.

```cpp
mark();
for all unmarked objects obj in the finalization list
  worklist.add(obj);
  set mark bit of obj
sweep();
```

그리고 background에서는 finalize를 호출하고, mark bit를 clear한 후 finalization list에서 삭제한다.

```
for all obj in the worklist
  obj.finalize();
  clear mark bit of obj
  remove obj from finalization list
```

두 과정을 거친 object는 다음 GC때 collect될 것이다. 하지만 finalize가 있는 경우 object dealloc이 delay됨을 확인할 수 있다.

 finalize 중간에 object가 다시 살아날 수도 있다.

```java
protected void finalize() throws Throwable {
  ...
  globalVar = this;
}
```

 이 경우 다음 GC때 이 object가 살아있음을 발견하고 GC하지 않는다. object가 다시 없어진 후에는 finalizer를 다시 부르지 않는다.

## Performance & Recent garbage collector

Performance[edit]
Performance of tracing garbage collectors – both latency and throughput – depends significantly on the implementation, workload, and environment. Naive implementations or use in very memory-constrained environments, notably embedded systems, can result in very poor performance compared with other methods, while sophisticated implementations and use in environments with ample memory can result in excellent performance.

In terms of throughput, tracing by its nature requires some implicit runtime overhead, though in some cases the amortized cost can be extremely low, in some cases even lower than one instruction per allocation or collection, outperforming stack allocation.[6] Manual memory management requires overhead due to explicit freeing of memory, and reference counting has overhead from incrementing and decrementing reference counts, and checking if the count has overflowed or dropped to zero.

In terms of latency, simple stop-the-world garbage collectors pause program execution for garbage collection, which can happen at arbitrary times and take arbitrarily long, making them unusable for real-time computing, notably embedded systems, and a poor fit for interactive use, or any other situation where low latency is a priority. However, incremental garbage collectors can provide hard real-time guarantees, and on systems with frequent idle time and sufficient free memory, such as personal computers, garbage collection can be scheduled for idle times and have minimal impact on interactive performance. Manual memory management (as in C++) and reference counting have a similar issue of arbitrarily long pauses in case of deallocating a large data structure and all its children, though these only occur at fixed times, not depending on garbage collection.

Manual heap allocation
search for best/first-fit block of sufficient size
free list maintenance
Garbage collection
locate reachable objects
copy reachable objects for moving collectors
read/write barriers for incremental collectors
search for best/first-fit block and free list maintenance for non-moving collectors
It is difficult to compare the two cases directly, as their behavior depends on the situation. For example, in the best case for a garbage collecting system, allocation just increments a pointer, but in the best case for manual heap allocation, the allocator maintains freelists of specific sizes and allocation only requires following a pointer. However, this size segregation usually cause a large degree of external fragmentation, which can have an adverse impact on cache behaviour. Memory allocation in a garbage collected language may be implemented using heap allocation behind the scenes (rather than simply incrementing a pointer), so the performance advantages listed above don't necessarily apply in this case. In some situations, most notably embedded systems, it is possible to avoid both garbage collection and heap management overhead by preallocating pools of memory and using a custom, lightweight scheme for allocation/deallocation.[7]

The overhead of write barriers is more likely to be noticeable in an imperative-style program which frequently writes pointers into existing data structures than in a functional-style program which constructs data only once and never changes them.

Some advances in garbage collection can be understood as reactions to performance issues. Early collectors were stop-the-world collectors, but the performance of this approach was distracting in interactive applications. Incremental collection avoided this disruption, but at the cost of decreased efficiency due to the need for barriers. Generational collection techniques are used with both stop-the-world and incremental collectors to increase performance; the trade-off is that some garbage is not detected as such for longer than normal.

Determinism[edit]
Tracing garbage collection is not deterministic in the timing of object finalization. An object which becomes eligible for garbage collection will usually be cleaned up eventually, but there is no guarantee when (or even if) that will happen. This is an issue for program correctness when objects are tied to non-memory resources, whose release is an externally visible program behavior, such as closing a network connection, releasing a device or closing a file. One garbage collection technique which provides determinism in this regard is reference counting.
Garbage collection can have a nondeterministic impact on execution time, by potentially introducing pauses into the execution of a program which are not correlated with the algorithm being processed. Under tracing garbage collection, the request to allocate a new object can sometimes return quickly and at other times trigger a lengthy garbage collection cycle. Under reference counting, whereas allocation of objects is usually fast, decrementing a reference is nondeterministic, since a reference may reach zero, triggering recursion to decrement the reference counts of other objects which that object holds.
Real-time garbage collection[edit]
While garbage collection is generally nondeterministic, it is possible to use it in hard real-time systems. A real-time garbage collector should guarantee that even in the worst case it will dedicate a certain number of computational resources to mutator threads. Constraints imposed on a real-time garbage collector are usually either work based or time based. A time based constraint would look like: within each time window of duration T, mutator threads should be allowed to run at least for Tm time. For work based analysis, MMU (minimal mutator utilization)[8] is usually used as a real-time constraint for the garbage collection algorithm.

One of the first implementations of hard real-time garbage collection for the JVM was based on the Metronome algorithm,[9] whose commercial implementation is available as part of the IBM WebSphere Real Time.[10] Another hard real-time garbage collection algorithm is Staccato, available in the IBM's J9 JVM, which also provides scalability to large multiprocessor architectures, while bringing various advantages over Metronome and other algorithms which, on the contrary, require specialized hardware.[11]

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
