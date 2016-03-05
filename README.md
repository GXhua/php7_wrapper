# php7_wrapper
写在前面：
swoole的出现让php进入了除了web的其他领域，php7的性能提升又再次拓宽了php的应用范围，例如游戏领域，但php7为了提升性能修改了大量的zend api，看到新版本的zend api相信大家得第一反应都是，改动太大了！为啥不兼容之前的，还是那句话都是为了性能提升（例如：php7去掉了MAKE_STD_ZVAL这个申请内存的宏定义，意在引导扩展开发者尽量多的用栈内存替代堆中申请内存，事实证明大部分的MAKE_STD_ZVAL都是不必要的），所以基本所有的php扩展都需要做一个分支出来支持新版本php，对扩展开发者来说需要同时维护两套代码，使用扩展的人就需要针对每个php版本编译不同的so文件比较麻烦，本文将从另一个角度来尝试解决这个问---通过引入一组兼容层函数来尽量少修改扩展代码并且一套代码支持所有php，并且此方案已经成功的应用在了swoole和php-cp(pdo和redis代理)项目中，下面将分为几个部分讲具体的实现以及php7中为什么改成这样



上面提到了php7中去掉了MAKE_STD_ZVAL和ALLOC_INIT_ZVAL等内存申请的宏定义来强行让扩展开发者将不必要的堆内存申请删除掉（当然如果业务确实需要你也可以emalloc内存），swoole和php-cp中是怎么做的呢？
首先，将扩展中所有的MAKE_STD_ZVAL批量替换为SW_MAKE_STD_ZVAL
其次，我们再看SW_MAKE_STD_ZVAL的定义：
<pre>
#if PHP_MAJOR_VERSION < 7
  #define SW_MAKE_STD_ZVAL(p)                   MAKE_STD_ZVAL(p)
#else
  #define SW_MAKE_STD_ZVAL(p)                   zval _stack_zval_##p; p = &(_stack_zval_##p)
#endif
</pre>
一：hash表相关

首先需要在扩展代码里面批量替换
例如：将zend_hash_del 修改为 sw_zend_hash_del
