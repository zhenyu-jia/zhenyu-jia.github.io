# 关于 FIFO 的一些细节处理

## 普通的 FIFO

先看一个普通的 FIFO 的实现：

```c
#include <stdint.h>
#include <stdio.h>

typedef struct
{
    uint8_t *buffer;
    uint32_t size;
    uint32_t head;
    uint32_t tail;
} fifo_t;

// 初始化 FIFO
void fifo_init(fifo_t *fifo, uint8_t *buf, uint32_t size)
{
    fifo->buffer = buf;
    fifo->size = size;
    fifo->head = fifo->tail = 0;
}

// 判断 FIFO 是否为空
int fifo_is_empty(fifo_t *fifo)
{
    return fifo->head == fifo->tail;
}

// 判断 FIFO 是否为满
int fifo_is_full(fifo_t *fifo)
{
    uint32_t next = fifo->head + 1;
    if (next >= fifo->size)
        next = 0;
    return next == fifo->tail;
}

// 返回 FIFO 中剩余空间
uint32_t fifo_space(fifo_t *fifo)
{
    if (fifo->head >= fifo->tail)
        return (fifo->size - 1) - (fifo->head - fifo->tail);
    else
        return (fifo->tail - fifo->head - 1);
}

// 入队一个字节
int fifo_put(fifo_t *fifo, uint8_t data)
{
    if (fifo_is_full(fifo))
        return -1;

    fifo->buffer[fifo->head] = data;
    fifo->head++;
    if (fifo->head >= fifo->size)
        fifo->head = 0;
    return 0;
}

// 出队一个字节
int fifo_get(fifo_t *fifo, uint8_t *data)
{
    if (fifo_is_empty(fifo))
        return -1;

    *data = fifo->buffer[fifo->tail];
    fifo->tail++;
    if (fifo->tail >= fifo->size)
        fifo->tail = 0;
    return 0;
}
```

下面是一个简单的测试代码，用于测试 FIFO 的功能。

```c
/* 测试代码 */
#include <stdio.h>

#define BUF_SIZE 8

int main(void)
{
    uint8_t buffer[BUF_SIZE];
    fifo_t fifo;

    fifo_init(&fifo, buffer, BUF_SIZE);

    // 读一下 FIFO 的空间
    printf("FIFO Size: %d\n", fifo_space(&fifo));

    // 写入数据
    for (uint8_t i = 0; i < 7; ++i)
    {
        if (fifo_put(&fifo, i) == 0)
            printf("Put %d\n", i);
        else
            printf("FIFO Full\n");
    }

    // 判断 FIFO 是否为满
    printf("FIFO Full: %d\n", fifo_is_full(&fifo));
    // 读一下 FIFO 的空间
    printf("FIFO Size: %d\n", fifo_space(&fifo));

    // 读取数据
    uint8_t data;
    while (!fifo_is_empty(&fifo))
    {
        if (fifo_get(&fifo, &data) == 0)
            printf("Got %d\n", data);
        else
            printf("FIFO Empty\n");
    }

    // 判断 FIFO 是否为空
    printf("FIFO Empty: %d\n", fifo_is_empty(&fifo));
    // 读一下 FIFO 的空间
    printf("FIFO Size: %d\n", fifo_space(&fifo));

    return 0;
}
```

输出结果：

```bash
FIFO Size: 7
Put 0
Put 1
Put 2
Put 3
Put 4
Put 5
Put 6
FIFO Full: 1
FIFO Size: 0
Got 0
Got 1
Got 2
Got 3
Got 4
Got 5
Got 6
FIFO Empty: 1
FIFO Size: 7
```

可以看出一个问题，这个 FIFO 可用的的 size 比初始化的 `fifo->size` 少 1, 这是因为是采用 `fifo->head + 1 == fifo->tail` 来判断 FIFO 是否已满。

再将其改成环形 FIFO，只需要改下面几个函数的实现即可

```c
// 判断 FIFO 是否为满
int fifo_is_full(fifo_t *fifo)
{
    return ((fifo->head + 1) % fifo->size) == fifo->tail;
}

// 入队一个字节
int fifo_put(fifo_t *fifo, uint8_t data)
{
    if (fifo_is_full(fifo))
        return -1;

    fifo->buffer[fifo->head] = data;
    fifo->head = (fifo->head + 1) % fifo->size;
    return 0;
}

// 出队一个字节
int fifo_get(fifo_t *fifo, uint8_t *data)
{
    if (fifo_is_empty(fifo))
        return -1;

    *data = fifo->buffer[fifo->tail];
    fifo->tail = (fifo->tail + 1) % fifo->size;
    return 0;
}
```

由于还是采用 `fifo->head + 1 == fifo->tail` 来判断 FIFO 是否已满，所以可用的 size 最大还是 `fifo->size - 1`, 不过用取余运算 `(fifo->head + 1) % fifo->size` 来减少了回绕的判断。
当然如果 size 为 2 的幂，可以用 `(fifo->head + 1) & fifo->size_mask` 来提高效率，其中 `fifo->size_mask = fifo->size - 1`;

## KFIFO

### 无符号数的溢出回绕特性

对于 C 语言中的无符号类型（如 `uint32_t`），当其值增加到最大值再加 1 时，会自动回绕到 0。例如：

```c
uint32_t a = 0xFFFFFFFF;
a = a + 1; // a 变为 0
```

这种特性可以用来实现“环形缓冲区”指针的回绕，无需手动判断和重置。更重要的是，利用无符号数的减法结果也能正确反映两个指针之间的距离，即使发生了回绕。例如：

```c
uint32_t head = 2, tail = 10;
uint32_t used = head - tail; // 如果 head < tail，结果会自动回绕为正数
```

在实现 FIFO 时，可以利用无符号整数的溢出回绕特性来简化指针的移动和空间计算。

KFIFO 是 Linux 内核提供的一个 FIFO 队列，其结构如下：

```c
#include <stdint.h>
#include <stdio.h>

// kfifo：缓冲区大小必须为 2 的幂，用掩码优化取模
typedef struct
{
    uint8_t *buffer;
    uint32_t size;
    uint32_t in;
    uint32_t out;
    uint32_t mask;
} kfifo_t;

// 初始化（size 必须是 2 的幂）
void kfifo_init(kfifo_t *fifo, uint8_t *buf, uint32_t size)
{
    fifo->buffer = buf;
    fifo->size = size;
    fifo->mask = size - 1;
    fifo->in = fifo->out = 0;
}

// 判断 FIFO 是否为空
int kfifo_is_empty(kfifo_t *fifo)
{
    return fifo->in == fifo->out;
}

// 判断 FIFO 是否为满
int kfifo_is_full(kfifo_t *fifo)
{
    return (fifo->in - fifo->out) == fifo->size;
}

// 返回可用空间
uint32_t kfifo_space(kfifo_t *fifo)
{
    return fifo->size - (fifo->in - fifo->out);
}

// 入队一个字节
int kfifo_put(kfifo_t *fifo, uint8_t data)
{
    if (kfifo_is_full(fifo))
        return -1;

    fifo->buffer[fifo->in & fifo->mask] = data;
    fifo->in++;
    return 0;
}

// 出队一个字节
int kfifo_get(kfifo_t *fifo, uint8_t *data)
{
    if (kfifo_is_empty(fifo))
        return -1;

    *data = fifo->buffer[fifo->out & fifo->mask];
    fifo->out++;
    return 0;
}
```

下面是一个简单的测试代码，用于测试 KFIFO 的功能。

```c
/* 测试代码 */
#include <stdio.h>

#define BUF_SIZE 8 // 缓冲区大小，必须是 2 的幂
#if BUF_SIZE & (BUF_SIZE - 1)
#error "BUF_SIZE must be a power of 2"
#endif

int main(void)
{
    uint8_t buffer[BUF_SIZE];
    kfifo_t fifo;

    kfifo_init(&fifo, buffer, BUF_SIZE);

    // 读一下 FIFO 的空间
    printf("FIFO Size: %d\n", kfifo_space(&fifo));

    // 写入数据
    for (uint8_t i = 0; i < 7; ++i)
    {
        if (kfifo_put(&fifo, i) == 0)
            printf("Put %d\n", i);
        else
            printf("FIFO Full\n");
    }

    // 判断 FIFO 是否为满
    printf("FIFO Full: %d\n", kfifo_is_full(&fifo));
    // 读一下 FIFO 的空间
    printf("FIFO Size: %d\n", kfifo_space(&fifo));

    // 读取数据
    uint8_t data;
    while (!kfifo_is_empty(&fifo))
    {
        if (kfifo_get(&fifo, &data) == 0)
            printf("Got %d\n", data);
        else
            printf("FIFO Empty\n");
    }

    // 判断 FIFO 是否为空
    printf("FIFO Empty: %d\n", kfifo_is_empty(&fifo));
    // 读一下 FIFO 的空间
    printf("FIFO Size: %d\n", kfifo_space(&fifo));

    return 0;
}
```

输出结果：

```bash
FIFO Size: 8
Put 0
Put 1
Put 2
Put 3
Put 4
Put 5
Put 6
FIFO Full: 0
FIFO Size: 1
Got 0
Got 1
Got 2
Got 3
Got 4
Got 5
Got 6
FIFO Empty: 1
FIFO Size: 8
```

可以看出来运行是正常的，而且 Fifo 的容量是 8，没有像普通的 FIFO 一样浪费一个空间来判断容量是否满了。

### 无符号数的另一个应用

在 stm32 HAL 库里面的 SysTick 中断函数中，有一个变量 `uwTick` 用来记录时间，这个变量的类型是 `uint32_t`，每次中断都会对这个变量进行加 1。

```c
void SysTick_Handler(void)
{
    /* USER CODE BEGIN SysTick_IRQn 0 */

    /* USER CODE END SysTick_IRQn 0 */
    HAL_IncTick();
    /* USER CODE BEGIN SysTick_IRQn 1 */

    /* USER CODE END SysTick_IRQn 1 */
}

__weak void HAL_IncTick(void)
{
    uwTick += uwTickFreq;
}

HAL_TickFreqTypeDef uwTickFreq = HAL_TICK_FREQ_DEFAULT;  /* 1KHz */

typedef enum
{
    HAL_TICK_FREQ_10HZ         = 100U,
    HAL_TICK_FREQ_100HZ        = 10U,
    HAL_TICK_FREQ_1KHZ         = 1U,
    HAL_TICK_FREQ_DEFAULT      = HAL_TICK_FREQ_1KHZ
} HAL_TickFreqTypeDef;
```

将上面的函数展开就得到下面的代码：

```c
void SysTick_Handler(void)
{
    uwTick++;
}
```

在 HAL 库里面经常有这样的函数，比如 `HAL_StatusTypeDef HAL_UART_Transmit(UART_HandleTypeDef *huart, const uint8_t *pData, uint16_t Size, uint32_t Timeout)`, 它有一个参数 `Timeout`，这个参数是等待发送完成或者超时的时间。这类函数大概是这样一个结构：

```c
HAL_StatusTypeDef HAL_UART_Transmit(UART_HandleTypeDef *huart, const uint8_t *pData, uint16_t Size, uint32_t Timeout)
{
    uint32_t tickstart = HAL_GetTick();

    while (/* 一些条件 */)
    {
        if (Timeout != HAL_MAX_DELAY)
        {
            if (((HAL_GetTick() - tickstart) > Timeout) || (Timeout == 0U))
            {
                return HAL_TIMEOUT;
            }
        }
    }
    return HAL_OK;
}
```

其中 `HAL_GetTick` 就是获取全局变量`uwTick` 的值

```c
uint32_t HAL_GetTick(void)
{
    return uwTick;
}
```

超时的原理就是用当前的时间减去开始时间，如果大于超时时间，就返回超时。在起初见到这个代码时会认为这个 `uwTick` 会溢出，所以要所溢出判断，其实考虑到这一点的大部分人也在这么做，难道 STM32 的 HAL 库的超时处理有 bug？当了解到无符号数的溢出回绕特性时就不会这么想了，仍然可以可靠判断是否超时。因此，STM32 HAL 中的超时机制是安全的。
