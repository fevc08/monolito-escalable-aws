# ADR-0004: Monitoreo y Alarmas

## Estado
Aceptado

## Contexto

El Auto Scaling Group (ADR-0003) escala automáticamente al superar 60% de CPU, hasta un máximo de 2 instancias. Sin embargo, el escalado automático puede no ser suficiente ante picos extremos de tráfico (limitado por `max=2`), o puede tardar varios minutos en reaccionar (monitoreo básico de 5 min). Se necesita una señal de **visibilidad operacional** independiente que alerte cuando la infraestructura está bajo presión, más allá de lo que el Auto Scaling puede resolver por sí solo.

## Decisión

Crear una **alarma de Amazon CloudWatch** sobre la métrica `CPUUtilization`, agregada a nivel del Auto Scaling Group (dimensión `AutoScalingGroupName`), con:
- **Umbral:** > 80% de CPU promedio
- **Estadística:** Average
- **Período de evaluación:** 5 minutos, 2 períodos consecutivos (para evitar falsos positivos por picos momentáneos)
- **Estados a validar:** `OK` (uso normal) y `ALARM` (uso sostenido sobre el umbral)

## Alternativas consideradas

| Alternativa | Resultado | Justificación |
|---|---|---|
| **CloudWatch Alarm sobre CPU del ASG (>80%)** | ✅ Elegida | Métrica nativa, sin costo adicional (monitoreo básico), y directamente relacionada con la capacidad de cómputo del monolito |
| Dashboard con múltiples métricas (ALB request count, latencia, healthy hosts) | ❌ Descartada para esta fase | Aporta valor, pero excede el alcance mínimo de la Lección 4; queda documentada como mejora futura |
| Monitoreo de terceros (Datadog, New Relic, etc.) | ❌ Descartada | Fuera del alcance de "recursos habilitados en AWS Academy Learner Lab" definido por la consigna |
| Alarma compuesta (Composite Alarm) combinando CPU + estado del Target Group | ❌ Descartada | Complejidad innecesaria para un monolito de una sola métrica crítica en este ejercicio |

## Alineación con AWS Well-Architected Framework

- **Operational Excellence:** la alarma provee una señal explícita de cuándo la infraestructura está operando cerca de su límite de diseño, sin necesidad de revisar métricas manualmente.
- **Reliability:** actúa como segunda línea de defensa, si el ASG alcanza su `max=2` y el CPU se mantiene sobre 80%, la alarma indica que la capacidad configurada ya no es suficiente para la demanda actual.
- **Cost Optimization:** se usa monitoreo básico (gratuito, granularidad de 5 min) en lugar de monitoreo detallado (de pago), consistente con la decisión de costos tomada en ADR-0003.
- **Security:** no requiere permisos IAM adicionales a los ya cubiertos por `LabRole`.

## Objetivos RTO/RPO

- **Relación con RTO:** esta alarma no ejecuta una acción de recuperación automática - es un mecanismo de **detección**, no de remediación. Complementa el RTO de escalado definido en ADR-0003: si el sistema entra en estado `ALARM` de forma sostenida, es una señal de que el `max=2` actual podría no ser suficiente y debe revisarse manualmente.
- **RPO objetivo:** No aplica - la alarma monitorea utilización de cómputo, no datos persistentes.

## Brecha entre diseño ideal y restricciones del AWS Academy Learner Lab

| Diseño ideal | Restricción del Lab / alcance | Ajuste aplicado |
|---|---|---|
| Notificación automática por SNS (email/Slack) al entrar en estado `ALARM` | La sesión del Lab es temporal y las suscripciones SNS por email requieren confirmación fuera de la ventana de la sesión | La validación del estado `OK`/`ALARM` se realiza directamente en la consola de CloudWatch como evidencia (captura de pantalla), sin acción de notificación automática |
| Dashboard consolidado con métricas de ALB + ASG + EC2 | Fuera del alcance mínimo definido por la Lección 4 | Se documenta como mejora futura; esta fase se limita a la alarma de CPU solicitada |

## Consecuencias

**Positivas:**
- Visibilidad operacional inmediata cuando la infraestructura excede el umbral de diseño, sin costo adicional.
- Complementa (no duplica) la política de escalado: 60% escala, 80% alerta.

**Negativas:**
- Al no configurar una acción de notificación (SNS), la alarma requiere revisión manual de la consola para ser detectada, aceptable para este ejercicio, no recomendable en un entorno productivo real.
- El monitoreo se basa únicamente en CPU; no captura degradación a nivel de aplicación (latencia, errores 5xx), lo cual queda como limitación conocida, consistente con lo señalado en ADR-0003.

## Referencias

- Consigna de Evaluación Módulo 6 (SOFOFA) - Lección 4
- AWS Well-Architected Framework
- Amazon CloudWatch Alarms - Documentación oficial AWS