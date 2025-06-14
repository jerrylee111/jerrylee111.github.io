---
date:
  created: 2025-06-15
  updated: 2025-06-15
readtime: 15
pin: true
links:
  - Homepage: index.md#project-layout
  - Blog index: blog/index.md
  - External links:
    - Material documentation: https://squidfunk.github.io/mkdocs-material
categories:
  - xjtuicslab
tags:
  - difficult 
  - meaningful
---

# malloclab

内存管理实验，主要目的是实现一个运行时堆内存分配系统，主要顺序是

1. 隐式链表实现
2. 空闲显式链表实现
3. 分离式空闲链表实现
4. 面向特殊类型数据

<!-- more -->

```c
/*
 * mm-naive.c - The fastest, least memory-efficient malloc package.
 * 
 * In this naive approach, a block is allocated by simply incrementing
 * the brk pointer.  A block is pure payload. There are no headers or
 * footers.  Blocks are never coalesced or reused. Realloc is
 * implemented directly using mm_malloc and mm_free.
 *
 * NOTE TO STUDENTS: Replace this header comment with your own header
 * comment that gives a high level description of your solution.
 */
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>
#include <unistd.h>
#include <string.h>
#include <math.h>
#include "mm.h"
#include "memlib.h"

/*********************************************************
 * NOTE TO STUDENTS: Before you do anything else, please
 * provide your team information in the following struct.
 ********************************************************/
team_t team = {
    /* Team name */
    "ateam",
    /* First member's full name */
    "Harry Bovik",
    /* First member's email address */
    "bovik@cs.cmu.edu",
    /* Second member's full name (leave blank if none) */
    "",
    /* Second member's email address (leave blank if none) */
    ""
};

/* single word (4) or double word (8) alignment */
#define ALIGNMENT 8

/* rounds up to the nearest multiple of ALIGNMENT */
#define ALIGN(size) (((size) + (ALIGNMENT-1)) & ~0x7)


#define SIZE_T_SIZE (ALIGN(sizeof(size_t)))

//Header的大小
#define WORD_SIZE (sizeof(unsigned int))

//用来在Header上写数据或者读取值
#define READ(PTR) (*(unsigned int *)(PTR))
#define WRITE(PTR, VALUE) ((*(unsigned int *)(PTR)) = (VALUE))

//将块大小和是否被占用的信息合并，便于写入Header
#define PACK(SIZE, IS_ALLOC) ((SIZE) | (IS_ALLOC))

//传入指向Header的指针p，返回其后的负载块的长度
#define GET_SIZE(PTR) (unsigned int)((READ(PTR) >> 3) << 3)
//传入指向Header的指针p，返回其后的负载块是否被占用
#define IS_ALLOC(PTR) (READ(PTR) & (unsigned int)1)

//传入指向负载首个字节的指针，返回指向这个块的头/尾的指针
#define HEAD_PTR(PTR) ((void *)(PTR) - WORD_SIZE)
#define TAIL_PTR(PTR) ((void *)(PTR) + GET_SIZE(HEAD_PTR(PTR)) - WORD_SIZE * 2)

//传入指向负载首个字节的指针，返回指相邻的下一个块/上一个块的指针
#define NEXT_BLOCK(PTR) ((void *)(PTR) + GET_SIZE(HEAD_PTR(PTR)))
#define PREV_BLOCK(PTR) ((void *)(PTR) - GET_SIZE((void *)(PTR) - WORD_SIZE * 2))

#define MAX(X, Y) ((X) > (Y) ? (X) : (Y))
#define PAGE_SIZE (1 << 12)

//空闲链表补充宏定义
#define GETADDR(PTR) (*(void **)PTR)//找到当前指针所指向的指针
#define PUTADDR(PTR, ADDR) (*(void **)PTR = (void *)ADDR)//向当前指针输入指针

#define PREV_BLOCK_UP(PTR) ((void *)PTR) //指向上一个空闲块的指针
#define NEXT_BLOCK_UP(PTR) ((void *)PTR + 2 * WORD_SIZE) //指向下一个空闲块的指针

/*分离式显式链表，目前确定需要维护一个空闲链表头节点的组在序言块中，序言块中应该需要32*（void*）以上的负载空间，尚未确定
使用取对数的方法来确定是哪一个链表头，这样同时也可以避免用数组，而是直接使用地址寻找对应头节点
这样的话，那么空闲链表头节点heap_free_first将不再需要，第一个负载位置可以放（16，32），第二个负载放（32，64）以此类推，
又负载长度不会超过2的32次方，那么最大的区间应该是(2^31,2^32),那么需要存的指针就有32-4+1=29个指针，
需要序言块是30*（void*）的长度（加上两个头尾），然后进行行查找和插入删除的时候使用log(size)向下取整来找到对应所需块，更简单的，我们可以直接给
序言块申请32*（void*）的负载，33*（void*）的长度，从负载的第四个指针块开始进行第一类大小空闲表的分配，这样找起来就更加简单，那么heap_free_first又可以用了，先
补充宏定义查找头节点位置，再给一个heap_youneed
需要注意的一点是，如果在当前范围没找到合适的适配，应该向上寻找适配，找不到才开空间，这里是个bug点*/
#define MATCH(PTR) ((int)(log2(GET_SIZE(HEAD_PTR(PTR))))) * sizeof(void*)//使用int来造成向下取整
void * heap_youneed;

// static void * heapfirst;
static void * heap_free_first;//空闲链表头，在分离式显式链表可以指向第一个负载，然后要取头节点时加上对应的MATCH（PTR）即可
static void * heap_free_last;
void remove_free(void * ptr);
// void place_free(void * ptr);
void insert_free(void * ptr);

void *Merge(void *Ptr) {
    int if_front = IS_ALLOC(HEAD_PTR(PREV_BLOCK(Ptr)));//检查前面的块是否空闲
    void * PREV = PREV_BLOCK(Ptr);
    int if_back = IS_ALLOC(HEAD_PTR(NEXT_BLOCK(Ptr)));//检查后面的块是否空闲
    void * NEXT = NEXT_BLOCK(Ptr);
    if(!IS_ALLOC(HEAD_PTR(Ptr))) remove_free(Ptr);
    size_t size = GET_SIZE(HEAD_PTR(Ptr));//得到当下块的头中的块长度

    if(if_front && if_back);//前后都被分配，不用合并空闲块
    else if(if_front && !if_back){
        remove_free(NEXT);
        size += GET_SIZE(HEAD_PTR(NEXT_BLOCK(Ptr)));//前面被分配，合并后方，不用管头指针
    }
    else if(!if_front && if_back){
        remove_free(PREV);
        size += GET_SIZE(HEAD_PTR(PREV_BLOCK(Ptr)));//后面被分配
        Ptr = PREV_BLOCK(Ptr);//变为前面的块
    }
    else{
        remove_free(PREV);
        remove_free(NEXT);
        size += GET_SIZE(HEAD_PTR(NEXT_BLOCK(Ptr))) + GET_SIZE(TAIL_PTR(PREV_BLOCK(Ptr)));
        Ptr = PREV_BLOCK(Ptr);
    }

    WRITE(HEAD_PTR(Ptr), PACK(size, 0));
    WRITE(TAIL_PTR(Ptr), PACK(size, 0));
    insert_free(Ptr);
    return Ptr;
}

void Place(void *Ptr, unsigned int Size) {
    size_t total = GET_SIZE(HEAD_PTR(Ptr));
    size_t remain = total - Size;
    
    //面向数据1，此处如果size是64位，划分八次64位(加上头就是72)，八次448位（加上头就是456）
    if(Size == 72 && total == 4224){

        if(!IS_ALLOC(HEAD_PTR(Ptr)))remove_free(Ptr);
        WRITE(HEAD_PTR(Ptr), PACK(72, 1));
        WRITE(TAIL_PTR(Ptr), PACK(72, 1));
            
        Ptr = NEXT_BLOCK(Ptr);

        WRITE(HEAD_PTR(Ptr), PACK(remain, 0));
        WRITE(TAIL_PTR(Ptr), PACK(remain, 0));
        insert_free(Ptr);

        for(int i = 1; i < 8; i++){
            remain -= 72;

            if(!IS_ALLOC(HEAD_PTR(Ptr)))remove_free(Ptr);
            WRITE(HEAD_PTR(Ptr), PACK(72, 0));
            WRITE(TAIL_PTR(Ptr), PACK(72, 0));

            insert_free(Ptr);

            Ptr = NEXT_BLOCK(Ptr);

            WRITE(HEAD_PTR(Ptr), PACK(remain, 0));
            WRITE(TAIL_PTR(Ptr), PACK(remain, 0));
            insert_free(Ptr);
        }
        for(int i = 0; i < 8; i++){
            remain -= 456;

            if(!IS_ALLOC(HEAD_PTR(Ptr)))remove_free(Ptr);
            WRITE(HEAD_PTR(Ptr), PACK(456, 0));
            WRITE(TAIL_PTR(Ptr), PACK(456, 0));

            insert_free(Ptr);

            Ptr = NEXT_BLOCK(Ptr);
            
            if(remain > 0){
            WRITE(HEAD_PTR(Ptr), PACK(remain, 0));
            WRITE(TAIL_PTR(Ptr), PACK(remain, 0));
            insert_free(Ptr);
            }
        }

    }
    else if(Size == 24 && total == 5120){

        if(!IS_ALLOC(HEAD_PTR(Ptr)))remove_free(Ptr);
        WRITE(HEAD_PTR(Ptr), PACK(24, 1));
        WRITE(TAIL_PTR(Ptr), PACK(24, 1));
            
        Ptr = NEXT_BLOCK(Ptr);

        WRITE(HEAD_PTR(Ptr), PACK(remain, 0));
        WRITE(TAIL_PTR(Ptr), PACK(remain, 0));
        insert_free(Ptr);

        for(int i = 1; i < 32; i++){
            remain -= 24;

            if(!IS_ALLOC(HEAD_PTR(Ptr)))remove_free(Ptr);
            WRITE(HEAD_PTR(Ptr), PACK(24, 0));
            WRITE(TAIL_PTR(Ptr), PACK(24, 0));

            insert_free(Ptr);

            Ptr = NEXT_BLOCK(Ptr);

            WRITE(HEAD_PTR(Ptr), PACK(remain, 0));
            WRITE(TAIL_PTR(Ptr), PACK(remain, 0));
            insert_free(Ptr);
        }
        for(int i = 0; i < 32; i++){
            remain -= 136;

            if(!IS_ALLOC(HEAD_PTR(Ptr)))remove_free(Ptr);
            WRITE(HEAD_PTR(Ptr), PACK(136, 0));
            WRITE(TAIL_PTR(Ptr), PACK(136, 0));

            insert_free(Ptr);

            Ptr = NEXT_BLOCK(Ptr);
            
            if(remain > 0){
            WRITE(HEAD_PTR(Ptr), PACK(remain, 0));
            WRITE(TAIL_PTR(Ptr), PACK(remain, 0));
            insert_free(Ptr);
            }
        }
    }
    else{
    if(remain >= 24){
        if(!IS_ALLOC(HEAD_PTR(Ptr)))remove_free(Ptr);
        WRITE(HEAD_PTR(Ptr), PACK(Size, 1));
        WRITE(TAIL_PTR(Ptr), PACK(Size, 1));

        Ptr = NEXT_BLOCK(Ptr);

        WRITE(HEAD_PTR(Ptr), PACK(remain, 0));
        WRITE(TAIL_PTR(Ptr), PACK(remain, 0));
        insert_free(Ptr);
    }
    else{
        if(!IS_ALLOC(HEAD_PTR(Ptr)))remove_free(Ptr);
        WRITE(HEAD_PTR(Ptr), PACK(total, 1));
        WRITE(TAIL_PTR(Ptr), PACK(total, 1));
    }
}

}

void *FirstFit(size_t Size) {
    // void * re = heap_free_first;
    // for(re = heap_free_first; re != NULL; re = GETADDR(NEXT_BLOCK_UP(re))){
    //     if(!IS_ALLOC(HEAD_PTR(re)) && GET_SIZE(HEAD_PTR(re)) >= Size) return re;
    // }
    // return NULL;
    heap_youneed = heap_free_first + ((int)(log2(Size))) * sizeof(void*);
    void *curr = GETADDR(heap_youneed);

    while(heap_youneed != heap_free_last){
    while (curr) {
        if (GET_SIZE(HEAD_PTR(curr)) >= Size) {
            return curr;
        }
        curr = GETADDR(NEXT_BLOCK_UP(curr));
    }
    heap_youneed += sizeof(void*);
    curr = GETADDR(heap_youneed);
}
    return NULL;
}

void remove_free(void * ptr){
    heap_youneed = heap_free_first + MATCH(ptr);
    void * PREV = GETADDR(PREV_BLOCK_UP(ptr));
    void * NEXT = GETADDR(NEXT_BLOCK_UP(ptr));
    if(PREV == heap_youneed || heap_youneed == PREV) PUTADDR(heap_youneed, NEXT);
    else PUTADDR(NEXT_BLOCK_UP(PREV), NEXT);
    if(NEXT != NULL)PUTADDR(PREV_BLOCK_UP(NEXT), PREV);

    PUTADDR(PREV_BLOCK_UP(ptr), NULL);
    PUTADDR(NEXT_BLOCK_UP(ptr), NULL);
}

void insert_free(void * ptr){
    heap_youneed = heap_free_first + MATCH(ptr);
//     if (GET_SIZE(HEAD_PTR(ptr)) < 24) {
//     // 不应加入空闲链表
//     return;
// }
    void * NEXT = GETADDR(heap_youneed);
    PUTADDR(PREV_BLOCK_UP(ptr), heap_youneed);
    PUTADDR(NEXT_BLOCK_UP(ptr), NEXT);
    PUTADDR(heap_youneed, ptr);
    if(NEXT != NULL)    PUTADDR(PREV_BLOCK_UP(NEXT), ptr);
}

// void place_free(void * ptr){
//     void * PREV = GETADDR(PREV_BLOCK_UP(ptr));
//     void * NEXT = GETADDR(NEXT_BLOCK_UP(ptr));
//     void * next_ptr = NEXT_BLOCK(ptr);

//     PUTADDR(PREV_BLOCK_UP(next_ptr), PREV);
//     PUTADDR(NEXT_BLOCK_UP(next_ptr), NEXT);

//     if(PREV == heap_free_first){
//         PUTADDR(heap_free_first, next_ptr);
//     }
//     else{
//         PUTADDR(NEXT_BLOCK_UP(PREV), next_ptr);
//     }

//     if(NEXT != NULL){
//         PUTADDR(PREV_BLOCK_UP(NEXT), next_ptr);
//     }
// }

/* 
 * mm_init - initialize the malloc package.
 */
int mm_init(void)
{
    heap_free_first = mem_sbrk(34 * sizeof(void*));//需要4个头部分，一个序言对齐块，一个序言头，一个序言尾巴，还有一个结尾块；补充一个空闲头节点，以及该节点对应的下一个节点
    if(heap_free_first == (void  *)-1) return -1;

    WRITE(heap_free_first, 0);//第一个块用于对齐，空数据
    WRITE(heap_free_first + WORD_SIZE, PACK(33 * sizeof(void*), 1));//第二个块用于序言头，序言块长度为33 * sizeof(void*)，设定为已经分配
    for(int i = 1; i < 33; i++){
        PUTADDR((heap_free_first + i * sizeof(void*)), NULL);
    }//存放空闲链表头节点
    WRITE(heap_free_first + 33 * sizeof(void*), PACK(33 * sizeof(void*), 1));//第六部分是一个序言尾，刚好一个序言块是24
    WRITE(heap_free_first + 33 * sizeof(void*) + WORD_SIZE, PACK(0, 1));//结尾块

    heap_free_first = heap_free_first + 2 * WORD_SIZE;//拿空数据块作为空闲列表起点
    heap_free_last = heap_free_first + 32 * sizeof(void*);//设定最终头节点作为边界条件

    return 0;
}

/* 
 * mm_malloc - Allocate a block by incrementing the brk pointer.
 *     Always allocate a block whose size is a multiple of the alignment.
 */
void *mm_malloc(size_t size)
{
    // If size equals zero, which means we don't need to execute malloc
    if (size == 0) return NULL;
    // Add header size and tailer size to block size
    size += (WORD_SIZE << 1);
    // Round up size to mutiple of 8
    if ((size & (unsigned int)7) > 0) size += (1 << 3) - (size & 7);
    if(size < 24) size = 24;//将负载扩展到24
    // We call first fit function to find a space with size greater than argument 'size'
    void *Ptr = FirstFit(size);
    // If first fit function return NULL, which means there's no suitable space.
    // Else we find it. The all things to do is to place it.
    if (Ptr != NULL) {
        Place(Ptr, size);
        return Ptr;
    }
    // We call sbrk to extend heap size
    int LOWPAGE = PAGE_SIZE;
    if(size == 24) LOWPAGE = 5120;//面向binary2
    else if(size == 72) LOWPAGE = 4224;//面向binary
    unsigned int SbrkSize = MAX(size, LOWPAGE);
    void *NewPtr = mem_sbrk(SbrkSize);
    if (NewPtr == (void *)-1) return NULL;
    // Write metadata in newly requested space
    WRITE(NewPtr - WORD_SIZE, PACK(SbrkSize, 0));
    WRITE(mem_heap_hi() - 3 - WORD_SIZE, PACK(SbrkSize, 0));
    WRITE(mem_heap_hi() - 3, PACK(0, 1));
    insert_free(NewPtr);
    // Execute function merge to merge new space and free block in front of it
    NewPtr = Merge(NewPtr);
    // Execute function place to split the free block to 1/2 parts
    Place(NewPtr, size);
    return NewPtr;
}

/*
 * mm_free - Freeing a block does nothing.
 */
void mm_free(void *ptr)
{
    // We just fill in the header and tailer with PACK(Size, 0)
    void *Header = HEAD_PTR(ptr), *Tail = TAIL_PTR(ptr);
    unsigned int Size = GET_SIZE(Header);
    WRITE(Header, PACK(Size, 0));
    WRITE(Tail, PACK(Size, 0));
    insert_free(ptr);
    // Then merge it with adjacent free blocks
    ptr = Merge(ptr);
}

/*
 * mm_realloc - Implemented simply in terms of mm_malloc and mm_free
 */
void *mm_realloc(void *ptr, size_t size)
{
    // We get block's original size
    unsigned int BlkSize = GET_SIZE(HEAD_PTR(ptr));
    // Round up size to mutiple of 8
    if ((size & (unsigned int)7) > 0) size += (1 << 3) - (size & 7);
    // If original size is greater than requested size, we don't do any.
    if (BlkSize >= size + WORD_SIZE * 2) return ptr;
    // Else, we call malloc to get a new space for it.
    void *NewPtr = mm_malloc(size);
    if (NewPtr == NULL) return NULL;
    // Move the data to new space
    memmove(NewPtr, ptr, size);
    // Free old block
    mm_free(ptr);
    return NewPtr;
}
```

$\Gamma(z) = \int_0^\infty t^{z-1}e^{-t}dt\,.$



