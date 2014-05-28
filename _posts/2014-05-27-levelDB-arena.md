---
layout: post
title: levelDB Arena分析
categories:
- Technology
tags:
- levelDB
- arena
- 内存池
---
# 概括
	Arena是一个内存管理类,
	类中使用一个std::vector block_来保存所有 分配的内存块.
		提供两种内存分配方式: 提供 "对齐内存分配", 与 "不对齐内存分配".
	对于小内存(小于1K大小), arena提供内置的分配方式,而不是每次都请求 系统的allocator.

# source

/util/arena.h

/util/arena.cc
	
接口

<pre class="prettyprint lang-html">
class Arena {
 public:
  Arena();
  ~Arena();

  // 返回 "bytes" 大小的内存块的 指针.
  char* Allocate(size_t bytes);

  // 分配对齐的内存块,由alloc来保证
  char* AllocateAligned(size_t bytes);

  // Returns an estimate of the total memory usage of data allocated
  // by the arena (including space allocated but not yet used for user
  // allocations).
  size_t MemoryUsage() const {
    return blocks_memory_ + blocks_.capacity() * sizeof(char*);
  }

 private:
  char* AllocateFallback(size_t bytes);
  char* AllocateNewBlock(size_t block_bytes);

  // 最后一次分配的 内存块(默认大小,提供给小于1K 请求)的 
  //尾指针(指向未利用内存), 与 剩余下内存的大小
  char* alloc_ptr_;
  size_t alloc_bytes_remaining_;

  //这里用vector来保存所有new [] allocated 内存块
  std::vector<char*> blocks_;

  // Bytes of memory in blocks allocated so far
  size_t blocks_memory_;

  // No copying allowed
  Arena(const Arena&);
  void operator=(const Arena&);
};
</pre>

分配代码

```cpp
//在外部(arena.cc)中 定义 了 默认分配块的 大小
static const int kBlockSize = 4096;

inline char* Arena::Allocate(size_t bytes) {
  // The semantics of what to return are a bit messy if we allow
  // 0-byte allocations, so we disallow them here (we don't need
  // them for our internal use).
  assert(bytes > 0);  //不允许0的分配请求
  if (bytes <= alloc_bytes_remaining_) { //如果上次剩余的块中 还有 足够的空间
    char* result = alloc_ptr_;
    alloc_ptr_ += bytes;
    alloc_bytes_remaining_ -= bytes;
    return result;
  }
  return AllocateFallback(bytes);  //否则 调用另一个分配函数.
}

//从Allocate()过来的话,那么 最后一次分配的默认块 所剩余的空间 已经不足"bytes"
char* Arena::AllocateFallback(size_t bytes) {
  if (bytes > kBlockSize / 4) {
    // Object is more than a quarter of our block size.  Allocate it separately  超过1K 则直接用new []分配
    // to avoid wasting too much space in leftover bytes.
    char* result = AllocateNewBlock(bytes); //直接调用new
    return result;
  }

	//如果 要的size 不足1K,那么就分配4K 
  // We waste the remaining space in the current block.
  alloc_ptr_ = AllocateNewBlock(kBlockSize);
  alloc_bytes_remaining_ = kBlockSize;

  //然后将 (4K - 要求size)作 剩余空间, 放在 当前block
  char* result = alloc_ptr_;
  alloc_ptr_ += bytes;
  alloc_bytes_remaining_ -= bytes;
  return result;
}

char* Arena::AllocateAligned(size_t bytes) {
  const int align = (sizeof(void*) > 8) ? sizeof(void*) : 8; //最小对齐 8bytes 比如32位系统,sizeof(void*) == 4;
  assert((align & (align-1)) == 0);   // Pointer size should be a power of 2
  size_t current_mod = reinterpret_cast<uintptr_t>(alloc_ptr_) & (align-1); //用(align-1)来获取 alloc_ptr_(最后一次默认分配的块) 地址的低x位置,
  size_t slop = (current_mod == 0 ? 0 : align - current_mod); //看看alloc_ptr_的后几位是否全0(对齐的意思); 如果不是零,则将 互补的大小 加到bytes中
  size_t needed = bytes + slop;
  char* result;
  if (needed <= alloc_bytes_remaining_) {  //经过对齐处理后 如果能在 最后分配的块 中 找到足够的 空间,则分配
    result = alloc_ptr_ + slop;
    alloc_ptr_ += needed;
    alloc_bytes_remaining_ -= needed;
  } else {
    // AllocateFallback always returned aligned memory
    result = AllocateFallback(bytes);  //这里系统提供的new[] 会满足对齐要求
  }
  assert((reinterpret_cast<uintptr_t>(result) & (align-1)) == 0); //再一次判断
  return result;
}
```
	
	