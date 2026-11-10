---
title: fish for Bash users 渣翻中译
published: 2026-11-10
tags: []
category: 'translations'
draft: false
---

(似乎网上没人翻译这篇文章，但对于从类bash shell迁移到fish的人来说，这篇教程其实非常足够入门fish了)

如果你是从bash(或是其他类似的比如说zsh或ksh)迁移过来的，并且想知道fish哪里不一样，那这篇文章就是一个快速概览. fish有意设计为posix不兼容，因此有些事情的做法需要变通一下.

大部分其实都差不多 - 这些shell都会展开命令行来执行命令，都有管道、重定向、变量、通配符，通过各种方式使用命令输出. 这篇文档就是给你快速展示区别所在的.

## 命令替换

fish的命令替换可以写成```$(command)```或是```(command)```，但不能是``` `command` ```.

还有，fish只通过换行来分割命令，而不是使用$IFS. 如果你想通过别的方法分割，使用[string split](https://fishshell.com/docs/current/cmds/string-split.html), [string split0](https://fishshell.com/docs/current/cmds/string-split.html), 或者[string collect](https://fishshell.com/docs/current/cmds/string-collect.html). 如果它们被用在命令替换的最后一条命令，这些分割同样会生效. 所以:

```fish
for i in (find . -print0 | string split0)
```

会正确处理所有需要的文件名.

## 变量

fish设置和删除变量都使用[set](https://fishshell.com/docs/current/cmds/set.html)而不是用```VAR=VAL```以及一堆分开的builtin，比如说```declare```,```unset```和```export```. ```set```通过参数来决定变量作用域以及是否export.

```fish
# Define $PAGER *g*lobal and e*x*ported,
# so this is like ``export PAGER=less``
set -gx PAGER less

# Define $alocalvariable only locally,
# like ``local alocalvariable=foo``
set -l alocalvariable foo
```

或是删除变量

```fish
set -e PAGER
```

```VAR=VAL```在环境变量覆盖中仍然可用

```fish
PAGER=cat git log
```

fish不进行分词. 一个变量的值被设置后不会做任何额外处理，所以像bash那样用双引号括起来变量的做法已经没必要了.(zsh默认也不分词，SH_WORD_SPLIT控制这个行为)

举个例子，这是bash的做法

```bash
> foo="bar baz"
> printf '"%s"\n' $foo
# will print two lines, because we didn't double-quote
# this is word splitting
"bar"
"baz"
```

这是fish的做法

```fish
set foo "bar baz"
printf '"%s"\n' $foo
```

```
# foo被设置为单一元素，
# 所以它也会被传递为单一元素，输出就只是一行
"bar baz"
```

所有变量都是“数组”(我们使用术语“列表”)，展开一个变量相当于展开它的所有元素，每个元素都相当于一个参数(就像bash的```"${var[@]}"```):

```fish
set var "foo bar" banana
printf %s\n $var
```

```
foo bar
banana
```

列表的特定元素可以被选取(索引)

```fish
echo $list[5..7]
```

传给set参数很符合直觉，所以你可以设置一个变量值为命令的输出:

```fish
# Set lines to all the lines in file, one element per line
set lines (cat file)
```

或是字面量和输出的混合:

```fish
> set numbers 1 2 3 (seq 5 8) 9
> printf '%s\n' $numbers
```

```
1
2
3
5
6
7
8
9
```

对于```set```，```=```是不必要且没有帮助的 - ```set foo = bar```会设置foo变量的值为=和bar . ```set foo=bar```会出错.

[Shell variables](https://fishshell.com/docs/current/language.html#variables)中有更多信息.

## 变量默认值(```${my_variable:-"default value"}```)

fish提供未初始化变量的默认值没有```${my_variable:-fallback}```这样的写法. 你可以通过检查变量是否已经被初始化来设置默认值.

```fish
# Ensure XDG_CONFIG_HOME is set or use a default value
set -q XDG_CONFIG_HOME || set XDG_CONFIG_HOME $HOME/.config
# now use XDG_CONFIG_HOME as normal
```

## 通配符

fish只支持```*```和```**```通配符(还有以及废弃的```?```). 如果通配符没有匹配到，就会使命令失败(类似bash的```failglob```)，除非命令是```for```,```set```或者```count```，或是被用在了环境变量覆盖上(```VAR=* command```)，这种情况下会展开为空(类似bash的```nullglob```).

展开变量时不会进行通配符匹配，所以:

```fish
set foo "*"
echo $foo
```

不会匹配任何文件.

通配行为没有设置来调整，所以总是这样工作.

```**```同样会匹配子文件夹. 在其他shell中这一项通常需要使用某项设置来开启，比如说bash中的```setopt globstar```.

和bash不同，fish会跟随符号链接，并且会对结果自然排序，当然数字也会和数字比较了. 这意味着它即使面对```1```而不是```01```也会正确排序.

（PS：这段的意思是说数字会按数字大小排序，而不是直接按ascii码大小排序）

[Wildcards](https://fishshell.com/docs/current/language.html#expand-wildcard)有更多信息.

## 引号

fish有两种引号形式，```""```和```''```. 变量都被展开为双引号形式的，任何展开都不会产生单引号形式.

没有```$''```这种写法，可变形的序列在没有引号括起来的时候会自动变形:

```fish
echo a\nb
```

```
a
b
```

[Quotes](https://fishshell.com/docs/current/language.html#quotes)中有更多信息.

## 字符串修改

fish没有```${foo%bar}```,```${foo#bar}```和```${foo/bar/baz}```这样的语法. 字符串修改都使用[string](https://fishshell.com/docs/current/cmds/string.html) builtin来完成.

例如，要把bar替换为baz:

```fish
string replace bar baz "bar luhrmann"
```

```
baz luhrmann
```

也可以分割字符串:

```fish
string split "," "foo,bar"
```

```
foo
bar
```

也可以匹配正则表达式，算是```grep```的一种替代:

```fish
echo bababa | string match -r 'aba$'
```

```
aba
```

也可以用任意字符做padding:

```fish
string pad -c x -w 20 "foo"
```

```
xxxxxxxxxxxxxxxxxfoo
```

也可以让字符串变为全小/大写:

```fish
string lower Foo
string upper Foo
```

```
foo
FOO
```

还可以重复，trim，转义或者打印字符串的长度或宽度.

## 特殊变量

一些bash变量和最接近的fish等价变量:

- ```$*```，```$@```，```$1```以及其他: ```$argv```
- ```$?```: ```$status```
- ```$$```: ```$fish_pid```
- ```$#```: 没有变量，使用```count $argv```
- ```$!```: ```$last_pid```
- ```$0```: ```status filename```
- ```$-```: 基本上是```status is-interactive```和```status is-login```

## 进程替换

fish不使用```<(command)```，而是```(command | psub)```. 没有```>(command)```的等价语法.

需要注意这两者都是bash常用的，大多数事情其实可以更简洁完成，比如不要用:

```fish
source (command | psub)
```

而用:

```fish
command | source
```

因为fish的[source](https://fishshell.com/docs/current/cmds/source.html)能读取stdin.

## Heredocs

fish没有```<<EOF```这样的heredocs. 不要用:

```bash
cat <<EOF
some string
some more string
EOF
```

而用:

```fish
printf %s\n "some string" "some more string"
```

或者:

```fish
echo "some string
some more string"

# or if you want the quotes on separate lines:

echo "\
some string
some more string\
"
```

换行不会终止引用.

heredocs所做的是:

1. 使用特定规则读取/解释字符串，直到遇见终止符.(例如，"EOF"只是一个约定，终止符可以是任意字符串，比如说"THISISTHEEND"也是可以的. 使用```<<-```可以trim掉tab(只有tab)，所以你可以做缩进，但必须用tab. 变量、命令替换都在heredoc内默认完成，但如果终止符用引号括起来了就不完成).
2. 把结果写到一个临时文件里
3. 启动heredoc附加到的命令，并把这个文件作为stdin.

所以基本上和从管道读取是一样的，所以:

```fish
echo "foo" | cat
```

差不多就是

```bash
cat <<EOF
foo
EOF
```

就和heredocs一样，命令必须准备好从stdin读取输入了. 有时候这要求使用特定的参数，通常是把```-```作为文件名.

例如:

```fish
echo "xterm
rxvt-unicode" | pacman --remove -

# is the same as (the `-` makes pacman read arguments from stdin)
pacman --remove xterm rxvt-unicode
```

在其他shell中也可以写作:

```
# This "-" is still necessary - the heredoc is *also* passed over stdin!
pacman --remove - << EOF
xterm
rxvt-unicode
EOF
```

所以heredocs就是一个小语法糖，介绍了很多特殊的规则，这就是fish不使用它们的原因了. 管道是一个核心概念，并且更简单，也更易组合.

你也可以把好几样东西都通过管道传给命令，比如说:

```fish
{ cat packages1; echo fish; } | pacman --sync -
```

## Test(```test```,```[```,```[[```)

fish有posix兼容的```test```或```[``` builtin. 没有```[[```，并且```test```也不接受```==```作为```=```的同义替换. 顺便一提，这的test能比较浮点数.

```set -q```可以用来判断一个变量是否存在，或者是否存在某个特定位置的元素(```set -q foo[2]```).

## 算术展开

fish没有```$((i+1))```这样的算术展开，计算工作由[math](https://fishshell.com/docs/current/cmds/math.html)处理:

```fish
math $i + 1
```

与bash的算术不同，这个能处理浮点数:

```fish
math 5 / 2
```

```
2.5
```

也有一些函数，比如三角函数:

```fish
math cox 2 x pi
```

```
1
```

你可以给```math```分开传参数，或是直接引号括起来. 因为fish使用```()```作为命令替换，如果你的表达式里面有括号，就需要用引号括起来.

```fish
math '(5 + 2) * 4'
```

```*```和```x```都可以表示乘法，但```*```需要用引号括起来，因为这是个通配符.

## Prompts

fish不使用```$PS1```，```$PS2```及类似的变量. prompt由[fish_prompt](https://fishshell.com/docs/current/cmds/fish_prompt.html)函数输出，如果[vi mode](https://fishshell.com/docs/current/interactive.html#vi-mode)启用了就还有[fish_mode_prompt](https://fishshell.com/docs/current/cmds/fish_mode_prompt.html)函数. [fish_right_prompt](https://fishshell.com/docs/current/cmds/fish_right_prompt.html)函数输出用于右侧prompt.

例如，这是一个比较简单的bash prompt:

```bash
# <$HOSTNAME> <$PWD in blue> <Prompt Sign in Yellow> <Rest in default light white>
PS1='\h\[\e[1;34m\]\w\[\e[m\] \[\e[1;32m\]\$\[\e[m\] '
```

还有一个fish等价版:

```fish
function fish_prompt
    set -l prompt_symbol '$'
    fish_is_root_user; and set prompt_symbol '#'

    echo -s (prompt_hostname) \
    (set_color blue) (prompt_pwd) \
    (set_color yellow) $prompt_symbol (set_color --reset)
end
```

这展示了几处区别:

- fish提供[set_color](https://fishshell.com/docs/current/cmds/set_color.html)来给文本上色. 它可以使用16种命名颜色，也可以用RGB序列(所以你也可以用```set_color 5555FF```)
- prompt是一个函数，而不是一堆转义字符，像是```\h```代表主机名之类的. 要达到```\h```的效果，fish提供了一些函数来帮助，比如[prompt_hostname](https://fishshell.com/docs/current/cmds/prompt_hostname.html)，打印一个短的主机名.
- fish也提供其他的帮助函数来给prompt加东西，例如[fish_vcs_prompt](https://fishshell.com/docs/current/cmds/fish_vcs_prompt.html)可以为常用vcs(git, mercurial, svn)添加显示，[prompt_pwd](https://fishshell.com/docs/current/cmds/prompt_pwd.html)显示一个短的```$PWD```(用户的home变为```~```，其他的路径部分都被缩短)

默认prompt已经有很多功能了，它的代码可以用```type fish_prompt```查看.

fish没有```$PS2```来应对接续的行，而只是把行缩进，这样就能看出来命令行还不完整了.

## 块和循环

fish的块结构和其他shell有点不一样，都由一个词起始，由```end```结束，并且没有另一个起始词紧跟.

```
for i in 1 2 3; do
   echo $i
done

# becomes

for i in 1 2 3
   echo $i
end

while true; do
   echo Weeee
done

# becomes

while true
   echo Weeeeeee
end

{
   echo Hello
}

# becomes

begin
   echo Hello
end

if true; then
   echo Yes I am true
else
   echo "How is true not true?"
fi

# becomes

if true
   echo Yes I am true
else
   echo "How is true not true?"
end

foo() {
   echo foo
}

# becomes

function foo
    echo foo
end

# (bash allows the word "function",
#  but this is an extension)
```

fish没有```until```. 用```while not```或者```while !```.

## 子shell

bash有一个特性叫子shell(subshells)，会启动另一个shell来处理一些事情. 那个shell启动后便独立，比如说，对变量所做的任何更改不会在主shell中体现.

像这样:

```bash
# A list of commands in `()` parentheses
(foo; bar) | baz

# Both sides of a pipe
foo | while read -r bar; do
    # This will not be visible outside of the loop.
    VAR=VAL
    # This background process will not be, either
    baz &
done
```

fish还不支持子shell. 你需要找一个其他的解决方案. 隔离通常可以通过划分变量作用域来实现，不过如果你真的需要在新shell中运行你的代码，可以用```fish -c 'your code here'```来显式完成.

```()```子shell通常会和```{}```分组混淆，分组不使用子shell. 如果你只需要分组，可以在fish中使用```begin; end```:
```
(foo; bar) | baz
# when it should really have been:
{ foo; bar; } | baz
# becomes
begin; foo; bar; end | baz
```

管道会在同一个进程中运行，所以```while read```循环可以在外部环境设置变量:

```fish
foo | while read bar
    set -g VAR VAL
    baz &
end

echo $VAR # will print VAL
jobs # will show "baz"
```

子shell也经常和[命令替换](#命令替换)混淆，bash写作``` `command` ```或```$(command)```，fish写作```$(command)```或```(command)```. bash也使用子shell来实现这些.

## Builtins以及其他命令

目前为止fish更专注于builtins和外部命令，而不是语法. 以下是一些有用的builtins还有bash的等价命令

- [string](https://fishshell.com/docs/current/cmds/string.html) - 替代了绝大部分的字符串转换，也可以用来替代```grep```和```sed```之类的工具.
- [math](https://fishshell.com/docs/current/cmds/math.html) - 替代了```$((i + 1))```算术，还可以做浮点运算，以及一些简单函数(三角函数一类).
- [argparse](https://fishshell.com/docs/current/cmds/argparse.html) - 处理脚本参数解析，bash大概要用```getopt```(zsh提供了```zparseopts```).
- [count](https://fishshell.com/docs/current/cmds/count.html)用于计数，替代了```$#```，也是```wc```的替代品.
- [status](https://fishshell.com/docs/current/cmds/status.html)提供shell的状态信息，比如是否处于交互状态或者当前行号是多少. 替代了```$-```，```$BASH_LINENO```和其他变量.
- ```seq(1)```用来替换```{1..10}```这种的范围展开. 如果你的系统没有安装```seq```，fish有一个替代用的函数.

## 其他...

bash有```set -x```或```set -o xtrace```来打印所有执行的命令. 在fish中，[$fish_trace](https://fishshell.com/docs/current/language.html#envvar-fish_trace)可以启用这个功能.

或者说，如果你想的是测试脚本中每行代码执行的时间，你可以用```fish --profile``` - 查看[page for the fish command](https://fishshell.com/docs/current/cmds/fish.html)获取更多信息.