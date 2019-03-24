# Ninja

> ninja 是一个快速增量编译的工具

本文算是 [官方文档](https://ninja-build.org/manual.html) 的一个中文翻译简单版，还是去看看官网的比较全面。

## Overview

#### 需要注意的点

* 编译产物强依赖于所使用的命令，也就是改变命令，比如 flag 值，都会重新编译
* 可以使用简写，比如 CC foo.o 来代替一大串命令行
* 并行运行，跟机器的核数相关， Underspecified build dependencies will result in incorrect builds.
* 命令的编译会被缓存



## 开始使用 Ninja

Ninja 的特点在于缩短修改后重新编译(edit-compile cycle)的时间。

gn 则是用于生成 Chrome 或其他工程的编译文件，它可以生成 Chrome 所能支持的平台的 Ninja 编译文件。(generate ninja?)

#### Ninja 的运行

Ninja 运行时，会在当前目录下找 `build.ninja` 并编译过期失效的 target。你也可以在命令行中指定要编译的目标。`ninja <target>`

`ninja -h` 输出帮助信息

Ninja 的很多参数跟 Make 很像，比如`ninja -C build -j 20` 表示移动到 build/ 并发运行 20 条命令。（默认 Ninja 会并发运行，所以其实不用 -j ）

#### 环境变量

Ninja 可以通过一个环境变量控制他的行为: `NINJA_STATUS`

默认的值是 `[%f/%t]` 或者 `[%u/%r/%f]`。其中：

| 符号 | 含义                                                         |
| ---- | ------------------------------------------------------------ |
| %f   | the number of finished edges                                 |
| %t   | the total number of edges that must be run to complete the build |
| %u   | the number of remaining edges to start                       |
| %r   | the number of currently running edges                        |

更多内容请看 [官方文档](https://ninja-build.org/manual.html) 「Environment variables」一节

#### 额外的工具

使用方式：`ninja -t <tool>`

| 工具   | 用途                                                         |
| ------ | ------------------------------------------------------------ |
| query  | dump 某 target 的输入和输出                                  |
| browse | 在浏览器中查看依赖图谱 `ninja -t browse --port=8000 --no-browser mytarget` |

更多工具请看 [官方文档](https://ninja-build.org/manual.html) 「Extra tools」一节



## 开始写 Ninja 文件

Ninja 计算文件间的依赖图，根据文件的修改时间，决定运行哪些命令可以使得某些过期的 target 更新至最新。这跟 Make 很相似。

build 文件 (默认是 build.ninja) 存放 rules 列表(即短名与相关命令的映射列表)

#### 举个例子来说一下语法

```makefile
cflags = -Wall

rule cc
  command = gcc $cflags -c $in -o $out

build foo.o: cc foo.c
```

变量，格式为 `variable = value` 使用时可以是这样 `$variable`

rule 定义一个短名对应一个命令，格式如上，`rule <short-name>` 接着是缩进后的一行 `variable = value` 

在一个 rule 里面，变量 command 代表要运行的命令，`$in` 表示所有的输入文件 `$out` 表示所有的输出文件。当然 rule 的变量还有很多：[reference](https://ninja-build.org/manual.html#ref_rule)

build 语句描述了输入文件与输出文件的关系。以 `build`开头，标准格式是 `build outputs: rulename inputs ` 

比如 output 文件不在，或者是 input 文件修改了，Ninja 就会重新运行 `rule` 重新生成 output 文件

build 语句下面也可以有 `key = value` 对，可以用在当前规则下覆盖原有的变量，如：

```shell
< 承接上部分代码 >
build sepcial.o: cc special.c
  cflags = -Wall -Werror

即 gcc -Wall -Werror -c special.c -o special.o  替换了 cflags 的值
```

##### 使用代码生成 Ninja 文件

[misc/ninja_syntax.py](https://github.com/ninja-build/ninja/blob/master/misc/ninja_syntax.py) 可以帮你生成 Ninja 文件，你可以用 Python 写如此的代码 `ninja.rule(name='foo', command='bar', depfile='$out.d')` 接着就会生成合适的语句了。`Feel free to just inline it into your project’s build system if it’s useful.`  `Long lives community`

## 更多细节

#### phony 规则

(暂时无法理解 ~(=-=)~

#### default target 语句

默认如果你没有在命令行中特别指定 target 的话，默认会 ` build every output that is not named as an input elsewhere` 也就是出度为 0 的节点。这样的话，相当于从产物开始从尾往前遍历(编译)所有依赖的模块了。

你可以使用 default 语句 `default targets`改变上述的情况，比如：

```shell
default foo bar
default baz
```

注意的是，需要把 default 语句放在生成 target 的 build 语句之后，比如:

```shell
build foo: cc foo.c
build bar: cc bar.c
default foo bar
```

如果 ninja 命令行中没有指定 target，会生成 foo bar，以及连带一些他们所依赖的文件

#### Ninja log

Ninja 会保存每一个编译过的产物的编译时间，在编译根目录中的 `.ninja_log` 中。(可以更改路径，修改 `builddir` 变量 `in the outermost scope`)

#### 版本兼容

Ninja 版本号遵循标准的 `major.minor.path` 格式。`major` 一般是向后不兼容的语法或行为变动。而 `minor` 则是一些小的改动。可以在 `build.ninja` 中声明最低 Ninja 版本如:

```shell
ninja_required_version = 1.1
```

一般是放在 build.ninja 文件的前边的位置。

有趣的是，如果你的 Ninja 版本跟 build.ninja 声明的 `major` 版本不一致的时候，Ninja 会警告(笔者按：毕竟可能存在 breaking change

#### C/C++ 头文件的依赖

(不是很理解，暂时空白)

#### Pools

pools 允许你声明某些 `rules` 或 `edges` 所能同时运行的作业数，并严格执行，其实际运行数将不会大于默认的并行量。(笔者按：也就是只能更少，不能更多)   其中 `depth` 变量表示同时运行的作业数

一般用于限制某些昂贵的 rule 语句或者是并行运行很差的 build 语句。(笔者按：你太慢了，还是把时间片多分给一些作业短的人吧) 如：

```shell
pool link_pool
  depth = 4

pool heavy_object_pool
  depth = 1

rule cc
  ...

rule link
  pool = link_pool
  ...

# 将会用到 link_pool. 同时最多 4 个 link 作业执行
build foo.exe: link input.obj

# 定义为空，则会重置成默认配置
build other.exe: link input.obj
  pool =

# 这个 build 语句使用了 heavy pool，与下面的竞争，同时只能运行一个 build
build heavy_object1.obj: cc heavy_obj1.cc
  pool = heavy_object_pool
# 哎呀~
build heavy_object2.obj: cc heavy_obj2.cc
  pool = heavy_object_pool
```

#### console pool

有个预定义的 pool:

```shell
pool console:
  depth = 1
  ...
```

这个 pool 的 task 能够直接访问到 Ninja 的 stdin, stdout, stderr 流，这个在一些交互式的 task (笔者按：比如 `Are you continue? (Y for yes, N for no)`) 或者是输出一些长时间运行的作业的状态 (笔者按：比如 `Download 20%`)

当一个有着 console 作为 pool 的任务在执行，Ninja 的一些输出将会被缓存住，直到这个任务结束。



## Ninja file 引用

`file` 是一系列声明的集合。包括：

* `rule rulename`
* `build output: rulename input`
* 变量声明 `name = zheng`
* `default target1`
* 依赖更多的 file 如 `subninja path` 或者 `include path`
* `pool poolname`

#### 语法

* `#` 是注释的开头

* 行分隔符，名字有空格的需要 `\ `

* 转移符只有 `$` 

  * `$` 后接换行，表示取消换行，相当于 shell 中 末尾的 `\`
  * 后接 text：表示一个变量 variable
  * `${varname}`: 同上
  * `$ `后接空格：转义空格符，一般用于路径中包含空格的情况
  * `$:` 转义冒号，`build abc.out: cc such$:a$:strange$:name`
  * `$$` 转义 `$` 符

* build 或者 default 语句中的变量是空格分隔的，也就是遇到包含空格的文件名需要 `$` 转义

* `name = value`语句呢，末尾带 `$`会转义掉换行符，以及第二行前面的空白会被忽略掉

  ``` shell
  two_words_with_one_space = foo $
    bar
  one_word_with_no_space = foo$
    bar
  ```

* 空格缩进也能够表示该行所处的代码块 `scope`，就像 `python` 的缩进一样

#### 最外层参数

| builddir               | Ninja 输出文件的目录      |
| ---------------------- | ------------------------- |
| ninja_required_version | 编译所需 Ninja 的最小版本 |

#### Rule 参数

`rule` 可以通过设置 `key = value` 的方式，影响 rule 的运行。比如:

```shell
rule cc:
  command = gcc -c $in -o $out
```

可设置的参数包括：

| 参数        | 含义                                                |
| ----------- | --------------------------------------------------- |
| command     | 配置 rule 所对应的命令，只能有一个                  |
| description | 描述项，-v 选项会输出之                             |
| in          | 后接的值可以是空格分隔的文件列表，替换上面的 `$in`  |
| out         | 后接的值可以是空格分隔的文件列表，替换上面的 `$out` |

\* 注意：

* command 里面的语句在不同平台，表现是有点不一样的，在类 Unix 系统中，是传递给 `sh -c `的，所以能够支持 `&&` 等符号；在 Windows 中，则是传递给 `CreateProcess` ，所以如果你想要支持 `&&` 的话，需要在命令前面加上 `cmd /c`.   
* Ninja 在遇到命令行长度过长的时候，一般的报错是 `invalid parameter`



#### Build 产物 (Build outputs)

build 产物有俩种

* 显式产物，build 语句中明确指出的。rule 语句中的 $out
* 隐式产物，build 语句中 `: ` 前边的` | out1 out2 +` 跟上面不同的是，不会指代到 $out 中。比如 `build xxx.exe | side_product.o: CC xxx.o side.c`

** 注：对 `| out1 out2 +` 中的 `+` 存疑，看了一下 chromium 的 build.ninja 并没有 + 的情况.

#### Build 依赖

Build 依赖有三种：

* 显式依赖，build 语句中出现，rule 语句中的 $in. 修改这些依赖会导致相关语句重新编译

* 隐式依赖，在 rule 语句中出现的 `depfile` 参数，或者是 build 语句末尾的 `| dep1 dep2` 与上面的区别在于这些依赖不会出现在 rule 语句中的 $in
* Order-only 依赖，build 语句末尾添加的：`|| dep1 dep2`  纯粹是这种依赖的改变，不会重新 build，但是如果此时需要 build 而 Order-only 依赖过期了，build 语句会等待这些依赖更新之后，才执行。一般 Order-only 依赖是编译期生成的文件，比如头文件之类的。

#### 变量展开 Variable expansion

赋值语句 `name = value` 执行的时候，会把 value 展开，替换里面的 `$other-var`。你不用担心他会被再次展开，此时 `$name` 将会是一个 static string。而 rule 语句的话，变量只会在使用的时候展开，所以请放心，还是不同的。例子如下：

```shell
rule demo
  command = echo "this is a demo of $foo"

build out: demo
  foo = bar
```

#### 赋值以及作用域 Evaluation and scoping

Top-level 变量的作用域是整个 file

Rule 的作用域也是整个 file（被 subninja 依赖的除外）

`subninja` 可以 include .ninja 文件，并使用里面的变量，rule 等，但是不会影响到被 include 文件的值。

如果想像 C `#include` 那种效果的话，请使用 `include` 关键字，替换掉 `subninja`。

在 build 块中的变量，作用域只在 build 块中。而 build 块中（或 rule 块）变量扩展遵循的顺序如下：

* 特殊的内置变量，如 `$in` `$out`
* build 块中 build 级别的变量
* rule 块中 rule 级别的变量，如 `command`
* file 级别的，同在 file 内的变量
* 被 subninja 引入的 file，里面的 变量