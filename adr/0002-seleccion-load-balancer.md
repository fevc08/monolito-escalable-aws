# ADR-0002: Selección de Load Balancer

## Estado
Aceptado

## Contexto

El monolito desplegado en EC2 necesita distribuir el tráfico HTTP entrante entre las instancias gestionadas por el Auto Scaling Group, y dejar de enrutar tráfico automáticamente hacia instancias no saludables. AWS ofrece tres tipos de Elastic Load Balancer, cada uno pensado para un caso de uso distinto.

## Decisión

Utilizar **Application Load Balancer (ALB)** como punto de entrada del tráfico hacia el Auto Scaling Group.

## Alternativas consideradas

| Alternativa | Resultado | Justificación |
|---|---|---|
| **Application Load Balancer (ALB)** | ✅ Elegida | Opera en capa 7 (HTTP/HTTPS), se integra nativamente con Target Groups de Auto Scaling, y soporta health checks a nivel de aplicación (no solo de red) |
| Network Load Balancer (NLB) | ❌ Descartada | Diseñada para tráfico TCP/UDP de altísimo rendimiento y baja latencia (ej. gaming, IoT); el monolito es una app web HTTP simple, no requiere ese nivel de capa 4 |
| Classic Load Balancer (CLB) | ❌ Descartada | Componente legado orientado a arquitecturas EC2-Classic; sin soporte nativo de host/path-based routing ni integración moderna con Target Groups |

## Alineación con AWS Well-Architected Framework

- **Reliability:** health checks HTTP a nivel de aplicación detectan instancias no saludables (no solo caídas de red), y el ALB deja de enrutarles tráfico automáticamente.
- **Performance Efficiency:** al operar en capa 7, el ALB solo hace el trabajo que la aplicación realmente necesita, sin la sobre-ingeniería de un NLB de capa 4.
- **Operational Excellence:** integración nativa con Target Groups del ASG, cuando el ASG escala, el ALB registra/desregistra instancias automáticamente, sin configuración manual adicional.
- **Cost Optimization:** el modelo de precio por LCU (Load Balancer Capacity Unit) del ALB es adecuado para el volumen de tráfico esperado en este ejercicio, sin pagar por capacidades de NLB que no se usan.

## Objetivos RTO/RPO

- **RTO objetivo:** el tiempo de detección de una instancia no saludable depende directamente de la configuración del health check (intervalo de chequeo y umbral de fallos consecutivos). Se configurará con un intervalo corto para minimizar la ventana en que el ALB sigue enrutando tráfico a una instancia caída.
- **RPO objetivo:** No aplica, el ALB no almacena estado ni datos persistentes.

## Brecha entre diseño ideal y restricciones del AWS Academy Learner Lab

| Diseño ideal | Restricción del Lab | Ajuste aplicado |
|---|---|---|
| Certificado ACM + dominio propio vía Route 53, tráfico HTTPS | El Lab no provee un dominio propio ni permite gestionar zonas Route 53 productivas | Se accede mediante el **DNS público autogenerado por el ALB** (`.elb.amazonaws.com`), en HTTP, tal como especifica la Lección 2 de la guía |
| WAF (AWS Web Application Firewall) asociado al ALB | Fuera del alcance definido por la consigna de esta evaluación | No se implementa en esta fase; queda como mejora futura documentada |

## Consecuencias

**Positivas:**
- Integración nativa con el Auto Scaling Group vía Target Groups, sin pasos manuales al escalar.
- Health checks a nivel de aplicación, más precisos que un simple chequeo de puerto TCP.
- Permite en el futuro enrutamiento por path/host sin cambiar de balanceador si el monolito crece.

**Negativas:**
- El ALB opera únicamente en capa 7 (HTTP/HTTPS/gRPC); si en el futuro se necesitara balancear tráfico TCP puro no HTTP, se requeriría un NLB adicional.
- Acceso solo por HTTP en esta fase, al no contar con certificado ACM ni dominio propio en el Lab.

## Referencias

- Consigna de Evaluación Módulo 6 (SOFOFA) - Lección 2
- AWS Well-Architected Framework
- Elastic Load Balancing - Documentación oficial AWS