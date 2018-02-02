---
title: 学习笔记：Makefile
tags: makefile
categories: 学习笔记
---
# Makefile
## 执行顺序
make的执行顺序不同于shell script的从上到下，而是先读取所有makefile建立依赖关系，然后根据依赖关系重建目标。比如
```Makefile
MSG="HELLO WORLD"

all:
    @echo $(MSG)

MSG="AFTER ALL"
```
执行make的结果为
```
AFTER ALL
```
因为在执行构建`all`前，变量`MSG`已经是"AFTER ALL"。  


## 赋值
`=`会将整个makefile展开后再决定变量的值。比如  
```makefile
MSG="BEFORE"  
MSG2=$(MSG)  
MSG="AFTER" 
```
变量`MSG2`的值为"AFTER"。  
  
`:=`会取决于变量在makefile中的位置，而不是makefile展开后的值。
```makefile
MSG="BEFORE"  
MSG2:=$(MSG)  
MSG="AFTER"  
```
变量`MSG2`的值为"BEFORE"。  
  
## 特殊符号
`$@`表示目标名称
`$^`表示所有依赖文件
`$<`表示第一个依赖文件

## 编译目标文件到指定目录
```makefile
ifeq ($(OUT_DIR),)
OUT_DIR = .
endif

SRCS=$(wildcard *.c)
OBJS=$(patsubst %.c,$(OUT_DIR)/%.o,$(SRCS))
CFLAGS=-Wall

all: $(OBJS)
    @echo "  LINK    $@"
    @cc -o $(OUT_DIR)/$@ $^ $(CFLAGS) 

$(OUT_DIR)/%.o: %.c 
    @mkdir -p $(OUT_DIR)
    @echo "  CC      $@"
    @cc -c $< -o $@
```
`OUT_DIR`为目标文件目录，默认为当前目录。  
`SRCS`为当前目录下所有源文件。  
`OBJS`为所有源文件对应的目标文件，文件名从替换源文件的`.c`为`.o`得到。这里用到了`patsubst`函数，把`$(SRCS)`中的`%.c`替换成`$(OUT_DIR)/%.o`。  
`cc -c $< -o $@`通过`-o`选项指定object文件的路径。  

