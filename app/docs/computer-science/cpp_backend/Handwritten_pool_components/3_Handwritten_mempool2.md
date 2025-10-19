# 理论部分
内存池（Memory Pool）是一种在 **高频小内存分配场景** 下提升性能的技术。

+ 一般情况下，每次 `malloc()` 都会陷入系统调用，代价大。
+ 内存池会**一次性从系统申请一大块内存**，然后在池内“切片”分配给用户。  
这样就能避免频繁的 `malloc/free`。

## 应用场景
nginx，kv存储等

## 为什么要分区大小块内存
Q：既然小内存的大小也是不固定的，那么为什么还要区分大内存和小内存呢。

A：小内存的意义在于：存放那些小的零碎的对象等。大内存的意义是：单间，一块large只放一个大对象，有另一套管理方式。

> 下面小块内存被称为node(block)，大块内存被称为large。
>

### 原因 1：防止“浪费”
假设我们内存池每块大小是 `4096B`（4KB），  
但用户突然请求一个 `8192B` 的内存（例如存放一个 JSON、图片 buffer、结构体数组等）。

这时如果仍然从 pool 的 4KB 小块中去“切”，就会出现两个问题：

1. **不够装** —— 4KB 不足以放下。
2. **即使强行分割多个块**，也会破坏对齐与连续性。

因此，对这种超出阈值的情况，内存池不从小块池分配，  
而是**直接用 malloc() 申请一整块“独立大内存”**。

### 原因 2：防止“碎片化”
+ 内存池内的小块通常按顺序分配，每次回收一整个node，不支持回收单个小块。如果大对象生命周期过短，内存池无法单独回收那块内存，造成**空间浪费**。

### 原因 3：提升小块分配效率
区分后，小块内存（小于阈值，比如 4KB）走快速分配路径：

+ 只移动指针；
+ 不加锁；
+ 零系统调用。

大块内存（大于阈值）走独立路径：

+ 单独 malloc；
+ 在销毁时集中 free；
+ 记录在 `mp_large_s` 链表中。

这样小对象的分配不会被大对象拖慢。

## 设计图
![画板](https://cdn.nlark.com/yuque/0/2025/jpeg/43055607/1760611550626-b0ae56c7-3a6f-472a-9ec4-5b64d00bcaeb.jpeg)

对应不定长的内存分配。可以将block分为大小两类(mp_large_s和mp_node_s)。

## 代码部分
### 结构体定义
一共3个部分，下面是“分分总”的顺序。

#### 管理block的结构体mp_node_s
```c
struct mp_node_s {
    unsigned char *last;     // 指向当前node中下一处可写的位置
    unsigned char *end;      // 指向当前node被分配的内存块末尾
    struct mp_node_s *next;  // 指向下一个node
    size_t failed;           // 
};
```

#### 管理大号内存的结构体mp_large_s
因为只存一个对象，所以不需要管理对应size。只需要有个alloc头就行。

> tips: malloc/free，实际分配的内存比我们指定的大小在前面多一块管理头，malloc返回的是管理头后面可用的内存地址的初始位置，也就是管理头末尾位置+1。管理头包含
>
> + 当前块的大小；
> + 是否已使用；
> + 前后块指针（用于合并碎片）；
> + 对齐信息；
> + 有时还有校验或魔数。
>

```c
struct mp_large_s {
    struct mp_large_s *next;
    void *alloc;   // void *以适应不同类型
};
```

#### 管理整个内存池的结构体mp_pool_s
```c
struct mp_pool_s {
    size_t max;                    // 硬区分大小块内存
    struct mp_node_s *current;     // 当前node
    struct mp_large_s *large;      // 第一个large
    struct mp_node_s head[0];      // 第一个node
};
```

### 接口声明
```c
// init一个内存池
struct mp_pool_s *mp_create_pool(size_t size);
// 销毁内存池
void mp_destory_pool(struct mp_pool_s *pool);
// 内存分配
void *mp_alloc(struct mp_pool_s *pool, size_t size);
void *mp_nalloc(struct mp_pool_s *pool, size_t size);
void *mp_calloc(struct mp_pool_s *pool, size_t size);
void mp_free(struct mp_pool_s *pool, void *p);
```



### 接口实现
#### 对外接口
##### 创建内存池 mp_create_pool
初始开一个mp_pool_s和一个mp_node_s（使用传来的size）

因此mp_pool_s和第一个mp_node_s在内存上是连续的

```c
struct mp_pool_s *mp_create_pool(size_t size) {
    struct mp_pool_s *p;
    int ret = posix_memalign((void **)&p, MP_ALIGNMENT, size + sizeof(struct mp_pool_s) + sizeof(struct mp_node_s));
    if (ret) {
        return NULL;
    }
    
    p->max = (size < MP_MAX_ALLOC_FROM_POOL) ? size : MP_MAX_ALLOC_FROM_POOL;
    p->current = p->head;
    p->large = NULL;

    p->head->last = (unsigned char *)p + sizeof(struct mp_pool_s) + sizeof(struct mp_node_s);
    p->head->end = p->head->last + size;

    p->head->failed = 0;

    return p;
}
```

##### 销毁内存池 mp_destory_pool
遍历销毁large和node内存

```c
void mp_destory_pool(struct mp_pool_s *pool) {
    struct mp_node_s *h, *n;
    struct mp_large_s *l;
    // 销毁大块
    for (l = pool->large; l; l = l->next) {
        if (l->alloc) {
            free(l->alloc);
        }
    }
    // 销毁小块
    h = pool->head->next;
    while (h) {
        n = h->next;
        free(h);
        h = n;
    }
    free(pool);
}
```

##### 重置内存池 mp_reset_pool
与上面的销毁内存池不同的是：node依靠 移动last到初始为止 来 逻辑重置内存块。

```c
void mp_reset_pool(struct mp_pool_s *pool) {
    struct mp_node_s *h;
    struct mp_large_s *l;

    for (l = pool->large; l; l = l->next) {
        if (l->alloc) {
            free(l->alloc);
        }
    }

    pool->large = NULL;
    for (h = pool->head; h; h = h->next) {
        h->last = (unsigned char *)h + sizeof(struct mp_node_s);
    }
}
```

##### 对齐分配 mp_alloc
+ 判断要分配的内存size是否大于规定的max
    - 小于：判断现有node（block）中是否有足够大的空闲空间
        * 有足够空间：分配插入
        * 没有足够空间：开一个新的node
    - 大于：施行large内存分配方法

```c
void *mp_alloc(struct mp_pool_s *pool, size_t size) {
    unsigned char *m;
    struct mp_node_s *p;
    if (size <= pool->max) {
        p = pool->current;
        do {
            m = mp_align_ptr(p->last, MP_ALIGNMENT);  // 这里是与mp_nalloc的区别
            if ((size_t)(p->end - m) >= size) {
                p->last = m + size;
                return m;
            }
            p = p->next;
        } while (p);
        return mp_alloc_block(pool, size);
    }
    return mp_alloc_large(pool, size);
}
```

##### 清0分配 mp_calloc
正常分配后 memset为0

```c
void *mp_calloc(struct mp_pool_s *pool, size_t size) {
    void *p = mp_alloc(pool, size);
    if (p) {
        memset(p, 0, size);
    }
    return p;
}
```

##### 非对齐分配 mp_nalloc
与mp_alloc只有一行区别

```c
void *mp_nalloc(struct mp_pool_s *pool, size_t size) {
    unsigned char *m;
    struct mp_node_s *p;
    if (size <= pool->max) {
        p = pool->current;
        do {
            m = p->last;                        // 这里是与mp_alloc的区别
            if ((size_t)(p->end - m) >= size) {
                p->last = m+size;
                return m;
            }
            p = p->next;
        } while (p);
        return mp_alloc_block(pool, size);
    }
    return mp_alloc_large(pool, size);
}
```

##### 释放大内存 mp_free
释放指定大内存

```c
void mp_free(struct mp_pool_s *pool, void *p) {
    struct mp_large_s *l;
    for (l = pool->large; l; l = l->next) {
        if (p == l->alloc) {
            free(l->alloc);
            l->alloc = NULL;
            return ;
        }
    }
}
```

#### 内部使用
##### 新建Block小号内存 mp_alloc_block
```c
static void *mp_alloc_block(struct mp_pool_s *pool, size_t size) {
    unsigned char *m;
    struct mp_node_s *h = pool->head;
    size_t psize = (size_t)(h->end - (unsigned char *)h);
    
    int ret = posix_memalign((void **)&m, MP_ALIGNMENT, psize);
    if (ret) return NULL;

    struct mp_node_s *p, *new_node, *current;
    new_node = (struct mp_node_s*)m;

    new_node->end = m + psize;
    new_node->next = NULL;
    new_node->failed = 0;

    m += sizeof(struct mp_node_s);
    m = mp_align_ptr(m, MP_ALIGNMENT);
    new_node->last = m + size;

    current = pool->current;

    for (p = current; p->next; p = p->next) {
        if (p->failed++ > 4) { //
            current = p->next;
        }
    }
    p->next = new_node;

    pool->current = current ? current : new_node;

    return m;
}
```

##### 新建large内存 mp_alloc_large
```c
static void *mp_alloc_large(struct mp_pool_s *pool, size_t size) {
    void *p = malloc(size);
    if (p == NULL) return NULL;

    size_t n = 0;
    struct mp_large_s *large;
    // 如果链表遍历过程中有空的alloc指针（代表以前释放过），则优先这个alloc分配
    for (large = pool->large; large; large = large->next) {
        if (large->alloc == NULL) {
            large->alloc = p;
            return p;
        }
        if (n ++ > 3) break;
    }
    // 判断一下是否小块，小块走node分配路线
    large = mp_alloc(pool, sizeof(struct mp_large_s));
    // 如果分配失败
    if (large == NULL) {
        free(p);
        return NULL;
    }
    // 头插法
    large->alloc = p;
    large->next = pool->large;
    pool->large = large;

    return p;
}
```

```c
void *mp_calloc(struct mp_pool_s *pool, size_t size) {

    void *p = mp_alloc(pool, size);
    if (p) {
        memset(p, 0, size);
    }

    return p;
}
```

## 使用example
```cpp
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

#include <fcntl.h>

#define MP_ALIGNMENT       		32
#define MP_PAGE_SIZE			4096
#define MP_MAX_ALLOC_FROM_POOL	(MP_PAGE_SIZE-1)

#define mp_align(n, alignment) (((n)+(alignment-1)) & ~(alignment-1))
#define mp_align_ptr(p, alignment) (void *)((((size_t)p)+(alignment-1)) & ~(alignment-1))

int main(int argc, char *argv[]) {
    
    int size = 1 << 12;
    // size 比 MP_MAX_ALLOC_FROM_POOL 大，所以实际开的是 MP_MAX_ALLOC_FROM_POOL 大小的block(node)
    struct mp_pool_s *p = mp_create_pool(size);

    // 对齐分配 10 次 512 内存，理论上前7次在一个block里，第8次新开了一个block
    int i = 0;
    for (i = 0;i < 10;i ++) {
        void *mp = mp_alloc(p, 512);
        //      mp_free(mp);
    }

    //printf("mp_create_pool: %ld\n", p->max);
    // 验证内存对齐宏
    printf("mp_align(123, 32): %d, mp_align(17, 32): %d\n", mp_align(123, 32), mp_align(17, 32));
    //printf("mp_align_ptr(p->current, 32): %lx, p->current: %lx, mp_align(p->large, 32): %lx, p->large: %lx\n", mp_align_ptr(p->current, 32), p->current, mp_align_ptr(p->large, 32), p->large);

    int j = 0;
    for (i = 0;i < 5;i ++) {
        char *pp = mp_calloc(p, 32);
        for (j = 0;j < 32;j ++) {
            if (pp[j]) {
                printf("calloc wrong\n");
            }
            printf("calloc success\n");
        }
    }

    //printf("mp_reset_pool\n");

    // 验证大内存的分配，并free掉该大内存
    for (i = 0;i < 5;i ++) {
        void *l = mp_alloc(p, 8192);
        mp_free(p, l);
    }

    mp_reset_pool(p);

    //printf("mp_destory_pool\n");
    for (i = 0;i < 58;i ++) {
        mp_alloc(p, 256);
    }

    mp_destory_pool(p);

    return 0;
}
```



## 补充
### 手动内存对齐宏的实现
[内存对齐实现](https://www.yuque.com/u41687375/inlses/mgpawveh123hym94)

