# Lección 5: Persistencia con DynamoDB

## Objetivo

Crear una tabla DynamoDB para registrar "visitas" a la aplicación, e insertar al menos un ítem, demostrando acceso a datos mediante el rol IAM de la instancia (sin credenciales estáticas).

## Decisiones aplicadas

- Motor de base de datos y modo de capacidad: [ADR-005](../../adr/ADR-005-persistencia-dynamodb.md)

## Prerrequisitos

- Lección 3 completada: ASG activo y funcional.
- Confirmar que el proceso "Launch" del ASG está **reactivado** (no suspendido) tras el debugging de la Lección 4.

## Paso a paso

### 1. Crear la tabla DynamoDB

1. Ve a **DynamoDB → Tables → Create table**.
2. **Table name:** `Visitas`.
3. **Partition key:** `id`, tipo **String**.
4. No agregues sort key (no es necesaria para este caso de uso).
5. **Table settings:** selecciona **Customize settings**.
6. **Read/write capacity settings:** selecciona **On-demand** (consistente con [ADR-005](../../adr/ADR-005-persistencia-dynamodb.md)).
7. Deja el resto de las opciones (encriptación, etc.) en sus valores por defecto.
8. Crea la tabla y espera a que el estado pase a **`Active`**.

### 2. Insertar un ítem manualmente desde la consola

1. Ve a la tabla `Visitas` → pestaña **Explore table items** → **Create item**.
2. Agrega los siguientes atributos:

   | Atributo | Tipo | Valor de ejemplo |
   |---|---|---|
   | `id` | String | `visita-001` |
   | `timestamp` | String | (fecha/hora actual, ej. `2026-09-10T21:00:00Z`) |
   | `origen` | String | `consola-manual` |

3. Guarda el ítem.
4. Confirma que aparece en la vista de "Explore table items".

### 3. (Recomendado) Insertar un ítem vía AWS CLI desde la instancia EC2

Esto demuestra que la instancia puede escribir en DynamoDB usando el rol IAM asociado (`LabRole`), sin ninguna clave de acceso embebida, el mismo mecanismo que usaría la aplicación real.

1. Conéctate a una instancia del ASG vía **Systems Manager → Session Manager**.
2. Verifica que el CLI de AWS está disponible (viene preinstalado en Amazon Linux 2023):

```bash
aws --version
```

3. Confirma que la instancia está usando el rol IAM de la instancia (no credenciales estáticas):

```bash
aws sts get-caller-identity
```

   Deberías ver un ARN que hace referencia a `LabRole` o `LabInstanceProfile`, **no** un usuario IAM con Access Key.

4. Inserta un ítem desde la CLI:

```bash
aws dynamodb put-item \
  --table-name Visitas \
  --item '{
    "id": {"S": "visita-002"},
    "timestamp": {"S": "'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'"},
    "origen": {"S": "ec2-cli-labrole"}
  }' \
  --region us-east-1
```

   > Ajusta `--region` si tu Lab está en una región distinta a `us-east-1`.

5. Verifica que el comando no arrojó error (sin salida = éxito, es el comportamiento normal de `put-item`).

### 4. Verificar ambos ítems en la consola

1. Vuelve a **DynamoDB → Tables → `Visitas` → Explore table items**.
2. Confirma que ahora hay **2 ítems**: `visita-001` (insertado manualmente) y `visita-002` (insertado vía CLI con `LabRole`).

## Evidencia a capturar (`evidence/leccion-5-dynamodb/`)

- [x] Configuración de la tabla al crearla (partition key `id`, modo On-Demand) (paso 1).
- [x] Tabla en estado `Active` (paso 1).
- [x] Ítem `visita-001` insertado manualmente desde la consola (paso 2).
- [x] Salida de `aws sts get-caller-identity` mostrando el rol `LabRole`/`LabInstanceProfile` (paso 3.3) — **importante:** recorta cualquier Account ID visible.
- [x] Comando `put-item` ejecutado sin errores (paso 3.4).
- [x] Vista final de la tabla mostrando ambos ítems (`visita-001` y `visita-002`) (paso 4).

## Checklist de la lección

- [x] Tabla `Visitas` creada con partition key `id`, modo On-Demand
- [x] Ítem insertado manualmente desde la consola
- [x] Acceso vía rol IAM de instancia confirmado (`get-caller-identity`)
- [x] Ítem insertado vía AWS CLI usando `LabRole`
- [x] Ambos ítems visibles en la tabla

## Próximo paso

[Lección 6: Representación visual](./leccion-6-diagrama.md): construir el diagrama de arquitectura en draw.io con todos los componentes implementados.