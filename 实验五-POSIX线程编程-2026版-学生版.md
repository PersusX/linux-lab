# 《Linux系统及应用》课程实验指导书（学生用·2026版）

# 实验五 POSIX 线程编程

| 项目 | 内容 |
|---|---|
| 课程名称 | Linux 系统及应用 |
| 实验学时 | 2 学时（1 次课内上机，2 × 45 分钟）＋课后完善 |
| 实验类型 | 综合性实验 |
| 先修要求 | 已完成实验四，理解进程、共享内存、并发执行与 C 指针 |
| 考核方式 | 现场检查＋实验报告 |

> 线程共享同一进程的地址空间和文件描述符，但每个线程有自己的栈、寄存器与调度状态。共享使数据交换更直接，也使竞态条件成为必须面对的问题。

---

## 一、实验目的

1. 理解进程与线程在资源共享、创建成本和故障影响方面的差异；
2. 掌握 `pthread_create()`、`pthread_join()` 和线程函数参数传递；
3. 验证线程对进程全局数据的共享；
4. 识别“共享数据”与“竞态条件”的区别；
5. 会为每个输入文件创建一个线程并汇总单词数；
6. 会处理线程创建失败、文件打开失败和线程返回结果；
7. 了解互斥量和条件变量的基本作用。

---

## 二、实验环境与时间安排

```bash
gcc --version | head -1
getconf GNU_LIBPTHREAD_VERSION
command -v gcc gdb
mkdir -p ~/linux-lab05/{src,bin,data,logs,report}
cd ~/linux-lab05
```

| 阶段 | 内容 | 用时 |
|---|---|---:|
| 第 1 阶段 | 任务 A：创建、汇合线程；任务 B：共享数据与竞态演示 | 35 分钟 |
| 第 2 阶段 | 任务 C：并发单词统计 | 40 分钟 |
| 验收 | 边界测试、解释结果 | 15 分钟 |

编译线程程序必须使用 `-pthread`：

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -g -O0 -pthread src/program.c -o bin/program
```

`-pthread` 不只是链接库，还会为编译和链接阶段设置与线程相关的选项，应优先使用它，而不是只写 `-lpthread`。

---

## 三、实验内容及操作步骤

### 任务 A 创建与汇合线程

创建 `src/thread_basic.c`：

```c
#include <errno.h>
#include <pthread.h>
#include <stdio.h>
#include <string.h>

struct worker_arg {
    int input;
    int output;
};

static void *worker(void *raw)
{
    struct worker_arg *arg = raw;
    printf("worker: input=%d\n", arg->input);
    arg->output = arg->input * arg->input;
    return NULL;
}

int main(void)
{
    pthread_t thread;
    struct worker_arg arg = {.input = 12, .output = 0};

    int error = pthread_create(&thread, NULL, worker, &arg);
    if (error != 0) {
        fprintf(stderr, "pthread_create: %s\n", strerror(error));
        return 1;
    }

    error = pthread_join(thread, NULL);
    if (error != 0) {
        fprintf(stderr, "pthread_join: %s\n", strerror(error));
        return 1;
    }

    printf("main: output=%d\n", arg.output);
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -g -O0 -pthread \
  src/thread_basic.c -o bin/thread_basic
./bin/thread_basic
```

**观察与解释：**

- `pthread_create()` 将 `&arg` 作为 `void *` 传给线程函数；
- `pthread_join()` 等待指定线程结束；
- 主线程在 `join` 之后读取 `arg.output`，此时工作线程已经完成写入；
- pthread 函数通常直接返回错误号，不能简单假设它们总是设置 `errno`。

---

### 任务 B 共享数据与竞态条件

#### B-1 可控的共享修改

创建 `src/shared_value.c`：

```c
#include <pthread.h>
#include <stdio.h>
#include <string.h>

static int shared_value = 0;

static void *set_value(void *raw)
{
    int value = *(int *)raw;
    shared_value = value;
    return NULL;
}

int main(void)
{
    pthread_t thread;
    int value = 42;
    int error = pthread_create(&thread, NULL, set_value, &value);
    if (error != 0) {
        fprintf(stderr, "pthread_create: %s\n", strerror(error));
        return 1;
    }
    error = pthread_join(thread, NULL);
    if (error != 0) {
        fprintf(stderr, "pthread_join: %s\n", strerror(error));
        return 1;
    }
    printf("shared_value=%d\n", shared_value);
    return 0;
}
```

本例只有一个工作线程写入，主线程在 `join` 之后读取，因此并不需要互斥量。“数据被共享”不等于“一定有竞态”；竞态取决于是否有多个线程并发访问同一数据，且至少一个访问为写入，同时又没有正确同步。

#### B-2 亲眼看见竞态条件

创建 `src/race_demo.c`：两个线程各对同一个全局计数器加 1,000,000 次。传入参数 `0` 时不加锁，传入 `1` 时加锁。

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define LOOPS 1000000L

static long counter = 0;
static int use_lock = 0;
static pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;

static void *add_many(void *raw)
{
    (void)raw;
    for (long i = 0; i < LOOPS; ++i) {
        if (use_lock) {
            pthread_mutex_lock(&lock);
        }
        ++counter;                 /* 读取—加一—写回，并不是一条不可分割的操作 */
        if (use_lock) {
            pthread_mutex_unlock(&lock);
        }
    }
    return NULL;
}

int main(int argc, char *argv[])
{
    if (argc != 2 || (strcmp(argv[1], "0") != 0 && strcmp(argv[1], "1") != 0)) {
        fprintf(stderr, "usage: %s 0|1   (0=不加锁, 1=加锁)\n", argv[0]);
        return 2;
    }
    use_lock = atoi(argv[1]);

    pthread_t t1, t2;
    int error = pthread_create(&t1, NULL, add_many, NULL);
    if (error != 0) {
        fprintf(stderr, "pthread_create: %s\n", strerror(error));
        return 1;
    }
    error = pthread_create(&t2, NULL, add_many, NULL);
    if (error != 0) {
        fprintf(stderr, "pthread_create: %s\n", strerror(error));
        pthread_join(t1, NULL);
        return 1;
    }
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    printf("expected=%ld actual=%ld %s\n", 2 * LOOPS, counter,
           counter == 2 * LOOPS ? "OK" : "LOST UPDATES");
    return counter == 2 * LOOPS ? 0 : 1;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -g -O0 -pthread \
  src/race_demo.c -o bin/race_demo
for n in 1 2 3 4 5; do ./bin/race_demo 0; done     # 不加锁：结果通常小于 2000000，且每次不同
for n in 1 2 3 4 5; do ./bin/race_demo 1; done     # 加锁：每次都是 2000000
```

> 虚拟机只分配了 1 个 CPU 核心时，不加锁的版本也可能偶尔得到正确结果，此时把虚拟机改为 2 核以上再试。**“这次结果是对的”并不能证明没有竞态**，这正是思考题 4 要讨论的问题。

用 `time` 比较两个版本的耗时，思考加锁的代价，以及为什么任务 C 选择“每个线程一个私有结果槽”而不是“共用一个加锁的计数器”。

#### B-3 互斥量基本形式

```c
static pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;

pthread_mutex_lock(&lock);
/* 读写共享状态 */
pthread_mutex_unlock(&lock);
```

互斥量保护的是“不变式和临界区”，而不是某个变量的名字。临界区应尽量短，不要在持有互斥量时执行长时间 I/O、`sleep()` 或可能阻塞的操作。

---

### 任务 C 每个文件一个线程的单词统计

#### C-1 单词定义

本实验将“由一个或多个英文字母或数字连续构成的序列”定义为单词。汉字 UTF-8 编码由多字节构成，不在本次简化规则的统计范围内。

准备数据：

```bash
printf 'Linux has processes and threads.\n' > data/a.txt
printf 'Thread 1 counts words; thread 2 counts too.\n' > data/b.txt
: > data/empty.txt
```

#### C-2 完整程序

创建 `src/word_count.c`：

```c
#include <ctype.h>
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

struct task {
    const char *path;
    unsigned long words;
    int failed;
};

static void *count_words(void *raw)
{
    struct task *task = raw;
    FILE *file = fopen(task->path, "rb");
    if (file == NULL) {
        perror(task->path);
        task->failed = 1;
        return NULL;
    }

    int in_word = 0;
    int ch;
    while ((ch = fgetc(file)) != EOF) {
        int is_word_char = isalnum((unsigned char)ch);
        if (is_word_char && !in_word) {
            ++task->words;
        }
        in_word = is_word_char;
    }
    if (ferror(file)) {
        fprintf(stderr, "%s: read error\n", task->path);
        task->failed = 1;
    }
    if (fclose(file) == EOF) {
        fprintf(stderr, "%s: close error\n", task->path);
        task->failed = 1;
    }
    return NULL;
}

int main(int argc, char *argv[])
{
    if (argc < 2) {
        fprintf(stderr, "usage: %s FILE...\n", argv[0]);
        return 2;
    }

    size_t count = (size_t)(argc - 1);
    pthread_t *threads = calloc(count, sizeof *threads);
    struct task *tasks = calloc(count, sizeof *tasks);
    if (threads == NULL || tasks == NULL) {
        fprintf(stderr, "memory allocation failed\n");
        free(threads);
        free(tasks);
        return 1;
    }

    size_t created = 0;
    for (size_t i = 0; i < count; ++i) {
        tasks[i].path = argv[i + 1];
        int error = pthread_create(&threads[i], NULL, count_words, &tasks[i]);
        if (error != 0) {
            fprintf(stderr, "pthread_create: %s\n", strerror(error));
            break;
        }
        ++created;
    }

    int status = created == count ? 0 : 1;
    unsigned long total = 0;
    for (size_t i = 0; i < created; ++i) {
        int error = pthread_join(threads[i], NULL);
        if (error != 0) {
            fprintf(stderr, "pthread_join: %s\n", strerror(error));
            status = 1;
            continue;
        }
        printf("%s: %lu\n", tasks[i].path, tasks[i].words);
        total += tasks[i].words;
        if (tasks[i].failed) {
            status = 1;
        }
    }
    printf("total: %lu\n", total);

    free(threads);
    free(tasks);
    return status;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -g -O0 -pthread \
  src/word_count.c -o bin/word_count
./bin/word_count data/a.txt data/b.txt data/empty.txt
echo "exit=$?"
```

#### C-3 交叉验证

```bash
wc -w data/a.txt data/b.txt data/empty.txt
./bin/word_count data/missing.txt
echo "missing exit=$?"
./bin/word_count
echo "no-argument exit=$?"
```

`wc -w` 对“单词”的定义与本程序不完全相同，因此它适合对简单英文样例做交叉验证，不能在所有输入上无条件当作“标准答案”。

#### C-4 为什么本程序不需要互斥量

每个线程只写自己的 `tasks[i]`，主线程在相应 `pthread_join()` 完成后才读取该结果。不同线程没有并发写同一个计数器，所以无需用互斥量保护。这种“每线程私有结果槽＋主线程汇总”的设计比让所有线程抢一把锁更简单。

---

## 四、课后选做：条件变量

用条件变量让两个线程按约定顺序修改共享变量 `g`。选做目标：

1. 线程 1 在互斥量保护下将 `g` 设为 1，然后发送条件信号；
2. 线程 2 用 `while (g != 1) pthread_cond_wait(...)` 等待，观察到 1 后将 `g` 设为 2；
3. 主线程汇合两个线程后检查 `g == 2`。

关键形式：

```c
pthread_mutex_lock(&lock);
while (!condition) {
    pthread_cond_wait(&condition_changed, &lock);
}
/* 在持锁状态下检查或修改共享状态 */
pthread_mutex_unlock(&lock);
```

必须用 `while` 重新检查条件，不应用 `if` 假定每次唤醒都代表目标条件已经成立。

---

## 五、常见问题

1. **把循环变量 `&i` 传给所有线程：**多个线程会共享同一地址，读到的值取决于调度时机。应为每个线程准备独立的参数结构。
2. **主线程使用 `return` 过早结束：**整个进程终止，尚未完成的工作线程也被终止。应 `join` 必须完成的线程。
3. **用 `errno` 解释 `pthread_create` 失败：**pthread 函数通常直接返回错误号，应对返回值使用 `strerror(error)`。
4. **单词数偶尔异常：**检查是否让多线程未同步地修改同一全局计数器。
5. **`isalnum(ch)` 在非 ASCII 输入上异常：**当 `char` 可为负数时，应先转换为 `unsigned char`。
6. **加锁后忘记解锁：**其他线程可能永久阻塞。每个错误路径都必须检查是否需要解锁和释放资源。

---

## 六、思考题

报告中选答不少于 5 题，第 3、5 题必答。

1. 线程与进程分别共享哪些资源？
2. `pthread_create()` 的线程函数为什么使用 `void *` 参数和返回值？
3. `pthread_join()` 除了“等待线程结束”，对结果可见性还有什么意义？
4. 什么是竞态条件？为什么程序“大多数时候正确”仍然不能接受？
5. `word_count` 为什么不需要互斥量？如果所有线程直接累加同一个 `total`，应如何改造？
6. 为什么互斥量临界区应尽量短？
7. 条件变量为什么必须和互斥量及共享状态配合使用？
8. 为什么 `pthread_cond_wait()` 外层应用 `while` 而不是 `if`？

---

## 七、实验报告与验收

### 7.1 报告内容

1. 实验环境和编译命令；
2. `thread_basic`、`shared_value`、`race_demo`、`word_count` 源码与运行证据（`race_demo` 需给出不加锁、加锁各至少 5 次的运行结果）；
3. 用图表说明每个线程的参数、写入结果槽和主线程汇总顺序；
4. 文本、空文件、缺失文件和无参数四类测试结果；
5. 指定思考题。

### 7.2 现场验收

- [ ] 所有程序使用 `-pthread` 且编译无警告。
- [ ] 能解释 `pthread_create()` 和 `pthread_join()` 的参数、返回值。
- [ ] 能说明 `shared_value` 为什么能在主线程中观察到。
- [ ] 能现场演示 `race_demo` 不加锁时丢失更新、加锁后结果正确，并解释 `++counter` 为什么不是原子操作。
- [ ] `word_count` 可同时处理至少 3 个文件，每文件结果和总数正确。
- [ ] 缺失文件时报错并返回非 0 状态。
- [ ] 能解释为什么本程序不需要互斥量。

### 7.3 提交文件

```text
linux-lab05/
├── src/
│   ├── thread_basic.c
│   ├── shared_value.c
│   ├── race_demo.c
│   └── word_count.c
├── data/
├── logs/
└── report/
    └── 实验五报告.pdf
```
