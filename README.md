# Licitar

Sistema de gestión de licitaciones desarrollado en Rust con Actix-Web. Permite a compradores crear y gestionar licitaciones, y a proveedores participar con ofertas. Incluye sistema de autenticación, notificaciones en tiempo real vía WebSocket, y algoritmo inteligente de matching de proveedores.

---

## Tabla de Contenido

- [Características Principales](#características-principales)
- [Arquitectura](#arquitectura)
- [Requisitos Previos](#requisitos-previos)
- [Instalación](#instalación)
  - [Instalación de Rust](#instalación-de-rust)
  - [Instalación de Dependencias](#instalación-de-dependencias)
- [Configuración](#configuración)
- [Uso](#uso)
- [Estructura del Proyecto](#estructura-del-proyecto)
- [Documentación](#documentación)

---

## Características Principales

### Autenticación y Autorización
- **Registro y login** de usuarios (compradores y proveedores)
- **Sesiones** basadas en tokens almacenados en base de datos
- **Middleware de autenticación** para proteger endpoints
- **Hashing de contraseñas** con Argon2
- **Sistema de roles** (manager, etc.)
- **Verificación de email** con tokens de 24 horas
- **Recuperación de contraseña** con tokens de 1 hora
- **Cambio de contraseña** para usuarios autenticados

### Sistema de Jerarquía de Usuarios
- **Jerarquía de 3 niveles**: Admin Root (nivel 0), Supervisor (nivel 1), Usuario Final (nivel 2)
- **Gestión de usuarios** por admin y supervisores
- **Restricciones de tipo**: Toda la jerarquía debe ser del mismo tipo (buyer o vendor)
- **Eliminación en cascada** de usuarios hijos
- **Validación de permisos** basada en jerarquía
- **Campos adicionales**: `parent_user_id`, `is_root_admin`, `hierarchy_level`, `hierarchy_path`

### Sistema de Empresas
- **Empresas** que agrupan toda una jerarquía de usuarios
- **Creación automática** de empresa al completar perfil del admin root
- **Herencia de empresa** para todos los usuarios hijos
- **Gestión centralizada** de datos de empresa (company_name, cuit, etc.)
- **Perfil completo** basado en datos de empresa para admin root

### Gestión de Licitaciones
- **Creación y actualización** de licitaciones
- **Estados de licitación**: Open, Closed, Awarded
- **Tipos de licitación**: Normal, Encrypted, Inverse
- **Historial de revisiones** y cambios de estado
- **Tags y categorización** de licitaciones
- **Adjuntos** de archivos para licitaciones
- **Notificaciones por email** al agregar/eliminar proveedores
- **Notificaciones de cambios** a participantes cuando se actualiza una licitación
- **Tracking de creador** (`created_by_user_id`) para auditoría

### Sistema de Ofertas
- **Creación y actualización** de ofertas por proveedores
- **Estados de oferta**: Pending, Won, Lost
- **Ofertas encriptadas** para licitaciones tipo Encrypted
- **Historial de cambios** de estado
- **Evaluación de ofertas** con criterios y scores
- **Notificaciones por email** al comprador cuando se crea una oferta
- **Razones de adjudicación** y rechazo para auditoría
- **Tracking de creador** en decisiones de adjudicación

### Pool de Proveedores
- **Algoritmo inteligente de matching** basado en tags
- **Pool público o fijo** (manual)
- **Cálculo de scores** (raw_score, normalized_score)
- **Matching jerárquico** de tags con factores de ponderación
- **Gestión manual** de proveedores (agregar/eliminar)

### Notificaciones en Tiempo Real
- **WebSocket** para notificaciones push
- **Notificaciones** de cambios en licitaciones y ofertas
- **Sistema de actores** para gestión de conexiones
- **Heartbeat** para mantener conexiones activas

### Sistema de Emails
- **Emails de verificación** al registrar usuario
- **Emails de recuperación** de contraseña
- **Emails de invitación** a licitaciones
- **Emails de notificación** de cambios en licitaciones
- **Emails de nuevas ofertas** para compradores
- **Configuración flexible** (development/production/disabled)
- **Soporte SMTP y Resend API**
- **Templates HTML** profesionales

### Directorio de Proveedores
- **Listado completo** de proveedores disponibles
- **Filtrado por tags** (`?tag_id=1`)
- **Búsqueda por texto** (`?search=empresa`)
- **Paginación opcional** (`?page=1&limit=20`)
- **Información completa**: company_name, cuit, phone, address, description, website, tags, certifications

### Sistema de Auditoría
- **Auditoría de licitaciones** por usuario creador
- **Análisis de tiempos de respuesta** a ofertas
- **Auditoría de razones de adjudicación**
- **Patrones de selección de proveedores**
- **Dashboard de analytics** para admin root
- **Reportes detallados** por usuario

### Estadísticas
- **Estadísticas de proveedores**: ofertas, licitaciones, promedios
- **Estadísticas de compradores**: licitaciones creadas, adjudicaciones
- **Estadísticas de supervisores**: vista global del sistema
- **Dashboard diferenciado** por tipo de usuario

### Perfiles
- **Perfiles de comprador** y **perfiles de proveedor**
- **Tags de proveedores** para matching
- **Certificaciones** y datos de empresa

---

## Arquitectura

El proyecto sigue una arquitectura en capas:

```
┌─────────────────┐
│   Handlers      │  ← Endpoints HTTP/WebSocket
├─────────────────┤
│   Services      │  ← Lógica de negocio
├─────────────────┤
│  Repositories   │  ← Acceso a base de datos
├─────────────────┤
│    Models       │  ← Estructuras de datos
└─────────────────┘
```

- **Handlers**: Manejan requests HTTP, validan inputs, llaman a servicios
- **Services**: Contienen la lógica de negocio, orquestan repositorios
- **Repositories**: Abstraen el acceso a PostgreSQL usando sqlx
- **Models**: Definen estructuras de datos, validaciones, serialización

---

## Requisitos Previos

- **PostgreSQL 12+** con la base de datos configurada
- **Rust 1.70+** (edition 2024)
- **Cargo** (incluido con Rust)
- Conexión a internet para descargar dependencias

---

## Instalación

### Instalación de Rust

#### Linux / macOS

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env
```

#### Windows

1. Descargá el instalador desde [rustup.rs](https://rustup.rs/)
2. Ejecutá `rustup-init.exe`
3. Verificá la instalación:
   ```powershell
   rustc --version
   ```

### Instalación de Dependencias

Las dependencias se instalan automáticamente al compilar:

```bash
cargo build
```

Principales dependencias:
- `actix-web`: Framework web asíncrono
- `sqlx`: Cliente PostgreSQL con type-safe queries
- `argon2`: Hashing de contraseñas
- `validator`: Validación de datos
- `tokio`: Runtime asíncrono
- `actix-web-actors`: Soporte para WebSocket

---

## Configuración

1. **Crear archivo `.env`** en la raíz del proyecto:

```env
DATABASE_URL=postgres://usuario:contraseña@localhost:5432/licitar
RUST_LOG=info

# Email Configuration
EMAIL_ENABLED=true
EMAIL_MODE=development  # development | production | disabled
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USERNAME=tu_email@gmail.com
SMTP_PASSWORD=tu_app_password
EMAIL_FROM=noreply@licitar.com
EMAIL_FROM_NAME=Licit.ar
EMAIL_BASE_URL=http://localhost:3000

# Resend API (alternativa a SMTP)
RESEND_API_KEY=re_xxxxxxxxxxxxx  # Opcional, si se usa Resend en lugar de SMTP
```

2. **Configurar la base de datos**:

```sql
CREATE DATABASE licitar;
```

3. **Ejecutar migraciones** (si aplica):

```bash
sqlx migrate run
```

O importar el schema desde `licitar_schema_only_dump.sql`.

---

## Uso

### Desarrollo

```bash
# Compilar
cargo build

# Ejecutar
cargo run

# Con logs detallados
RUST_LOG=debug cargo run
```

### Producción

```bash
# Compilar en modo release
cargo build --release

# Ejecutar el binario
./target/release/licitar
```

El servidor inicia en `http://localhost:4000` por defecto.

### Testing

```bash
cargo test
```

---

## Estructura del Proyecto

```
licitar/
├── src/
│   ├── main.rs                 # Punto de entrada, configuración del servidor
│   ├── handlers/               # Handlers HTTP/WebSocket
│   │   ├── auth_handlers.rs
│   │   ├── tender_handlers.rs
│   │   ├── offer_handlers.rs
│   │   ├── profile_handlers.rs
│   │   ├── statistics_handlers.rs
│   │   ├── notification_server.rs
│   │   └── notification_ws_handler.rs
│   ├── services/               # Lógica de negocio
│   │   ├── auth_service.rs
│   │   ├── tender_service.rs
│   │   ├── offer_service.rs
│   │   ├── profile_service.rs
│   │   ├── statistics_service.rs
│   │   └── notification_service.rs
│   ├── repositories/           # Acceso a base de datos
│   │   ├── user_repository.rs
│   │   ├── tender_repository.rs
│   │   ├── offer_repository.rs
│   │   ├── profile_repository.rs
│   │   └── role_repository.rs
│   ├── models/                 # Estructuras de datos
│   │   ├── auth_model.rs
│   │   ├── tender_model.rs
│   │   ├── offer_model.rs
│   │   ├── profile_model.rs
│   │   └── notification_model.rs
│   ├── routes/                 # Definición de rutas
│   │   ├── auth_routes.rs
│   │   ├── tender_routes.rs
│   │   ├── offer_routes.rs
│   │   └── ...
│   ├── middleware/             # Middleware (autenticación)
│   │   └── auth_user.rs
│   ├── utils/                  # Utilidades
│   │   ├── token.rs
│   │   └── validation.rs
│   └── errors.rs               # Manejo de errores
├── docs/                       # Documentación
│   ├── tecnica.md             # Documentación técnica
│   ├── funcional.md           # Documentación funcional
│   ├── algoritmo_vendor_pool.md  # Algoritmo de matching de proveedores
│   ├── cambios_frontend.md   # Cambios para frontend
│   ├── directorio_proveedores.md  # Directorio de proveedores
│   └── email_setup.md         # Configuración de emails
├── Cargo.toml                  # Configuración del proyecto
└── README.md                   # Este archivo
```

---

## Documentación

- **[Documentación Técnica](docs/tecnica.md)**: Arquitectura, diseño, APIs, base de datos
- **[Documentación Funcional](docs/funcional.md)**: Casos de uso, flujos, reglas de negocio
- **[Algoritmo de Pool de Proveedores](docs/algoritmo_vendor_pool.md)**: Documentación detallada del algoritmo de matching
- **[Cambios para Frontend](docs/cambios_frontend.md)**: Resumen de cambios implementados
- **[Directorio de Proveedores](docs/directorio_proveedores.md)**: Endpoint y funcionalidades del directorio
- **[Configuración de Emails](docs/email_setup.md)**: Setup y configuración del sistema de emails

---
### Convenciones de Código

- **Nombres**: snake_case para funciones y variables
- **Estructura**: Seguir la arquitectura en capas (handlers → services → repositories)

---