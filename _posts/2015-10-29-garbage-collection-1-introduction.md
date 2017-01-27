---
layout: post
title: "Garbage Collection 1 - Introduction"
description: ""
date: 2015-10-29
categories: language
tags: ["garbage collection", "가비지 콜렉션"]
---

## Introduction

 Garbage collection(이하 GC) 이란, 앞으로 접근되지 않을 object의 할당을 해제해 줌으로써 사용할 수 있는 메모리를 확보하는 과정을 말한다. C/C++과 같은 시스템 레벨 언어에서는 GC를 지원하지 않아, 수동으로 malloc(), free()를 호출하여 메모리의 할당과 해제를 관리하지만, Java, Python, Javascript, Golang과 같은 대부분의 High-level languange 에서는 수동으로 메모리를 관리하는 불편함을 없애기 위해 GC를 언어 레벨에서 지원하고 있다.

 GC는, ***"어떤 객체가 접근하지 않을 객체인가"*** 를 판단하는 것부터 시작한다. 이를 알기 위해, 객체가 누군가에게 참조되고 있는지 아닌지 체크하고, 아무에게도 참조되고 있지 않은 객체를 해제한다.

 예를 들어, Javascript에서 객체가 선언되는 과정을 살펴보자. 새로운 Object를 선언 한 후 변수 a에 이를 저장해 보자.
 
```python 
def action():
  a = Object()
```

 그러면, Object가 만들어지면서  [Heap](https://en.wikipedia.org/wiki/Memory_management#HEAP) 영역에 새로운 메모리가 할당되고, 이 메모리에 대한 주소값이 a에 저장된다. 그 이후,

```python 
  ...
  return
```

 main 함수가 끝나면, 위에서 만든 Object를 참조하는 변수 a가 사라지면서, 더이상 해당 Object의 주소를 알 수 없게 된다. 이 때 Object 해제가 가능해진다.

 언뜻 보면 굉장히 간단하고 심플한 문제이다. 어떤 메모리 영역이 누군가에게 참조되고 있는지 아닌지를 단순히 카운트 한다음에, 참조되고 있는 곳이 사라지면 해당 메모리를 해제하면 된다.

하지만, 최근 생겨나고 있는 많은 언어들에게 GC를 어떻게 구현할 것인지는 굉장히 어렵고 큰 이슈이다. GC과정이 실제 프로그램의 동작에 영향을 주지 않아야 하는데, 위의 방법 대로 GC를 수행하려면, 

     1. 프로그램의 모든 동작을 멈추고 (stop-the-world) 할당된 변수들이 실제로 접근가능한지 파악한다
     2. 접근 불가능한 변수들을 메모리 해제한다. 

의 과정을 거쳐야 한다. 이 과정이 잦고 길어질수록 프로그램의 동작이 버벅거리고 길어진다. 그래서 최근의 GC 들은, stop-the-world를 하지 않거나, 이 시간을 최대한 줄이는 것을 목표로 한다.

 이러한 overhead에도 불구하고 GC를 언어가 지원하는 이유는 여러가지가 있다.
 
 - Memory leaks: 아무한테도 참조되지 않지만 free되지 않아 죽은 memory 영역이 되는 문제
 - Dangling pointer bugs: 여러 곳에서 참조되고 있는데, 한 곳에서 memory를 해제하여 해제된 memory를 참조하고 있는 문제
 - Double free bugs: 이미 해제된 메모리를 다시 해제하는 문제
 - 프로그래머가 memory allocation / deallocation을 신경을 쓰지 않아도 되어 해결되는 멘탈 문제

이제 앞으로 몇차례 글에서, GC의 기초적인 기술부터 최근의 GC 경향까지 소개하고자 한다.

 이번 블로그 글에서는 가장 대표적이고 기본적인 GC 방법 몇가지를 소개한다.

## Reference counting

 GC 알고리즘 중 가장 쉬운 알고리즘이다. 각각의 object에 대해, 참조가 이루어지면 +1을 하고, 참조가 없어지면 -1을 한다. 최종적으로 0이 되면 memory를 해제한다. 위의 예제에서,
 
```python 
def action():
  a = Object() # +1
  ...
  return       # -1
```

 Object에 대한 참조를 a가 들고 있어 Object의 count가 1이 된다. 이후 함수가 끝나면서 변수 a가 사라져 Object의 count가 1 줄어들어 0이 된다.

 간단해보이는 이 알고리즘은 한가지 큰 문제를 내포하고 있다.

```python
def action():
  foo = Foo()   # Foo() reference count +1
  bar = Bar()   # Bar() reference count +1
  foo.bar = bar # Bar() reference count +1
  bar.foo = foo # Foo() reference count +1
```

 foo와 bar는 서로를 참조한다. action 함수가 끝나더라도, foo와 bar가 사라짐으로써 Foo(), Bar()에 대한 reference count가 1씩 줄어들지만, 서로 참조하고 있는 count가 1이 남아있기 때문에, 이 두 object는 영원히 해제되지 않는다. 이를 cyclic reference라 한다. 이 문제를 해결하기 위해 reference counting을 쓰는 garbage collector들은 cyclic reference를 해결하는 알고리즘을 추가로 구현하고 있다. [[CPython GC]](https://docs.python.org/release/2.5.2/ext/refcounts.html)

 또는, 이 방법을 해결하기 위해 weak reference를 도입하기도 한다. weak reference란, 참조는 하지만 reference count를 늘리지 않는 pointer로, 위에서 foo.bar나 bar.foo가 weak reference라면 cycle이 발생하지 않아 문제를 해결 할 수 있을 것이다. [[Weak reference, Wikipedia]](https://en.wikipedia.org/wiki/Weak_reference)

 이외에도, reference counting 알고리즘은 여러가지 단점이 있다. 

     1. object들의 reference count를 저장할 공간을 추가로 필요로 한다.
     2. object가 추가로 참조되거나 참조해제 될때 reference count를 변경시켜야 하므로 performance가 좋지 않다.
     3. multi-thread 환경에서는 reference count를 atomic하게 변경시켜야 한다.


## Mark and sweep

 Mark and sweep 역시 굉장히 유명한 GC 방법중 하나이다. object들이 참조되는 일련의 path를 따라가면서, reference가 일어났는지 확인한다. 먼저, global reference된 object를 root set에 저장하고, 이 root set에서 참조된 object들을 돌며 참조가 되면 mark 한다. 이 mark를 위해 각 object할당 영역에 1비트의 공간을 둔다.

```python
class A(object):
  b = None

class B(object):
  c = None

class C(object):
  pass

a = A()
a.b = B()
b.c = C()
```

 이렇게 선언된 object들이 있다고 하자. Root set에는 a가 저장되어 있어 먼저 a를 mark한다. 그 후 a에서 참조하고 있는 b를 찾아가 다시 mark하고, b에서 참조하고 있는 c를 찾아가 다시 mark한다. 이를 그림으로 표현하면,

<center>![Mark and sweep, Wikipedia](https://upload.wikimedia.org/wikipedia/commons/4/4a/Animation_of_the_Naive_Mark_and_Sweep_Garbage_Collector_Algorithm.gif)</center>

 이렇게 하면 어떤 object들이 최종적으로 mark되었는지 알 수 있고, mark되지 않은 object들은 접근 불가능한(unreachable) 객체이므로 해제(sweep) 하면 된다. 이를 Mark and sweep 알고리즘이라 한다.

 이 알고리즘은 위에서 언급되었던 stop-the-world 문제를 가지고 있다. mark를 하는 중간에 프로그램이 계속 실행되어 변형이 일어나면 올바르게 mark할 수 없기 때문에 일단 프로그램의 동작이 정지되어야 한다. 이는 곧 GC가 일어날 때 마다 주기적으로 프로그램이 정지해야 하므로, 시간에 민감한 프로그램 (ex. 게임) 에서는 적합하지 않다.

## Tri-color marking

 Mark and sweep이 항상 stop-the-world를 강요하는 문제점을 가지고 있는데에 비해, Tri-color marking은 프로그램 실행중에도 marking이 가능한 알고리즘이다.

 이 알고리즘에서는 object를 3가지 색깔로 분류한다.

 * White: GC로부터 검사되지 않은 object. 더이상 접근 불가능한 object이거나 (GC 대상) 아직 검사되지 않았거나.
 * Black: 접근 가능한 object.
 * Gray: 접근 가능한 object이지만, 이 object에서 가리키는 object들은 아직 검사X

그리고 다음과 같은 순서로 진행된다.

     1. 처음 모든 object들은 white이다.
     2. root set으로부터 pointing된 object들은 gray로 색칠한다.
     3. Gray object를 하나 선택하고 Black으로 칠한다. 이 object가 가리키는 모든 object를 gray로 표시한다.
     4. Gray object가 하나도 남지 않을때까지 3을 반복한다.
     5. Gray object가 없으면 scan이 끝나고, black set은 접근 가능한 객체이며 white set은 접근 불가능한 객체이다.

 이 알고리즘의 이점은, stop-the-world없이 mark를 할 수 있다. mark를 하는 중에 object가 추가되더라도 gray로 추가되기 때문에 추후 coloring을 할 수 있고, gray가 없어지면 object들의 reference를 체크했다는 의미가 되므로 안전하게 white object들을 해제 할 수 있다. 또한, gray가 없다면 원하는 타이밍에 garbage collection을 진행할 수 있다는 장점도 있다.

 다만, 이 알고리즘의 가장 중요한 단서는 black이 white를 가리켜서는 안된다는 점이다. (block object는 이미 검사한 object이기 때문에, 여기서 reference되는 object를 다시 검사하지 않는다.) 따라서 GC 중에 mutator가 reference를 바꾸면 이 알고리즘이 작동하지 않을 수 있다. 아래의 그림을 보면 알 수 있다.

<center> ![Tri color marking, weakness](http://www.math.grin.edu/~rebelsky/Courses/CS302/99S/Presentations/GC/Tricolor.jpg)</center>

 위 그림을 보면, Step 1과 Step 2까시 순조롭게 garbage collection이 진행되다가, Step 3에서 mutator가 reference를 바꾸어 버렸다. 따라서 D(white) 를 A(black)이 가리키게 되었고, D는 해제되면 안되지만 gray가 모두 없어졌을 때 해제되게 된다.

 따라서 이 GC를 하려면, compiler의 mutator가 garbage collector와 호환되도록 구현되어야 한다. 이를 위해, 3가지 방법이 존재한다.

     1. Mutator가 white를 읽지 못하게 한다. 즉, mutator가 white를 읽는 순간, 그 object를 gray로 칠한다. (Read barrier, white를 다시 검사해야 할 대상으로 만듬)
     2. White가 black에 의해 참조되는 순간, black을 gray로 칠한다. (Write barrier, black에 의해 참조된 객체들을 다시 검사해야 할 대상으로 만듬)
     3. Garbage collection을 시작하기 전 Snapshot을 준비한다.

 (좀더 자세한 내용은 이 블로그의 범위를 넘어서니, Reference 4의 50페이지를 참고하자)

## Stop and copy

 Stop and copy 방법은,  Heap을 2개의 part로 나눈다. 하나를 fromSpace, 다른 하나를 toSpace라 한다. 새로운 object들은 fromSpace에 할당되고, fromSpace가 일정 이상 차면 프로그램이 copy가 시작된다.

     1. root에서 참조되고 있는 object들을 toSpace로 옮긴다.
     2. toSpace에서 참조되고 있는 fromSpace 객체들을 toSpace로 옮긴다.
     3. toSpace에서 fromSpace로의 참조가 없을 때까지 2를 반복한다.
     4. fromSpace의 object들을 할당 해제한다.

 이 알고리즘은 간단하고, 메모리를 새로 allocate하기 때문에 할당된 객체들이 heap space위에 모여 있게 된다. (일반적인 경우에는, heap space에서 차례대로 memory를 할당하다가 객체들이 해제되면 듬성듬성하게 free space가 남게 된다.) 이를 Heap을 compact하게 사용할 수 있다고 표현한다. 이 방법은, 이후에 Generational GC의 기초적인 아이디어로 사용된다. 
  하지만, 이 알고리즘은 heap space를 반밖에 사용할 수 없다는 단점이 있으며, copying의 cost가 존재한다.

### 마치며
 이번 글에서는, GC가 무엇인지, 그리고 기초적인 GC 알고리즘 (Reference counting, Mark-and-sweep, Tri-color marking, Stop-and-copy) 에 대해 다루었다. 이 GC 알고리즘은 현재 사용되는 다양한 언어들의 GC 시스템에 가장 중요하고 기본적인 토대가 되는 이론들이며, 이 알고리즘들에서 몇가지 문제들을 보완하여 실제로 사용되기도 한다. 다음 글에서는, 이번에 다루었던 GC들의 improvement 버전에 대해 소개한다. 


## References
 1. [Wikipedia - Garbage collection](https://en.wikipedia.org/wiki/Garbage_collection_(computer_science))
 2. [Wikipedia - Tracing garbage collection](https://en.wikipedia.org/wiki/Tracing_garbage_collection)
 3. [Garbage collection lecture - grin](http://www.math.grin.edu/~rebelsky/Courses/CS302/99S/Presentations/GC/)
 4. [Garbage collection lecture - jku](http://ssw.jku.at/Misc/SSW/02.GarbageCollection.pdf)
