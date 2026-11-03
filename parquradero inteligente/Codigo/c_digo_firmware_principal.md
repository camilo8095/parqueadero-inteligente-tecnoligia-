/*
  ===========================================================================
  PARQUEADERO INTELIGENTE - ESP32 (System Bulls)
  ===========================================================================
  Hardware:
    - ESP32 DevKit (30 o 38 pines)
    - LCD 1602A v2.0 en modo paralelo de 4 bits (RW a GND)
    - GY-271 magnetometro (QMC5883L, QMC5883P o HMC5883L; autodetectado)
    - Servo SG90 (barrera de entrada/salida)
    - Lector RFID RC522 (tarjetas y llaveros de 13,56 MHz)

  Paginas web (servidas por el ESP32):
    - http://<IP-del-ESP32>/        -> App del cliente (PWA)
    - http://<IP-del-ESP32>/admin   -> Panel del administrador
  ===========================================================================
*/

#include <WiFi.h>
#include <WebServer.h>
#include <ESPmDNS.h>
#include <Preferences.h>
#include <LittleFS.h>
#include <Wire.h>
#include <SPI.h>
#include <LiquidCrystal.h>
#include <MFRC522.h>
#include <ESP32Servo.h>
#include <ArduinoJson.h>
#include <time.h>
#include <vector>

#include "web_admin.h"
#include "web_app.h"

// ===================== CONFIGURACION =====================
const char* WIFI_SSID   = "";                   // Red WiFi local (opcional)
const char* WIFI_PASS   = "";
const char* AP_SSID     = "Parqueadero-ESP32";  // Red propia del ESP32
const char* AP_PASS     = "parqueo123";         // Clave AP (minimo 8 caracteres)

const char* ADMIN_USER  = "admin";
const char* ADMIN_PASS  = "admin123";
const char* NOMBRE_MDNS = "parqueadero";

// ===================== ASIGNACION DE PINES =====================
const int LCD_RS = 27, LCD_E = 26, LCD_D4 = 25, LCD_D5 = 33, LCD_D6 = 32, LCD_D7 = 14;
const int RFID_SCK = 18, RFID_MISO = 19, RFID_MOSI = 23, RFID_SS = 5, RFID_RST = 4;
const int I2C_SDA = 21, I2C_SCL = 22;
const int SERVO_PIN = 13;

// ===================== PARÁMETROS Y OBJETOS =====================
const int SERVO_CERRADO = 0;
const int SERVO_ABIERTO = 90;
const unsigned long BARRERA_MS = 4000;
const size_t MAX_CLIENTES  = 50;
const size_t MAX_HISTORIAL = 40;
const uint32_t EPOCH_VALIDO = 1700000000UL;

LiquidCrystal lcd(LCD_RS, LCD_E, LCD_D4, LCD_D5, LCD_D6, LCD_D7);
MFRC522 rfid(RFID_SS, RFID_RST);
Servo barrera;
WebServer server(80);
Preferences prefs;

struct Cliente {
  String uid, nombre, placa, pin;
  bool dentro = false;
  uint32_t entrada = 0;
};

struct Registro {
  String nombre, placa;
  uint32_t entrada = 0, salida = 0, costo = 0;
};

std::vector<Cliente> clientes;
std::vector<Registro> historial;

uint32_t tarifaMinuto = 150;
uint8_t  cupos        = 5;
uint16_t umbralMag    = 150;

int64_t relojDesfase = 0;
bool relojValido = false;

uint8_t magDir = 0;
enum TipoMag { MAG_NINGUNO, MAG_QMC5883L, MAG_QMC5883P, MAG_HMC5883L } magTipo = MAG_NINGUNO;
int16_t magX = 0, magY = 0, magZ = 0;
float baseX = 0, baseY = 0, baseZ = 0;
float desvio = 0;
bool puestoOcupado = false;
unsigned long cambioDesde = 0, ultimaLecturaMag = 0;

bool barreraAbierta = false;
unsigned long barreraHasta = 0;

String lcdL1, lcdL2;
unsigned long mensajeHasta = 0, ultimoRefrescoLcd = 0;

String ultimaTarjeta;
unsigned long ultimaTarjetaMs = 0;
String tarjetaDesconocida;
uint32_t tarjetaDesconocidaHora = 0;
String ultimoEvento = "Sistema iniciado";

// ===================== FUNCIONES DE RELOJ Y UTILIDADES =====================
uint32_t ahora() {
  return (uint32_t)(relojDesfase + (int64_t)(esp_timer_get_time() / 1000000LL));
}

void fijarHora(uint32_t epoch) {
  if (relojValido || epoch < EPOCH_VALIDO) return;
  int64_t nuevo = (int64_t)epoch - (int64_t)(esp_timer_get_time() / 1000000LL);
  int64_t delta = nuevo - relojDesfase;
  relojDesfase = nuevo;
  relojValido = true;
  for (auto& c : clientes) if (c.dentro && c.entrada < EPOCH_VALIDO) c.entrada += delta;
  for (auto& r : historial) {
    if (r.entrada < EPOCH_VALIDO) r.entrada += delta;
    if (r.salida  < EPOCH_VALIDO) r.salida  += delta;
  }
  Serial.printf("Reloj sincronizado: %lu\n", (unsigned long)epoch);
}

String horaCorta() {
  if (!relojValido) return "--:--";
  time_t t = ahora();
  struct tm tm;
  localtime_r(&t, &tm);
  char b[6];
  strftime(b, sizeof b, "%H:%M", &tm);
  return b;
}

String sinTildes(String s) {
  const char* de[]  = {"á","é","í","ó","ú","Á","É","Í","Ó","Ú","ñ","Ñ","ü","Ü"};
  const char* a[]   = {"a","e","i","o","u","A","E","I","O","U","n","N","u","U"};
  for (int i = 0; i < 14; i++) s.replace(de[i], a[i]);
  return s;
}

String pad16(String s) {
  s = sinTildes(s);
  if (s.length() > 16) s = s.substring(0, 16);
  while (s.length() < 16) s += ' ';
  return s;
}

String normalizarPlaca(String p) {
  p.trim(); p.toUpperCase(); p.replace(" ", ""); p.replace("-", "");
  return p;
}

String pesos(uint32_t v) {
  String s = String(v), r;
  int n = s.length();
  for (int i = 0; i < n; i++) {
    r += s[i];
    int faltan = n - i - 1;
    if (faltan > 0 && faltan % 3 == 0) r += '.';
  }
  return "$" + r;
}

String duracionTexto(uint32_t seg) {
  char b[17];
  snprintf(b, sizeof b, "%luh %02lum", (unsigned long)(seg / 3600), (unsigned long)((seg % 3600) / 60));
  return b;
}

uint32_t calcularCosto(uint32_t seg) {
  uint32_t minutos = (seg + 59) / 60;
  if (minutos == 0) minutos = 1;
  return minutos * tarifaMinuto;
}

int clientesDentro() {
  int n = 0;
  for (auto& c : clientes) if (c.dentro) n++;
  return n;
}

int cuposLibres() {
  int l = (int)cupos - clientesDentro();
  return l < 0 ? 0 : l;
}

int buscarUid(const String& uid) {
  for (size_t i = 0; i < clientes.size(); i++) if (clientes[i].uid == uid) return i;
  return -1;
}

int buscarPlaca(const String& placa) {
  for (size_t i = 0; i < clientes.size(); i++) if (clientes[i].placa == placa) return i;
  return -1;
}

// ===================== BARRERA Y LCD =====================
void abrirBarrera() {
  barrera.write(SERVO_ABIERTO);
  barreraAbierta = true;
  barreraHasta = millis() + BARRERA_MS;
}

void atenderBarrera() {
  if (barreraAbierta && (long)(millis() - barreraHasta) >= 0) {
    barrera.write(SERVO_CERRADO);
    barreraAbierta = false;
  }
}

void lcdMostrar(const String& a, const String& b) {
  String A = pad16(a), B = pad16(b);
  if (A == lcdL1 && B == lcdL2) return;
  lcdL1 = A; lcdL2 = B;
  lcd.setCursor(0, 0); lcd.print(A);
  lcd.setCursor(0, 1); lcd.print(B);
}

void lcdMensaje(const String& a, const String& b, unsigned long ms) {
  lcdMostrar(a, b);
  mensajeHasta = millis() + ms;
}

void lcdReposo() {
  int libres = cuposLibres();
  String l1 = "Libres " + String(libres) + "/" + String(cupos);
  while (l1.length() < 11) l1 += ' ';
  l1 = l1.substring(0, 11) + horaCorta();
  lcdMostrar(l1, libres > 0 ? "Acerque tarjeta" : "LLENO");
}

// ===================== MAGNETÓMETRO GY-271 =====================
void escribirReg(uint8_t reg, uint8_t val) {
  Wire.beginTransmission(magDir);
  Wire.write(reg); Wire.write(val);
  Wire.endTransmission();
}

bool existeI2C(uint8_t dir) {
  Wire.beginTransmission(dir);
  return Wire.endTransmission() == 0;
}

bool magIniciar() {
  if (existeI2C(0x0D)) {
    magDir = 0x0D; magTipo = MAG_QMC5883L;
    escribirReg(0x0B, 0x01); escribirReg(0x09, 0x1D);
  } else if (existeI2C(0x2C)) {
    magDir = 0x2C; magTipo = MAG_QMC5883P;
    escribirReg(0x29, 0x06); escribirReg(0x0B, 0x08); escribirReg(0x0A, 0xCD);
  } else if (existeI2C(0x1E)) {
    magDir = 0x1E; magTipo = MAG_HMC5883L;
    escribirReg(0x00, 0x70); escribirReg(0x01, 0x20); escribirReg(0x02, 0x00);
  } else {
    magTipo = MAG_NINGUNO;
    return false;
  }
  return true;
}

bool magLeer(int16_t& x, int16_t& y, int16_t& z) {
  if (magTipo == MAG_NINGUNO) return false;
  uint8_t reg = (magTipo == MAG_QMC5883L) ? 0x00 : (magTipo == MAG_QMC5883P ? 0x01 : 0x03);
  Wire.beginTransmission(magDir);
  Wire.write(reg);
  if (Wire.endTransmission(false) != 0) return false;
  if (Wire.requestFrom((int)magDir, 6) != 6) return false;
  uint8_t b[6];
  for (int i = 0; i < 6; i++) b[i] = Wire.read();
  if (magTipo == MAG_HMC5883L) {
    x = (int16_t)(b[0] << 8 | b[1]); z = (int16_t)(b[2] << 8 | b[3]); y = (int16_t)(b[4] << 8 | b[5]);
  } else {
    x = (int16_t)(b[1] << 8 | b[0]); y = (int16_t)(b[3] << 8 | b[2]); z = (int16_t)(b[5] << 8 | b[4]);
  }
  return true;
}

bool calibrarSensor() {
  if (magTipo == MAG_NINGUNO) return false;
  float sx = 0, sy = 0, sz = 0; int n = 0;
  for (int i = 0; i < 40; i++) {
    int16_t x, y, z;
    if (magLeer(x, y, z)) { sx += x; sy += y; sz += z; n++; }
    delay(20);
  }
  if (n == 0) return false;
  baseX = sx / n; baseY = sy / n; baseZ = sz / n;
  desvio = 0; puestoOcupado = false;
  ultimoEvento = "Sensor calibrado";
  return true;
}

void atenderSensor() {
  if (magTipo == MAG_NINGUNO || millis() - ultimaLecturaMag < 100) return;
  ultimaLecturaMag = millis();
  int16_t x, y, z;
  if (!magLeer(x, y, z)) return;
  magX = x; magY = y; magZ = z;
  float d = sqrtf((x - baseX) * (x - baseX) + (y - baseY) * (y - baseY) + (z - baseZ) * (z - baseZ));
  desvio = desvio * 0.7f + d * 0.3f;
  bool ocupado = puestoOcupado ? (desvio > umbralMag * 0.6f) : (desvio > umbralMag);
  if (ocupado != puestoOcupado) {
    if (cambioDesde == 0) cambioDesde = millis();
    else if (millis() - cambioDesde > 1500) {
      puestoOcupado = ocupado;
      cambioDesde = 0;
      ultimoEvento = puestoOcupado ? "Vehiculo detectado en puesto 1" : "Puesto 1 libre";
      Serial.println(ultimoEvento);
    }
  } else {
    cambioDesde = 0;
  }
}

// ===================== PROCESAMIENTO RFID =====================
void registrarSalida(Cliente& c, bool abrir) {
  uint32_t t = ahora();
  uint32_t seg = (t >= c.entrada) ? t - c.entrada : 0;
  uint32_t costo = calcularCosto(seg);
  Registro r;
  r.nombre = c.nombre; r.placa = c.placa; r.entrada = c.entrada; r.salida = t; r.costo = costo;
  historial.push_back(r);
  while (historial.size() > MAX_HISTORIAL) historial.erase(historial.begin());
  c.dentro = false; c.entrada = 0;
  ultimoEvento = "Salida " + c.placa + " " + duracionTexto(seg) + " " + pesos(costo);
  if (abrir) {
    lcdMensaje("Tiempo " + duracionTexto(seg), "Pagar " + pesos(costo), 6000);
    abrirBarrera();
  }
}

void procesarTarjeta(const String& uid) {
  Serial.println("Tarjeta RFID leida: " + uid);
  int i = buscarUid(uid);
  if (i < 0) {
    tarjetaDesconocida = uid;
    tarjetaDesconocidaHora = ahora();
    ultimoEvento = "Tarjeta no registrada " + uid;
    lcdMensaje("Tarjeta no", "registrada", 3000);
    return;
  }
  Cliente& c = clientes[i];
  if (!c.dentro) {
    if (cuposLibres() == 0) {
      ultimoEvento = "Entrada rechazada (lleno) " + c.placa;
      lcdMensaje("Parqueadero", "LLENO", 3000);
      return;
    }
    c.dentro = true; c.entrada = ahora();
    ultimoEvento = "Entrada " + c.placa;
    lcdMensaje("Bienvenido", c.nombre, 4000);
    abrirBarrera();
  } else {
    registrarSalida(c, true);
  }
}

void atenderRFID() {
  if (!rfid.PICC_IsNewCardPresent() || !rfid.PICC_ReadCardSerial()) return;
  String uid;
  for (byte i = 0; i < rfid.uid.size; i++) {
    if (rfid.uid.uidByte[i] < 0x10) uid += '0';
    uid += String(rfid.uid.uidByte[i], HEX);
  }
  uid.toUpperCase();
  rfid.PICC_HaltA(); rfid.PCD_StopCrypto1();
  if (uid == ultimaTarjeta && millis() - ultimaTarjetaMs < 5000) return;
  ultimaTarjeta = uid; ultimaTarjetaMs = millis();
  procesarTarjeta(uid);
}

// ===================== SETUP Y LOOP PRINCIPAL =====================
void setup() {
  Serial.begin(115200);
  lcd.begin(16, 2);
  lcdMostrar("Parqueadero", "Iniciando...");

  ESP32PWM::allocateTimer(0);
  barrera.setPeriodHertz(50);
  barrera.attach(SERVO_PIN, 500, 2400);
  barrera.write(SERVO_CERRADO);

  SPI.begin(RFID_SCK, RFID_MISO, RFID_MOSI, RFID_SS);
  rfid.PCD_Init();

  Wire.begin(I2C_SDA, I2C_SCL);
  magIniciar();

  WiFi.mode(WIFI_AP_STA);
  WiFi.softAP(AP_SSID, AP_PASS);
  Serial.println("Red AP activa: " + String(AP_SSID) + " IP: " + WiFi.softAPIP().toString());

  lcdMostrar("Sistema listo", WiFi.softAPIP().toString());
  delay(2000);
}

void loop() {
  server.handleClient();
  atenderRFID();
  atenderSensor();
  atenderBarrera();

  if ((long)(millis() - mensajeHasta) >= 0 && millis() - ultimoRefrescoLcd > 500) {
    ultimoRefrescoLcd = millis();
    lcdReposo();
  }
}