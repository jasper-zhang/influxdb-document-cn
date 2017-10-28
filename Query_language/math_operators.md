# 数学运算符

数学运算符遵循标准的操作顺序。也就是说，圆括号优先于除法和乘法，而乘除法优先于加法和减法。例如`5 / 2 + 3 * 2 = (5 / 2) + (3 * 2)`和`5 + 2 * 3 - 2 = 5 + (2 * 3) - 2`。

## 数学运算符
### 加法
加一个常数。

```
SELECT "A" + 5 FROM "add"
SELECT * FROM "add" WHERE "A" + 5 > 10
```

两个字段相加。

```
SELECT "A" + "B" FROM "add"
SELECT * FROM "add" WHERE "A" + "B" >= 10
```

### 减法
减法里带常数。

```
SELECT 1 - "A" FROM "sub"
SELECT * FROM "sub" WHERE 1 - "A" <= 3
```

两个字段做减法。

```
SELECT "A" - "B" FROM "sub"
SELECT * FROM "sub" WHERE "A" - "B" <= 1
```

### 乘法
乘以一个常数。

```
SELECT 10 * "A" FROM "mult"
SELECT * FROM "mult" WHERE "A" * 10 >= 20
```

两个字段相乘。

```
SELECT "A" * "B" * "C" FROM "mult"
SELECT * FROM "mult" WHERE "A" * "B" <= 80
```

乘法和其他运算符混用。

```
SELECT 10 * ("A" + "B" + "C") FROM "mult"
SELECT 10 * ("A" - "B" - "C") FROM "mult"
SELECT 10 * ("A" + "B" - "C") FROM "mult"
```

### 除法
除法里带常数。

```
SELECT 10 / "A" FROM "div"
SELECT * FROM "div" WHERE "A" / 10 <= 2
```

两个字段相除。

```
SELECT "A" / "B" FROM "div"
SELECT * FROM "div" WHERE "A" / "B" >= 10
```

除法和其他运算符混用。

```
SELECT 10 / ("A" + "B" + "C") FROM "mult"
```

### 求模
模一个常数。

```
SELECT "B" % 2 FROM "modulo"
SELECT "B" FROM "modulo" WHERE "B" % 2 = 0
```

两个字段求模。

```
SELECT "A" % "B" FROM "modulo"
SELECT "A" FROM "modulo" WHERE "A" % "B" = 0
```

### 按位与
你可以在任何整数和布尔值中使用这个操作符，无论是字段或常数。该操作符不支持浮点数或字符串数据类型。并且不能混合使用整数和布尔值。

```
SELECT "A" & 255 FROM "bitfields"
SELECT "A" & "B" FROM "bitfields"
SELECT * FROM "data" WHERE "bitfield" & 15 > 0
SELECT "A" & "B" FROM "booleans"
SELECT ("A" ^ true) & "B" FROM "booleans"
```

### 按位或
你可以在任何整数和布尔值中使用这个操作符，无论是字段或常数。该操作符不支持浮点数或字符串数据类型。并且不能混合使用整数和布尔值。

```
SELECT "A" | 5 FROM "bitfields"
SELECT "A" | "B" FROM "bitfields"
SELECT * FROM "data" WHERE "bitfield" | 12 = 12
```

### 按位异或
你可以在任何整数和布尔值中使用这个操作符，无论是字段或常数。该操作符不支持浮点数或字符串数据类型。并且不能混合使用整数和布尔值。

```
SELECT "A" ^ 255 FROM "bitfields"
SELECT "A" ^ "B" FROM "bitfields"
SELECT * FROM "data" WHERE "bitfield" ^ 6 > 0
```

### 数学运算符的常见问题
#### 问题一：带有通配和正则的数学运算符
InfluxDB在`SELECT`语句中不支持正则表达式或通配符。下面的查询是不合法的，系统会返回一个错误。

数学运算符和通配符一起使用。

```
> SELECT * + 2 FROM "nope"
ERR: unsupported expression with wildcard: * + 2
```

数学运算符和带函数的通配符一起使用。

```
> SELECT COUNT(*) / 2 FROM "nope"
ERR: unsupported expression with wildcard: count(*) / 2
```

数学运算符和正则表达式一起使用。

```
> SELECT /A/ + 2 FROM "nope"
ERR: error parsing query: found +, expected FROM at line 1, char 12
```

数学运算符和带函数的正则表达式一起使用。

```
> SELECT COUNT(/A/) + 2 FROM "nope"
ERR: unsupported expression with regex field: count(/A/) + 2
```

#### 问题二：数学运算符和函数
在函数里面使用数学运算符现在不支持。

例如：

```
SELECT 10 * mean("value") FROM "cpu"
```

是允许的，但是

```
SELECT mean(10 * "value") FROM "cpu"
```

将会返回一个错误。

>InfluxQL支持子查询，这样可以达到函数里面使用数学运算符相同的功能。

## 不支持的运算符
### 比较
在`SELECT`中使用任意的`=,!=,<,>,<=,>=,<>`，都会返回空。

### 逻辑运算符
使用`!|,NAND,XOR,NOR`都会返回解析错误。

此外，在一个查询的`SELECT`子句中使用`AND`和`OR`不会像数学运算符只产生空的结果，因为他们是InfluxQL的关键字。然而，你可以对布尔字段使用按位运算`&`，`|`和`^`。

### 按位非
没有按位NOT运算符，因为你期望的结果取决于位的宽度。InfluxQL不知道位的宽度，所以无法实现适当的按位NOT运算符。

例如，如果位是8位的，那么你来说，1代表`0000 0001`。

按位非本应该返回`1111 1110`，即整数254。

然而，如果位是16位的，那么1代表`0000 0000 0000 0001`。按位非本应该返回`1111 1111 1111 1110`，即整数65534。

#### 解决方案
你可以使用`^`加位数的数值来实现按位非的效果：

对于8位的数据：

```
SELECT "A" ^ 255 FROM "data"
```

对于16位数据：

```
SELECT "A" ^ 65535 FROM "data"
```

对于32位数据：

```
SELECT "A" ^ 4294967295 FROM "data"
```

里面的每一个常数的计算方式为`(2 ** width) - 1`。