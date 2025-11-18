# Documentación Funcional - Licitar

## Índice

1. [Introducción](#introducción)
2. [Actores del Sistema](#actores-del-sistema)
3. [Casos de Uso](#casos-de-uso)
4. [Flujos Principales](#flujos-principales)
5. [Reglas de Negocio](#reglas-de-negocio)
6. [Estados y Transiciones](#estados-y-transiciones)

---

## Introducción

**Licitar** es un sistema de gestión de licitaciones que permite a organizaciones (compradores) publicar licitaciones y recibir ofertas de proveedores. El sistema incluye matching inteligente de proveedores, notificaciones en tiempo real, sistema de jerarquía de usuarios, gestión de empresas, y auditoría completa.

---

## Actores del Sistema

### Admin Root (Comprador o Proveedor)

- **Rol**: Usuario raíz que representa una empresa completa
- **Nivel**: `hierarchy_level = 0`, `is_root_admin = TRUE`
- **Capacidades**:
  - Crear y gestionar licitaciones (si es buyer)
  - Crear supervisores y usuarios finales
  - Gestionar toda la jerarquía de usuarios
  - Completar perfil de empresa (crea la empresa)
  - Ver estadísticas globales de su jerarquía
  - Acceso a dashboard de analytics (solo admin root)

### Supervisor (Comprador o Proveedor)

- **Rol**: Usuario intermedio que gestiona usuarios finales
- **Nivel**: `hierarchy_level = 1`, `is_root_admin = FALSE`
- **Capacidades**:
  - Crear usuarios finales (buyer/vendor del mismo tipo que el admin root)
  - Gestionar usuarios que creó
  - Ver licitaciones creadas por usuarios de su jerarquía
  - Acceso a auditoría de licitaciones y ofertas
  - Ver estadísticas de supervisión

### Usuario Final (Comprador o Proveedor)

- **Rol**: Usuario que realiza operaciones de negocio
- **Nivel**: `hierarchy_level = 2`, `is_root_admin = FALSE`
- **Capacidades**:
  - Crear y gestionar licitaciones (si es buyer)
  - Crear y gestionar ofertas (si es vendor)
  - Ver estadísticas personales
  - Gestionar perfil personal

### Comprador (Buyer)

- **Rol**: Organización que necesita adquirir bienes o servicios
- **Puede ser**: Admin Root, Supervisor o Usuario Final
- **Capacidades**:
  - Crear y gestionar licitaciones
  - Seleccionar proveedores para pool fijo
  - Evaluar ofertas
  - Adjudicar licitaciones
  - Ver estadísticas de sus licitaciones
  - Recibir notificaciones por email cuando se crean ofertas

### Proveedor (Vendor)

- **Rol**: Empresa que ofrece bienes o servicios
- **Puede ser**: Admin Root, Supervisor o Usuario Final
- **Capacidades**:
  - Crear y actualizar perfil
  - Asociar tags a su perfil
  - Ver licitaciones disponibles (según matching)
  - Crear y gestionar ofertas
  - Ver estadísticas de sus ofertas
  - Recibir notificaciones por email de invitaciones a licitaciones

### Sistema

- **Rol**: Automatización y procesos internos
- **Capacidades**:
  - Matching automático de proveedores
  - Generación de notificaciones (WebSocket y Email)
  - Cálculo de estadísticas
  - Gestión de sesiones
  - Envío de emails (verificación, recuperación, notificaciones)
  - Auditoría y analytics

---

## Casos de Uso

### CU-01: Registro de Usuario

**Actor**: Usuario nuevo

**Precondiciones**: 
- El usuario no existe en el sistema

**Flujo Principal**:
1. Usuario completa formulario de registro (nombre, email, contraseña, tipo)
2. Sistema valida datos (email válido, contraseña >= 8 caracteres con mayúscula, minúscula y número)
3. Sistema hashea la contraseña con Argon2
4. Sistema crea usuario en base de datos con:
   - `is_root_admin = TRUE`
   - `hierarchy_level = 0`
   - `parent_user_id = NULL`
   - `hierarchy_path = <user_id>`
   - `company_id = NULL` (se asigna al completar perfil)
5. Sistema crea perfil vacío según tipo (buyer/vendor)
6. Sistema genera token de verificación de email (válido 24 horas)
7. Sistema envía email de verificación
8. Sistema crea sesión y retorna token

**Postcondiciones**:
- Usuario creado como admin root y autenticado
- Perfil básico creado
- Sesión activa
- Email de verificación enviado

**Excepciones**:
- Email duplicado → Error "el usuario o correo ya está en uso"
- Validación fallida → Error con detalles de campos inválidos

---

### CU-02: Inicio de Sesión

**Actor**: Usuario registrado

**Precondiciones**:
- Usuario existe en el sistema
- Credenciales válidas

**Flujo Principal**:
1. Usuario envía email y contraseña
2. Sistema busca usuario por email
3. Sistema verifica contraseña con Argon2
4. Sistema elimina sesiones anteriores del usuario
5. Sistema crea nueva sesión
6. Sistema retorna token de sesión (cookie HTTP-only)

**Postcondiciones**:
- Sesión activa
- Token almacenado en cookie
- Sesiones anteriores eliminadas

**Excepciones**:
- Email no existe → Error "credenciales inválidas"
- Contraseña incorrecta → Error "credenciales inválidas"
- Usuario inactivo → Error "cuenta desactivada"
- Sesiones anteriores se eliminan automáticamente (no hay mensaje de sesión activa)

---

### CU-03: Verificación de Email

**Actor**: Usuario registrado

**Precondiciones**:
- Usuario tiene token de verificación válido

**Flujo Principal**:
1. Usuario hace clic en link de verificación del email
2. Frontend llama a `POST /auth/verify-email` con token
3. Sistema valida token (debe existir y no estar expirado)
4. Sistema marca email como verificado
5. Sistema invalida el token

**Postcondiciones**:
- Email verificado
- Token invalidado

**Excepciones**:
- Token inválido o expirado → Error "token inválido o expirado"

---

### CU-04: Recuperación de Contraseña

**Actor**: Usuario registrado

**Precondiciones**:
- Usuario existe en el sistema

**Flujo Principal**:
1. Usuario solicita recuperación (`POST /auth/forgot-password`)
2. Sistema genera token de recuperación (válido 1 hora)
3. Sistema envía email con link de recuperación
4. Usuario hace clic en link y completa formulario
5. Sistema valida token y nueva contraseña
6. Sistema actualiza contraseña
7. Sistema invalida el token

**Postcondiciones**:
- Contraseña actualizada
- Token invalidado

**Excepciones**:
- Token inválido o expirado → Error "token inválido o expirado"
- Contraseña débil → Error "contraseña no cumple requisitos"

---

### CU-05: Crear Usuario en Jerarquía

**Actor**: Admin Root o Supervisor

**Precondiciones**:
- Usuario autenticado es admin root o supervisor
- Datos válidos

**Flujo Principal**:
1. Admin/Supervisor completa formulario (nombre, email, contraseña, tipo)
2. Sistema valida permisos:
   - Admin root puede crear supervisor (`hierarchy_level = 1`) o usuario (`hierarchy_level = 2`)
   - Supervisor solo puede crear usuario (`hierarchy_level = 2`)
   - El tipo debe coincidir con el tipo del admin root
3. Sistema crea usuario con:
   - `parent_user_id = <usuario_autenticado_id>`
   - `hierarchy_level` según permisos
   - `hierarchy_path = generate_hierarchy_path(<nuevo_user_id>, <parent_id>)`
   - `is_root_admin = FALSE`
   - `company_id` heredado del padre
4. Sistema crea perfil vacío
5. Sistema envía email de bienvenida (opcional)

**Postcondiciones**:
- Usuario creado en la jerarquía
- Perfil básico creado
- `company_id` heredado

**Excepciones**:
- Sin permisos → Error 403
- Tipo no coincide con admin root → Error 400
- Email duplicado → Error 409

---

### CU-06: Completar Perfil de Empresa (Admin Root)

**Actor**: Admin Root

**Precondiciones**:
- Usuario es admin root (`is_root_admin = TRUE`)
- Perfil no está completo

**Flujo Principal**:
1. Admin root completa datos de empresa (company_name, cuit, etc.)
2. Sistema valida datos (cuit válido, tags mínimos para vendor)
3. Sistema crea empresa en `business.companies`
4. Sistema asigna `company_id` al admin root
5. Sistema asigna `company_id` a todos los usuarios hijos existentes
6. Sistema actualiza perfil del admin root
7. Sistema marca `profile_complete = true`

**Postcondiciones**:
- Empresa creada
- `company_id` asignado a admin y todos sus hijos
- Perfil completo

**Excepciones**:
- Datos inválidos → Error de validación
- No es admin root → Error 403

---

### CU-07: Crear Licitación

**Actor**: Comprador

**Precondiciones**:
- Usuario autenticado como comprador
- Datos de licitación válidos

**Flujo Principal**:
1. Comprador completa formulario (título, descripción, tipo, fecha finalización, tags, tipo de pool)
2. Sistema valida datos (título obligatorio, etc.)
3. Sistema inicia transacción
4. Sistema crea licitación con estado "open" y `created_by_user_id = <user_id>`
5. Sistema asocia tags con pesos (100, 80, 60, ...)
6. Si pool es "public": Sistema ejecuta algoritmo de matching
7. Si pool es "fixed": Sistema agrega proveedores especificados
8. Sistema crea registro en historial de estado
9. Sistema confirma transacción
10. Sistema envía notificaciones WebSocket a proveedores del pool
11. Sistema envía emails a todos los usuarios de las empresas invitadas

**Postcondiciones**:
- Licitación creada con estado "open"
- Tags asociados
- Pool de proveedores generado
- Notificaciones enviadas (WebSocket y Email)

**Reglas de Negocio**:
- Versión inicial es 1
- Estado inicial es "open"
- Si pool es "public", se genera automáticamente
- Si pool es "fixed", requiere lista de vendor_ids
- `created_by_user_id` se guarda para auditoría

---

### CU-08: Crear Oferta

**Actor**: Proveedor

**Precondiciones**:
- Usuario autenticado como proveedor
- Proveedor está en el pool de la licitación
- Licitación está en estado "open"
- Datos de oferta válidos

**Flujo Principal**:
1. Proveedor selecciona licitación
2. Proveedor completa datos de oferta (monto, detalles, etc.)
3. Sistema valida que proveedor esté en pool
4. Sistema valida que licitación esté "open"
5. Sistema inicia transacción
6. Sistema crea oferta con estado "pending"
7. Sistema asocia oferta a versión actual de licitación
8. Sistema crea registro en historial de estado
9. Sistema confirma transacción
10. Sistema envía notificación WebSocket al comprador
11. Sistema envía email al comprador con detalles de la oferta

**Postcondiciones**:
- Oferta creada con estado "pending"
- Historial de estado registrado
- Notificaciones enviadas (WebSocket y Email)

**Reglas de Negocio**:
- Solo proveedores en el pool pueden ofertar
- Solo se pueden crear ofertas en licitaciones "open"
- El monto es opcional (puede ser oferta encriptada)

---

### CU-09: Adjudicar Licitación

**Actor**: Comprador

**Precondiciones**:
- Usuario autenticado como comprador
- Es el comprador dueño de la licitación
- Licitación está en estado "open" o "closed"
- Existe oferta ganadora

**Flujo Principal**:
1. Comprador selecciona oferta ganadora
2. Sistema valida que comprador sea dueño
3. Sistema verifica si la oferta ganadora es la más barata
4. Si NO es la más barata, sistema requiere `award_reason` (mínimo 10 caracteres)
5. Sistema inicia transacción
6. Sistema actualiza estado de licitación a "awarded"
7. Sistema asocia `winning_offer_id` a la licitación
8. Sistema actualiza estado de oferta ganadora a "won"
9. Sistema actualiza estados de otras ofertas a "lost"
10. Sistema crea decisión de adjudicación con `award_reason`, `award_notes`, `created_by_user_id`
11. Sistema crea registros en historiales
12. Sistema confirma transacción
13. Sistema envía notificaciones a todos los proveedores

**Postcondiciones**:
- Licitación en estado "awarded"
- Oferta ganadora en estado "won"
- Otras ofertas en estado "lost"
- Decisión de adjudicación registrada para auditoría
- Notificaciones enviadas

**Reglas de Negocio**:
- Solo el comprador dueño puede adjudicar
- Una vez adjudicada, no se pueden crear nuevas ofertas
- Todas las ofertas no ganadoras pasan a "lost"
- Si la oferta ganadora NO es la más barata, se requiere razón de adjudicación

---

### CU-10: Matching de Proveedores (Pool Público)

**Actor**: Sistema

**Precondiciones**:
- Licitación creada con pool tipo "public"
- Tags asociados a la licitación

**Flujo Principal**:
1. Sistema obtiene configuración del algoritmo
2. Sistema obtiene tags de la licitación con pesos
3. Sistema ejecuta CTE recursivo para recorrer jerarquía de tags
4. Sistema calcula coincidencias con tags de proveedores
5. Sistema aplica factores de jerarquía
6. Sistema calcula raw_score y normalized_score
7. Sistema filtra proveedores por threshold
8. Sistema asegura min_pool_size
9. Sistema ordena por normalized_score descendente
10. Sistema inserta proveedores en vendor_pool

**Postcondiciones**:
- Pool de proveedores generado
- Scores calculados y almacenados

**Reglas de Negocio**:
- Tags con mayor peso tienen más importancia
- Tags jerárquicos (hijos) tienen factor de ponderación
- Threshold mínimo para incluir en pool
- Si no hay suficientes, se incluyen los mejores disponibles

---

### CU-11: Actualizar Perfil de Proveedor

**Actor**: Proveedor

**Precondiciones**:
- Usuario autenticado como proveedor
- Perfil existe o se creará

**Flujo Principal**:
1. Proveedor completa datos del perfil (empresa, CUIT, dirección, etc.)
2. Sistema valida datos
3. Sistema inicia transacción
4. Si perfil existe: Sistema actualiza datos
5. Si perfil no existe: Sistema crea nuevo perfil
6. Sistema asocia tags al proveedor (si se proporcionan)
7. Sistema confirma transacción

**Postcondiciones**:
- Perfil actualizado o creado
- Tags asociados

**Reglas de Negocio**:
- Upsert: crea si no existe, actualiza si existe
- Tags reemplazan los anteriores (no se agregan)
- Peso de tags se calcula automáticamente (100, 80, 60, ...)

---

### CU-12: Cerrar Licitación

**Actor**: Comprador

**Precondiciones**:
- Usuario autenticado como comprador
- Es el comprador dueño de la licitación
- Licitación está en estado "open"

**Flujo Principal**:
1. Comprador solicita cerrar licitación
2. Sistema valida permisos
3. Sistema inicia transacción
4. Sistema actualiza estado a "closed"
5. Sistema crea registro en historial
6. Sistema confirma transacción
7. Sistema envía notificaciones a proveedores

**Postcondiciones**:
- Licitación en estado "closed"
- No se pueden crear nuevas ofertas
- Notificaciones enviadas

---

## Flujos Principales

### Flujo 1: Ciclo Completo de Licitación

```
1. Comprador crea licitación
   ↓
2. Sistema genera pool de proveedores (si es público)
   ↓
3. Proveedores reciben notificación (WebSocket y Email)
   ↓
4. Proveedores crean ofertas
   ↓
5. Comprador recibe notificación por email de cada oferta
   ↓
6. Comprador evalúa ofertas
   ↓
7. Comprador adjudica licitación (con razón si no es la más barata)
   ↓
8. Proveedores reciben notificación de resultado
```

### Flujo 2: Registro y Configuración de Proveedor

```
1. Usuario se registra como proveedor (admin root)
   ↓
2. Sistema crea perfil vacío
   ↓
3. Sistema envía email de verificación
   ↓
4. Usuario verifica email
   ↓
5. Proveedor completa perfil (crea empresa)
   ↓
6. Proveedor asocia tags
   ↓
7. Sistema calcula matching para licitaciones existentes
   ↓
8. Proveedor ve licitaciones relevantes
```

### Flujo 3: Jerarquía de Usuarios

```
1. Admin root se registra (crea empresa al completar perfil)
   ↓
2. Admin root crea supervisores
   ↓
3. Supervisores crean usuarios finales
   ↓
4. Todos los usuarios heredan company_id
   ↓
5. Admin root gestiona toda la jerarquía
   ↓
6. Supervisor gestiona solo sus usuarios
```

### Flujo 4: Notificaciones en Tiempo Real

```
1. Evento ocurre (ej: nueva licitación)
   ↓
2. NotificationService crea notificación
   ↓
3. NotificationService envía a NotificationServer
   ↓
4. NotificationServer busca sesiones del usuario
   ↓
5. NotificationServer envía a cada sesión WebSocket
   ↓
6. Cliente recibe notificación en tiempo real
```

### Flujo 5: Sistema de Emails

```
1. Evento ocurre (ej: registro de usuario)
   ↓
2. Sistema genera token (si aplica)
   ↓
3. EmailService construye template HTML
   ↓
4. EmailService envía email (SMTP o Resend API)
   ↓
5. En modo development, email_info se retorna en respuesta
   ↓
6. Usuario recibe email con link de acción
```

---

## Reglas de Negocio

### Jerarquía de Usuarios

1. **Registro Público**:
   - Solo puede crear `buyer` o `vendor`
   - Crea admin root (`is_root_admin = TRUE`, `hierarchy_level = 0`)
   - `parent_user_id = NULL`
   - `hierarchy_path = <user_id>`

2. **Admin Root**:
   - Puede crear supervisores (`hierarchy_level = 1`)
   - Puede crear usuarios finales (`hierarchy_level = 2`)
   - Puede ver y gestionar toda su jerarquía
   - Puede eliminar usuarios con cascada

3. **Supervisor**:
   - Solo puede crear usuarios finales (`hierarchy_level = 2`)
   - Solo puede ver y gestionar usuarios que creó
   - NO puede crear otros supervisores

4. **Restricción de Tipo**:
   - Toda la jerarquía debe ser del mismo tipo (buyer o vendor)
   - Si admin root es `vendor`, todos los hijos deben ser `vendor`
   - Si admin root es `buyer`, todos los hijos deben ser `buyer`

5. **Eliminación**:
   - Eliminar supervisor elimina todos sus usuarios hijos (cascada)
   - No se puede eliminar supervisor con hijos si `force=false`

### Sistema de Empresas

1. **Creación**:
   - Se crea cuando admin root completa su perfil
   - Requiere `company_name` y `cuit` (para vendor también requiere tags)
   - Todos los usuarios hijos heredan el `company_id`

2. **Herencia**:
   - Al crear usuario hijo, hereda `company_id` del padre
   - Si admin root aún no tiene empresa, hijos tienen `company_id = NULL`
   - Cuando admin root crea empresa, se actualiza a todos los hijos

3. **Perfil Completo**:
   - Para admin root: `profile_complete = true` cuando empresa está completa
   - Para usuarios hijos: `profile_complete` se calcula según perfil individual

### Licitaciones

1. **Creación**:
   - Título es obligatorio
   - Estado inicial es "open"
   - Versión inicial es 1
   - Solo compradores pueden crear
   - `created_by_user_id` se guarda para auditoría

2. **Estados**:
   - "open": Acepta nuevas ofertas
   - "closed": No acepta nuevas ofertas, pero no adjudicada
   - "awarded": Adjudicada a una oferta

3. **Transiciones de Estado**:
   - "open" → "closed": Por comprador
   - "open" → "awarded": Por comprador (adjudicación)
   - "closed" → "awarded": Por comprador (adjudicación)
   - No se puede volver a "open" una vez cerrada o adjudicada

4. **Pool de Proveedores**:
   - Pool "public": Generado automáticamente por algoritmo
   - Pool "fixed": Proveedores seleccionados manualmente
   - Se puede agregar proveedores a pool "fixed" después de crear
   - Al agregar/eliminar proveedores, se envían emails a todos los usuarios de la empresa

5. **Versiones**:
   - Cada actualización incrementa la versión
   - Las ofertas se asocian a la versión actual al crearse

6. **Notificaciones de Cambios**:
   - Si `notify_participants = true`, se envían emails a todos los proveedores del pool
   - Email incluye `change_summary` con resumen de cambios

### Ofertas

1. **Creación**:
   - Solo proveedores en el pool pueden ofertar
   - Solo en licitaciones "open"
   - Estado inicial es "pending"
   - Monto es opcional (puede ser encriptado)
   - Al crear, se envía email al comprador

2. **Estados**:
   - "pending": Oferta creada, no evaluada
   - "won": Oferta ganadora (adjudicada)
   - "lost": Oferta no ganadora

3. **Transiciones**:
   - "pending" → "won": Por comprador (adjudicación)
   - "pending" → "lost": Automático cuando otra gana
   - No se puede cambiar de "won" o "lost"

4. **Actualización**:
   - Solo el proveedor dueño puede actualizar
   - Solo si está en estado "pending"
   - Solo si la licitación sigue "open"

5. **Rechazo**:
   - Se puede incluir `rejection_reason` al cambiar estado
   - Se guarda en historial para auditoría

### Adjudicación

1. **Validación**:
   - Solo el comprador dueño puede adjudicar
   - Si la oferta ganadora NO es la más barata, se requiere `award_reason` (mínimo 10 caracteres)
   - `award_notes` es opcional

2. **Registro**:
   - Se crea registro en `business.award_decisions`
   - Incluye `award_reason`, `award_notes`, `created_by_user_id`
   - Se usa para auditoría

### Perfiles

1. **Proveedor**:
   - Upsert: crea si no existe, actualiza si existe
   - Tags reemplazan los anteriores (no se agregan)
   - Peso de tags: 100, 80, 60, ... (según orden)
   - Para admin root: completar perfil crea empresa

2. **Comprador**:
   - Upsert: crea si no existe, actualiza si existe
   - No tiene tags
   - Para admin root: completar perfil crea empresa

### Autenticación

1. **Sesiones**:
   - Un usuario puede tener una sola sesión activa
   - Al hacer login, se eliminan sesiones anteriores automáticamente
   - Tokens se almacenan en base de datos
   - Logout elimina la sesión específica

2. **Contraseñas**:
   - Mínimo 8 caracteres
   - Debe incluir mayúscula, minúscula y número
   - Hash con Argon2
   - Nunca se almacenan en texto plano

3. **Verificación de Email**:
   - Token válido por 24 horas
   - Se invalida después de uso
   - Email se marca como verificado

4. **Recuperación de Contraseña**:
   - Token válido por 1 hora
   - Se invalida después de uso
   - Siempre retorna éxito (por seguridad, no revela si email existe)

### Matching de Proveedores

1. **Algoritmo**:
   - Basado en tags compartidos
   - Considera jerarquía de tags
   - Aplica factores de ponderación
   - Filtra por threshold mínimo
   - Asegura min_pool_size

2. **Scores**:
   - raw_score: Suma de pesos de tags coincidentes
   - normalized_score: Score normalizado (0-100)
   - matched_base_tags: Cantidad de tags base coincidentes

### Sistema de Emails

1. **Modos**:
   - `development`: No envía emails reales, retorna `email_info` en respuestas
   - `production`: Envía emails reales por SMTP o Resend API
   - `disabled`: No envía ni loguea emails

2. **Templates**:
   - HTML profesional con estilos
   - Links de acción claros
   - Información relevante del contexto

3. **Proveedores**:
   - SMTP: Configuración tradicional (Gmail, SendGrid, etc.)
   - Resend API: Alternativa moderna con mejor deliverability

### Auditoría

1. **Tracking**:
   - `created_by_user_id` en licitaciones
   - `created_by_user_id` en decisiones de adjudicación
   - `award_reason` y `award_notes` en adjudicaciones
   - `rejection_reason` en rechazos de ofertas

2. **Endpoints**:
   - Solo supervisores y admin root pueden acceder
   - Analytics solo para admin root
   - Filtros por usuario, fechas, estado, etc.

---

## Estados y Transiciones

### Estados de Licitación

```
[open] ──(cerrar)──> [closed]
  │                    │
  │                    │
  └──(adjudicar)───────┴──> [awarded]
```

### Estados de Oferta

```
[pending] ──(adjudicar)──> [won]
    │
    │
    └──(otra gana)──> [lost]
```

### Estados de Sesión

```
[activa] ──(logout)──> [eliminada]
```

### Estados de Verificación de Email

```
[no_verificado] ──(verificar)──> [verificado]
```

---

## Validaciones

### Registro de Usuario

- Email: Formato válido, único en sistema
- Contraseña: Mínimo 8 caracteres, debe incluir mayúscula, minúscula y número
- Nombre completo: Mínimo 5 caracteres
- Tipo: Debe ser "buyer" o "vendor" (no supervisor)

### Licitación

- Título: Obligatorio, mínimo 1 carácter
- Tipo: Debe ser "normal", "encrypted", o "inverse"
- Pool type: Debe ser "public" o "fixed"
- Tags: Array de IDs válidos

### Oferta

- Tender ID: Obligatorio, debe existir
- Monto: Opcional, si existe debe ser >= 0
- Proveedor debe estar en pool de la licitación
- Licitación debe estar "open"

### Perfil

- Campos opcionales pero validados si se proporcionan
- Tags: Array de IDs válidos (si se proporcionan)
- Para admin root: `company_name` y `cuit` son requeridos
- Para vendor admin root: mínimo 1 tag requerido

### Adjudicación

- Si la oferta ganadora NO es la más barata: `award_reason` requerido (mínimo 10 caracteres)
- `award_notes` es opcional

---

## Notificaciones

### Eventos que Generan Notificaciones WebSocket

1. **Licitación creada**: A todos los proveedores del pool
2. **Licitación actualizada**: A todos los proveedores del pool
3. **Licitación cerrada**: A todos los proveedores del pool
4. **Licitación adjudicada**: A todos los proveedores del pool
5. **Oferta creada**: Al comprador de la licitación
6. **Oferta actualizada**: Al comprador de la licitación
7. **Oferta aceptada/rechazada**: Al proveedor dueño de la oferta

### Eventos que Generan Notificaciones por Email

1. **Registro de usuario**: Email de verificación con token (24 horas)
2. **Recuperación de contraseña**: Email con link de recuperación (1 hora)
3. **Invitación a licitación**: Email a todos los usuarios de la empresa invitada
4. **Remoción de licitación**: Email cuando se elimina un proveedor del pool
5. **Cambios en licitación**: Email cuando se actualiza con `notify_participants=true`
6. **Nueva oferta**: Email al comprador cuando un proveedor crea una oferta

### Contenido de Notificaciones

- Tipo de notificación
- Título descriptivo
- Mensaje con detalles
- Timestamp
- Estado de lectura
- Links de acción (en emails)

---

## Estadísticas

### Proveedor

- Total de ofertas creadas
- Ofertas ganadoras
- Ofertas perdidas
- Promedio de montos ofertados
- Licitaciones en las que participó

### Comprador

- Total de licitaciones creadas
- Licitaciones adjudicadas
- Licitaciones cerradas sin adjudicar
- Promedio de ofertas por licitación

### Supervisor

- Total de licitaciones en su jerarquía
- Total de ofertas en su jerarquía
- Usuarios activos
- Estadísticas globales
