---
title: Virtually mapped kernel stacks By Jonathan Corbet
updated: 2017-07-04 23:37
---

**NOTE:** Enjoy!
https://lwn.net/Articles/692208/

The kernel stack in Linux is arguably a weak point in the system's design: it is small enough that kernel developers must constantly be aware of what they put on the stack to avoid overflows. But those overflows happen anyway, even in the absence of an attacker who is trying to force the issue — and, as Jann Horn recently demonstrated, there are reasons why attackers might want to force a stack overflow. When an overflow does occur, the kernel is poorly placed to even detect the problem, much less act on it. The stack has changed little over the lifetime of the kernel, but some recent work has the potential to, finally, make the kernel stack much more robust.

리눅스 커널 스택은 시스템 디자인에서 틀림 없이 취약한 부분이다. 
커널 개발자들은 오버플로우를 예방하기 위해 스택에 넣는게 무엇인지 끊임없이 주의해야 할만큼 작은 공간이다. 
그러나 이 오버플로우들은 발생하도록 공략하는 공격자가 없더라도 결국 발생하며, 
Jann Horn 이 최근 설명은  공격자가 스택 오버플로우를 공략하는 이유를 보여주고 있다.
오버플로우가 발생하게 되면, 커널은 이 문제를 감지하는 것 조차 어렵게 되며, 더군다나 그에 따른 동작을 하지 못하게 된다.
스택은 커널의 생존시간 을 넘어 약간 변경되었으며, 최근의 작업은 커널 스택을 좀더 견고하게 만들수 있는 잠재력을 가지고 있다.

How current kernels stack up
현 커널 스택은 어떻게 증가 하는가 ?

Each process has its own stack for use when it is running in the kernel; in current kernels, that stack is sized at either 8KB or (on 64-bit systems) 16KB of memory. The stack lives in directly-mapped kernel memory, so it must be physically contiguous. That requirement alone can be problematic since, as memory gets fragmented, finding two or four physically contiguous pages can become difficult. The use of directly mapped memory also rules out the use of guard pages — non-accessible pages that would trap an overflow of the stack — because adding a guard page would require wasting an actual page of memory.

각 프로세스들은 커널에서 동작시 사용하기 위한 프로세스 소유의 스택이 존재하며, 스택의 크기는 8KB 이거나 16KB (64 비트 시스템) 이다.
스택은 Directly-mapped kernel memory 에 존재하며, 물리적으로 연속된 공간에 위치해야 한다.
이 요구사항만으로도 (메모리는 파편화되는 이유로) 물리적으로 연속된 2개 혹은 4개 페이지들을 찾는 것은 어려워 질수 있기 때문에 문제가 된다. 
Guard Page 를 추가하는 것은 메모리의 페이지가 낭비 되는 이유로 직접 맵핑된 메모리의 사용은 Guard Pages 사용을 못하게 한다.
(Guard Page 는 스택 오버플로우를 감시하는 접근 불가 페이지 이다.)

As a result, there is no immediate indication if the kernel stack has overflowed. Instead, a stack that grows too large simply overwrites whatever memory is located just below the allocated range (below because stacks grow downward on most architectures). There are options to detect overflows by putting canaries on the stack, and development options can track stack usage. But if a stack overflow is detected at all on a production system, it is often well after the actual event and after an unknown amount of damage has been done.

결과적으로, 커널 스택이 오버플로우 되었을때 즉각적인 징후는 없다.
대신에, 너무 크게 증가된 스택은 할당된 범위 바로 밑에 위치하는 어떠한 메모리를 덮어 쓰게 된다.
스택에 캐너리를 넣음으로써 오버플로우를 감지할수 있으며, 개발 옵션은 스택 사용을 추적 할 수 있다.
그러나 만약 스택 오버플로우가 실제 시스템에서 모두 감지 된다면, 이미 오버플로우가 발생되고 알수 없을 만큼의 피해가 발생된 이후 이다.

For added fun, there is also a crucial data structure — the thread_info structure — placed at the bottom of the stack area. So if the kernel stack overflows, the thread_info, which provides access to almost everything the kernel knows about the running process, will be overwritten first. Needless to say, that makes stack overruns even more interesting to attackers; it's hard to know what will be placed below the stack in memory, but the thread_info structure is a known quantity.

재미 삼아,  중요한 데이터 구조체가 (가령 thread_info 구조체) 스택 영역의 바로 밑에 위치하고 있다.
커널 스택이 오버플로우 된다면 동작중인 프로세스에 대해 커널이 알고 있는 모든 것에 접근 하도록 제공하는 thread_info 가 제일 먼저 덮어 쓰여진다.
말할 필요도 공격자들이 흥미로 할만한 점이다.
메모리의에 존재하는 스택의 바로 밑에 무엇이 존재하는지 알기 어려우나 thread_info 구조체의 크기는 알수 있다.

It is not surprising, then, that kernel developers work hard to avoid stack overflows. On-stack allocations (usually in the form of automatic variables) are examined closely, and, as a general rule, recursion is not allowed. But surprises can come in a number of forms, from a careless variable declaration to unexpectedly deep call chains. The storage subsystem, where filesystems, storage technologies, and networking code can be stacked up to arbitrary depths, is particularly prone to such problems. This sort of surprise led to the expansion of the x86-64 kernel stack to 16KB for the 3.15 release, but there are limits to how big the kernel stack can be. Since there is one stack for every process in the system, any increase in its size is felt many times over.

놀랄만한 점은 아니며, 커널 개발자들은 스택 오버플로우를 예방하기 위해 노력을 한다.
스택에서의 할당은 면밀하게 조사되어 지며, 일반적인 원칙은 리컬전 을 허용하지 않는 것이다.
그러나 놀라운점은 형태의 개수에 의해 가능하며, 부주의한 변수의 선언으로 부터 예상치 못한 깊은 콜 체인들이다.
이 방식들이 x86-64 커널 스택의 팽창을 이끌게 되지만, 커널 스택이 얼마나 커질지에 대한 제한이 존재 한다.
시스템에서 모든 프로세스 에 대한 하나의 스택이 있기 때문에, 스택의 크기가 증가하는 것은 많은 시간을 넘기게 된다.

The problem of avoiding stack overflows is likely to remain a challenge for kernel developers for some time, but it should be possible for the kernel to respond better when an overflow does happen. The key to doing so, as can be seen in Andy Lutomirski's virtually mapped stacks patch set, is to change how kernel stacks are allocated.

스택 오버플로우를 예방하는 문제는 꽤 오랜 시간 동안 커널 개발자에게는 난재로 남을 것 같다.
그러나 오버플로우가 발생 했을때 커널이 이를 대처 할수 있는 가능성이 있다.
그 실마리는 Andy Lutomirski 의 Virtually Mapped Stacks 패치를 통해서 커널 스택을 할당하는 방법을 변경하는 것이다.

Virtually mapped stacks

Virtually mapped Stacks

Almost all memory that is directly accessed by the kernel is reached via addresses in the directly mapped range. That range is a large chunk of address space that is mapped to physical memory in a simple, linear fashion, so that, for all practical purposes, it looks as if the kernel is working with physical memory addresses. On 64-bit systems, all of memory is mapped in this way; 32-bit systems do not have the ability to fully map the amount of memory found in current systems, so more complicated games must be played.

커널에 의해 직접 접근 되는 거의 모든 메모리는 직접 맵핑된 범위의 주소를 통해 접근 한다.
이영역은 완전 선형적인 방법으로 맵핑되어 있 주소 공간의 큰 덩어리 이다.
결과적으로 커널은 물리 메모리 주소로 작동하고 있는 것처럼 된다. 
64 비트 시스템에서 모든 메모리는 이 방법으로 맵핑하며, 32 비트 시스템은 현재 시스템에 적용된 모든 메모리를 완전 맵핑 할 수 없다.
그러므로 좀더 복잡한 방법이 사용 된다.


Linux is a virtual-memory system, though, and so the kernel uses virtual addresses to reach memory, even in the directly mapped range. As it happens, the kernel reserves another range of addresses for virtually mapped memory; this range is used when memory is allocated with vmalloc() and is, consequently, called the "vmalloc range." Allocations in this range are pieced together a page at a time and are not physically contiguous. Traditionally, the use for this range is to obtain a relatively large chunk of memory that needs to be virtually contiguous, but which can be physically scattered.

리눅스는 가상 메모리 시스템이이며, 직접 맵핑된 영역 조차도 커널은 메모리 접근을 위해 가상 주소를 사용한다.
공교롭게도 커널은 가상 메모리를 위해 다른 주소 범위를 예약한다. 이 영역은 메모리가 vmlloc() 에 의해 할당 될때 사용된다. 이것을 vmalloc range 라 한다.
이 영역에서의 할당은 한번에 한페이지의 조각 이며, 물리적으로 연속되어 있지 않다. 
전통적으로 이 영역의 사용은 가상적으로 연속될 필요가 있으나 물리적으로는 분리되어 있는 메모리의 큰 덩어리를 획득 하게 위함이다. 

There is (almost! — see below) no need for kernel stacks to be physically contiguous, so they could, in principle, be allocated as individual pages and mapped into the vmalloc area. Doing so would eliminate one of the biggest uses of larger (physically contiguous) allocations in the kernel, making the system more robust when memory is fragmented. It also would allow the placement of no-access guard pages around the allocated stacks without the associated memory waste (since all that is required is a page-table entry), allowing the kernel to know immediately if it ever overruns an allocated stack. Andy's patch does just this — it allocates kernel stacks from the vmalloc area. While he was at it, he added graceful handling of overflows; a proper, non-corrupt oops message is printed, and the overflowing process is killed.

커널 스택을 위해 물리적으로 연속될 필요는 없으니, 원칙적으로 페이지별 할당 가능하며 vmalloc 공간에 맵핑하는 것이 가능하다.
이렇게 함으로써, 커널에서 크게 할당된 것의 가장큰 사용 중 하나를 제거 할 수 있으며, 메모리를 파편화 하는 것이 시스템을 좀더 견고하게 만들어준다.
또한, 사상된 메모리의 낭비 없이 할당된 메모리 주변에 no-access guard pages 의 배치를 허용 한다. ( 요청되는 모든 것들은 페이지 테이블 엔트리 임으로 )
할당된 스택에서 발생하는 오버플로우는 즉시 커널이 알수 있도록 허용 한다. 
Andy 의 패치는 커널 스택을 vmalloc 영역으로 부터 할당 하도록 하며, non-corrupt oops 메시지를 출력하도록 하여 매끄럽게 처리 하였으며, 오버플로우된 프로세스를 종료토록 한다.

The patch set itself is relatively simple, with most of the patches dealing with the obnoxious architecture-specific details needed to make it work. It seems like a significant improvement to the kernel, and the reviews have been generally positive. There are a few outstanding issues, though.

이 패치는 상대적으로 간단하며, 이와 같이 동작하도록 필요로 하는 아키텍처 전용의 패치로 이루어져 있다.
커널에게는 상당이 개선된 것으로 보이며, 리뷰는 긍정적이다. 그렇지만 몇몇 해결되지 않은 문제들이 있다.

Inconvenient details

불편한 설명

One of those is performance; allocating a stack from the vmalloc area, Andy says, makes creating a process with clone() take about 1.5µs longer. Some workloads are highly sensitive to process-creation overhead and would suffer with this change, so it is perhaps unsurprising that Linus responded by saying that "that problem needs to be fixed before this should be merged." Andy thinks that much of the cost could be fixed by making vmalloc() (which has never been seriously optimized for performance) faster; Linus, instead, suggests keeping a small, per-CPU cache of preallocated stacks. He has, in any case, made it clear that he wants the performance regression dealt with before the change can go in.

성능이다 , Andy 는 vmalloc 영역의로부터 stack 를 할당하는 것은 clone() 에 의한 프로세스 생성을 1.5 microsecond 정도 소모하도록 한다고 한다.
프로세스 생성에게 약간의 부하는 꽤 민감한 문제이다. 이 변화로 손해를 보게 될수 있다. Linus 는 코드가 적용되기 이전에 이문제를 해야해야 할 필요가 있다고 답하였다.
Andy 는 vmalloc() 를 빠르게 만든다면 이 문제는 고쳐질 것이라 생각 한다.  Linus 는 대신에 미리 할당된 스택들의 CPU 캐쉬를 작게 유지 할 것을 제안한다.
그는 어떠한 경우라도 변화 이전에 성능 감소 처리 방안을 명확하게 해야 한다고 한다.

Another potential cost that has not yet been measured is an increase in translation misses. The directly mapped area uses huge-page mappings, so the entire kernel (all of its code, data, and stacks) can fit in a single translation lookaside buffer (TLB) entry. The vmalloc area, instead, creates another window into memory using single-page mappings. Since references to kernel stacks are common, the possibility of an increase in TLB misses is real if those stacks are reached via the vmalloc area.

아직 판단되지 않은 잠재적 손실은 변환 실패가 증가 되는 점이다. 직접 맵핑된 영역은 대규모 페이지 맵핑을 사용 함으로, 커널의 전체 영역은 단일 TLB 항목에 적합할 수 있다.
대신에 vmlloc 영역은 단일 페이지 맵핑을 사용하는 메모리에 다른 원도우를 생성한다.
커널 스택들을 참조하는 것은 일반적이기 때문에 TLB 실패 증가의 가능성은 사실 발생한다. 이 스택들이 vmalloc 영역을 통해 도달 한다면...

One other important little detail is that, while allocations from the vmalloc area include guard pages, those pages are placed after the allocation. For normal heap memory, that is where overruns tend to happen. But stacks grow downward, so a stack overrun will overwrite memory ahead of the allocation instead. In practice, as long as a guard page is placed at the beginning of the vmalloc area, the current code will ensure that there are guard pages between each pair of allocations, so the pre-allocation page should be there. But, given that the guard pages are one of the primary goals of the patch set, some tweaks may be needed to be sure that they are always placed at the beginning of each stack.

또다른 중요한 점은 vmalloc 영역으로부터의 할당은 guard pages 를 포함하고 있기 때문에, 이 페이지들은 할당 이후에 배치된다. 
일반적인 힙 메모리의 경우, 오버런이 발생되는 경향이 있다.
그러나 스택들은 아래 방향으로 증가하며, 스택 오버런은 할당에 앞서 발생하게 될 것이다.
실상황에서 guard page 가 vmalloc 영역의 시작 위치에 배치되는 한 현재 코드에서 각 할당된 영역 사이에 guard pages 들이 위치하게 된다.
따라서 미리 할당된 페이지가 존재해야한다. 그러나 주어진 guard pages 들은 patch 의 중요한 목표중 하나이며, 각 스택의 앞 부분에 배치되도록 약간의 개조가 필요하다. 

Memory mapped into the vmalloc range has one specific constraint: it cannot easily be used for direct memory access (DMA) I/O. That is because such I/O expects a memory range to be physically contiguous, and because the virtual-to-physical mapping address functions do not expect addresses in that range. As long as no kernel code attempts to perform DMA from the stack this should not be a problem. DMA from the stack is problematic for other reasons, but it turns out that there is some code in the kernel that does it anyway. That code will have to be fixed before this patch can be widely used.

vmalloc 영역에 맵핑된 메모리는 하나의 특별한 제약상을 갖는데, DMA I/O 를 위해 쉽게 사용될 수 없다.
몇몇 I/O 는 메모리 영역이 물리적으로 연속적인 것을 예상한다. 그리고 가상에서 물리 메모리 주소로 맵핑 하는 기능은 이러한 영역의 주소를 기다하지 않는 때문이다.
스택으로 부터 DMA 를 수행하는 커널 코드가 없는한 이점은 문제되지 않는다.
스택으로 부터의 DMA 는 다른 이유들로 문제가 될 수 있다. 그러나 커널에 몇몇 코드는 결국에 이를 수행하게 되어 있고
그 코드들은 이 패치가 넓게 사용되기 이전에 수정되어야 한다.

Finally, kernels with this patch set applied will detect an overflow of the kernel stack, but there is still the little problem of the thread_info structure living at the bottom of each stack. An overrun that overwrites only this structure, without overrunning the stack as a whole, will not be detected. The proper solution here is to move the thread_info structure away from the kernel stack entirely. The current patch set does not do that, but Andy has said that he intends to tackle that problem once these patches are accepted.

마지막으로 이패치가 적용된 커널은 커널 스택의 오버플로으를 감지할 하겠지만, 각 스택의 하단 부분에 위치하게 되는 thread_info 구조체의 문제는 여전히 존재 한다.
전체적으로 스택을 오버플로하는 것없이 오직 이 구조치를 덮어 쓰는 오버런의 경우 감지 되지 않을 것이다.
여기 정확한 해결책은 전체적으로 커널 스택으로부터 thread_info 구조체를 옴기는 것이다. 
현재 패치 셋은 이러하지 못하지만 Andy 는 일단 이 패치들이 수락되면 문제를 해결할 의도를 가지고 있다.


That acceptance seems likely once the current problems have been dealt with. Giving the kernel proper detection and handling of stack overruns will remove an important attack vector and simply make Linux systems more robust. It is hard to complain about changes like that.

수락하는 것은 일단 현재 문제를 해야 하는 것처럼 보인다. 스택 오버런을 감지하고 처리하는 커널은 리눅스 시스템을 좀더 견고하게 만들 것이며, 주요한 공격 벡터를 제거할 것이다.
이러한 변화에 대해 불말을 표하긴 어려울 것이다.

## Typography Elements in One

Let's start with a informative paragraph. **This text is bolded.** But not this one! _How about italic text?_ Cool right? Ok, let's **_combine_** them together. Yeah, that's right! I have code to highlight, so `ThisIsMyCode()`. What a nice! Good people will hyperlink away, so [here we go](#) or [http://www.example.com](http://www.example.com).

<div class="divider"></div>

## Headings H1 to H6

# H1 Heading

## H2 Heading

### H3 Heading

#### H4 Heading

##### H5 Heading

###### H6 Heading

<div class="divider"></div>

## Footnote

Let's say you have text that you want to refer with a footnote, you can do that too! This is an example for the footnote number one [^1]. You can even add more footnotes, with link! [^2]

<div class="divider"></div>


**NOTE:** This theme does NOT support nested blockquotes.

<div class="divider"></div>

## List Items

1. First order list item
2. Second item

* Unordered list can use asterisks
- Or minuses
+ Or pluses

<div class="divider"></div>

## Code Blocks

```javascript
var s = "JavaScript syntax highlighting";
alert(s);
```

```python
s = "Python syntax highlighting"
print s
```

```
No language indicated, so no syntax highlighting.
But let's throw in a <b>tag</b>.
```

<div class="divider"></div>

## Mathematics

The theme comes ready with [mathjax](https://www.mathjax.org/) support built in, allowing for both simple inline equations like $$ax^2 + bx + c = 0$$ and much more complex mathematical expressions such as equation $$\eqref{eq:sample}$$ below.

$$
\begin{align}
\nabla \times \vec{\mathbf{B}} -\, \frac1c\, \frac{\partial\vec{\mathbf{E}}}{\partial t}  &= \frac{4\pi}{c}\vec{\mathbf{j}} \\   
\nabla \cdot \vec{\mathbf{E}} &= 4 \pi \rho \tag{2} \label{eq:sample}\\
\nabla \times \vec{\mathbf{E}}\, +\, \frac1c\, \frac{\partial\vec{\mathbf{B}}}{\partial t}  &= \vec{\mathbf{0}} \\
\nabla \cdot \vec{\mathbf{B}}  &= 0\\
\end{align}
$$

<div class="divider"></div>


## Table

### Table 1: With Alignment

| Tables        | Are           | Cool  |
| ------------- |:-------------:| -----:|
| col 3 is      | right-aligned | $1600 |
| col 2 is      | centered      |   $12 |
| zebra stripes | are neat      |    $1 |

### Table 2: With Typography Elements

Markdown | Less | Pretty
--- | --- | ---
*Still* | `renders` | **nicely**
1 | 2 | 3

<div class="divider"></div>

## Horizontal Line

The HTML `<hr>` element is for creating a "thematic break" between paragraph-level elements. In markdown, you can create a `<hr>` with any of the following:

* `___`: three consecutive underscores
* `---`: three consecutive dashes
* `***`: three consecutive asterisks

renders to:

___

---

***

<div class="divider"></div>

## Media

### YouTube Embedded Iframe

<iframe width="560" height="315" src="https://www.youtube.com/embed/n1a7o44WxNo" frameborder="0" allowfullscreen></iframe>

### Image

![Minion](http://octodex.github.com/images/minion.png)

[^1]: Footnote number one yeah baby! Long sentence test of footnote to see how the words are wrapping between each other. Might overflowww!
[^2]: A footnote you can link to - [click here!](#)
