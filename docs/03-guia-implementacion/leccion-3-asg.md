# Lección 3 — Escalabilidad automática

## Objetivo

Reemplazar la gestión manual de la instancia EC2 por un Auto Scaling Group que cree, registre y escale instancias automáticamente en función de la demanda de CPU.

## Decisiones aplicadas

- Política y parámetros de escalado: [ADR-003](../../adr/ADR-003-estrategia-auto-scaling.md)
- Integración con el ALB: [02-arquitectura-general.md](../02-arquitectura-general.md)

## Decisión práctica: qué hacer con la instancia manual de la Lección 1

Se opta por **terminar la instancia `monolito-app-01`** y dejar que el Auto Scaling Group cree sus propias instancias desde un Launch Template, en vez de adjuntar (`attach`) la instancia existente al ASG. Esto es consistente con el principio de infraestructura inmutable: las instancias no se gestionan ni se "adoptan" manualmente, se reemplazan completas cuando es necesario.

## Prerrequisitos

- Lección 2 completada: ALB activo, Target Group `tg-monolito-app` con la instancia en estado `healthy`.
- Al menos 2 Availability Zones ya identificadas (las mismas usadas al crear el ALB).

## Paso a paso

### 1. Crear el Launch Template

1. Ve a **EC2 → Launch Templates → Create launch template**.
2. Nombre: `lt-monolito-app`.
3. Descripción: `Launch template para el ASG del monolito escalable`.
4. **AMI:** la misma Amazon Linux 2023 usada en la Lección 1.
5. **Instance type:** `t2.micro`.
6. **Key pair:** el mismo key pair por defecto del Lab (`vockey`) — igual que en la Lección 1, no se usará para conectarse.
7. **Network settings:** no fijes una subred específica aquí (el ASG la asignará según las AZs que configures en el paso 3). Sí selecciona el Security Group `ec2-sg`.
8. **Advanced details → IAM instance profile:** el mismo perfil usado en la Lección 1 (`LabInstanceProfile`).
9. **Advanced details → User data:** pega exactamente el mismo script de la Lección 1:

```bash
#!/bin/bash
dnf update -y
dnf install -y nginx
systemctl enable nginx
systemctl start nginx
echo "<h1>Hello World - Monolito Escalable en AWS</h1><p>Instancia: $(hostname -f)</p>" > /usr/share/nginx/html/index.html
```

10. Crear el Launch Template.

### 2. Terminar la instancia manual de la Lección 1

1. Ve a **EC2 → Instances**.
2. Selecciona `monolito-app-01` → **Instance state → Terminate instance**.
3. Confirma. El Target Group la mostrará como `unhealthy`/`draining` momentáneamente, es esperado, el ASG la reemplazará en el siguiente paso.

### 3. Crear el Auto Scaling Group

1. Ve a **EC2 → Auto Scaling Groups → Create Auto Scaling group**.
2. Nombre: `asg-monolito-escalable`.
3. **Launch template:** selecciona `lt-monolito-app` (creado en el paso 1).
4. **Network:** selecciona la misma VPC, y marca **al menos las 2 subredes/AZs** que usaste al crear el ALB.
5. **Load balancing:**
   - Selecciona **"Attach to an existing load balancer"**.
   - Elige el Target Group `tg-monolito-app` (el que ya está conectado al ALB).
6. **Health checks:**
   - Marca **"Turn on Elastic Load Balancing health checks"**, además del health check de EC2 por defecto. Esto es clave: sin esto, el ASG solo detecta instancias caídas a nivel de sistema operativo, no instancias que estén `unhealthy` según el ALB (por ejemplo, si nginx se cae pero el sistema operativo sigue funcionando).
   - Health check grace period: deja el valor por defecto (300 segundos), le da tiempo a la instancia de terminar de ejecutar el User Data antes de evaluarla.
7. **Group size:**
   - Desired capacity: `1`
   - Minimum capacity: `1`
   - Maximum capacity: `2`
8. **Scaling policies:**
   - Selecciona **"Target tracking scaling policy"**.
   - Métrica: **Average CPU Utilization**.
   - Target value: `60`.
   - Deja "Instance warmup" en el valor por defecto.
9. **Notifications:** omite este paso (consistente con [ADR-004](../../adr/ADR-004-monitoreo-y-alarmas.md), sin SNS en esta fase).
10. Revisa y crea el Auto Scaling Group.

### 4. Verificar el estado inicial del ASG

1. Ve a **EC2 → Auto Scaling Groups → `asg-monolito-escalable`**.
2. En la pestaña **Activity**, confirma que aparece un evento de lanzamiento de instancia.
3. En la pestaña **Instance management**, espera a que la nueva instancia pase a **`InService`** y **`Healthy`**.
4. Ve al Target Group `tg-monolito-app` → pestaña **Targets** y confirma que la nueva instancia aparece registrada automáticamente y en estado `healthy`, sin que la hayas registrado tú manualmente.
5. Vuelve a probar el DNS del ALB en el navegador, debería seguir mostrando "Hello World", ahora servido por la instancia creada por el ASG.

### 5. Probar el escalado automático (generar carga de CPU)

1. Ve a **Systems Manager → Session Manager → Start session** y conéctate a la instancia activa del ASG.
2. Instala una herramienta simple de estrés de CPU:

```bash
sudo dnf install -y stress
```

3. Genera carga sostenida de CPU (300 segundos es suficiente para superar el período de evaluación de la política):

```bash
stress --cpu 1 --timeout 300
```

4. Ve a **CloudWatch → Metrics → EC2 → Per-Auto Scaling Group Metrics** y observa cómo sube `CPUUtilization`.
5. Espera unos minutos (la política de Target Tracking reacciona en base al monitoreo básico, cada 5 min) y revisa la pestaña **Activity** del ASG, debería aparecer un evento de **"Launching a new EC2 instance"** al superar el 60% de CPU.
6. Verifica en el Target Group que la nueva instancia también llega a estado `healthy`.
7. Verifica en **Auto Scaling Groups → Instance management** que ahora hay **2 instancias**, idealmente una en cada AZ.

### 6. Verificar el escalado hacia abajo (scale-in)

1. Deja que el comando `stress` termine su timeout de 300 segundos (o detenlo manualmente con `Ctrl+C` / `pkill stress`).
2. Espera unos minutos a que el promedio de CPU baje del 60%.
3. Revisa la pestaña **Activity** del ASG, debería aparecer un evento de **"Terminating EC2 instance"**, volviendo a `desired=1`.

## Evidencia a capturar (`evidence/leccion-3-asg/`)

- [x] Configuración del Launch Template (AMI, tipo de instancia, Security Group, IAM profile, User Data) (paso 1).
- [x] Configuración de creación del ASG: tamaño de grupo, subredes/AZs, Target Group asociado, health checks ELB habilitados, política de Target Tracking al 60% (paso 3).
- [x] Pestaña Activity mostrando el lanzamiento inicial de la instancia (paso 4).
- [x] Target Group mostrando la instancia del ASG registrada automáticamente y `healthy` (paso 4).
- [x] Gráfico de CloudWatch mostrando el aumento de `CPUUtilization` durante la prueba de carga (paso 5).
- [x] Pestaña Activity mostrando el evento de escalado hacia arriba ("Launching…") (paso 5).
- [x] Instance management mostrando 2 instancias activas, en AZs distintas (paso 5).
- [x] Pestaña Activity mostrando el evento de escalado hacia abajo ("Terminating…") (paso 6).

## Checklist de la lección

- [x] Launch Template creado
- [x] Instancia manual de la Lección 1 terminada
- [x] Auto Scaling Group creado (min=1, desired=1, max=2) y conectado al Target Group
- [x] Health checks de ELB habilitados en el ASG
- [x] Política de Target Tracking (CPU 60%) configurada
- [x] Escalado hacia arriba verificado con prueba de carga
- [x] Escalado hacia abajo verificado tras liberar la carga

## Próximo paso

[Lección 4: Monitoreo básico](./leccion-4-cloudwatch.md): crear la alarma de CloudWatch (CPU > 80%) sobre el mismo Auto Scaling Group, y validar sus estados `OK`/`ALARM`.