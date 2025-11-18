# Documentación Técnica - Licitar

## Índice

1. [Arquitectura](#arquitectura)
2. [Stack Tecnológico](#stack-tecnológico)
3. [Base de Datos](#base-de-datos)
4. [APIs y Endpoints](#apis-y-endpoints)
5. [WebSocket y Notificaciones](#websocket-y-notificaciones)
6. [Algoritmo de Pool de Proveedores](#algoritmo-de-pool-de-proveedores)
7. [Seguridad](#seguridad)
8. [Manejo de Errores](#manejo-de-errores)

---

## Arquitectura

### Patrón de Diseño

El proyecto sigue una **arquitectura en capas** (Layered Architecture):

```
┌─────────────────────────────────────┐
│         HTTP/WebSocket Layer        │
│         (Actix-Web Routes)          │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│         Handlers Layer               │
│  - Validación de inputs             │
│  - Transformación de requests       │
│  - Respuestas HTTP                   │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│         Services Layer               │
│  - Lógica de negocio                │
│  - Orquestación de operaciones      │
│  - Validaciones de negocio          │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│         Repositories Layer           │
│  - Acceso a base de datos           │
│  - Queries SQL (sqlx)               │
│  - Transacciones                     │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│         Models Layer                │
│  - Estructuras de datos             │
│  - Validaciones (validator)         │
│  - Serialización (serde)            │
└─────────────────────────────────────┘
```

### Flujo de una Request

1. **Request HTTP** → Actix-Web recibe el request
2. **Routing** → Se enruta al handler correspondiente
3. **Middleware** → `AuthUser` extrae y valida el usuario
4. **Handler** → Valida inputs, llama al servicio
5. **Service** → Ejecuta lógica de negocio, llama a repositorios
6. **Repository** → Ejecuta queries SQL, retorna datos
7. **Response** → Se serializa y retorna al cliente

---

## Stack Tecnológico

### Framework y Runtime

- **Actix-Web 4**: Framework web asíncrono para Rust
- **Tokio**: Runtime asíncrono con soporte para I/O no bloqueante
- **Actix**: Sistema de actores para WebSocket y notificaciones

### Base de Datos

- **PostgreSQL 12+**: Base de datos relacional
- **sqlx 0.8**: Cliente PostgreSQL con type-safe queries en tiempo de compilación
- **Transacciones**: Soporte para transacciones ACID

### Autenticación y Seguridad

- **Argon2**: Algoritmo de hashing de contraseñas (resistente a ataques)
- **Tokens de sesión**: Generados con `rand` y codificados en base64
- **Cookies HTTP-only**: Almacenamiento seguro de tokens

### Validación y Serialización

- **validator**: Validación de datos de entrada
- **serde/serde_json**: Serialización y deserialización JSON

### Utilidades

- **uuid**: Generación de UUIDs v4
- **chrono**: Manejo de fechas y timestamps
- **rust_decimal**: Precisión decimal para montos monetarios
- **actix-multipart**: Manejo de archivos multipart

### Email

- **lettre**: Cliente SMTP para envío de emails
- **resend-rs**: Cliente Resend API (alternativa a SMTP)
- **Templates HTML**: Templates profesionales para emails

---

## Base de Datos

### Esquemas Principales

#### `auth` Schema

- **users**: Usuarios del sistema (compradores y proveedores)
  - Campos de jerarquía: `parent_user_id`, `is_root_admin`, `hierarchy_level`, `hierarchy_path`
  - Campos de empresa: `company_id`
  - Campos adicionales: `dni`, `phone`
- **sessions**: Sesiones activas con tokens
- **roles**: Roles del sistema
- **user_roles**: Asignación de roles a usuarios
- **tokens**: Tokens de verificación y recuperación de contraseña

#### `business` Schema

- **companies**: Empresas que agrupan jerarquías de usuarios
- **tenders**: Licitaciones
  - Campo de auditoría: `created_by_user_id`
- **tender_tags**: Relación licitación-tags con pesos
- **tender_revision_history**: Historial de revisiones
- **tender_status_history**: Historial de cambios de estado
- **offers**: Ofertas de proveedores
- **offer_status_history**: Historial de estados de ofertas
  - Campo de auditoría: `rejection_reason`
- **offer_evaluations**: Evaluaciones de ofertas
- **award_decisions**: Decisiones de adjudicación
  - Campos de auditoría: `award_reason`, `award_notes`, `created_by_user_id`
- **vendor_pool**: Pool de proveedores para licitaciones
- **vendor_pool_config**: Configuración del algoritmo de matching
- **buyer_profiles**: Perfiles de compradores
- **vendor_profiles**: Perfiles de proveedores
- **vendor_tags**: Tags asociados a proveedores
- **tags**: Catálogo de tags jerárquico
- **attachments**: Archivos adjuntos (licitaciones y ofertas)

### Tipos Enum

- **user_type_enum**: `buyer`, `vendor`
- **tender_status_enum**: `open`, `closed`, `awarded`
- **tender_type_enum**: `normal`, `encrypted`, `inverse`
- **vendor_pool_type_enum**: `public`, `fixed`
- **offer_status_enum**: `pending`, `won`, `lost`

### Índices y Constraints

- Índices en `users.email` (único)
- Índices en `sessions.session_token`
- Foreign keys entre tablas relacionadas
- Constraints de integridad referencial

---

## APIs y Endpoints

### Autenticación (`/auth`)

- `POST /auth/register`: Registro de usuario (crea admin root)
- `POST /auth/login`: Inicio de sesión (elimina sesiones anteriores)
- `POST /auth/logout`: Cierre de sesión
- `GET /auth/me`: Obtener usuario autenticado (incluye jerarquía y empresa)
- `POST /auth/verify-email`: Verificación de email con token
- `POST /auth/forgot-password`: Solicitud de recuperación de contraseña
- `POST /auth/reset-password`: Reset de contraseña con token
- `POST /auth/change-password`: Cambio de contraseña (usuario autenticado)

### Gestión de Jerarquía (`/auth/admin`)

- `POST /auth/admin/create-user`: Crear usuario en la jerarquía
- `GET /auth/admin/users`: Listar usuarios de la jerarquía
- `PATCH /auth/admin/users/{user_id}`: Actualizar usuario
- `DELETE /auth/admin/users/{user_id}`: Eliminar usuario (con cascada)
- `GET /auth/admin/hierarchy`: Obtener árbol de jerarquía
- `GET /auth/admin/validate-permission`: Validar permisos

### Licitaciones (`/tenders`)

- `POST /tenders/tender`: Crear licitación
- `GET /tenders/tender/{id}`: Obtener licitación completa
- `PATCH /tenders/tender/{id}`: Actualizar licitación
- `POST /tenders/tender/{id}/status`: Cambiar estado
- `GET /tenders`: Listar licitaciones de un usuario
- `POST /tenders/tender/{id}/close`: Cerrar licitación
- `POST /tenders/tender/{id}/award`: Adjudicar licitación
- `POST /tenders/tender/{id}/vendors`: Agregar proveedores al pool
- `GET /tenders/{id}/vendors`: Obtener pool de proveedores
- `GET /tenders/tags`: Listar todos los tags
- `GET /tenders/{id}/tags`: Obtener tags de una licitación
- `POST /tenders/tender/{id}/attachments`: Subir adjunto
- `GET /tenders/tender/{id}/attachments`: Listar adjuntos
- `GET /tenders/attachments/{id}/download`: Descargar adjunto
- `DELETE /tenders/attachments/{id}`: Eliminar adjunto

### Ofertas (`/offers`)

- `POST /offers`: Crear oferta
- `GET /offers/{id}`: Obtener oferta completa
- `PATCH /offers/{id}`: Actualizar oferta
- `POST /offers/{id}/status`: Cambiar estado de oferta
- `GET /offers/tender/{tender_id}`: Listar ofertas de una licitación
- `GET /offers/buyer/{buyer_id}`: Listar ofertas para un comprador
- `GET /offers/vendor`: Listar ofertas del proveedor autenticado
- `POST /offers/{id}/attachments`: Subir adjunto a oferta
- `GET /offers/{id}/attachments`: Listar adjuntos de oferta
- `GET /offers/attachments/{id}/download`: Descargar adjunto
- `DELETE /offers/attachments/{id}`: Eliminar adjunto

### Perfiles (`/profile`)

- `POST /profile/vendor`: Crear/actualizar perfil de proveedor (crea empresa si es admin root)
- `POST /profile/buyer`: Crear/actualizar perfil de comprador (crea empresa si es admin root)
- `POST /profile/vendor/tags`: Actualizar tags de proveedor
- `GET /profile/vendor/tags`: Obtener tags del proveedor
- `GET /profile/buyer/{user_id}`: Obtener perfil de comprador
- `GET /profile/vendor/{user_id}`: Obtener perfil de proveedor
- `GET /profile/vendors`: Listar todos los proveedores (con filtros y paginación)
- `GET /profile/company/me`: Obtener información de la empresa del usuario autenticado

### Estadísticas (`/dashboard`)

- `GET /dashboard/buyer/stats`: Estadísticas para comprador
- `GET /dashboard/vendor/stats`: Estadísticas para proveedor
- `GET /dashboard/supervisor/stats`: Estadísticas para supervisor

### Administración (`/admin`)

- `GET /admin/tenders/all`: Listar todas las licitaciones (solo supervisores)
- `GET /admin/users`: Listar todos los usuarios (solo supervisores)
- `PUT /admin/users/{user_id}/status`: Actualizar estado de usuario
- `GET /admin/reports/tenders`: Reportes de licitaciones

### Auditoría (`/admin/audit`)

- `GET /admin/audit/tenders/by-user`: Auditoría de licitaciones por usuario
- `GET /admin/audit/response-times`: Análisis de tiempos de respuesta
- `GET /admin/audit/award-reasons`: Auditoría de razones de adjudicación
- `GET /admin/audit/vendor-selection-patterns`: Patrones de selección de proveedores

### Analytics (`/admin/analytics`)

- `GET /admin/analytics/dashboard`: Dashboard analítico avanzado (solo admin root)
- `GET /admin/analytics/users/{user_id}/report`: Reporte detallado de usuario (solo admin root)

### WebSocket (`/ws/notifications`)

- `WS /ws/notifications`: Conexión WebSocket para notificaciones en tiempo real

---

## WebSocket y Notificaciones

### Arquitectura

El sistema de notificaciones usa **Actix Actors**:

```
┌─────────────────────────────────────┐
│    NotificationServer (Actor)       │
│  - Gestiona sesiones WebSocket     │
│  - Envía notificaciones            │
└──────────────┬──────────────────────┘
               │
       ┌───────┴────────┐
       │                │
┌──────▼──────┐  ┌──────▼──────┐
│ NotificationWs │  │ NotificationWs │
│  (Actor)       │  │  (Actor)       │
│  - Heartbeat   │  │  - Heartbeat   │
│  - Ping/Pong   │  │  - Ping/Pong   │
└───────────────┘  └───────────────┘
```

### Flujo de Notificaciones

1. Cliente establece conexión WebSocket
2. `NotificationWs` actor se crea y registra en `NotificationServer`
3. Se mantiene heartbeat cada 5 segundos
4. Cuando hay un evento (ej: cambio en licitación), `NotificationService` envía mensaje al `NotificationServer`
5. `NotificationServer` busca todas las sesiones del usuario y envía la notificación
6. `NotificationWs` serializa y envía el JSON al cliente

### Tipos de Notificaciones

- `TenderCreated`: Nueva licitación creada
- `TenderUpdated`: Licitación actualizada
- `TenderClosed`: Licitación cerrada
- `TenderAwarded`: Licitación adjudicada
- `OfferCreated`: Nueva oferta creada
- `OfferUpdated`: Oferta actualizada
- `OfferDeleted`: Oferta eliminada
- `OfferAccepted`: Oferta aceptada
- `OfferRejected`: Oferta rechazada

---

## Algoritmo de Pool de Proveedores

### Descripción

El algoritmo calcula un **score de matching** entre proveedores y licitaciones basado en:
- Tags compartidos (con jerarquía)
- Expansión jerárquica de tags (padres e hijos)
- Configuración de umbrales y factores de ponderación

**📖 Documentación completa:** Ver [`docs/algoritmo_vendor_pool.md`](algoritmo_vendor_pool.md)

### Componentes

1. **Configuración** (`vendor_pool_config`):
   - `threshold`: Umbral mínimo de score normalizado (0-1)
   - `min_pool_size`: Tamaño mínimo del pool
   - `hierarchy_factor`: Factor de ponderación para tags jerárquicos (0-1)
   - `effective_from`: Fecha de vigencia de la configuración

2. **Cálculo de Scores**:
   - **raw_score**: Suma de pesos de tags coincidentes (incluyendo expansión jerárquica)
   - **normalized_score**: Score normalizado (0-1) basado en el mejor score encontrado
   - **matched_base_tags**: Cantidad de tags base (depth=1) que coinciden

3. **Proceso**:
   - Se obtienen los tags de la licitación (con pesos)
   - Se expanden jerárquicamente usando CTE recursivo (hasta 3 niveles)
   - Se calculan coincidencias con tags de proveedores
   - Se aplican factores de jerarquía a tags padres
   - Se normaliza el score dividiendo por el máximo
   - Se filtran por threshold (`qualified_vendors`)
   - Se completa con fallback hasta alcanzar `min_pool_size`

### Tipos de Pool

- **Public**: Pool generado automáticamente por el algoritmo de matching
- **Fixed**: Pool manual, proveedores seleccionados por el comprador

### Funcionalidades Implementadas

-  Generación automática al crear licitaciones públicas
-  Regeneración automática al actualizar tags de licitaciones públicas
-  Gestión manual de proveedores (agregar/eliminar)
-  Persistencia de scores para análisis y auditoría
-  Configuración histórica con `effective_from`

---

## Seguridad

### Autenticación

- **Tokens de sesión**: 32 bytes aleatorios codificados en base64
- **Almacenamiento**: Tokens en base de datos, no en memoria
- **Cookies**: HTTP-only, Secure, SameSite=Strict
- **Validación**: Middleware `AuthUser` valida cada request

### Contraseñas

- **Hashing**: Argon2 (resistente a GPU, ASIC, side-channel)
- **Salt**: Generado aleatoriamente por usuario
- **Nunca se almacenan en texto plano**

### Autorización

- **Middleware**: `AuthUser` extrae usuario del request
- **Validación de permisos**: En handlers y servicios
- **Roles**: Sistema de roles para permisos granulares

### Validación de Inputs

- **validator**: Validación de tipos, rangos, formatos
- **Sanitización**: Prevención de SQL injection (sqlx usa prepared statements)
- **Límites**: Validación de tamaños de archivos, campos, etc.

---

## Manejo de Errores

### Estructura de Errores

```rust
pub enum AuthErrorCode {
    DbError,
    DuplicateEmail,
    InvalidEmail,
    InvalidPassword,
    InvalidCurrentPassword,
    ActiveSession,
    ValidationError,
    ArgonError,
}

pub struct AuthErrorResponse {
    pub success: bool,
    pub code: AuthErrorCode,
    pub message: String,
}
```

### Códigos HTTP

- `200 OK`: Operación exitosa
- `400 Bad Request`: Error de validación o datos inválidos
- `401 Unauthorized`: No autenticado o sesión inválida
- `403 Forbidden`: Sin permisos para la operación
- `404 Not Found`: Recurso no encontrado
- `500 Internal Server Error`: Error del servidor

### Validación de Errores

```json
{
  "errors": {
    "email": ["el email no es válido"],
    "password": ["la pass debe tener >=6 chars"]
  }
}
```

---

## Consideraciones de Performance

### Base de Datos

- **Índices**: En campos frecuentemente consultados
- **Queries optimizadas**: Uso de CTEs, JOINs eficientes
- **Connection pooling**: PgPool con máximo de conexiones

### Async/Await

- **I/O no bloqueante**: Todas las operaciones de DB son async
- **Tokio runtime**: Multiplexing eficiente de tareas
- **Actix-Web**: Manejo concurrente de requests

### WebSocket

- **Heartbeat**: Mantiene conexiones activas, detecta desconexiones
- **Timeout**: 30 segundos sin respuesta cierra la conexión
- **Actors**: Sistema eficiente de mensajería

---

## Testing

### Estrategia

- **Unit tests**: Para funciones puras, validaciones
- **Integration tests**: Para endpoints, flujos completos
- **Database tests**: Con base de datos de prueba

### Ejecución

```bash
# Todos los tests
cargo test

# Tests específicos
cargo test test_nombre

# Con logs
RUST_LOG=debug cargo test
```

---

## Deployment

### Compilación

```bash
cargo build --release
```

### Variables de Entorno

- `DATABASE_URL`: URL de conexión a PostgreSQL
- `RUST_LOG`: Nivel de logging (debug, info, warn, error)

### Consideraciones

- **Migrations**: Ejecutar antes del despliegue
- **Connection pool**: Configurar según carga esperada
- **Logging**: Configurar rotación de logs
- **Monitoring**: Health checks, métricas

