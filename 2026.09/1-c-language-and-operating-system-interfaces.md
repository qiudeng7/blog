# 从 C 语言到操作系统接口

上一篇对 MaaS 网关的性能调研，让我开始对 Linux system call 和 C 语言产生兴趣。

我首先注意到一个问题：Linux 内核是不是原生只提供了 C 接口？进一步说，我们平时使用的各种编程语言，究竟是如何调用 Linux 和 Windows 提供的操作系统 API 的？

作为一个原生的 TypeScript 和 Python 开发者，我大多数时候不需要区分语言能力与系统能力，也很少直接考虑操作系统之间的差异。读写文件、创建进程、建立网络连接，看起来都只是标准库或第三方库提供的普通函数。库在背后抹平了平台差异，我看到的是一套相对统一的语言接口。

但是 C 似乎不太一样。C 和操作系统的边界更近，也因此迫使我把过去混在一起理解的几个概念重新拆开：哪些能力属于 C 语言本身，哪些来自 C 标准库，哪些由操作系统提供，又有哪些只是某个编译器或运行时的实现？

如果完全没有操作系统接口，C 本身还剩下哪些标准能力，又能做什么？当程序需要文件、网络、线程、进程和虚拟内存时，C 是怎样进入 Linux system call 或 Windows API 的？同一份 C 程序面对两个操作系统时，哪些部分能够保持不变，哪些部分必须分别实现？

内存是另一个让我困惑的地方。我听说 C 的内存模型、Linux 的内存模型和 Windows 的内存模型并不是同一件事。C 的内存模型描述的是什么？进程地址空间、虚拟内存、分页和内存映射又属于哪一层？语言中的指针与操作系统管理的虚拟地址之间，到底是什么关系？

这些问题最后指向了一条新的学习路径：作为 C 开发者，学完语法、类型、指针和标准库之后，应该怎样继续了解系统能力？是否需要把 Linux 接口作为一套独立知识来学习？如果需要，这套知识的边界在哪里，又应该从 system call、POSIX 还是 Linux 编程接口开始？Windows 下是否存在一条对应的路径？

我还希望反过来理解 TypeScript 和 Python 中那些习以为常的能力是怎样出现的：如何用 C 封装操作系统能力暴露给其他语言调用？当跨平台库替我们隐藏差异时，它具体隐藏了什么？

我预想的路径是先学足够使用的 C，然后再接触不同的操作系统和高级语言相关的知识。

## 学习足够使用的 C

问了一下AI，对于我的目的，足够使用的C知识大概包括：

- 基本类型、整数转换
- 数组、字符串和指针
- struct、enum、union
- 栈、堆、静态存储期
- malloc、free
- 头文件和多文件编译
- 预处理器
- 函数指针
- 错误处理
- 编译、汇编、链接的大致过程

我们直接让AI给出代码示例来学习。

### 编译流程和基本类型

下面是本节的参考代码，本节围绕这段代码展开说明 C 的编译流程、基本类型和类型转换。

```c
#include <inttypes.h>
#include <stdint.h>
#include <stdio.h>

int main(void) {
    int32_t signed_value = -1;
    uint32_t unsigned_value = (uint32_t)signed_value;
    double ratio = 3.75;
    int truncated = (int)ratio;

    printf("signed = %" PRId32 "\n", signed_value);
    printf("unsigned = %" PRIu32 "\n", unsigned_value);
    printf("truncated = %d\n", truncated);
    return 0;
}
```

#### 预处理

对于参考代码中的 `#include`：
- `#`表示这是预处理指令，预处理是在正式编译之前的处理。
- `#include` 指令用于加载头文件，`.h`文件被称为头文件。

常见的预处理指令除了 include 还有宏 `#define` 和条件编译 `#if`, `#endif`，预处理器在遇到预处理指令的时候会做一些简单的操作。

想象一个类似 jinja，或者 go 的模板语言，他们都会有一种叫做“模板渲染”的过程，即把模板文本中的某些变量值处理成实际值，预处理器的工作大致可以这样理解。

`#include` 可以理解为预处理器执行的“文件插入”操作。它会读取指定的头文件，把头文件中的内容展开到当前源文件中，然后再继续处理展开出来的内容：

```c
// config.h
#define BUFFER_SIZE 1024
```

```c
// main.c
#include "config.h"

char buffer[BUFFER_SIZE];
```

预处理器处理 `main.c` 时，会把 `config.h` 的内容插入 `#include` 所在的位置，因此后续代码就可以使用头文件中声明的函数、类型和宏。头文件还可能继续 `#include` 其他头文件，所以这个过程可以递归发生。

尖括号通常用于系统或编译器提供的头文件，双引号通常用于当前项目中的头文件。

include并不完全是简单的复制粘贴，还有很多其他情况需要处理，比如说头文件中可能会有其他的宏、条件编译等预处理指令，以及还有重复 include、循环 include 等各种边界情况要处理。


头文件通常使用 `.h` 作为文件扩展名，用来存放可以被多个源文件共享的声明，例如函数声明、类型定义、宏和常量声明：

C 标准库也提供了一些可以直接使用的头文件。例如，`stdio.h`（standard input/output）声明了输入输出相关的函数和类型，`printf` 就来自这个头文件：

```c
#include <stdio.h>

int main(void) {
    printf("hello\n");
    return 0;
}
```

其他常见的标准库头文件包括：

- `stdlib.h`：动态内存分配、程序退出、数字转换等；
- `string.h`：字符串和内存区域操作，例如 `strlen`、`memcpy`；
- `stdint.h`：具有明确宽度的整数类型，例如 `int32_t`；
- `stddef.h`：`size_t`、`ptrdiff_t`、`NULL` 等基础定义；
- `errno.h`：错误码 `errno` 及相关定义。

使用尖括号的 `#include <stdio.h>` 通常表示从编译器和系统配置的头文件目录中查找；使用双引号的 `#include "calculator.h"` 通常表示优先从当前项目中查找。这是一种常见的查找约定，具体搜索路径还取决于编译器参数和构建环境。

```c
// calculator.h
#ifndef CALCULATOR_H
#define CALCULATOR_H

int add(int left, int right);

#endif
```

源文件通过 `#include "calculator.h"` 引入这些声明，就可以调用其他源文件中定义的 `add` 函数。头文件一般不直接生成独立的可执行程序，而是被预处理器展开到引用它的每个源文件中。`#ifndef`、`#define` 和 `#endif` 组成的 include guard，可以防止同一个头文件在一次编译中被重复展开。

头文件和C源文件，一般只是从规范上来讲，一个用来写声明，一个用来写实现，但实际语法上是没有区别的。

宏可以理解为预处理器执行的文本替换规则。`#define BUFFER_SIZE 1024` 定义的是一个没有参数的宏：预处理器遇到 `BUFFER_SIZE` 时，会把它替换成 `1024`，然后编译器再处理替换后的代码。

宏也可以定义参数，写法是在宏名后面紧跟括号和参数名。使用时虽然也写成类似函数调用的形式，但它实际进行的是代码片段替换。一个实际使用场景是计算数组中有多少个元素：

```c
#define ARRAY_COUNT(array) (sizeof(array) / sizeof((array)[0]))

int values[] = {10, 20, 30};
size_t count = ARRAY_COUNT(values);  // 预处理后大致为 sizeof(values) / sizeof((values)[0])
```

这里的 `array` 是宏参数，不是函数参数；`ARRAY_COUNT(values)` 也不是一次函数调用，而是在编译前展开成一段表达式，传入 `i++` 这类带副作用的表达式可能被求值多次。


条件编译让预处理器根据某个宏是否定义，决定一段代码是否交给编译器处理。例如，可以用它控制调试日志是否启用：

```c
#include <stdio.h>

#if defined(DEBUG)
#define LOG(message) fprintf(stderr, "debug: %s\n", (message))
#else
#define LOG(message) ((void)0)
#endif

int main(void) {
    LOG("program started");
    return 0;
}
```

没有定义 `DEBUG` 时，`LOG("program started")` 会被替换为空操作；使用 `cc -DDEBUG main.c` 编译时，`LOG` 才会展开为输出调试信息的代码。`#if` 和 `#endif` 标记条件代码块，`#else` 提供条件不成立时的另一条分支；也可以使用 `#ifdef` 和 `#ifndef` 判断一个宏是否已经定义。条件编译常用于调试日志、平台差异和可选功能。

#### 基本类型和转换

C 提供了 `char`、`short`、`int`、`long`、浮点数等基本类型，但标准通常只规定它们的最小范围和相对大小，不保证 `int` 在所有平台上都是固定宽度。`stdint.h` 中的 `int32_t` 和 `uint32_t` 适合需要明确位宽的场景。

把 `-1` 转换为 `uint32_t` 时，结果会对 `2^32` 取模，因此通常得到 `4294967295`。把 `3.75` 转为 `int` 时，小数部分会向零截断。显式转换并不代表转换一定安全：如果一个浮点数超出了目标整数类型能表示的范围，行为可能是未定义的。编译时开启 `-Wconversion` 可以帮助发现许多意外转换。

#### 配置编译

C 代码的编译流程如下

```text
main.c
  ↓ 预处理
预处理后的 C 源代码
  ↓ 编译
汇编代码 main.s
  ↓ 汇编
目标文件 main.o
  ↓ 链接
可执行文件 main
```

第一步是预处理。预处理器展开 `#include`、宏和条件编译，生成预处理后的 C 源代码。第二步是编译，编译器读取这份 C 源代码，检查语法、类型和语义，并把它转换成当前目标平台的汇编代码。第三步是汇编，汇编器把汇编代码转换成包含机器指令的目标文件。最后是链接，链接器把一个或多个目标文件以及库组合起来，解析函数和变量等符号引用，生成动态链接或者静态链接的二进制文件，生成最终的可执行文件。

严格来说，第二步才是“编译器”进行的编译；但在日常交流中，人们也常把预处理、编译、汇编和链接这一整套过程统称为编译。

可以先看一条让 `cc` 自动完成全部阶段、直接生成可执行文件的命令：

```bash
cc -std=c17 -Wall -Wextra -Wpedantic -g -O0 main.c -o main
```

这里的 `cc` 是系统提供的通用 C 编译器命令，也是一个编译器驱动程序。它会依次调用预处理器、编译器、汇编器和链接器。`cc` 并不是某个具体编译器的品牌：在不同环境中，它背后可能是 GCC，也可能是 Clang。可以用下面的命令查看当前实际使用的实现：

```bash
cc --version
```

常见的 C 编译器主要有：

- GCC：GNU Compiler Collection 中的 C 编译器，在 Linux 和许多类 Unix 系统中很常见，具体命令通常是 `gcc`；
- Clang：LLVM 项目的 C 编译器前端，诊断信息清晰，在 macOS 和许多现代开发环境中很常见，具体命令通常是 `clang`；
- MSVC：Microsoft 提供的 C/C++ 工具链，主要用于 Windows，编译器命令是 `cl`。

C 语言之所以可以有多个编译器，是因为 C 标准主要规定源代码应该具有怎样的含义，并不要求只能由某一个程序实现。不同编译器可以面向不同的操作系统和处理器，也会在优化能力、错误提示、编译速度、平台工具链和扩展功能上作出不同选择。

刚开始学习时，通常直接使用系统默认的 `cc` 就可以。在 Linux 上选择 GCC 或 Clang 都很合适；在 macOS 上通常直接使用系统提供的 Clang；主要开发 Windows 原生程序时可以使用 MSVC。如果项目已经提供 Makefile、CMake 配置或明确指定了编译器，应优先跟随项目的工具链。学习到需要关注可移植性时，还可以同时使用 GCC 和 Clang 编译，让两套诊断帮助发现问题。

回到上面的命令，`-o main` 指定最终输出文件名；其他参数会被 `cc` 驱动程序传递给对应的阶段。

预处理阶段处理头文件、宏和条件编译。`-D` 定义宏，`-I` 增加头文件搜索目录，`-E` 让流程停在预处理之后：

```bash
cc -DDEBUG -Iinclude -E main.c -o main.i
```

这条命令会生成预处理后的 C 源文件 `main.i`。例如，`-DDEBUG` 可以让源代码中的 `#if defined(DEBUG)` 条件成立，`-Iinclude` 则让 `#include "project.h"` 能够在 `include` 目录中找到头文件。

编译器阶段把预处理后的 C 代码翻译成汇编代码。`-std=c17` 指定使用的 C 标准，`-Wall`、`-Wextra`、`-Wpedantic` 和 `-Wconversion` 控制警告，`-S` 让流程停在汇编代码：

```bash
cc -std=c17 -Wall -Wextra -Wpedantic -Wconversion -S main.c -o main.s
```

`-Wall` 并不代表“所有警告”，通常会和 `-Wextra` 一起使用。开发 C 程序时尽早处理这些警告，可以发现未使用变量、类型转换、格式化字符串和控制流等问题。

汇编器把汇编代码转换为目标文件。`-c` 表示只生成目标文件，不进行链接：

```bash
cc -std=c17 -Wall -Wextra -c main.c -o main.o
```

目标文件通常以 `.o` 结尾，里面包含机器指令以及等待链接器处理的符号和重定位信息。多个 `.c` 文件可以分别使用 `-c` 编译，最后再统一链接。

链接阶段把目标文件和库组合成可执行文件。`-L` 增加库搜索目录，`-l` 指定要链接的库：

```bash
cc main.o -Llib -ltools -o main
```

`-ltools` 通常表示查找名为 `libtools.so` 或 `libtools.a` 的库。像 `printf` 这样的标准库函数，也需要在链接阶段找到对应的实现。

调试和测试参数可能同时影响编译、链接和运行时检查。`-g` 生成调试信息，`-O0` 通常便于调试，`-O2` 通常用于更接近发布版本的优化：

```bash
cc -std=c17 -Wall -Wextra -g -O0 main.c -o main
cc -std=c17 -Wall -Wextra -O2 main.c -o main
```

##### 编译安全约束

C 编译器会检查语法和类型，但 C 的类型系统不会自动表达数组长度、指针生命周期和内存所有权，也不会在运行时自动检查数组边界。工程中通常会组合多层工具，对代码施加比语言最低要求更严格的安全约束。

第一层是编译器诊断。`-Wall`、`-Wextra`、`-Wpedantic`、`-Wconversion`、`-Wshadow` 和 `-Wformat=2` 等参数可以发现可疑转换、未初始化变量、格式化字符串错误和容易误解的代码。项目也可以在持续集成中使用 `-Werror`，把警告当作编译错误；不过升级编译器可能带来新的警告，因此是否启用需要由项目统一决定。

```bash
cc -std=c17 -Wall -Wextra -Wpedantic \
    -Wconversion -Wshadow -Wformat=2 \
    main.c -o main
```

第二层是 Lint 和静态分析。这类工具不运行程序，而是分析语法树、控制流和函数之间的数据传递：

- `clang-tidy` 和 Cppcheck 类似 TypeScript 项目中的 ESLint，用来检查容易出错的写法、编码规则和可维护性问题；
- Clang Static Analyzer 可以通过 `clang --analyze` 或 `scan-build` 分析可能发生错误的执行路径；
- GCC 的 `-fanalyzer` 可以检查空指针解引用、重复释放、资源泄漏等跨语句问题；
- Coverity、PVS-Studio 和 CodeQL 等工具适合进行更大规模的代码库与安全分析。

```bash
clang-tidy main.c --checks='bugprone-*,cert-*,clang-analyzer-*' --
clang --analyze main.c
gcc -fanalyzer main.c
```

静态分析可以检查没有实际运行到的代码，但普通 C 类型往往没有携带足够的长度和生命周期信息。分析器需要在“报告所有无法证明安全的代码”和“减少误报”之间作出取舍；更严格的项目还会通过函数契约、注解以及 MISRA C、CERT C 等编码规则补充这些信息。

第三层是 Sanitizer。它是一类由编译器和配套运行库共同提供的动态检查工具。编译器会向程序插入检查逻辑，程序运行时再根据真实的指针、输入和线程状态发现问题：

- AddressSanitizer（ASan）：检查越界访问、use-after-free 等内存错误；
- UndefinedBehaviorSanitizer（UBSan）：检查一部分未定义行为；
- ThreadSanitizer（TSan）：检查多线程程序中的数据竞争；
- MemorySanitizer（MSan）：检查对未初始化内存的读取。

```bash
cc -std=c17 -Wall -Wextra -g \
    -fsanitize=address,undefined main.c -o main
```

Sanitizer 只能检查实际运行到的路径，所以通常要和单元测试、集成测试以及 Fuzzing 配合使用。它会增加程序的运行时间和内存开销，因此主要用于本地开发、自动化测试和排查问题，通常不会直接用于正式发布版本。

对安全要求更高的代码，还可以使用 Frama-C、CBMC 等工具进行契约检查、模型检查或形式化验证。这些方法能够提供更强的静态保证，但通常需要限制代码写法、添加注解并投入更多验证成本。

因此，C 的安全保障不是依赖某一个开关，而是一组互补措施：编译器诊断负责发现明显问题，Lint 和静态分析检查潜在路径，Sanitizer 验证真实运行状态，测试和 Fuzzing 扩大被执行的输入与路径，更严格的规则和形式化工具则用于关键代码。

当然，写这么多的安全约束会增加很多工程上的复杂性，所以很多人选择不如直接去使用rust，rust的编译器原生提供了强大的安全约束，我也决定后续再去学一下rust，但我仍然打算继续走完从C到操作系统这条路，就像别人说越接近钱才能赚到钱，技术也是一样，越接近底层才能够学到技术。

### 数组、字符串和指针

本节示例代码如下

```c
#include <stddef.h>
#include <stdio.h>

int main(void) {
    int numbers[] = {10, 20, 30};
    int *cursor = numbers;
    char message[] = "hello";

    for (size_t i = 0; i < sizeof numbers / sizeof numbers[0]; i++) {
        printf("%d\n", *(cursor + i));
    }

    message[0] = 'H';
    printf("%s, length = %zu\n", message, sizeof message - 1);
    return 0;
}
```

#### `stddef.h` 和 `size_t`

这一段代码比前面的示例多包含了一个 `stddef.h`。它是 C 标准库提供的基础头文件，定义了一些与对象大小、地址差值和内存布局有关的通用类型与宏，例如 `size_t`、`ptrdiff_t`、`NULL` 和 `offsetof`。

示例中的循环变量 `i` 使用了 `size_t`。`size_t` 是一种无符号整数类型，用来表示对象占用的字节数，也是 `sizeof` 运算符返回结果的类型。它的具体宽度取决于目标平台，因此不应直接假设它就是 `unsigned int` 或 `unsigned long`。`printf` 使用 `%zu` 输出 `size_t`。

虽然 `stdio.h` 等其他标准头文件也可能提供 `size_t` 的定义，但代码直接使用与对象大小相关的基础类型时，显式包含 `stddef.h` 可以更清楚地表达依赖。

#### 数组

数组由一组类型相同、在内存中连续存放的元素组成。`int numbers[] = {10, 20, 30}` 创建了一个包含三个 `int` 元素的数组，数组长度由初始化值推断出来。`sizeof numbers` 得到整个数组占用的字节数，`sizeof numbers[0]` 得到一个元素占用的字节数，因此两者相除可以算出数组元素数量。

数组的长度不会随使用过程自动增长，C 也不会自动检查下标是否越界。只有下标 `0`、`1` 和 `2` 对这个数组有效；访问 `numbers[3]` 会越过数组边界，产生未定义行为，这类行为可以用前文提到的安全约束去进行规范。


#### 指针

在 C 中，变量可以保存值。这个值可以是 `10` 这样的具体数据，也可以是另一个对象的位置。如果一个变量保存的是对象位置，那么它就是一个指针变量。

```c
int value = 10;
int *pointer = &value;

printf("%d\n", *pointer);
*pointer = 20;
```

这里存在两个变量：`value` 保存的值是整数 `10`，`pointer` 保存的值则是 `value` 的地址。指针也是值的一种，只是这个值具有“指向另一个对象”的特殊含义。

```text
变量       类型       保存的值
value      int        10
pointer    int *      value 的地址
```

`int *` 是一种类型，表示“指向 `int` 的指针”。指针并不是一个没有类型约束的地址：它的类型告诉编译器目标对象应该被解释为什么类型、解引用时应该访问什么数据，以及进行指针运算时一次应该移动多远。

`&` 用来取得对象的地址，因此 `&value` 的类型是 `int *`。`*` 用来沿着指针访问它所指向的对象，这个操作称为解引用：

```c
pointer;   // pointer 保存的地址
*pointer;  // 该地址处的 int，也就是 value
&pointer;  // pointer 变量自身的地址，类型是 int **
```

读取 `*pointer` 得到 `value` 当前保存的值，给 `*pointer` 赋值则会修改同一个对象。因此执行 `*pointer = 20` 后，`value` 也会变成 `20`，这里并没有复制出另一个整数对象。

回到参考代码，`int *cursor = numbers` 把数组首元素的位置保存到 `cursor` 中。在大多数表达式里，数组名 `numbers` 会转换为指向首元素 `numbers[0]` 的指针，因此这行代码也可以写成：

```c
int *cursor = &numbers[0];
```

这两种写法的意思都是“`cursor` 指向 `numbers` 的第一个元素”，而不是“`cursor` 保存了一个与第一个元素相同的值”。可以通过下面两组关系区分地址和值：

```c
cursor == &numbers[0];   // 两边都是第一个元素的地址
*cursor == numbers[0];  // 两边都是第一个元素保存的值
```

下面这种写法则不是指向数组第一位：

```c
int *cursor = numbers[0]; // 错误：numbers[0] 是 int，不是 int *
```

`numbers[0]` 得到的是第一个元素保存的整数 `10`，但 `cursor` 需要的是一个 `int *` 指针值。这行代码相当于试图把整数 `10` 当作地址。编译器通常会给出类型不匹配警告，之后解引用这个指针会产生未定义行为。

指针可以在同一个数组范围内做加减运算。`cursor + i` 不是简单地把地址增加 `i` 个字节，而是向后移动 `i` 个 `int` 元素；`*(cursor + i)` 再解引用对应元素。因此：

```c
numbers[i]
*(numbers + i)
*(cursor + i)
```

在这个例子中访问的是同一个元素。指针最多可以计算到数组末尾之后的位置，但不能解引用这个位置；也不能随意跨越到另一个无关对象。数组和指针仍然不是同一种类型，例如 `sizeof numbers` 得到整个数组的大小，而 `sizeof cursor` 得到指针自身的大小。

所以，目前可以先把指针理解为一种有类型约束的特殊值：它表示另一个对象的位置。但是一个数值看起来像地址，并不代表它就是可以安全使用的指针。只有目标对象确实存在、仍在生命周期内、允许访问，并且访问方式符合指针类型时，才能进行解引用。未初始化的指针、空指针、指向已经释放对象的指针以及越界指针都不能随意解引用。

#### 字符串

C 字符串是以 `\0` 结尾的字符数组。`"hello"` 实际包含 6 个字符，最后一个是终止符。这里使用 `char message[]` 得到可修改的数组；若写成 `char *message = "hello"`，指针会指向字符串字面量，再尝试修改字符串字面量会产生未定义行为。

标准字符串函数依靠 `\0` 判断字符串在哪里结束，并不会额外保存长度。如果字符数组缺少终止符，或者函数在遇到终止符前已经越过数组边界，程序就会访问无效内存。因此，处理字符串时不仅要记录可用空间，还要为结尾的 `\0` 留出一个字符的位置。

### `struct`、`enum` 和 `union`

```c
#include <stdio.h>

enum EventType {
    EVENT_CONNECTED,
    EVENT_DATA,
    EVENT_CLOSED
};

struct Event {
    enum EventType type;
    union {
        int socket_fd;
        struct {
            const char *bytes;
            unsigned long length;
        } data;
    } payload;
};

int main(void) {
    struct Event event = {
        .type = EVENT_DATA,
        .payload.data = {.bytes = "hello", .length = 5}
    };

    if (event.type == EVENT_DATA) {
        printf("%.*s\n", (int)event.payload.data.length, event.payload.data.bytes);
    }
    return 0;
}
```

`struct` 把多个字段组织成一个对象，每个字段都有自己的存储空间。编译器可能为了满足对齐要求在字段之间插入 padding，所以结构体大小不一定等于所有字段大小之和，也不能想当然地把它的内存布局当成网络协议格式。

`enum` 为一组整数常量提供名称。`union` 的多个成员共享同一段存储，同一时刻通常只应读取最近写入的那个成员。这个例子用 `type` 记录 `payload` 当前有效的成员，组成一个简单的 tagged union；它类似 TypeScript 中通过判别字段实现的联合类型，但 C 编译器不会替我们保证 `type` 和成员一定匹配。

### 栈、堆和静态存储期

```c
#include <stdio.h>
#include <stdlib.h>

static int process_total;

void count_request(void) {
    int current_request = ++process_total;
    int *result = malloc(sizeof *result);

    if (result == NULL) {
        return;
    }

    *result = current_request * 2;
    printf("%d\n", *result);
    free(result);
}

int main(void) {
    count_request();
    return 0;
}
```

这里有三种不同的生命周期。文件作用域的 `process_total` 具有静态存储期，在整个程序运行期间都存在；`current_request` 具有自动存储期，进入函数时创建，离开函数后失效；`result` 指向动态分配的对象，该对象一直存在到程序调用 `free` 为止。

“自动对象通常在栈上、动态对象通常在堆上”是常见实现方式，但 C 标准描述的是存储期，并不要求实现一定使用名为栈和堆的数据结构。指针变量 `result` 本身是自动对象，它指向的 `int` 才是动态分配的对象。

### `malloc` 和 `free`

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    size_t count = 4;
    int *values = malloc(count * sizeof *values);

    if (values == NULL) {
        fprintf(stderr, "allocation failed\n");
        return EXIT_FAILURE;
    }

    for (size_t i = 0; i < count; i++) {
        values[i] = (int)(i * i);
    }

    for (size_t i = 0; i < count; i++) {
        printf("%d\n", values[i]);
    }

    free(values);
    values = NULL;
    return EXIT_SUCCESS;
}
```

`malloc` 从动态存储区申请一段至少满足基本对齐要求的未初始化内存，并返回 `void *`。在 C 中，`void *` 可以自动转换为其他对象指针，因此不需要也通常不应该强制转换 `malloc` 的返回值。使用 `sizeof *values` 能让分配大小跟随指针所指类型变化。

申请可能失败，所以必须检查 `NULL`。成功申请的内存需要且只能释放一次；忘记释放会泄漏，释放后继续访问是 use-after-free，重复释放则是 double-free。把本地指针设为 `NULL` 只能减少误用，无法让指向同一对象的其他指针自动失效。

### 头文件和多文件编译

```c
/* calculator.h */
#ifndef CALCULATOR_H
#define CALCULATOR_H

int add(int left, int right);

#endif
```

```c
/* calculator.c */
#include "calculator.h"

int add(int left, int right) {
    return left + right;
}
```

```c
/* main.c */
#include <stdio.h>
#include "calculator.h"

int main(void) {
    printf("%d\n", add(2, 3));
    return 0;
}
```

可以分开编译，再由链接器组合：

```bash
cc -Wall -Wextra -Wpedantic -c calculator.c
cc -Wall -Wextra -Wpedantic -c main.c
cc calculator.o main.o -o calculator
```

头文件主要放声明，源文件放定义。编译 `main.c` 时，编译器只需要通过声明知道 `add` 的参数和返回类型；链接阶段才会在 `calculator.o` 中寻找函数定义。include guard 防止同一头文件在一个翻译单元内被重复展开。

### 预处理器：从示例代码理解预处理指令

```c
#include <stdio.h>

#define ARRAY_LENGTH(array) (sizeof(array) / sizeof((array)[0]))

#if defined(DEBUG)
#define DEBUG_LOG(message) fprintf(stderr, "debug: %s\n", (message))
#else
#define DEBUG_LOG(message) ((void)0)
#endif

int main(void) {
    int values[] = {1, 2, 3};
    DEBUG_LOG("program started");
    printf("%zu\n", ARRAY_LENGTH(values));
    return 0;
}
```

这段代码把预处理器常见的三类工作放在了一起：`#include` 引入头文件，`#define` 定义宏，`#if` 和 `#else` 根据条件决定哪些代码保留。预处理发生在正式编译之前，可以把它理解为编译器处理 C 代码前的一道准备工序。预处理器读取源文件中的预处理指令，展开或删除相应内容，然后把结果交给编译器。

#### `#include`：引入头文件

`#include` 会把指定头文件的内容展开到当前位置。尖括号通常用于编译器或系统提供的头文件，双引号通常用于当前项目中的头文件：

```c
#include <stdio.h>
#include "calculator.h"

int main(void) {
    printf("2 + 3 = %d\n", add(2, 3));
    return 0;
}
```

这里并不是在运行时打开或加载一个 `.h` 文件，而是预处理器在编译前把头文件中的声明展开进当前源文件。头文件通常放声明，源文件放实现；`#ifndef`、`#define` 和 `#endif` 组成的 include guard，可以避免同一个头文件被重复展开。

#### `#define`：定义宏

宏是预处理器执行的替换规则。没有参数的宏可以给常量或配置取一个名字：

```c
#define BUFFER_SIZE 1024
#define APP_NAME "file-viewer"

char buffer[BUFFER_SIZE];
printf("%s\n", APP_NAME);
```

预处理器会把 `BUFFER_SIZE` 替换为 `1024`，把 `APP_NAME` 替换为字符串字面量。宏不是变量，不会占用一块可以修改的存储空间，也不会进行变量或类型检查。如果只是定义一个有类型的常量，现代 C 代码中也可以考虑使用 `const` 对象。

宏也可以定义参数。使用时虽然也写成类似函数调用的形式，但传入的内容只是替换对应参数。一个实际使用场景是计算数组中有多少个元素：

```c
#define ARRAY_COUNT(array) (sizeof(array) / sizeof((array)[0]))

int values[] = {10, 20, 30};
size_t count = ARRAY_COUNT(values);
```

`ARRAY_COUNT(values)` 不是函数调用，预处理后大致会变成 `sizeof(values) / sizeof((values)[0])`。`array` 只是替换规则中的占位符，不是函数参数。这个宏只适用于真正的数组；数组作为函数参数后会调整为指针，此时 `ARRAY_COUNT` 会算出错误结果。宏参数也可以替换成表达式；如果参数最终出现在表达式中，通常要给参数和整个结果都加括号，避免运算符优先级改变含义。

宏参数在替换结果中出现多次时，传入的表达式也可能被求值多次。因此，如果宏定义为 `#define TWICE(value) ((value) + (value))`，就不要传入 `i++` 这类带副作用的表达式：`TWICE(i++)` 会展开为类似 `((i++) + (i++))` 的代码，结果可能是未定义行为。能用普通函数或 `static inline` 函数表达时，通常优先使用它们。

#### `#if`、`#else` 和 `#endif`：条件编译

条件编译让预处理器根据宏是否定义，决定一段代码是否交给编译器：

```c
#include <stdio.h>

#if defined(DEBUG)
#define LOG(message) fprintf(stderr, "debug: %s\n", (message))
#else
#define LOG(message) ((void)0)
#endif

int main(void) {
    LOG("program started");
    return 0;
}
```

没有定义 `DEBUG` 时，`LOG("program started")` 会被替换为空操作；定义 `DEBUG` 后，它才会展开为输出调试信息的代码。可以在编译命令中使用 `-DDEBUG` 定义宏：

```bash
cc -DDEBUG main.c -o main
```

除了 `defined(NAME)`，也可以使用 `#ifdef NAME` 和 `#ifndef NAME` 分别测试宏是否已定义或尚未定义。条件编译常用于调试日志、平台差异和可选功能，但它会让同一份源代码产生多种编译结果，因此条件分支应尽量保持简单。

### 函数指针

```c
#include <stdio.h>

typedef int (*BinaryOperation)(int, int);

int add(int left, int right) {
    return left + right;
}

int multiply(int left, int right) {
    return left * right;
}

int calculate(BinaryOperation operation, int left, int right) {
    return operation(left, right);
}

int main(void) {
    printf("%d\n", calculate(add, 2, 3));
    printf("%d\n", calculate(multiply, 2, 3));
    return 0;
}
```

函数也有地址。函数指针描述了可调用函数的参数和返回类型，可以把具体行为作为参数传递。它是 C 中实现 callback、策略表、事件处理器和简单多态的基础。

调用约定和函数签名必须匹配；把不兼容的函数地址强行转换后调用会产生未定义行为。操作系统 API、动态链接和其他语言的 FFI 也经常需要函数指针，因此这一概念之后还会反复出现。

### 错误处理

```c
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    FILE *file = fopen("config.txt", "r");

    if (file == NULL) {
        int error = errno;
        errno = error;
        perror("cannot open config.txt");
        return EXIT_FAILURE;
    }

    if (fclose(file) == EOF) {
        perror("cannot close config.txt");
        return EXIT_FAILURE;
    }

    return EXIT_SUCCESS;
}
```

C 没有内建异常机制，函数通常通过特殊返回值报告失败，并可能把更具体的错误码放入 `errno`。只有文档明确说明失败时会设置 `errno` 的函数，才能这样读取它；成功调用后 `errno` 的旧值没有意义。

应当在确认失败后立即保存或使用 `errno`，因为后续函数调用可能改变它。`perror` 会根据当前 `errno` 输出错误说明。这里保存后再恢复看起来有些多余，却展示了“先捕获错误码，再调用可能影响状态的代码”的习惯。真实程序还需要保证失败路径释放此前已经获得的文件、内存和锁等资源。

### 从编译、汇编到链接

```c
/* main.c */
#include <stdio.h>

static int square(int value) {
    return value * value;
}

int main(void) {
    printf("%d\n", square(7));
    return 0;
}
```

可以停在构建过程的不同阶段观察产物：

```bash
cc -E main.c -o main.i       # 预处理后的 C
cc -S main.i -o main.s       # 汇编代码
cc -c main.s -o main.o       # 可重定位目标文件
cc main.o -o main            # 链接为可执行文件

nm main.o
objdump -d main
ldd main
```

预处理器先展开头文件和宏，编译器把 C 翻译为目标架构的汇编，汇编器生成包含机器码、符号和重定位信息的目标文件，链接器再解析不同目标文件及库之间的符号引用，生成可执行文件。

`printf` 并不是一条 C 语言指令，它是标准库声明的函数。链接器把这里的函数调用连接到 C 标准库实现；程序运行时，标准库最终还需要借助操作系统完成输出。这正是接下来继续区分“语言能力、标准库能力与系统能力”的入口。
