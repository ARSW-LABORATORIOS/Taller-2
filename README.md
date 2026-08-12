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

1. Ver y leer el valor actual de `availableSeats` y hacer la comparación con `requestedSeats`
2. Puede que hayan cupos disponibles, por lo que calcula el nuevo valor restando `requestedSeats` a `availableSeats`
3. Se escribe el nuevo valor en `availableSeats` y retorna `true`

### 1.3 Invariante

Defina la invariante principal que debe mantenerse.

> Hay que definir la condición que el sistema debe cumplir, con ello el valor de `availableSeats` no puede ser negativo y a su vez no puede ser reservado dos veces, es decir, la suma total de cupos reservados nunca puede superar el número inicial de cupos disponibles.

### 1.4 ¿Existe una condición de carrera?

- [x] Sí
- [ ] No

**Justificación:**

> La operación `reserve` no es atómica, por lo que internamente hace una lectura de `availableSeats`, una comparación y después una escritura, como tres pasos separados. Si dos hilos entran casi al mismo tiempo, ambos pueden leer el mismo valor de `availableSeats` antes de que ninguno haya escrito el resultado. Como los dos ven que hay cupo disponible, ambos pasan la validación y ambos terminan restando y devuelven `true`, haciendo que se reserve el mismo cupo dos veces aunque solo haya uno disponible.

---

## Parte 2 - Construya el interleaving

*Tiempo recomendado: 10 minutos*

Complete una secuencia posible que haga que las dos reservas sean aceptadas aunque solo exista un cupo.

| Paso | Thread A | Thread B | availableSeats |
|------|----------|----------|-----------------|
| 1    | --       | --       | 1               |
| 2    | lee `availableSeats` obtiene 1 | -- | 1 |
| 3    | --       | lee `availableSeats` obtiene 1 | 1 |
| 4    | evalúa `1 >= 1` → true | -- | 1 |
| 5    | escribe `availableSeats = 1 - 1 = 0`, retorna `true` | -- | 0 |
| 6    | --       | evalúa `1 >= 1` → true con el valor que leyó en el paso 3, antes de que el hilo A escribiera, escribe `availableSeats = 1 - 1 = 0` y retorna `true` | 0 |

**Pregunta:** ¿Por qué una prueba que ejecuta el método una sola vez de forma secuencial no detecta este problema?

> La prueba en forma secuencial no genera concurrencia real, cada vez que se llama a `reserve` se ejecutan todos los pasos (lectura, validación y escritura) antes de que inicie el siguiente. El problema radica en que solo aparece cuando dos hilos se intercalan en esos pasos y ambos leen el estado antes de que el otro escriba su resultado. Sin embargo, una ejecución de un solo hilo, así se haga una sola vez, nunca puede reproducir ese entrelazado.

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

> Las instrucciones que deben permanecer dentro de la región crítica son: la lectura de `availableSeats`, la comparación con `requestedSeats` (el `if`) y la escritura del nuevo valor. Ya que no basta con proteger solo la resta porque  si la verificación queda fuera de la región protegida, dos hilos pueden leer el mismo valor de `availableSeats`, pasar la validación y ejecutar la resta, lo que viola la invariante aunque la escritura en sí esté protegida.

### 3.2 Evalúe las alternativas

| Alternativa | ¿Preserva la invariante? | ¿Qué protege? | Riesgo o costo |
|-------------|---------------------------|----------------|-----------------|
| A | Sí | Todo el método, mediante el monitor intrínseco de `this` | Bloquea de más si el método crece con lógica no relacionada con `availableSeats`; el lock es implícito sobre `this`, por lo que código externo podría sincronizar accidentalmente sobre el mismo objeto |
| B | No | Solo la resta (`availableSeats -= requestedSeats`), no la validación previa | Permite doble reserva: dos hilos pueden leer `availableSeats` y pasar el `if` antes de que cualquiera escriba, ya que la comparación queda fuera del bloque sincronizado |
| C | Sí | Un lock privado y dedicado (`inventoryLock`) que envuelve validación + escritura | Mismo costo de exclusión mutua que A, pero con mejor encapsulamiento: nadie externo puede sincronizar por accidente sobre `inventoryLock` |
### 3.3 Selección


¿Cuál elegiría para una única instancia de la aplicación dentro de una JVM y por qué?

**Decisión:**

> Alternativa C.

**Justificación:**

> Elegimos la alternativa C porque, aunque A también protege bien la invariante, C lo hace con un diseño más cuidado. En A el bloqueo se hace sobre `this`, es decir, sobre el objeto completo, y eso significa que cualquier otro código que tenga acceso a ese mismo objeto podría terminar sincronizando sobre él sin que nosotros nos demos cuenta, generando bloqueos que no tienen nada que ver con `reserve()`. En cambio, en C el lock (`inventoryLock`) es privado y existe únicamente para proteger `availableSeats`, así que nadie de afuera puede interferir con él. Nos parece que esto deja mucho más claro qué se está protegiendo y por qué, lo cual ayuda a que el código sea más fácil de mantener a futuro.
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
- [x] No

**Explique:**

> No, porque se tiene  que ver con cómo funciona todo este montaje ahora. El load balancer es básicamente el que recibe las peticiones que llegan y decide a cuál de las tres apps (A, B o C) se la manda, entonces las solicitudes se van repartiendo entre las tres instancias en lugar de ir siempre a la misma. El tema es que cada una de esas instancias (App A, App B, App C) corre en su propia JVM, o sea en su propio proceso de Java con su propia memoria, totalmente separada de las otras. `synchronized` lo que hace es bloquear un objeto para que solo un hilo pueda entrar a la vez, pero eso solo aplica dentro de la misma JVM, entre hilos que están compartiendo esa memoria. App A y App B no comparten memoria entre sí, cada una tiene su propia instancia de `SeatInventory` con su propia copia de `availableSeats` guardada por su lado. Entonces el lock que crea `synchronized` en App A ni siquiera existe para App B, es como si App B no supiera que ese lock existe. Si las dos reciben al mismo tiempo una solicitud para el último cupo, cada una revisa su propia copia local, ve que hay disponibilidad, la reserva y ninguna se entera de lo que está haciendo la otra. El problema entonces ya no es de concurrencia dentro de un solo proceso (como en la Parte 3), sino de consistencia entre procesos distintos que terminan compartiendo un recurso externo, que es la base de datos. Y eso nos daría una sobreescritura y un resultado erróneo sobre el inventario real en ese momento.

### 4.2 ¿Dónde debe vivir ahora la garantía de consistencia?

Seleccione una o más alternativas razonables y justifique:

- [ ] Monitor `synchronized` en cada JVM.
- [ ] `AtomicInteger` en cada instancia.
- [x] Transacción en base de datos.
- [x] Restricción/constraint de base de datos.
- [ ] Control optimista de concurrencia/versionado.
- [ ] Lock distribuido.
- [ ] Otra: _______________________________

**Decisión:**

> Como ya vimos en el punto anterior, `synchronized` y `AtomicInteger` no sirven acá porque los dos viven dentro de la memoria de una sola JVM, y el problema ya no está ahí, sino entre procesos que ni se conocen entre sí. Entonces hay que pensar qué es lo único que App A, App B y App C sí tienen en común, y eso es la base de datos. Por eso la garantía de consistencia se tiene que mover para allá. Usando una transacción que junte la lectura, la validación y la escritura del cupo en una sola operación atómica, básicamente estaríamos haciendo lo mismo que hacía el `synchronized` antes, pero ahora a nivel de base de datos en vez de a nivel de JVM, que es donde realmente se necesita ahora. Y además le sumamos una restricción tipo `CHECK (available_seats >= 0)` directo en la tabla, que funciona como un respaldo extra: aunque algo se nos escape en la lógica de la aplicación o alguien intente escribir directo en la base de datos, esa constraint no deja que el inventario quede en un estado que no tiene sentido, como quedar en negativo.

### 4.3 Atributos de calidad

| Atributo | Impacto esperado | ¿Cómo lo mediría? |
|---|---|---|
| Correctitud / consistencia | Mejora bastante, porque ahora la validación y la escritura del cupo quedan como una sola operación atómica a nivel de base de datos, que es el recurso que sí comparten las tres instancias. Ya no debería poder pasar que dos apps reserven el mismo cupo al mismo tiempo. | Haciendo pruebas de carga con varias instancias mandando solicitudes concurrentes para el mismo cupo y revisando que el inventario final en la base de datos nunca quede en negativo ni por debajo de cero. |
| Rendimiento / latencia | Empeora un poco, porque una transacción en base de datos siempre va a ser más lenta que un `synchronized` en memoria, ya que ahí sí hay que ir hasta la base de datos, esperar el commit, etc. | Midiendo el tiempo de respuesta promedio (latencia) de `reserve()` antes y después del cambio, por ejemplo con algo como un p95 o p99 de tiempos de respuesta. |
| Escalabilidad | Puede verse afectada porque si agregamos más instancias (App D, App E...), todas van a seguir dependiendo de la misma base de datos para cada reserva, entonces la base de datos se puede volver un cuello de botella. | Aumentando poco a poco el número de instancias y de peticiones concurrentes y viendo en qué punto la base de datos empieza a saturarse o a generar más errores/timeouts. |
| Disponibilidad | Baja un poco la independencia entre instancias, porque ahora todas dependen de que la base de datos esté funcionando para poder reservar, antes cada JVM podía seguir funcionando aunque otra tuviera problemas. | Simulando una caída o lentitud de la base de datos y viendo qué tan afectado queda el servicio de reservas en las tres instancias. |
| Mantenibilidad | Mejora porque la lógica de consistencia queda centralizada en un solo lugar (la base de datos, con la transacción y la constraint), en vez de tener que repetir o sincronizar esa lógica en cada instancia por separado. | Revisando qué tan fácil es para el equipo entender y modificar la lógica de reservas sin tener que tocar cada instancia por separado, por ejemplo contando cuántos archivos o componentes hay que cambiar si se necesita ajustar la regla de negocio. |
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
