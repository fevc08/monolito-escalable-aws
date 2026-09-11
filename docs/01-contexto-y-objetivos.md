# 01. Contexto y Objetivos

## Contexto de negocio

Una empresa de servicios digitales opera su aplicación en un **único servidor on-premise**. Este modelo presenta dos problemas recurrentes:

1. **Caídas ante picos de tráfico:** al no existir capacidad de escalar, la aplicación se degrada o deja de responder cuando la demanda supera la capacidad del servidor.
2. **Altos costos de mantenimiento:** la infraestructura física propia implica costos fijos de hardware, energía y soporte, independientemente del nivel real de uso.

La organización decide migrar la aplicación a AWS, con una condición explícita: **mantener el enfoque monolítico** de la aplicación. No se busca, en esta fase, descomponerla en microservicios ni reescribir su arquitectura interna, el objetivo es resolver los problemas de disponibilidad y escalabilidad a nivel de infraestructura.

## Problema a resolver

> ¿Cómo migrar una aplicación monolítica desde un servidor on-premise hacia AWS, de forma que responda automáticamente a la demanda, esté disponible ante fallos de infraestructura, y sea observable operacionalmente, sin modificar su arquitectura interna?

## Objetivo general

Diseñar e implementar una arquitectura monolítica escalable en AWS Academy Learner Lab que resuelva los problemas de disponibilidad y escalabilidad identificados, utilizando exclusivamente servicios administrados de AWS.

## Objetivos específicos

- Desplegar el monolito en instancias **Amazon EC2**, reemplazando el servidor on-premise.
- Garantizar **alta disponibilidad** mediante un Application Load Balancer.
- Garantizar **escalabilidad automática** mediante un Auto Scaling Group.
- Establecer **monitoreo básico** con Amazon CloudWatch para tener visibilidad operacional.
- Incorporar **persistencia gestionada** con DynamoDB para datos que no deben depender del ciclo de vida de una instancia EC2 específica (ver [ADR-005](../adr/ADR-005-persistencia-dynamodb.md)).
- Documentar la arquitectura resultante mediante un diagrama simple y decisiones de diseño trazables (ADRs).

## Alcance del proyecto

- Migración de infraestructura (cómputo, balanceo, escalado, monitoreo, persistencia).
- Documentación de arquitectura y decisiones de diseño.
- Evidencia de implementación funcional dentro de AWS Academy Learner Lab.

## Fuera de alcance

- Refactorización o descomposición del monolito en microservicios.
- Migración de dominio propio o configuración de HTTPS/certificados (ver brecha documentada en [ADR-002](../adr/ADR-002-seleccion-load-balancer.md)).
- Notificaciones automáticas de alarmas vía SNS (ver [ADR-004](../adr/ADR-004-monitoreo-y-alarmas.md)).
- Estrategia de recuperación ante desastres multi-región.

## Restricciones

- Solo se pueden utilizar recursos disponibles en **AWS Academy Learner Lab**.
- Permisos limitados a los otorgados por el rol predefinido `LabRole` (no es posible crear roles ni políticas IAM personalizadas).
- Sesión de trabajo temporal (el Lab no garantiza persistencia de la cuenta más allá de la ventana de la sesión activa).

## Referencias

- Consigna de Evaluación Módulo 6 (SOFOFA)
- [ADR-001: Estrategia de Migración - Monolito en EC2](../adr/ADR-001-estrategia-migracion-monolito-ec2.md)