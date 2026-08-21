# STM32 UART Communication (Interrupt & DMA)

A practice project demonstrating UART communication on the **STM32F103C8T6** (Blue Pill), built with STM32CubeMX and Keil MDK-ARM. The board transmits/receives data over USART1, communicating with another device (e.g. ESP32).

This repo demonstrates **two different UART receive methods**:
- **Interrupt (IT)** — receiving data byte by byte
- **DMA (Idle Line Detection)** — receiving a full data packet with minimal CPU usage

## Table of Contents

- [1. Hardware Configuration](#1-hardware-configuration)
- [2. Method 1 — UART Receive via Interrupt (IT)](#2-method-1--uart-receive-via-interrupt-it)
  - [CubeMX Configuration](#cubemx-configuration)
  - [Variable Declarations](#variable-declarations-user-code-begin-0)
  - [Starting Interrupt-based Reception](#starting-interrupt-based-reception-user-code-begin-2)
  - [Main Loop](#main-loop-user-code-begin-3)
  - [Receive-Complete Callback](#receive-complete-callback-user-code-begin-4)
  - [Verifying via Debug](#verifying-via-debug-watch-window)
  - [Pros / Cons of IT](#pros--cons-of-it)
- [3. Method 2 — UART Receive via DMA (Idle Line Detection)](#3-method-2--uart-receive-via-dma-idle-line-detection)
  - [CubeMX Configuration](#cubemx-configuration-1)
  - [Variable Declarations](#variable-declarations-user-code-begin-0-1)
  - [Starting Idle-Line Reception](#starting-idle-line-reception-user-code-begin-2)
  - [Idle-Detected Callback](#idle-detected-callback-user-code-begin-4)
  - [Pros / Cons of DMA](#pros--cons-of-dma)
- [4. Quick Comparison: IT vs DMA](#4-quick-comparison-it-vs-dma)
- [5. Project Structure](#5-project-structure)
- [6. Notes](#6-notes)

---

## 1. Hardware Configuration

| Parameter              | Value                           |
|-------------------------|----------------------------------|
| MCU                     | STM32F103C8T6 (LQFP48)          |
| Peripheral              | USART1                          |
| Mode                    | Asynchronous                    |
| Hardware Flow Control   | Disable                         |
| Baudrate                | As configured in `MX_USART1_UART_Init()` |
| TX Pin                  | PA9 (USART1_TX)                 |
| RX Pin                  | PA10 (USART1_RX)                |

Configured in **STM32CubeMX**: `Connectivity → USART1 → Mode: Asynchronous`.

<img width="700" alt="usart1_config" src="https://github.com/user-attachments/assets/3c27d2ae-87da-40fd-86b4-118e46abd119" />

---

## 2. Method 1 — UART Receive via Interrupt (IT)

### CubeMX Configuration
- Enable **USART1** only — no DMA required.
- Under **NVIC Settings**, `USART1 global interrupt` is enabled automatically once UART is turned on.

### Variable Declarations (USER CODE BEGIN 0)

```c
uint8_t tx_buff[] = "hello esp32 from stm32\r\n";

uint8_t rx_byte;
char    rx_buff[100];
uint8_t rx_index = 0;
```

<img width="700" alt="it_main_declarations" src="https://github.com/user-attachments/assets/92f3da60-44f1-4d56-998f-70be36cfebca" />

- `rx_byte`: temporary buffer holding the single byte just received by the interrupt.
- `rx_buff`: buffer that accumulates bytes into a complete string.
- `rx_index`: current write position in `rx_buff`.

### Starting Interrupt-based Reception (USER CODE BEGIN 2)

```c
HAL_UART_Receive_IT(&huart1, &rx_byte, 1);
```

Called once before the `while(1)` loop to start listening for the first incoming byte.

### Main Loop (USER CODE BEGIN 3)

```c
while (1)
{
    HAL_UART_Transmit(&huart1, tx_buff, sizeof(tx_buff) - 1, 1000);
    HAL_Delay(2000);
}
```

Every 2 seconds, the STM32 transmits the string `"hello esp32 from stm32\r\n"` over UART.

<img width="700" alt="it_main_init_loop" src="https://github.com/user-attachments/assets/a660189e-2411-4376-bfa5-0dd1985985c7" />

### Receive-Complete Callback (USER CODE BEGIN 4)

```c
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        if (rx_index < sizeof(rx_buff) - 1)
        {
            rx_buff[rx_index++] = rx_byte;

            if (rx_byte == '\n')
            {
                rx_buff[rx_index] = '\0';
                rx_index = 0;
            }
        }
        else
        {
            rx_index = 0;
        }

        // Re-arm interrupt reception for the next byte
        HAL_UART_Receive_IT(&huart1, &rx_byte, 1);
    }
}
```

<img width="700" alt="it_rx_callback" src="https://github.com/user-attachments/assets/27fdb417-b317-41a6-bf0f-df8338498f87" />

**How it works:**
1. Every time UART finishes receiving one byte, an interrupt fires and HAL automatically calls `HAL_UART_RxCpltCallback`.
2. That byte is appended to `rx_buff`.
3. When a `'\n'` character is detected (end of line), the string is considered complete and `rx_index` is reset to prepare for the next packet.
4. `HAL_UART_Receive_IT()` **must be called again** at the end of the callback — otherwise UART will only receive a single byte and then stop listening.

### Verifying via Debug (Watch Window)

Using Keil's Debug mode, add `rx_buff` to the **Watch 1** window to observe incoming data in real time (e.g. seeing `"Hello st..."` when the ESP32 sends a string).

<img width="700" alt="it_debug_watch" src="https://github.com/user-attachments/assets/00398a80-f208-4124-ae4f-0eb9db761e75" />

### Pros / Cons of IT
| Pros | Cons |
|---|---|
| Simple to understand and debug | Interrupt fires on every single byte — CPU load increases at high baud rates |
| No DMA configuration needed | Data can be lost if the callback processing is too slow |

---

## 3. Method 2 — UART Receive via DMA (Idle Line Detection)

### CubeMX Configuration

1. Enable **USART1** as usual.
2. Go to the **DMA Settings** tab → click **Add** → add two DMA requests:

| DMA Request | Channel | Direction |
|---|---|---|
| USART1_RX | DMA1 Channel 5 | Peripheral To Memory |
| USART1_TX | DMA1 Channel 4 | Memory To Peripheral |

<img width="700" alt="dma_config" src="https://github.com/user-attachments/assets/f0a77cd4-2f3d-4e92-b574-7a72739b7aad" />

3. CubeMX will automatically generate the `MX_DMA_Init()` function and the following handles:

```c
DMA_HandleTypeDef hdma_usart1_rx;
DMA_HandleTypeDef hdma_usart1_tx;
```

### Variable Declarations (USER CODE BEGIN 0)

```c
uint8_t tx_buff[] = "hello esp32 from stm32\r\n";
uint8_t rx_buff[100];
```

<img width="700" alt="dma_main_declarations" src="https://github.com/user-attachments/assets/36c83cfe-e07e-41f6-8047-bdc3ec7010cd" />

No `rx_byte` or `rx_index` are needed here — DMA automatically writes the whole incoming packet directly into `rx_buff`.

### Starting Idle-Line Reception (USER CODE BEGIN 2)

```c
MX_GPIO_Init();
MX_DMA_Init();
MX_USART1_UART_Init();

HAL_UARTEx_ReceiveToIdle_DMA(&huart1, rx_buff, sizeof(rx_buff));
```

<img width="700" alt="dma_main_init" src="https://github.com/user-attachments/assets/401679bb-fc45-447c-9ff9-6a22e0141987" />

`HAL_UARTEx_ReceiveToIdle_DMA` continuously receives data into `rx_buff` and automatically notifies the application once the line becomes **idle** — meaning the sender has stopped transmitting. Unlike the IT method, this does not depend on a specific terminating character such as `'\n'`.

### Idle-Detected Callback (USER CODE BEGIN 4)

```c
void HAL_UART_RxEventCallback(UART_HandleTypeDef *huart, uint16_t size)
{
    if (huart->Instance == USART1)
    {
        rx_buff[size] = '\0';
        HAL_UARTEx_ReceiveToIdle_DMA(&huart1, rx_buff, sizeof(rx_buff));
    }
}
```

<img width="700" alt="dma_rx_callback" src="https://github.com/user-attachments/assets/bf4ecb2a-c454-4f92-a227-1992841b9b5e" />

**How it works:**
1. DMA receives data into `rx_buff` in the background, without CPU intervention per byte.
2. When the UART line goes "idle" (no new data for a certain period), HAL automatically calls `HAL_UART_RxEventCallback`, passing `size` — the actual number of bytes received.
3. A null terminator `'\0'` is added at position `size` to complete the string.
4. `HAL_UARTEx_ReceiveToIdle_DMA()` is called again to keep listening for the next packet.

### Pros / Cons of DMA
| Pros | Cons |
|---|---|
| Receives full packets, independent of a terminating character like `\n` | Requires additional DMA channel configuration |
| Very low CPU usage (DMA runs in the background) | More complex to debug compared to IT |
| Suitable for high baud rates or larger data payloads | |

---

## 4. Quick Comparison: IT vs DMA

| Criteria | Interrupt (IT) | DMA (Idle Line) |
|---|---|---|
| Reception granularity | Byte by byte | Full data block |
| CPU load | Higher (frequent interrupts) | Lower (DMA runs in background) |
| Needs known packet length in advance | No (relies on `'\n'`) | No (relies on Idle Line detection) |
| Configuration complexity | Simple | Requires DMA channel setup |
| Best suited for | Small payloads, low baud rate | Large payloads, high baud rate, performance-critical applications |

---

## 5. Project Structure

```
04_STM32_UART/
├── Core/
├── MDK-ARM/
├── .mxproject
├── 04_STM32_UART.ioc
└── README.md
```

---

## 6. Notes

- The project was generated with **STM32CubeMX** and built/flashed using **Keil MDK-ARM (µVision)**.
- All user-added code is placed inside `/* USER CODE BEGIN */ ... /* USER CODE END */` blocks so that CubeMX does not overwrite it when regenerating code.
