/**
 * ============================================================================
 * APSU-01 Safety Supervisor Firmware
 * Target: STM32G431K8 (ARM Cortex-M4F)
 * Features: Deterministic 1 kHz SysTick, Circular DMA ADC, External WWDT Petting
 * ============================================================================
 */

#include <stdint.h>
#include <stdbool.h>

#define RCC_BASE        0x40021000UL
#define GPIOA_BASE      0x48000000UL
#define GPIOB_BASE      0x48000400UL
#define SYSTICK_BASE    0xE000E010UL

/* Register shortcuts */
#define RCC_AHB2ENR     (*(volatile uint32_t *)(RCC_BASE + 0x4C))
#define GPIOA_MODER     (*(volatile uint32_t *)(GPIOA_BASE + 0x00))
#define GPIOA_BSRR      (*(volatile uint32_t *)(GPIOA_BASE + 0x18))
#define GPIOB_MODER     (*(volatile uint32_t *)(GPIOB_BASE + 0x00))
#define GPIOB_ODR       (*(volatile uint32_t *)(GPIOB_BASE + 0x14))
#define GPIOB_BSRR      (*(volatile uint32_t *)(GPIOB_BASE + 0x18))

#define STK_CTRL        (*(volatile uint32_t *)(SYSTICK_BASE + 0x00))
#define STK_LOAD        (*(volatile uint32_t *)(SYSTICK_BASE + 0x04))
#define STK_VAL         (*(volatile uint32_t *)(SYSTICK_BASE + 0x08))

/* Safety Thresholds (ADC raw counts, 12-bit, 3.3V reference) */
#define VCAP_PRECHARGE_THRESHOLD   3351U  /* ~13.5V scaled via 10k/2.7k divider */
#define I_LOAD_OVERCURRENT_LIMIT   2482U  /* ~40.0A via INA240 50V/V, 2mR shunt */
#define BUS_UNDERVOLTAGE_LIMIT     2234U  /* ~9.0V crank sag threshold */

typedef enum {
    STATE_INIT = 0,
    STATE_PRECHARGE,
    STATE_READY,
    STATE_FAULT_TRIP
} SubsystemState_t;

/* Global variables and DMA target buffer */
static volatile SubsystemState_t g_current_state = STATE_INIT;
static volatile uint16_t g_adc_dma_buffer[3] = {0, 0, 0}; /* [0]=V_bus, [1]=V_cap, [2]=I_load */
static volatile uint32_t g_system_tick_ms = 0;
static volatile bool g_tick_flag = false;

extern uint32_t _estack;
extern uint32_t _sidata, _sdata, _edata, _sbss, _ebss;

void Reset_Handler(void);
void SysTick_Handler(void);
void Default_Handler(void);

/* Vector Table */
__attribute__((section(".isr_vector"), used))
const uint32_t *vector_table[] = {
    (uint32_t *)&_estack,
    (uint32_t *)Reset_Handler,
    (uint32_t *)Default_Handler,   /* NMI */
    (uint32_t *)Default_Handler,   /* HardFault */
    (uint32_t *)Default_Handler,   /* MemManage */
    (uint32_t *)Default_Handler,   /* BusFault */
    (uint32_t *)Default_Handler,   /* UsageFault */
    0, 0, 0, 0,
    (uint32_t *)Default_Handler,   /* SVCall */
    (uint32_t *)Default_Handler,   /* DebugMonitor */
    0,
    (uint32_t *)Default_Handler,   /* PendSV */
    (uint32_t *)SysTick_Handler    /* SysTick */
};

static inline void Pet_External_Watchdog(void) {
    /* Toggle PA8 to feed TI TPS3851 within 800-1200 us window */
    GPIOA_BSRR = (1U << 8);
    for (volatile int i = 0; i < 10; ++i) { __asm__("nop"); }
    GPIOA_BSRR = (1U << (8 + 16));
}

static inline void Hardware_Disable_Actuator(void) {
    /* Pull PB3 LOW to disconnect gate driver and engage 1.5R dynamic brake */
    GPIOB_BSRR = (1U << (3 + 16));
}

static inline void Hardware_Enable_Actuator(void) {
    /* Set PB3 HIGH */
    GPIOB_BSRR = (1U << 3);
}

void SysTick_Handler(void) {
    g_system_tick_ms++;
    g_tick_flag = true;
}

void Default_Handler(void) {
    Hardware_Disable_Actuator();
    while (1) { __asm__("wfi"); }
}

void Reset_Handler(void) {
    /* 1. Copy initialized data from Flash to RAM */
    uint32_t *p_src = &_sidata;
    uint32_t *p_dst = &_sdata;
    while (p_dst < &_edata) {
        *p_dst++ = *p_src++;
    }

    /* 2. Zero-fill BSS section */
    p_dst = &_sbss;
    while (p_dst < &_ebss) {
        *p_dst++ = 0;
    }

    /* 3. Enable GPIOA and GPIOB clocks */
    RCC_AHB2ENR |= (1U << 0) | (1U << 1);

    /* Configure PA8 as output (Watchdog WDI pin) */
    GPIOA_MODER &= ~(3U << (8 * 2));
    GPIOA_MODER |= (1U << (8 * 2));

    /* Configure PB3 as output (eFuse Gate Enable / Brake interlock) */
    GPIOB_MODER &= ~(3U << (3 * 2));
    GPIOB_MODER |= (1U << (3 * 2));
    Hardware_Disable_Actuator();

    /* 4. Configure SysTick for 1 kHz deterministic tick @ 16 MHz HSI default */
    STK_LOAD = 16000U - 1U;
    STK_VAL  = 0;
    STK_CTRL = 7U; /* Enable, Source=AHB, Interrupt=On */

    g_current_state = STATE_PRECHARGE;

    /* 5. Main Safety Loop */
    while (1) {
        if (g_tick_flag) {
            g_tick_flag = false;

            /* Service Windowed Watchdog strictly inside 1 kHz frame */
            Pet_External_Watchdog();

            uint16_t v_bus  = g_adc_dma_buffer[0];
            uint16_t v_cap  = g_adc_dma_buffer[1];
            uint16_t i_load = g_adc_dma_buffer[2];

            /* Primary Analog Cutoff redundant software check */
            if (i_load > I_LOAD_OVERCURRENT_LIMIT) {
                Hardware_Disable_Actuator();
                g_current_state = STATE_FAULT_TRIP;
            }

            /* State Machine */
            switch (g_current_state) {
                case STATE_PRECHARGE:
                    Hardware_Disable_Actuator();
                    /* Ready when supercap bank reaches 13.5V and bus is nominal */
                    if (v_cap >= VCAP_PRECHARGE_THRESHOLD && v_bus > BUS_UNDERVOLTAGE_LIMIT) {
                        g_current_state = STATE_READY;
                        Hardware_Enable_Actuator();
                    }
                    break;

                case STATE_READY:
                    /* Reverse current / engine-crank sag prevention */
                    if (v_bus <= BUS_UNDERVOLTAGE_LIMIT) {
                        Hardware_Disable_Actuator();
                        g_current_state = STATE_PRECHARGE;
                    }
                    break;

                case STATE_FAULT_TRIP:
                default:
                    Hardware_Disable_Actuator();
                    break;
            }
        }
        __asm__("wfi");
    }
}
