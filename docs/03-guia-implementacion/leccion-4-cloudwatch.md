# Lección 4: Monitoreo básico

## Objetivo

Configurar una alarma de Amazon CloudWatch que detecte cuando el Auto Scaling Group está operando bajo presión sostenida de CPU (por encima del umbral de escalado), como mecanismo de visibilidad operacional independiente del escalado automático.

## Decisiones aplicadas

- Umbral, estadística y período de la alarma: [ADR-004](../../adr/ADR-004-monitoreo-y-alarmas.md)

## Prerrequisitos

- Lección 3 completada: ASG funcionando con política de Target Tracking activa (CPU 60%).

## Paso a paso

### 1. Crear la alarma de CloudWatch

1. Ve a **CloudWatch → Alarms → All alarms → Create alarm**.
2. **Select metric:**
   - Namespace: **EC2**
   - Categoría: **By Auto Scaling Group**
   - Selecciona la métrica `CPUUtilization` para `asg-monolito-escalable`.
3. **Specify metric and conditions:**
   - Estadística: **Average**
   - Período: **5 minutes**
   - Condición: **Greater than** `80`
   - **Additional configuration → Datapoints to alarm:** `2 out of 2` (esto implementa el criterio de "2 períodos consecutivos" del ADR-004, evitando que un pico momentáneo dispare la alarma).
4. **Configure actions:**
   - En "Notification", **no agregues** ningún SNS topic (consistente con la brecha documentada en ADR-004, sin notificación automática en esta fase).
   - Puedes dejar sin acciones configuradas, o remover cualquier acción sugerida por defecto.
5. **Add name and description:**
   - Nombre: `alarm-cpu-alto-monolito`
   - Descripción: `Alarma de CPU > 80% sostenido sobre el ASG del monolito escalable`.
6. Revisa y crea la alarma.

### 2. Verificar el estado inicial

1. Ve a **CloudWatch → Alarms → All alarms**.
2. Confirma que `alarm-cpu-alto-monolito` aparece en estado **`OK`** (o `Insufficient data` si acaba de crearse y aún no hay suficientes puntos de métrica).

### 3. Provocar el estado `ALARM`

1. Conéctate por Session Manager a una instancia del ASG.
2. Genera carga de CPU sostenida, esta vez con un `timeout` más largo para asegurar que se cubran los 2 períodos consecutivos de 5 minutos que exige la alarma:

```bash
stress --cpu 1 --timeout 900
```

> ⚠️ Con este nivel de carga es probable que **también** se dispare el escalado del ASG (política del 60%) además de la alarma (80%), es el comportamiento esperado, ambos mecanismos están observando la misma métrica con umbrales distintos. Aprovecha para confirmar en la pestaña Activity del ASG que el escalado también reacciona.

3. Ve a **CloudWatch → Alarms** y espera a que `alarm-cpu-alto-monolito` pase a estado **`In alarm`**.

### 4. Verificar el retorno a `OK`

1. Detén el `stress` (`Ctrl+C` o espera a que termine su timeout).
2. Espera unos minutos a que el promedio de CPU baje de 80%.
3. Confirma que la alarma vuelve a estado **`OK`**.

## Evidencia a capturar (`evidence/leccion-4-cloudwatch/`)

- [x] Configuración de la alarma: métrica, estadística, período, umbral (80%), datapoints to alarm (2 de 2) (paso 1).
- [x] Alarma en estado `OK` antes de la prueba (paso 2).
- [x] Alarma en estado `In alarm` durante la prueba de carga (paso 3).
- [x] Alarma de vuelta en estado `OK` tras liberar la carga (paso 4).
- [x] Historial de la alarma mostrando los intentos fallidos originales (solo OK, sin In alarm), la del gráfico con los dos picos en punta.
- [x] "Suspended processes" con "Launch" marcado durante la prueba corregida.

## Checklist de la lección

- [x] Alarma `alarm-cpu-alto-monolito` creada con umbral > 80%, 2 períodos consecutivos
- [x] Estado `OK` verificado en condiciones normales
- [x] Estado `In alarm` verificado bajo carga sostenida
- [x] Retorno a estado `OK` verificado tras liberar la carga

## Próximo paso

[Lección 5: Persistencia con DynamoDB](./leccion-5-dynamodb.md): crear la tabla `Visitas` e insertar un ítem de prueba.