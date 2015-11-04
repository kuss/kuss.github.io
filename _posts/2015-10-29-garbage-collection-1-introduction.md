---
layout: post
title: "Garbage Collection 1 - Introduction"
description: ""
date: 2015-10-29
categories: language
tags: ["garbage collection", "가비지 콜렉션"]
---

## Introduction

 Garbage collection이란, 앞으로 접근되지 않을 object의 할당을 해제해 줌으로써 사용할 수 있는 메모리를 확보하는 과정을 말한다. C/C++과 같은 시스템 레벨 언어에서는 Garbage collection이 따로 없어 수동으로 malloc(), free()를 호출하여 메모리의 할당과 해제를 관리하지만, Java, Python, Javascript, Golang과 같은 대부분의 High level languange에서는 수동으로 메모리를 관리하는 불편함을 없애기 위해 Garbage collection을 언어 레벨에서 지원하고 있다.

 기본적으로 Garbage collection의 원리는, 미래에 접근되지 않을 object를 판단하는 것부터 시작한다. 가장 쉬운 방법은 어떤 object가 누군가에게 참조되고 있는가를 판단하면, 아무에게도 참조되고 있지 않은 object는 메모리를 해제해도 된다.

 보통 어떤 메모리 영역을 할당하고, 변수에 그 메모리를 가르키도록 주소값을 할당한다.

```javascript 
func foo() {
  var a = new Object();
  ...
```

 그러면, Object가 만들어지면서 heap영역에 새로운 메모리가 할당되고, 이 메모리에 대한 주소값이 a에 저장된다. 그 이후,

```javascript 
  ...
  return null;
}
```

 foo라는 function이 끝나면, 위에서 만든 Object를 참조하는 변수 a가 사라지면서, 더이상 Object의 주소값을 알 수 없게된다. 이 때 Object가 해제가 가능해진다.

 언뜻 보면 굉장히 간단하고 심플한 문제이다. 어떤 메모리 영역이 누군가에게 참조되고 있는지 아닌지를 단순히 카운트 한다음에, 참조되고 있는 곳이 사라지면 해당 메모리를 해제하면 된다.

 그렇다면 Garbage collection을 하면 수동으로 하는 것에 비해 어떤 이점이 있을까?

  * Memory leaks: 아무한테도 참조되지 않지만 free되지 않아 죽은 memory 영역이 되는 문제
  * Dangling pointer bugs: 여러 곳에서 참조되고 있는데, 한 곳에서 memory를 해제하여 해제된 memory를 참조하고 있는 문제
  * Double free bugs: 이미 해제된 메모리를 다시 해제하는 문제
  * 프로그래머가 memory allocation / deallocation을 신경을 쓰지 않아도 되어 해결되는 멘탈 문제

최근 생겨나고 있는 많은 언어들에게 Garbage collection을 어떻게 구현할 것인지는 굉장히 어렵고 큰 이슈이다. Garbage collection과정이 실제 프로그램의 동작에 영향일 끼치지 않아야 하는데, garbage collection을 하려면 프로그램의 모든 동작을 멈추고 할당된 변수들이 실제로 접근가능한지 파악하고, 해제하는 일련의 과정을 수행해야 한다. 이를 **stop-the-world** 라 한다. stop-the-world과정이 잦고 길어질수록 프로그램의 동작이 버벅거리고 길어지기 때문에 이 stop-the-world시간을 줄이거나 없애는 것이 관건이다. 최대한 실행 중에 background에서 작업할 수 있는 것들을 하고, stop-the-world의 시간을 최소화 한다.

 이번 블로그 글에서는 가장 대표적이고 기본적인 garbage collection 방법 몇가지를 소개한다.

## Reference counting

 Garbage collection 알고리즘 중 가장 쉬운 알고리즘이다. 각각의 할당된 object에 대해, 참조가 이루어지면 +1을 하고, 참조가 없어지면 -1을 한다. 최종적으로 0이 되면 memory를 해제한다.

 간단해보이는 이 알고리즘은 한가지 큰 문제를 내포하고 있다.

```python
class Bar(object):
  foo = None

class Foo(object):
  bar = None

def action():
  foo = Foo()   # Foo() reference count +1
  bar = Bar()   # Bar() reference count +1
  foo.bar = bar # Bar() reference count +1
  bar.foo = foo # Foo() reference count +1

def main():
  ...
```

 이 프로그램에서, foo와 bar는 서로를 참조한다. 즉, action이라는 함수가 끝나더라도 foo와 bar에 대한 reference count가 1씩 남아있기 때문에, 이 두 object는 영원히 해제되지 않는다. 이를 cyclic reference라 하는데, 이 문제를 해결하기 위해 reference counting을 쓰는 garbage collector들은 cyclic reference를 해결하는 알고리즘을 추가로 구현해야 한다. [[CPython GC]](https://docs.python.org/release/2.5.2/ext/refcounts.html)

 또는, 이 방법을 해결하기 위해 weak reference를 도입하기도 한다. weak pointer란, 참조는 하지만 reference count를 늘리지 않는 pointer이다. foo.bar나 bar.foo가 weak reference라면 cycle이 발생하지 않아 문제를 해결 할 수 있을 것이다.

 이 외에도, reference counting 알고리즘은 여러가지 단점이 있다. 먼저 object들의 reference count를 저장할 공간을 추가로 필요로 하며, object가 추가로 참조되거나 참조해제 될때 reference count를 변경시켜야 하므로 속도의 문제도 발생한다. 또한, multi-thread 환경에서는 reference count를 atomic하게 변경시켜야 하므로 추가로 atomic operation의 지원이 필요하다.

## Mark and sweep

 object들이 참조되는 일련의 path를 따라가면서, 참조되었는지를 확인하는 방법이다. 먼저, globally하기 참조된 object들을 root set에 저장하고, 이 root set에서 참조된 object들을 돌면서 참조가 되면 mark한다. 이 mark를 위해 각 object할당 영역에 1비트의 공간을 둔다.

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

 ![Mark and sweep, Wikipedia](https://upload.wikimedia.org/wikipedia/commons/4/4a/Animation_of_the_Naive_Mark_and_Sweep_Garbage_Collector_Algorithm.gif)

 이렇게 하면 어떤 object들이 최종적으로 mark되었는지 알 수 있고, mark되지 않은 object들은 접근 불가능한(unreachable) 객체이므로 해제(sweep) 하면 된다. 이를 Mark and sweep 알고리즘이라 한다.

 이 알고리즘에도 해결해야 하는 문제가 있는데, stop-the-world problem이다. mark를 하는 중간에 mutation이 일어나면 올바르게 mark할 수 없기 때문에 일단 프로그램의 동작이 정지되어야 한다. 즉, GC가 일어날 때 마다 주기적으로 프로그램이 정지하므로 시간에 critical한 프로그램 (ex. 게임) 에서는 적합하지 않다.

## Stop and copy

 Heap을 2개의 part로 나눈다. 하나를 fromSpace, 다른 하나를 toSpace라 한다. 새로운 object들은 fromSpace에 할당되고, fromSpace가 일정 이상 차면 프로그램이 copy가 시작된다.

 1. root에서 참조되고 있는 object들을 toSpace로 옮긴다.
 2. toSpace에서 참조되고 있는 fromSpace 객체들을 toSpace로 옮긴다.
 3. toSpace에서 fromSpace로의 참조가 없을 때까지 2를 반복한다.
 4. fromSpace의 object들을 할당 해제한다.

 이 알고리즘은 간단하고 (marking이 필요없다), heap을 compact하게 사용할 수 있다. 하지만, copying cost가 있으며 heap size의 반 정도 밖에 활용하지 못한다는 단점이 있다.

## Tri-color marking

 Mark and sweep이 항상 stop-the-world를 강요하는 문제점을 가지고 있는데에 비해, Tri-color marking은 프로그램 실행중에도 marking이 가능한 알고리즘이다.

 이 알고리즘에서는 object를 3가지 색깔로 분류한다.

 * The white set: collector로부터 검사되지 않은 object. 즉, 더이상 접근 불가능한 object이어서 garbage collection 후보이거나, 아직 검사되지 않았거나 두가지 경우가 있다.
 * The black set: 접근 가능한 object이라고 이미 판별된
 * The gray set: 접근 가능한 object이지만, object에서 가리키는 object들은 검사하지 않음

 그리고 다음과 같은 순서로 진행된다.

 1. 처음에 root stack으로부터 pointing된 object들은 gray로 색칠되어 있다. (나머지 object는 white로 색칠되어 있다)
 2. Gray set에 있는 object를 하나 선택하고 Black으로 칠한다. 이 object가 가리키는 모든 object를 gray로 표시한다.
 3. Gray set에 object가 하나도 남지 않을때까지 2를 반복한다.
 4. Gray set에 object가 없으면 scan이 끝나고, black set은 접근 가능한 객체이며 white set은 접근 불가능한 객체이다.

 이 알고리즘의 이점은, stop-the-world없이 mark를 할 수 있다. mark를 하는 중에 object가 추가되더라도 gray set으로 추가되기 때문에 추후 coloring을 할 수 있고, gray set이 비워지면 모든 object들의 reference를 체크했다는 의미가 되므로 안전하게 white set에 있는 object들을 garbage collection 할 수 있다. 또한, gray set이 비워져 있다면 원하는 타이밍에 garbage collection을 진행할 수 있다는 장점도 있다.

 다만, 이 알고리즘의 가장 중요한 invariant는 black object가 white object를 가리켜서는 안된다는 점이다. (block object는 이미 검사한 object이기 때문에, 여기서 reference되는 object를 다시 검사하지 않는다.) 따라서 garbage collecting 중에 mutator가 reference를 바꾸면 이 알고리즘이 작동하지 않을 수 있다. 아래의 그림을 보면 알 수 있다.

 ![Tri color marking, weakness](http://www.math.grin.edu/~rebelsky/Courses/CS302/99S/Presentations/GC/Tricolor.jpg)

 위 그림을 보면, Step 1과 Step 2까시 순조롭게 garbage collection이 진행되다가, Step 3에서 mutator가 reference를 바꾸어 버렸다. 따라서 white인 D object를 black인 A object가 직접 가리키게 되었고, D는 garbage collecting 되면 안되지만 이 알고리즘에 따라 gray set이 비워졌을때 메모리가 해제되게 되었다.

 이 알고리즘의 약점을 보완하려면, mutator가 garbage collector와 호환되도록 구현되어야 한다. 이러한 상황을 막을 수 있는 3가지 방법이 있는데,

 1. Mutator가 white object를 읽지 못하게 한다. 즉, mutator가 white object를 읽는 순간, 그 object를 gray로 칠한다. (Read barrier)
 2. White object가 black object에 의해 참조되는 순간, black object를 gray로 칠한다. (Write) barrier)
 3. Garbage collection을 시작하기 전 Snapshot을 준비한다.

 (좀더 친절한 그림과 설명이 필요하다면 reference 4의 50페이지를 참고하자)

# References
 1. [Wikipedia - Garbage collection](https://en.wikipedia.org/wiki/Garbage_collection_(computer_science))
 2. [Wikipedia - Tracing garbage collection](https://en.wikipedia.org/wiki/Tracing_garbage_collection)
 3. [Garbage collection lecture - grin](http://www.math.grin.edu/~rebelsky/Courses/CS302/99S/Presentations/GC/)
 4. [Garbage collection lecture - jku](http://ssw.jku.at/Misc/SSW/02.GarbageCollection.pdf)
