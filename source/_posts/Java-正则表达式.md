---
title: Java-正则表达式
abbrlink: 82880790
date: 2020-01-15 11:46:49
categories:
  - Java
  - 基础
---


{% cq %}
正则表达式(Regular Expression)是一种文本模式，包括普通字符（例如，a 到 z 之间的字母）和特殊字符（称为"元字符"）。

正则表达式使用单个字符串来描述、匹配一系列匹配某个句法规则的字符串。

摘自[菜鸟-正则表达式](https://www.runoob.com/regexp/regexp-tutorial.html)

{% endcq %}

<!-- More -->



# 正则表达式

## 普通字符

- 普通字符包括没有显式指定为元字符的所有可打印和不可打印字符。包括所有大小写字母、数字、标点符号和其他符号。

## 非打印字符

- 非打印字符也可以是正则表达式的组成部分。下表列出了表示非打印字符的转义序列

| 字符 | 说明                                                         |
| ---- | ------------------------------------------------------------ |
| \cx  | 匹配**由x指明的控制字符**。例如，`\cM`匹配一个Control-M或回车符。x的值不许为A-Z或a-z之一。否则，将c视为一个愿意的`c`字符 |
| \f   | 匹配一个**换页符**。等价于`\x0C`和`\cl`                      |
| \n   | 匹配一个**换行符**。等价于`\x0a`和`\cj`                      |
| \r   | 匹配一个**回车符**。等价于`\x0d`和`\cM`                      |
| \s   | 匹配任何**空白字符**，包括空格、制表符、换页符等等。等价于`[\f\n\r\t\v]` |
| \S   | 匹配任何**非空白字符**                                       |
| \t   | 匹配一个**制表符**，等价于`\x09`和`\cl`                      |
| \v   | 匹配一个**垂直制表符**。等价于`\x0b`和`\cK`                  |



## 元字符

| 字符 | 说明                                                         |
| ---- | ------------------------------------------------------------ |
| *    | 匹配前面的子表达式零次或多次。要匹配`*`字符，需要转移`\*`    |
| +    | 匹配前面的子表达式一次或多次。                               |
| .    | 匹配除换行符`\n`之外的任何单字符                             |
| ？   | 匹配前面的子表达式零次或一次，或指明一个**非贪婪(按最小次数进行匹配)**限定符。 |
| ^    | 匹配输入字符串的开始位置，在方括号表达式中使用时，表示语法逻辑**非** |
| &    | 匹配字符串的结尾位置                                         |
| ()   | 标记一个子表达式的开始和结束位置。子表达式可以获取供以后使用 |
| \    | 将下一个字符标记为特殊字符、或原义字符、或向后引用、或八进制转义符。例如，`n`匹配字符`'n'`。`\n`匹配换行符。序列`\\`匹配`'\'` |
| [    | 标记一个中括号表达式的开始                                   |
| {    | 标记限定符表达式的开始                                       |
| \|   | 指代逻辑语法**或**                                           |

## 限定符

- 限定符用来指定正则表达式匹配次数

| 字符  | 说明                                                         |
| ----- | ------------------------------------------------------------ |
| *     | 匹配前面的子表达式零次或多次。例如`zo*`可以匹配`"z"`以及`"zoo"`。`*`等价于`{0,}` |
| +     | 匹配前面的子表达式一次或多次                                 |
| ?     | 匹配前面的子表达式零次或一次。例如`do(es)`可以匹配`do`、`does`中的`does`、`doxy`中的`do`。等价于`{0,1}` |
| {n}   | 表示匹配确定的n次。例如`o{2}`不能匹配`Bob`中的`o`,但是能匹配`food`中的`oo` |
| {n,}  | n 是一个非负整数。至少匹配n 次。例如，'o{2,}' 不能匹配 "Bob" 中的 'o'，但能匹配 "foooood" 中的所有 o。'o{1,}' 等价于 'o+'。'o{0,}' 则等价于 'o*'。 |
| {n,m} | m 和 n 均为非负整数，其中n <= m。最少匹配 n 次且最多匹配 m 次。例如，"o{1,3}" 将匹配 "fooooood" 中的前三个 o。'o{0,1}' 等价于 'o?'。请注意在逗号和两个数之间不能有空格。 |

`*`和`+`限定符都是贪婪的，它们会尽可能多的匹配文字，在它们后面加一个`？`就可以实现非贪婪匹配(最小匹配)

## 定位符

定位符能够将正则比倒是固定到行首或行尾

| 字符 | 说明                                 |
| ---- | ------------------------------------ |
| ^    | 匹配字符串开始的位置                 |
| $    | 匹配输入字符串结束的位置             |
| \b   | 匹配一个单词边界，即字与空格间的位置 |
| \B   | 非单词边界匹配                       |

- 举例

  - 文本内容如下:

    ```tex
    Chapter 1
    Chapter 2
    Chapter 3
    Chapter 3的概述
    	概述内容
    	Chapter 2中说过，你不是一个人
    Chapter 4
    	详见下次讲解Chapter 5,terminal 的应用
    Chapter 19
    	aptitude 是个单词
    ```

  - 正则表达式如下:

    1. `Chapter [1-9][0-9]{0,1}`
    2. `^Chapter [1-9][0-9]{0,1}`
    3. `^Chapter [1-9][0-9]{0,1}$`
    4. `\bCha`
    5. `ter\b`
    6. `\Bapt`

## 选择

用圆括号将所有选择项括起来，相邻的选择项之间用|分隔。但用圆括号会有一个副作用，使相关的匹配会被缓存，此时可用?:放在第一个选项前来消除这种副作用。

其中 **?:** 是非捕获元之一，还有两个非捕获元是 **?=** 和 **?!**，这两个还有更多的含义，前者为正向预查，在任何开始匹配圆括号内的正则表达式模式的位置来匹配搜索字符串，后者为负向预查，在任何开始不匹配该正则表达式模式的位置来匹配搜索字符串。

## 反向引用

对一个正则表达式模式或部分模式两边添加圆括号将导致相关匹配存储到一个临时缓冲区中，所捕获的每个子匹配都按照在正则表达式模式中从左到右出现的顺序存储。缓冲区编号从 1 开始，最多可存储 99 个捕获的子表达式。每个缓冲区都可以使用 **\n** 访问，其中 n 为一个标识特定缓冲区的一位或两位十进制数。

- 举例

  - 文本内容如下

  ```tex
  Is is the cost of of gasoline going up up?
  ```

  - 正则表达式为`\b([a-c]|[c-z]+) \1\b`
  
- 说明，关于分组引用，可以查看匹配并获取的左小括号，是第几个就是第几组

## 元字符列表

| 字符         | 描述                                                         |
| ------------ | ------------------------------------------------------------ |
| (pattern)    | 匹配pattern并获取匹配结果。产生的匹配结果可以通过反向引用来使用。 |
| (?:pattern)  | 匹配 pattern 但不获取匹配结果，也就是说这是一个非获取匹配，不进行存储供以后使用。这在使用 "或" 字符 `(\|)`来组合一个模式的各个部分是很有用。例如， `industr(?:y|ies)`就是一个比`industry|industries`更简略的表达式 |
| (?=pattern)  | look ahead positive assert<br/>**a(?=b)** : 匹配后面有 b 的 a |
| (?!pattern)  | look ahead negative assert<br/>**a(?!b)** : 匹配后面没有b的a |
| (?<=pattern) | look behind positive assert<br/>**(?<=b)a** : 匹配前面有b的a |
| (?<!pattern) | look behind negative assert<br/>**(?<!b)a** : 匹配前面没有b的a |
| x\|y         | 匹配x或y                                                     |
| [abc]        | 字符合集，匹配其中任意一个字符                               |
| [^abc]       | 非字符合集，匹配不等于合集中任意字符的字符                   |
| [a-z]        | 字符范围。匹配指定范围内的任意字符。例如，'[a-z]' 可以匹配 'a' 到 'z' 范围内的任意小写字母字符。 |
| [^a-z]       | 负值字符范围。匹配任何不在指定范围内的任意字符。例如，'[^a-z]' 可以匹配任何不在 'a' 到 'z' 范围内的任意字符。 |



## 运算符优先级

- 正则表达式从左到右进行计算，并遵循优先级顺序。下面从高到低列出运算符：

| 运算符                    | 说明                                                         |
| ------------------------- | ------------------------------------------------------------ |
| \                         | 转义符                                                       |
| (),(?:),(?=),[]           | 圆括号和方括号                                               |
| *,+,?,{n},{n,},{n,m}      | 限定符                                                       |
| ^,$,\任何元字符、任何字符 | 定位点和序列                                                 |
| \|                        | 替换“或”操作<br/>字符具有高于替换运算符的优先级，是的m\|food匹配`"m"`或`"food"`。若要匹配`mood`或`food`,应使用括号创建子表达式，即`(m|f)ood` |



# Java中的正则表达式

## 类

- Pattern 类

  `Pattern`对象是一个正则表达式的编译表示。没有公共构造方法，想要创建需要调用其静态方法，传入一个正则表达式作为参数。

- Matcher类

  `Matcher`对象是对输入字符串进行解释和匹配操作的引擎，与`Pattern`类一样，Matcher没有公共构造方法。需要调用`#pattern.matcher`方法获取一个`Matcher`对象

- PatternSyntaxException

  `PatternSyntaxException`是一个非强制异常类，它表示一个正则表达式模式中的语法错误

## 捕获组

- 捕获组是把多个字符当一个单元进行处理的方法，它通过对括号内的字符分组来创建

## 实例

```java
import java.util.regex.Matcher;
import java.util.regex.Pattern;
 
public class RegexMatches
{
    public static void main( String args[] ){
 
      // 按指定模式在字符串查找
      String line = "This order was placed for QT3000! OK?";
      String pattern = "(\\D*)(\\d+)(.*)";
 
      // 创建 Pattern 对象
      Pattern r = Pattern.compile(pattern);
 
      // 现在创建 matcher 对象
      Matcher m = r.matcher(line);
      if (m.find( )) {
         System.out.println("Found value: " + m.group(0) );
         System.out.println("Found value: " + m.group(1) );
         System.out.println("Found value: " + m.group(2) );
         System.out.println("Found value: " + m.group(3) ); 
      } else {
         System.out.println("NO MATCH");
      }
   }
}
```

- 运行结果如下：

```java
Found value: This order was placed for QT3000! OK?
Found value: This order was placed for QT
Found value: 3000
Found value: ! OK?
```

- 正则表达式语法

> 在Java中，`\\`表示一个斜线，即匹配数字需要使用`\\d`

## Mathcer的方法

### 索引方法

索引方法提供了有用的索引值，精确表明输入字符串中在哪能找到匹配：

| 序号 | 方法及说明                                                   |
| ---- | ------------------------------------------------------------ |
| 1    | **public int start()**<br/>返回以前匹配的初始索引            |
| 2    | **public int start(int group)**<br/>返回在以前的匹配操作期间，由给定组所捕获的子序列的初始索引 |
| 3    | **public int end()**<br/>返回最后匹配字符之后的偏移量        |
| 4    | **public int end(int group)**<br/>返回在以前的匹配操作期间，由给定组所捕获子序列的最后字符之后的偏移量 |



### 研究方法

研究方法用来检查输入字符串并返回一个布尔值，表示是否找到该模式

| 序号 | 方法及说明                                                   |
| ---- | ------------------------------------------------------------ |
| 1    | **public boolean lookingAt()**<br/>尝试将从区域开头开始的输入序列与该模式匹配 |
| 2    | **public boolean find()**<br/>尝试查找与该模式匹配的输入序列的下一个子序列 |
| 3    | **public boolean find(int start)**<br/>重置此匹配器，然后尝试查找该匹配模式、从指定索引开始的输入序列的下一个子序列 |
| 4    | **public boolean matches()**<br/>尝试将整个区域与模式匹配    |



### 替换方法

替换方法是替换输入字符串里文本的方法：

| 序号 | 方法及说明                                                   |
| ---- | ------------------------------------------------------------ |
| 1    | **public Matcher appendReplacement(StringBuffer sb, String replacement)**<br/>实现非终端添加和替换步骤 |
| 2    | **public StringBuffer appendTail(StringBuffer sb)**<br/>实现终端添加和替换步骤 |
| 3    | **public String replaceAll(String replacement)**<br/>替换模式与给定替换字符串相匹配的输入序列的每一个子序列 |
| 4    | **public String replaceFirst(String replacement)**<br/>替换模式与给定替换字符串匹配的输入序列的第一个子序列 |
| 5    | **public static String quoteReplacement(String s)**<br/>返回指定字符串的字面替换字符串。这个方法返回一个字符串，就像传递给`Matcher`类的`appendReplacement`方法一个字面字符串一样工作 |



### start 和 end 方法

```java
    public static void testStartEnd() {
        String regex = "\\bcat\\b";
        String input = "cat cat cat cattie cat";

        Pattern p = Pattern.compile(regex);
        Matcher m = p.matcher(input);

        int count = 0;
        while (m.find()) {
            count++;
            System.out.println("Match number " + count);
            System.out.println("start() : " + m.start());
            System.out.println("end() : " + m.end());
        }
    }
```

- 运行结果如下：

  ```java
  Match number 1
  start(): 0
  end(): 3
  Match number 2
  start(): 4
  end(): 7
  Match number 3
  start(): 8
  end(): 11
  Match number 4
  start(): 19
  end(): 22
  ```

### matches 和 lookingAt方法

`matches`和`lookingAt`方法都用来尝试匹配一个输入序列模式。不同的是`matches`要求整个序列都匹配，而`lookingAt`不要求,但是`lookingAt`要求从第一个字符就开始匹配。

```java
    private static void testMatcherMethod(){
        String regex = "foo";
        String input1 = "foo";
        String input2 = "foooooooo";
        String input3 = "ooofooooo";

        Pattern p = Pattern.compile(regex);
        Matcher m1 = p.matcher(input1);
        Matcher m2 = p.matcher(input2);
        Matcher m3 = p.matcher(input3);

        System.out.println("m1.lookingAt : "+m1.lookingAt());
        System.out.println("m2.lookingAt : "+m2.lookingAt());
        System.out.println("m3.lookingAt : "+m3.lookingAt());

        System.out.println("m1.matches : "+m1.matches());
        System.out.println("m2.matches : "+m2.matches());
        System.out.println("m3.matches : "+m3.matches());
    }
```

### replaceFirst和replaceAll方法

replaceFirst 和 replaceAll 方法用来替换匹配正则表达式的文本。不同的是，replaceFirst 替换首次匹配，replaceAll 替换所有匹配。

```java
    private static void testReplace(){
        String regex = "dog";
        String input = "The dog says meow. " +
                "All dogs say meow.";
        String REPLACE = "cat";


        Pattern p = Pattern.compile(regex);
        // get a matcher object
        Matcher m = p.matcher(input);
        input = m.replaceAll(REPLACE);
        System.out.println(input);
    }
```



## appendReplacement 和 appendTail 方法

Matcher 类也提供了appendReplacement 和 appendTail 方法用于文本替换

```java
    private static void testAppendReplace(){
        String regex = "a*b";
        String input = "aabfooaabfooabfoobkkk";
        String replace = "-";

        Pattern p = Pattern.compile(regex);

        Matcher m = p.matcher(input);
        StringBuffer sb = new StringBuffer();
        while(m.find()){
            m.appendReplacement(sb,replace);
        }
        m.appendTail(sb);
        System.out.println(sb.toString());
    }
```

