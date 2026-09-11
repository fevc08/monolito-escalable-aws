# 02. Arquitectura General

## Vista general del flujo

```
                    Internet
                       │
                       ▼
        ┌─────────────────────────────┐
        │  Application Load Balancer  │
        |(subredes públicas, multi-AZ)|
        └──────────────┬──────────────┘
                       │  Target Group (health checks)
                       ▼
        ┌─────────────────────────────┐
        │     Auto Scaling Group      │  
        |  (min=1, desired=1, max=2)  |
        │  ┌────────┐     ┌────────┐  │
        │  │ EC2 #1 │     │ EC2 #2 │  │  (subredes públicas, AZs distintas)
        │  │(monol.)│     │(monol.)│  │
        │  └───┬────┘     └───┬────┘  │
        └──────┼───────────────┼──────┘
               │               │
               ▼               ▼
        ┌─────────────────────────────┐
        │       Amazon DynamoDB       │  (tabla "Visitas")
        └─────────────────────────────┘

        ┌─────────────────────────────┐
        │       Amazon CloudWatch     │
        │  • Métrica CPUUtilization   │──► Target Tracking (ASG) → escala
        │  • Alarma CPU > 80%         │──► Estado OK / ALARM (visibilidad)
        └─────────────────────────────┘
```

## Componentes y responsabilidades

| Componente | Servicio AWS | Responsabilidad | Decisión relacionada |
|---|---|---|---|
| Punto de entrada | Application Load Balancer | Recibe tráfico HTTP, distribuye entre instancias sanas | [ADR-002](../adr/ADR-002-seleccion-load-balancer.md) |
| Cómputo | Amazon EC2 (vía ASG) | Ejecuta el monolito | [ADR-001](../adr/ADR-001-estrategia-migracion-monolito-ec2.md) |
| Escalado | EC2 Auto Scaling | Ajusta el número de instancias según CPU | [ADR-003](../adr/ADR-003-estrategia-auto-scaling.md) |
| Monitoreo | Amazon CloudWatch | Alarma de visibilidad ante CPU sostenida > 80% | [ADR-004](../adr/ADR-004-monitoreo-y-alarmas.md) |
| Persistencia | Amazon DynamoDB | Almacena registros de la tabla "Visitas" | [ADR-005](../adr/ADR-005-persistencia-dynamodb.md) |
| Red | Amazon VPC | Aísla la infraestructura y controla el tráfico entre componentes | Ver sección siguiente |

## Topología de red

- **VPC:** una única VPC dentro de la región del Learner Lab.
- **Subredes:** públicas, distribuidas en al menos **2 Availability Zones**, para que tanto el ALB como el ASG puedan operar en múltiples AZs (requisito de alta disponibilidad).
- **Security Groups:**

| Security Group | Regla de entrada | Origen permitido | Propósito |
|---|---|---|---|
| `alb-sg` | Puerto 80 (HTTP) | `0.0.0.0/0` | El ALB es el único componente expuesto directamente a Internet |
| `ec2-sg` | Puerto de la app (80) | **Solo desde `alb-sg`** | Las instancias EC2 no reciben tráfico directo de Internet, solo del ALB |

> **Nota de diseño:** se usan subredes públicas para EC2 (en vez de privadas + NAT Gateway) para mantener el ejercicio dentro del alcance del Learner Lab, compensando la exposición mediante el Security Group `ec2-sg`, que bloquea todo tráfico que no provenga del `alb-sg`. En un entorno productivo real, la instancia iría en subred privada.

## Flujo de una petición HTTP

1. El cliente resuelve el DNS público del ALB (`*.elb.amazonaws.com`).
2. El ALB recibe la petición en el puerto 80 y la evalúa contra su Target Group.
3. El ALB enruta la petición a una instancia EC2 registrada y **healthy** en el Target Group.
4. La instancia EC2 (ejecutando el monolito) procesa la petición y responde.
5. Si la petición requiere registrar una visita, la aplicación escribe un ítem en la tabla DynamoDB `Visitas` usando las credenciales del `LabRole` asociado a la instancia (sin claves estáticas).

## Flujo de escalado automático

1. CloudWatch recolecta la métrica `CPUUtilization` de las instancias del ASG (monitoreo básico, cada 5 min).
2. La política de Target Tracking compara el promedio contra el objetivo de 60%.
3. Si el promedio supera el 60% de forma sostenida, el ASG lanza una nueva instancia EC2 (hasta `max=2`).
4. La nueva instancia se registra automáticamente en el Target Group del ALB.
5. El ALB comienza a enrutarle tráfico en cuanto pasa su health check.
6. Si la demanda baja, el proceso se revierte: el ASG termina la instancia sobrante (hasta volver a `desired=1`).

## Flujo de monitoreo (alarma)

1. CloudWatch evalúa la misma métrica `CPUUtilization`, esta vez contra el umbral de la alarma (> 80%, 2 períodos consecutivos).
2. Si se supera el umbral de forma sostenida, la alarma pasa de estado `OK` a `ALARM`.
3. Este estado es visible directamente en la consola de CloudWatch (no hay notificación automática — ver brecha documentada en [ADR-004](../adr/ADR-004-monitoreo-y-alarmas.md)).

## Referencias

- [ADR-001 a ADR-005](../adr/)
- [01-contexto-y-objetivos.md](./01-contexto-y-objetivos.md)