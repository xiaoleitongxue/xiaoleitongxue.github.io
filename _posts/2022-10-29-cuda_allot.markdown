---
layout: post
title:  "CUDA Warp-level Primitive __all_sync, __any_sync和__ballot_sync"
date:   2022-10-29 22:13:55 +0800
excerpt: CUDA原语
category: CUDA
---

最近被__ballot_sync这个原语函数搞晕了，对它的理解比较模糊，故研究了一下，把它写出来供大家一起学习。

在CUAD中，在一个SMs中，只有32个线程会一起执行，这32个线程组被称为warp，中文翻译为线程束。

在一个线程束中，线程之间可以使用warp-level primitive进行通信，
```
__all_sync()
__any__sync()
__ballot_sync()
```

以上三个函数的作用都是投票，但又有所差别。

这三个函数的函数原型为

```
__all_sync(unsigned mask, predicate)
__any_sync(unsigned mask, predicate)
__ballot_sync(unsigned mask, predicate)
```

函数有两个参数

- mask：掩码
- predicate：当前线程的投票

## 函数执行过程如下图所示

### 第一步：每个线程都投票，即将predicate设为自己想要的任意值,并分别和0比较，prediction不为0时，结果为1，否则为0.

| thread_index     | 4    | 3    | 2    | 1    | 0    |
|---|---|---|---|---|---|
| lane_index       | 4    | 3    | 2    | 1    | 0    |
| predicate        | 10   | 0    | 5    | 8    | 7    |
| predicate和0比较 | 1    | 0    | 1    | 1    | 1    |

综上，第一步得到的结果为10111

### 第二步：将上一步得到的结果和mask进行比较

假如mask为11111，即31(十进制)

mask与投票结果做逐位&运算

```
11111
  &
10111
10111
```

综上，结果为result = 10111

### 第三步

根据调用函数的不同

1. 若调用__all_sync()

   result中全为1，则函数返回1，否则返回0

2. 若调用__any_sync()

   result中只要有一个是1，就返回1，否则返回0

3. 若调用__ballot_sync()

   返回result的十进制表示，10111的十进制表示为22