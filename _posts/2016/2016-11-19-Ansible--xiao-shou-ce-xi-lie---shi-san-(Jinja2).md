---
layout: post
title: "Ansible 小手册系列 十三(Jinja2)"
date: "2016-11-19 18:14:53"
categories: Ansible
excerpt: "完整的jinja2说明文档,请移步:http://docs.jinkan.org/docs/jinja2/ 用于playbook中的jinja ..."
auth: lework
---
* content
{:toc}
{% raw %}
> 完整的jinja2说明文档,请移步:http://docs.jinkan.org/docs/jinja2/

## 用于playbook中的jinja 2过滤器
---

**更改数据格式,其结果是字符串**
```
{{  some_variable | to_json  }} 
{{  some_variable | to_yaml  }}
```
**对于人类可读的输出**
```
{{  some_variable | to_nice_json  }} 
{{  some_variable | to_nice_yaml  }}
```
还可以增加参数( new in 2.2)
```
{{ some_variable | to_nice_json(indent=2) }}
{{ some_variable | to_nice_yaml(indent=8) }}
```
**从json字符串读取,其结果为json类型**
```
{{ some_variable | from_json }}
```
**从yaml字符串读取,其结果为yaml类型**
```
{{ some_variable | from_yaml }}
```
常用示例:
```
tasks:
  - shell: cat /some/path/to/file.json
    register: result
- set_fact: myvar="{{ result.stdout | from_json }}"
```

**强制定义变量**

> 如果变量未定义,则来自ansible和ansible.cfg的默认行为为失败,但您可以将其关闭.

```
{{ variable | mandatory }}
```

**使用未定义的变量**

```
{{ result.cmd|default(5) }}
```
> result.cmd 如果没由定义的话,则其默认值为5

**省略参数**

```
- name: touch files with an optional mode
  file: dest={{item.path}} state=touch mode={{item.mode|default(omit)}}
  with_items:
    - path: /tmp/foo
    - path: /tmp/bar
    - path: /tmp/baz
      mode: "0444"
```

> 对于列表中的前两个文件,默认mode将由系统的umask确定,因为mode=parameter 不会发送到文件模块,而最后得文件将接收mode=0444选项.

**列表过滤**

取最小的值
```
{{ list1 | min }}
```

取最大的值
```
{{ [3, 4, 2] | max }}
```

**数据集过滤**

对列表唯一过滤
```
{{ list1 | unique }}
```

对两个列表去重合并
```
{{ list1 | union(list2) }}
```

对两个列表做交集
```
{{ list1 | intersect(list2) }}
```

找到两个列表差异部分(在list1 不在list2 的差异)
```
{{ list1 | difference(list2) }}
```

找到两个列表都互相不在对方列表的部分
```
{{ list1 | symmetric_difference(list2) }}
```

**随机数过滤**

从列表中随机获取元素
```
{{ ['a','b','c','d','e','f']|random }}
```
从0-59 的整数中随机获取一个数
```
{{ 59 |random}}
```
从0-100 中随机获取能被10 整除的数(可以理解为0 10 20 30 40 50 ...100 的随机数)
```
{{ 100 |random(step=10) }}
```

从0-100 中随机获取1 开始步长为10 的数(可以理解为1 11 21 31 41...91 的随机数)
```
{{ 100 |random(1, 10) }}
{{ 100 |random(start=1, step=10) }}
```

**合并散列**

```
{{ {'a':1, 'b':2}|combine({'b':3}) }}
```
结果
```
{'a':1, 'b':3}
```

支持递归合并
```
{{ {'a':{'foo':1, 'bar':2}, 'b':2}|combine({'a':{'bar':3, 'baz':4}}, recursive=True) }}
```
结果
```
{'a':{'foo':1, 'bar':3, 'baz':4}, 'b':2}
```

**提取过滤器**

```
{{ groups['x']|map('extract', hostvars, 'ec2_ip_address')|list }}
```
这需要组“x”中的主机列表,在hostvars中查找它们,然后查找结果中的ec2_ip_address. 最终结果是组“x”中的主机的IP地址列表.

**注释过滤器**

```
{{ "Plain style (default)" | comment }}
```
输出
```
#
# Plain style (default)
#
```

输出各种语言的注释风格
```
{{ "C style" | comment('c') }}
{{ "C block style" | comment('cblock') }}
{{ "Erlang style" | comment('erlang') }}
{{ "XML style" | comment('xml') }}
```

还可以自定义
```
{{ "Custom style" | comment('plain', prefix='#######\n#', postfix='#\n#######\n   ###\n    #') }}
```

正则匹配
```
when: ansible_os_family | match("Red[Hh]at" )
when: url | search("/users/.*/resources/.*")
```

** 其他的常用过滤**

为shell增加双引号
```
- shell: echo {{ string_value | quote }}
```

根据True,False来返回值
```
{{ ('name' == 'John') | ternary('Mr','Ms') }}
```

列表转换字符
```
{{ list | join(" ") }}
```

获取路径的文件名
```
{{ path | basename }}
```

windows平台下获取路径的文件名
```
{{ path | win_basename }}
```

获取路径中的目录
```
{{ path | dirname }}
```

获取软连接的真实路径
```
{{ path | realpath }}
```

获取文件名的名称和扩展名
```
{{ path | splitext }}
```

base64编码
```
{{ encoded | b64decode }}
{{ decoded | b64encode }}
```

从字符串创建UUID(1.9版中的新功能)
```
{{ hostname | to_uuid }}
```

将转换为布尔类型,如"True" 字符串转换为True
```
- debug: msg=test
  when: some_string_value | bool
```

查看变量的python类型
```
{{ myvar | type }}
```

随机列表过滤

给已存在的列表随机排序
```
{{ ['a','b','c']|shuffle }} => ['c','a','b']
{{ ['a','b','c']|shuffle }} => ['b','c','a']
```

**数学**

获取对数
```
{{ myvar | log }}
{{ myvar | log(10) }}
```
获取n次幂
```
{{ myvar | pow(2) }}
{{ myvar | pow(5) }}
```

获取平方根
```
{{ myvar | root }}
{{ myvar | root(5) }}
```

**ip地址过滤**

字符串转ip地址
```
{{ myvar | ipaddr }}
```

字符串转ip协议地址
```
{{ myvar | ipv4 }}
{{ myvar | ipv6 }}
```

从cidr中获取地址信息
```
{{ '192.0.2.1/24' | ipaddr('address') }}
```

**哈希过滤器**

获取字符串得hash值
```
{{ 'test1'|hash('sha1') }}
{{ 'test1'|hash('md5') }}
```

获取字符串校验和
```
{{ 'test2'|checksum }}
```

获取sha512密码哈希
```
{{ 'passwordsaresecret'|password_hash('sha512') }}
{{ 'secretpassword'|password_hash('sha256', 'mysecretsalt') }}
```

## playbook中的测试
---

除了过滤器,所谓的“测试”也是可用的.测试可以用于对照普通表达式测试一个变量. 要测试一个变量或表达式,你要在变量后加上一个 is 以及测试的名称.例如,要得出 一个值是否定义过,你可以用 name is defined ,这会根据 name 是否定义返回 true 或 false.
测试也可以接受参数.如果测试只接受一个参数,你可以省去括号来分组它们.例如, 下面的两个表达式做同样的事情:
```
{% if loop.index is divisibleby 3 %}
{% if loop.index is divisibleby(3) %}

```

**测试字符串**
```
vars:
  url: "http://example.com/users/foo/resources/bar"

tasks:
    - shell: "msg='matched pattern 1'"
      when: url | match("http://example.com/users/.*/resources/.*")

    - debug: "msg='matched pattern 2'"
      when: url | search("/users/.*/resources/.*")

    - debug: "msg='matched pattern 3'"
      when: url | search("/users/")
```
'match'需要在字符串中完全匹配,而'search'只需要匹配字符串的子集.匹配成功返回True,任务则执行.

**版本比较**

检查ansible_distribution_version版本是否大于或等于'12 .04',条件成立返回True.
```
{{ ansible_distribution_version | version_compare('12.04', '>=') }}
```

**进行严格的版本检查**
```
{{ sample_version_var | version_compare('1.0', operator='lt', strict=True) }}
```

**可接受的运算符**
```
<, lt, <=, le, >, gt, >=, ge, ==, =, eq, !=, <>, ne
```
**包含测试**
测试一个列表是否包含另一个列表.
```
vars:
    a: [1,2,3,4,5]
    b: [2,3]
tasks:
    - debug: msg="A includes B"
      when: a|issuperset(b)

    - debug: msg="B is included in A"
      when: b|issubset(a)
```
**路径测试**
```
- debug: msg="path is a directory"
  when: mypath|is_dir
- debug: msg="path is a file"
  when: mypath|is_file
- debug: msg="path is a symlink"
  when: mypath|is_link
- debug: msg="path already exists"
  when: mypath|exists
- debug: msg="path is {{ (mypath|is_abs)|ternary('absolute','relative')}}"
- debug: msg="path is the same file as path2"
  when: mypath|samefile(path2)
- debug: msg="path is a mount"
  when: mypath|ismount
```

**任务测试**

以下playbook是检查任务状态的测试.
```
tasks:
- shell: /usr/bin/foo
    register: result
    ignore_errors: True
- debug: msg="it failed"
    when: result|failed
  - debug: msg="it changed"
    when: result|changed
- debug: msg="it succeeded in Ansible >= 2.1"
    when: result|succeeded
- debug: msg="it succeeded"
    when: result|success
- debug: msg="it was skipped"
    when: result|skipped
```

## 用于 jinja2 模版中的一些语法
---

**变量**

变量可以通过 过滤器 修改.过滤器与变量用管道符号( | )分割,并且也 可以用圆括号传递可选参数.多个过滤器可以链式调用,前一个过滤器的输出会被作为 后一个过滤器的输入.

下面2种方式效果是一样的
```
{{ foo.bar }}
{{ foo['bar'] }}
```
如果变量或属性不存在,会返回一个未定义值.

** 注释**

要把模板中一行的部分注释掉,默认使用 {# ... #} 注释语法.


**转义**

简单的使用单引号进行转义
对于较大的段落,使用raw进行转义
```
{{ "{% raw %}" }}
    <ul>
    {% for item in seq %}
        <li>{{ item }}</li>
    {% endfor %}
    </ul>
{{ "{%"}}{{ "endraw %}" }}
```
包含 > , < , & 或 " 字符的变量,必须要手动转义
```
 {{ user.username|e }} 
```

**控制结构**

控制结构指的是所有的那些可以控制程序流的东西 —— 条件(比如 if/elif/ekse ), for 循环,以及宏和块之类的东西.控制结构在默认语法中以{% .. %}块的形式 出现.

**For**

遍历序列
```
{% for user in users %}
  <li>{{ user.username|e }}</li>
{% endfor %}
```

迭代字典
```
{% for key, value in my_dict.iteritems() %}
    <dt>{{ key|e }}</dt>
    <dd>{{ value|e }}</dd>
{% endfor %}
```

循环 10 次迭代之后会终止处理
```
{% for user in users %}
    {%- if loop.index >= 10 %}{% break %}{% endif %}
{%- endfor %}

{% for user in users if loop.index <= 10 %}
    {{ loop.index }}
{%- endfor %}
```
注:使用`break`, 需要开启轮询控制. 具体是在`ansible.cfg`的`jinja2_extensions`变量加上`jinja2.ext.loopcontrols`.

在一个 for 循环块中你可以访问这些特殊的变量:

|变量|	描述|
|:----|:----|
|loop.index	|当前循环迭代的次数(从 1 开始)|
|loop.index0	|当前循环迭代的次数(从 0 开始)|
|loop.revindex	|到循环结束需要迭代的次数(从 1 开始)|
|loop.revindex0|	到循环结束需要迭代的次数(从 0 开始)|
|loop.first	|如果是第一次迭代,为 True.|
|loop.last	|如果是最后一次迭代,为 True.|
|loop.length|	序列中的项目数.|
|loop.cycle	|在一串序列间期取值的辅助函数.见下面的解释.|

**if 语句**

Jinja 中的 if 语句可比 Python 中的 if 语句.
```
{% if kenny.sick %}
    Kenny is sick.
{% elif kenny.dead %}
    You killed Kenny!  You bastard!!!
{% else %}
    Kenny looks okay --- so far
{% endif %}
```

** 过滤器**

过滤器段允许你在一块模板数据上应用常规 Jinja2 过滤器.只需要把代码用 filter 节包裹起来:
```
{% filter upper %}
    This text becomes uppercase
{% endfilter %}
```
**赋值**

在代码块中,你也可以为变量赋值.在顶层的(块,宏,循环之外)赋值是可导出的,即 可以从别的模板中导入.

赋值使用 set 标签,并且可以为多个变量赋值:
```
{% set navigation = [('index.html', 'Index'), ('about.html', 'About')] %}
{% set key, value = call_something() %}
```

** 表达式**

`{% ... %}`   用于执行诸如 for 循环 或赋值的语句
`{{ ... }}`  把表达式的结果打印到模板上

**if 表达式**
一般的语法是 

`<do something> if <something is true> else <do something else>}`

例如:
```
{{ '[%s]' % page.title if page.title is defined else 'undefined' }}
```

**字面量**

表达式最简单的形式就是字面量.字面量表示诸如字符串和数值的 Python 对象.下面 的字面量是可用的:

|字面量 |说明|
|:---|:---|
|"Hello World"	|双引号或单引号中间的一切都是字符串.无论何时你需要在模板中使用一个字 符串(比如函数调用,过滤器或只是包含或继承一个模板的参数),它们都是 有用的.|
|42/42.23	|直接写下数值就可以创建整数和浮点数.如果有小数点,则为浮点数,否则为 整数.记住在 Python 里, 42 和 42.0 是不一样的.|
|['list','of','objects']	|一对中括号括起来的东西是一个列表.列表用于存储和迭代序列化的数据.|
|('tuple','of','values')|	元组与列表类似,只是你不能修改元组.如果元组中只有一个项,你需要以逗号 结尾它.元组通常用于表示两个或更多元素的项.更多细节见上面的例子.|
|{dict':'of','key':'and','value':'pairs'}	|Python 中的字典是一种关联键和值的结构.键必须是唯一的,并且键必须只有一个 值.字典在模板中很少使用,罕用于诸如 xmlattr() 过滤器之类.|
|true/false|	true 永远是 true ,而 false 始终是 false.|

> 特殊常量 true , false 和 none 实际上是小写的.因为这在过去会导致 混淆,过去 True扩展为一个被认为是 false 的未定义的变量.所有的这三个 常量也可以被写成首字母大写( True , False 和 None ).尽管如此, 为了一致性(所有的 Jinja 标识符是小写的),你应该使用小写的版本.

**算术**

Jinja 允许你用计算值.这在模板中很少用到,但是为了完整性允许其存在.支持下面的 运算符:

|运算符|说明|
|:---|:---|
|+	|把两个对象加到一起.通常对象是素质,但是如果两者是字符串或列表,你可以用这 种方式来衔接它们.无论如何这不是首选的连接字符串的方式！连接字符串见 ~ 运算符. {{ 1 + 1 }} 等于 2 .|
|-	|用第一个数减去第二个数. {{ 3 - 2 }} 等于 1 .|
|/	|对两个数做除法.返回值会是一个浮点数. {{ 1 / 2 }} 等于 {{ 0.5 }} .|
|//	|对两个数做除法,返回整数商. {{ 20 // 7 }} 等于 2 .|
|%	|计算整数除法的余数. {{ 11 % 7 }} 等于 4 .|
|*	|用右边的数乘左边的操作数. {{ 2 * 2 }} 会返回 4 .也可以用于重 复一个字符串多次. {{ '=' * 80 }} 会打印 80 个等号的横条.|
|**	|取左操作数的右操作数次幂. {{ 2**3 }} 会返回 8 .|

**比较**

|比较符|说明|
|:---|:---|
|==	|比较两个对象是否相等.|
|!=	|比较两个对象是否不等.|
|> 	|如果左边大于右边,返回 true .|
|>=	|如果左边大于等于右边,返回 true .|
|<	|如果左边小于右边,返回 true .|
|<=	|如果左边小于等于右边,返回 true .|

**逻辑**

对于 if 语句,在 for 过滤或 if 表达式中,它可以用于联合多个表达式:

|逻辑符|说明|
|:----|:----|
|and	|如果左操作数和右操作数同为真,返回 true.|
|or	|如果左操作数和右操作数有一个为真,返回 true.|
|not	|对一个表达式取反(见下).|
|(expr)	|表达式组.|

> is 和 in 运算符同样支持使用中缀记法: foo is not bar 和 foo not in bar 而不是 not foois bar 和 not foo in bar .所有的 其它表达式需要前缀记法 not (foo and bar) .

**其它运算符**

下面的运算符非常有用,但不适用于其它的两个分类:

|运算符|说明|
|:----|:----|
|in| 运行序列/映射包含检查.如果左操作数包含于右操作数,返回 true.比如 {{ 1 in [1,2,3] }} 会返回 true.|
|is	|运行一个 测试 .|
||	|应用一个 过滤器 .|
|~	|把所有的操作数转换为字符串,并且连接它们. {{ "Hello " ~ name ~ "!" }} 会返回(假设 name 值为 ''John' ) Hello John! .|
|()	|调用一个可调用量:{{ post.render() }} .在圆括号中,你可以像在 python 中一样使用位置参数和关键字参数: {{ post.render(user, full=true) }} .|
|. / []	|获取一个对象的属性.|
---
更多文章请看 [Ansible 专题文章总览](http://www.jianshu.com/p/c56a88b103f8)
{% endraw %}