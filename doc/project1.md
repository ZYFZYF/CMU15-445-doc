# PROJECT #1 - BUFFER POOL

## TASK #1 - LRU REPLACEMENT POLICY

### 主要思路

- 这里的LRU其实相当于一个削弱版的LRU，因为一般的LRU中会有把链表中间的元素移到链表尾部的操作（也就是use操作，但该场景下只有frame第一次进来的时候会被use）
- 采用通用的map + 双向链表的方式

### 一些优化

- 为了效率采用unordered_map
- 为了避免频繁的new/delete可以先把node全开出来，并且用一个队列去回收不用的节点，下次需要插入时直接从队列里去取
- 为了双向链表维护简单用了两个空节点dummy_head和dummy_tail，这样保证正在使用的节点一定是prev和next都不为空，插入删除的时候都会比较方便
- UPDATE: 发现了std::list这个容器，重写了这块儿的代码，是真的香（最好的一点是list::iterator在容器增删后不会失效，所以可以用来当索引），可能会慢一些，但是这样代码可读性更强，更容易维护

### 一些坑点

- 提交了60次，发现出现以下问题

`The autograder failed to execute correctly. Please ensure that your submission is valid. Contact your course staff for help in debugging this issue. Make sure to include a link to this page so that they can help you most effectively._
`

- 有两个原因：
    - 第一个是修改了别的地方的文件，但是没有及时上传（例如我在config.h里加了一个常量）
    - 第二个是使用了C++17的结构化绑定
        - 这样是错的
        - ` for (auto &[page_id, _] : page_table_) { FlushPgImp(page_id); }`
        - 这样是对的
        - `for (auto page : page_table_) { FlushPgImp(page.first); }`

## TASK #2 - BUFFER POOL MANAGER INSTANCE

### 设计理解

- 一个buffer pool是针对一个文件做的，就是为了缓存该文件的一些页在内存里，使得操作的时候可以直接操作内存，长时间不访问的话再flush到磁盘（或者手动flush）
- 可以有多个线程同时持有一个Page的数据，因此Page内部有读写锁
- buffer pool 为了thread-safe要上锁
- 如果一个文件就这么一把锁，多线程的优势就不明显了（访问buffer还是单线程），所以搞了多个buffer pool instance，并且用mod划分page到不同的pool（会使得相邻页属于不同的buffer pool
  instance）

### 主要思路

- 其实要做的注释里都写好了
- 区分New和Fetch，前者是新建一个页，并且直接放到缓存中，后者是获取已有的某一页，不在缓存中的话要从磁盘中读取并加载到缓存中
- 区分Delete和Flush，前者是从缓存中删除，后者只是刷到磁盘（当前者执行时，并且是脏页则要Flush）
- Flush是无条件写入磁盘，因为caller可能不想set isDirty
- 因为要保证**每个方法**的线程安全，而为了实现方便**方法之间又有调用关系**，所以我们使用recursive_lock

### 一些坑点

- NewPageImp的时候不要置isDirty为True，而是要直接写回一次磁盘，置isDirty为False
- DeletePageImp的时候要记得从lruReplacer里去掉该page对应的frame

### 收获感想

- 学习了std::scope_lock，再也不用自己unlock了，lock_guard的主要区别是可以同时对多个上锁，避免ABBA式的死锁

### 参考资料

- [scope_lock与lock_guard区别](https://www.cnblogs.com/defen/p/4409904.html)

## TASK #3 - PARALLEL BUFFER POOL MANAGER

### 设计理解

- Parallel buffer pool manager其实就是做个转发，把请求根据page_id的映射关系发到具体的instance
- newPage分配的时候是轮询，这可能导致分配的页面在物理上不连续，轮询的好处是保证相邻的new在不同的buffer pool，从而可以更好的并发

### 主要思路

- 也没什么好说的