# javascript中“~~”运算符的特性研究

-----------------------

这几天要弄一个AMD模块的加载器，在处理依赖的数组对象时用到了indexOf方法和forEach方法，不过这两个方法在IE6/7/8下是不支持，找了两个现成的解决方案，都是在Array原型上扩展，比较实用的：

```javascript
    // forEach for IE678
    if (!Array.prototype.forEach) {
        Array.prototype.forEach = function(fun /*, thisp*/ ) {
            var len = this.length;
            if (typeof fun != "function")
                throw new TypeError();

            var thisp = arguments[1];
            for (var i = 0; i < len; i++) {
                if (i in this)
                    fun.call(thisp, this[i], i, this);
            }
        };
    }
    // indexOf for IE678
    if (!Array.prototype.indexOf) {
        Array.prototype.indexOf = function(el, index) {
            var n = this.length >>> 0,
                i = ~~index;
            if (i < 0) i += n;
            for (; i < n; i++)
                if (i in this && this[i] === el) return i;
            return -1;
        }
    }

```

在Array添加indexOf方法 中，发现 `i = ~~index` 比较有意思，`~~` 是由两个 `~` **按位“非”** 运行符组成，它有什么作用呢？

先来看看以下的测试代码

**测试代码：**
```javascript
    console.log(~~undefined); //0  Number
    console.log(~~null); //0  Number
    console.log(~~true); //1 Number
    console.log(~~false); //0  Number
    console.log(~~''); //0  Number
    console.log(~~'123'); //123  Number
    console.log(~~'0abc'); //0  Number
    console.log(~~'1abc'); //0  Number
```
## 测试结果分析：

### 结论1：
### `~~` 在处理 `undefined`，`null`，`false` 或 `''`(空字符) 时，结果都返回 `0`

这个比较有用，比如：定义一个根据某个条件的真或假来返回1或0的变量
一般就这么写：
```javascript
    var condition;
    var a = condition ? 1 : 0;
```
现在可以这么写：
```javascript
    var condition;
    var b = ~~condition;
```
因此，用 `~~` 来处理条件为 `空字符串` 或 `布尔值` 时，实用性是非常不错的。

### 结论2：
### 对象进行 `~~` 运算后返回的都是数字 ,也存在隐式变量类型转换

测试代码：
```javascript
    console.log("1234" - 0);   //returns   1234,number
    console.log(~~"1234");   //returns   1234,number
```
也就是，除了 `-` 减号运算符外，`~~` 也存在隐式变量类型转换，两者的结果都是数字

### 结论3：
###在绝大多数情况下，对数字或数字型文本的处理结果，`~~` 与 `parseInt` 作用几乎一致

`~~` 与 `parseInt` 的对比测试
```javascript
    // parseInt
    console.log(parseInt("1234"));   //returns   1234,number
    console.log(parseInt("1234abcd"));   //returns   1234

    console.log(parseInt("0xA"));   //returns   10
    console.log(parseInt(0xA));   //returns   10

    console.log(parseInt("22.5"));   //returns   22
    console.log(parseInt(22.5));   //returns   22

    console.log(parseInt(010));   //returns   8
    console.log(parseInt("010"));   //returns   10

    console.log(parseInt(017));   //returns   15
    console.log(parseInt("017"));   //returns   17
    console.log(parseInt(018));   //returns   18
    console.log(parseInt("018"));   //returns   18

    // ‘~~’运算
    console.log(~~"1234");   //returns   1234
    console.log(~~"1234abcd");   //returns   0

    console.log(~~"0xA");   //returns   10
    console.log(~~0xA);   //returns   10

    console.log(~~"22.5");   //returns   22
    console.log(~~22.5);   //returns   22

    console.log(~~010);   //returns   8
    console.log(~~"010");   //returns   10

    console.log(~~017);   //returns   15
    console.log(~~"017");   //returns   17
    console.log(~~018);   //returns   18
    console.log(~~"018");   //returns   18
```

上面的测试，可以看到 `~~` 与 `parseInt` 只有一种情况不相同，即处理数字+字母的字符串时结果不一致外，其它时候两者的作用是一样的，这个结论很有意思。

--------------------------

# javascript中“~~”运算符的特性研究2

-------------------


之前对于 `~~` 运算符在 javascript 中的一些特性进了一些研究，有几点结论：

## 结论1：
### `~~` 在处理 `undefined`，`null`，`false` 或 `''`(空字符) 时，结果都返回 `0`

## 结论2：
### 对象进行 `~~` 运算后返回的都是数字 ,也存在隐式变量类型转换

## 结论3：
###在绝大多数情况下(除了处理数字+字母的字符串时结果不一致外)，对数字或数字型字符串的处理结果，`~~` 与`parseInt` 作用是一致的。

这些结论是非常有意思的，但为什么 `~~` 有这样的特性呢？这值得继续研究。

拆分来看，得先搞清楚 ~ 运算符是什么。

 ~ 是 ECMAScript 标准中的位运算符之一 —— `NOT` 的实例表达，此外还与 按位与AND (&)和 按位或OR (|)，它们都是在数字底层（即表示数字的 32 个数位）进行操作的。

### 重温整数

ECMAScript 整数有两种类型，即有符号整数（允许用正数和负数）和无符号整数（只允许用正数）。在 ECMAScript 中，所有整数字面量默认都是有符号整数，这意味着什么呢？

有符号整数使用 31 位表示整数的数值，用第 32 位表示整数的符号，0 表示正数，1 表示负数。数值范围从 -2147483648 到 2147483647。

可以以两种不同的方式存储二进制形式的有符号整数，一种用于存储正数，一种用于存储负数。正数是以真二进制形式存储的，前 31 位中的每一位都表示 2 的幂，从第 1 位（位 0）开始，表示 20，第 2 位（位 1）表示 21。没用到的位用 0 填充，即忽略不计。
例如，下图展示的是数 18 的表示法。
![整数 18 的32位表示法][2]
32 位二进制表示的有符号整数
18 的二进制版本只用了前 5 位，它们是这个数字的有效位。把数字转换成二进制字符串，就能看到有效位：

```javascript
var iNum = 18;
alert(iNum.toString(2));    //输出 "10010"
```

这段代码只输出 "10010"，而不是 18 的 32 位表示。其他的数位并不重要，因为仅使用前 5 位即可确定这个十进制数值。如下图所示：

![5 位二进制表示的整数 18][3]
5 位二进制表示的整数 18

### 整数的求负过程

负数也存储为二进制代码，不过采用的形式是二进制补码。计算数字二进制补码的步骤有三步：
确定该数字的非负版本的二进制表示（例如，要计算 -18的二进制补码，首先要确定 18 的二进制表示）
求得二进制反码，即要把 0 替换为 1，把 1 替换为 0
在二进制反码上加 1
要确定 -18 的二进制表示，首先必须得到 18 的二进制表示，如下所示：

```markup
0000 0000 0000 0000 0000 0000 0001 0010
```

接下来，计算二进制反码，如下所示：

```markup
1111 1111 1111 1111 1111 1111 1110 1101
```

最后，在二进制反码上加 1，如下所示：

```markup
1111 1111 1111 1111 1111 1111 1110 1101
                                    + 1
---------------------------------------
1111 1111 1111 1111 1111 1111 1110 1110
```
因此，-18 的二进制表示即 1111 1111 1111 1111 1111 1111 1110 1110。

在处理有符号整数时，开发者不能访问 31 位的，因此当把负整数转换成二进制字符串后，ECMAScript 并不以二进制补码的形式显示，而是用数字绝对值的标准二进制代码前面加负号的形式输出。例如：
```javascript
var iNum = -18;
alert(iNum.toString(2));    //输出 "-10010"
```
这段代码输出的是 "-10010"，而非二进制补码，这是为避免访问位 31，ECMAScript 用一种简单的方式处理整数，使得开发者不必关心它们的用法。

另一方面，无符号整数把最后一位作为另一个数位处理。在这种模式中，第 32 位不表示数字的符号，而是值 231。由于这个额外的位，无符号整数的数值范围为 0 到 4294967295。对于小于 2147483647 的整数来说，无符号整数看来与有符号整数一样，而大于 2147483647 的整数则要使用位 31（在有符号整数中，这一位总是 0）。

把无符号整数转换成字符串后，只返回它们的有效位。

注意：所有整数字面量都默认存储为有符号整数。只有 ECMAScript 的位运算符才能创建无符号整数。

### 位运算 NOT

位运算 NOT 由否定号（~）表示，它是 ECMAScript 中为数不多的与二进制算术有关的运算符之一。
位运算 NOT 是三步的处理过程：
- 把运算数转换成 32 位数字
- 把二进制数转换成它的二进制反码
- 把二进制数转换成浮点数

例如：
```javascript
var iNum1 = 25;     //25 等于 00000000000000000000000000011001
var iNum2 = ~iNum1; //转换为 11111111111111111111111111100110
alert(iNum2);       //输出 "-26"
```

> 位运算 NOT 的数学意义是：对数字求负，然后减 1。

```javascript
var iNum1 = 25;
var iNum2 = -iNum1 -1;
alert(iNum2);   //输出 -26
```
> 即按二进制形式将所有数字取反，

比如：对25进行按位非运算，然后根据 `~` 的数字意义进行解析，运算过程是这样的

> `~25 = (-25) - 1 = -26`

如果我们对-26进行 ~ 运算，结果也正好等1，这个过程的数学解析如下：
-2求负之后的结果再-1，即

> `~-26 = -(-26) - 1 =26 - 1 = 25`

> **结论：**`对于任意一个整数（包括正整数和负整数），连续的两次按位非运算的结果都是等于它本身。`

这个结果似乎没什么意义，说了等于没说似的……但是，若果我们将 ~~ 作用于 字符串时，~ 的数学运算过程就很容易理解，`~~`的两个特性：

## 1，为什么 `~~` 具有隐式的变量类型转换?

我是这样理解的：
因为 `~` 其实是对其作用的对象进行求负，然后减 1，这个过程中 `-` 运算符在对象上作用了两次，而我们应该知道 ** `-` 运算符 在javascript中，存在隐式的类型转换**，而 '~' 正好继承了这个特性，因此它存在隐式的格式转换的特性

### “`-`”运算符
“`-`”可以是一元运算符（取负），也可以是二元（减法运算）的。如
```javascript
var a = 1, b = '';
var c = a - b;
alert(typeof c); //--> number

var aa = '123',bb = '123abc';
alert(typeof ~aa); // --> number
alert(typeof ~bb); // --> number
```
这里有一篇文章讲述了[JavaScript中的隐式类型转换][4]，说得比较清楚，这里不再赘述。

## 2，除了处理数字+字母的字符串时结果不一致外，对数字或数字型字符串的处理结果，`~~` 与`parseInt` 作用是一致的

### 理解 `~~` 与`parseInt` 一致性

比如 a = '123'；那么，~~a的运算过程是这样的：

首看~a，过程是对'123'字符串求负，然后-1，即：

```
~a = -'123' - 1 = -123 - 1
~a = -124
```
那么，~~a的运算过程其实可以分解为对 `~a（即-124）求负再-1`：

```
~~a = ~(~a) = ~(-124) = -(-124) - 1 = 124 -1
~~a = 123
``

公式就是
```
~~A = -(-A - 1) -1 = -(-A) -(-1) - 1
```
在这个公式中，最重要的是对A的求负运算。



  [2]: http://www.w3school.com.cn/i/ct_js_integer_binary_signed_32bits.gif
  [3]: http://www.w3school.com.cn/i/ct_js_integer_binary_number18.gif
  [4]: http://www.cnblogs.com/snandy/archive/2011/03/18/1987940.html