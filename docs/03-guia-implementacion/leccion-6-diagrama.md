# Lección 6: Representación visual

## Objetivo

Representar visualmente la arquitectura implementada (VPC, ALB, ASG, EC2, CloudWatch, DynamoDB) en un diagrama simple pero técnicamente correcto, usando notación estándar de AWS.

## Decisiones aplicadas

- Todas las decisiones de [ADR-001](../../adr/ADR-001-estrategia-migracion-monolito-ec2.md) a [ADR-005](../../adr/ADR-005-persistencia-dynamodb.md)
- Topología de red descrita en [02-arquitectura-general.md](../02-arquitectura-general.md)

## Herramienta utilizada

**draw.io** (diagrams.net), con la librería de íconos oficiales de AWS integrada.

## Componentes a incluir

- Internet / Cliente
- VPC (contenedor)
- 2 Availability Zones (contenedores dentro de la VPC)
- Application Load Balancer
- Auto Scaling Group (contenedor lógico alrededor de las instancias EC2)
- 2 instancias EC2 (una por AZ)
- Amazon DynamoDB (tabla "Visitas")
- Amazon CloudWatch (métricas + alarma)

## Archivo fuente y exportación

- Fuente editable: `diagrams/src/arquitectura-monolito-escalable.drawio`
- Exportado para el documento final: `diagrams/export/arquitectura-monolito-escalable.png`

## Evidencia a capturar (`evidence/leccion-6-diagrama/`)

- [ ] Captura del diagrama final completo.

## Checklist de la lección

- [ ] Diagrama incluye los 6 componentes mínimos exigidos por la consigna
- [ ] Diagrama refleja correctamente el flujo Internet → ALB → ASG → EC2
- [ ] Diagrama refleja la relación de EC2 con DynamoDB y CloudWatch
- [ ] Archivo `.drawio` guardado en `diagrams/src/`
- [ ] Exportación PNG/SVG guardada en `diagrams/export/`