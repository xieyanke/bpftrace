# bpftrace 参考指南

这是一个不断进展的项目，如果缺少了某些内容，请查看 bpftrace 的源码，看看是否在那里

如果你发现某些内容过时了，请提交一个问题或者一个更新文档的 PR。


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
