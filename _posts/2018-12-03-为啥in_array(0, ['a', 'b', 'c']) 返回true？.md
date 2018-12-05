# 为啥in_array(0,['a', 'b', 'c']) 返回为true？

问题背景：  
在具体PHP编码过程中，总会出现一些我们认为不可能的情况，如下几例：

```
in_array(0, ['a', 'b', 'c'])	 // 返回bool(true)，相当于数组中有0 
array_search(0, ['a', 'b', 'c']) // 返回int(0)，相当于是第一个值的下标 
0 == 'abc'			// 返回bool(true)，相当于等值 
```
但是，直观上看， 0并没有包含在['a', 'b', 'c']数组中，也不会等于'abc'这个字符串。
那怎么解释上述的返回结果呢？

## 1、类型转换
究其原因：**在数据比较前，PHP做了类型转换**。
引用PHP官网关于“String conversion to numbers”解释如下：

{% highlight markdown %}
When a string is evaluated in a numeric context, the resulting value and type are determined as follows.

If the string does not contain any of the characters '.', 'e', or 'E' and the numeric value fits into integer type limits (as defined by PHP_INT_MAX), the string will be evaluated as an integer. In all other cases it will be evaluated as a float.

The value is given by the initial portion of the string. If the string starts with valid numeric data, this will be the value used. Otherwise, the value will be 0 (zero). Valid numeric data is an optional sign, followed by one or more digits (optionally containing a decimal point), followed by an optional exponent. The exponent is an 'e' or 'E' followed by one or more digits.
{% endhighlight %}

文章开篇例子中，string类型数据第一个字符不是数字，就会转换为0，例如：

```
echo intval('abc');  // 输出0
```
in_array()和array_search()默认都是松散比较，相当于==，即得到true。

## 2、严格比较
那怎么得到我们预期的结果呢？
使用严格比较，如下所示：

```
in_array(0, ['a', 'b', 'c'], true)		// 返回bool（false）
array_search(0, ['a', 'b', 'c'], true) 	// 返回bool（false）
0 === 'abc'								// 返回bool（false）
```

## 3、false 与 null
那么，如果用false和null与字符串数组比较，结果会如何呢？

```
in_array(null, ['a', 'b', 'c'])	// 返回bool（false）
in_array(false, ['a', 'b', 'c']) // 返回bool（false）
```
null与false做比较值，字符串数组是不会转换为int型的。

## 4、数组中有true
另一个看起来比较奇怪的现象

```
in_array('a', [true, 'b', 'c'])		// 返回bool(true)，相当于数组里面有'a'
array_search('a', [true, 'b', 'c']) // 返回int(0),相当于找到了字符串'a'
```
至于原因：松散比较下，任何string都等于true，若强制校验，用严格比较即可。

<br>

_The end_