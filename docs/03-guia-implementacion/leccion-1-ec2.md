# Lección 1: Despliegue base en EC2

## Objetivo

Desplegar una instancia EC2 con un servidor web (nginx) funcionando, como base sobre la cual se construirá el resto de la arquitectura (ALB, ASG, CloudWatch, DynamoDB).

## Decisiones aplicadas

- Tipo de cómputo y enfoque de migración: [ADR-001](../../adr/ADR-001-estrategia-migracion-monolito-ec2.md)
- Topología de red y Security Groups: [02-arquitectura-general.md](../02-arquitectura-general.md)

## Prerrequisitos

- Sesión activa de **AWS Academy Learner Lab** iniciada.
- Región de trabajo identificada (la que asigna el Lab por defecto).
- Acceso a la consola de EC2 y VPC.

## Paso a paso

### 1. Crear el Security Group temporal para EC2

1. Ve a **VPC → Security Groups → Create security group**.
2. Nombre: `ec2-sg` (recuerda: no puede empezar con `sg-`, es un prefijo reservado por AWS).
3. Descripción: `Security Group temporal para instancia EC2 - Monolito Escalable`.
4. VPC: la VPC por defecto del Lab.
5. **Reglas de entrada (temporales):**

   | Tipo | Protocolo | Puerto | Origen |
   |---|---|---|---|
   | HTTP | TCP | 80 | `0.0.0.0/0` *(temporal, se restringirá en Lección 2)* |

6. Reglas de salida: dejar la regla por defecto (todo el tráfico permitido).
7. Crear el Security Group.

> ⚠️ **Nota:** este origen abierto (`0.0.0.0/0`) es solo para validar el despliegue mientras no existe el ALB. En la Lección 2 vamos a editar esta regla para que el origen sea el Security Group del ALB (`alb-sg`), no el mundo entero.

### 2. Lanzar la instancia EC2

1. Ve a **EC2 → Instances → Launch instance**.
2. **Nombre:** `monolito-app-01`.
3. **AMI:** Amazon Linux 2023 (AMI gratuita, capa Free Tier).
4. **Tipo de instancia:** `t2.micro`.
5. **Par de claves:** selecciona el key pair por defecto disponible en el Lab (normalmente `vockey`). Lo seleccionamos porque el wizard lo exige, pero no lo usaremos para conectarnos — la administración de la instancia se hace vía **AWS Systems Manager Session Manager** (paso 4), sin necesidad de abrir el puerto 22.
6. **Configuración de red:**
   - VPC: la VPC por defecto del Lab.
   - Subred: una subred pública (anota la Availability Zone, la necesitarás para Lección 2/3).
   - Auto-assign public IP: **Enable**.
   - Security Group: selecciona el `ec2-sg` creado en el paso anterior.
7. **Perfil de instancia IAM (Advanced details → IAM instance profile):** selecciona el rol/perfil disponible en el Lab (habitualmente `LabInstanceProfile`, asociado a `LabRole`). Esto es lo que permitirá más adelante que la instancia escriba en DynamoDB sin credenciales estáticas.
8. **User data** (Advanced details → User data), pega el siguiente script:

```bash
#!/bin/bash
dnf update -y
dnf install -y nginx
systemctl enable nginx
systemctl start nginx
echo "<h1>Hello World - Monolito Escalable en AWS</h1><p>Instancia: $(hostname -f)</p>" > /usr/share/nginx/html/index.html
```

9. Revisa el resumen y **Launch instance**.

### 3. Verificar el despliegue

1. Espera a que el **Status check** de la instancia pase a `2/2 checks passed`.
2. Copia la **IP pública** de la instancia desde la consola de EC2.
3. Abre `http://<IP-PUBLICA>` en el navegador.
4. Deberías ver el mensaje "Hello World - Monolito Escalable en AWS" junto al hostname de la instancia.

### 4. Verificar acceso administrativo sin SSH

1. Ve a **Systems Manager → Session Manager → Start session**.
2. Selecciona la instancia `monolito-app-01`.
3. Confirma que puedes abrir una sesión de terminal sin haber configurado el puerto 22 en el Security Group.
4. Ejecuta `systemctl status nginx` para confirmar que el servicio está activo.

## Evidencia a capturar (`evidence/leccion-1-ec2/`)

- [x] Security Group `ec2-sg` con la regla temporal (paso 1).
- [x] Configuración de lanzamiento de la instancia (AMI, tipo, subred, IAM profile).
- [x] User data configurado (paso 2.8).
- [x] Instancia en estado `running` con status checks `2/2`.
- [x] Navegador mostrando "Hello World" desde la IP pública.
- [x] Sesión activa de Session Manager con `systemctl status nginx` mostrando `active (running)`.

## Checklist de la lección

- [x] Security Group `ec2-sg` creado
- [x] Instancia EC2 lanzada con User Data
- [x] Nginx respondiendo en el puerto 80
- [x] Acceso administrativo verificado vía Session Manager (sin SSH)

## Próximo paso

[Lección 2: Balanceo de carga y alta disponibilidad](./leccion-2-alb.md): crear el ALB, asociar esta instancia, y **restringir** `ec2-sg` para que solo acepte tráfico del ALB.