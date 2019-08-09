---
layout: post
title:  "linux常用命令"
categories: linux
tags:  linux 常用命令
author: 网络
---

* content
{:toc}









## 三剑客

### grep

grep主要用于查找字符串，常用的参数如下：

* --color=auto 或者 --color：表示对匹配到的文本着色显示
* -i：在搜索的时候忽略大小写
* -n：显示结果所在行号
* -c：统计匹配到的行数，注意，是匹配到的总行数，不是匹配到的次数
* -o：只显示符合条件的字符串，但是不整行显示，每个符合条件的字符串单独显示一行
* -v：输出不带关键字的行（反向查询，反向匹配）
* -w：匹配整个单词，如果是字符串中包含这个单词，则不作匹配
* -Ax：在输出的时候包含结果所在行之后的指定行数，这里指之后的x行，A：after
* -Bx：在输出的时候包含结果所在行之前的指定行数，这里指之前的x行，B：before
* -Cx：在输出的时候包含结果所在行之前和之后的指定行数，这里指之前和之后的x行，C：context
* -q：静默模式，不输出任何信息，当我们只关心有没有匹配到，却不关心匹配到什么内容时，我们可以使用此命令，然后，使用"echo $?"查看是否匹配到，0表示匹配到，1表示没有匹配到。
* -P：表示使用兼容perl的正则引擎。
* -E：使用扩展正则表达式，而不是基本正则表达式，在使用"-E"选项时，相当于使用egrep。

```bash
# 准备一个文本文件供查找
cat >> testgrep.txt << EOF
zhangsan 30 2000-01-01 basketball
ZHANGSAN 30 2000-01-01 basketball
lisi 40 2001-09-10 football
EOF
# 查找包含zhangsan的行，-i参数代表忽略大小写，下面2种写法一样，-n显示行号
cat testgrep.txt | grep -i -n zhangsan
grep -i -n zhangsan testgrep.txt

# 查找包含zhangsan或者lisi的行
grep -e zhangsan -e lisi testgrep.txt
# 使用正则表达式匹配zhangsan开头，ball结尾的行
grep -E "^zhangsan.*ball$" testgrep.txt
```

### sed

sed主要用于文本修改，参数：

* -i 修改文件内容
* -e 对同一个文件作多次修改

功能选项：

* a：追加内容到指定行后
* i：插入内容到指定行前
* d：删除指定行
* c：用新行替换旧行(不常用)
* s：对每一行第一次匹配到的内容进行替换，配合标志g可以将一行中所有匹配到的内容进行替换
* p：输出指定内容，默认会输出2次匹配到的内容

```bash
# 1. 替换文本
# 不带g只替换第一个找到的内容，带g则全部替换
sed -i 's/原字符串/新字符串/' /home/1.txt
sed -i 's/原字符串/新字符串/g' /home/1.txt
# 示例修改配置来关闭selinux，不需要打开文件，直接一行代码直接修改
sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config && setenforce 0
# 用分号隔开，替换多个文本
sed -i 's/line1/line11/;s/line2/line22/' test.txt

# 2. 添加行
# 在第2行之前新增几行，行与行之间用\n作为换行符
sed -i '2i test line1\ntest line2' test.txt
# 在第4行之后新增几行，行与行之间用\n作为换行符
sed -i '4a test line1\ntest line2' test.txt
# 在文件末尾新增一行
sed -i '$a this is new line' test.txt

# 3. 删除
# 将包含字符串的行全部删除
sed -i '/somestring/d' test.txt
# 删除第2行
sed -i '2d' test.txt
# 删除第1-3行
sed -i '1,3d' test.txt
```

### awk

awk主要用于文本格式化，awk常用的内置变量以及其作用如下：

* FS：输入字段分隔符， 默认为空白字符
* OFS：输出字段分隔符， 默认为空白字符
* RS：输入记录分隔符(输入换行符)， 指定输入时的换行符
* ORS：输出记录分隔符（输出换行符），输出时用指定符号代替换行符
* NF：number of Field，当前行的字段的个数(即当前行被分割成了几列)，字段数量
* NR：行号，当前处理的文本行的行号。
* FNR：各文件分别计数的行号
* FILENAME：当前文件名
* ARGC：命令行参数的个数
* ARGV：数组，保存的是命令行所给定的各参数

```bash
# 1. 基本打印功能
# 准备测试文件
cat >> test1.txt << EOF
zhangsan 30 2000-01-01 basketball
lisi 40 2001-09-10 football
EOF

# 打印出每一列数据及列长度，$0代表整行，$NF代表最后一列数据，NF代表多少列，双引号中添加常量列，NR代表行号
awk '{print NR,$1,$2,$3,$NF,NF,"hello"}' test1.txt

# 打印test1.txt内容之前和之后做一些其它事情
awk 'BEGIN{print "aaa","bbb"} {print $0} END{print "ccc","ddd"}' test1.txt

# 2. 分隔符
# 准备带#分隔符的测试文件
cat >> test2.txt << EOF
zhangsan#30#2000-01-01#basketball
lisi#40#2001-09-10#football
EOF
# 指定分隔符为#，-F# 和 -v FS='#' 效果是一样的
awk -F# '{print $1,$2}' test2.txt
awk -v FS='#' '{print $1,$2}' test2.txt
# 通过 -v OFS='%' 来制定输出打印的分隔符为%
awk -F# -v OFS='%' '{print $1,$2}' test2.txt
awk -v FS='#' -v OFS='%' '{print $1,$2}' test2.txt

# 3. 自定义变量
awk -v myvar='myvalue' 'BEGIN{print myvar}'
# 自定义变量引用其它变量
testkey=testvalue
awk -v myvar=$testkey 'BEGIN{print myvar}'
```

```bash
# awk练习
# 准备练习文件
cat >> test3.txt <<EOF
wang     4
cui      3
zhao     4
liu      3
liu      3
chang    5
li       2
EOF

# 找出第一列字符串长度为4的行
awk 'length($1)==4{print $1}' test3.txt
# 当第二个字段长度超过4，则执行系统命令创建一个空白文件，名称为第一个字段
awk '{if($2>4){system ("touch "$1)}}' test3.txt
# 第二列求和
awk '{a+=$2}END{print a}' test3.txt
# 第二列求平均值
awk '{a+=$2}END{print a/NR}' test3.txt
# 第二列求最大值
awk 'BEGIN{a=0}{if($2>a){a=$2}}END{print a}' test3.txt
# 计算出第一列的值出现的次数，以及对应的第二列的总和
awk '{a[$1]++;b[$1]+=$2}END{for(i in a){print i,a[i],b[i]}}' test3.txt
```

## 内存分析

```bash
## 查看机器剩余内存
free -mh
# 查看所有进程的内存，降序，单位kb
ps aux --sort -rss
# 查看所有进程的内存，降序，单位kb，前10个高内存进程
ps aux --sort=-rss | head -10
# 杀掉某个名称的所有进程
kill $(ps aux --sort=-rss | grep <app name> | awk '{print $2}')



# 查找java进程
ps -ef|grep java|grep -v grep
# 将活跃对象信息导出到log文件
jmap -histo:live [pid] >a.log
# 利用jmap来生成dump
jmap -dump:live,format=b,file=heap-dump.bin <pid>
# 启动jar包的启动JVM参数中添加以下参数可以在OOM时自动生成dump文件
-XX:+HeapDumpOnOutOfMemoryError

```

## 磁盘占用

```bash
df -h
```

## 文件查找

```bash
# 在当前文件目录下面查找
find . -name "*keyword*"
```

## 参考

[linux命令大全](https://www.runoob.com/linux/linux-command-manual.html)
