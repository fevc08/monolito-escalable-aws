# ADR-0003: Estrategia de Auto Scaling

## Estado
Aceptado

## Contexto

El monolito debe responder automáticamente a variaciones de demanda sin intervención manual. La consigna define parámetros concretos: un Auto Scaling Group con `min=1`, `desired=1`, `max=2`, y una política de escalado basada en CPU > 60%. Sobre esa base, corresponde definir el **tipo de política de escalado** más adecuado.

## Decisión

Configurar el Auto Scaling Group con:
- `min=1`, `desired=1`, `max=2`
- Política de **Target Tracking Scaling** basada en la métrica `CPUUtilization`, con un objetivo de **60%** de utilización promedio.

## Alternativas consideradas

| Alternativa | Resultado | Justificación |
|---|---|---|
| **Target Tracking Scaling (CPU 60%)** | ✅ Elegida | AWS ajusta automáticamente el número de instancias para mantener el promedio de CPU cerca del objetivo, sin necesidad de definir manualmente umbrales de alarma por separado |
| Step Scaling | ❌ Descartada | Requiere definir manualmente múltiples escalones de alarmas CloudWatch; complejidad innecesaria para un ASG acotado a `max=2` |
| Scheduled Scaling | ❌ Descartada | Asume patrones de tráfico predecibles por horario; el escenario de negocio describe picos de tráfico variables, no programados |
| Escalado manual | ❌ Descartada | Contradice directamente el objetivo de "Auto Scaling para responder a la demanda" definido en la consigna |

## Alineación con AWS Well-Architected Framework

- **Reliability:** ante un aumento sostenido de carga, el ASG añade una segunda instancia antes de que la primera se sature, evitando degradación del servicio.
- **Performance Efficiency:** Target Tracking ajusta capacidad de forma proporcional a la demanda real medida, sin sobre-dimensionar.
- **Cost Optimization:** `min=1` evita mantener instancias ociosas en horarios de baja demanda; `max=2` pone un techo explícito al costo máximo posible durante el ejercicio.
- **Operational Excellence:** no requiere mantenimiento manual de umbrales de alarma, AWS gestiona la alarma subyacente automáticamente al crear la política.

## Objetivos RTO/RPO

- **RTO objetivo:** el tiempo entre que el CPU promedio supera el umbral y la nueva instancia queda **healthy** y recibiendo tráfico depende de tres factores: el período de agregación de la métrica CPU, el tiempo de arranque de la instancia (User Data), y el `health check grace period` configurado en el ASG. Se documentará el tiempo real observado durante la Lección 3 como evidencia.
- **RPO objetivo:** No aplica, el ASG no gestiona datos persistentes; las instancias son reemplazables sin pérdida de información (arquitectura stateless a nivel de cómputo).

## Brecha entre diseño ideal y restricciones del AWS Academy Learner Lab

| Diseño ideal | Restricción del Lab / alcance | Ajuste aplicado |
|---|---|---|
| Escalado basado en métricas de aplicación (ej. request count por target, latencia) además de CPU | La consigna define explícitamente CPU > 60% como criterio de escalado para este ejercicio | Se implementa únicamente escalado por CPU, dejando métricas de aplicación como mejora futura documentada |
| Monitoreo detallado (granularidad de 1 minuto) para reacción más rápida | Monitoreo detallado tiene costo adicional; el Lab tiene sesión de tiempo acotado | Se usa monitoreo básico (5 minutos) por defecto, aceptando una reacción de escalado más lenta como trade-off consciente |
| `max` más alto para absorber picos extremos | La consigna fija `max=2` explícitamente | Se respeta el límite definido por el ejercicio; en un escenario productivo real este valor se dimensionaría con datos de tráfico histórico |

## Consecuencias

**Positivas:**
- Respuesta automática a la demanda sin intervención manual, cumpliendo el objetivo central del ejercicio.
- Configuración simple de mantener y explicar (una sola métrica, un solo objetivo).

**Negativas:**
- El uso de CPU como única señal de escalado puede no reflejar cuellos de botella reales si la aplicación es I/O-bound en vez de CPU-bound.
- Con monitoreo básico (5 min), puede haber una ventana de varios minutos entre el inicio del pico de tráfico y la incorporación de la nueva instancia.
- `max=2` limita la capacidad de absorber picos que excedan el doble de la capacidad base, aceptable para este ejercicio, no para producción sin revisión.

## Referencias

- Consigna de Evaluación Módulo 6 (SOFOFA) - Lección 3
- AWS Well-Architected Framework
- Amazon EC2 Auto Scaling - Documentación oficial AWS