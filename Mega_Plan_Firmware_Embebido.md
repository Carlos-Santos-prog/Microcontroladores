# Mega Plan de Estudio — De C Estándar a Firmware Profesional
### Mentor: Ingeniero Senior de Firmware | Plataformas: NXP FRDM-MCXA156 (MCUXpresso SDK) y ESP32 (ESP-IDF/FreeRTOS)

> **Filosofía de este plan:** No vas a memorizar funciones de un SDK. Vas a entender *qué registro toca cada función por debajo*, y vas a practicar la separación de capas (`driver` vs `app`) desde el ejercicio #1, porque ese hábito es lo que separa a un estudiante de un ingeniero de firmware.

---

# FASE 0 — Refuerzo Intensivo de "C para Hardware"

Antes de tocar un solo periférico, necesitas que estas tres herramientas sean **reflejo**, no algo que googleas cada vez.

## 0.1 Operaciones a Nivel de Bits (Bitwise)

En hardware, un registro de 32 bits no es "una variable". Es un panel de interruptores físicos. Cada bit controla algo distinto (un pin, un flag, un modo de reloj). Por eso nunca escribes el registro completo a ciegas — modificas *solo los bits que te interesan*.

### Las 4 operaciones que vas a usar el 90% del tiempo

```c
#define BIT(n)  (1u << (n))   // Máscara: "quiero apuntar al bit n"

// 1. SET (encender un bit) -> OR
REGISTRO |= BIT(5);           // Pone el bit 5 en 1, no toca los demás

// 2. CLEAR (apagar un bit) -> AND con NOT
REGISTRO &= ~BIT(5);          // Pone el bit 5 en 0, no toca los demás

// 3. TOGGLE (invertir un bit) -> XOR
REGISTRO ^= BIT(5);           // Si estaba en 1 pasa a 0 y viceversa

// 4. READ (leer un bit) -> AND + comparación
if (REGISTRO & BIT(5)) {      // ¿Está encendido el bit 5?
    // ...
}
```

**¿Por qué `~BIT(5)` y no escribir el número directo?** Porque `~BIT(5)` se lee como intención ("todo excepto el bit 5"), mientras que un número mágico como `0xFFFFFFDF` no le dice nada a quien lea tu código (ni a ti en 3 meses).

### Máscaras de múltiples bits (campos de configuración)

Muchos registros no son "on/off", son **campos** de varios bits. Ejemplo típico: seleccionar la fuente de reloj de un periférico, que ocupa los bits 4-5 de un registro.

```c
#define CLK_SRC_MASK   (0x3u << 4)   // Máscara: bits 4 y 5
#define CLK_SRC_SHIFT  4

// Para escribir un valor de 2 bits (ej: 0b10) en ese campo:
REGISTRO = (REGISTRO & ~CLK_SRC_MASK) | ((0x2u << CLK_SRC_SHIFT) & CLK_SRC_MASK);
```

Este patrón "**limpiar la máscara, luego OR con el valor desplazado**" es EXACTAMENTE lo que hacen las macros de los SDKs de NXP y Espressif por debajo (ej: `GPIO_PinInit`, `gpio_set_direction`). Cuando lo entiendas, dejarás de ver el SDK como "magia" y empezarás a ver que es solo bitwise empaquetado en funciones con nombres bonitos.

### Ejercicio Didáctico 0.1
Tienes un registro simulado de 8 bits `uint8_t GPIO_PDOR` que representa el estado de 8 LEDs.
1. Escribe una función `void led_set(uint8_t pin)` que encienda solo un LED sin apagar los demás.
2. Escribe `void led_pattern_alternate(void)` que encienda los LEDs pares (0,2,4,6) y apague los impares, usando **una sola operación** con una máscara literal `0b01010101`.
3. **Reto:** Escribe una función `uint8_t leds_count_on(void)` que cuente cuántos LEDs están encendidos usando solo operadores bitwise y corrimientos (sin `if` dentro de un bucle bit a bit... piensa en `n & (n-1)`).

---

## 0.2 Punteros y Memoria en Hardware

### Punteros a registros de hardware

Un microcontrolador mapea sus periféricos en direcciones de memoria fijas (esto se llama **Memory-Mapped I/O**). Un "registro" no es más que una dirección de memoria específica que el fabricante documenta en el *Reference Manual*.

```c
// Esto es, literalmente, lo que hay detrás de cualquier macro del SDK
#define GPIOA_BASE      0x40020000u
#define GPIOA_PDOR      (*(volatile uint32_t *)(GPIOA_BASE + 0x00))

GPIOA_PDOR |= BIT(5);   // Esto ES un acceso directo a hardware
```

### `volatile` — por qué es obligatorio y no opcional

El compilador, al optimizar, asume que si tú no modificas una variable en tu código C, su valor **no cambia** entre lecturas, y puede cachearla en un registro de la CPU o eliminar lecturas "redundantes".

El problema: un registro de hardware puede cambiar **sin que tu código lo escriba** (ej: un flag de interrupción que pone el hardware en 1 cuando llega un dato por UART). Si no marcas el puntero como `volatile`, el compilador puede optimizar tu `while (!(STATUS & FLAG_RX)) {}` a un bucle infinito real, porque "cree" que `STATUS` nunca cambia.

```c
// SIN volatile: el compilador puede optimizar este bucle a infinito
uint32_t *status = (uint32_t *)STATUS_REG_ADDR;
while (!(*status & RX_READY)) { }   // PELIGRO en -O2

// CON volatile: el compilador SIEMPRE relee la dirección de memoria
volatile uint32_t *status = (volatile uint32_t *)STATUS_REG_ADDR;
while (!(*status & RX_READY)) { }   // Correcto
```

**Regla práctica que vas a aplicar toda tu carrera:**
> Si una variable puede cambiar por: (a) una interrupción, (b) el hardware directamente, o (c) otra tarea/hilo → es `volatile`. Si solo cambia por tu propio flujo secuencial de código → no lo necesita.

### Punteros a función (para callbacks)

Cuando configuras una interrupción o un driver de comunicación, casi nunca defines tú qué hace la ISR internamente — le pasas una **función que el driver va a llamar cuando ocurra el evento**. Esto se llama patrón *callback*.

```c
// Definimos un "tipo" de función: recibe un uint8_t, no retorna nada
typedef void (*uart_rx_callback_t)(uint8_t received_byte);

// El driver guarda un puntero a la función que tú le des
static uart_rx_callback_t g_user_callback = NULL;

void uart_set_rx_callback(uart_rx_callback_t cb) {
    g_user_callback = cb;
}

// Dentro de la ISR real del UART:
void UART0_IRQHandler(void) {
    uint8_t byte = UART0_DATA_REG;
    if (g_user_callback != NULL) {
        g_user_callback(byte);   // Aquí se ejecuta TU función de aplicación
    }
}
```

Esto es exactamente cómo funcionan `GPIO_PinInterruptCallbackInstall` en NXP y `gpio_isr_handler_add` en ESP-IDF por debajo.

### Ejercicio Didáctico 0.2
Diseña (en papel o pseudo-C) una estructura `sensor_driver_t` que contenga un puntero a función `read(void)` y un puntero a función `on_error(int code)`. El objetivo: que un mismo módulo de "lectura periódica" funcione con **cualquier sensor** (temperatura, distancia, etc.) sin que el módulo genérico sepa qué sensor es. Esto es tu primer contacto con **abstracción de hardware (HAL)**.

---

## 0.3 Structs y Uniones para Hardware

### Structs para telemetría (empaquetar datos lógicos)

```c
typedef struct {
    float    temperatura_c;
    float    humedad_pct;
    uint32_t timestamp_ms;
    uint8_t  sensor_id;
} telemetria_t;
```
Simple, pero nota: el **orden de los campos importa** por el *padding* del compilador (alineación de memoria). Si vas a transmitir esto por UART/SPI byte a byte, necesitas controlar el empaquetado:

```c
#pragma pack(push, 1)   // Fuerza que no haya relleno entre campos
typedef struct {
    uint8_t  sensor_id;
    uint32_t timestamp_ms;
    float    temperatura_c;
} __attribute__((packed)) telemetria_wire_t;  // GCC/ARM-GCC
#pragma pack(pop)
```

### Structs para mapear registros de hardware (Register Map)

Este es EL patrón que vas a ver en todos los headers `.h` de vendor (NXP, ST, Espressif). En vez de definir cada registro con su dirección a mano, se define un `struct` que coincide exactamente con el mapa de memoria del periférico:

```c
typedef struct {
    volatile uint32_t PDOR;   // Offset 0x00 - Port Data Output Register
    volatile uint32_t PSOR;   // Offset 0x04 - Set bits
    volatile uint32_t PCOR;   // Offset 0x08 - Clear bits
    volatile uint32_t PTOR;   // Offset 0x0C - Toggle bits
} GPIO_Type;

#define GPIOA  ((GPIO_Type *)0x40020000u)

// Ahora escribir a hardware se ve como acceso normal a struct:
GPIOA->PSOR = BIT(5);   // Enciende el pin 5
GPIOA->PCOR = BIT(5);   // Apaga el pin 5
```

**¿Por qué cada campo es `volatile` y no el struct completo?** Porque quieres que el compilador respete *cada acceso individual* al registro, no solo al puntero del struct.

### Uniones para reinterpretar datos crudos (ej: un frame de comunicación)

```c
typedef union {
    uint32_t raw;
    struct {
        uint32_t motor_id   : 4;   // 4 bits
        uint32_t direccion  : 1;   // 1 bit
        uint32_t velocidad  : 8;   // 8 bits
        uint32_t reservado  : 19;
    } campos;
} comando_motor_t;

comando_motor_t cmd;
cmd.raw = 0x00000A5;          // Llega un dato crudo por SPI/UART
// Accedes a los campos ya decodificados:
uint8_t vel = cmd.campos.velocidad;
```
Esto se llama **bitfield** dentro de una unión, y es el mecanismo real detrás de cómo los SDKs decodifican registros de configuración complejos (ej: registros de PWM con duty+prescaler+modo en un solo word de 32 bits).

### Ejercicio Didáctico 0.3
Diseña un `struct` `imu_config_t` que represente el registro de configuración de un acelerómetro (inventa: 2 bits para rango de escala, 3 bits para tasa de muestreo, 1 bit de habilitación). Escribe las funciones `imu_config_pack()` y `imu_config_unpack()` para convertir entre tu struct legible y el `uint8_t` crudo que se envía por I2C.

---

# FASE 1 — NXP FRDM-MCXA156 (Bare-Metal / MCUXpresso SDK)

**Principio de arquitectura que vas a aplicar en TODOS los ejercicios de esta fase:**

```
proyecto/
├── main.c                  <- Solo orquesta: init() + super-loop. CERO lógica de bits aquí.
├── drivers/
│   ├── motor_pwm.h / .c    <- Capa de abstracción de TU aplicación
│   ├── sensor_i2c.h / .c
├── (SDK de NXP: fsl_gpio.h, fsl_pwm.h, fsl_i2c.h — no se tocan, se usan)
```
La regla de oro: **`main.c` nunca debería tener una llamada directa a `fsl_*`**. Esas llamadas viven dentro de tus propios drivers (`motor_pwm.c`, etc.). Esto es lo que hace que tu código sea escalable: si mañana cambias de MCU, solo reescribes la capa `drivers/`, no `main.c`.

## 1.1 GPIO y Relojes

**Concepto clave — ¿por qué hay que "activar el reloj" antes de usar un pin?**
Los periféricos en un MCU están apagados por defecto para ahorrar energía. Antes de configurar CUALQUIER periférico (GPIO incluido, en muchas familias) debes habilitar su *clock gate* en el bus correspondiente. Si te saltas este paso, escribir al registro no hace nada (o cuelga el MCU), porque físicamente el periférico no está recibiendo señal de reloj.

**Multiplexación de pines (`PORT`):** un mismo pin físico puede ser GPIO, UART_TX, PWM, etc. El registro `PORT->PCR[n]` decide "qué función lógica" tiene ese pin físico. Esto es el "MUX".

### Funciones clave del SDK (fsl_gpio.h / fsl_clock.h / fsl_port.h)
```c
CLOCK_EnableClock(kCLOCK_GpioA);          // 1. Enciende el reloj del periférico
PORT_SetPinMux(PORTA, 5U, kPORT_MuxAsGpio); // 2. Configura el MUX del pin
gpio_pin_config_t config = {
    .pinDirection = kGPIO_DigitalOutput,
    .outputLogic  = 0U,
};
GPIO_PinInit(GPIOA, 5U, &config);          // 3. Configura dirección
GPIO_PinWrite(GPIOA, 5U, 1U);               // 4. Escribe el estado
```

### Ejercicio 1.1
**Objetivo:** Encender un LED con un botón, sin usar interrupciones (polling), aplicando separación de capas.
**Jerarquía:** `gpio_driver.h/.c` (funciones genéricas `gpio_led_on/off`, `gpio_button_read`) + `main.c` (solo el `while(1)`).
**Esqueleto:**
```c
// gpio_driver.h
#ifndef GPIO_DRIVER_H
#define GPIO_DRIVER_H
#include "fsl_gpio.h"

void gpio_driver_init(void);
void led_set(bool state);
bool button_is_pressed(void);

#endif
```
```c
// main.c
int main(void) {
    BOARD_InitHardware();      // Generado por MCUXpresso Config Tools
    gpio_driver_init();

    while (1) {
        led_set(button_is_pressed());
    }
}
```

---

## 1.2 Generación de PWM (Prioridad Alta)

**Concepto clave:** un PWM lo genera un **Timer** contando hasta un valor máximo (`periodo`) y comparando contra un valor de umbral (`duty`). Cuando el contador está por debajo del umbral, la salida está en alto; por encima, en bajo. Frecuencia = velocidad del contador / periodo. Duty cycle = umbral / periodo.

### Jerarquía de código
```
drivers/
├── motor_pwm.h   <- API: motor_pwm_init(), motor_pwm_set_duty(uint8_t pct)
├── motor_pwm.c   <- Usa fsl_pwm.h / fsl_ctimer.h internamente
main.c            <- Llama motor_pwm_set_duty(), nunca toca PWM_ registros
```

### Funciones clave (fsl_pwm.h — familia con módulo PWM dedicado; en MCXA se usa a menudo TPM/CTIMER, la lógica es análoga)
```c
pwm_config_t pwmConfig;
PWM_GetDefaultConfig(&pwmConfig);
PWM_Init(PWM0, kPWM_Module_0, &pwmConfig);

pwm_signal_param_t pwmSignal = {
    .pwmChannel       = kPWM_PwmA,
    .dutyCyclePercent = 25U,           // <- esto es lo que vas a variar
    .level            = kPWM_HighTrue,
};
PWM_SetupPwm(PWM0, kPWM_Module_0, &pwmSignal, 1U, kPWM_EdgeAligned, 5000U /*Hz*/, CLOCK_GetFreq(kCLOCK_BusClk));
PWM_StartTimer(PWM0, kPWM_Module_0);
```

### Ejercicio 1.2 — Control gradual de velocidad de un ventilador DC
**Objetivo:** Rampa de velocidad suave (0% → 100% → 0%) usando PWM, sin bloqueos duros basados en `delay` ciego (usa un contador de ciclos del super-loop).
**Jerarquía:**
```
drivers/motor_pwm.h/.c   -> motor_pwm_init(), motor_pwm_set_duty(uint8_t pct)
app/rampa_control.h/.c   -> lógica de "subir/bajar duty cada N ms" (capa de aplicación, NO sabe de PWM_ registros)
main.c                   -> orquesta
```
**Esqueleto:**
```c
// motor_pwm.h
void motor_pwm_init(uint32_t frecuencia_hz);
void motor_pwm_set_duty(uint8_t duty_pct);   // 0-100

// rampa_control.c
void rampa_actualizar(void) {
    static uint8_t duty = 0;
    static int8_t  paso = 1;
    duty += paso;
    if (duty >= 100 || duty == 0) paso = -paso;
    motor_pwm_set_duty(duty);
}
```
**Punto de reflexión que quiero que resuelvas tú:** ¿Qué pasa si `rampa_actualizar()` se llama demasiado rápido? ¿Cómo controlarías el tiempo entre pasos sin usar `vTaskDelay` (esto es bare-metal, no tienes RTOS todavía)? Pista: revisa el uso de un Timer de periodo fijo con su propia interrupción, o un contador de "ticks" del SysTick.

---

## 1.3 Protocolos de Comunicación (I2C / SPI)

**Diferencia conceptual clave:**
- **SPI**: full-duplex, síncrono, sin dirección (se selecciona el esclavo con un pin físico `CS`). Más rápido, más pines.
- **I2C**: half-duplex (en la práctica), síncrono, con dirección de 7 bits sobre 2 líneas (`SDA`/`SCL`). Menos pines, más lento, permite múltiples esclavos en el mismo bus sin más cableado.

### Jerarquía
```
drivers/
├── rfid_spi.h/.c    <- API: rfid_read_uid(uint8_t *buffer)
├── lcd_i2c.h/.c      <- API: lcd_print(const char *texto)
```

### SPI — Esqueleto (fsl_spi.h)
```c
spi_master_config_t spiConfig;
SPI_MasterGetDefaultConfig(&spiConfig);
spiConfig.baudRate_Bps = 500000U;
SPI_MasterInit(SPI0, &spiConfig, CLOCK_GetFreq(kCLOCK_BusClk));

spi_transfer_t xfer = {
    .txData = txBuffer,
    .rxData = rxBuffer,
    .dataSize = 4,
};
SPI_MasterTransferBlocking(SPI0, &xfer);
```

### I2C — Esqueleto (fsl_i2c.h)
```c
i2c_master_config_t i2cConfig;
I2C_MasterGetDefaultConfig(&i2cConfig);
I2C_MasterInit(I2C0, &i2cConfig, CLOCK_GetFreq(kCLOCK_BusClk));

i2c_master_transfer_t masterXfer = {
    .slaveAddress   = 0x27,             // dirección típica de un módulo LCD I2C
    .direction      = kI2C_Write,
    .subaddress     = 0x00,
    .subaddressSize = 0,
    .data           = data,
    .dataSize       = len,
};
I2C_MasterTransferBlocking(I2C0, &masterXfer);
```

### Ejercicio 1.3
**Objetivo:** Elige UNO: (a) leer el UID de un RFID por SPI, (b) escribir un string en LCD por I2C.
**Requisito de diseño:** tu función pública (`rfid_read_uid()` o `lcd_print()`) debe retornar un código de error (`enum`), no un `void` ciego. Esto te obliga a pensar en manejo de fallos de comunicación desde ya (timeout, NACK, etc.) — hábito de ingeniero, no de estudiante.

---

## 1.4 Interrupciones (NVIC)

**Concepto clave — ¿por qué la ISR debe ser corta?**
Mientras el procesador está dentro de una ISR, generalmente las interrupciones de igual o menor prioridad están bloqueadas (según configuración). Si tu ISR hace trabajo pesado (ej: procesar un frame completo de datos), estás bloqueando el resto del sistema — incluyendo el super-loop y otras interrupciones críticas. La ISR debe: **capturar el dato/evento, levantar una bandera, y salir.** El procesamiento real se hace en el `main()` revisando esa bandera.

```c
volatile bool g_boton_presionado = false;   // volatile: la ISR la modifica

void PORTA_IRQHandler(void) {
    GPIO_PortClearInterruptFlags(GPIOA, BIT(5));  // Limpiar flag de hardware SIEMPRE
    g_boton_presionado = true;                     // Solo levantar bandera
}

int main(void) {
    EnableIRQ(PORTA_IRQn);
    while (1) {
        if (g_boton_presionado) {
            g_boton_presionado = false;
            // Procesamiento real AQUÍ, fuera de la ISR
        }
    }
}
```

**Advertencia de novato común (para que la razones si te pasa):** si tu ISR nunca se vuelve a disparar después de la primera vez, ¿qué registro específico olvidaste tocar? (Pista: no es la bandera de tu variable — es un registro de hardware que "reconoce" la interrupción atendida.)

### Ejercicio 1.4
Combina 1.2 + 1.4: controla el duty cycle del PWM del ventilador con un botón de interrupción (cada pulsación sube 25%, cíclico). Jerarquía: `motor_pwm.h/.c` + `boton_isr.h/.c` (bandera) + `main.c` (conecta ambos).

---

# FASE 2 — ESP32 (ESP-IDF / FreeRTOS)

**Cambio de paradigma importante:** en Bare-Metal tenías UN flujo de ejecución (`main` + ISRs). En FreeRTOS tienes **múltiples "hilos" lógicos (tareas)** que el *Scheduler* reparte en el tiempo de CPU. Tu `while(1)` dentro de una tarea NO bloquea el sistema porque `vTaskDelay()` le dice al scheduler "quítame de la CPU por N ticks y dale el turno a otra tarea" — a diferencia de un `delay()` bare-metal que sí congela todo.

### Jerarquía general recomendada en ESP-IDF
```
main/
├── main.c              <- Solo crea tareas (xTaskCreate) e inicializa periféricos globales
├── tasks/
│   ├── task_sensor.h/.c
│   ├── task_motor.h/.c
├── drivers/
│   ├── motor_driver.h/.c   <- Envuelve driver/ledc.h, driver/gpio.h
│   ├── sensor_driver.h/.c  <- Envuelve driver/i2c.h, driver/adc.h
```

## 2.1 Fundamentos de FreeRTOS

```c
void task_led(void *pvParameters) {
    while (1) {
        gpio_set_level(LED_PIN, 1);
        vTaskDelay(pdMS_TO_TICKS(500));   // Cede la CPU 500ms — NO bloquea el sistema
        gpio_set_level(LED_PIN, 0);
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

void app_main(void) {
    xTaskCreate(task_led, "task_led", 2048 /*stack*/, NULL, 5 /*prioridad*/, NULL);
}
```

**Por qué `vTaskDelay` no es un `delay()` bare-metal:** un `delay()` clásico ejecuta un bucle vacío consumiendo CPU activamente (busy-wait). `vTaskDelay` mueve la tarea al estado **Blocked**, y el Scheduler literalmente ejecuta OTRA tarea (o entra en modo bajo consumo) durante esos ticks — la CPU no se "desperdicia".

**Prioridades:** un número mayor = mayor prioridad en el scheduler de FreeRTOS por defecto en ESP-IDF (a diferencia de algunas otras implementaciones — siempre verifica en la documentación del puerto específico). Una tarea de prioridad más alta que esté "Ready" siempre interrumpe a una de menor prioridad que esté corriendo (preemption).

### Ejercicio 2.1
Crea 2 tareas con distinta prioridad que impriman su nombre por UART/monitor cada cierto tiempo. Usa `vTaskDelay` con tiempos distintos. **Reto de razonamiento:** ¿qué pasa si dentro de una tarea usas `while(1) {}` sin ningún `vTaskDelay`? (Ejecútalo y observa — luego explica con tus palabras por qué el *Watchdog Timer* del ESP32 puede reiniciar la placa.)

---

## 2.2 Sincronización: Colas, Semáforos y Mutex

**¿Cuándo usar cada uno?**
- **Queue (Cola):** para *pasar datos* entre tareas de forma segura (ej: un valor de sensor).
- **Semáforo binario:** para *señalizar un evento* entre tareas o entre una ISR y una tarea (ej: "llegó un dato, procésalo").
- **Mutex:** para *proteger un recurso compartido* de acceso simultáneo (ej: dos tareas que quieren escribir en el mismo bus I2C — evita condiciones de carrera).

### Jerarquía del ejercicio
```
tasks/task_sensor.c   -> Lee el sensor, hace xQueueSend()
tasks/task_procesa.c  -> Hace xQueueReceive(), procesa el dato
main.c                -> Crea la QueueHandle_t global (o inyectada por parámetro) y ambas tareas
```

### Esqueleto
```c
QueueHandle_t g_cola_sensor;   // Declarado antes de app_main, o pasado como parámetro

void task_sensor(void *pv) {
    float lectura;
    while (1) {
        lectura = leer_adc();              // Tu driver
        xQueueSend(g_cola_sensor, &lectura, portMAX_DELAY);
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}

void task_procesador(void *pv) {
    float dato;
    while (1) {
        if (xQueueReceive(g_cola_sensor, &dato, portMAX_DELAY) == pdTRUE) {
            // Procesar "dato" aquí — esta tarea NUNCA toca el ADC directamente
        }
    }
}

void app_main(void) {
    g_cola_sensor = xQueueCreate(5, sizeof(float));
    xTaskCreate(task_sensor, "sensor", 2048, NULL, 5, NULL);
    xTaskCreate(task_procesador, "procesador", 2048, NULL, 4, NULL);
}
```

### Ejercicio 2.2
Implementa exactamente el esqueleto de arriba, pero con un sensor real (I2C o ADC). **Requisito de diseño:** `task_sensor` no debe conocer NADA sobre qué hace `task_procesador` con el dato — deben estar completamente desacopladas, comunicándose solo por la Queue.

---

## 2.3 PWM Avanzado (LEDC / MCPWM)

**LEDC** (LED Control, reutilizado para PWM genérico) es ideal para PWM simple (motores DC, brillo de LED). **MCPWM** está diseñado específicamente para control de motores (con detección de falla, complementary outputs para puentes H, etc.) — úsalo cuando necesites control más sofisticado (ej: motores BLDC).

### Jerarquía
```
drivers/motor_hbridge.h/.c   -> API: motor_set(int8_t velocidad)  // -100 a 100, signo = dirección
```

### Esqueleto (driver/ledc.h)
```c
ledc_timer_config_t timer_conf = {
    .speed_mode      = LEDC_LOW_SPEED_MODE,
    .duty_resolution = LEDC_TIMER_10_BIT,
    .timer_num       = LEDC_TIMER_0,
    .freq_hz         = 5000,
};
ledc_timer_config(&timer_conf);

ledc_channel_config_t channel_conf = {
    .gpio_num   = MOTOR_PWM_PIN,
    .speed_mode = LEDC_LOW_SPEED_MODE,
    .channel    = LEDC_CHANNEL_0,
    .timer_sel  = LEDC_TIMER_0,
    .duty       = 0,
};
ledc_channel_config(&channel_conf);

// En tu driver, para variar velocidad:
ledc_set_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, duty_value);
ledc_update_duty(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0);
```

### Ejercicio 2.3 — Puente H para vehículo RC
**Objetivo:** `motor_set(int8_t velocidad)` donde el signo controla 2 pines de dirección (IN1/IN2) y la magnitud controla el duty del PWM (LEDC).
**Diseño a resolver tú:** ¿Cómo evitas el caso "peligroso" en un puente H donde IN1 e IN2 están en alto simultáneamente (short-brake/cortocircuito según el driver)? Escribe la tabla de verdad de tu función ANTES de codificar.

---

## 2.4 Protocolos y Conectividad — UART y Telemetría

### Jerarquía (el ejercicio más "completo" del plan)
```
main/
├── main.c
├── tasks/
│   ├── task_uart_rx.c     -> Lee UART, parsea comando, envía por Queue de comandos
│   ├── task_motor.c       -> Recibe de la Queue de comandos, llama motor_set()
├── drivers/
│   ├── motor_hbridge.h/.c
│   ├── uart_comm.h/.c     -> Envuelve driver/uart.h
```

### Esqueleto UART (driver/uart.h)
```c
uart_config_t uart_config = {
    .baud_rate = 115200,
    .data_bits = UART_DATA_8_BITS,
    .parity    = UART_PARITY_DISABLE,
    .stop_bits = UART_STOP_BITS_1,
};
uart_param_config(UART_NUM_1, &uart_config);
uart_set_pin(UART_NUM_1, TX_PIN, RX_PIN, UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE);
uart_driver_install(UART_NUM_1, 1024, 0, 0, NULL, 0);

// Dentro de task_uart_rx:
uint8_t data[32];
int len = uart_read_bytes(UART_NUM_1, data, sizeof(data), pdMS_TO_TICKS(100));
```

### Ejercicio 2.4 — Sistema de control remoto por comandos
**Objetivo:** Recibir por UART strings tipo `"M:50\n"` (motor al 50%) o `"D:-1\n"` (dirección reversa), parsearlos en `task_uart_rx`, y enviar una `comando_motor_t` (revisa tu ejercicio de la Fase 0.3 — reutiliza esa unión/struct) por Queue a `task_motor`.

**Pregunta de arquitectura para que respondas tú antes de codificar:** ¿por qué el parseo del string debe ocurrir en `task_uart_rx` y NO dentro de `task_motor`? (Pista: piensa en qué pasaría si mañana cambias la fuente de comandos de UART a Bluetooth — ¿cuántos archivos tendrías que tocar en cada diseño?)

---

# Cómo evaluar tu propio progreso (Criterio de Salida de cada Fase)

| Fase | Sabes que dominas el tema cuando... |
|---|---|
| 0 | Puedes leer un registro de un datasheet real (ej: Reference Manual de la MCX) y escribir el código bitwise para configurarlo SIN mirar ejemplos. |
| 1 | Puedes crear un nuevo driver bare-metal (`.h`/`.c`) para un periférico que nunca has usado, solo leyendo el Reference Manual y el header `fsl_*.h` correspondiente. |
| 2 | Puedes diseñar la arquitectura de tareas/colas de un sistema ANTES de escribir código (en un diagrama), y ese diseño no cambia significativamente cuando lo implementas. |

Cuando termines un ejercicio, tráemelo. No te voy a corregir el código de inmediato — te voy a señalar la línea donde tu lógica bitwise, tu inicialización de reloj, o tu jerarquía de capas está fallando, para que lo razones tú mismo.
