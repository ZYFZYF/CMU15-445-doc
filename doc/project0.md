# PROJECT #0 - C++ PRIMER

## 主要思路

- 实现预定义好的类Matrix、RowMatrix、RowMatrixOperations中的方法

## 一些坑点

- 要遵循[Google编程规范](https://google.github.io/styleguide/cppguide.html)
- 最好不要有中文路径，不然某些工具可能会出现奇怪的问题（比方检查格式的format系列工具）

## 收获感想

- 学习了下std::unique_ptr、std::function与std::bind，感慨C++的博大精深
- std::unique_ptr无法赋值、无法值传递，可以std::move()转移所有权，或者通过作为函数返回值来延长生命周期，或者用.get()获得普通指针进行参数传值
- std::bind类似于对std::function的一个绑定、转发、适配器，也有点像scala里的currying

## 参考资料
- [【C++11】 之 std::unique_ptr 详解](https://blog.csdn.net/lemonxiaoxiao/article/details/108603916)
- [std::function和std::bind](http://xueguoliang.cn/index.php/archives/16/)
- [C++ std::bind](https://www.jianshu.com/p/82407fb43475)