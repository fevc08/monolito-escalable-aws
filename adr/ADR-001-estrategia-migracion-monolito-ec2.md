# ADR-001: Estrategia de Migración - Monolito en EC2

## Estado
Aceptado

## Contexto

La aplicación de la organización corre actualmente en un único servidor on-premise, lo que genera dos problemas concretos:
- Caídas ante picos de tráfico (no hay capacidad de escalar).
- Altos costos de mantenimiento de infraestructura física propia.

El negocio ha definido explícitamente que se debe **mantener el enfoque monolítico** de la aplicación, no se solicita descomponerla en microservicios en esta fase. Adicionalmente, el proyecto está acotado a los recursos disponibles en **AWS Academy Learner Lab**, lo que impone restricciones de cuenta, permisos y duración de sesión.

## Decisión

Migrar el monolito, sin modificar su arquitectura interna, a instancias **Amazon EC2**, desplegadas detrás de un **Application Load Balancer (ALB)** y gestionadas por un **Auto Scaling Group (ASG)**, con **Amazon CloudWatch** para monitoreo básico.

## Alternativas consideradas

| Alternativa | Resultado | Justificación |
|---|---|---|
| **EC2 + ALB + ASG** (monolito tradicional) | ✅ Elegida | Resuelve escalabilidad y disponibilidad sin tocar el código de la aplicación |
| Contenerización en ECS/Fargate | ❌ Descartada | Requeriría dockerizar la app; fuera de alcance dado el requerimiento de "mantener el enfoque monolito" |
| Arquitectura serverless (Lambda + API Gateway) | ❌ Descartada | Implicaría refactorizar el monolito en funciones independientes; contradice el requerimiento del negocio |

## Alineación con AWS Well-Architected Framework

- **Reliability:** ALB multi-AZ + ASG proveen redundancia y reemplazo automático de instancias no saludables.
- **Performance Efficiency:** el ASG ajusta la capacidad de cómputo a la demanda real, evitando el sobre/sub-aprovisionamiento del servidor on-premise.
- **Cost Optimization:** se paga solo por instancias activas (ASG con `min=1`), a diferencia del costo fijo de mantener hardware propio.
- **Operational Excellence:** CloudWatch entrega visibilidad automatizada, reemplazando el monitoreo manual on-premise.
- **Security:** el control de acceso se apoya en `LabRole` y Security Groups restrictivos (detalle en ADR-004).

## Objetivos RTO/RPO

- **RTO objetivo:** minutos, ante la falla de una instancia, el ASG debe reemplazarla automáticamente y el ALB debe dejar de enrutarle tráfico vía health checks, sin intervención manual.
- **RPO objetivo:** No aplica a nivel de cómputo, el monolito EC2 permanece stateless. El estado persistente se externaliza a DynamoDB; su objetivo de RPO se define en ADR-005.

## Brecha entre diseño ideal y restricciones del AWS Academy Learner Lab

| Diseño ideal | Restricción del Lab | Ajuste aplicado |
|---|---|---|
| Multi-región con DR activo-pasivo | Cuenta de Lab limitada a una región/sesión temporal | Se acota a multi-AZ dentro de una sola región |
| Roles IAM granulares por servicio | `iam:CreateRole` bloqueado | Se usa el rol predefinido `LabRole` |
| Acceso administrativo vía bastion host + SSH | Puerto 22 no disponible / sin permisos para crear bastion custom | Se usa AWS Systems Manager Session Manager |

## Consecuencias

**Positivas:**
- Migración de bajo riesgo, sin reescritura de la aplicación.
- Escalabilidad y disponibilidad resueltas a nivel de infraestructura.

**Negativas:**
- El monolito sigue siendo una única unidad de despliegue: un cambio pequeño requiere reemplazar toda la instancia/AMI.
- No hay aislamiento de fallos entre componentes internos del monolito.
- Esta decisión pospone, no resuelve, la deuda técnica de acoplamiento del monolito.

## Referencias

- Consigna de Evaluación Módulo 6 (SOFOFA)
- AWS Well-Architected Framework
- AWS Academy Learner Lab - restricciones documentadas