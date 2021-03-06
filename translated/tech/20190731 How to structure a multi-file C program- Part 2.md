[#]: collector: (lujun9972)
[#]: translator: (mengxinayan)
[#]: reviewer: ( )
[#]: publisher: ( )
[#]: url: ( )
[#]: subject: (How to structure a multi-file C program: Part 2)
[#]: via: (https://opensource.com/article/19/7/structure-multi-file-c-part-2)
[#]: author: (Erik O'Shaughnessy https://opensource.com/users/jnyjny)

如何组织构建多文件 C 语言程序（二）
======
我将在本系列的第二篇中深入研究由多个文件组成的C程序的结构。
![4 个 manilla 文件，黄色，绿色，紫色，蓝色][1]

在 [（第一篇）][2] 中，我设计了一个名为 [MeowMeow][3] 的多文件 C 程序，该程序实现了一个玩具——[codec][4]。我提到了程序设计中的 Unix 哲学，即在一开始创建多个空文件，并建立一个好的结构。最后，我创建了一个 Makefile 文件夹并阐述了它的作用。在本文中将另一个方向展开：现在我将介绍简单但具有指导性的 MeowMeow 编/解码器的实现。

当读过我的 "[如何写一个好的 C 语言 main 函数][5]." 后，你便会知道 `main.c` 文件中 `meow` 和 `unmeow` 的结构，其主体结构如下：

```
/* main.c - MeowMeow 流编码器和解码器 */

/* 00 system includes */
/* 01 project includes */
/* 02 externs */
/* 03 defines */
/* 04 typedefs */
/* 05 globals (but don't)*/
/* 06 ancillary function prototypes if any */
   
int main(int argc, char *argv[])
{
  /* 07 variable declarations */
  /* 08 check argv[0] to see how the program was invoked */
  /* 09 process the command line options from the user */
  /* 10 do the needful */
}
   
/* 11 ancillary functions if any */
```

### 包含项目头文件

位于第二部分中的 `/* 01 project includes */` 的源代码如下：


```
/* main.c - MeowMeow 流编码器和解码器 */
...
/* 01 project includes */
#include "main.h"
#include "mmecode.h"
#include "mmdecode.h"
```

`#include` 是 C 语言的预处理命令，它会将其后面的文件内容拷贝到当前文件中。如果程序员在头文件名称周围使用双引号，编译器将会在当前目录寻找该文件。如果文件被尖括号包围，编译器将在一组预定义的目录中查找该文件。

[main.h][6] 文件中包含了 [main.c][7] 文件中用到的定义和别名。我喜欢在头文件里尽可能多的声明，以便我想在我的程序的其他位置使用这些定义。

头文件 [mmencode.h][8] 和 [mmdecode.h][9] 几乎相同，因此我以 `mmencode.h` 为例来分析。


```
 /* mmencode.h - MeowMeow 流编码器和解码器 */
   
 #ifndef _MMENCODE_H
 #define _MMENCODE_H
   
 #include &lt;stdio.h&gt;
   
 int mm_encode(FILE *src, FILE *dst);
   
 #endif /* _MMENCODE_H */
```

`#ifdef`，`#define`，`#endif` 指令统称为 “防护” 指令。其可以防止 C 编译器在一个文件中多次包含同一文件。如果编译器在一个文件中发现多个定义/原型/声明，它将会产生警告。因此这些防护措施是必要的。

在防护内部，它只做两件事：`#include` 指令和函数原型声明。我将包含 `stdio.h` 头文件，以便于能在函数原型中使用 `FILE` 流。函数原型也可以被包含在其他 C 文件中，以便于在文件的命名空间中创建它。你可以将每个文件视为一个命名空间，其中的变量和函数不能被另一个文件中的函数或者变量使用。

编写头文件很复杂，并且在大型项目中很难管理它。不要忘记使用防护。

### MeowMeow 编码的实现

该程序的功能是按照字节进行 `MeowMeow` 字符串的编解码，事实上这是该项目中最简单的部分。截止目前我所做的工作便是支持允许在适当的位置调用此函数：解析命令行，确定要使用的操作，并打开将要操作的文件。下面的循环是编码的过程：


```
 /* mmencode.c - MeowMeow 流编码器 */
 ...
     while (![feof][10](src)) {
   
       if (![fgets][11](buf, sizeof(buf), src))
         break;
             
       for(i=0; i&lt;[strlen][12](buf); i++) {
         lo = (buf[i] &amp; 0x000f);
         hi = (buf[i] &amp; 0x00f0) &gt;&gt; 4;
         [fputs][13](tbl[hi], dst);
         [fputs][13](tbl[lo], dst);
       }
            }
```

简单的说，上面代码循环读取文件的一部分，剩下的内容通过 `feof(3)` 函数和 `fgets(3)` 函数读取。然后将读入的内容的每个字节分成 `hi` 和 `lo` 半个字节。半个字节是 4 个比特。这里的奥妙之处在于可以用 4 个比特来编码16个值。我将字符串 `hi` 和 `lo` 用作 16 字符串查找表 `tbl` 的索引，表中包含了对每半个字节编码后的 `MeowMeow` 字符串。这些字符串使用 `fputs(3)` 函数写入目标 `FILE` 流，然后移动到缓存区的下一个字节。

该表使用 [table.h][14] 中的宏定义进行初始化，在没有特殊原因（比如：要展示包含了另一个项目的本地头文件）时，我喜欢使用宏来进行初始化。我将在未来的文章中进一步探讨原因。

### MeowMeow 解码的实现

我承认在开始工作前花了一些时间。解码的循环与编码类似：读取 `MeowMeow` 字符串到缓冲区，将编码从字符串转换为字节


```
 /* mmdecode.c - MeowMeow 流解码器 */
 ...
 int mm_decode(FILE *src, FILE *dst)
 {
   if (!src || !dst) {
     errno = EINVAL;
     return -1;
   }
   return stupid_decode(src, dst);
 }
```

这不是所期望的吗？

在这里，我通过外部公开的 `mm_decode()` 函数公开了 `stupid_decode()` 函数细节。我上面所说的“外部”是指在这个文件之外。因为 `stupid_decode()` 函数不在头文件中，因此无法在其他文件中调用它。

当我们想发布一个可靠的公共接口时，有时候会这样做，但是我们还没有完全使用函数解决问题。在本例中，我编写了一个 I/O 密集型函数，该函数每次从源中读取 8 个字节，然后解码获得 1 个字节写入目标流中。较好的实现是一次处理多于 8 个字节的缓冲区。更好的实现还可以通过缓冲区输出字节，进而减少目标流中单字节的写入次数。


```
 /* mmdecode.c - MeowMeow 流解码器 */
 ...
 int stupid_decode(FILE *src, FILE *dst)
 {
   char           buf[9];
   decoded_byte_t byte;
   int            i;
     
   while (![feof][10](src)) {
     if (![fgets][11](buf, sizeof(buf), src))
       break;
     byte.field.f0 = [isupper][15](buf[0]);
     byte.field.f1 = [isupper][15](buf[1]);
     byte.field.f2 = [isupper][15](buf[2]);
     byte.field.f3 = [isupper][15](buf[3]);
     byte.field.f4 = [isupper][15](buf[4]);
     byte.field.f5 = [isupper][15](buf[5]);
     byte.field.f6 = [isupper][15](buf[6]);
     byte.field.f7 = [isupper][15](buf[7]);
       
     [fputc][16](byte.value, dst);
   }
   return 0;
 }
```

我并没有使用编码器中使用的位移方法，而是创建了一个名为 `decoded_byte_t` 的自定义数据结构。

```
 /* mmdecode.c - MeowMeow 流解码器 */
 ...

 typedef struct {
   unsigned char f7:1;
   unsigned char f6:1;
   unsigned char f5:1;
   unsigned char f4:1;
   unsigned char f3:1;
   unsigned char f2:1;
   unsigned char f1:1;
   unsigned char f0:1;
 } fields_t;
   
 typedef union {
   fields_t      field;
   unsigned char value;
 } decoded_byte_t;
```

初次看到代码时可能会感到有点儿复杂，但不要放弃。`decoded_byte_t` 被定义为 `fields_t` 和 `unsigned char` 的 **联合体** 。可以将联合中的命名成员看作同一内存区域的别名。在这种情况下，`value` 和 `field` 指向相同的 8 比特内存区域。将 `field.f0` 设置为 1 也将设置 `value` 中的最低有效位。

虽然 `unsigned char` 并不神秘，但是对 `fields_t` 的 别名(`typedef`) 也许看起来有些陌生。现代 C 编译器允许程序员在结构体中指定单个 bit 的值。`field` 成员是一个无符号整数类型，可以在成员标识符后紧跟一个冒号和一个整数，该整数指定了比特字段的长度。

这种数据结构使得按 `field` 名称访问每个比特变得简单。我们依赖编译器生成正确的移位指令来访问 `field`，这可以在调试时为你节省不少时间。

最后，因为`stupid_decode()` 函数一次仅从源 `FILE` 流中读取 8 个字节，所以它效率并不高。通常我们尝试最小化读写次数，以提高性能和降低调用系统调用的开销。请记住：少次数的读取/写入大的块比多次数的读取/写入小的块好得多。

### 总结

用 C 语言编写一个多文件程序需要程序员做更多计划，而不仅仅是一个 `main.c`。但是当你添加功能或者重构时，只需要多花费一点儿努力便可以节省大量时间以及避免让你头痛的问题。

回顾一下，我更喜欢这样做：多个文件，每个文件仅有简单功能；通过头文件公开那些文件中的小部分功能；把数字常量和字符串常量保存在头文件中；使用 `Makefile` 而不是 Bash 脚本来自动化处理事务；使用 `main()` 函数来处理和解析命令行参数并作为程序主要功能的框架。

我知道我只是涉及了这个简单程序中发生的事情，并且我很高兴知道哪些事情对您有所帮助以及哪些主题需要详细的解释。在评论中分享您的想法，让我知道。

--------------------------------------------------------------------------------

via: https://opensource.com/article/19/7/structure-multi-file-c-part-2

作者：[Erik O'Shaughnessy][a]
选题：[lujun9972][b]
译者：[萌新阿岩](https://github.com/mengxinayan)
校对：[校对者ID](https://github.com/校对者ID)

本文由 [LCTT](https://github.com/LCTT/TranslateProject) 原创编译，[Linux中国](https://linux.cn/) 荣誉推出

[a]: https://opensource.com/users/jnyjny
[b]: https://github.com/lujun9972
[1]: https://opensource.com/sites/default/files/styles/image-full-size/public/lead-images/file_system.jpg?itok=pzCrX1Kc (4 manilla folders, yellow, green, purple, blue)
[2]: https://opensource.com/article/19/7/how-structure-multi-file-c-program-part-1
[3]: https://github.com/jnyjny/MeowMeow.git
[4]: https://en.wikipedia.org/wiki/Codec
[5]: https://opensource.com/article/19/5/how-write-good-c-main-function
[6]: https://github.com/JnyJny/meowmeow/blob/master/main.h
[7]: https://github.com/JnyJny/meowmeow/blob/master/main.c
[8]: https://github.com/JnyJny/meowmeow/blob/master/mmencode.h
[9]: https://github.com/JnyJny/meowmeow/blob/master/mmdecode.h
[10]: http://www.opengroup.org/onlinepubs/009695399/functions/feof.html
[11]: http://www.opengroup.org/onlinepubs/009695399/functions/fgets.html
[12]: http://www.opengroup.org/onlinepubs/009695399/functions/strlen.html
[13]: http://www.opengroup.org/onlinepubs/009695399/functions/fputs.html
[14]: https://github.com/JnyJny/meowmeow/blob/master/table.h
[15]: http://www.opengroup.org/onlinepubs/009695399/functions/isupper.html
[16]: http://www.opengroup.org/onlinepubs/009695399/functions/fputc.html
