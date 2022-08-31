# bpftrace 参考指南

这是一个在不断演进中的项目，如果某些内容缺失，请查看 bpftrace 的源码，看看那里是否可以找到。

如果你发现某些内容已经过时了，请提交一个问题或者一个更新文档的 PR。


# 用法

没有命令选项时 bpftrace 汇总输出命令行的用法

```
# bpftrace
USAGE:
    bpftrace [options] filename
    bpftrace [options] -e 'program'

OPTIONS:
    -B MODE        output buffering mode ('line', 'full', or 'none')
    -d             debug info dry run
    -dd            verbose debug info dry run
    -e 'program'   execute this program
    -h             show this help message
    -I DIR         add the specified DIR to the search path for include files.
    --include FILE adds an implicit #include which is read before the source file is preprocessed.
    -l [search]    list probes
    -p PID         enable USDT probes on PID
    -c 'CMD'       run CMD and enable USDT probes on resulting process
    -q             keep messages quiet
    -v             verbose messages
    -k             emit a warning when a bpf helper returns an error (except read functions)
    -kk            check all bpf helper functions
    --version      bpftrace version

ENVIRONMENT:
    BPFTRACE_STRLEN             [default: 64] bytes on BPF stack per str()
    BPFTRACE_NO_CPP_DEMANGLE    [default: 0] disable C++ symbol demangling
    BPFTRACE_MAP_KEYS_MAX       [default: 4096] max keys in a map
    BPFTRACE_MAX_PROBES         [default: 512] max number of probes bpftrace can attach to
    BPFTRACE_MAX_BPF_PROGS      [default: 512] max number of generated BPF programs
    BPFTRACE_CACHE_USER_SYMBOLS [default: auto] enable user symbol cache
    BPFTRACE_VMLINUX            [default: none] vmlinux path used for kernel symbol resolution
    BPFTRACE_BTF                [default: none] BTF file

EXAMPLES:
bpftrace -l '*sleep*'
    list probes containing "sleep"
bpftrace -e 'kprobe:do_nanosleep { printf("PID %d sleeping...\n", pid); }'
    trace processes calling sleep
bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'
    count syscalls by process name
```

## 1. Hello World

一个 bfptrace 最基础的示例程序

```
# bpftrace -e 'BEGIN { printf("Hello, World!\n"); }'
Attaching 1 probe...
Hello, World!
^C
```

这个程序的语法将在 [语言](https://) 章节进行解释，本章我们主要关注工具的用法。
程序将在敲击 Ctrl-C 或 exist() 函数被调用后停止运行，当一个程序退出所有被占用的 map 会被打印：后面的章节会详细解释这个行为以及 maps。

## 2. -e 'program'：单行程序

-e 选项可以指定一段程序，这也是构建当行程序的一种方法。

```
# bpftrace -e 'tracepoint:syscalls:sys_enter_nanosleep { printf("%s is sleeping.\n", comm); }'
Attaching 1 probe...
iscsid is sleeping.
irqbalance is sleeping.
iscsid is sleeping.
iscsid is sleeping.
[...]
```

当程序调用 nanosleep 系统调用时，这个实例程序就会打印。同样，程序的语法也将会在[语言](https://) 章节进行解释。

## 3. 文件名：程序文件

被保存在文件中的程序通常叫做脚本，可以通过指定文件的名称来执行它。我们通常使用 .bt 作为文件的拓展名，取自 bpftrace 的缩写，但是文件拓展名是被忽略的。

例如，通过 cat -n 罗列 sleepers.bt 文件（这会输出行信息）

```
# cat -n sleepers.bt
1 tracepoint:syscalls:sys_enter_nanosleep
2 {
3   printf("%s is sleeping.\n", comm);
4 }
```

执行 sleepers.bt
```
# bpftrace sleepers.bt
Attaching 1 probe...
iscsid is sleeping.
iscsid is sleeping.
[...]
```

也可以把它制作成可执行程序独立运行。首先在文件的首行添加解析器行，其中包含安装 bpftrace 的安装路径（/usr/local/bin 是默认值）或者是 env 的路径（通常是 /usr/bin/env），然后是 bpftrace（所以会在 $PATH 环境变量中找到 bpftrace）
```
1 #!/usr/local/bin/bpftrace
2
3 tracepoint:syscalls:sys_enter_nanosleep
4 {
5   printf("%s is sleeping.\n", comm);
6 }
```

然后使其可执行：
```
# chmod 755 sleepers.bt
# ./sleepers.bt
Attaching 1 probe...
iscsid is sleeping.
iscsid is sleeping.
[...]
```

## 4. -l: 列出所有探针

通过 -l 可以列出所有 tracepoint 与 kprobe 库的探针
```
# bpftrace -l | more
tracepoint:xfs:xfs_attr_list_sf
tracepoint:xfs:xfs_attr_list_sf_all
tracepoint:xfs:xfs_attr_list_leaf
tracepoint:xfs:xfs_attr_list_leaf_end
[...]
# bpftrace -l | wc -l
46260
```

## 5. -d: 调试选项

-d 选项会在不运行程序的情况下输出调试信息。这对 bpftrace 程序进行调试自身问题的时候是非常有用的。你也可以通过 -dd 选项生成更详细的调试信息，它会打印出未经优化的 IR。

如果你只是 bpftrace 的终端用户，你可能通常不会使用到 -d 或者 -v 选项，你可以直接跳到 [语言]() 章节。
```
# bpftrace -d -e 'tracepoint:syscalls:sys_enter_nanosleep { printf("%s is sleeping.\n", comm); }'
Program
 tracepoint:syscalls:sys_enter_nanosleep
  call: printf
   string: %s is sleeping.\n
   builtin: comm
[...]
```

输出以 Program 开始，然后是一个抽象语法树表示程序。

紧接着：
```
[...]
%printf_t = type { i64, [16 x i8] }
[...]
define i64 @"tracepoint:syscalls:sys_enter_nanosleep"(i8*) local_unnamed_addr section "s_tracepoint:syscalls:sys_enter_nanosleep" {
entry:
  %comm = alloca [16 x i8], align 1
  %printf_args = alloca %printf_t, align 8
  %1 = bitcast %printf_t* %printf_args to i8*
  call void @llvm.lifetime.start.p0i8(i64 -1, i8* nonnull %1)
  %2 = getelementptr inbounds [16 x i8], [16 x i8]* %comm, i64 0, i64 0
  %3 = bitcast %printf_t* %printf_args to i8*
  call void @llvm.memset.p0i8.i64(i8* nonnull %3, i8 0, i64 24, i32 8, i1 false)
  call void @llvm.lifetime.start.p0i8(i64 -1, i8* nonnull %2)
  call void @llvm.memset.p0i8.i64(i8* nonnull %2, i8 0, i64 16, i32 1, i1 false)
  %get_comm = call i64 inttoptr (i64 16 to i64 (i8*, i64)*)([16 x i8]* nonnull %comm, i64 16)
  %4 = getelementptr inbounds %printf_t, %printf_t* %printf_args, i64 0, i32 1, i64 0
  call void @llvm.memcpy.p0i8.p0i8.i64(i8* nonnull %4, i8* nonnull %2, i64 16, i32 1, i1 false)
  %pseudo = call i64 @llvm.bpf.pseudo(i64 1, i64 1)
  %get_cpu_id = call i64 inttoptr (i64 8 to i64 ()*)()
  %perf_event_output = call i64 inttoptr (i64 25 to i64 (i8*, i8*, i64, i8*, i64)*)(i8* %0, i64 %pseudo, i64 %get_cpu_id, %printf_t* nonnull %printf_args, i64 24)
  call void @llvm.lifetime.end.p0i8(i64 -1, i8* nonnull %1)
  ret i64 0
[...]
```
这段展示了 LLVM 的 IR，随后将会被编译到 BPF 中。

## 6. -v: 详细输出
-v 选项会在程序运行的时候打印更多的信息：

```
# bpftrace -v -e 'tracepoint:syscalls:sys_enter_nanosleep { printf("%s is sleeping.\n", comm); }'
Attaching 1 probe...

The verifier log:
0: (bf) r6 = r1
1: (b7) r1 = 0
2: (7b) *(u64 *)(r10 -24) = r1
3: (7b) *(u64 *)(r10 -32) = r1
4: (7b) *(u64 *)(r10 -40) = r1
5: (7b) *(u64 *)(r10 -8) = r1
6: (7b) *(u64 *)(r10 -16) = r1
7: (bf) r1 = r10
8: (07) r1 += -16
9: (b7) r2 = 16
10: (85) call bpf_get_current_comm#16
11: (79) r1 = *(u64 *)(r10 -16)
12: (7b) *(u64 *)(r10 -32) = r1
13: (79) r1 = *(u64 *)(r10 -8)
14: (7b) *(u64 *)(r10 -24) = r1
15: (18) r7 = 0xffff9044e65f1000
17: (85) call bpf_get_smp_processor_id#8
18: (bf) r4 = r10
19: (07) r4 += -40
20: (bf) r1 = r6
21: (bf) r2 = r7
22: (bf) r3 = r0
23: (b7) r5 = 24
24: (85) call bpf_perf_event_output#25
25: (b7) r0 = 0
26: (95) exit
processed 26 insns (limit 131072), stack depth 40

Attaching tracepoint:syscalls:sys_enter_nanosleep
iscsid is sleeping.
iscsid is sleeping.
[...]

```
这里包含 bpf 验证器的日志：从内核的验证器得到的日志信息

## 7. 预处理器选项
-I 可以用来向 bpftrace 查找头文件的的目录列表中添加目录。可以被定义多次。
```
# cat program.bt
#include <foo.h>

BEGIN { @ = FOO }

# bpftrace program.bt
definitions.h:1:10: fatal error: 'foo.h' file not found

# /tmp/include
foo.h

# bpftrace -I /tmp/include program.bt
Attaching 1 probe...
```

--include 选项默认用来添加头文件。可以被定义多次。头文件在定义的时候被引入，头文件被引入的顺序与它被定义的顺序相同，并且它们要优先于程序中引入的头文件。

```
# bpftrace --include linux/path.h --include linux/dcache.h \
    -e 'kprobe:vfs_open { printf("open path: %s\n", str(((struct path *)arg0)->dentry->d_name.name)); }'
Attaching 1 probe...
open path: .com.google.Chrome.ASsbu2
open path: .com.google.Chrome.gimc10
open path: .com.google.Chrome.R1234s
```

## 8. 其他选项
- --version 选项打印 bpftrace 的版本
```
# bpftrace --version
bpftrace v0.8-90-g585e-dirty
```

- --no-warnings 选项用来关闭告警信息

## 9. 环境变量

### 9.1 BPFTRACE_STRLEN

默认值：64
str() 函数为 string 在 BPF 栈空间中申请的的字节数量
如果你希望读取更大的字符串可以调大该值
需要注意的是 BPF 的占空间很小（512 bytes），并且在你使用 printf() 函数的时候还要话费双倍的空间（它同时构建了输出缓冲区）。因此在实践中你最多只能把该值调大到 200 bytes。

支持更大的字符串在这里讨论 [讨论中]()。

### 9.2 BPFTRACE_NO_CPP_DEMANGLE

默认值：0
C++ 的符号解析在用户空间的栈追踪中默认是开启的。
这个功能可以通过设置这个环境变量的值为 1 进行关闭

### 9.3 BPFTRACE_MAP_KEYS_MAX

默认值：4096

这是一个 map 中可以存储 key 的最大数值。增加这个值会消耗更多的内存以及延长启动时间。
有一些场景你会希望调大它，例如：栈追踪的采样，为每个页记录时间戳等

### 9.4 BPFTRACE_MAX_PROBES

默认值：512
这是 bpftace 可以链接探针的最大数量，增加这个值会消耗更多的内存以及延长启动时间并且会带来更高的性能开销甚至导致系统僵死或者崩溃。

### 9.5 BPFTRACE_CACHE_USER_SYMBOLS

默认值：如果 ASLR（地址中间配置随机化）启用且 -c 选项没有被提供时，默认值为 0 ，否则为 1
默认只有当关闭 ASLR(地址空间随机化)时， bpftrace 才缓存符号解析结果。这是因为如果启动 ASLR 每次程序执行符号地址都会进行改变。然而关闭缓存会导致一些性能问题。因此可以设置该环境变量的值为 1 来强制开启缓存，如果仅仅追踪一个程序的执行是十分推荐这样做的。


### 9.6 BPFTRACE_VMLINUX

默认值：None
它指定了在将 kprobe 附加 offset 时，解析内核符号的 vmlinux 的路径。如果没有指定该值，bpftrace 会在预声明的位置查找 vmlinux。详情请查看 src/attached_probe.cpp:find_vmlinux()

### 9.7 BPFTRACE_BTF

默认值：None
BTF 文件的路径。默认情况，bpftrace 在几个位置查找 BTF 文件。详情请查看 src/btf.cpp 文件。

### 9.8 BPFTRACE_PERF_RB_PAGES

默认值：64
每个 CPU 分配给 perf 环缓冲区的页数。这个值必须是 2 的幂。

如果你获取很多丢弃的事件，导致 bpftrace 来不得及处理环缓冲区中的事件。调高这个值令更多的事件进入队列是有帮助的。代价就是消耗了更多的内存。

### 9.9 BPFTRACE_MAX_BPF_PROGS

默认值：512

bpftrace 可以生成 BPF 程序（函数）的最大数量。这个限制的目的是为了防止 bpftrace 生成许多的 probe 占用过多的资源导致死机。

## 10. Clang 的环境变量
bpftrace 使用 Clang 的 C 接口 libclang 来解析头文件，因此可以使用环境变量改变 Clang 的工具链。例如：如果 header 文件不在默认的文件夹中，可以通过设置 CPATH 或者 C_INCLUDE_PATH 环境变量来指定 header 文件。更多关于环境变量的详细信息及使用方法请查看 Clang 的文档。

# 语言

## 1. {...}: Action 代码块
语法：probe,[probe,...] /filter/ { action }
一个 bpftrace 程序可以有多个 action 代码块。过滤器是可选的。

例如：
```
# bpftrace -e 'kprobe:do_sys_open { printf("opening: %s\n", str(arg1)); }'
Attaching 1 probe...
opening: /proc/cpuinfo
opening: /proc/stat
opening: /proc/diskstats
opening: /proc/stat
opening: /proc/vmstat
[...]
```
这是一个单行 bpftrace 程序。probe 为 kprobe:do_sys_open。当 probe 开火（探测指标时间发生）后，由 printf() 语句构成的 action 就会被执行。接下来几个章节将对 probe 与 action 做详细介绍。

## 2. /.../: Filtering

语法：/filter/

过滤器（正如其名）可以在 probe 名称的后面添加。probe 仍然可以探测，但是除非过滤器的值为 true，否则将会跳过 action 的执行。

示例：
```
# bpftrace -e 'kprobe:vfs_read /arg2 < 16/ { printf("small read: %d byte buffer\n", arg2); }'
Attaching 1 probe...
small read: 8 byte buffer
small read: 8 byte buffer
small read: 8 byte buffer
small read: 8 byte buffer
small read: 8 byte buffer
small read: 12 byte buffer
^C
```

```
# bpftrace -e 'kprobe:vfs_read /comm == "bash"/ { printf("read by %s\n", comm); }'
Attaching 1 probe...
read by bash
read by bash
read by bash
read by bash
^C
```

## 3. //, /*: 注释
语法
```
// 单行注释
/*
 * 多行注释
 */
```

## 4. 字面量

支持整型、字符型、字符串字面量，整型字面量是一串数字序列，可选包含 _ 作为数量位的分隔符。也支持科学计数法，但是仅支持整型的方式，因为 BPF 不支持浮点数。
```
# bpftrace -e 'BEGIN { printf("%lu %lu %lu", 1000000, 1e6, 1_000_000)}'
Attaching 1 probe...
1000000 1000000 1000000
```

字符型字面量被单引号包裹着，例如：'a' 以及 '@'。
字符串型字面量被双引号包裹着，例如："a string"。

## 5. -> : C 结构体访问符

tracepoint 示例：
```
# bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s %s\n", comm, str(args->filename)); }'
Attaching 1 probe...
snmpd /proc/diskstats
snmpd /proc/stat
snmpd /proc/vmstat
[...]
```

这里返回的是 args 结构的成员 filename，该结构为 tracepoint probes 提供了所需的 tracepoint arguments 。关于这个结构体的详细内容请查看[静态追踪，内核参数章节]()。

kprobe 示例：

```
# cat path.bt
#include <linux/path.h>
#include <linux/dcache.h>

kprobe:vfs_open
{
	printf("open path: %s\n", str(((struct path *)arg0)->dentry->d_name.name));
}

# bpftrace path.bt
Attaching 1 probe...
open path: dev
open path: if_inet6
open path: retrans_time_ms
[...]
```

这里使用的是动态追踪的内核函数 vfs_open()，通过简短的 path.bt 的脚本。一些内核的头文件需要被引入用来解析 path 与 dentry 的结构体。

## 6. 结构体：结构体的定义

例子：
```
// from fs/namei.c:
struct nameidata {
        struct path     path;
        struct qstr     last;
        // [...]
};
```

在需要的时候你可以定义自己的结构体。在一些场景，内核的结构体并没有被定义在内核的头文件包中，而是在 bpftrace 工具中手工定义的（或者部分结构就可以解引用所有的成员）

## 7. ？ ： ：三元运算符
例子：
```
# bpftrace -e 'tracepoint:syscalls:sys_exit_read { @error[args->ret < 0 ? - args->ret : 0] = count(); }'
Attaching 1 probe...
^C

@error[11]: 24
@error[0]: 78
```

```
# bpftrace -e 'BEGIN { pid & 1 ? printf("Odd\n") : printf("Even\n"); exit(); }'
Attaching 1 probe...
Odd
```

## 8. if () {...} else {...} : if else 语句
例子：
```
# bpftrace -e 'tracepoint:syscalls:sys_enter_read { @reads = count();
    if (args->count > 1024) { @large = count(); } }'
Attaching 1 probe...
^C
@large: 72

@reads: 80
```

## 9. unroll() {...}: 循环展开
例子：
```
# bpftrace -e 'kprobe:do_nanosleep { $i = 1; unroll(5) { printf("i: %d\n", $i); $i = $i + 1; } }'
Attaching 1 probe...
i: 1
i: 2
i: 3
i: 4
i: 5
^C
```

## 10. ++ 与 --：自增运算符

++ 与 -- 可以很方便的增加或减少 map 或者变量定义中的计数

需要注意的是如果没有被声明或者定义的时候 map 会隐式的声明并初始化为 0 。变量必须在使用这些运算符之前进行初始化。

示例 - 变量：
```
bpftrace -e 'BEGIN { $x = 0; $x++; $x++; printf("x: %d\n", $x); }'
Attaching 1 probe...
x: 2
^C
```

示例 - map:
```
bpftrace -e 'k:vfs_read { @++ }'
Attaching 1 probe...
^C

@: 12807
```

示例 - 有键的 map：
```
# bpftrace -e 'k:vfs_read { @[probe]++ }'
Attaching 1 probe...
^C

@[kprobe:vfs_read]: 13369
```

## 11. []：数组的访问

你可以使用数组访问操作符访问一维常量数组。
示例：
```
# bpftrace -e 'struct MyStruct { int y[4]; } uprobe:./testprogs/array_access:test_struct {
    $s = (struct MyStruct *) arg0; @x = $s->y[0]; exit(); }'
Attaching 1 probe...

@x: 1
```

## 12. 整型转换
整型在内部通常表示为 64 位有符号整数。如果你需要另外一种表示，你可能需要使用如下内置类型进行转换：
| 类型 | 解释 |
|----|----|
|uint8|unsigned 8 bit integer|
|int8|signed 8 bit integer|
|uint16|unsigned 16 bit integer|
|int16|signed 16 bit integer|
|uint32|unsigned 32 bit integer|
|int32|signed 32 bit integer|
|uint64|unsigned 64 bit integer|
|int64|signed 64 bit integer|

示例：
```
# bpftrace -e 'BEGIN { $x = 1<<16; printf("%d %d\n", (uint16)$x, $x); }'
Attaching 1 probe...
0 65536
^C
```

## 13. 循环结构
实验性的：
内核版本：5.3
bpftrace 支持 C 样式的 while 循环：
```
# bpftrace -e 'i:ms:100 { $i = 0; while ($i <= 100) { printf("%d ", $i); $i++} exit(); }'
```

使用 continue 与 break 可以使循环短路。

## 14. return: 提前终止
关键字 return 用来退出当前的 probe。与 exit() 的区别在于 return 并没有退出 bpftrace

## 15. ( , )：元组
在 N 大于等于 1 时， N-元组是被支持的。
通过 . 操作符可以支持元组的索引。元组一旦被创建就不可以更改
示例：
```
# bpftrace -e 'BEGIN { $t = (1, 2, "string"); printf("%d %s\n", $t.1, $t.2); }'
Attaching 1 probe...
2 string
^C
```

# 采样器

- kprobe - 内核函数开始时采样
- kretprobe - 内核函数返回时采样
- uprobe - 用户函数开始时采样
- uretprobe - 用户函数返回时采样
- tracepoint - 内核的静态追踪点
- usdt - 用户级的静态追踪点
- profile - 周期性采样
- interval - 周期性输出
- software - 内核软事件
- hardware - 处理器级事件

一些类型的探测器允许通过通配符匹配多个探测，例如：kprobe:vfs_*。你也可以通过一个以逗号分割的 action 代码块列表为多个探测点指定 action。
双引号字符串可以被用来在追踪点定义时使用转义字符。

## 1. kprobe/kretprobe: 内核级的动态追踪
语法：
```
kprobe:function_name[+offset]
kretprobe:function_name
```

这里使用 kprobes（一种内核能力）。kprobe 在内核函数开始的时候开始探测，kretprobe 在函数返回的时候开始探测。

示例：
```
# bpftrace -e 'kprobe:do_nanosleep { printf("sleep by %d\n", tid); }'
Attaching 1 probe...
sleep by 1396
sleep by 3669
sleep by 1396
sleep by 27662
sleep by 3669
^C
```

kprobe 页允许在被探测的函数中指定偏移量：

```
# gdb -q /usr/lib/debug/boot/vmlinux-`uname -r` --ex 'disassemble do_sys_open'
Reading symbols from /usr/lib/debug/boot/vmlinux-5.0.0-32-generic...done.
Dump of assembler code for function do_sys_open:
   0xffffffff812b2ed0 <+0>:     callq  0xffffffff81c01820 <__fentry__>
   0xffffffff812b2ed5 <+5>:     push   %rbp
   0xffffffff812b2ed6 <+6>:     mov    %rsp,%rbp
   0xffffffff812b2ed9 <+9>:     push   %r15
...
# bpftrace -e 'kprobe:do_sys_open+9 { printf("in here\n"); }'
Attaching 1 probe...
in here
...
```

这个地址通过 vmlinux （调试符号）进行检查如果与指令对齐则添加到函数中，否则将不会添加成功。

```
# bpftrace -e 'kprobe:do_sys_open+1 { printf("in here\n"); }'
Attaching 1 probe...
Could not add kprobe into middle of instruction: /usr/lib/debug/boot/vmlinux-5.0.0-32-generic:do_sys_open+1
```
如果 bpftrace 编译的之后配置了 ALLOW_UNSAFE_PROBE 选项，你可以使用 --unsafe 选项来跳过检查。但这种情况 linux 内核仍然会进行指令集的对齐检查的。

原始示例在：[]() []()

## 2. kprobe/kretprobe：动态追踪内核参数
语法：
```
kprobe: arg0, arg1, ..., argN
kretprobe: retval
```

实参可以通过这些变量名称访问。arg0 是第一个参数并且只能被一个 kprobe 探测器访问。retval 是一个被探测函数的返回值，并且只能被 kretprobe 探测器访问。

示例：
```
# bpftrace -e 'kprobe:do_sys_open { printf("opening: %s\n", str(arg1)); }'
Attaching 1 probe...
opening: /proc/cpuinfo
opening: /proc/stat
opening: /proc/diskstats
opening: /proc/stat
opening: /proc/vmstat
[...]
```

```
# bpftrace -e 'kprobe:do_sys_open { printf("open flags: %d\n", arg2); }'
Attaching 1 probe...
open flags: 557056
open flags: 32768
open flags: 32768
open flags: 32768
[...]
```

```
# bpftrace -e 'kretprobe:do_sys_open { printf("open flags: %d\n", retval); }'
Attaching 1 probe...
open flags: 557056
open flags: 32768
open flags: 32768
open flags: 32768
[...]
```

一个结构体实参的示例：
```
# cat path.bt
#include <linux/path.h>
#include <linux/dcache.h>

kprobe:vfs_open
{
	printf("open path: %s\n", str(((struct path *)arg0)->dentry->d_name.name));
}

# bpftrace path.bt
Attaching 1 probe...
open path: dev
open path: if_inet6
open path: retrans_time_ms
[...]
```
这里的 arg0 被强制转换为了 struct path 指针类型，因为那是 vfs_open() 函数的第一个参数。结构体的支持与 bcc 相同，并且基于可用过的 linux headers。这就意味着并不是所有的结构体都可以使用，并且你可能需要手工定义一些结构体。

如果内核有 BTF 数据，所有的内核结构体都可以在不定义他们的前提下使用，例如：
```
# bpftrace -e 'kprobe:vfs_open { printf("open path: %s\n", \
                                 str(((struct path *)arg0)->dentry->d_name.name)); }'
Attaching 1 probe...
open path: cmdline
open path: interrupts
[...]
```

更多详细信息请查看 [BTF 的支持]()

原始示例：[kprobe]() [kretprobe]()

## 3. uprobe / uretprobe：用户级动态追踪
语法：
```
uprobe:library_name:function_name[+offset]
uprobe:library_name:address
uretprobe:library_name:function_name
```

这里使用的是 uprobes。uprobe 在用户级函数开始时开始探测， uretprobo 得函数返回时开始探测。

为了列出可用的 uprobes，可以使用任意程序来列出二进制程序的文本段符号，比如 objdump 与 nm。例如：
```
# objdump -tT /bin/bash | grep readline
00000000007003f8 g    DO .bss	0000000000000004  Base        rl_readline_state
0000000000499e00 g    DF .text	00000000000001c5  Base        readline_internal_char
00000000004993d0 g    DF .text	0000000000000126  Base        readline_internal_setup
000000000046d400 g    DF .text	000000000000004b  Base        posix_readline_initialize
000000000049a520 g    DF .text	0000000000000081  Base        readline
[...]
```

这里从 /bin/bash 中列出了多个包含 readline 的函数。这些都可以通过 uprobe 与 uretprobe 进行探测。

示例：
```
# bpftrace -e 'uretprobe:/bin/bash:readline { printf("read a line\n"); }'
Attaching 1 probe...
read a line
read a line
read a line
^C
```

在追踪的时候，出发了几次 /bin/bash 中的 readline() 函数。这个例子在下面的章节中仍会继续介绍。

通过虚拟地址来指定探测点也是可以的，就像这样：
```
# objdump -tT /bin/bash | grep main
...
000000000002ec00 g    DF .text  0000000000001868  Base        main
...
# bpftrace -e 'uprobe:/bin/bash:0x2ec00 { printf("in here\n"); }'
Attaching 1 probe...
```

在探测器的函数内部也可以指定被探测函数的偏移量：
```
# objdump -d /bin/bash
...
000000000002ec00 <main@@Base>:
   2ec00:       f3 0f 1e fa             endbr64
   2ec04:       41 57                   push   %r15
   2ec06:       41 56                   push   %r14
   2ec08:       41 55                   push   %r13
   ...
# bpftrace -e 'uprobe:/bin/bash:main+4 { printf("in here\n"); }'
Attaching 1 probe...
...
```
偏移的地址会被检查是否与指令对齐。如果没有对齐，探测器会添加失败。

```
# bpftrace -e 'uprobe:/bin/bash:main+1 { printf("in here\n"); }'
Attaching 1 probe...
Could not add uprobe into middle of instruction: /bin/bash:main+1
```

如果 bpftrace 指定了 ALLOW_UNSAFE_PROBE 选项被编译，你可以使用 --unsafe 选项来跳过检查
```
# bpftrace -e 'uprobe:/bin/bash:main+1 { printf("in here\n"); } --unsafe'
Attaching 1 probe...
Unsafe uprobe in the middle of the instruction: /bin/bash:main+1
```

使用 --unsafe 选项你也可以探测任意的地址。在二进制程序被精简后会很有帮助
```
$ echo 'int main(){return 0;}' | gcc -xc -o bin -
$ nm bin | grep main
...
0000000000001119 T main
...
$ strip bin
# bpftrace --unsafe -e 'uprobe:bin:0x1119 { printf("main called\n"); }'
Attaching 1 probe...
WARNING: could not determine instruction boundary for uprobe:bin:4377 (binary appears stripped). Misaligned probes can lead to tracee crashes!
```

当追踪库的时候，推荐指定库的名称来替代完成路径。具体路径会被 /etc/ld.so.cache 自动解析出来。

```
# bpftrace -e 'uprobe:libc:malloc { printf("Allocated %d bytes\n", arg0); }'
Allocated 4 bytes
...
```

原始示例：[]() []()

## 4. uprobe/uretprobe: 动态追踪用户级参数
语法：
```
uprobe: arg0, arg1, ..., argN
uretprobe: retval
```

可以通过这些变量的名字访问实参。arg0 是第一个参数，并只能被 uprobe 访问。 retval 是被探测函数的额返回值，并只能被 uretprobe 访问

示例：
```
# bpftrace -e 'uprobe:/bin/bash:readline { printf("arg0: %d\n", arg0); }'
Attaching 1 probe...
arg0: 19755784
arg0: 19755016
arg0: 19755784
^C
```
/bin/bash 中的 readline() 函数的第一个参数包含什么？我并不知道，我需要查看 bash 的源代码来找出它的参数是什么。

```
# bpftrace -e 'uprobe:/lib/x86_64-linux-gnu/libc-2.23.so:fopen { printf("fopen: %s\n", str(arg0)); }'
Attaching 1 probe...
fopen: /proc/filesystems
fopen: /usr/share/locale/locale.alias
fopen: /proc/self/mountinfo
^C
```

这个例子中我知道 libc 的 fopen() 函数的第一个参数是路径名称，所以我用 uprobe 去追踪它。请调整与你操作系统中的路径进行匹配（你的可能不是 lib-2.23.so）。一个 str() 函数被用来将 char 指针转换为字符串，这些在随后的章节会进行解释.

```
# bpftrace -e 'uretprobe:/bin/bash:readline { printf("readline: \"%s\"\n", str(retval)); }'
Attaching 1 probe...
readline: "echo hi"
readline: "ls -l"
readline: "date"
readline: "uname -r"
^C
```

回到 bash readline() 函数的示例：查看完源码后，我知道了返回值是一个读到的字符串。因此我可以使用 uretprobe 与 retval 变量来查看读到的字符串。
---

如果被追踪的二进制程序开启了 DWARF，uprobe 也可以通形参的名称访问

语法：
```
uprobe: args->NAME
```

参数可以通过解引用 args 来进行访问，并且可以通参数的名称进行访问。

函数的参数列表可以通过详细列表选项获得：
```
# bpftrace -lv 'uprobe:/bin/bash:rl_set_prompt'
uprobe:/bin/bash:rl_set_prompt
    const char* prompt
```

示例（需要为 /bin/bash 安装调试信息）：
```
# bpftrace -e 'uprobe:/bin/bash:rl_set_prompt { printf("prompt: %s\n", str(args->prompt)); }'
Attaching 1 probe...
prompt: [user@localhost ~]$
^C
```

原始示例 [xxx]()

## 5. tracepoint: 内核级的静态追踪

语法：```tracepoint:name```

这里使用是 tracepoint（内核的一种能力）
```
# bpftrace -e 'tracepoint:block:block_rq_insert { printf("block I/O created by %d\n", tid); }'
Attaching 1 probe...
block I/O created by 28922
block I/O created by 3949
block I/O created by 883
block I/O created by 28941
block I/O created by 28941
block I/O created by 28941
[...]
```

原始示例 [search/tools](https://github.com/iovisor/bpftrace/search?q=tracepoint%3A+path%3Atools&type=Code)

## 6. tracepoint: 静态追踪内核参数

示例：
```
# bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s %s\n", comm, str(args->filename)); }'
Attaching 1 probe...
irqbalance /proc/interrupts
irqbalance /proc/stat
snmpd /proc/diskstats
snmpd /proc/stat
snmpd /proc/vmstat
snmpd /proc/net/dev
[...]
```

除了 filename 成员，还可以打印 flags，mode 等。开始打印的在 common 后的成员，对 tracepoint 来说有特殊含义

原始示例：[search /tools](https://github.com/iovisor/bpftrace/search?q=tracepoint%3A+path%3Atools&type=Code)

## 7. usdt: 用户级静态追踪
语法：
```
usdt:binary_path:probe_name
usdt:binary_path:[probe_namespace]:probe_name
usdt:library_path:probe_name
usdt:library_path:[probe_namespace]:probe_name
```

如果 probe_name 在二进制程序中是唯一的 probe 的命名空间是可选的

示例：
```
# bpftrace -e 'usdt:/root/tick:loop { printf("hi\n"); }'
Attaching 1 probe...
hi
hi
hi
hi
hi
^C
```

探测器的命名空间可以被自动推导。如果二进制程序 /roo/tick 包含多个名称为 loop 的探测器（例如：tick:loop 与 tock:loop），将不会有探测器开始探测。这需要通过手工指定命名空间或者使用通配来解决。

```
# bpftrace -e 'usdt:/root/tick:loop { printf("hi\n"); }'
ERROR: namespace for usdt:/root/tick:loop not specified, matched 2 probes
INFO: please specify a unique namespace or use '*' to attach to all matched probes
No probes to attach

# bpftrace -e 'usdt:/root/tick:tock:loop { printf("hi\n"); }'
Attaching 1 probe...
hi
hi
^C

# bpftrace -e 'usdt:/root/tick:*:loop { printf("hi\n"); }'
Attaching 2 probes...
hi
hi
hi
hi
^C
```

bpftrace 也支持 USDT 信号。如果你的环境与 bpftrace 都支持 uprobe recounts，那么在探测器探测的时候所有的进程都将自动激活 USDT 信号（--usdt-file-activation 将失效）。你可以通过运行一下内容检查你的系统是否支持 uprobe reounts：
```
# bpftrace --info 2>&1 | grep "uprobe refcount"
  bcc bpf_attach_uprobe refcount: yes
  uprobe refcount (depends on Build:bcc bpf_attach_uprobe refcount): yes
```

如果你的系统不支持 uprobe refount 你可以通过传递 -p $PID 或者 --usdt-file-activation 激活信号。 --usdt-file-activation 通过 /proc 查找将 probe 的二进制文件映射到其地址空间的进程，然后尝试附加探针。注意，文件激活只发生一次(在附加时)。换句话说，如果在稍后的跟踪会话中生成了带有可执行文件的新进程，那么当前的跟踪会话将不会激活新进程。还要注意--usdt-file-activation 是根据文件路径匹配，这意味着，如果 bpftrace 从根主机运行，那么如果有进程从私有挂载名称空间或绑定挂载目录执行，事情可能不会像预期的那样工作。一种解决方法是在适当的名称空间(即容器)中运行bpftrace。