# BPF指令集架构简介

> 参考：
>
> [BPF Standardization — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/bpf/standardization/index.html)
>
> [/tools/include/uapi/linux/bpf.h  Linux kernel source tree](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/tools/include/uapi/linux/bpf.h?h=v6.12-rc5)
>
> [/tools/include/uapi/linux/bpf_common.h  Linux kernel source tree](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/tools/include/uapi/linux/bpf_common.h?h=v6.12-rc5)

这篇文章介绍了BPF寄存器和BPF指令。

## 零、 BPF寄存器

BPF有十个通用寄存器和一个帧指针寄存器，所有寄存器都为64bit。

寄存器调用约定如下：

R0：函数调用的返回值和BPF程序的退出值

R1-R5：函数调用的参数

R6-R9：被调用函数的寄存器的值在发生函数调用时将保存下来

R10：帧指针寄存器

R0-R5为易失寄存器，因此在函数调用时应得到保存

BPF 程序在执行 EXIT 之前需要将返回值存储到寄存器 R0 中

在源码中：

<img src="C:\Users\OptimusPrime\AppData\Roaming\Typora\typora-user-images\image-20241101151120285.png" alt="image-20241101151120285" style="zoom:67%;" />

## 一、一致性组（Conformance groups）

文档中规定了一些一致性组，OS必须要支持base32一致性组，选择性支持其他一致性组（类似RVA Profile）。支持一个一致性组意味着要支持该组中所有指令。

文档中规定了如下的一致性组：

1. base32：文档中没有额外说明的指令即属于base32
2. base64：包括base32，外加额外说明的指令
3. atomics32：包括32位原子操作指令
4. atomics64：包括atomics32，外加64位原子操作指令
5. divmul32：包括32位的乘法、除法、取余指令
6. divmul64：包括divmul32，外加64位的乘法、除法、模指令
7. packet：弃用的数据包访问指令

## 二、指令编码格式

BPF有两种指令编码格式：

1. 基本指令编码，一条指令为64位
2. 宽指令编码，一条指令为128位

### 2.1 基本指令编码

基本指令按如下格式编码：

<img src="C:\Users\OptimusPrime\AppData\Roaming\Typora\typora-user-images\image-20241031200237641.png" alt="image-20241031200237641" style="zoom: 67%;" />

* opcode：要进行的操作，按如下方式编码：

  <img src="C:\Users\OptimusPrime\AppData\Roaming\Typora\typora-user-images\image-20241031201154948.png" alt="image-20241031201154948" style="zoom:67%;" />

  * class：指令的类
  * specific：这些位的格式因指令的类而异

* regs：源寄存器和目的寄存器数

  小段序按如下方式编码：

  <img src="C:\Users\OptimusPrime\AppData\Roaming\Typora\typora-user-images\image-20241031201523765.png" alt="image-20241031201523765" style="zoom:67%;" />

  大端序按如下方式编码：

  <img src="C:\Users\OptimusPrime\AppData\Roaming\Typora\typora-user-images\image-20241031201713027.png" alt="image-20241031201713027" style="zoom:67%;" />

* offset：与指针运算一起使用的有符号整数偏移量

* imm：有符号整数立即数

需要注意的是，imm和offset中的值跟随主机的字节序

在源码中：

<img src="C:\Users\OptimusPrime\AppData\Roaming\Typora\typora-user-images\image-20241101151207670.png" alt="image-20241101151207670" style="zoom:67%;" />

### 2.2 宽指令编码

宽指令按如下格式编码：

<img src="C:\Users\OptimusPrime\AppData\Roaming\Typora\typora-user-images\image-20241031203023669.png" alt="image-20241031203023669" style="zoom:67%;" />

* opcode：要进行的操作
* regs：源寄存器和目的寄存器
* offset：与指针运算一起使用的有符号整数偏移量
* imm：有符号整数立即数
* reserved：全为0
* next_imm：第二个有符号整数立即数

*未在源码中找到，疑似未支持*

### 2.3 指令类

所有的指令类如下 ：

![image-20241031203710591](C:\Users\OptimusPrime\AppData\Roaming\Typora\typora-user-images\image-20241031203710591.png)

在源码中：

<img src="C:\Users\OptimusPrime\AppData\Roaming\Typora\typora-user-images\image-20241101151335956.png" alt="image-20241101151335956" style="zoom:67%;" />

*BPF_RET和BPF_MISC应该是被舍弃掉，但在源码中仍存在，疑惑*

![image-20241101151359712](C:\Users\OptimusPrime\AppData\Roaming\Typora\typora-user-images\image-20241101151359712.png)



## 三、算数和跳转指令

对于算术和跳转指令（ALU、ALU64、JMP、JMP32），opcode域划分为如下三个部分：

<img src="C:\Users\OptimusPrime\AppData\Roaming\Typora\typora-user-images\image-20241031204033352.png" alt="image-20241031204033352" style="zoom: 67%;" />

* code：操作码，根据指令类而变化

* s(source)：源操作数域，除非特别说明，为下面两者之一

  <img src="C:\Users\OptimusPrime\AppData\Roaming\Typora\typora-user-images\image-20241031204414320.png" alt="image-20241031204414320" style="zoom:67%;" />

  在源码中：

  <img src="C:\Users\OptimusPrime\AppData\Roaming\Typora\typora-user-images\image-20241101152932822.png" alt="image-20241101152932822" style="zoom:67%;" />

* class：指令类

### 3.1 算术指令

ALU使用32位操作数，ALU使用64位操作数。ALU64指令属于base64一致性组。code域定义的指令如下所示：

<img src="C:\Users\OptimusPrime\AppData\Roaming\Typora\typora-user-images\image-20241031204828043.png" alt="image-20241031204828043" style="zoom:67%;" />

在源码中：

<img src="C:\Users\OptimusPrime\AppData\Roaming\Typora\typora-user-images\image-20241101151820641.png" alt="image-20241101151820641" style="zoom:67%;" />

![image-20241101152407481](C:\Users\OptimusPrime\AppData\Roaming\Typora\typora-user-images\image-20241101152407481.png)

可以看到，code域相同的指令在源码中未重复列出，因为它们将在offset域做区分

算术指令中产生的溢出是被允许的，因此此值会发生回绕。如果BPF程序导致被0除的操作发生，目的寄存器会被置0。如果产生了模0操作，对于ALU64，目的寄存器的值不变，对于ALU ，目的寄存器的高32位被置0。

对于ALU指令，源寄存器和目的寄存器的高32位被置为0。

需要注意的是，大部分算术指令将offset置为0，只有三个指令（SDIV、SMOD、MOVSX）有非零的的offset。

除法和模指令同时支持有符号和无符号数。

对于无符号操作（DIV和MOD），对于ALU，imm为32位无符号数。对于ALU64，imm先从32位符号扩展到64位，然后解释为64位有符号数。

需要注意的是，被除数或除数为负数时，有符号模操作有着不同的定义，具体实现往往因语言而异。本规范要求有符号的模操作必须使用截断除法。

MOVSX指令执行符号扩展的移动操作。``` {MOVSX, X, ALU}```将8bit、16bit操作数符号扩展到32bit，高32位保持为0。``` {MOVSX, X, ALU64}```将8bit、16bit、32bit操作数符号扩展到64bit。MOVSX的操作数为寄存器。

NEG指令的操作数为立即数。

对于64位操作，移位指令的掩码为0x3F，对32为操作，则为0x1F。

### 3.2 字节交换指令

字节交换指令的指令类为ALU和ALU64，code域为END。

字节交换指令只对目标寄存器进行操作，不使用源寄存器或立即数。

对于ALU，1bit的code域用来选择操作数要转换的字节序（如下图）。对于ALU，1bit的code域必须置为0。

<img src="C:\Users\OptimusPrime\AppData\Roaming\Typora\typora-user-images\image-20241031215211203.png" alt="image-20241031215211203" style="zoom:67%;" />

imm域编码为交换操作的宽度，支持以下宽度：16、32和64bit。64位操作属于base64一致性组，其他宽度指令属于base32一致性组。

### 3.3 跳转指令

JMP32操作数为32位，属于base32一致性组，JMP操作数为64位，属于base64一致性组。code域编码格式如下：

![image-20241031220950025](C:\Users\OptimusPrime\AppData\Roaming\Typora\typora-user-images\image-20241031220950025.png)

在源码中：

<img src="C:\Users\OptimusPrime\AppData\Roaming\Typora\typora-user-images\image-20241101153321193.png" alt="image-20241101153321193" style="zoom:67%;" />

<img src="C:\Users\OptimusPrime\AppData\Roaming\Typora\typora-user-images\image-20241101153531011.png" alt="image-20241101153531011" style="zoom:67%;" />

增量的偏移量offset以64bit为单位。

需要注意的是有两种JA指令。JMP类允许使用offset域的16bit偏移量，而JMP32类允许使用imm的32bit偏移量。大于16bit的条件跳转会被转换为小于16bit的条件跳转加32bit的非条件跳转。

CALL和JA指令属于base32一致性组。

## 四、加载存储指令

对于加载和存储指令（LD、LDX、ST和STX），8bit opcode域编码格式如下：

<img src="C:\Users\OptimusPrime\AppData\Roaming\Typora\typora-user-images\image-20241031223724283.png" alt="image-20241031223724283" style="zoom: 67%;" />

* mode为下列之一：

  <img src="C:\Users\OptimusPrime\AppData\Roaming\Typora\typora-user-images\image-20241031223843517.png" alt="image-20241031223843517" style="zoom:67%;" />

  在源码中：

  <img src="C:\Users\OptimusPrime\AppData\Roaming\Typora\typora-user-images\image-20241101153738119.png" alt="image-20241101153738119" style="zoom:67%;" />

  *BPF_LEN和BPF_MSH指令似乎已经遗弃*

  <img src="C:\Users\OptimusPrime\AppData\Roaming\Typora\typora-user-images\image-20241101153919505.png" alt="image-20241101153919505" style="zoom:67%;" />

* size为下列之一：

  <img src="C:\Users\OptimusPrime\AppData\Roaming\Typora\typora-user-images\image-20241031224021241.png" alt="image-20241031224021241" style="zoom:67%;" />

  在源码中：

  <img src="C:\Users\OptimusPrime\AppData\Roaming\Typora\typora-user-images\image-20241101154158151.png" alt="image-20241101154158151" style="zoom:67%;" />

  <img src="C:\Users\OptimusPrime\AppData\Roaming\Typora\typora-user-images\image-20241101154224859.png" alt="image-20241101154224859" style="zoom:67%;" />

* class：指令类

### 4.1 常规加载存储指令

mode域为MEM时，为常规加载存储指令，该指令用于在寄存器和内存直接传输数据。STX的源操作数为寄存器中的值。

### 4.2 符号扩展加载操作

mode域为MEMSX时，为符号扩展常规加载存储指令，该指令用于在寄存器和内存直接传输数据。

### 4.3 原子操作

原子操作对一块内存进行操作，并且不能被其他对该内存区域的操作所中断。所有原子操作mode域编码为ATOMIC

```{ATOMIC, W, STX}```为32位操作，属于atomic32一致性组

```{ATOMIC, DW, STX}```为64位操作，属于atomic64一致性组	

不支持8bit和16bit宽的原子操作

imm域用做编码实际的原子操作：

<img src="C:\Users\OptimusPrime\AppData\Roaming\Typora\typora-user-images\image-20241031225917006.png" alt="image-20241031225917006" style="zoom:67%;" />

除了上述简单的原子操作，还有一个修饰符和两个复杂原子操作：

<img src="C:\Users\OptimusPrime\AppData\Roaming\Typora\typora-user-images\image-20241031230148823.png" alt="image-20241031230148823" style="zoom:67%;" />

在源码中：<img src="C:\Users\OptimusPrime\AppData\Roaming\Typora\typora-user-images\image-20241101154339627.png" alt="image-20241101154339627" style="zoom:67%;" />

对于简单原子操作，FETCH是可选的，对于复杂原子操作，则是一直设置的。

XCHG操作原子交换src寄存器和dst+offset地址中的值

CMPXCHG操作原子比较dst+offset地址中的值和R0寄存器中的值，如果它们相等，将会用src寄存器中的值替换dst+offset地址中的值。

### 4.4 64bit立即数指令

mode域使用IMM编码使用宽指令编码，并且使用基本指令中的src_reg域来保存操作子类型。

下表定义了一系列使用子类型的```{IMM, DW, LD}```指令

<img src="C:\Users\OptimusPrime\AppData\Roaming\Typora\typora-user-images\image-20241031231740969.png" alt="image-20241031231740969" style="zoom:67%;" />

其中

* map_by_fd(imm)意味着将32位文件描述符转换为map地址
* map_by_idx(imm)意味着将32位索引转换为map地址
* map_val(map)获取给定map中第一个值的地址
* var_addr（imm）获取具有给定ID的平台变量的地址
* code_addr(imm) 获取（64 位）指令中指定相对偏移处的指令地址
* 反汇编程序可以使用“imm type”进行显示
* “dst 类型”可用于验证和 JIT 编译

#### 4.4.1 Maps

在一些平台上，maps是BPF程序可以访问的共享内存。

如果平台支持，每个map将会有一个文件描述符，而map_by_fd(imm)意味着通过特定的文件描述符得到map。每个BPF程序在加载的时候也可以定义为能够使用一组跟程序相关的map，map_by_idx(imm)意味着在这个BPF程序相关的一组maps中，通过索引得到map。

#### 4.4.2 平台变量

平台变量是内存区域，由整数id确定，在运行时暴露出来，在某些平台上可以被BPF程序访问到。var_addr(imm)操作的意思是获取给定id标识的内存区域的地址。































































