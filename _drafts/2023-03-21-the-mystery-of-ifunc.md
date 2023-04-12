---
layout: single
title: The Mystery of ifunc
date: 2023-03-21 22:39 +0800
---


# Introduction

Imagine that for a function, you want to use different implementations base on runtime informations
like supported CPU features. A trivial approach might be:

```
int foo() {
	if (cpu_supports_feature_x) {
		/* Implementation 1 */
		...
	} else {
		/* Implementation 2 */
		...
	}
}
```

The drawback of this approach is that, every time `foo()` is called, the if/else branch will also be
gone through. But since the program is running on the same CPU, the branching result will always be the same.
Wouldn't it be good if it only need to determine once?

That's where ifunc (indirect function) comes to safe the world!


The pseudo code of the above example with ifunc looks like:

```

/* This line tells compiler that foo() is an ifunc, and the actual implementation is determined
   by another function calls foo_resolver.
*/
int foo() __attribute__ ((ifunc ("foo_resolver")));

int foo_imp1() {
	/* Implementation 1*/
}

int foo_imp2() {
	/* Implementation 2*/
}

void foo_reolver() {
	if (cpu_supports_feature_x) {
		return foo_imp1
	} else {
		return foo_imp2;
	}
}

```
