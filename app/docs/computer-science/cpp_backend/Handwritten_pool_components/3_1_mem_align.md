## 数值对齐
第一个参数是要对齐的数（在使用时是要对齐的内存地址），第二个参数是对齐指标（如32，代表对齐后的数/地址只能是32的倍数）。

```c
#define mp_align(n, alignment) (((n)+(alignment-1)) & ~(alignment-1))
```

## 指针对齐
由于指针不解引用，就是代表地址。所以用指针对齐来做地址对齐。

原理就是上面的数值对齐，只不过将指针（地址值）转为了size_t，位运算完成后，再转回void *

```c
#define mp_align_ptr(p, alignment) (void *)((((size_t)p)+(alignment-1)) & ~(alignment-1))
```

## 位运算公式
((n)+(alignment-1)) & ~(alignment-1)

自行带入数字体会。

## 使用场景
在做不定长内存池时，我们使用了posix_memalign来分配新的block。

> 相比于posix_memalign（用第二个参数做对齐目标），malloc也会自动对齐，只不过不像posix_memalign可以自定义的，malloc取决于系统架构。
>
> 下面是malloc在不同平台下的对齐标准。
>

| 平台 | 对齐字节 | 示例 |
| --- | --- | --- |
| 32-bit x86 | 8 字节 | 能存放 `double`、`long long` |
| 64-bit x86_64 | 16 字节 | 能存放 `long double`、SSE类型 |
| ARM64 | 16 字节 | 一致性要求更高 |
| macOS / glibc | 16 字节 | 与堆页对齐规则一致 |


在分配完整个block后，我们要在这个block块里，基于last指针分配当前所需的内存。此时就用我们自己定义的对齐宏对齐，即上面提到的mp_align_ptr。





