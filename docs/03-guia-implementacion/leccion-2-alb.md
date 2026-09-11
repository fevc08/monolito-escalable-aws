# Lección 2: Balanceo de carga y alta disponibilidad

## Objetivo

Distribuir el tráfico hacia la instancia EC2 mediante un Application Load Balancer, y cerrar el acceso directo a la instancia para que solo reciba tráfico a través del ALB.

## Decisiones aplicadas

- Tipo de Load Balancer: [ADR-002](../../adr/ADR-002-seleccion-load-balancer.md)
- Topología de red y Security Groups: [02-arquitectura-general.md](../02-arquitectura-general.md)

## Prerrequisitos

- Lección 1 completada: instancia EC2 `monolito-app-01` corriendo con nginx.
- Availability Zone de la instancia anotada (de la Lección 1).

## Paso a paso

### 1. Crear el Security Group del ALB

1. Ve a **VPC → Security Groups → Create security group**.
2. Nombre: `alb-sg`.
3. Descripción: `Security Group para el Application Load Balancer - Monolito Escalable`.
4. VPC: la misma VPC por defecto usada en la Lección 1.
5. **Reglas de entrada:**

   | Tipo | Protocolo | Puerto | Origen |
   |---|---|---|---|
   | HTTP | TCP | 80 | `0.0.0.0/0` |

6. Reglas de salida: dejar la regla por defecto.
7. Crear el Security Group.

### 2. Crear el Target Group

1. Ve a **EC2 → Target Groups → Create target group**.
2. Tipo de destino: **Instances**.
3. Nombre: `tg-monolito-app`.
4. Protocolo: **HTTP**, puerto **80**.
5. VPC: la misma VPC por defecto.
6. **Health checks:**
   - Protocolo: HTTP
   - Path: `/`
   - Umbrales: dejar los valores por defecto (healthy threshold: 5, unhealthy threshold: 2, intervalo: 30s), son razonables para este ejercicio.
7. En **Register targets**, selecciona la instancia `monolito-app-01` y agrégala en el puerto 80.
8. Crear el Target Group.

### 3. Crear el Application Load Balancer

1. Ve a **EC2 → Load Balancers → Create load balancer → Application Load Balancer**.
2. Nombre: `alb-monolito-escalable`.
3. Esquema: **Internet-facing**.
4. Tipo de dirección IP: IPv4.
5. **Network mapping:**
   - VPC: la misma VPC por defecto.
   - Selecciona **al menos 2 Availability Zones** y, para cada una, una subred pública (recuerda: aunque tu instancia esté solo en una AZ, el ALB requiere el mapeo a 2 AZs mínimo).
6. **Security groups:** selecciona `alb-sg` (creado en el paso 1). Si aparece un SG "default" preseleccionado, quítalo.
7. **Listeners and routing:**
   - Protocolo: HTTP, Puerto: 80.
   - Forward to: el Target Group `tg-monolito-app` creado en el paso 2.
8. Crear el Load Balancer.
9. Espera a que el estado pase de `Provisioning` a **`Active`** (puede tardar 1–2 minutos).

### 4. Restringir el acceso directo a la instancia EC2

Ahora que el ALB existe, cerramos el acceso directo que dejamos abierto temporalmente en la Lección 1.

1. Ve a **VPC → Security Groups → `ec2-sg`**.
2. Edita las **reglas de entrada**.
3. **Elimina** la regla `HTTP / 80 / 0.0.0.0/0`.
4. **Agrega** una nueva regla:

   | Tipo | Protocolo | Puerto | Origen |
   |---|---|---|---|
   | HTTP | TCP | 80 | `alb-sg` *(selecciona el Security Group, no un rango de IP)* |

5. Guarda los cambios.

> 💡 Al seleccionar `alb-sg` como origen (en vez de un CIDR), estás diciendo "solo acepto tráfico que venga de recursos con ese Security Group asociado", es una regla dinámica: si mañana el ALB cambia de IP, la regla sigue funcionando sin tocarla.

### 5. Verificar el health check del Target Group

1. Ve a **EC2 → Target Groups → `tg-monolito-app` → pestaña Targets**.
2. Espera a que el estado de la instancia pase de `initial` a **`healthy`**.
3. Si queda en `unhealthy`, revisa que la regla del paso 4 esté correctamente guardada (causa más común: el origen no quedó apuntando a `alb-sg`).

### 6. Probar el acceso vía DNS del ALB

1. Ve a **EC2 → Load Balancers → `alb-monolito-escalable`**.
2. Copia el valor de **DNS name** (algo como `alb-monolito-escalable-123456789.us-east-1.elb.amazonaws.com`).
3. Ábrelo en el navegador con `http://` adelante.
4. Deberías ver el mismo "Hello World" de la Lección 1, pero ahora sirviéndose a través del ALB, no de la IP directa de la instancia.
5. **Verifica que la IP directa de la instancia ya NO responde** (intenta acceder directamente a `http://<IP-de-la-instancia>`, debería fallar o quedarse cargando, confirmando que el Security Group está correctamente restringido).

## Evidencia a capturar (`evidence/leccion-2-alb/`)

- [x] Security Group `alb-sg` con su regla de entrada (paso 1).
- [x] Target Group `tg-monolito-app` con configuración de health check (paso 2).
- [x] Configuración de creación del ALB (subredes/AZs seleccionadas, security group) (paso 3).
- [x] ALB en estado `Active` (paso 3.9).
- [x] Regla actualizada de `ec2-sg` mostrando origen = `alb-sg` (paso 4).
- [x] Target Group mostrando la instancia en estado `healthy` (paso 5).
- [x] Navegador mostrando "Hello World" accedido vía DNS del ALB (paso 6).
- [x] Captura mostrando que el acceso directo a la IP de la instancia ya no responde.

## Checklist de la lección

- [x] Security Group `alb-sg` creado
- [x] Target Group creado con health check configurado
- [x] ALB creado y en estado `Active`
- [x] `ec2-sg` restringido a solo aceptar tráfico de `alb-sg`
- [x] Instancia en estado `healthy` dentro del Target Group
- [x] Acceso verificado vía DNS del ALB

## Próximo paso

[Lección 3: Escalabilidad automática](./leccion-3-asg.md): crear el Auto Scaling Group, migrar la instancia existente bajo su gestión, y configurar la política de escalado por CPU.