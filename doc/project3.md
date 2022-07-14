# PROJECT #3 - QUERY EXECUTION

## 设计理解

- 封装的很好
    - 一张表是一个对象Table，每一列是一个对象Column，甚至插入的信息也是一个对象，每一行的数据也是一个对象Tuple
    - 类型是抽象类，表达式是抽象类，执行计划也是一个抽象类
    - Plan通过ExecutorFactory构建出Executor，而产生unique_ptr通过移动语义产生成一棵树
    - Page是一个类，储存表的TablePage也继承于Page，用迭代器模式来遍历一个table
    - 为了支持事务之类的，把Transaction传到了很细节的很多类里
- 目前不支持sql，为了写一句sql要写一段很长的代码，是否有可能做词法分析和语法分析，输入一个sql语句，得到一个执行计划(plan)，这样测试和使用起来都方便地多
- 支持联合索引

## 主要思路

- 要做这块儿需要对BusTub的设计有更多的理解，需要更多的从源码入手
- 我选择从最简单的入手：
    - <del>Limit</del> 结果发现limit依赖seqScan的实现
    - SeqScan
        - ExecutorContext上下文 → Catalog目录 → TableInfo表信息 → Schema / TableHeap / TableName / TableId
        - 有了这些后就可以拿来构造iterator（但这个iterator还是不好用...）
        - 坑点：需要进行schema转换，例如select colB as B from test_1 where colA < 500，原表的schema和该select语句的schema是不一致的
    - Limit
        - 调用SeqScan的接口即可
    - Insert
        - 主要是搞明白TableHeap和Index的插入接口如何调用
        - Insert是root plan，因此一次next执行完毕，并且返回false，而不是一条一条insert(update和delete也是一样)
    - Update
        - update只支持Set和Add
        - 更新表可以直接用updateTuple，但是更新索引要先删除再插入
    - Delete
        - 直接删除即可
    - NestLoopJoin
        - 维护左表的循环位置，对于左表的每一个位置，要把右表完整遍历一遍(重新init之后再next)
        - 注意有一个坑点，就是有表为空的时候要保证不会调用超过常数次next，并且要保证内存安全，否则会IO TEST FAILED（被这个困扰了好久...）
    - HashJoin
        - 哈希表是(Join key value, List[Tuple])的类型
        - 需要让unordered_map支持自定义的数据类型（Value），为此需要重载==以及特化struct hash<>，并且这个特化需要放在声明unordered_map之前
        - 左表在init的时候就全部插入到unordered_map中，然后记录右表的位置以及对应Tuple List的下标，以便下一次输出
        - (并不是很理解作业提示里说的可能要构造多attribute的key)
        - 同上，有一个表为空的时候（哈希表或者右表）直接返回false，不过多的调用next
    - Aggregation
        - is_group_by_term_其实是用来区分group by的字段当做返回值的时候是不用聚合的，然后会把返回值里的聚合以及having里的聚合一起都放在聚合值里求解，最后再根据schema来汇总输出的结果
        - 维护HashTable的iterator即可
    - Distinct
        - 本来想用list[value]作为set的key，但是后来发现distinct的output和child的output的schema是一致的，不需要转换，所以要从tuple里拿value里麻烦一点
        - 所以直接用tuple做key，但是tuple没有提供==，所以封一层，实现==和特化struct hash<>
    - 最后在leaderboard上排名很靠后，猜测可能是因为没有对NestLoopJoin之类的做一些优化？

## 参考资料

- [C++ 中移动语义的用法和实现原理](https://xiaotaoguo.com/p/cpp-move-semantics/)