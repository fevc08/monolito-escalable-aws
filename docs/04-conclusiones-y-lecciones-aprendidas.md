# 04. Conclusiones y Lecciones Aprendidas

## Resumen de lo implementado

Se migró exitosamente una aplicación monolítica desde un modelo conceptual on-premise hacia una arquitectura en AWS que resuelve los dos problemas originales del negocio:

- **Caídas en picos de tráfico** → resuelto mediante Auto Scaling Group con política de Target Tracking (CPU 60%), validado con pruebas de carga reales.
- **Altos costos de mantenimiento** → resuelto mediante servicios administrados (EC2 On-Demand mínimo necesario, ALB, DynamoDB On-Demand), sin infraestructura fija sobreaprovisionada.

Adicionalmente, se incorporó monitoreo básico con CloudWatch y persistencia gestionada con DynamoDB, ambos validados con evidencia funcional real, no solo configuración estática.

## Hallazgos técnicos durante la implementación

A diferencia de un despliegue que "funciona a la primera", este proyecto tuvo dos fallas reales que requirieron diagnóstico, y ambas resultaron más valiosas como aprendizaje que si todo hubiera salido perfecto al primer intento.

### Hallazgo 1: Política de escalado no guardada silenciosamente

**Qué pasó:** al crear el Auto Scaling Group (Lección 3), se generó carga de CPU con `stress` y el ASG nunca escaló a una segunda instancia, pese a que la CPU superó ampliamente el umbral configurado.

**Diagnóstico:** revisando la pestaña "Automatic scaling" del ASG, la sección "Dynamic scaling policies" estaba completamente vacía. El wizard de creación del ASG permite avanzar y completarse **sin error visible** aunque la política de Target Tracking no se haya guardado, es un paso fácil de omitir sin darse cuenta.

**Lección:** la ausencia de un mensaje de error no equivale a una configuración correcta. Verificar el estado final de cada componente crítico (en este caso, confirmar que la política aparece listada) es tan importante como seguir el wizard paso a paso.

### Hallazgo 2: Una alarma sobre el promedio del grupo puede quedar enmascarada por el propio Auto Scaling

**Qué pasó:** tras corregir el Hallazgo 1, se generó carga de CPU para validar la alarma de CloudWatch (umbral > 80%, ADR-004). La política de Target Tracking (60%) sí se disparaba correctamente, pero la alarma de 80% permanecía en `OK` pese a que la métrica visualmente cruzaba el umbral.

**Diagnóstico:** la alarma evalúa el **promedio de CPU de todo el Auto Scaling Group**, no de una instancia individual. Al generar carga en una sola instancia, el ASG reaccionaba (correctamente) lanzando una segunda instancia con CPU baja, lo que "diluía" el promedio del grupo antes de que la alarma pudiera confirmar 2 períodos consecutivos sobre el umbral. Se confirmó suspendiendo temporalmente el proceso "Launch" del ASG, lo que permitió que el promedio se mantuviera alto el tiempo suficiente para disparar la alarma.

**Lección (más allá del ejercicio):** este es un hallazgo arquitectónico real, no solo un problema de configuración. Diseñar una alarma sobre la métrica promedio de un grupo que está **activamente respondiendo** a esa misma métrica puede ocultar condiciones que sí ocurren a nivel de instancia individual. En un entorno productivo, esto se resolvería con una alarma basada en el **máximo** de CPU entre instancias (usando Metric Math), en vez del promedio del grupo, queda documentado aquí como mejora futura, no implementada en esta fase por estar fuera del alcance mínimo de la consigna.

## Decisiones más difíciles del proyecto

- **Proceso de Debugging:** Durante el proceso cuando ocurrieron los fallos, pude ver mejor la realidad entre lo diseñado y lo ejecutado. Además, que se entiende por ejemplo que ambas alarmas sirven para eventos distintos.
- **Fallo de la Alerta en Cloudwatch:** Dude de sí seguir haciendo pruebas para que este fallara y tuve que recurrir a apagar la carga de autoescalado para que pudiera funcionar la alarma.
- **Proactividad:** Esto me permitió ver el comportamiento real del sistema y saber como puedo mejorar el diseño para un cliente real. 

## Mejoras futuras identificadas (fuera del alcance de esta evaluación)

- Alarma de CloudWatch basada en máximo de CPU por instancia (Metric Math), en vez de promedio del grupo, evita el enmascaramiento descrito en el Hallazgo 2.
- Notificaciones automáticas vía SNS ante cambios de estado de alarma (ver brecha en [ADR-004](../adr/ADR-004-monitoreo-y-alarmas.md)).
- HTTPS con certificado ACM y dominio propio (ver brecha en [ADR-002](../adr/ADR-002-seleccion-load-balancer.md)).
- Point-in-Time Recovery en DynamoDB para RPO cercano a segundos (ver [ADR-005](../adr/ADR-005-persistencia-dynamodb.md)).
- Subredes privadas para las instancias EC2 con NAT Gateway, en vez de subredes públicas con Security Group restrictivo.

## Conclusión general

El ejercicio demuestra que es posible resolver problemas reales de disponibilidad y escalabilidad de un monolito **sin reescribir su arquitectura interna**, apoyándose en servicios administrados de AWS. Los dos hallazgos de debugging refuerzan un principio central de la ingeniería de infraestructura: la configuración correcta no se asume, se verifica con evidencia, exactamente el enfoque aplicado en cada lección de este repositorio.

## Referencias

- [ADR-001 a ADR-005](../adr/)
- [02-arquitectura-general.md](./02-arquitectura-general.md)
- [03-guia-implementacion/](./03-guia-implementacion/)