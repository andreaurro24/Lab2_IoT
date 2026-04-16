# 🔆 Práctica FreeRTOS — Sensores Touch + ESP32


## 🏗️ Arquitectura hipotética del proyecto final

- Camila Leon
- Andrea Urdaneta
- Carlos Sanchez
- Jorge Diaz


### Descripción del proyecto
**Sistema IoT de iluminación inteligente** que usa un sensor PIR (movimiento) y un sensor de luz (LDR) conectados a un ESP32. Las luces se encienden solo cuando hay poca luz ambiente **y** hay presencia de personas, y se apagan automáticamente cuando no hay nadie, optimizando el consumo de energía.

---

### Diagrama de componentes

```
┌─────────────────────────────────────────────────────────────────┐
│                          ESP32                                   │
│                                                                  │
│  ┌──────────────┐    Queue_PIR     ┌───────────────────────┐    │
│  │  Tarea       │ ──────────────► │   Tarea               │    │
│  │  LeerPIR     │                 │   ControlLuz          │    │
│  │  (T1)        │                 │   (T3)                │    │
│  └──────────────┘                 │                       │    │
│                                   │   [Mutex_Relay]       │    │
│  ┌──────────────┐    Queue_LDR    │   xSemaphoreTake()   │    │
│  │  Tarea       │ ──────────────► │   → Activa/apaga     │    │
│  │  LeerLDR     │                 │     relay/LED         │    │
│  │  (T2)        │                 └───────────────────────┘    │
│  └──────────────┘                                               │
│                                                                  │
│  ┌──────────────┐                 ┌───────────────────────┐    │
│  │  Tarea       │                 │   Tarea               │    │
│  │  EnviarMQTT  │ ◄─────────────  │   MonitorEstado       │    │
│  │  (T5)        │   Queue_Log     │   (T4)                │    │
│  └──────────────┘                 └───────────────────────┘    │
│        │                                                         │
│   [Mutex_Serial]                                                 │
│   Protege acceso                                                 │
│   concurrente al                                                 │
│   puerto serial                                                  │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
   Broker MQTT / Dashboard
```

---

### Tareas identificadas

| Tarea | Descripción | Prioridad |
|-------|-------------|-----------|
| `LeerPIR` | Lee el sensor PIR cada 200ms y pone el valor en `Queue_PIR` | Alta (3) |
| `LeerLDR` | Lee el sensor de luz cada 500ms y pone el valor en `Queue_LDR` | Alta (3) |
| `ControlLuz` | Lee ambas colas y decide si encender/apagar el relay | Media (2) |
| `MonitorEstado` | Registra eventos (encendido/apagado) en `Queue_Log` | Baja (1) |
| `EnviarMQTT` | Lee `Queue_Log` y envía datos al broker MQTT por WiFi | Baja (1) |

### Colas identificadas

| Cola | Productor | Consumidor | Datos |
|------|-----------|------------|-------|
| `Queue_PIR` | `LeerPIR` | `ControlLuz` | `{valor, timestamp}` |
| `Queue_LDR` | `LeerLDR` | `ControlLuz` | `{lux, timestamp}` |
| `Queue_Log` | `MonitorEstado` | `EnviarMQTT` | `{evento, timestamp}` |

### Recursos compartidos con control de concurrencia

| Recurso | Tipo de control | Razón |
|---------|----------------|-------|
| Puerto Serial | Mutex | `LeerPIR` y `LeerLDR` pueden intentar imprimir debug al mismo tiempo |
| Relay / LED | Mutex | `ControlLuz` y `MonitorEstado` no deben cambiar el estado del relay simultáneamente |
| Variable `estadoLuz` | Mutex | Variable global leída/escrita por múltiples tareas |

---

## ❓ Preguntas y respuestas

---

### 1. ¿Cómo se ejecutan tareas de FreeRTOS con la misma función pero distintos parámetros?

Se pasa un puntero a una estructura con los parámetros al crear la tarea con `xTaskCreate`. El cuarto argumento (`pvParameters`) acepta un `void*` que apunta a cualquier tipo de dato.

```cpp
// Definir estructura de parámetros
typedef struct {
  int       pin;
  int       sensorId;
  QueueHandle_t queue;
} ReaderParams_t;

// Crear dos instancias con parámetros diferentes
static ReaderParams_t params1 = { GPIO_NUM_4, 1, queue1 };
static ReaderParams_t params2 = { GPIO_NUM_2, 2, queue2 };

// Ambas tareas ejecutan la MISMA función taskReadSensor
xTaskCreate(taskReadSensor, "Reader1", 2048, &params1, 2, NULL);
xTaskCreate(taskReadSensor, "Reader2", 2048, &params2, 2, NULL);
```

> ⚠️ Los parámetros deben declararse como `static` o en memoria heap para garantizar que existan durante toda la vida de la tarea.

---

### 2. ¿Cuál es el tipo de dato que recibe una tarea? ¿Cómo se convierte?

Toda tarea de FreeRTOS recibe un `void*`. Para usarlo con el tipo correcto, se hace un cast explícito al inicio de la función:

```cpp
void taskReadSensor(void *pvParameters) {
  // Cast de void* al tipo específico
  ReaderParams_t *params = (ReaderParams_t *) pvParameters;

  // Ahora se puede acceder a los campos
  int pin      = params->pin;
  int sensorId = params->sensorId;
  QueueHandle_t q = params->queue;
}
```

El uso de `void*` permite que FreeRTOS sea genérico — cualquier función de tarea tiene la misma firma, independientemente de los datos que necesite.

---

### 3. ¿Qué pasa cuando una cola se llena y una tarea quiere insertar nuevos elementos?

Depende del timeout pasado a `xQueueSend`:

| Timeout | Comportamiento |
|---------|---------------|
| `0` | Retorna `errQUEUE_FULL` inmediatamente sin bloquear |
| `pdMS_TO_TICKS(100)` | Espera hasta 100ms; si no hay espacio, retorna `errQUEUE_FULL` |
| `portMAX_DELAY` | Se bloquea indefinidamente hasta que haya espacio disponible |

```cpp
// Ejemplo con manejo del error
if (xQueueSend(queue, &data, pdMS_TO_TICKS(100)) != pdTRUE) {
  Serial.println("Cola llena — dato descartado");
}
```

En el proyecto de iluminación, si la cola de logs se llena (WiFi caído), los datos más nuevos del sensor se descartan para no bloquear la tarea de lectura.

---

### 4. ¿Es posible que varias tareas lean y escriban a la misma cola?

**Sí.** FreeRTOS garantiza que las operaciones sobre colas son **thread-safe** internamente mediante secciones críticas. Múltiples tareas pueden ser productoras y/o consumidoras de la misma cola sin necesidad de mutex adicional.

```cpp
// Tarea 1 — escribe en la cola compartida
void tarea1(void *p) {
  while(true) {
    xQueueSend(colaCompartida, &dato1, portMAX_DELAY);
  }
}

// Tarea 2 — también escribe en la misma cola
void tarea2(void *p) {
  while(true) {
    xQueueSend(colaCompartida, &dato2, portMAX_DELAY);
  }
}

// Tarea 3 — lee de la misma cola
void tarea3(void *p) {
  MiDato_t dato;
  while(true) {
    xQueueReceive(colaCompartida, &dato, portMAX_DELAY);
  }
}
```

---

### 5. ¿Qué es un deadlock?

Un **deadlock** ocurre cuando dos o más tareas se bloquean mutuamente esperando un recurso que la otra tiene, sin que ninguna pueda avanzar.

#### Ejemplo de deadlock

```cpp
SemaphoreHandle_t mutexA, mutexB;

// Tarea 1: toma A primero, luego B
void tarea1(void *p) {
  while (true) {
    xSemaphoreTake(mutexA, portMAX_DELAY); // ✅ Toma A
    vTaskDelay(pdMS_TO_TICKS(10));
    xSemaphoreTake(mutexB, portMAX_DELAY); // ❌ Espera B — BLOQUEADO
    // Nunca llega aquí
    xSemaphoreGive(mutexB);
    xSemaphoreGive(mutexA);
  }
}

// Tarea 2: toma B primero, luego A
void tarea2(void *p) {
  while (true) {
    xSemaphoreTake(mutexB, portMAX_DELAY); // ✅ Toma B
    vTaskDelay(pdMS_TO_TICKS(10));
    xSemaphoreTake(mutexA, portMAX_DELAY); // ❌ Espera A — BLOQUEADO
    // Nunca llega aquí
    xSemaphoreGive(mutexA);
    xSemaphoreGive(mutexB);
  }
}
```

#### Diagrama del deadlock

```
  Tarea 1                    Tarea 2
     │                          │
     ▼                          ▼
 Toma mutexA ✅           Toma mutexB ✅
     │                          │
     ▼                          ▼
 Espera mutexB ⏳ ◄──────► Espera mutexA ⏳
     │                          │
     ✗ BLOQUEADA               ✗ BLOQUEADA
          ← DEADLOCK →
```

#### ¿Por qué ocurre?
Tarea1 tiene A y espera B. Tarea2 tiene B y espera A. Ninguna puede liberar lo que tiene porque necesita el otro recurso primero.

#### Cómo evitarlo

```cpp
// SOLUCIÓN: siempre tomar los mutex en el mismo orden
void tarea1(void *p) {
  xSemaphoreTake(mutexA, portMAX_DELAY); // Siempre A primero
  xSemaphoreTake(mutexB, portMAX_DELAY); // Luego B
  // ... lógica ...
  xSemaphoreGive(mutexB);
  xSemaphoreGive(mutexA);
}

void tarea2(void *p) {
  xSemaphoreTake(mutexA, portMAX_DELAY); // También A primero
  xSemaphoreTake(mutexB, portMAX_DELAY); // Luego B
  // ... lógica ...
  xSemaphoreGive(mutexB);
  xSemaphoreGive(mutexA);
}
```

Otras estrategias: usar timeout en lugar de `portMAX_DELAY`, o evitar tener más de un mutex a la vez.

---

### 6. Datos inconsistentes sin mutex vs. con mutex

#### Sin mutex — Problema de concurrencia

```cpp
// Variable compartida global
int contadorEventos = 0;

// Tarea 1 — incrementa el contador
void tareaIncrementar(void *p) {
  while (true) {
    // PROBLEMA: read-modify-write NO es atómica
    // Entre la lectura y escritura, otra tarea puede intervenir
    contadorEventos++;       // Puede corromperse
    vTaskDelay(pdMS_TO_TICKS(10));
  }
}

// Tarea 2 — lee e imprime el contador
void tareaImprimir(void *p) {
  while (true) {
    // Puede leer un valor a medio actualizar
    Serial.print("Eventos: ");
    Serial.println(contadorEventos);  // Valor inconsistente
    vTaskDelay(pdMS_TO_TICKS(15));
  }
}

// Posible salida incorrecta:
// Eventos: 5
// Eventos: 5   ← mismo valor dos veces (escritura perdida)
// Eventos: 7   ← saltó un valor
```

#### Con mutex — Solución correcta

```cpp
int contadorEventos = 0;
SemaphoreHandle_t mutexContador;

void tareaIncrementar(void *p) {
  while (true) {
    // Proteger el acceso con mutex
    if (xSemaphoreTake(mutexContador, portMAX_DELAY) == pdTRUE) {
      contadorEventos++;
      xSemaphoreGive(mutexContador);
    }
    vTaskDelay(pdMS_TO_TICKS(10));
  }
}

void tareaImprimir(void *p) {
  while (true) {
    if (xSemaphoreTake(mutexContador, portMAX_DELAY) == pdTRUE) {
      Serial.print("Eventos: ");
      Serial.println(contadorEventos);  // Valor siempre consistente
      xSemaphoreGive(mutexContador);
    }
    vTaskDelay(pdMS_TO_TICKS(15));
  }
}

void setup() {
  Serial.begin(115200);
  mutexContador = xSemaphoreCreateMutex();
  xTaskCreate(tareaIncrementar, "Incrementar", 2048, NULL, 2, NULL);
  xTaskCreate(tareaImprimir,    "Imprimir",    2048, NULL, 1, NULL);
}

void loop() { vTaskDelay(portMAX_DELAY); }

// Salida correcta con mutex:
// Eventos: 1
// Eventos: 2
// Eventos: 3   ← siempre consistente y ordenado
```

---

## 📊 Diagrama del sistema de la práctica

```
┌─────────────────────────────────────────────────────────┐
│                        ESP32                             │
│                                                          │
│  GPIO4 (Touch1)         GPIO2 (Touch2)                  │
│       │                       │                         │
│       ▼                       ▼                         │
│  ┌──────────┐           ┌──────────┐                    │
│  │ Reader1  │           │ Reader2  │  ← misma función A │
│  │ (T1)     │           │ (T2)     │    distintos params │
│  └────┬─────┘           └────┬─────┘                    │
│       │                       │                         │
│       ▼                       ▼                         │
│   Queue1 (cap:10)        Queue2 (cap:10)                │
│       │                       │                         │
│       ▼                       ▼                         │
│  ┌──────────┐           ┌──────────┐                    │
│  │ Writer1  │           │ Writer2  │  ← misma función B │
│  │ (T3)     │           │ (T4)     │    distintos params │
│  └────┬─────┘           └────┬─────┘                    │
│       │                       │                         │
│       └───────────┬───────────┘                         │
│                   ▼                                      │
│            [Mutex Serial]                                │
│            xSemaphoreTake()                             │
│                   │                                      │
│                   ▼                                      │
│             Puerto Serial                               │
│      {"sensor":1,"value":32,"timestamp":1500}           │
│      {"sensor":2,"value":28,"timestamp":1502}           │
└─────────────────────────────────────────────────────────┘
```

---

## 🔌 Conexiones físicas

| Pin ESP32 | Componente | Notas |
|-----------|-----------|-------|
| GPIO 4 (T0) | Sensor Touch 1 | Cable o papel aluminio |
| GPIO 2 (T2) | Sensor Touch 2 | Cable o papel aluminio |
| GND | GND común | — |
| USB | PC | Alimentación + Serial 115200 baud |

> Los sensores touch del ESP32 son capacitivos internos. No se necesitan componentes adicionales — solo tocar el pin con el dedo activa el sensor.

---

*Práctica FreeRTOS — Sistemas Embebidos*
