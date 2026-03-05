# micro-ROS on STM32F446ZE (NUCLEO-F446ZE)

Bidirectional communication between ROS2 Humble (PC) and STM32F446ZE using micro-ROS over UART DMA transport with FreeRTOS.

---

## Hardware
- Board: NUCLEO-F446ZE (STM32F446ZETx)
- Transport: USART3 + DMA (ST-Link virtual COM port)
- FPU: FPV4-SP-D16, hard float ABI
- RAM: 128KB | Flash: 512KB
- Clock: 168MHz (HSE bypass, PLL)

---

## Software Stack
- STM32CubeIDE 1.19.0
- FreeRTOS (CMSIS-RTOS V2)
- micro-ROS static library (microros_static_library_ide)
- ROS2 Humble (PC)
- micro-ROS agent (native install)

---

## Project Structure

```
microros_pubsub/
├── Core/
│   ├── Inc/
│   └── Src/
│       ├── main.c                      <- micro-ROS node, pub/sub logic
│       ├── custom_memory_manager.c     <- copied from micro_ros_stm32cubemx_utils
│       ├── microros_allocators.c       <- copied from micro_ros_stm32cubemx_utils
│       └── microros_time.c             <- copied from micro_ros_stm32cubemx_utils
├── extra_sources/
│   └── microros_transports/
│       └── dma_transport.c             <- DMA UART transport (source registered in project)
├── micro_ros_stm32cubemx_utils/
│   └── microros_static_library_ide/
│       └── libmicroros/
│           ├── libmicroros.a           <- static library
│           └── include/                <- micro-ROS headers
├── Drivers/
├── Middlewares/
└── STM32F446ZETX_FLASH.ld
```

---

## Step-by-Step Setup

### Step 1 — Clone micro-ROS utils into project root

```bash
cd ~/STM32CubeIDE/workspace_1.19.0/microros_pubsub
git clone -b humble https://github.com/micro-ROS/micro_ros_stm32cubemx_utils.git
```

### Step 2 — Copy required source files into Core/Src

These three files must be in Core/Src so the IDE never excludes them:

```bash
cp micro_ros_stm32cubemx_utils/extra_sources/custom_memory_manager.c Core/Src/
cp micro_ros_stm32cubemx_utils/extra_sources/microros_allocators.c Core/Src/
cp micro_ros_stm32cubemx_utils/extra_sources/microros_time.c Core/Src/
```

### Step 3 — Project Properties Configuration

In STM32CubeIDE, right-click project -> Properties:

#### C/C++ Build -> Settings -> MCU/MPU GCC Compiler -> Include paths
Add the following:
```
../extra_sources
../micro_ros_stm32cubemx_utils/microros_static_library_ide/libmicroros/include
```

#### C/C++ Build -> Settings -> MCU/MPU GCC Linker -> Libraries
- Library search path (-L): `../micro_ros_stm32cubemx_utils/microros_static_library_ide/libmicroros`
- Library (-l): `microros`

#### Register dma_transport.c as source
The file `extra_sources/microros_transports/dma_transport.c` must be compiled.
It gets picked up automatically if `extra_sources/microros_transports` is a registered source path.
This is set in `.cproject` under `<sourceEntries>`:

```xml
<entry flags="VALUE_WORKSPACE_PATH|RESOLVED" kind="sourcePath"
       name="extra_sources/microros_transports"/>
```

> WARNING: STM32CubeIDE may mark these files as "Exclude from build".
> If that happens, copy dma_transport.c to Core/Src as well.

### Step 4 — FreeRTOS Configuration (CubeMX)

Open `.ioc` -> Middleware -> FreeRTOS -> Config parameters:

| Parameter               | Value  |
|-------------------------|--------|
| configTOTAL_HEAP_SIZE   | 30000  |
| microros_task Stack Size| 3000 (words) = 12000 bytes |
| microros_task Priority  | Normal |

> WARNING: Never reduce microros_task stack below 3000 words.

### Step 5 — Build

Project -> Build (Ctrl+B)

Expected output:
```
   text    data     bss     dec
  97932     712   88576  187220
0 errors, 0 warnings
```

### Step 6 — Flash and Run

Flash via STM32CubeIDE Run button (F11) or:
```bash
st-flash write Debug/microros_pubsub.elf 0x08000000
```

---

## Running on PC

Always activate ROS2 environment first in every new terminal:
```bash
rosenv
```

Start micro-ROS agent:
```bash
ros2 run micro_ros_agent micro_ros_agent serial --dev /dev/ttyACM0 -b 115200
```

Check topics:
```bash
ros2 topic list
# Expected:
# /stm32_command
# /stm32_reply
```

Send command to STM32:
```bash
ros2 topic pub /stm32_command std_msgs/msg/Int32 "{data: 5}"
```

Read reply from STM32:
```bash
ros2 topic echo /stm32_reply std_msgs/msg/Int32
# Expected: data: 50  (STM32 multiplies received value by 10)
```

---

## Communication Flow

```
PC (ROS2)                             STM32F446ZE
──────────────────────────────────────────────────
ros2 topic pub                        subscription_callback()
/stm32_command Int32(5)   ─────────►  receives msg.data = 5
                                      pub_msg.data = 5 * 10 = 50
ros2 topic echo           ◄─────────  rcl_publish(&publisher, &pub_msg)
/stm32_reply Int32(50)

Transport path:
ROS2 Node -> micro-ROS agent -> UART (115200) -> STM32 DMA -> FreeRTOS task
```

---

## Working Code Pattern (main.c)

```c
// Globals
rcl_publisher_t publisher;
rcl_subscription_t subscriber;
std_msgs__msg__Int32 pub_msg;
std_msgs__msg__Int32 sub_msg;

// Callback — called when message received from ROS2
void subscription_callback(const void *msgin)
{
    const std_msgs__msg__Int32 *msg = (const std_msgs__msg__Int32 *)msgin;
    pub_msg.data = msg->data * 10;
    rcl_publish(&publisher, &pub_msg, NULL);
}

// micro-ROS task
void start_microros_task(void *argument)
{
    rmw_uros_set_custom_transport(true, (void *)&huart3,
        cubemx_transport_open, cubemx_transport_close,
        cubemx_transport_write, cubemx_transport_read);

    rcl_allocator_t freeRTOS_allocator = rcutils_get_zero_initialized_allocator();
    freeRTOS_allocator.allocate      = microros_allocate;
    freeRTOS_allocator.deallocate    = microros_deallocate;
    freeRTOS_allocator.reallocate    = microros_reallocate;
    freeRTOS_allocator.zero_allocate = microros_zero_allocate;
    rcutils_set_default_allocator(&freeRTOS_allocator);

    rclc_support_t support;
    rclc_support_init(&support, 0, NULL, &freeRTOS_allocator);

    rcl_node_t node;
    rclc_node_init_default(&node, "stm32_node", "", &support);

    rclc_publisher_init_default(&publisher, &node,
        ROSIDL_GET_MSG_TYPE_SUPPORT(std_msgs, msg, Int32), "stm32_reply");

    rclc_subscription_init_default(&subscriber, &node,
        ROSIDL_GET_MSG_TYPE_SUPPORT(std_msgs, msg, Int32), "stm32_command");

    rclc_executor_t executor;
    rclc_executor_init(&executor, &support.context, 1, &freeRTOS_allocator);
    rclc_executor_add_subscription(&executor, &subscriber, &sub_msg,
                                   &subscription_callback, ON_NEW_DATA);
    for(;;)
    {
        rclc_executor_spin_some(&executor, RCL_MS_TO_NS(10));
        osDelay(10);
    }
}
```

---

## Common Pitfalls

### 1. Extra source files excluded from build
**Symptom:** Undefined reference to `microros_allocate` etc. at link time.  
**Cause:** IDE auto-checks "Exclude resource from build" on non-standard folders.  
**Fix:** Copy `custom_memory_manager.c`, `microros_allocators.c`, `microros_time.c`
into `Core/Src/`. IDE never excludes files from Core/Src.

### 2. Wrong path to source files
**Symptom:** `cp: cannot stat ... No such file or directory`  
**Cause:** Files are inside `micro_ros_stm32cubemx_utils/extra_sources/`, not `extra_sources/` directly.  
**Fix:** Use find to locate files before copying:
```bash
find ~/STM32CubeIDE/workspace_1.19.0/microros_pubsub -name "microros_allocators.c"
```

### 3. RAM overflow at link time
**Symptom:** `section .bss will not fit in region RAM`  
**Cause:** micro-ROS static buffers + FreeRTOS heap exceeds 128KB.  
**Fix:** Set `configTOTAL_HEAP_SIZE` = `30000` in CubeMX.
Do not set higher than 45000 on STM32F446ZE with micro-ROS.

### 4. microros_task minimum stack
**Symptom:** CubeMX rejects stack size or board crashes at runtime.  
**Cause:** micro-ROS needs minimum 3000 words for its task.  
**Fix:** Never reduce microros_task stack below `3000` words.

### 5. ros2 command not found
**Symptom:** `ros2: command not found` in terminal.  
**Cause:** ROS2 environment not sourced.  
**Fix:** Run `rosenv` in every new terminal before any ros2 command.

### 6. Topic not receiving with --once
**Symptom:** "Waiting for at least 1 matching subscription(s)..."  
**Cause:** STM32 micro-ROS task takes a few seconds to initialize. `--once` exits too fast.  
**Fix:** Publish continuously without `--once` for a few seconds.

### 7. IDE rewrites .cproject
**Symptom:** Manual .cproject edits are reverted after any GUI change.  
**Cause:** STM32CubeIDE regenerates .cproject on every property change.  
**Fix:** Avoid editing .cproject directly. Use Core/Src for all source files.

### 8. CubeMX regeneration overwrites code
**Symptom:** Custom code disappears after regenerating from .ioc.  
**Fix:** Always place custom code inside USER CODE BEGIN/END blocks:
```c
/* USER CODE BEGIN 0 */
// your code here — safe from regeneration
/* USER CODE END 0 */
```

---

## Next Project Goal
- ROS2 sends rover commands (direction + RPM) to STM32
- STM32 drives 4x BLDC motors
- STM32 sends back IMU + 4x encoder feedback to ROS2
- ROS2 closes the control loop based on feedback
