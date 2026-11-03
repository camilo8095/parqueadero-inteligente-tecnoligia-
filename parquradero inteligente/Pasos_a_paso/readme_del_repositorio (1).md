# 🚗 System Bulls - Parqueadero Inteligente ESP32

Sistema de gestión de estacionamientos totalmente automatizado e independiente desarrollado sobre la plataforma **ESP32 DevKit**. El sistema integra control de accesos mediante **RFID**, monitoreo magnético de plazas con **GY-271**, control de barrera física con **Servomotor SG90**, pantalla **LCD 1602A** y servidores web locales (**Panel de Administración** y **PWA de Cliente**).

---

## 📁 Estructura del Repositorio

```text
PARQUEADERO-INTELIGENTE/
│
├── README.md                           # Presentación del proyecto y guía rápida
├── .gitignore                          # Filtro de archivos para Git
│
├── DOCUMENTACION/
│   └── Manual_Tecnico_Parqueadero_Inteligente.html  # Documento técnico completo
│
├── CODIGO/
│   ├── parqueadero_inteligente.ino     # Firmware principal del microcontrolador ESP32
│   ├── web_admin.h                      # Interfaz web de gestión del administrador
│   └── web_app.h                        # Aplicación web móvil (PWA) para clientes
│
└── HARDWARE/
    └── Conexiones_y_Componentes.md     # Tabla de asignación de pines y guía eléctrica
```

---

## 🛠️ Componentes Utilizados

| Componente | Función Principal | Conexión / Bus |
| :--- | :--- | :--- |
| **ESP32 DevKit** | Unidad Central de Procesamiento y Servidor Web HTTP | Micro-USB / 5V |
| **RFID RC522** | Lectura de tarjetas y llaveros de usuarios (13.56 MHz) | SPI (GPIOs 5, 18, 19, 23, 4) |
| **GY-271 (Magnetómetro)** | Detección de vehículo en Puesto 1 por campo magnético | I2C (SDA 21, SCL 22) |
| **Servo SG90** | Mecanismo de apertura/cierre de la barrera de acceso | PWM (GPIO 13) |
| **LCD 1602A v2.0** | Display de bienvenida, cobro, tarifas y estado de cupos | Paralelo 4 bits (GPIOs 27, 26, 25, 33, 32, 14) |

---

## 🚀 Guía de Inicio Rápido

1. **Requisitos de Software:**
   - [Arduino IDE 2.x](https://www.arduino.cc/en/software).
   - Paquete de placas `esp32` de Espressif Systems (v3.x).
   - Librerías: `MFRC522`, `ESP32Servo`, `ArduinoJson` (v7), `LiquidCrystal`.

2. **Carga del Programa:**
   - Abre `CODIGO/parqueadero_inteligente.ino` en el IDE de Arduino.
   - Selecciona la placa **ESP32 Dev Module**.
   - Conecta el ESP32 por USB y presiona **Subir**.

3. **Acceso a las Aplicaciones Web:**
   - Conecta tu celular a la red WiFi emitida por el ESP32: **`Parqueadero-ESP32`** (Clave: `parqueo123`).
   - **Panel del Administrador:** `http://192.168.4.1/admin` (Usuario: `admin` | Clave: `admin123`).
   - **App del Cliente:** `http://192.168.4.1/`.