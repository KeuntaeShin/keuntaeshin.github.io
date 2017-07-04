---
title: Virtually mapped kernel stacks By Jonathan Corbet
updated: 2017-07-04 23:37
---

**NOTE:** Enjoy!
https://lwn.net/Articles/692208/

The kernel stack in Linux is arguably a weak point in the system's design: it is small enough that kernel developers must constantly be aware of what they put on the stack to avoid overflows. But those overflows happen anyway, even in the absence of an attacker who is trying to force the issue — and, as Jann Horn recently demonstrated, there are reasons why attackers might want to force a stack overflow. When an overflow does occur, the kernel is poorly placed to even detect the problem, much less act on it. The stack has changed little over the lifetime of the kernel, but some recent work has the potential to, finally, make the kernel stack much more robust.
How current kernels stack up

Each process has its own stack for use when it is running in the kernel; in current kernels, that stack is sized at either 8KB or (on 64-bit systems) 16KB of memory. The stack lives in directly-mapped kernel memory, so it must be physically contiguous. That requirement alone can be problematic since, as memory gets fragmented, finding two or four physically contiguous pages can become difficult. The use of directly mapped memory also rules out the use of guard pages — non-accessible pages that would trap an overflow of the stack — because adding a guard page would require wasting an actual page of memory.

As a result, there is no immediate indication if the kernel stack has overflowed. Instead, a stack that grows too large simply overwrites whatever memory is located just below the allocated range (below because stacks grow downward on most architectures). There are options to detect overflows by putting canaries on the stack, and development options can track stack usage. But if a stack overflow is detected at all on a production system, it is often well after the actual event and after an unknown amount of damage has been done.

For added fun, there is also a crucial data structure — the thread_info structure — placed at the bottom of the stack area. So if the kernel stack overflows, the thread_info, which provides access to almost everything the kernel knows about the running process, will be overwritten first. Needless to say, that makes stack overruns even more interesting to attackers; it's hard to know what will be placed below the stack in memory, but the thread_info structure is a known quantity.

It is not surprising, then, that kernel developers work hard to avoid stack overflows. On-stack allocations (usually in the form of automatic variables) are examined closely, and, as a general rule, recursion is not allowed. But surprises can come in a number of forms, from a careless variable declaration to unexpectedly deep call chains. The storage subsystem, where filesystems, storage technologies, and networking code can be stacked up to arbitrary depths, is particularly prone to such problems. This sort of surprise led to the expansion of the x86-64 kernel stack to 16KB for the 3.15 release, but there are limits to how big the kernel stack can be. Since there is one stack for every process in the system, any increase in its size is felt many times over.

The problem of avoiding stack overflows is likely to remain a challenge for kernel developers for some time, but it should be possible for the kernel to respond better when an overflow does happen. The key to doing so, as can be seen in Andy Lutomirski's virtually mapped stacks patch set, is to change how kernel stacks are allocated.

Virtually mapped stacks

Almost all memory that is directly accessed by the kernel is reached via addresses in the directly mapped range. That range is a large chunk of address space that is mapped to physical memory in a simple, linear fashion, so that, for all practical purposes, it looks as if the kernel is working with physical memory addresses. On 64-bit systems, all of memory is mapped in this way; 32-bit systems do not have the ability to fully map the amount of memory found in current systems, so more complicated games must be played.

Linux is a virtual-memory system, though, and so the kernel uses virtual addresses to reach memory, even in the directly mapped range. As it happens, the kernel reserves another range of addresses for virtually mapped memory; this range is used when memory is allocated with vmalloc() and is, consequently, called the "vmalloc range." Allocations in this range are pieced together a page at a time and are not physically contiguous. Traditionally, the use for this range is to obtain a relatively large chunk of memory that needs to be virtually contiguous, but which can be physically scattered.

There is (almost! — see below) no need for kernel stacks to be physically contiguous, so they could, in principle, be allocated as individual pages and mapped into the vmalloc area. Doing so would eliminate one of the biggest uses of larger (physically contiguous) allocations in the kernel, making the system more robust when memory is fragmented. It also would allow the placement of no-access guard pages around the allocated stacks without the associated memory waste (since all that is required is a page-table entry), allowing the kernel to know immediately if it ever overruns an allocated stack. Andy's patch does just this — it allocates kernel stacks from the vmalloc area. While he was at it, he added graceful handling of overflows; a proper, non-corrupt oops message is printed, and the overflowing process is killed.

The patch set itself is relatively simple, with most of the patches dealing with the obnoxious architecture-specific details needed to make it work. It seems like a significant improvement to the kernel, and the reviews have been generally positive. There are a few outstanding issues, though.

Inconvenient details

One of those is performance; allocating a stack from the vmalloc area, Andy says, makes creating a process with clone() take about 1.5µs longer. Some workloads are highly sensitive to process-creation overhead and would suffer with this change, so it is perhaps unsurprising that Linus responded by saying that "that problem needs to be fixed before this should be merged." Andy thinks that much of the cost could be fixed by making vmalloc() (which has never been seriously optimized for performance) faster; Linus, instead, suggests keeping a small, per-CPU cache of preallocated stacks. He has, in any case, made it clear that he wants the performance regression dealt with before the change can go in.

Another potential cost that has not yet been measured is an increase in translation misses. The directly mapped area uses huge-page mappings, so the entire kernel (all of its code, data, and stacks) can fit in a single translation lookaside buffer (TLB) entry. The vmalloc area, instead, creates another window into memory using single-page mappings. Since references to kernel stacks are common, the possibility of an increase in TLB misses is real if those stacks are reached via the vmalloc area.

One other important little detail is that, while allocations from the vmalloc area include guard pages, those pages are placed after the allocation. For normal heap memory, that is where overruns tend to happen. But stacks grow downward, so a stack overrun will overwrite memory ahead of the allocation instead. In practice, as long as a guard page is placed at the beginning of the vmalloc area, the current code will ensure that there are guard pages between each pair of allocations, so the pre-allocation page should be there. But, given that the guard pages are one of the primary goals of the patch set, some tweaks may be needed to be sure that they are always placed at the beginning of each stack.

Memory mapped into the vmalloc range has one specific constraint: it cannot easily be used for direct memory access (DMA) I/O. That is because such I/O expects a memory range to be physically contiguous, and because the virtual-to-physical mapping address functions do not expect addresses in that range. As long as no kernel code attempts to perform DMA from the stack this should not be a problem. DMA from the stack is problematic for other reasons, but it turns out that there is some code in the kernel that does it anyway. That code will have to be fixed before this patch can be widely used.

Finally, kernels with this patch set applied will detect an overflow of the kernel stack, but there is still the little problem of the thread_info structure living at the bottom of each stack. An overrun that overwrites only this structure, without overrunning the stack as a whole, will not be detected. The proper solution here is to move the thread_info structure away from the kernel stack entirely. The current patch set does not do that, but Andy has said that he intends to tackle that problem once these patches are accepted.

That acceptance seems likely once the current problems have been dealt with. Giving the kernel proper detection and handling of stack overruns will remove an important attack vector and simply make Linux systems more robust. It is hard to complain about changes like that.

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
