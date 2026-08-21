ESP32 UART Communication (ESP-IDF + FreeRTOS)

A practice project demonstrating UART communication on the ESP32, built with ESP-IDF and FreeRTOS. The board transmits and receives data over UART1, communicating with another device (e.g. STM32).

This project uses two independent FreeRTOS tasks:

TX Task — periodically sends a message over UART
RX Task — continuously listens for incoming UART data
Table of Contents
1. Hardware Configuration
2. Project Structure
3. UART Initialization
4. Sending Data — TX Task
5. Receiving Data — RX Task
6. Application Entry Point
7. Build & Run
8. Sample Output
9. Notes
1. Hardware Configuration

UART pins and communication parameters are configurable through menuconfig (Kconfig), instead of being hardcoded in the source file.

Parameter	Kconfig Option	Default
Baud rate	CONFIG_BAUD_RATE	115200
TX Pin	CONFIG_TX_PIN	GPIO17
RX Pin	CONFIG_RX_PIN	GPIO16
Task stack size	CONFIG_TASK_STACK_SIZE	4096
Kconfig:

menu "UART CONFIG"
    config BAUD_RATE
        int "UART communication speed"
        range 1200 115200
        default 115200
        help
            UART communication speed

    config TX_PIN
        int "UART TX_PIN"
        range 0 48
        default 17
        help
            UART TX_PIN Number

    config RX_PIN
        int "UART RX_PIN"
        range 0 48
        default 16
        help
            UART RX_PIN Number

    config TASK_STACK_SIZE
        int "UART TASK_STACK_SIZE"
        range 1024 8192
        default 4096
        help
            UART TASK_STACK_SIZE
endmenu

To change these values, run:

bash
idf.py menuconfig

Then navigate to UART CONFIG.

2. Project Structure
ESP32_UART_EX/
├── main/
│   ├── ESP32_UART_EX.c
│   ├── Kconfig.projbuild
│   └── CMakeLists.txt
├── CMakeLists.txt
├── sdkconfig
└── README.md
Root CMakeLists.txt
cmake
# The following five lines of boilerplate have to be in your project's
# CMakeLists in this exact order for cmake to work correctly
cmake_minimum_required(VERSION 3.16)
include($ENV{IDF_PATH}/tools/cmake/project.cmake)
project(ESP32_UART_EX)
main/CMakeLists.txt
cmake
idf_component_register(SRCS "ESP32_UART_EX.c"
                    REQUIRES esp_driver_uart esp_driver_gpio
                    INCLUDE_DIRS ".")
3. UART Initialization
c
static const int RX_BUF_SIZE = 1024;

#define TXD_PIN (CONFIG_TX_PIN)
#define RXD_PIN (CONFIG_RX_PIN)

void init(){
    const uart_config_t uart_config = {
        .baud_rate  = CONFIG_BAUD_RATE,
        .data_bits  = UART_DATA_8_BITS,
        .parity     = UART_PARITY_DISABLE,
        .stop_bits  = UART_STOP_BITS_1,
        .flow_ctrl  = UART_HW_FLOWCTRL_DISABLE,
        .source_clk = UART_SCLK_DEFAULT,
    };

    // Install UART driver
    uart_driver_install(UART_NUM_1, RX_BUF_SIZE * 2, 0, 0, NULL, 0);
    uart_param_config(UART_NUM_1, &uart_config);
    uart_set_pin(UART_NUM_1, TXD_PIN, RXD_PIN, UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE);
}

How it works:

uart_config_t defines the communication parameters (baud rate, data bits, parity, stop bits, flow control) — all pulled from Kconfig instead of hardcoded values.
uart_driver_install() allocates the internal RX/TX ring buffers used by the driver.
uart_param_config() applies the configuration to UART_NUM_1.
uart_set_pin() maps the TX/RX signals to the physical GPIO pins defined in Kconfig.
4. Sending Data — TX Task
c
int sendata(const char* logname, const char* data){
    const int len = strlen(data);
    const int txBytes = uart_write_bytes(UART_NUM_1, data, len);
    ESP_LOGI(logname, "Wrote: %d bytes", txBytes);
    return txBytes;
}

static void tx_task(void *arg){
    static const char *TX_TASK_TAG = "TX_TASK";
    esp_log_level_set(TX_TASK_TAG, ESP_LOG_INFO);

    while (1) {
        sendata(TX_TASK_TAG, "Hello stm32 from esp32\r\n");
        vTaskDelay(2000 / portTICK_PERIOD_MS);
    }
}

How it works:

sendata() wraps uart_write_bytes() and logs how many bytes were actually transmitted.
tx_task runs forever, sending the string "Hello stm32 from esp32\r\n" every 2 seconds.
5. Receiving Data — RX Task
c
static void rx_task(void *arg){
    static const char *RX_TASK_TAG = "RX_TASK";
    esp_log_level_set(RX_TASK_TAG, ESP_LOG_INFO);

    uint8_t* data = (uint8_t*)malloc(RX_BUF_SIZE + 1);

    while (1) {
        const int rxBytes = uart_read_bytes(UART_NUM_1, data, RX_BUF_SIZE, 1000 / portTICK_PERIOD_MS);

        if (rxBytes > 0) {
            data[rxBytes] = 0;
            ESP_LOGI(RX_TASK_TAG, "Read %d bytes: '%s'", rxBytes, data);
            ESP_LOG_BUFFER_HEXDUMP(RX_TASK_TAG, data, rxBytes, ESP_LOG_INFO);
        }
    }

    free(data);
}

How it works:

uart_read_bytes() blocks for up to 1000 ms waiting for incoming data, returning the number of bytes actually read.
If data was received, a null terminator is appended so it can be logged as a C string.
ESP_LOG_BUFFER_HEXDUMP() additionally prints the raw bytes in hex — useful for verifying non-printable characters (like \r\n) were received correctly.
6. Application Entry Point
c
void app_main(void)
{
   init();
   xTaskCreate(rx_task, "uart_rx_task", CONFIG_TASK_STACK_SIZE, NULL, configMAX_PRIORITIES - 1, NULL);
   xTaskCreate(tx_task, "uart_tx_task", CONFIG_TASK_STACK_SIZE, NULL, configMAX_PRIORITIES - 2, NULL);
}
init() sets up the UART peripheral.
rx_task is created with a higher priority (configMAX_PRIORITIES - 1) than tx_task (configMAX_PRIORITIES - 2), since receiving incoming data should not be delayed by the transmit loop.
7. Build & Run
bash
idf.py set-target esp32
idf.py menuconfig     # optional: adjust TX/RX pins, baud rate, stack size
idf.py build
idf.py -p <PORT> flash monitor

Replace <PORT> with your board's serial port (e.g. COM5 on Windows, /dev/ttyUSB0 on Linux).

8. Sample Output

Example output from the VS Code integrated terminal (ESP-IDF Monitor), showing both TX and RX tasks running simultaneously while communicating with an STM32 board over UART:

<img width="700" alt="uart_terminal_output" src="https://github.com/user-attachments/assets/PASTE_YOUR_LINK_HERE" />
I (8607) RX_TASK: 0x3ffb6df0  68 65 6c 6c 6f 20 65 73  70 33 32 20 66 72 6f 6d  |hello esp32 from|
I (8607) RX_TASK: 0x3ffb6e00  20 73 74 6d 33 32 0d 0a                          | stm32..|
I (10297) TX_TASK: Wrote: 24 bytes
I (10617) RX_TASK: Read 24 bytes: 'hello esp32 from stm32
'
I (10617) RX_TASK: 0x3ffb6df0  68 65 6c 6c 6f 20 65 73  70 33 32 20 66 72 6f 6d  |hello esp32 from|
I (10617) RX_TASK: 0x3ffb6e00  20 73 74 6d 33 32 0d 0a                          | stm32..|
TX_TASK: Wrote: 24 bytes confirms the ESP32 successfully transmitted its message.
RX_TASK: Read 24 bytes: '...' shows the message received back from the STM32 (in this test setup, STM32 echoes/sends its own greeting), together with the raw hex dump.
9. Notes
This project uses UART_NUM_1 (not the default USB-serial console UART), so a separate physical TX/RX wiring is required to the other device (e.g. STM32).
All configurable parameters (baud rate, pins, stack size) are exposed via idf.py menuconfig under UART CONFIG, avoiding hardcoded values in the source file.
rx_task is given higher priority than tx_task to minimize the chance of missing incoming bytes.
