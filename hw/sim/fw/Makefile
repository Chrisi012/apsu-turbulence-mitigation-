# ==============================================================================
# APSU-01 Bare-Metal Firmware Build System
# Target: STM32G431K8 (ARM Cortex-M4F @ 170 MHz)
# Toolchain: arm-none-eabi-gcc
# ==============================================================================

TARGET = apsu_fw
BUILD_DIR = build

CC = arm-none-eabi-gcc
OBJCOPY = arm-none-eabi-objcopy
SIZE = arm-none-eabi-size

# Source files
C_SOURCES = \
src/main.c

# Assembly startup
ASM_SOURCES = 

# MCU Flags
CPU = -mcpu=cortex-m4
FPU = -mfpu=fpv4-sp-d16
FLOAT-ABI = -mfloat-abi=hard
MCU = $(CPU) -mthumb $(FPU) $(FLOAT-ABI)

# Linker Script
LDSCRIPT = linker/stm32g431xb.ld

# Compiler Flags
CFLAGS = $(MCU) -std=c11 -Wall -Wextra -Werror -O2 -g3 \
         -ffunction-sections -fdata-sections \
         -fno-builtin -nostdlib

# Linker Flags
LDFLAGS = $(MCU) -T$(LDSCRIPT) -Wl,--gc-sections \
          -Wl,-Map=$(BUILD_DIR)/$(TARGET).map,--cref \
          -nostartfiles

all: $(BUILD_DIR)/$(TARGET).elf $(BUILD_DIR)/$(TARGET).bin $(BUILD_DIR)/$(TARGET).hex

OBJECTS = $(addprefix $(BUILD_DIR)/,$(notdir $(C_SOURCES:.c=.o)))
vpath %.c $(sort $(dir $(C_SOURCES)))

$(BUILD_DIR)/%.o: %.c | $(BUILD_DIR)
	$(CC) -c $(CFLAGS) $< -o $@

$(BUILD_DIR)/$(TARGET).elf: $(OBJECTS) | $(BUILD_DIR)
	$(CC) $(OBJECTS) $(LDFLAGS) -o $@
	$(SIZE) $@

$(BUILD_DIR)/%.bin: $(BUILD_DIR)/%.elf | $(BUILD_DIR)
	$(OBJCOPY) -O binary $< $@

$(BUILD_DIR)/%.hex: $(BUILD_DIR)/%.elf | $(BUILD_DIR)
	$(OBJCOPY) -O ihex $< $@

$(BUILD_DIR):
	mkdir -p $@

clean:
	rm -rf $(BUILD_DIR)

.PHONY: all clean
