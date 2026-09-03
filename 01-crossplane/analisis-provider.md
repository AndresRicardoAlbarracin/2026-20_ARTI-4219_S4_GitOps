# =====================================================================
# Análisis del Provider PostgreSQL
# =====================================================================

## Provider: tages/provider-postgresql v0.1.0

### 1. Managed Resources disponibles

El provider expone un CRD por cada tipo de objeto que sabe administrar dentro de un servidor PostgreSQL. Los principales son:

| Managed Resource | Para qué sirve |
|---|---|
| **Database** | Crea y administra una base de datos (`CREATE DATABASE`). |
| **Role** (grupo `postgresql`) | Crea y administra un rol/usuario de PostgreSQL (`CREATE ROLE`), con su contraseña y permisos básicos. |
| **Role** (grupo `grant`) | Asigna la membresía de un rol dentro de otro rol (ej: agregar el usuario `app` al grupo `readonly`). |
| **Schema** | Crea y administra un esquema dentro de una base de datos (`CREATE SCHEMA`). |
| **Extension** | Instala/administra una extensión de PostgreSQL en una base de datos (ej: `pgcrypto`, `uuid-ossp`). |
| **Function** | Crea y administra una función SQL dentro de la base de datos. |
| **Grant** | Otorga privilegios (SELECT, INSERT, etc.) a un rol sobre un esquema o sus objetos. |
| **Privileges** | Define privilegios *por defecto* que se aplicarán a los objetos que se creen en el futuro dentro de un esquema. |
| **Publication** | Crea una publicación para replicación lógica (usado para sincronizar datos entre servidores). |
| **Subscription** | Crea una suscripción que consume una publicación de otro servidor (el otro lado de la replicación lógica). |
| **Slot** | Crea un slot de replicación lógica. |
| **ReplicationSlot** | Crea un slot de replicación física (usado para réplicas en caliente/standby). |
| **Server** | Registra un servidor foráneo (foreign server), usado junto a `postgres_fdw` para consultar otra base de datos externa. |
| **Mapping** | Crea un "user mapping": asocia un usuario local con credenciales para conectarse a ese servidor foráneo. |

En resumen: con este provider se puede administrar prácticamente todo lo que normalmente haría un DBA a mano con SQL (`CREATE DATABASE`, `CREATE ROLE`, `GRANT`, etc.), pero de forma declarativa desde Kubernetes.

### 2. Campos requeridos del recurso Database

Revisando el CRD `databases.postgresql.postgresql.upbound.io` instalado en el clúster:

**Requerido:**
- `spec.forProvider.name` — el nombre de la base de datos a crear. Es el único campo obligatorio (el propio CRD lo valida con una regla: *"spec.forProvider.name is a required parameter"*).

**Opcionales** (si no se indican, PostgreSQL usa sus valores por defecto):
- `owner` — rol dueño de la base de datos (por defecto, el usuario que ejecuta el comando).
- `allowConnections` — si se permite o no conectarse a la base de datos (por defecto `true`).
- `connectionLimit` — máximo de conexiones concurrentes (`-1` = sin límite).
- `encoding` — codificación de caracteres (ej: `UTF8`).
- `isTemplate` — si la base puede ser clonada por cualquier usuario con permiso `CREATEDB`.
- `lcCollate` / `lcCtype` — configuración regional para ordenamiento y clasificación de caracteres.
- `tablespaceName` — tablespace donde se almacenarán los objetos por defecto.
- `template` — base de datos plantilla a partir de la cual se crea la nueva.

Además, a nivel de `spec` (no dentro de `forProvider`), siempre hay que indicar `providerConfigRef.name`, que le dice al recurso qué `ProviderConfig` usar para conectarse.

### 3. Información requerida por el ProviderConfig

El `ProviderConfig` necesita las credenciales de conexión al servidor PostgreSQL. La forma que usamos en este PoC (`source: Secret`) requiere:

- `credentials.source: Secret` — indica que las credenciales vienen de un Secret de Kubernetes (en vez de estar escritas directamente en el manifiesto).
- `credentials.secretRef.namespace` — namespace donde vive el Secret.
- `credentials.secretRef.name` — nombre del Secret.
- `credentials.secretRef.key` — la clave dentro del Secret que contiene la información de conexión.

Esa clave debe contener un JSON con los datos reales de conexión al servidor:

```json
{
  "host": "postgresql.postgresql.svc.cluster.local",
  "port": "5432",
  "username": "postgres",
  "password": "platform123",
  "database": "postgres",
  "sslmode": "disable"
}
```

En pocas palabras: host, puerto, usuario, contraseña, base de datos por defecto a la que conectarse, y el modo SSL.
