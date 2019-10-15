cmake_minimum_required(VERSION 3.5)
project(Rocbot)

set(CMAKE_C_COMPILER arm-none-eabi-gcc)
set(CMAKE_CXX_COMPILER arm-none-eabi-g++)
set(CMAKE_OBJCOPY arm-none-eabi-objcopy)
set(CMAKE_SIZE arm-none-eabi-size)

add_definitions(
-DSTM32F40_41xxx
-DUSE_STDPERIPH_DRIVER
-DARM_MATH_CM4
)

#set(CMAKE_BUILD_TYPE "Debug")
#set(CMAKE_BUILD_TYPE "Release")
set(MCU_FLAGS "-mcpu=cortex-m4 -mfloat-abi=hard -mfpu=fpv4-sp-d16")
set(CMAKE_C_FLAGS "${MCU_FLAGS} -w -Wno-unknown-pragmas")
set(CMAKE_C_FLAGS_DEBUG "-O0 -g2 -ggdb")
set(CMAKE_C_FLAGS_RELEASE "-O3")

include_directories(
    sdk/stm32lib
    sdk/stm32lib/inc
    sdk/stm32lib/CORE
    sdk/stm32lib/USB
    sdk/stm32lib/USB/STM32_USB_Device_Library/Class/cdc/inc
    sdk/stm32lib/USB/STM32_USB_Device_Library/Core/inc
    sdk/stm32lib/USB/STM32_USB_OTG_Driver/inc
    sdk/stm32lib/SDIO
    sdk/freertos/Source/include
    sdk/freertos/Source/portable/GCC/ARM_CM4F
    sdk/fatfs
    sdk/mavlink
    sdk/mavlink/common
    src/CONTROL 
    src/DRIVER 
    src/LOG 
    src/MATH 
    src/MESSAGE 
    src/MODULE 
    src/NAVIGATION 
    src/SENSOR
    src/SYSTEM
    src/TASK
    ${CMAKE_CURRENT_BINARY_DIR}
)

set_property(SOURCE sdk/stm32lib/startup/GCC/startup_stm32f40_41xxx.s PROPERTY LANGUAGE C)

add_library(stm32_lib
    sdk/stm32lib/startup/GCC/startup_stm32f40_41xxx.s
    sdk/stm32lib/src/misc.c 
    sdk/stm32lib/src/stm32f4xx_adc.c 
    sdk/stm32lib/src/stm32f4xx_can.c 
    sdk/stm32lib/src/stm32f4xx_dma.c 
    sdk/stm32lib/src/stm32f4xx_flash.c 
    sdk/stm32lib/src/stm32f4xx_rcc.c 
    sdk/stm32lib/src/stm32f4xx_gpio.c 
    sdk/stm32lib/src/stm32f4xx_tim.c 
    sdk/stm32lib/src/stm32f4xx_spi.c 
    sdk/stm32lib/src/stm32f4xx_pwr.c 
    sdk/stm32lib/src/stm32f4xx_sdio.c 
    sdk/stm32lib/src/stm32f4xx_usart.c 
    sdk/stm32lib/src/stm32f4xx_syscfg.c 
    sdk/stm32lib/system_stm32f4xx.c
    sdk/stm32lib/USB/usb_bsp.c 
    sdk/stm32lib/USB/usbd_desc.c
    sdk/stm32lib/USB/STM32_USB_Device_Library/Class/cdc/src/usbd_cdc_core.c
    sdk/stm32lib/USB/STM32_USB_Device_Library/Core/src/usbd_ioreq.c
    sdk/stm32lib/USB/STM32_USB_Device_Library/Core/src/usbd_req.c
    sdk/stm32lib/USB/STM32_USB_Device_Library/Core/src/usbd_core.c
    sdk/stm32lib/USB/STM32_USB_OTG_Driver/src/usb_core.c
    sdk/stm32lib/USB/STM32_USB_OTG_Driver/src/usb_dcd_int.c
    sdk/stm32lib/USB/STM32_USB_OTG_Driver/src/usb_dcd.c
)

add_library(freertos
    sdk/freertos/Source/tasks.c 
    sdk/freertos/Source/list.c 
    sdk/freertos/Source/queue.c 
    sdk/freertos/Source/portable/GCC/ARM_CM4F/port.c 
    sdk/freertos/Source/portable/MemMang/heap_4.c
)

add_library(fatfs
    sdk/fatfs/diskio.c 
    sdk/fatfs/ff.c 
    sdk/fatfs/option/ccsbcs.c
    sdk/stm32lib/SDIO/stm32f4_sdio_sd_LowLevel.c
    sdk/stm32lib/SDIO/stm32f4_sdio_sd.c
)
target_link_libraries(fatfs
    stm32_lib
)

file(GLOB SRC_DRIVER src/driver/*.c)
file(GLOB SRC_CONTROL src/control/*.c)
file(GLOB SRC_LOG src/log/*.c)
file(GLOB SRC_MATH src/math/*.c)
file(GLOB SRC_MESSAGE src/message/*.c)
file(GLOB SRC_MODULE src/module/*.c)
file(GLOB SRC_NAVIGATION src/navigation/*.c)
file(GLOB SRC_SENSOR src/sensor/*.c)
file(GLOB SRC_SYSTEM src/system/*.c)
file(GLOB SRC_TASK src/task/*.c)

add_library(${PROJECT_NAME}
    ${SRC_DRIVER}
    ${SRC_CONTROL}
    ${SRC_LOG}
    ${SRC_MATH}
    ${SRC_MESSAGE}
    ${SRC_MODULE}
    ${SRC_NAVIGATION}
    ${SRC_SENSOR}
    ${SRC_SYSTEM}
    ${SRC_TASK}
)
target_link_libraries(${PROJECT_NAME} -lm
    stm32_lib
    fatfs
    freertos
)

set(LINKER_SCRIPT ${CMAKE_CURRENT_SOURCE_DIR}/sdk/stm32lib/STM32F405RGTx_FLASH.ld)
set(CMAKE_EXE_LINKER_FLAGS
"--specs=nano.specs -specs=nosys.specs -nostartfiles -T${LINKER_SCRIPT} -Wl,-Map=${PROJECT_BINARY_DIR}/${PROJECT_NAME}.map,--cref -Wl,--gc-sections"
)

add_executable(${PROJECT_NAME}.elf src/main.c)
target_link_libraries(${PROJECT_NAME}.elf
    ${PROJECT_NAME}
)

set(ELF_FILE ${PROJECT_BINARY_DIR}/${PROJECT_NAME}.elf)
set(HEX_FILE ${PROJECT_BINARY_DIR}/${PROJECT_NAME}.hex)
set(BIN_FILE ${PROJECT_BINARY_DIR}/${PROJECT_NAME}.bin)

add_custom_command(TARGET "${PROJECT_NAME}.elf" POST_BUILD
    COMMAND ${CMAKE_OBJCOPY} -Obinary ${ELF_FILE} ${BIN_FILE}
    COMMAND ${CMAKE_OBJCOPY} -Oihex  ${ELF_FILE} ${HEX_FILE}
    COMMENT "Building ${PROJECT_NAME}.bin and ${PROJECT_NAME}.hex"

    COMMAND ${CMAKE_COMMAND} -E copy ${HEX_FILE} "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}.hex"
    COMMAND ${CMAKE_COMMAND} -E copy ${BIN_FILE} "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}.bin"
    
    COMMAND ${CMAKE_SIZE} --format=berkeley ${PROJECT_NAME}.elf ${PROJECT_NAME}.hex
    COMMENT "Invoking: Cross ARM GNU Print Size"
)
