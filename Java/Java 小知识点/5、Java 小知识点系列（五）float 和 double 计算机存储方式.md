# float 和 double





## 1、问题产生

我们首先需要知道，float 是 4 个字节，32 位， double 是 8 个字节，64 位（类比 int 和 long）

**整型默认情况下是 int ，浮点型默认情况下是 double**



运行代码

```java
System.out.println(2.00 - 1.10);
```

输出结果：

```java
0.8999999999999999
```



为什么会产生这样的结果？

我们从底层浮点数的存储来看



## 2、浮点数在计算机中的存储  

|        | 符号位 | 指数   | 尾数   |
| ------ | ------ | ------ | ------ |
| float  | 1 bit  | 8 bit  | 23 bit |
| double | 1 bit  | 11 bit | 52 bit |



我们可以发现，在计算机存储的时候，只存储了 符号位、指数、尾数

那么这几个数是如何表示一个浮点数的呢？

我们以 float 来讲解：

我们需要知道，计算机只看得懂 二进制，因此无论什么存储都是需要转换为二进制的，浮点数也不例外



> ### 浮点数转换为二进制

将浮点数 2.625 转换为二进制

首先，浮点数中包含 整数 和 小数两部分，整数单独计算，小数单独计算

这里整数为 2，转换为二进制数为 10，小数部分为 0.625，转换结果为 101（具体转换图如下）

那么最终结果为 10.101

<img src="https://pic2.zhimg.com/80/v2-50a035c4d5d988dc8c955ac378c6643d_720w.jpg" style="zoom:70%;" />





> ### 将浮点数二进制转换为计算机存储方式

上面获取到 浮点数二进制为 10.101

先将小数点移到第一个非 0 的数的后面，那么 10.101 转变成 1.0101 * 2（进制数为 2，所以是 *2，而不是 *10）

那么可以根据这个形式得到以下信息

- **符号位：**正数，所以二进制数为 0（1bit）

- **阶码部分：**127 + 1 = 128，所以二进制数为 10000000（8bit）

- **尾数部分：**尾数即小数点后面的数  0101，由于计算机固定存储为 23 位，所以后面需要 补0，即 0101000000...（23bit）



这里我们可以对于 符号位 和 尾数部分 可能一眼能够看明白，

关于尾数部分，这里需要说明的一点是：小数的位置前面永远都是第一个非 0 的数 1，即尾数部分为 0101，那么前面必定存在一个 `1.`，这样从计算机中获取的时候，就是获取尾数部分，然后跟 `1.` 进行拼接，变成 `1.0101`，然后再根据指数部分确定原来小数点的位置，这是一个固定的形式，这也就是为什么我们上面 10.101 要将小数点左移变成 1.0101 的原因



阶码部分，就有点学问了

**阶码部分 代表 指数部分**，不过由于是 二进制，所以是 *2 的个数，正如我们十进制，是 *10 的个数

阶码有 8bit，那么能够表示的指数范围为 [0, 255]

但是由于浮点数协议将 0 和 255 作为特殊的用途，所以**指数表示范围为 [1, 254]**

由于存在负指数，所以它按照顺序将 [1, 254] 划分为 [-126, 127]

这很反直觉，因为对于我们来讲，特殊用途的 0 和 255 的二进制数是 00000000 和 11111111

这样的话，**根据 int 型表示正数负数的方式**，因为剔除了 0 和 -128，那么最终的指数范围不应该是 [-127, 0) ∪ (0, 127] 吗，

但是显然，不能这么做，因为指数存在 0 的形式，所以指数范围必须包含 0

这样的话，由于 00000000 已经不能够代表 0 了，那么就只能使用别的 二进制数代表 0



它的做法是，[1, 254] 中 1 代表 -126、2 代表 -125、。。。、 127 代表 0、128 代表 1、。。。、254 代表 127

即它是按照 [1, 254] 进行排序的，最小的 1 代表最小的 -126，这样换算下去，127 就代表 0 了，这个就是我们还没学习计算机前的想法了，而**不是按照 int 那种根据 高位 0 和 1 来决定正负**

由于使用了一位二进制数代表 0，所以只能从 正数或负数 中抽取出一位代表 0，这样指数范围为 [-126, 127] 或者 [-127, 126]

**最终选择了 [-126, 127]**

而 偏码 = 127，即代表了 127 代表指数 0，正数负数都是在这个偏码上进行运算，比如 上面的 1.0101 * 2，这里的指数是 1

所以它的阶码为 127 + 1 = 128，即 128 就表示指数 1



**因此我们就明白了计算机中是如何存储浮点数的**



> ### 1 中问题的产生原因

浮点数是有一定存储范围的，我们将一个十进制数存储进计算机的时候，实际上是转换为二进制数

上面的 1.10，整数部分 1 二进制就是 1 ，而小数 0.10 的转换如下：

0.1 * 2 = 0.2，取整数部分 0，余数为 0.2

0.2 * 2 = 0.4，取整数部分 0，余数为 0.4

0.4 * 2 = 0.8，取整数部分 0，余数为 0.8

0.8 * 2 = 1.6，取整数部分 1，余数为 0.6

。。。

。。。

我们可以看出，无论如何 0.1 都无法取到尽头，但是由于 double 类型的尾数部分只能存储 52 bit，因此导致超出的部分被舍去了

导致了精度丢失

而上面输出 0.8999 同样是精度丢失的问题，因为 0.1 无法精确转化为二进制数，只能无限接近，导致转换的值比 0.9 大，因此才出现了 0.0000000...1 的差距



**因此当需要使用浮点数比较时，需要设置一个 精确度 `exp`，然后 Math.abs(a - b) <= exp 的话就算作满足条件**

**在判断某个数是否为浮点数的时候，如果不转换为 String 判断是否存在小数点，那么就使用 exp 的精度差的形式判断**



## 3、为什么 float 和 double 之间的比较不能使用 ==

```java
float f1 = 0.5f;
double d1 = 0.5;

float f2 = 0.1f;
double d2 = 0.1;

System.out.println(f1 == d1); // true
System.out.println(f2 == d2); // false
```

上面代码出现不同情况的原因同样是因为精度丢失

由于 0.5 能够准确转换为 二进制数，因此 float 和 double 结果是一样的

但是 0.1 无法准确转换为 二进制数，而 float 的尾数位数 比 double 少，因此精度比 double 小，即转换结果为 f2 < d2