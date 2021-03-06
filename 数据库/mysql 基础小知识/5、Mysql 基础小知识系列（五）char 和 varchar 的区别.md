# char 和 varchar 的区别



[用char还是varchar？答案和原理都告诉你 - 简书](https://www.jianshu.com/p/a1aa86e17bf7)



mysql 使用的是 Unicode 字符集，根据选择的不同的编码方式，那么对应每个字符占用的 字节数 也不同

我们主要使用的是 UTF 系列的编码方式，它是一种可变长编码，即不是每个字符都是占用相同的字节数的

比如字母就占用 1B，常用汉字占用 3B，生僻汉字占用 4B，是根据 字符对应分配到的 unicode 的 唯一 ID 来进行 编码成二进制 的，是可变的。字母排在前面，肯定分配到的 ID 就小，所以就只需要使用 1B 来表示

char(11) 和 varchar(255) 的括号中的 11 和 255 **代表的是 字符长度，而不是 字节长度**，即**最多可以存储多少个字符**，它只看字符数，不看字节数，存储 3 个 1B 的字符 和 存储 3 个 3B 的字符都是一样的，不会说 可以存储 3 个 3B 的字符就意味着可以存储 9 个 1B 的字符

- 在 utf8 中，每个字符占用字节数为 1B - 3B，可以表示的字符有 字母 和 常用汉字，比如 char(3) 可以表示 汉字的 3 个常用汉字 `卧槽啊` 以及 3 个字母 `abc`，每个汉字 和 字母都是 1 个字符，字母占用 1B，汉字占用 3B
- 在 utf8mb4 中，每个字符为 4B，可以表示的字符有 字母、常用汉字、生僻汉字 和 表情，比如 char(3) 可以表示 汉字的 3 个汉字 `卧槽啊` 以及 3 个字母 `abc`，每个汉字 和 字母都是 1 个字符，字母占用 1B，汉字占用 3B - 4B



这里扩展一下，int(11) 并不是表示 int 大小为 11B，实际上这里的 int 跟 Java 的 int 一样，占用 4B（32bit），最大表示范围为 [Integer.MIN_VALUE，Integer.MAX_VALUE]；

那么这个 int(11) 是什么意思呢？

它表示显示的数据宽度，比如我们设置 int(4)，我们插入的数据是 12，那么它显示出来的就是 0012，它会自动填充 0 达到设置的宽度，当然，我们现在使用的 navicate 自动进行处理，不会显示出 0



好了，现在回到 char 和 varchar 的问题上来

varchar 和 char 字符间的存储如下：

<img src="https://upload-images.jianshu.io/upload_images/3433021-3e101de4685cfa45.jpg?imageMogr2/auto-orient/strip|imageView2/2/format/webp" style="zoom:50%;" />

varchar 前面需要 1B 来记录后面的字符的真实长度，而 char 则不需要，如果存储的字符长度不够，那么会自动使用空格进行填充

因此，mysql 读取每个 varchar 时，都需要先从磁盘上读取这 1B 来获取 字符串的真实长度（如果长度超过 255，那么 1B 是不够表示的，需要使用 2B，所以一般我们使用 varchar 的时候，navicate 都是默认设置为 255），然后根据真实长度再去读取

而 char 的话由于字符串必定是固定长度了， mysql 每次都只需要按照固定长度直接读取即可