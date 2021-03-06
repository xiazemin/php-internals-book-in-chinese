# 基础结构
 一个zval（“Zend value”的缩写）代表一个任意类型的PHP变量。所以，它很可能是PHP中最重要的数据结构，同时你将会频繁地使用它。本章节讲述zvals的基础概念及其使用方法。

## 类型和值
 每一个zval都会存储某个值和其对应的类型。这点非常重要，因为PHP是一门动态类型语言，所以变量的类型只有当运行时才会确定，并不是在编译时就能够确定。此外，zval的类型在其生命周期是可以改变的，所以如果这个zval在最初存储了一个整形，那么在之后的某个时间点他也可能会存储了一个字符串。
 类型是存储在一个整形的标签中（一个 unsigned char 类型的变量）。它有8中类型的值，分别对应着PHP中的8中变量类型。这些值可以用诸如IS_TYPE形式的常量来使用。比如：`IS_NULL`对应null类型，`IS_STRING`对应字符串类型。
 
 真实的值是存储在一个联合体中，如下所示：
 ```c
 typedef union _zvalue_value {
    long lval;
    double dval;
    struct {
        char *val;
        int len;
    } str;
    HashTable *ht;
    zend_object_value obj;
} zvalue_value;
```
写给那些不了解联合体概念的同学：一个联合体定义了多个不同类型的成员，但是在某一刻只能使用其中一个成员。比如：如果给`value.lval`成员赋值了，那么你就需要用`value.lval`的方式来访问这个值，而不能用其他成员（否则会侵犯“strict aliasing”规则，并导致未定义的行为 - undefined behaviour）。这是因为联合体将其所有成员存储再一个相同的内存地址中，并根据访问的是哪一个成员来访问值。联合体的大小由其中的最大的成员决定。

在使用zvals时，类型标签用来寻找正在使用的是联合体中的哪一个成员。在介绍一些相关的APIs之前，我们先来大致过一下PHP支持的不同类型及其存储方式。
最简单的类型是`IS_NULL`：事实上不需要存储任何值，因为它本身就保留一个`null`值。

为了存储数字，PHP提供了`IS_LONG`和`IS_DOUBLE`类型，它们分别用于表达`long lval`和`double dval`两种成员。前者用于存储整形，然而后者用于存储浮点型数字。

针对于`long`类型有一些应该注意的事情：首先，这是一个有符号的整形类型，比如：他可以存储正数和负数，但是通常来说不适用于做一些位操作。第二，`long`类型在不同的平台上有不同的大小：在32位系统上是32位/4个字节，但是在64位系统上，要么是4个字节要么是8个字节。其中在64位的Unix系统上时，是8个字节，然而在64位的Windows系统上时仅仅使用了4个字节。

所以，你不应该依赖于`long`类型的大小。可以用`LONG_MIN`和`LONG_MAX`宏来判断当前`long`类型可以存储的最小值和最大值，同时可以使用`SIZEOF_LONG`（和`sizeof(long)`不同的是，它可以用在`#if`指令中）来获取该类型当前的大小。由于只需要存储两种值，那么理论上应该使用一个占用空间更小的类型（像`zend_bool`这样的无符号字符型），但是由于`zvalue_value`联合体的大小取决于其成员中最大的一个，所以这么做并不会实际减少内容的使用。故此，复用了这个`lval`成员。

字符串（`IS_STRING`）存储在结构体中`struct { char * val; int len; } str`，也就是说由一个`char *`类型的字符串和`int`类型的长度值组成。为了在字符串中使用NUL字节（`'\0'`），PHP字符串需要存储一个显示表示长度的字段（保证二进制安全）。先不考虑这些，PHP的字符串还是要以NUL结尾的，这是为了让一些不接收长度参数并且期望以NUL结尾的字符串的库函数更易使用。当然，在这个案例中字符串就不再需要是二进制安全的了，且它会在第一个出现NUL字节的地方被截断。比如很多文件系统相关的函数和大多数libc库的字符串函数都是这样的。

字符串的长度是以字节为单位（并不是Unicode code points），同时不包含其中的结尾字符 - NUL：字符串`"foo"`长度是3，尽管他实际上需要4个字节的存储空间。如果你需要利用`sizeof`来获取一个常量字符串的长度，那么要保证减少一个字节：`strlen("foo") == sizeof("foo") - 1`。

除此之外，很关键的一点是字符串的长度信息是存储在`int`类型中，而不是`long`类型中，或者其他什么类型中。这是一个不行的历史遗留问题，因为这会导致字符串的最大长度为2147483647字节。如果在字符串变量中存储比这多的数据会导致内存溢出（这样会导致长度为负数）。

剩下的三种类型在这里仅作简单的介绍，会在相应的章节中进行详细的阐述：

数组使用的是`IS_ARRAY`标签，存储在`HashTable *ht`成员中。将会在<b>Hashtables</b>章节中讨论`Hashtable`结构体是如何工作的。

对象（`IS_OBJECT`）使用`zend_object_value obg`成员， 它是由一个“object handle”和一组“object handlers”构成，其中“object hanle”是一个整型ID，用于寻找当前对象实际的值；“object handlers”定义了对象如何运作。将会在<b>Classes and objects</b>章节中对PHP的类和对象系统进行阐述。

资源（`IS_RESOURCE`）和对象类似，因为他们同样都存储了一个唯一ID，这个ID都是用来去寻找真实的值。ID存储在`long lval`成员中。将会在资源章节中（目前还没有完成）进行阐述。

为了有个形象的概念，下表给出了所有可用类型的标签和相应的值的存储位置：

<table>
    <tr>
        <th>类型标签</th>
        <th>存储位置</th>
    </tr> 
    <tr>
        <td>IS_NULL</td>
        <td>none</td>
    </tr>
    <tr>
        <td>IS_BOOL</td>
        <td>long lval</td>
    </tr>
    <tr>
        <td>IS_LONG</td>
        <td>long lval</td>
    </tr>
    <tr>
        <td>IS_DOUBLE</td>
        <td>double dval</td>
    </tr>
    <tr>
        <td>IS_STRING</td>
        <td>struct { char *val; int len; } str</td>
    </tr>
    <tr>
        <td>IS_ARRAY</td>
        <td>Hashtable *ht</td>
    </tr>
    <tr>
        <td>IS_OBJECT</td>
        <td>zend_object_value obj</td>
    </tr>
    <tr>
        <td>IS_RESOURCE</td>
        <td>long lval</td>
    </tr>
</table>

## 访问宏

现在我们看看`zval`的真实面目吧：

```c
typedef struct _zval_struct {
    zvalue_value value;
    zend_uint refcount__gc;
    zend_uchar type;
    zend_uchar is_ref__gc;
} zval;
```

正如前面提到的，zval结构体中包含了存储值和类型两个成员。其中值是存储在前文介绍过的`zvalue_value`联合体中，类型是存储在一个`zend_uchar`的类型成员中。此外，该结构体还包括两个以`__gc`结尾的属性，这两个属性是用于PHP的垃圾回收机制。本章节不过多介绍，将会在后面章节做具体讲解。

在了解了`zval`结构体后，你可以这样使用它：

```c
zval *zv_ptr = /* ... get zval from somewhere */;

if (zv_ptr->type == IS_LONG) {
    php_printf("Zval is a long with value %ld\n", zv_ptr->value.lval);
} else /* ... handle other types */
```

尽管上面的代码会达到预期效果，但是这一种非常规的写法。因为它直接使用了结构体的成员属性，并没有使用一些宏来访问。

```c
zval *zv_ptr = /* ... */;

if (Z_TYPE_P(zv_ptr) == IS_LONG) {
    php_printf("Zval is a long with value %ld\n", Z_LVAL_P(zv_ptr));
} else /* ... */
```

上面代码使用了`Z_TYPE_P()`宏来获取类型，`Z_LVAL_P`会返回一个long类型的值。所有这些访问的宏都会以不定个数的`_P`结尾，也许是`_P`，也许是`_PP`，也有可能没有。这取决于你传递的参数是`zval`，`zval*`还是`zval**`类型的：

```c
zval zv;
zval *zv_ptr;
zval **zv_ptr_ptr;
zval ***zv_ptr_ptr_ptr;

Z_TYPE(zv);                 // = zv.type
Z_TYPE_P(zv_ptr);           // = zv_ptr->type
Z_TYPE_PP(zv_ptr_ptr);      // = (*zv_ptr_ptr)->type
Z_TYPE_PP(*zv_ptr_ptr_ptr); // = (**zv_ptr_ptr_ptr)->type
```

通常来说`P`的个数和`*`的个数应该是一致的。但是最多只能到`zval**`，也就是说没有能够处理`zval***`类型的特殊宏，因为这种情况在实践中非常少见（你仅需要使用`*`运算符来取消引用）。

针对其它类型，同样也会有和`Z_LVAL`类似的访问值的宏。我们通过创建如下一个打印zval值的函数来说明如何使用这些宏：

```c
PHP_FUNCTION(dump)
{
    zval *zv_ptr;

    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "z", &zv_ptr) == FAILURE) {
        return;
    }

    switch (Z_TYPE_P(zv_ptr)) {
        case IS_NULL:
            php_printf("NULL: null\n");
            break;
        case IS_BOOL:
            if (Z_BVAL_P(zv_ptr)) {
                php_printf("BOOL: true\n");
            } else {
                php_printf("BOOL: false\n");
            }
            break;
        case IS_LONG:
            php_printf("LONG: %ld\n", Z_LVAL_P(zv_ptr));
            break;
        case IS_DOUBLE:
            php_printf("DOUBLE: %g\n", Z_DVAL_P(zv_ptr));
            break;
        case IS_STRING:
            php_printf("STRING: value=\"");
            PHPWRITE(Z_STRVAL_P(zv_ptr), Z_STRLEN_P(zv_ptr));
            php_printf("\", length=%d\n", Z_STRLEN_P(zv_ptr));
            break;
        case IS_RESOURCE:
            php_printf("RESOURCE: id=%ld\n", Z_RESVAL_P(zv_ptr));
            break;
        case IS_ARRAY:
            php_printf("ARRAY: hashtable=%p\n", Z_ARRVAL_P(zv_ptr));
            break;
        case IS_OBJECT:
            php_printf("OBJECT: ???\n");
            break;
    }
}

const zend_function_entry funcs[] = {
    PHP_FE(dump, NULL)
    PHP_FE_END
};
```

执行结果如下：
```c
dump(null);                 // NULL: null
dump(true);                 // BOOL: true
dump(false);                // BOOL: false
dump(42);                   // LONG: 42
dump(4.2);                  // DOUBLE: 4.2
dump("foo");                // STRING: value="foo", length=3
dump(fopen(__FILE__, "r")); // RESOURCE: id=???
dump(array(1, 2, 3));       // ARRAY: hashtable=0x???
dump(new stdClass);         // OBJECT: ???
```

访问变量值的宏业非常好记：`Z_BVAL`获取布尔值，`Z_LVAL`获取长整型，`Z_DVAL`获取双精度。`S_STRVAL`宏会返回一个`char *`类型的字符串，然而`Z_STRLEN`会返回它的长度。可以利用`Z_RESVAL`宏获取资源ID，利用`Z_ARRVAL`获取数组的`HashTable *ht`成员。由于对象的获取会涉及到一些背景知识，所以这里不会涉及到。

当你想访问zval的值是，你需要总是使用宏来获取，而不能直接使用它的成员属性。因为这样维护了一层抽象的接口，并且能够提高代码可读性：举个例子来说，如果你直接使用`lval`成员来访问值的话，对于看代码的人来说无法确定你是在获取一个布尔值，长整形值还是一个资源ID。所以，如果你使用了`Z_BVAL`，`Z_LVAL`和`Z_RESVAL`宏的话，那么就不会有歧义了。此外还有一个原因就是为了兼容后续PHP版本对`zval`结构体内部的调整。

## 赋值

前文中提到的大部分宏可以用来访问`zval`结构体的成员，因此你可以使用它们来进行读取和赋值操作。举个例子，如下面的输出"hello world!"的函数：

```c
PHP_FUNCTION(hello_world) {
    Z_TYPE_P(return_value) = IS_STRING;
    Z_STRVAL_P(return_value) = estrdup("hello world!");
    Z_STRLEN_P(return_value) = strlen("hello world!");
};

/* ... */
    PHP_FE(hello_world, NULL)
/* ... */
```

执行`php -r "echo hello_world();"后，会输出`hello world!`到终端中。

在上面的例子中，我们队`return_value`进行了赋值，它是`PHP_FUNCTION`宏中的一个`zval *`类型的变量。在后面的章节中我们会详细的讨论这个变量，现在只需明白这个变量就是函数的返回值就够了。默认情况下，它是`IS_NULL`类型的。

使用宏来给zval进行赋值真是简单的不得了，但是有几件事情务必要记在心中：首先你需要明确`zval`的类型是有其成员变量`type`的值决定的。仅仅赋值是不够的（通过`Z_STRVAL`和`Z_STRLEN`宏），需要同时改变`type`成员的值。

此外，你需要清楚的是在大多数情况下，`zval`“本身有”值，同时相对于在它被赋值的语句块中，会有一个更长的声明周期。尽管有时候在处理临时的`zval`变量时并没有什么作用，但却是是这样的。

通过上面的例子，我们了解到`return_value`的生命周期会比当前的函数的声明周期要长（这很明显，否则没有人能拿到这个函数的返回值了），所以，`return_value`是不可以使用任何函数体内的临时变量。比如，如果这样写`Z_STRVAL_P(return_value) = "hello world!"`是错误的，因为这个字符串常量`"hello world"`将在函数体执行完毕后被销毁（在C中，被分配到栈中的变量都是如此 笔者注：如果你不了解C的内存分配，可以看看这篇[c语言stack(栈)和heap(堆)的使用详解](http://blog.ihuxu.com/the-usage-of-the-stack-and-heap-for-c/)）。

鉴于此我们需要使用`estrdup()`来拷贝字符串。这样一来将会在推上（heap）创建一个独立的拷贝。由于`zval`“拥有”它的值，在`zval`销毁时一定会释放这个拷贝。对于其他存储了复杂值的`zval`变量也同样有效。比如：如果你给数组赋值一个`HashTable *ht`，那么`zval`将会获取所有权并在自身销毁时释放它。当使用原始的数据类型时，比如整型或者双精度，你显然不必担心了，因为他们总是进行拷贝的。

最后，值得一提的是并不是所有的访问宏都会然会一个成员。比如下面的`Z_BVAL`宏：

```c
#define Z_BVAL(zval) ((zend_bool)(zval).value.lval)
```

因为这个宏里面包含了一个类型转化，所以你不能够直接这样写`Z_BVAL_P(return_value) = 1`。除了一些对象相关的宏之外，其他的访问宏都是可以进行赋值操作的。

实际上，你不必担心最后一点：由于对`zval`赋值是一件非常寻常的事情，所以PHP已经提供了另外一组宏来进行赋值操作。这些宏可以同时进行类型和值的赋值操作。使用这些宏重写前面的例子：

```c
PHP_FUNCTION(hello_world) {
    ZVAL_STRINGL(return_value, estrdup("hello world!"), strlen("hello world!"), 0);
}
```

由于几乎每次在给`zval`赋值时需要对字符串进行单独拷贝，所以`ZVAL_STRINGL`的最后一个参数（布尔值）可以进行配置。如果是0的话，那么就使用字符串本身，如果是1的话，那么就用`estrndup()`函数进行拷贝。所以，可以将代码改写成如下的样子：

```c
PHP_FUNCTION(hello_world) {
    ZVAL_STRINGL(return_value, "hello world!", strlen("hello world!"), 1);
}
```

此外，我们也不必手动计算字符串的长度，可以使用`ZVAL_STRING`宏在内部进行计算（没有`L`）：

```c
PHP_FUNCTION(hello_world) {
    ZVAL_STRING(return_value, "hello world!", 1);
}
```

如果你已经知道该字符串的长度（因为有可能以某种方式传给了你），你应该总是使用`ZVAL_STRINGL`宏来使用它，因为这样可以保证二进制安全。如果你不知道长度（或者你知道字符串不包含NUL字节，通常是在字面量的情况下），你可以使用`ZVAL_STRING`。

除了`ZVAL_STRING(L)`，还有几个用来赋值的宏，如下：

```c
ZVAL_NULL(return_value);

ZVAL_BOOL(return_value, 0);
ZVAL_BOOL(return_value, 1);
/* or better */
ZVAL_FALSE(return_value);
ZVAL_TRUE(return_value);

ZVAL_LONG(return_value, 42);
ZVAL_DOUBLE(return_value, 4.2);
ZVAL_RESOURCE(return_value, resource_id);

ZVAL_EMPTY_STRING(return_value);
/* = ZVAL_STRING(return_value, "", 1); */

ZVAL_STRING(return_value, "string", 1);
/* = ZVAL_STRING(return_value, estrdup("string"), 0); */

ZVAL_STRINGL(return_value, "nul\0string", 10, 1);
/* = ZVAL_STRINGL(return_value, estrndup("nul\0string", 10), 10, 0); */
```

指的注意的是这些宏可以进行赋值，但是不会销毁任何值，其中这些值有可能`zval`之前持有过。对于`return_value`变量来说这无关紧要，因为它在初始化时被赋值为`IS_NULL`类型（不需要被释放），但是在其他情况下，你需要在一开始就销毁这些老的值，我们将会在后面的章节中介绍如何操作。

