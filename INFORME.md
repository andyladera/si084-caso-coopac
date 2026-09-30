# Informe de auditoría · Core financiero de la COOPAC Santa Rosa

**SI-084 · Auditoría de Sistemas** · Examen práctico de Unidad I

| | |
|---|---|
| **Apellidos y nombres** | Calizaya Ladera Andy Michael |
| **Código de estudiante** | 2022074258 |
| **URL del repositorio** | `https://github.com/andyladera/si084-caso-coopac` |
| **Fecha** | 2025-12-31 |

## 1. Resultados de los procedimientos

| Regla | Resultado, con cifras | ¿Cumple? | Archivo de evidencia |
|---|---|---|---|
| R1 | El contenedor `sr_bd` publica el puerto en **0.0.0.0:55432→5432/tcp**, es decir, está expuesto a toda la red (dirección 0.0.0.0) | No | `evidencias/P1_puertos.txt` |
| R2 | La contraseña del administrador (`postgres`) está en texto plano en `docker-compose.yml`, línea 10: `POSTGRES_PASSWORD: coopac2023`. Tiene **10 caracteres** (menor a los 12 requeridos) | No | `evidencias/P2_credenciales.txt` |
| R3 | Además de `postgres`, la cuenta **`app_core`** tiene el atributo **Superuser** | No | `evidencias/P3_roles.txt` |
| R4 | **16 cuentas activas** pertenecen a **10 personas cesadas**. El cese más antiguo es **2015-12-18** (Trabajador 005, usuario u059). Además, hay **22 cuentas activas sin documento** (genéricas/sin responsable), de las cuales **4 tienen perfil ADMIN**: `backup`, `backup_3`, `consulta01_3`, `temporal` | No | `evidencias/P4_cesados.txt` · `evidencias/P4_genericas.txt` |
| R5 | Existen **23 desembolsos** donde la misma persona registró y aprobó la operación superando el umbral de S/ 15 000. El monto total es **S/ 709 370.47** y los realizaron **21 usuarios distintos** | No | `evidencias/P5_segregacion.txt` |
| R6 | `log_connections = off` (debería ser `on`) y `log_statement = none` (debería ser `mod`). Se configura en `docker-compose.yml`, línea 11: `command: postgres -c log_connections=off -c log_statement=none` | No | `evidencias/P6_registro.txt` |
| R7 | El último respaldo exitoso es **core_2025-11-14.sql** (fecha 2025-11-14). Han pasado **47 días** sin respaldo hasta el corte del 31/12/2025. Desde el 2025-11-15 todos los respaldos fallan con `ERROR: No space left on device`. Al restaurar, solo se recuperan las tablas `empleados` y `usuarios`; **falta la tabla `desembolsos`** porque el script `respaldo.sh` la excluye con `--exclude-table=desembolsos` (línea 4, comentario: «el respaldo tarda mucho; se excluye la tabla más pesada») | No | `evidencias/P7_respaldos.txt` · `evidencias/P7_restauracion.txt` |

## 2. Hallazgo 1

| Elemento | Contenido |
|---|---|
| Título | Cuentas activas de empleados cesados y cuentas genéricas sin persona responsable en el core financiero |
| Condición | Se encontraron **16 cuentas activas** que pertenecen a **10 personas cesadas**, la más antigua desde el 18/12/2015 (más de 10 años sin desactivar, usuario `u059` — Trabajador 005). Adicionalmente existen **22 cuentas activas sin documento asociado** (sin persona responsable), de las cuales **4 tienen perfil ADMIN** (`backup`, `backup_3`, `consulta01_3`, `temporal`), lo que les otorga privilegios elevados sin control de quién las usa. Evidencia en `evidencias/P4_cesados.txt` y `evidencias/P4_genericas.txt`. |
| Criterio | **Regla R4** de la Política de Seguridad v2.0: «La cuenta de una persona se desactiva el día de su cese. No existen cuentas activas sin persona responsable». Controles de referencia: **A.5.16 Gestión de identidades** y **A.5.18 Derechos de acceso** (NTP-ISO/IEC 27001:2022). |
| Causa | No existe un procedimiento automatizado que vincule la fecha de cese del empleado (registrada en la tabla `empleados`) con la desactivación de su cuenta en la tabla `usuarios`. El área de Recursos Humanos y el área de Sistemas no tienen un proceso de comunicación formal para dar de baja las cuentas al momento del cese. Las cuentas genéricas fueron creadas para distintos propósitos (soporte, interfaces, proveedores) sin asignar un responsable documentado ni una política de revisión periódica. |
| Efecto | Un ex-empleado o cualquier persona que conozca las credenciales de estas cuentas podría acceder al core financiero y consultar, registrar o aprobar operaciones financieras sin ser identificado. Las 4 cuentas ADMIN genéricas podrían usarse para realizar operaciones privilegiadas de forma anónima. En caso de fraude, no es posible determinar quién ejecutó la acción. El riesgo financiero es alto considerando que el core maneja desembolsos con montos superiores a S/ 100 000. |
| Recomendación | **1)** El Jefe de Sistemas debe desactivar de inmediato las 16 cuentas de personas cesadas y las 22 cuentas genéricas sin responsable (plazo: 5 días hábiles). **2)** Implementar un proceso automático que desactive la cuenta del usuario el mismo día que se registra su cese en el sistema de RRHH (plazo: 30 días). **3)** Asignar un responsable documentado a cada cuenta genérica que se determine necesaria, eliminando las que no lo sean (plazo: 15 días). **4)** Establecer una revisión trimestral de cuentas activas vs. empleados vigentes. |

## 3. Hallazgo 2

| Elemento | Contenido |
|---|---|
| Título | Respaldo del core financiero incompleto y detenido durante 47 días, sin incluir la tabla de desembolsos |
| Condición | El último respaldo exitoso data del **14/11/2025** (`core_2025-11-14.sql`). Desde el **15/11/2025 hasta el 31/12/2025** (47 días consecutivos), todos los respaldos fallaron con el error `No space left on device`, como consta en `respaldo.log`. Además, el script `respaldo.sh` excluye deliberadamente la tabla `desembolsos` (`--exclude-table=desembolsos`), que es la tabla más crítica del core financiero. Al restaurar el último respaldo exitoso en la base `restauracion`, solo se recuperaron las tablas `empleados` y `usuarios`; la tabla `desembolsos` no se restauró. Evidencia en `evidencias/P7_respaldos.txt` y `evidencias/P7_restauracion.txt`. |
| Criterio | **Regla R7** de la Política de Seguridad v2.0: «Respaldo diario completo, que incluye la tabla de desembolsos. Su restauración se prueba cada trimestre». Control de referencia: **A.8.13 Respaldo de la información** (NTP-ISO/IEC 27001:2022). |
| Causa | El Jefe de Sistemas decidió excluir la tabla `desembolsos` del respaldo porque «el respaldo tarda mucho» (comentario en `respaldo.sh`, fechado 01/11/2025), priorizando la velocidad sobre la completitud. El disco del servidor se llenó el 15/11/2025 y nadie atendió las alertas del log durante 47 días, lo que indica que no se monitorean los registros de la tarea programada. Tampoco se realizó la prueba de restauración trimestral que habría detectado la ausencia de la tabla `desembolsos`. |
| Efecto | Ante una falla catastrófica del servidor, la cooperativa perdería **todos los registros de desembolsos** (4 813 operaciones por un monto que supera varios millones de soles), sin posibilidad de recuperación. Los 47 días sin respaldo de ninguna tabla significan que incluso los datos de empleados y usuarios de ese período se perderían. La cooperativa quedaría sin poder sustentar sus operaciones ante la SBS, los socios ni auditorías externas, con riesgo de sanciones regulatorias y pérdida de confianza. |
| Recomendación | **1)** El Jefe de Sistemas debe restablecer el respaldo completo incluyendo la tabla `desembolsos` de inmediato, eliminando el flag `--exclude-table` del script (plazo: 1 día). **2)** Liberar espacio en disco o ampliar el almacenamiento del servidor de respaldos para evitar el error `No space left on device` (plazo: 3 días). **3)** Implementar monitoreo automático que alerte cuando un respaldo falle (plazo: 7 días). **4)** Ejecutar y documentar la prueba de restauración trimestral, verificando que todas las tablas del core (incluida `desembolsos`) se restauren correctamente (plazo: 15 días). **5)** Implementar una política de retención y rotación de respaldos antiguos para prevenir la saturación del disco. |
