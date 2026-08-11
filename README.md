# Taller - Semana 2

| Integrante | Código / ID | GitHub |
|---|---|---|
| Juan Eduardo Vera Acero | 1000091871 | juanvera |
| Mabel Fernanda Bernal Amaya | 1000100629 | MabelBernalAmaya |
| Nicolás David Prieto Ramos | 1000091873 | NicolasPrieto12 |

## Condiciones de carrera, regiones críticas y decisiones arquitectónicas

**Asignatura:** Arquitecturas de Software - ARSW
**Periodo:** 2026-2
**Duración máxima:** 60 minutos
**Modalidad sugerida:** Parejas
**Entrega:** Un único README.md por pareja
**Código:** No se requiere modificar código durante este taller

> **Idea guía:** El objetivo no es "poner un `synchronized`". El objetivo es decidir qué consistencia debe garantizarse, dónde debe garantizarse y qué costo arquitectónico introduce esa decisión.

---

## 1. Propósito

Al finalizar el taller, usted debe estar en capacidad de:

- Identificar estado mutable compartido.
- Explicar una condición de carrera mediante un interleaving concreto.
- Formular la invariante que debe conservar el sistema.
- Delimitar la región crítica mínima.
- Comparar mecanismos de sincronización según atributos de calidad.
- Reconocer cuándo una solución válida dentro de una JVM deja de ser suficiente al escalar la arquitectura.

### Caso: reserva concurrente de cupos

Un servicio administra los cupos disponibles para una actividad universitaria.

```java
public final class SeatInventory {
    private int availableSeats;

    public SeatInventory(int initialSeats) {
        this.availableSeats = initialSeats;
    }

    public boolean reserve(int requestedSeats) {
        if (availableSeats >= requestedSeats) {
            availableSeats = availableSeats - requestedSeats;
            return true;
        }
        return false;
    }

    public int availableSeats() {
        return availableSeats;
    }
}
```

Suponga que inicialmente existe 1 cupo y llegan al mismo tiempo dos solicitudes:

- **Thread A:** `reserve(1)`
- **Thread B:** `reserve(1)`

> **Requisito de negocio:** Un mismo cupo no puede reservarse dos veces y el inventario nunca puede quedar en un estado inconsistente.

---

## Parte 1 - Diagnóstico de la carrera

*Tiempo recomendado: 10 minutos*

### 1.1 Estado compartido

¿Cuál es el estado mutable compartido?

> El estado mutable compartido es la variable `availableSeats` de la clase `SeatInventory`, por lo que es un campo de instancia de tipo `int`, que podría ser leído y modificado simultáneamente por múltiples hilos (threads) sin ningún sistema de sincronización.

### 1.2 Operación compuesta

La expresión de reserva parece una sola operación de negocio, pero realmente contiene varias acciones. Descompóngala en pasos lógicos.

1. _(paso 1)_
2. _(paso 2)_
3. _(paso 3)_

### 1.3 Invariante

Defina la invariante principal que debe mantenerse.

> _(respuesta)_

### 1.4 ¿Existe una condición de carrera?

- [ ] Sí
- [ ] No

**Justificación:**

> _(respuesta)_

---

## Parte 2 - Construya el interleaving

*Tiempo recomendado: 10 minutos*

Complete una secuencia posible que haga que las dos reservas sean aceptadas aunque solo exista un cupo.

| Paso | Thread A | Thread B | availableSeats |
|------|----------|----------|-----------------|
| 1    |          |          | 1               |
| 2    |          |          |                 |
| 3    |          |          |                 |
| 4    |          |          |                 |
| 5    |          |          |                 |
| 6    |          |          |                 |

**Pregunta:** ¿Por qué una prueba que ejecuta el método una sola vez de forma secuencial no detecta este problema?

> _(respuesta)_

---

## Parte 3 - Delimite la región crítica

*Tiempo recomendado: 10 minutos*

Considere estas tres alternativas.

### Alternativa A - Sincronizar todo el método

```java
public synchronized boolean reserve(int requestedSeats) {
    if (availableSeats >= requestedSeats) {
        availableSeats -= requestedSeats;
        return true;
    }
    return false;
}
```

### Alternativa B - Sincronizar solo la escritura

```java
public boolean reserve(int requestedSeats) {
    if (availableSeats >= requestedSeats) {
        synchronized (this) {
            availableSeats -= requestedSeats;
        }
        return true;
    }
    return false;
}
```

### Alternativa C - Proteger validación + modificación con un lock privado

```java
private final Object inventoryLock = new Object();

public boolean reserve(int requestedSeats) {
    synchronized (inventoryLock) {
        if (availableSeats >= requestedSeats) {
            availableSeats -= requestedSeats;
            return true;
        }
        return false;
    }
}
```

### 3.1 Región crítica mínima

¿Qué instrucciones deben permanecer dentro de la misma región crítica para preservar la invariante?

> _(respuesta)_

### 3.2 Evalúe las alternativas

| Alternativa | ¿Preserva la invariante? | ¿Qué protege? | Riesgo o costo |
|-------------|---------------------------|----------------|-----------------|
| A           |                           |                |                 |
| B           |                           |                |                 |
| C           |                           |                |                 |

### 3.3 Selección

¿Cuál elegiría para una única instancia de la aplicación dentro de una JVM y por qué?

**Decisión:**

> _(respuesta)_

**Justificación:**

> _(respuesta)_

---

## Parte 4 - El problema se vuelve arquitectónico

*Tiempo recomendado: 15 minutos*

El sistema crece. Ahora existen tres instancias del servicio detrás de un balanceador:

```
                Request
                   |
             Load Balancer
             /     |      \
         App A   App B   App C
             \     |      /
          Base de datos compartida
```

Cada instancia tiene su propio proceso y su propia memoria.

### 4.1 Pregunta crítica

¿Un `synchronized` en App A evita que App B reserve el mismo último cupo al mismo tiempo?

- [ ] Sí
- [ ] No

**Explique:**

> _(respuesta)_

### 4.2 ¿Dónde debe vivir ahora la garantía de consistencia?

Seleccione una o más alternativas razonables y justifique:

- [ ] Monitor `synchronized` en cada JVM.
- [ ] `AtomicInteger` en cada instancia.
- [ ] Transacción en base de datos.
- [ ] Restricción/constraint de base de datos.
- [ ] Control optimista de concurrencia/versionado.
- [ ] Lock distribuido.
- [ ] Otra: _______________________________

**Decisión:**

> _(respuesta)_

### 4.3 Atributos de calidad

| Atributo                    | Impacto esperado | ¿Cómo lo mediría? |
|------------------------------|-------------------|---------------------|
| Correctitud / consistencia   |                   |                     |
| Rendimiento / latencia       |                   |                     |
| Escalabilidad                |                   |                     |
| Disponibilidad                |                   |                     |
| Mantenibilidad                |                   |                     |

---

## Parte 5 - Mini decisión arquitectónica

*Tiempo recomendado: 10 minutos*

Redacte una decisión de máximo 8 líneas.

### Contexto

> _(respuesta)_

### Decisión

> _(respuesta)_

### Trade-off principal

> _(respuesta)_

### Evidencia que pediría antes de aprobar la decisión

Mencione por lo menos dos métricas o pruebas.

1. _(respuesta)_
2. _(respuesta)_

---

## Criterios de evaluación

| Criterio                                                          | Peso |
|---------------------------------------------------------------------|------|
| Identificación correcta de estado, carrera e invariante             | 25%  |
| Interleaving técnicamente válido                                    | 20%  |
| Delimitación correcta de la región crítica                          | 20%  |
| Razonamiento al pasar de una JVM a varias instancias                | 20%  |
| Relación entre decisión, atributos de calidad y evidencia           | 15%  |
| **TOTAL**                                                            | **100%** |

---

## Entrega

El `README.md` debe incluir:

- [ ] Respuestas completas.
- [ ] Interleaving diligenciado.
- [ ] Tabla de alternativas.
- [ ] Decisión para una JVM.
- [ ] Decisión para múltiples instancias.
- [ ] Atributos de calidad y métricas.
- [ ] Mini decisión arquitectónica final.

> **Criterio de calidad:** No se califica por cantidad de texto. Se califica la precisión del razonamiento.
