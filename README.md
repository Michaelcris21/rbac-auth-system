# Sistema de Autenticación y Autorización

Sistema backend de autenticación y autorización implementado desde cero, sin utilizar el sistema predeterminado de Django.

## 📋 Índice

- [Descripción General](#descripción-general)
- [Decisiones Técnicas](#decisiones-técnicas)
- [Arquitectura del Sistema](#arquitectura-del-sistema)
- [Modelo de Base de Datos](#modelo-de-base-de-datos)
- [Sistema de Permisos](#sistema-de-permisos)
- [Flujo de Autenticación](#flujo-de-autenticación)
- [API Endpoints](#api-endpoints)
- [Instalación y Configuración](#instalación-y-configuración)
- [Datos de Prueba](#datos-de-prueba)
- [Ejemplos de Uso](#ejemplos-de-uso)

## 📖 Descripción General

Este proyecto implementa un sistema completo de autenticación y autorización basado en roles y permisos granulares. El sistema permite controlar el acceso a recursos de manera flexible, distinguiendo entre permisos sobre objetos propios vs todos los objetos.

### Características Principales

- ✅ Autenticación mediante JWT (JSON Web Tokens)
- ✅ Sistema de roles jerárquico
- ✅ Control de acceso granular por recurso
- ✅ Distinción entre permisos "own" (propios) vs "all" (todos)
- ✅ Soft delete de usuarios
- ✅ API administrativa para gestión de permisos
- ✅ Manejo de errores 401 (No autenticado) y 403 (Sin permisos)

## 🛠️ Decisiones Técnicas

### Stack Tecnológico

- **Framework**: Django 4.2 + Django REST Framework 3.14
- **Base de Datos**: PostgreSQL 14+
- **Lenguaje**: Python 3.10
- **Autenticación**: JSON Web Tokens (JWT)
- **Hashing de Contraseñas**: bcrypt
- **Documentación API**: drf-spectacular (OpenAPI/Swagger)

### ¿Por qué estas tecnologías?

**Django REST Framework**: Proporciona herramientas robustas para construir APIs, pero permite implementar autenticación y autorización personalizadas.

**PostgreSQL**: Base de datos relacional que maneja excelentemente relaciones complejas entre roles, permisos y recursos.

**JWT**: Tokens stateless que permiten autenticación sin mantener sesiones en servidor, ideal para APIs RESTful.

**bcrypt**: Algoritmo de hashing diseñado específicamente para contraseñas, con factor de trabajo ajustable y protección contra ataques de fuerza bruta.

## 🏗️ Arquitectura del Sistema

```
┌─────────────────┐
│   Client        │
│  (Web/Mobile)   │
└────────┬────────┘
         │
         │ HTTP Request + JWT Token
         │
┌────────▼────────────────────────────┐
│      Django Middleware              │
│  ┌──────────────────────────────┐  │
│  │  JWTAuthenticationMiddleware │  │
│  │  - Verifica token            │  │
│  │  - Asigna request.user       │  │
│  └──────────────────────────────┘  │
└────────┬────────────────────────────┘
         │
         │ request.user disponible
         │
┌────────▼────────────────────────────┐
│         Views/Endpoints             │
│  ┌──────────────────────────────┐  │
│  │   Permission Checker         │  │
│  │  - Obtiene rol del usuario   │  │
│  │  - Consulta access_rules     │  │
│  │  - Verifica permisos         │  │
│  │  - Permite/Deniega acceso    │  │
│  └──────────────────────────────┘  │
└────────┬────────────────────────────┘
         │
         │ 200 OK / 401 / 403
         │
┌────────▼────────┐
│   PostgreSQL    │
│   Database      │
└─────────────────┘
```

## 💾 Modelo de Base de Datos

### Diagrama de Relaciones

```
┌──────────────────────┐
│       users          │
│──────────────────────│
│ id (PK)              │
│ first_name           │
│ last_name            │
│ patronymic           │
│ email (UNIQUE)       │
│ password_hash        │
│ role_id (FK)         │
│ is_active            │
│ created_at           │
│ updated_at           │
└──────────┬───────────┘
           │
           │ role_id
           │
┌──────────▼───────────┐
│       roles          │
│──────────────────────│
│ id (PK)              │
│ name (UNIQUE)        │
│ description          │
│ level                │
└──────────┬───────────┘
           │
           │
           │
┌──────────▼───────────────────┐      ┌──────────────────────┐
│      access_rules            │      │  business_elements   │
│──────────────────────────────│      │──────────────────────│
│ id (PK)                      │      │ id (PK)              │
│ role_id (FK)                 ├──────┤ name (UNIQUE)        │
│ element_id (FK)              │      │ description          │
│ can_read                     │      │ endpoint_prefix      │
│ can_read_all                 │      └──────────────────────┘
│ can_create                   │
│ can_update_own               │
│ can_update_all               │
│ can_delete_own               │
│ can_delete_all               │
└──────────────────────────────┘
```

### Descripción de Tablas

#### `users`
Almacena la información de usuarios del sistema.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| id | SERIAL | Identificador único |
| first_name | VARCHAR(100) | Nombre |
| last_name | VARCHAR(100) | Apellido |
| patronymic | VARCHAR(100) | Patronímico (opcional) |
| email | VARCHAR(255) | Email único |
| password_hash | VARCHAR(255) | Contraseña hasheada con bcrypt |
| role_id | INTEGER | Referencia a tabla roles |
| is_active | BOOLEAN | Estado del usuario (soft delete) |
| created_at | TIMESTAMP | Fecha de creación |
| updated_at | TIMESTAMP | Última actualización |

#### `roles`
Define los roles disponibles en el sistema.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| id | SERIAL | Identificador único |
| name | VARCHAR(50) | Nombre del rol (admin, manager, user, guest) |
| description | TEXT | Descripción del rol |
| level | INTEGER | Nivel jerárquico (mayor = más privilegios) |

**Roles predefinidos:**
- `admin` (level: 100) - Acceso total al sistema
- `manager` (level: 50) - Gestión de recursos de otros usuarios
- `user` (level: 10) - Usuario estándar con acceso a sus propios recursos
- `guest` (level: 1) - Solo lectura de recursos públicos

#### `business_elements`
Representa los tipos de recursos/objetos del sistema.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| id | SERIAL | Identificador único |
| name | VARCHAR(100) | Nombre del recurso (users, products, orders, permissions) |
| description | TEXT | Descripción del recurso |
| endpoint_prefix | VARCHAR(100) | Prefijo de URL (ej: /api/products/) |

#### `access_rules`
Define las reglas de acceso entre roles y recursos.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| id | SERIAL | Identificador único |
| role_id | INTEGER | Referencia a tabla roles |
| element_id | INTEGER | Referencia a tabla business_elements |
| can_read | BOOLEAN | Puede leer objetos propios |
| can_read_all | BOOLEAN | Puede leer todos los objetos |
| can_create | BOOLEAN | Puede crear objetos |
| can_update_own | BOOLEAN | Puede actualizar objetos propios |
| can_update_all | BOOLEAN | Puede actualizar todos los objetos |
| can_delete_own | BOOLEAN | Puede eliminar objetos propios |
| can_delete_all | BOOLEAN | Puede eliminar todos los objetos |

## 🔐 Sistema de Permisos

### Lógica de Autorización

El sistema implementa un modelo de permisos basado en:

1. **Roles**: Cada usuario tiene asignado un rol
2. **Recursos**: Los elementos del negocio (business_elements)
3. **Reglas**: Matriz de permisos (access_rules) que conecta roles con recursos

### Tipos de Permisos

**Permisos "Own" (Propios):**
- `can_read`: Ver solo recursos propios
- `can_update_own`: Editar solo recursos propios
- `can_delete_own`: Eliminar solo recursos propios

**Permisos "All" (Todos):**
- `can_read_all`: Ver todos los recursos
- `can_update_all`: Editar todos los recursos
- `can_delete_all`: Eliminar todos los recursos

**Permisos Generales:**
- `can_create`: Crear nuevos recursos

### Flujo de Verificación de Permisos

```python
def verificar_permiso(usuario, recurso, accion, objeto_id=None):
    """
    1. Obtener rol del usuario
    2. Buscar regla: access_rules WHERE role_id = usuario.role_id 
                                    AND element_id = recurso.id
    3. Si acción es sobre objeto específico:
       - Verificar si objeto pertenece al usuario (owner_id)
       - Si es propio: verificar permiso "own"
       - Si no es propio: verificar permiso "all"
    4. Retornar True/False
    """
```

### Matriz de Permisos por Defecto

| Rol | Recurso | Read | Read All | Create | Update Own | Update All | Delete Own | Delete All |
|-----|---------|------|----------|--------|------------|------------|------------|------------|
| guest | products | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| guest | orders | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| user | products | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| user | orders | ✅ | ❌ | ✅ | ✅ | ❌ | ✅ | ❌ |
| manager | products | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| manager | orders | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| admin | * | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

## 🔄 Flujo de Autenticación

### 1. Registro de Usuario

```
Cliente                    Servidor                    Base de Datos
  │                           │                              │
  ├─POST /api/auth/register──>│                              │
  │  {                        │                              │
  │    first_name,            │──validate_data()             │
  │    last_name,             │                              │
  │    email,                 │──check_email_exists()───────>│
  │    password,              │<─────────────────────────────┤
  │    password_confirm       │                              │
  │  }                        │──bcrypt.hash(password)       │
  │                           │                              │
  │                           │──create_user()──────────────>│
  │<──201 Created─────────────┤<─────────────────────────────┤
  │  {                        │                              │
  │    id,                    │                              │
  │    email,                 │                              │
  │    role                   │                              │
  │  }                        │                              │
```

### 2. Login (Autenticación)

```
Cliente                    Servidor                    Base de Datos
  │                           │                              │
  ├─POST /api/auth/login─────>│                              │
  │  {                        │                              │
  │    email,                 │──get_user_by_email()────────>│
  │    password               │<─────────────────────────────┤
  │  }                        │                              │
  │                           │──bcrypt.verify(password)     │
  │                           │                              │
  │                           │──jwt.encode(user_id)         │
  │<──200 OK──────────────────┤                              │
  │  {                        │                              │
  │    token: "eyJ...",       │                              │
  │    user: {...}            │                              │
  │  }                        │                              │
```

### 3. Requests Autenticados

```
Cliente                    Middleware                   View
  │                           │                           │
  ├─GET /api/products────────>│                           │
  │  Authorization:           │                           │
  │  Bearer eyJ...            │──jwt.decode(token)        │
  │                           │                           │
  │                           │──get_user_by_id()         │
  │                           │                           │
  │                           │──request.user = user      │
  │                           │                           │
  │                           ├──────────────────────────>│
  │                           │                           │──check_permissions()
  │                           │                           │
  │<──200 OK──────────────────┴───────────────────────────┤
  │  [productos]              │                           │
```

### 4. Logout

```
Cliente                    Servidor
  │                           │
  ├─POST /api/auth/logout────>│
  │  Authorization:           │
  │  Bearer eyJ...            │──invalidate_token()
  │                           │  (opcional: blacklist)
  │<──200 OK──────────────────┤
  │  {                        │
  │    message: "Logout"      │
  │  }                        │
```

### Detalles de Implementación JWT

**Estructura del Token:**
```json
{
  "user_id": 123,
  "email": "user@example.com",
  "role_id": 3,
  "iat": 1640000000,
  "exp": 1640086400
}
```

**Headers de Request:**
```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Expiración:**
- Token válido por 24 horas por defecto
- Después de expiración: error 401 Unauthorized

## 📡 API Endpoints

### Autenticación

#### POST `/api/auth/register`
Registro de nuevo usuario.

**Request:**
```json
{
  "first_name": "Juan",
  "last_name": "Pérez",
  "patronymic": "Carlos",
  "email": "juan@example.com",
  "password": "SecurePass123!",
  "password_confirm": "SecurePass123!"
}
```

**Response: 201 Created**
```json
{
  "id": 123,
  "email": "juan@example.com",
  "first_name": "Juan",
  "last_name": "Pérez",
  "role": "user"
}
```

#### POST `/api/auth/login`
Autenticación de usuario.

**Request:**
```json
{
  "email": "juan@example.com",
  "password": "SecurePass123!"
}
```

**Response: 200 OK**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": 123,
    "email": "juan@example.com",
    "role": "user"
  }
}
```

#### POST `/api/auth/logout`
Cierre de sesión.

**Headers:**
```
Authorization: Bearer {token}
```

**Response: 200 OK**
```json
{
  "message": "Logout exitoso"
}
```

#### GET `/api/auth/me`
Obtener información del usuario autenticado.

**Headers:**
```
Authorization: Bearer {token}
```

**Response: 200 OK**
```json
{
  "id": 123,
  "email": "juan@example.com",
  "first_name": "Juan",
  "last_name": "Pérez",
  "role": "user",
  "is_active": true
}
```

#### PUT `/api/auth/me`
Actualizar perfil del usuario autenticado.

**Headers:**
```
Authorization: Bearer {token}
```

**Request:**
```json
{
  "first_name": "Juan Carlos",
  "last_name": "Pérez López"
}
```

**Response: 200 OK**

#### DELETE `/api/auth/me`
Eliminar cuenta (soft delete).

**Headers:**
```
Authorization: Bearer {token}
```

**Response: 200 OK**
```json
{
  "message": "Cuenta eliminada exitosamente"
}
```

### Recursos Mock (Ejemplos)

#### GET `/api/products`
Listar productos (mock).

**Headers:**
```
Authorization: Bearer {token}
```

**Response: 200 OK**
```json
[
  {"id": 1, "name": "Producto A", "price": 100},
  {"id": 2, "name": "Producto B", "price": 200}
]
```

#### GET `/api/orders`
Listar órdenes (mock).

**Headers:**
```
Authorization: Bearer {token}
```

**Response: 200 OK** (usuario ve solo sus órdenes)
```json
[
  {"id": 1, "user_id": 123, "total": 300, "status": "pending"}
]
```

#### POST `/api/orders`
Crear orden (mock).

**Headers:**
```
Authorization: Bearer {token}
```

**Request:**
```json
{
  "product_id": 1,
  "quantity": 2
}
```

**Response: 201 Created**

### Administración (Solo Admin)

#### GET `/api/admin/permissions`
Listar todas las reglas de acceso.

**Headers:**
```
Authorization: Bearer {admin_token}
```

**Response: 200 OK**
```json
[
  {
    "id": 1,
    "role": "user",
    "element": "orders",
    "can_read": true,
    "can_read_all": false,
    "can_create": true,
    "can_update_own": true,
    "can_update_all": false,
    "can_delete_own": true,
    "can_delete_all": false
  }
]
```

#### PUT `/api/admin/permissions/{id}`
Actualizar regla de acceso.

**Headers:**
```
Authorization: Bearer {admin_token}
```

**Request:**
```json
{
  "can_update_all": true
}
```

**Response: 200 OK**

### Códigos de Error

| Código | Descripción |
|--------|-------------|
| 400 | Bad Request - Datos inválidos |
| 401 | Unauthorized - No autenticado o token inválido |
| 403 | Forbidden - Sin permisos para realizar la acción |
| 404 | Not Found - Recurso no encontrado |
| 500 | Internal Server Error |

## 🚀 Instalación y Configuración

### Requisitos Previos

- Python 3.10+
- PostgreSQL 14+
- pip y virtualenv

### Paso 1: Clonar y Configurar Entorno

```bash
# Clonar repositorio
git clone <repository_url>
cd auth-system

# Crear entorno virtual
python3.10 -m venv venv
source venv/bin/activate  # Linux/Mac
# venv\Scripts\activate  # Windows

# Instalar dependencias
pip install -r requirements.txt
```

### Paso 2: Configurar Base de Datos

```bash
# Crear base de datos en PostgreSQL
psql -U postgres
CREATE DATABASE auth_system_db;
CREATE USER auth_user WITH PASSWORD 'secure_password';
GRANT ALL PRIVILEGES ON DATABASE auth_system_db TO auth_user;
\q
```

### Paso 3: Variables de Entorno

Crear archivo `.env`:

```env
SECRET_KEY=your-secret-key-here
DEBUG=True
DATABASE_NAME=auth_system_db
DATABASE_USER=auth_user
DATABASE_PASSWORD=secure_password
DATABASE_HOST=localhost
DATABASE_PORT=5432
JWT_SECRET_KEY=your-jwt-secret-key
JWT_EXPIRATION_HOURS=24
```

### Paso 4: Migraciones y Datos Iniciales

```bash
# Ejecutar migraciones
python manage.py makemigrations
python manage.py migrate

# Cargar datos de prueba
python manage.py loaddata fixtures/initial_data.json
```

### Paso 5: Ejecutar Servidor

```bash
python manage.py runserver
```

La API estará disponible en `http://localhost:8000/api/`

## 🧪 Datos de Prueba

El archivo `fixtures/initial_data.json` incluye:

### Usuarios de Prueba

| Email | Password | Rol |
|-------|----------|-----|
| admin@test.com | admin123 | admin |
| manager@test.com | manager123 | manager |
| user@test.com | user123 | user |
| guest@test.com | guest123 | guest |

### Roles Predefinidos

- **admin**: Acceso total
- **manager**: Gestión de recursos de otros usuarios
- **user**: Acceso a recursos propios
- **guest**: Solo lectura

### Business Elements

- users
- products
- orders
- permissions

### Access Rules

Matriz completa de permisos según tabla mostrada anteriormente.

## 📝 Ejemplos de Uso

### Ejemplo 1: Registro y Login

```bash
# Registrar usuario
curl -X POST http://localhost:8000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "first_name": "María",
    "last_name": "García",
    "email": "maria@test.com",
    "password": "Pass123!",
    "password_confirm": "Pass123!"
  }'

# Login
curl -X POST http://localhost:8000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "maria@test.com",
    "password": "Pass123!"
  }'

# Respuesta incluye token JWT
```

### Ejemplo 2: Acceso a Recursos

```bash
# Ver productos (guest puede)
curl -X GET http://localhost:8000/api/products \
  -H "Authorization: Bearer {guest_token}"

# Crear orden (guest NO puede -> 403)
curl -X POST http://localhost:8000/api/orders \
  -H "Authorization: Bearer {guest_token}" \
  -H "Content-Type: application/json" \
  -d '{"product_id": 1, "quantity": 2}'

# Crear orden (user SÍ puede -> 201)
curl -X POST http://localhost:8000/api/orders \
  -H "Authorization: Bearer {user_token}" \
  -H "Content-Type: application/json" \
  -d '{"product_id": 1, "quantity": 2}'
```

### Ejemplo 3: Gestión de Permisos (Admin)

```bash
# Listar permisos
curl -X GET http://localhost:8000/api/admin/permissions \
  -H "Authorization: Bearer {admin_token}"

# Modificar permiso
curl -X PUT http://localhost:8000/api/admin/permissions/5 \
  -H "Authorization: Bearer {admin_token}" \
  -H "Content-Type: application/json" \
  -d '{"can_update_all": true}'
```

## 🧠 Conceptos Clave Implementados

### Autenticación vs Autorización

**Autenticación** (¿Quién eres?):
- Proceso de verificar la identidad del usuario
- Implementado mediante login con email/password
- Genera JWT token como prueba de identidad
- Token válido por 24 horas

**Autorización** (¿Qué puedes hacer?):
- Proceso de verificar permisos del usuario
- Basado en roles y reglas de acceso
- Verifica cada request contra access_rules
- Retorna 403 si sin permisos

### JWT (JSON Web Tokens)

**Ventajas:**
- Stateless (no requiere almacenar sesiones)
- Contiene información del usuario (claims)
- Firmado criptográficamente (seguro)
- Compatible con arquitecturas distribuidas

**Estructura:**
```
header.payload.signature
```

### Hashing con bcrypt

**¿Por qué bcrypt?**
- Diseñado específicamente para passwords
- Factor de trabajo ajustable (cost factor)
- Incluye salt automático
- Resistente a ataques de fuerza bruta y rainbow tables

**Ejemplo:**
```python
import bcrypt

# Hash
password = "SecurePass123!"
hashed = bcrypt.hashpw(password.encode('utf-8'), bcrypt.gensalt())

# Verify
is_valid = bcrypt.checkpw(password.encode('utf-8'), hashed)
```

### Control de Acceso Basado en Roles (RBAC)

**Modelo implementado:**
1. Usuario → tiene → Rol
2. Rol → tiene → Permisos sobre Recursos
3. Permisos distinguen entre "own" y "all"

**Flexibilidad:**
- Fácil agregar nuevos roles
- Fácil agregar nuevos recursos
- Granularidad fina (7 tipos de permisos)
- Escalable

## 📚 Estructura del Proyecto

```
auth-system/
├── core/
│   ├── __init__.py
│   ├── models.py              # Modelos: User, Role, AccessRule, BusinessElement
│   ├── serializers.py         # DRF Serializers
│   ├── views.py               # Endpoints de autenticación
│   ├── permissions.py         # Custom permission classes
│   ├── middleware.py          # JWT Authentication Middleware
│   └── admin.py               # Django admin config
├── auth_utils/
│   ├── __init__.py
│   ├── jwt_handler.py         # Generación y validación JWT
│   ├── password_handler.py    # Hashing con bcrypt
│   └── decorators.py          # @require_permission
├── mock_resources/
│   ├── __init__.py
│   └── views.py               # Mock endpoints (products, orders)
├── fixtures/
│   └── initial_data.json      # Datos de prueba
├── config/
│   ├── __init__.py
│   ├── settings.py            # Django settings
│   ├── urls.py                # URL routing
│   └── wsgi.py
├── requirements.txt
├── manage.py
├── .env.example
└── README.md
```

## 🔒 Consideraciones de Seguridad

1. **Passwords**: Nunca almacenadas en texto plano, siempre hasheadas con bcrypt
2. **JWT Secret**: Debe ser fuerte y mantenerse secreto
3. **HTTPS**: En producción, siempre usar HTTPS
4. **SQL Injection**: Protección mediante ORM de Django
5. **CORS**: Configurar apropiadamente para producción
6. **Rate Limiting**: Implementar para prevenir ataques de fuerza bruta
7. **Token Expiration**: Tokens expiran después de 24 horas

## 🎯 Casos de Uso Demostrados

### Caso 1: Usuario Guest
```
✅ Puede ver productos
❌ No puede crear órdenes
❌ No puede ver órdenes de otros
```

### Caso 2: Usuario Regular
```
✅ Puede ver productos
✅ Puede crear sus propias órdenes
✅ Puede editar/eliminar sus propias órdenes
❌ No puede ver/editar órdenes de otros
```

### Caso 3: Manager
```
✅ Puede ver todos los productos
✅ Puede crear productos
✅ Puede ver todas las órdenes
✅ Puede editar/eliminar cualquier orden
```

### Caso 4: Administrador
```
✅ Acceso total a todos los recursos
✅ Puede gestionar permisos del sistema
✅ Puede asignar/modificar roles
```

## 📖 Recursos Adicionales

- [Django REST Framework Documentation](https://www.django-rest-framework.org/)
- [JWT.io - JWT Debugger](https://jwt.io/)
- [bcrypt Documentation](https://github.com/pyca/bcrypt/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)

## 👨‍💻 Autor

Desarrollado como prueba técnica para Effective Mobile.

---

**Nota**: Este sistema está diseñado con propósitos educativos y de demostración. Para uso en producción, considere implementaciones adicionales de seguridad como rate limiting, refresh tokens, blacklist de tokens, auditoría de accesos, etc.
