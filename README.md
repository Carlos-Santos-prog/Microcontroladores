# ⚡ Microcontroladores ITLA — Prácticas de Laboratorio Embebido

Repositorio académico para la asignatura de **Microcontroladores** en el **ITLA**, enfocado en el desarrollo de prácticas de laboratorio en **C/C++** orientadas a hardware real.

## 🛠️ Tecnologías y herramientas

- **Lenguajes:** C y C++
- **Placas objetivo:** NXP **FRDM-MCXA156** y **ESP32**
- **SDKs / Frameworks:** **MCUXpresso SDK** y **ESP-IDF**
- **Build system:** **CMake**
- **Entorno de desarrollo:** **Visual Studio Code**
- **Control de versiones:** **Git y GitHub**

## 📁 Estructura recomendada del repositorio

```text
Microcontroladores/
├── README.md
├── practicas/
│   ├── 01-introduccion-gpio/
│   │   ├── nxp-frdm-mcxa156/
│   │   └── esp32/
│   ├── 02-interrupciones-timers/
│   │   ├── nxp-frdm-mcxa156/
│   │   └── esp32/
│   └── 03-comunicacion-uart/
│       ├── nxp-frdm-mcxa156/
│       └── esp32/
└── docs/
    └── guias/
```

> Sugerencia: cada práctica debe incluir su propio `README.md` con objetivos, conexión de hardware, pasos de compilación y evidencias.

## 🚀 Compilación y flasheo (básico)

### NXP FRDM-MCXA156 (MCUXpresso SDK)

1. Configura un proyecto con el **MCUXpresso SDK** para la placa.
2. Desde la carpeta del proyecto, genera build con CMake:
   ```bash
   cmake -S . -B build
   cmake --build build
   ```
3. Flashea el binario con la herramienta configurada en tu entorno (por ejemplo, desde VS Code o utilidades de NXP/OpenOCD).

### ESP32 (ESP-IDF)

1. Abre terminal en la carpeta del proyecto.
2. Configura target (si aplica) y compila:
   ```bash
   idf.py set-target esp32
   idf.py build
   ```
3. Flashea y monitorea:
   ```bash
   idf.py -p <PUERTO_SERIAL> flash monitor
   ```
