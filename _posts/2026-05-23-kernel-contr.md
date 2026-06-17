---
layout: post
title: "Linux Kernel Development — Kernel Contribution"
date: 2026-05-23
categories: [linux-kernel, mac0470]
tags: [linux, mac0470]
---


# Linux Kernel Contribution

## Introduction

After completing the tutorial series and setting up the development environment, the next step in the discipline *Free and Open Source Software Development* (MAC0470) was to actually contribute to the Linux kernel.

For this task, I worked together with a colleague. Our goal was to make a contribution to the kernel and go through the entire development and review process until the patch was accepted upstream.

The patch I was mainly responsible for modified the driver `drivers/iio/gyro/adxrs290.c` and can be found on Lore [here](https://lore.kernel.org/all/20260504190455.9841-1-guilhermeabreu200105@usp.br/).

At the same time, I also helped with another patch that modified `drivers/counter/intel-qep.c`, available [here](https://lore.kernel.org/all/20260512173058.14858-1-jplinaris@usp.br/).

Both contributions involved modernizing lock handling code using the newer guard lock infrastructure adopted by the kernel.

## The Idea Behind the Change

The objective of the patch was to replace traditional mutex management with the `guard()` mechanism.

Many kernel functions still follow the classic pattern of manually acquiring and releasing mutexes using `mutex_lock()` and `mutex_unlock()`. While this works perfectly fine, it often requires extra labels and `goto` statements to ensure the mutex is released correctly in every execution path.

A simplified example of the traditional approach looks like this:

```C
mutex_lock(&st->lock);

if (error_condition) {
	ret = -EINVAL;
	goto out;
}

/* function logic */

out:
mutex_unlock(&st->lock);
return ret;
```

Using `guard()`, the code becomes shorter and easier to read:

```C
guard(mutex)(&st->lock);

if (error_condition)
	return -EINVAL;

/* function logic */
```

The mutex is automatically released when execution leaves the scope, reducing boilerplate code and making mistakes less likely.

Although this type of change does not alter the driver's behavior, it improves readability and keeps the code aligned with modern kernel development practices.

## Development and Review Process

Initially, the task looked fairly simple. The driver contained several places where explicit mutex handling could be replaced by guard locks without changing functionality.

The first version of the patch was submitted after making these replacements. Everything compiled successfully and the changes seemed straightforward. However, during review, a missing return path was identified.

What made this particularly interesting was that neither the compiler nor the editor complained about it. Still, it was a mistake that should have been identified before the patch was sent for review.

After fixing the issue, new versions of the patch were submitted. The following review rounds focused mostly on coding style rather than functionality. Reviewers pointed out formatting details, spacing preferences, and other conventions commonly followed in kernel code.

At first, some of these comments felt surprisingly specific, but after a while it became clear why they matter. When a project contains millions of lines of code maintained by thousands of developers, consistency becomes extremely valuable.

In the end, the patch went through five versions before being accepted.

## An Interesting Review Suggestion

One of the most interesting pieces of feedback I received during the review process was related to how `guard()` should be used when a dedicated scope is needed.

In some situations, simply replacing `mutex_lock()` and `mutex_unlock()` with `guard()` is not enough. Since a guard lock remains active until the end of its scope, there are cases where we need to create a smaller scope explicitly so that the lock is released before the function returns.

My first instinct was to use `scoped_guard()`, which was specifically designed for this purpose and would make the intention very clear. However, the maintainer suggested using an explicit scope created with a `do { ... } while (0)` block instead.

The resulting pattern looked roughly like this:

```C
do {
	guard(mutex)(&st->lock);

	/* critical section */

} while (0);

/* code executed after the mutex is released */
```

At first, this may look unusual. After all, the loop executes exactly once. However, the purpose is not iteration but scope creation. By wrapping the guard inside a `do { } while (0)` block, the lifetime of the lock becomes explicit, ensuring that the mutex is released immediately after the block finishes.

Functionally, this achieves the same objective as `scoped_guard()`, but follows the style preferred by the maintainer for this particular driver.

This was a good example of something that is difficult to learn by simply reading documentation. The code was correct in both cases, but the review process provided insight into the conventions and preferences commonly adopted by experienced kernel developers.

## What I Learned

One thing that quickly became apparent is that contributing to the Linux kernel is not just about writing correct code.

Understanding the coding style, common patterns, and maintainer expectations is just as important. A patch can be functionally correct and still require several review rounds before it is considered ready.

Another lesson was that refactoring should never be treated as a mechanical process. Even when the goal is simply to modernize code without changing behavior, every modification needs to be reviewed carefully.

Finally, experiencing the review process firsthand was extremely valuable. Reading discussions on mailing lists is one thing; having your own code reviewed by kernel maintainers is a completely different experience.

## Conclusion

Overall, contributing to the Linux kernel was a great experience. Besides learning more about kernel development itself, I gained a much better understanding of how large open-source projects are maintained and how the review process works in practice.

What started as a relatively small cleanup patch ended up teaching lessons about software quality, maintainability, collaboration, and attention to detail that go far beyond this specific contribution.

I would also like to thank FLUSP for providing the tutorials, infrastructure, and support that made this contribution possible.


[Stray_change](https://lore.kernel.org/all/20260423223958.100487-1-guilhermeabreu200105@usp.br/)

[Chatting](https://lore.kernel.org/all/20260430200515.7915-1-guilhermeabreu200105@usp.br/)