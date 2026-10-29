Codigo:
/*
 * ==========================================================================
 *  LLENADO AUTOMÁTICO DE ESTANQUE CON FLOTADOR P35  -  ESP32 + MÓDULO RELÉ
 * ==========================================================================
 *
 *  QUÉ HACE
 *  --------
 *  - El ÚNICO sensor es el flotador P35 dentro del estanque.
 *  - Cuando el estanque se VACÍA (flotador abajo)  -> el ESP32 activa el relé
 *    y el relé enciende la bomba de 220V.
 *  - Cuando el estanque se LLENA (flotador arriba) -> el ESP32 desactiva el
 *    relé y la bomba se detiene.
 *  - El ESP32 nunca toca los 220V: solo manda la señal al pin IN del relé.
 *
 *  LIBRERÍAS
 *  ---------
 *  - Arduino.h : incluida en el paquete de placas "esp32 by Espressif
 *                Systems" del Arduino IDE. No hay que instalar nada más.
 *
 *  CONEXIONES (baja tensión)
 *  -------------------------
 *    Flotador P35, cable 1 -> GPIO 27
 *    Flotador P35, cable 2 -> GND      (usa la pull-up interna del ESP32)
 *    Relé VCC -> 5V (VIN) | Relé GND -> GND | Relé IN -> GPIO 26
 *
 *  CONEXIONES (220V, SIEMPRE CON LA ENERGÍA CORTADA)
 *  -------------------------------------------------
 *    Fase (L) -> COM del relé
 *    NO del relé -> Fase de la bomba
 *    Neutro y tierra -> directo a la bomba
 *
 *  LÓGICA (máquina de estados)
 *  ---------------------------
 *    APAGADA ──(flotador marca VACÍO)──> ESPERANDO_ARRANQUE
 *    ESPERANDO_ARRANQUE ──(sigue vacío "retardoArranqueS")──> ENCENDIDA
 *    ENCENDIDA ──(flotador marca LLENO)──> ESPERANDO_CORTE
 *    ESPERANDO_CORTE ──(sigue lleno "retardoCorteS")──> DESCANSO
 *    DESCANSO ──(pasa "descansoMinimoS")──> APAGADA
 *
 *    Los retardos evitan que la bomba prenda y apague muchas veces seguidas
 *    cuando el agua se mueve y el flotador oscila.
 * ==========================================================================
 */

#include <Arduino.h>


// ==========================================================================
// 1. PARÁMETROS DEL ESTANQUE   <<<<<  ESTO ES LO ÚNICO QUE MODIFICAS  >>>>>
// ==========================================================================

const char*    NOMBRE_ESTANQUE   = "Estanque principal";

const float    CAPACIDAD_LITROS  = 1000.0;  // Tamaño del estanque (litros)

const uint32_t RETARDO_ARRANQUE_S = 10;     // Segundos que el flotador debe marcar
                                            // VACÍO antes de ENCENDER la bomba

const uint32_t RETARDO_CORTE_S    = 0;      // Segundos que la bomba sigue funcionando
                                            // después de que el flotador marca LLENO
                                            // (0 = corta inmediatamente)

const uint32_t DESCANSO_MINIMO_S  = 60;     // Segundos mínimos con la bomba apagada
                                            // antes de permitir otro arranque
                                            // (protege el motor)

// Ejemplo para una botella de 1 litro:
//   CAPACIDAD_LITROS = 1.0;  RETARDO_ARRANQUE_S = 1;
//   RETARDO_CORTE_S  = 0;    DESCANSO_MINIMO_S  = 10;


// ==========================================================================
// 2. CONFIGURACIÓN DE HARDWARE   (se ajusta una sola vez)
// ==========================================================================

const uint8_t PIN_FLOTADOR = 27;   // Señal del flotador P35
const uint8_t PIN_RELE     = 26;   // Pin IN del módulo relé
const uint8_t PIN_LED      = 2;    // LED integrado: encendido = bomba funcionando

// La mayoría de los módulos relé se activan con señal LOW.
// Si el tuyo funciona al revés, cambia a false.
const bool RELE_ACTIVO_EN_BAJO = true;

// Qué significa que el contacto del flotador esté CERRADO:
//   true  -> contacto cerrado = estanque VACÍO (flotador abajo)
//   false -> contacto cerrado = estanque LLENO (flotador arriba)
// Si la bomba hace lo contrario de lo esperado, cambia este valor.
const bool CONTACTO_CERRADO_ES_VACIO = true;

const uint32_t ANTIRREBOTE_MS       = 80;    // Filtro eléctrico del contacto
const uint32_t INTERVALO_REPORTE_MS = 5000;  // Cada cuánto informa por Serial


// ==========================================================================
// 3. VARIABLES INTERNAS   (no se modifican)
// ==========================================================================

enum class Estado {
  APAGADA,             // Estanque lleno, bomba detenida
  ESPERANDO_ARRANQUE,  // El flotador marca vacío, confirmando antes de encender
  ENCENDIDA,           // Bomba llenando el estanque
  ESPERANDO_CORTE,     // El flotador marca lleno, la bomba cumple el retardo de corte
  DESCANSO             // Bomba apagada, protegiendo el motor
};

Estado   estado        = Estado::DESCANSO;
uint32_t tInicioEstado = 0;

// Antirrebote del flotador
bool     lecturaCruda       = false;
bool     vacioEstable       = false;
uint32_t tUltimoCambioCrudo = 0;

uint32_t tUltimoReporte = 0;


// ==========================================================================
// 4. FUNCIONES AUXILIARES
// ==========================================================================

const char* nombreEstado(Estado e) {
  switch (e) {
    case Estado::APAGADA:            return "APAGADA";
    case Estado::ESPERANDO_ARRANQUE: return "ESPERANDO ARRANQUE";
    case Estado::ENCENDIDA:          return "ENCENDIDA";
    case Estado::ESPERANDO_CORTE:    return "ESPERANDO CORTE";
    case Estado::DESCANSO:           return "DESCANSO";
  }
  return "?";
}

// Tiempo transcurrido; funciona aunque millis() se desborde (~49 días).
uint32_t transcurrido(uint32_t desde) {
  return millis() - desde;
}

// Envía la orden al relé (resuelve si es activo en bajo o en alto).
void escribirRele(bool encender) {
  bool nivelPin = RELE_ACTIVO_EN_BAJO ? !encender : encender;
  digitalWrite(PIN_RELE, nivelPin ? HIGH : LOW);
}

// Convierte la lectura del pin en "estanque vacío" sí/no.
bool lecturaVacio() {
  // INPUT_PULLUP: contacto cerrado -> LOW
  bool contactoCerrado = (digitalRead(PIN_FLOTADOR) == LOW);
  return CONTACTO_CERRADO_ES_VACIO ? contactoCerrado : !contactoCerrado;
}

// Lee el flotador con antirrebote. Devuelve true si el estanque está VACÍO.
bool estanqueVacio() {
  bool vacio = lecturaVacio();

  if (vacio != lecturaCruda) {          // La lectura cambió:
    lecturaCruda = vacio;               // se reinicia el cronómetro
    tUltimoCambioCrudo = millis();
  }
  if (transcurrido(tUltimoCambioCrudo) >= ANTIRREBOTE_MS) {
    vacioEstable = lecturaCruda;        // Valor estable: se acepta
  }
  return vacioEstable;
}

void cambiarEstado(Estado nuevo) {
  Serial.printf("[%lu s] %s -> %s\n",
                millis() / 1000, nombreEstado(estado), nombreEstado(nuevo));
  estado = nuevo;
  tInicioEstado = millis();
}

// El relé solo se activa en ENCENDIDA o ESPERANDO_CORTE.
// En cualquier otro estado queda apagado.
void actualizarSalidas() {
  bool bombaOn = (estado == Estado::ENCENDIDA ||
                  estado == Estado::ESPERANDO_CORTE);
  escribirRele(bombaOn);
  digitalWrite(PIN_LED, bombaOn ? HIGH : LOW);
}


// ==========================================================================
// 5. MÁQUINA DE ESTADOS   (toda la decisión depende del flotador)
// ==========================================================================

void actualizarMaquina(bool vacio) {
  uint32_t enEstado = transcurrido(tInicioEstado);

  switch (estado) {

    case Estado::APAGADA:
      // El estanque bajó: empezar a confirmar
      if (vacio) cambiarEstado(Estado::ESPERANDO_ARRANQUE);
      break;

    case Estado::ESPERANDO_ARRANQUE:
      if (!vacio) {
        // Fue solo una oscilación del agua
        cambiarEstado(Estado::APAGADA);
      } else if (enEstado >= RETARDO_ARRANQUE_S * 1000UL) {
        cambiarEstado(Estado::ENCENDIDA);
      }
      break;

    case Estado::ENCENDIDA:
      // El estanque se llenó: preparar el corte
      if (!vacio) cambiarEstado(Estado::ESPERANDO_CORTE);
      break;

    case Estado::ESPERANDO_CORTE:
      if (vacio) {
        // El flotador volvió a bajar: seguir llenando
        cambiarEstado(Estado::ENCENDIDA);
      } else if (enEstado >= RETARDO_CORTE_S * 1000UL) {
        cambiarEstado(Estado::DESCANSO);
      }
      break;

    case Estado::DESCANSO:
      if (enEstado >= DESCANSO_MINIMO_S * 1000UL) {
        cambiarEstado(Estado::APAGADA);
      }
      break;
  }
}


// ==========================================================================
// 6. REPORTE POR MONITOR SERIE (115200 baudios)
// ==========================================================================

void reportarEstado(bool vacio) {
  if (transcurrido(tUltimoReporte) < INTERVALO_REPORTE_MS) return;
  tUltimoReporte = millis();

  Serial.printf("Estanque: %-5s | Bomba: %-18s | En este estado: %lu s\n",
                vacio ? "VACIO" : "LLENO",
                nombreEstado(estado),
                transcurrido(tInicioEstado) / 1000);
}


// ==========================================================================
// 7. SETUP Y LOOP
// ==========================================================================

void setup() {
  // Relé en APAGADO antes de configurarlo como salida: evita que la bomba
  // dé un golpe al energizar el ESP32.
  escribirRele(false);
  pinMode(PIN_RELE, OUTPUT);
  escribirRele(false);

  pinMode(PIN_FLOTADOR, INPUT_PULLUP);
  pinMode(PIN_LED, OUTPUT);

  Serial.begin(115200);
  delay(300);

  // Lectura inicial del flotador
  lecturaCruda = lecturaVacio();
  vacioEstable = lecturaCruda;
  tUltimoCambioCrudo = millis();

  // Se parte en DESCANSO: tras un corte de luz la bomba no arranca de golpe.
  estado = Estado::DESCANSO;
  tInicioEstado = millis();

  Serial.println("\n=========================================");
  Serial.println(" Llenado automatico con flotador P35");
  Serial.println("=========================================");
  Serial.printf(" Estanque          : %s\n", NOMBRE_ESTANQUE);
  Serial.printf(" Capacidad         : %.1f L\n", CAPACIDAD_LITROS);
  Serial.printf(" Retardo arranque  : %lu s\n", RETARDO_ARRANQUE_S);
  Serial.printf(" Retardo corte     : %lu s\n", RETARDO_CORTE_S);
  Serial.printf(" Descanso minimo   : %lu s\n", DESCANSO_MINIMO_S);
  Serial.println("=========================================\n");
}

void loop() {
  bool vacio = estanqueVacio();  // 1) Leer el flotador P35
  actualizarMaquina(vacio);      // 2) Decidir
  actualizarSalidas();           // 3) Mandar la señal al relé
  reportarEstado(vacio);         // 4) Informar

  delay(10);
}








# Código Fuente

## Archivos Principales

### `main.ino`
Archivo principal del proyecto Arduino/ESP32

**Estructura:**
- Setup: Inicialización del sistema
- Loop: Ciclo principal de ejecución

### Otros archivos (si aplica)
- `config.h` - Configuraciones y constantes
- `sensores.h` / `sensores.cpp` - Funciones de sensores
- `actuadores.h` / `actuadores.cpp` - Control de actuadores
- `comunicacion.h` / `comunicacion.cpp` - WiFi/Bluetooth

---

## Estándares de Código

### Comentarios
```cpp
// Comentarios en español
// Explicar QUÉ hace y POR QUÉ (no solo el CÓMO)

/**
 * Función: leerTemperatura
 * Descripción: Lee temperatura del sensor DHT22
 * Retorna: float - Temperatura en grados Celsius
 */
float leerTemperatura() {
  // código
}
```

### Nombres de Variables
- Variables: `camelCase`
- Constantes: `UPPER_SNAKE_CASE`
- Funciones: `camelCase()`

---

## Testing y Debugging

### Modo Debug
Descomentar para activar modo debug:
```cpp
#define DEBUG true
```

### Monitor Serial
- Baudrate: 115200
- Mensajes informativos con prefijos:
  - `[INFO]` - Información general
  - `[ERROR]` - Errores
  - `[DEBUG]` - Debug
