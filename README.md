# php7_wrapper
### 写在前面：
> swoole的出现让php进入了除了web的其他领域，php7的性能提升又再次拓宽了php的应用范围，例如游戏领域，但php7为了提升性能修改了大量的zendapi，看到新版本的zendapi相信大家得第一反应都是，改动太大了！为啥不兼容之前的，还是那句话都是为了性能提升，所以基本所有的php扩展都需要做一个分支出来支持新版本php，对扩展开发者来说需要同时维护两套代码，使用扩展的人就需要针对每个php版本编译不同的so文件比较麻烦，本文将从另一个角度来尝试解决这个问---通过引入一组兼容层函数来尽量少修改扩展代码并且一套代码支持所有php，并且此方案已经成功的应用在了swoole和php-cp(pdo和redis代理)项目中，下面将分为几个部分讲具体的实现以及php7中为什么改成这样

#### 内存和字符串相关

### 1、MAKE_STD_ZVAL
> php7的改动很大的就是内存的部分，首先新设计的zval、zend_string、hashtable(ng中也叫zend_array)等数据结构体积很小并且很紧凑，能充分利用cpu的cache，其次大量用栈空间代替在堆中申请内存，反应到扩展层面就是去掉了MAKE_STD_ZVAL和ALLOC_INIT_ZVAL等内存申请的宏定义，因为phpng的zendapi都不需要在堆上申请内存然后把地址传给zendapi（当然如果业务确实需要你也可以手动emalloc内存），swoole和php-cp中是怎么做的呢？
首先，将扩展中所有的MAKE_STD_ZVAL批量替换为SW_MAKE_STD_ZVAL
其次，我们再看SW_MAKE_STD_ZVAL的定义：

```c
#if PHP_MAJOR_VERSION < 7
  #define SW_MAKE_STD_ZVAL(p)            MAKE_STD_ZVAL(p)
#else
  #define SW_MAKE_STD_ZVAL(p)            zval _stack_zval_##p; p = &(_stack_zval_##p)  //宏展开后就是在栈上定义的zval
#endif
```
> 可以看到在phpng中SW_MAKE_STD_ZVAL就是在栈上定义了一个zval 然后把地址赋值给p，在之前的php版本中就是直接调用MAKE_STD_ZVAL
注意：此方法能适应大部分用到MAKE_STD_ZVAL的地方，但是有些时候（例如：函数内部将MAKE_STD_ZVAL得到的指针作为函数返回值返回了）是不行的，需要手动emalloc，就如开头说的"通过引入一组兼容层函数来尽量少修改扩展代码"。

### 2、zval_ptr_dtor
> zval_ptr_dtor在之前的php版本中传递参数是指针的指针，phpng中是zval的指针.
首先，将扩展中所有的zval_ptr_dtor替换成sw_zval_ptr_dtor（以下都如此不再赘述）。
兼容宏代码如下：
```c
#if PHP_MAJOR_VERSION < 7
  #define sw_zval_ptr_dtor               zval_ptr_dtor
#else
  #define sw_zval_ptr_dtor(p)            zval_ptr_dtor(*p) //用*操作把指针的指针变成指针（好绕口）
#endif
```
有人可能会问，在上面说了是通过栈来模拟malloc内存，那这里的释放内存是不是会段错误？
```c
if (Z_REFCOUNTED_P(zval_ptr)) {
		if (!Z_DELREF_P(zval_ptr)) {
			_zval_dtor_func_for_ptr(Z_COUNTED_P(zval_ptr) ZEND_FILE_LINE_RELAY_CC);
		} else {
			GC_ZVAL_CHECK_POSSIBLE_ROOT(zval_ptr);
		}
	}
```
如上，因为phpng中释放内存之前会判断如果不是引用变量类型就跳过不做任何操作。

### 3.zval_add_ref
>添加引用计数的方法:
```c
#if PHP_MAJOR_VERSION < 7
 #define sw_zval_add_ref             zval_add_ref
#else
 #define sw_zval_add_ref(p)          Z_TRY_ADDREF_P(*p)//phpng会先判断类型然后再加引用
#endif
```
###4.
