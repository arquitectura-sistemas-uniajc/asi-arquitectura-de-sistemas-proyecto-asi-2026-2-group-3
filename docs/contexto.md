## 1. ¿Qué duele hoy? (Contexto y Problema)
El dolor principal es que el cálculo de nómina quincenal para 200 empleados es un proceso pesado que debe ser exacto. Un error afecta directamente el salario de una persona real. Además, el corte de nómina genera un reto arquitectónico crítico: procesa a todos los empleados con todos sus registros por lotes y, mientras corre este proceso pesado, los empleados siguen entrando a consultar sus horas. Si no se separa la carga transaccional de la analítica, el sistema puede bloquearse o ralentizarse.

## 2. ¿Quién sufre ese dolor? (Usuarios Afectados)
Principalmente los **empleados** (quienes sufren las consecuencias de errores en sus salarios o la imposibilidad de consultar sus horas y desprendibles durante el corte). También lo sufre el **contador** y el **jefe de personal**, quienes deben garantizar que el proceso de liquidación sea exacto y que los desprendibles se generen y envíen correctamente por correo.

## 3. ¿Qué pasa si el sistema se cae una hora? (Disponibilidad)
El impacto depende totalmente del momento de la caída. Si el sistema se cae una hora durante un día normal, el impacto es moderado: los empleados no podrán registrar su entrada/salida ni consultar sus horas. Sin embargo, si el sistema se cae o el proceso falla a la mitad durante el cierre quincenal (liquidación), el impacto es altísimo: se retrasa el pago del salario de 200 personas reales. Esto obliga a que el sistema sea capaz de retomar la liquidación desde donde falló sin duplicar los pagos.


## Actores del Sistema (Usuarios)
El sistema interactúa con tres usuarios principales para cumplir con la funcionalidad mínima requerida[cite: 40]:
*   **Empleados:** Registran su entrada y salida diariamente, y consultan las horas trabajadas en el periodo histórico[cite: 40]. Ellos generan la carga transaccional constante.
*   **Jefe de personal:** Administra al personal, valida novedades y gestiona registros de asistencia[cite: 40].
*   **Contador:** Es el encargado de ejecutar el proceso pesado (la liquidación de nómina quincenal) y de consultar el histórico de los cálculos.

---

### 1. ¿El corte de nómina bloquea la consulta de los empleados? Demuéstrenlo bajo carga.
**Respuesta:** Arquitectónicamente, **no debe bloquearla**. El reto principal es que, mientras corre el proceso por lotes pesado de 200 empleados, estos siguen consultando sus horas, lo que obliga a separar la carga transaccional de la analítica[cite: 40].

*   **Solución Técnica:** Implementaremos el patrón de **Segregación de Responsabilidades (CQRS)** o utilizaremos **Réplicas de Lectura (Read Replicas)** en la base de datos. Cuando el *Contador* ejecute la liquidación, este proceso pesado de escritura correrá en la base de datos primaria o mediante *workers* asíncronos en segundo plano. Simultáneamente, las consultas de los *Empleados* se dirigirán a una base de datos de solo lectura o a una caché.
*   **Demostración:** Bajo carga, esto se demostrará usando una herramienta de pruebas de estrés (como JMeter). Simularemos al *Contador* lanzando el proceso pesado de liquidación y, al mismo tiempo, enviaremos 200 peticiones concurrentes de *Empleados* consultando sus horas. El tiempo de respuesta de las consultas no debe degradarse, demostrando el aislamiento de recursos.

### 2. Si el proceso falla a la mitad, ¿qué pasa? ¿Se puede volver a ejecutar sin duplicar pagos?
**Respuesta:** En este sistema, el cálculo debe ser exacto porque un error afecta el salario de una persona real[cite: 40]. Si falla, el sistema debe ser capaz de reanudarse de forma segura. **Sí, se debe poder volver a ejecutar sin duplicar pagos**.

*   **Solución Técnica:** Se resuelve aplicando **Idempotencia** y manejo de **Transacciones ACID**. El proceso por lotes se diseñará para guardar el estado de cada empleado procesado en una tabla de control (ej. `estado_liquidacion: PENDIENTE, COMPLETADO, FALLIDO`).
*   **Mecanismo:** Si el proceso falla en el empleado número 100 de los 200, la base de datos hace un *rollback* únicamente de esa transacción incompleta. Cuando el *Contador* vuelva a ejecutar la liquidación, el sistema leerá la tabla de control, omitirá a los 99 que ya están `COMPLETADOS` y retomará desde el 100 que está `PENDIENTE`. Además, se genera una clave de idempotencia única por cada pago quincenal para garantizar que el sistema externo bancario no procese duplicados.

### 3. ¿Cómo auditan quién modificó un registro de asistencia y cuándo?
**Respuesta:** Dado que la liquidación depende de la exactitud del cálculo de horas[cite: 40], cualquier alteración en la asistencia (por ejemplo, correcciones manuales del Jefe de personal) debe ser totalmente trazable e irrefutable.

*   **Solución Técnica:** Implementaremos un patrón de **Audit Trail (Bitácora de Auditoría)**. Ningún registro de asistencia se modifica de manera destructiva (no se hace un `UPDATE` simple sobreescribiendo el dato original).
*   **Mecanismo:** Se creará una tabla inmutable llamada `auditoria_asistencia`. Cada vez que se altere un registro de entrada o salida[cite: 40], se insertará una nueva fila en esta tabla capturando:
    *   **Quién:** El ID del actor que hizo el cambio.
    *   **Cuándo:** Marca de tiempo exacta (Timestamp) provista por el servidor de base de datos.
    *   **Qué:** El valor anterior (antes de la modificación) y el valor nuevo.
    *   **Por qué:** Un campo de justificación obligatoria para la novedad.