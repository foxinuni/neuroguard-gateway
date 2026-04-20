# NeuroGuard — Documentación Técnica Completa

> Sistema IoT para monitoreo continuo y detección de crisis epilépticas tónico-clónicas.
> Este documento explica la arquitectura completa del proyecto, desde el hardware hasta el dashboard web.

---

## Tabla de contenidos

1. [¿Qué es NeuroGuard?](#1-qué-es-neuroguard)
2. [Arquitectura general del sistema](#2-arquitectura-general-del-sistema)
3. [Capa de percepción — Hardware ESP32](#3-capa-de-percepción--hardware-esp32)
4. [Protocolo MQTT — Transmisión de datos](#4-protocolo-mqtt--transmisión-de-datos)
5. [Backend Python — Subscriber y procesamiento](#5-backend-python--subscriber-y-procesamiento)
6. [Algoritmo de detección de crisis](#6-algoritmo-de-detección-de-crisis)
7. [Firebase Firestore — Persistencia y estructura](#7-firebase-firestore--persistencia-y-estructura)
8. [Dashboard Web — React + Firebase](#8-dashboard-web--react--firebase)
9. [Tiempo real — Cómo se proyectan los datos](#9-tiempo-real--cómo-se-proyectan-los-datos)
10. [Componentes del dashboard](#10-componentes-del-dashboard)
11. [Análisis de datos de crisis](#11-análisis-de-datos-de-crisis)
12. [Autenticación y perfiles](#12-autenticación-y-perfiles)
13. [Modo claro/oscuro](#13-modo-clarooscuro)
14. [Infraestructura y despliegue](#14-infraestructura-y-despliegue)

---

## 1. ¿Qué es NeuroGuard?

NeuroGuard es una solución IoT clínica orientada al monitoreo continuo de pacientes con epilepsia. Su objetivo es **detectar en tiempo real eventos compatibles con crisis tónico-clónicas** y visualizarlos en un dashboard médico accesible desde cualquier navegador.

El sistema captura señales fisiológicas (frecuencia cardíaca y saturación de oxígeno) y de movimiento (aceleración y velocidad angular) mediante un wearable basado en ESP32. Estos datos se transmiten por MQTT a un broker en la nube, donde un backend en Python los procesa, aplica un algoritmo de detección y persiste los resultados en Firebase Firestore. Un dashboard web en React lee esos datos en tiempo real y los visualiza con gráficas interactivas.

**Usuarios objetivo:**
- Pacientes con epilepsia diagnosticada
- Familiares y cuidadores
- Neurólogos que requieren histórico de eventos y métricas objetivas

---

## 2. Arquitectura general del sistema

El sistema sigue el modelo clásico de IoT en **tres capas**:

```
┌────────────────────────────────────────────────┐
│  CAPA DE PERCEPCIÓN (Edge)                     │
│  ESP32 + ADXL345 + ITG3205 + MAX30102          │
│  → Lectura de sensores cada 500ms              │
│  → Publicación JSON por MQTT/TLS               │
└──────────────────────┬─────────────────────────┘
                       │ WiFi · MQTT · TLS · Puerto 8883
                       ▼
┌────────────────────────────────────────────────┐
│  BROKER MQTT — HiveMQ Cloud                    │
│  b63334e69deb428284644bb4228f807c.s1.eu.       │
│  hivemq.cloud                                  │
│  → Distribuye mensajes entre publicador        │
│    (ESP32) y suscriptor (backend Python)       │
└──────────────────────┬─────────────────────────┘
                       │ Suscripción MQTT con wildcards
                       ▼
┌────────────────────────────────────────────────┐
│  BACKEND PYTHON — Subscriber + Detector        │
│  paho-mqtt · firebase-admin · Dockerizado      │
│  → Recibe cada lectura del ESP32               │
│  → Aplica el algoritmo de detección            │
│  → Escribe en Firestore (latest + historial    │
│    + eventos de crisis)                        │
└──────────────────────┬─────────────────────────┘
                       │ Firebase Admin SDK · HTTPS
                       ▼
┌────────────────────────────────────────────────┐
│  FIREBASE FIRESTORE — Base de datos en nube    │
│  → Documento `latest/current` actualizado      │
│    con cada lectura (tiempo real)              │
│  → Colección `readings` con historial          │
│  → Colección `events` con crisis detectadas   │
└──────────────────────┬─────────────────────────┘
                       │ onSnapshot() listeners · SDK web
                       ▼
┌────────────────────────────────────────────────┐
│  DASHBOARD WEB — React + Vite + Tailwind CSS   │
│  → Signos vitales en tiempo real               │
│  → Gráficas de monitoreo continuo              │
│  → Historial y análisis de crisis              │
└────────────────────────────────────────────────┘
```

El flujo completo desde que el sensor captura un dato hasta que aparece en la pantalla tiene una **latencia típica de 1-3 segundos**, limitada principalmente por la propagación por MQTT, el procesamiento del backend y la propagación por Firestore.

---

## 3. Capa de percepción — Hardware ESP32

El dispositivo wearable está construido sobre un **ESP32**, microcontrolador con WiFi integrado. Se conectan dos módulos de sensores por el bus I²C:

### 3.1 Sensores utilizados

| Módulo | Sensor | Variable medida | Dirección I²C |
|--------|--------|-----------------|---------------|
| GY-85  | ADXL345 | Aceleración 3 ejes (ax, ay, az) en g | 0x53 |
| GY-85  | ITG3205 | Velocidad angular 3 ejes (gx, gy, gz) en °/s | 0x68 |
| —      | MAX30102 | Fotopletismografía → HR (bpm) y SpO₂ (%) | por defecto |

### 3.2 Variables calculadas en el ESP32

Además de los valores brutos de los ejes, el firmware calcula magnitudes vectoriales:

```
acc_mag  = √(ax² + ay² + az²)    [g]
gyro_mag = √(gx² + gy² + gz²)    [°/s]
```

Estas magnitudes son independientes de la orientación del dispositivo y son las que usa el algoritmo de detección. Ambas señales pasan por un **filtro de media exponencial (EMA)** para atenuar el ruido:

- Acelerómetro: α = 0.85 (suavizado moderado)
- Giroscopio: α = 0.90 (suavizado mayor)

Para la frecuencia cardíaca y SpO₂, el MAX30102 proporciona señales de luz infrarroja (`ir`) y roja (`red`). El firmware aplica la fórmula empírica:

```
R    = (red / ir)
SpO₂ = 110 − 25·R            [%]
HR   = detección de picos en señal IR filtrada   [bpm, rango válido 40–200]
finger = true si ir > umbral mínimo de contacto
```

El campo `finger` es crítico: cuando es `false`, el paciente no tiene el dedo apoyado sobre el sensor y las lecturas de HR y SpO₂ no son válidas. El dashboard lo indica visualmente.

### 3.3 Frecuencia de muestreo y publicación

El ESP32 lee los sensores y publica un paquete JSON completo cada **500 ms** (2 Hz). El tamaño del payload es de aproximadamente 200 bytes, lo que representa un consumo de red muy bajo (~3.2 KB/minuto).

### 3.4 Payload JSON publicado por el ESP32

```json
{
  "device": "esp32_001",
  "imu": {
    "ax": 0.012,
    "ay": -0.005,
    "az": 0.998,
    "acc_mag": 0.999,
    "gx": 0.139,
    "gy": -0.210,
    "gz": 0.000,
    "gyro_mag": 0.249
  },
  "max30102": {
    "ir": 124500,
    "red": 98200,
    "hr": 72.45,
    "spo2": 97.10,
    "finger": true
  }
}
```

---

## 4. Protocolo MQTT — Transmisión de datos

### 4.1 Broker

Se utiliza **HiveMQ Cloud** (plan gratuito) como broker MQTT en la nube. La comunicación usa **TLS sobre el puerto 8883**, lo que garantiza cifrado en tránsito.

```
Host:    b63334e69deb428284644bb4228f807c.s1.eu.hivemq.cloud
Puerto:  8883 (MQTT/TLS)
```

### 4.2 Estructura de tópicos

El esquema de tópicos sigue el patrón:

```
neuroguard/{patient_id}/{device_id}/{tipo}
```

| Tópico | Dirección | Descripción | QoS |
|--------|-----------|-------------|-----|
| `neuroguard/paciente_001/esp32_001/telemetry` | ESP32 → broker | Datos continuos de sensores, cada 500ms | 1 |
| `neuroguard/paciente_001/esp32_001/status` | ESP32 → broker | Estado online/offline del dispositivo (retained) | 1 |
| `neuroguard/paciente_001/esp32_001/event` | ESP32 → broker | Evento de crisis detectado localmente (futuro) | 1 |

**QoS 1** (at-least-once) garantiza que ningún mensaje de telemetría se pierde sin confirmación, a cambio de posibles duplicados (que el backend maneja).

### 4.3 Suscripción con wildcards

El backend usa wildcards MQTT (`+`) para suscribirse a cualquier paciente y dispositivo, lo que permite escalar a múltiples pacientes sin cambiar configuración:

```
neuroguard/+/+/telemetry
neuroguard/+/+/status
neuroguard/+/+/event
```

### 4.4 Reconexión automática

El cliente MQTT del backend (`paho-mqtt`) está configurado con reconexión automática con backoff exponencial (mínimo 2s, máximo 30s). Ante una desconexión del broker, el proceso se reconecta automáticamente sin intervención.

---

## 5. Backend Python — Subscriber y procesamiento

El backend (`app/main.py`) es un proceso Python de larga duración que:

1. Se conecta al broker HiveMQ con TLS
2. Se suscribe a los tres tópicos con wildcard
3. Por cada mensaje recibido, extrae `patient_id`, `device_id` y tipo del tópico
4. Rutealo al handler correspondiente

### 5.1 Handler de telemetría (`handle_telemetry`)

Es el handler más importante. Se ejecuta cada 500ms por dispositivo activo:

```
Mensaje de telemetría recibido
        │
        ▼
Añadir timestamp UTC + patient_id + device_id al payload
        │
        ├──► set_latest_telemetry()   → sobrescribe Firestore latest/current
        │                               (el dashboard react escucha esto)
        │
        ├──► add_telemetry_reading()  → guarda en historial (submuestreo 1/5)
        │                               → una lectura histórica cada ~2.5s
        │
        └──► detector.evaluate()      → ejecuta el algoritmo de detección
                    │
                    └── si detección positiva → add_event() en Firestore
```

### 5.2 Handler de estado (`handle_status`)

Cuando el ESP32 publica en el tópico `/status`, el backend actualiza el campo `status` y `last_seen` del documento del dispositivo en la colección global `devices/`. El dashboard lee este estado para mostrar el indicador online/offline en tiempo real.

### 5.3 Handler de eventos del dispositivo (`handle_event`)

Reservado para detección local futura en el ESP32. Si el firmware del dispositivo detectara una crisis localmente, publicaría en `/event` y el backend la guardaría marcándola con `source: "device"` (en contraste con `source: "backend"` cuando detecta el servidor).

### 5.4 Submuestreo del historial

A 500ms por lectura, guardar cada muestra en Firestore generaría 120 documentos por minuto por dispositivo, lo que saturería la cuota gratuita de Firebase. Por eso se aplica un **submuestreo configurable**:

```python
HISTORY_SUBSAMPLE = 5  # 1 de cada 5 lecturas se persiste en historial
```

Esto resulta en **1 lectura histórica cada 2.5 segundos** (~24 por minuto), reduciendo el uso de Firestore en un 80% sin sacrificar resolución clínica relevante.

---

## 6. Algoritmo de detección de crisis

El componente `CrisisDetector` (`app/crisis_detector.py`) implementa un algoritmo **multimodal con ventana temporal deslizante**. La filosofía es: una crisis tónico-clónica no se detecta por un punto puntual anómalo, sino por **actividad sostenida en el tiempo combinada con respuesta fisiológica**.

### 6.1 Base clínica — ¿Por qué estas señales?

Las crisis tónico-clónicas generalizadas se caracterizan clínicamente por:

| Fase | Descripción clínica | Señal medible |
|------|--------------------|-|
| Tónica (10–20s) | Contracción muscular generalizada, rigidez | acc_mag elevado, gyro_mag elevado |
| Clónica (30–90s) | Espasmos musculares rítmicos, sacudidas | acc_mag y gyro_mag con patrón oscilatorio |
| Postictal | Confusión, relajación | signales motoras bajan |

Adicionalmente, las crisis generalizadas producen respuesta autonómica:
- **Taquicardia ictal**: la FC puede superar 120–140 bpm durante la convulsión
- **Desaturación de oxígeno**: apnea o respiración comprometida pueden bajar el SpO₂ por debajo del 90%

Estas correlaciones son el fundamento de los umbrales elegidos.

### 6.2 Umbrales de detección

```python
UMBRAL_ACC_MAG   = 2.0    # g     — magnitud de aceleración indicativa de convulsión
UMBRAL_GYRO_MAG  = 150.0  # °/s   — actividad angular elevada
UMBRAL_HR_ALTO   = 120    # bpm   — taquicardia ictal
UMBRAL_SPO2_BAJO = 90.0   # %     — desaturación significativa
VENTANA_SEGUNDOS = 10     # s     — duración de la ventana de análisis
COOLDOWN_SEGUNDOS = 60    # s     — tiempo mínimo entre dos alertas
```

Un `acc_mag > 2.0g` representa aproximadamente el doble de la gravedad terrestre, valor que no se alcanza en movimientos cotidianos (marcha, gestos) pero sí en espasmos convulsivos. El umbral de giroscopio `> 150 °/s` equivale a menos de media vuelta por segundo, coherente con las sacudidas clónicas.

### 6.3 Criterio de detección (lógica booleana)

Para generar un evento de posible crisis deben cumplirse **simultáneamente**:

```
crisis_detectada = (
    porcentaje_motor  >= 60%   ← actividad motora sostenida en la ventana
    AND (hr_alto OR spo2_bajo) ← confirmación fisiológica
)
```

**Condición motora:** se evalúa qué porcentaje de las muestras de los últimos 10 segundos supera los umbrales de acc o gyro. El umbral del 60% evita falsos positivos por picos transitorios (golpe, caída).

**Confirmación fisiológica:** requiere que la última lectura muestre taquicardia o desaturación. Este segundo criterio actúa como filtro de especificidad: no toda actividad motora intensa es una crisis (un paciente puede estar haciendo ejercicio).

> Si `finger = false` (sin contacto con el MAX30102), las señales fisiológicas se consideran no disponibles y **no pueden confirmar** la crisis. Esto protege de falsos positivos por sensor desconectado.

### 6.4 Ventana deslizante

La ventana es un buffer circular (`deque`) que mantiene las lecturas de los últimos 10 segundos. Con cada nueva lectura se purgan las muestras más antiguas:

```
tiempo →  [t-10s ... t-5s ... t-3s ... ahora]
buffer:   [  ○  ,  ○  ,  ●  ,  ●  ,  ●  ,  ●  ,  ●  ,  ●  ]
                                ↑
                          60% elevadas → criterio motor cumplido
```

El sistema requiere **mínimo 10 muestras** en el buffer antes de evaluar (equivale a 5 segundos de datos). Esto evita falsos positivos en el inicio del monitoreo.

### 6.5 Cooldown anti-repetición

Una vez generado un evento, se establece un período de cooldown de **60 segundos** durante el cual no se generan nuevos eventos para ese dispositivo. Esto evita que un mismo episodio prolongado genere decenas de alertas redundantes.

### 6.6 Clasificación de severidad

Cada evento se clasifica en tres niveles basándose en un sistema de puntuación:

```python
score = 0
if pct_motor >= 80%: score += 2
elif pct_motor >= 60%: score += 1
if hr > 140: score += 2
elif hr > 120: score += 1
if spo2 < 85%: score += 2
elif spo2 < 90%: score += 1

"high"   si score >= 4
"medium" si score >= 2
"low"    si score < 2
```

| Severidad | Color dashboard | Significado clínico |
|-----------|----------------|---------------------|
| `low`     | Amarillo | Criterios mínimos cumplidos, actividad motora moderada |
| `medium`  | Naranja | Actividad significativa + compromiso fisiológico |
| `high`    | Rojo | Convulsión intensa + taquicardia severa o desaturación importante |

### 6.7 Payload del evento generado

```json
{
  "type": "possible_tonic_clonic",
  "timestamp": "2026-04-18T15:32:10.123Z",
  "start_timestamp": "2026-04-18T15:32:00.123Z",
  "end_timestamp":   "2026-04-18T15:32:10.123Z",
  "duration_seconds": 10,
  "is_nocturnal": false,
  "device_id": "esp32_001",
  "source": "backend",
  "motor": {
    "pct_elevated": 75.0,
    "acc_mag_max":  4.21,
    "acc_mag_mean": 3.08,
    "gyro_mag_max": 280.5,
    "gyro_mag_mean": 195.3
  },
  "physiological": {
    "hr_basal_bpm": 72.0,
    "hr_peak_bpm": 138.5,
    "spo2_min": 87.2,
    "hr_elevated": true,
    "spo2_low": true
  },
  "severity": "high"
}
```

El campo `is_nocturnal` se determina si la hora UTC está entre las 22:00 y las 06:00, dato que permite el análisis de distribución nocturna/diurna en el dashboard.

---

## 7. Firebase Firestore — Persistencia y estructura

Firestore es la base de datos NoSQL en la nube que conecta el backend con el dashboard. Su modelo orientado a documentos y sus **listeners en tiempo real** (`onSnapshot`) son la clave de la experiencia reactiva del dashboard.

### 7.1 Estructura de colecciones

```
patients/
  {patient_id}/                              ← ej: "paciente_001"
    last_event_timestamp: "2026-04-18..."    ← campo del documento raíz
    last_event_id: "abc123"
    devices/
      {device_id}/                           ← ej: "esp32_001"
        latest/
          current                            ← DOCUMENTO ÚNICO
                                               Sobreescrito con CADA lectura
                                               El dashboard escucha aquí
        readings/
          {auto_id}: { ...telemetría }       ← historial submuestreado
                                               ~1 doc cada 2.5 s
    events/
      {auto_id}: { ...crisis }               ← crisis detectadas (máx 50 últimas)

devices/
  {device_id}                                ← estado online/offline global
    status: "online" | "offline"
    last_seen: "2026-04-18T15:32:10Z"
    patient_id: "paciente_001"
```

### 7.2 Patrón de escritura — `latest/current`

El documento `patients/{id}/devices/{id}/latest/current` es el **corazón del tiempo real**. El backend lo sobreescribe completamente con cada lectura nueva (500ms). En Firestore, una operación `set()` sobre un documento existente es atómica y económica.

El dashboard React abre un listener `onSnapshot()` sobre este documento exactamente, que dispara un callback de actualización de estado React cada vez que el documento cambia. Así, la UI refleja los datos del sensor con ~1-3s de latencia total.

---

## 8. Dashboard Web — React + Firebase

El dashboard es una SPA (Single Page Application) construida con:

| Tecnología | Versión | Uso |
|-----------|---------|-----|
| React | 18 | Librería de UI |
| Vite | 5 | Bundler y servidor de desarrollo |
| Tailwind CSS | v4 | Estilos utilitarios |
| Firebase JS SDK | 10 | Comunicación con Firestore y Auth |
| Recharts | 2 | Gráficas de área, línea, barras y pie |
| React Router | 6 | Navegación entre páginas |
| date-fns | 3 | Formateo y cálculo de fechas |

### 8.1 Páginas de la aplicación

```
/              → HomePage      — Presentación pública con features del sistema
/login         → LoginPage     — Autenticación con email/contraseña o Google
/register      → RegisterPage  — Registro de nuevo paciente + ID de dispositivo
/setup-device  → SetupDevicePage — Configuración inicial de dispositivo post-registro
/dashboard     → DashboardPage — Panel principal de monitoreo (requiere auth)
```

### 8.2 Flujo de autenticación

```
Usuario abre app
      │
      ▼
AuthContext verifica estado Firebase Auth
      │
      ├── No autenticado → redirige a /login
      │
      └── Autenticado
              │
              ├── Sin patientId (primer acceso) → redirige a /setup-device
              │
              └── Con patientId → accede a /dashboard
```

El contexto `AuthContext` mantiene el estado del usuario y expone `user`, `patientId` y `loading` a toda la aplicación a través de React Context API.

---

## 9. Tiempo real — Cómo se proyectan los datos

La reactividad del dashboard se basa en cuatro hooks personalizados que envuelven listeners de Firestore:

### 9.1 `useLatestTelemetry(patientId, deviceId)`

```javascript
// Abre un listener onSnapshot sobre patients/{id}/devices/{id}/latest/current
// Cada vez que el backend escribe una nueva lectura (cada 500ms),
// Firestore notifica al cliente web y React re-renderiza los componentes
const ref = doc(db, "patients", patientId, "devices", deviceId, "latest", "current");
onSnapshot(ref, (snap) => {
  if (snap.exists()) setData(snap.data());
});
```

Este hook devuelve la **lectura más reciente** del dispositivo. Es la fuente de verdad para los "vitales en tiempo real".

### 9.2 `useTelemetryHistory(telemetry)`

Recibe como input la lectura más reciente y construye un buffer circular de las **últimas 120 lecturas** (~2 minutos de datos a ~1 lectura/segundo):

```javascript
const MAX_POINTS = 120; // ~2 min

// Cada vez que llega una nueva lectura via useLatestTelemetry...
const point = {
  ts,
  label: new Date(ts).toLocaleTimeString("es-CO"),
  hr:       telemetry.max30102?.hr,
  spo2:     telemetry.max30102?.spo2,
  acc_mag:  telemetry.imu?.acc_mag,
  gyro_mag: telemetry.imu?.gyro_mag,
};
// Se añade al buffer y se descarta el punto más antiguo si supera MAX_POINTS
```

Este buffer es lo que alimenta las **gráficas de área en tiempo real** de RealtimeCharts. No consulta Firestore directamente; acumula los datos que llegan por el listener de `latest/current`.

### 9.3 `useEvents(patientId, maxEvents)`

```javascript
// Escucha la colección events del paciente,
// ordenada por timestamp descendente, limitada a 50 eventos
const q = query(
  collection(db, "patients", patientId, "events"),
  orderBy("timestamp", "desc"),
  limit(50)
);
onSnapshot(q, (snap) => {
  setEvents(snap.docs.map((d) => ({ id: d.id, ...d.data() })));
});
```

Se actualiza automáticamente cuando el backend detecta una nueva crisis y la escribe en Firestore.

### 9.4 `useDeviceStatus(deviceId)`

Escucha el documento `devices/{deviceId}` y devuelve el estado `online`/`offline` del ESP32 en tiempo real. El ESP32 publica un mensaje retained de status al conectarse; el backend lo persiste en Firestore.

### 9.5 `useEventReadings(patientId, deviceId, crisisId)`

Recupera las lecturas históricas **asociadas a una crisis específica** para construir las gráficas de análisis por fase. Filtra la colección `readings` por `crisis_id == crisisId`.

---

## 10. Componentes del dashboard

### 10.1 PatientHeader

Barra superior del dashboard. Muestra:
- **Inicial y nombre** del paciente (de su perfil en Firestore)
- **Tipo de epilepsia** registrado en el perfil
- **Indicador online/offline** del dispositivo con animación de pulso cuando está conectado

El indicador es un círculo verde animado (`pulse-live`) para estado online, gris para offline.

### 10.2 VitalsRealtime

Cuatro tarjetas con los **signos vitales actuales**, actualizadas con cada `useLatestTelemetry`. Cada tarjeta muestra:

| Vital | Sensor | Unidad | Alerta |
|-------|--------|--------|--------|
| Frecuencia Cardíaca | MAX30102 | bpm | HR > 120 o sin contacto |
| SpO₂ | MAX30102 | % | SpO₂ < 90 o sin contacto |
| Acelerómetro | GY-85 ADXL345 | g | acc_mag > 2.0 |
| Giroscopio | GY-85 ITG3205 | °/s | gyro_mag > 150 |

Cuando se supera el umbral de alerta, la tarjeta cambia visualmente a fondo rojo con borde marcado. El texto "Sin contacto" aparece bajo HR y SpO₂ si `finger = false`.

### 10.3 RealtimeCharts — Gráficas de monitoreo continuo

Cuatro gráficas de área (Recharts `AreaChart`) que representan la evolución temporal de los cuatro vitales durante los últimos ~2 minutos:

| Gráfica | Color | Dominio Y |
|---------|-------|-----------|
| Frecuencia Cardíaca | Rojo `#EF4444` | 40–180 bpm |
| SpO₂ | Azul `#3B82F6` | 70–100% |
| Acelerómetro | Ámbar `#F59E0B` | 0–6 g |
| Giroscopio | Violeta `#8B5CF6` | 0–400 °/s |

Cada gráfica tiene un degradado de relleno que va desde `30% opacidad → 2%`, lo que da la apariencia de área con sombra. El eje X muestra la hora formateada (`HH:MM:SS`) con el intervalo optimizado para legibilidad (`interval="preserveStartEnd"`). La animación está desactivada (`isAnimationActive={false}`) para evitar artefactos visuales al actualizar en tiempo real.

Los colores de los ejes, grid y tooltips se adaptan automáticamente al modo claro u oscuro (`useTheme()`).

### 10.4 CrisisSummaryCards — Resumen de crisis

Tres tarjetas de estadísticas rápidas calculadas en el cliente a partir del array de eventos:

- **Última Crisis**: tiempo transcurrido desde el evento más reciente (ej: "hace 3 horas")
- **Última Semana**: número total de crisis en los últimos 7 días + número de prolongadas (> 2 minutos)
- **Último Mes**: número total de crisis en los últimos 30 días + prolongadas

El cálculo lo realiza la función `countCrises(events, days)` en `lib/analytics.js`, que filtra los eventos cuyo timestamp supera el umbral de días y cuenta los que tienen `duration_seconds > 120` como prolongadas.

### 10.5 LastEventDetail — Detalle de la última crisis

Tarjeta detallada con toda la información del evento seleccionado (por defecto el más reciente). Muestra:

- **Badge de severidad** (`low`/`medium`/`high`) coloreado
- **Hace cuánto** ocurrió (fecha relativa humanizada con `date-fns`)
- **Duración formateada** (ej: "1 min 30 seg")
- Tres métricas en tarjetas individuales:
  - **HR pico** durante la crisis (bpm) vs frecuencia basal (72 bpm por defecto)
  - **SpO₂ mínima** durante la crisis (%)
  - **% de actividad motora elevada** en la ventana de detección

Si hay más de un evento, aparece un selector `<select>` que permite navegar entre todas las crisis del historial.

### 10.6 EventPhaseChart — Actividad motora por fase

Gráfica de líneas (`LineChart`) que muestra la evolución temporal de `acc_mag` y `gyro_mag` durante un evento de crisis específico, con regiones de fondo coloreadas por fase clínica:

| Fase | Color de fondo |
|------|---------------|
| `pre` — Pre-ictal | Azul claro |
| `tonic` — Tónica | Rojo claro |
| `clonic` — Clónica | Rojo más intenso |
| `post` — Post-ictal | Verde claro |

Los datos provienen de `useEventReadings`, que recupera las lecturas históricas asociadas a esa crisis (campo `crisis_id`). El eje X es el tiempo en segundos desde el inicio del evento (t₀).

Esta gráfica permite observar cómo la aceleración y el giroscopio evolucionan a través de las distintas fases de la convulsión: pico en la fase tónica, patrón rítmico en la clónica, decaimiento en la postictal.

### 10.7 HrSpo2PhaseChart — HR y SpO₂ por fase

Igual al anterior pero muestra la respuesta fisiológica cardiovascular durante el evento:

- Línea roja: HR en bpm (eje Y izquierdo, 40–180)
- Línea azul: SpO₂ en % (eje Y derecho, 70–100)
- Mismas regiones de fase de fondo

Permite correlacionar visualmente cuándo ocurre la taquicardia ictal y la desaturación con respecto a las fases motoras de la crisis.

---

## 11. Análisis de datos de crisis

Todas las funciones analíticas residen en `src/lib/analytics.js` y operan sobre el array de eventos cargado por `useEvents`. Son cálculos puramente en cliente, sin consultas adicionales a Firestore.

### 11.1 Tendencia semanal — `computeWeeklyCounts(events, weeks)`

Divide el historial en N semanas consecutivas (por defecto 4) y cuenta cuántas crisis ocurrieron en cada semana. También cuenta las **prolongadas** (> 120 segundos).

El resultado alimenta `CrisisTrendBar`, que es un `BarChart` con doble serie de barras (total vs prolongadas) para visualizar si las crisis se están intensificando o reduciendo en el tiempo.

```
Crisis por semana:
  Sem 1 ████████ 8 (2 prolongadas)
  Sem 2 █████ 5
  Sem 3 ██████ 6
  Sem 4 ███ 3   ← tendencia positiva (reducción)
```

### 11.2 Distribución nocturna/diurna — `nightDayDistribution(events)`

Cuenta qué proporción de las crisis ocurrieron de noche (22:00–06:00 UTC) vs de día, usando el campo `is_nocturnal` calculado por el backend en el momento de la detección.

El resultado alimenta `CrisisDistributionPie`, un gráfico de dona (`PieChart`) que muestra los porcentajes con dos colores diferenciados (azul noche, dorado día). Debajo del gráfico se muestran las cifras absolutas y el porcentaje de cada categoría.

La distribución nocturna es clínicamente relevante porque las crisis nocturnas son especialmente peligrosas (el paciente no puede ser asistido y no recuerda el evento) y frecuentes en ciertos tipos de epilepsia como la LGS o la epilepsia frontal nocturna.

### 11.3 Conteo de crisis recientes — `countCrises(events, days)`

Función base usada por `CrisisSummaryCards` para los conteos de última semana y último mes. Filtra por ventana temporal con `date-fns` y aplica la regla de duración para prolongadas.

### 11.4 Formateo temporal — `lib/formatters.js`

| Función | Ejemplo de salida |
|---------|------------------|
| `timeAgo(iso)` | "hace 3 horas", "hace 2 días" |
| `formatDuration(s)` | "1 min 30 seg", "45 seg" |
| `severityStyle(sev)` | Objeto con clases CSS para badge de color |
| `formatBpm(v)` | "138" |
| `formatSpo2(v)` | "87%" |
| `formatG(v)` | "3.1" |

---

## 12. Autenticación y perfiles

La autenticación se gestiona con **Firebase Authentication** (email/contraseña y Google Sign-In).

### 12.1 Registro e inicio de sesión

- `/register`: al registrarse, además de crear la cuenta en Firebase Auth, se crea un documento en Firestore en `patients/{uid}` con `name`, `deviceId`, y otros campos del perfil.
- `/login`: autenticación estándar. También acepta "Continuar con Google" (OAuth 2.0 via Firebase).
- Ambas páginas tienen validación de campos y mensajes de error descriptivos.

### 12.2 Perfil del paciente — `usePatientProfile`

Hook que lee el documento `patients/{uid}` de Firestore y devuelve los campos del perfil (nombre, tipo de epilepsia, ID de dispositivo). Este dato se usa en `PatientHeader` para mostrar el nombre y en `DashboardPage` para saber qué `patientId` y `deviceId` usar en los demás hooks.

### 12.3 Configuración inicial — SetupDevicePage

Tras el primer registro, si el paciente no tiene `patientId` configurado, el sistema redirige automáticamente a `/setup-device`. Esta pantalla solicita el **ID del dispositivo ESP32** (ej: `esp32_001`), que está en la etiqueta física del wearable. Una vez guardado, el paciente queda vinculado a su dispositivo en Firestore.

---

## 13. Modo claro/oscuro

El dashboard implementa un sistema completo de tema claro/oscuro con las siguientes características:

- **Persistencia**: la preferencia se guarda en `localStorage` bajo la clave `ng-theme`
- **Implementación CSS**: Tailwind CSS v4 con `@custom-variant light`, donde el modo oscuro es el **default** y el modo claro se activa añadiendo la clase `.light` al elemento `<html>`
- **Toggle**: botón sol/luna disponible en la barra de navegación de todas las páginas
- **Cobertura**: todos los componentes (incluyendo los estilos inline de las gráficas Recharts) responden al tema

El contexto `ThemeContext` expone `dark` (bool) y `toggleTheme()`. Los componentes de Recharts usan `dark` para cambiar condicionalmente los colores de grid, ejes y tooltips entre paletas oscura y clara.

---

## 14. Infraestructura y despliegue

### 14.1 Backend

El backend Python está completamente **dockerizado**:

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app/ .
COPY firebase-credentials.json .
CMD ["python", "main.py"]
```

El `docker-compose.yml` simplifica el despliegue con configuración de variables de entorno y política de reinicio automático (`restart: always`).

Dependencias principales (`requirements.txt`):
- `paho-mqtt` — cliente MQTT
- `firebase-admin` — SDK de Firebase para Python
- `python-dotenv` — carga de variables de entorno desde `.env`

### 14.2 Dashboard

El dashboard se construye con Vite (`npm run build`) generando archivos estáticos en `dist/`. Puede desplegarse en cualquier hosting estático (Vercel, Netlify, Firebase Hosting).

### 14.3 Variables de entorno críticas

| Variable | Descripción |
|----------|-------------|
| `MQTT_HOST` | Dirección del broker HiveMQ |
| `MQTT_PORT` | Puerto TLS (8883) |
| `MQTT_USER` / `MQTT_PASSWORD` | Credenciales MQTT |
| `FIREBASE_CREDENTIALS_PATH` | Ruta al JSON de credenciales Firebase Admin |
| `HISTORY_SUBSAMPLE` | Factor de submuestreo del historial (default: 5) |

> **Seguridad**: los archivos `firebase-credentials.json` y `.env` están incluidos en `.gitignore` y no deben subirse a repositorios públicos.

---

## Flujo completo — De la convulsión a la pantalla

```
[1] Paciente con convulsión
        │
        ▼
[2] ESP32 detecta acc_mag > 2.0g y gyro_mag > 150°/s
    + MAX30102 registra HR > 120 bpm
        │
        ▼ MQTT/TLS · 500ms
[3] HiveMQ Cloud recibe el mensaje
        │
        ▼ Suscripción wildcard
[4] Backend Python recibe el payload en on_message()
    → Añade timestamp UTC al payload
    → Sobreescribe latest/current en Firestore
    → Acumula en el buffer del CrisisDetector
        │
        ▼ tras 10s de actividad sostenida (≥60% muestras elevadas)
[5] CrisisDetector.evaluate() detecta la crisis
    → Calcula severidad (low/medium/high)
    → Construye payload del evento con métricas motoras y fisiológicas
    → add_event() escribe en patients/{id}/events/
        │
        ▼ Firestore onSnapshot() notifica al cliente web
[6] Dashboard React recibe el nuevo evento
    → useEvents() actualiza el estado React
    → CrisisSummaryCards muestra las cifras actualizadas
    → LastEventDetail muestra el detalle del evento
    → Gráficas de fase muestran la actividad durante la crisis
```

---

*NeuroGuard — Pontificia Universidad Javeriana · Proyecto IoT 2026*
