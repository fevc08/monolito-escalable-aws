# ADR-005: Persistencia con DynamoDB

## Estado
Aceptado

## Contexto

La consigna permite, de forma opcional, incorporar persistencia de datos mediante una tabla DynamoDB llamada "Visitas", con clave primaria `id`, para registrar al menos un ítem. A diferencia de los componentes anteriores (EC2, ALB, ASG, CloudWatch), esta es la primera decisión de la arquitectura que introduce **estado persistente** real, por lo que corresponde definir tanto el motor de base de datos como su modo de capacidad.

## Decisión

Implementar una tabla **Amazon DynamoDB** llamada `Visitas`, con:
- **Clave primaria (partition key):** `id` (tipo String)
- **Modo de capacidad:** On-Demand
- **Atributos adicionales sugeridos:** `timestamp` (fecha/hora del registro), `origen` (ej. IP o identificador de la instancia EC2 que generó el registro)

## Alternativas consideradas

| Alternativa | Resultado | Justificación |
|---|---|---|
| **DynamoDB (NoSQL, On-Demand)** | ✅ Elegida | Servicio totalmente administrado, sin necesidad de aprovisionar ni gestionar servidores de base de datos; encaja naturalmente con el patrón de registro simple tipo "clave-valor" que pide la consigna (`id` → visita) |
| Amazon RDS | ❌ Descartada | Requeriría aprovisionar y gestionar una instancia de base de datos relacional (parcheo, backups manuales, subred de BD); sobre-ingeniería para un caso de uso de un solo atributo clave sin relaciones entre tablas |
| Persistencia local en la instancia EC2 (ej. archivo o SQLite) | ❌ Descartada | Contradice el propósito de alta disponibilidad: los datos se perderían al reemplazar la instancia (comportamiento esperado y deseado del ASG), y no serían compartidos entre las réplicas del monolito |

## Modo de capacidad: On-Demand vs. Provisioned

| Alternativa | Resultado | Justificación |
|---|---|---|
| **On-Demand** | ✅ Elegida | El volumen de escritura/lectura de este ejercicio es mínimo e impredecible (un registro de prueba); On-Demand cobra solo por solicitud real, sin riesgo de sobre-aprovisionar capacidad que no se usará |
| Provisioned (con Auto Scaling de capacidad) | ❌ Descartada | Requiere estimar unidades de capacidad (RCU/WCU) para un patrón de tráfico que, en este ejercicio, es prácticamente nulo - complejidad sin beneficio real en este contexto |

## Alineación con AWS Well-Architected Framework

- **Reliability:** DynamoDB replica los datos automáticamente entre múltiples zonas de disponibilidad dentro de la región, sin configuración adicional.
- **Performance Efficiency:** al ser un servicio administrado serverless, escala transparentemente sin intervención manual, consistente con el enfoque de "administrado" ya aplicado en ALB/ASG.
- **Cost Optimization:** el modo On-Demand evita pagar por capacidad no utilizada, alineado con la misma lógica de costos aplicada en ADR-003 (ASG `min=1`).
- **Security:** el acceso desde la instancia EC2 hacia DynamoDB se realiza a través de `LabRole` (no se crean credenciales estáticas ni se embeben access keys en el User Data).

## Objetivos RTO/RPO

- **RTO objetivo:** no aplica un tiempo de recuperación distinto, DynamoDB es un servicio administrado con disponibilidad multi-AZ nativa; no hay una "instancia" que reemplazar.
- **RPO objetivo:** para este ejercicio, se acepta un **RPO no crítico** (los datos son de prueba, no productivos). En un escenario productivo real, se activaría **Point-in-Time Recovery (PITR)** para lograr un RPO cercano a segundos; se documenta como mejora futura, no implementada en esta fase por estar fuera del alcance mínimo de la Lección 5.

## Brecha entre diseño ideal y restricciones del AWS Academy Learner Lab

| Diseño ideal | Restricción del Lab / alcance | Ajuste aplicado |
|---|---|---|
| Point-in-Time Recovery (PITR) activado | No exigido por la consigna; tiempo de sesión del Lab acotado | Se documenta como mejora futura, no se activa en esta fase |
| Cifrado con clave KMS administrada por el cliente (CMK) | `kms:CreateKey` restringido para roles personalizados en el Lab | Se usa el cifrado por defecto de DynamoDB (clave administrada por AWS), suficiente para este ejercicio |
| Acceso vía rol IAM con política de mínimo privilegio específica para la tabla `Visitas` | `iam:CreateRole` y `iam:CreatePolicy` bloqueados en el Lab | Se usa `LabRole`, cuyo alcance de permisos es más amplio de lo ideal - limitación conocida y aceptada, consistente con ADR-001 |

## Consecuencias

**Positivas:**
- Persistencia funcional sin gestionar infraestructura de base de datos.
- Costo prácticamente nulo dado el volumen de uso de este ejercicio (On-Demand).
- Refuerza el mensaje de portafolio: el monolito puede integrarse con servicios administrados sin abandonar su naturaleza monolítica.

**Negativas:**
- `LabRole` otorga permisos más amplios que los que tendría una política de mínimo privilegio diseñada específicamente para la tabla `Visitas`, aceptable en el Lab, no recomendable en producción.
- Sin PITR activado, un borrado accidental de datos no sería recuperable en esta fase, riesgo aceptado dado que los datos son de prueba.

## Referencias

- Consigna de Evaluación Módulo 6 (SOFOFA) - Lección 5 (opcional)
- AWS Well-Architected Framework
- Amazon DynamoDB - Documentación oficial AWS