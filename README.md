# Mr. Sushi — Sistema de pedidos multi-sede

Sistema de pedidos y gestión de cocina para la cadena Mr. Sushi. Cada una de las
8 sedes físicas opera como un tenant independiente: su propia cola de cocina,
despacho y reparto. Clientes y programa de puntos (Neki Puntos) son
compartidos entre todas las sedes.

## Estructura del repositorio

```
web/
├── mrsushi-backend-main/     Backend serverless (AWS Lambda + DynamoDB + Step Functions)
├── frontend-trabajadores/    Panel web para el personal (React + Vite)
└── mrsushi_clientes/         Sitio web público para clientes (HTML/CSS/JS estático)
```

## Arquitectura

```
mrsushi_clientes ──┐
                    ├──► API Gateway (HTTP API) ──► Lambda ──► DynamoDB
frontend-trabajadores ┘                              │
                                                       └──► Step Functions
                                                            (flujo de cocina)
```

### Backend: 6 microservicios independientes (`mrsushi-backend-main/`)

Cada carpeta `ms-*` es un servicio Serverless Framework separado, con su
propio `serverless.yml`, tabla(s) DynamoDB y funciones Lambda.

| Servicio | Responsabilidad | Endpoints |
|---|---|---|
| `ms-sedes` | Registro de las 8 sedes (nombre, dirección, coordenadas, radio de cobertura) | `GET /sedes` |
| `ms-autenticacion` | Cuentas de trabajadores (Cocinero, Despachador, Repartidor, Admin), cada una atada a **una sede** | `POST /auth/register`, `POST /auth/login`, `GET /auth/me`, `GET /auth/workers` |
| `ms-clientes` | Cuentas de clientes — **globales**, no atadas a sede — direcciones, Neki Puntos | `POST /clientes/register`, `POST /clientes/login`, `GET /clientes/me`, `PATCH /clientes/me`, `POST/GET /clientes/me/direcciones`, `GET /clientes/me/neki-puntos`, `PATCH /clientes/{clienteId}/neki-puntos` |
| `ms-pedidos` | Creación y consulta de pedidos. Resuelve y valida la sede en el servidor (nunca confía en el cliente) | `POST /pedidos`, `GET /pedidos/{pedidoId}`, `GET /pedidos` |
| `ms-flujo-trabajo` | Avance del pedido por las etapas de cocina, invocado por Step Functions y por el personal | `POST /flujo-trabajo/completar`, `GET /flujo-trabajo/{pedidoId}` |
| `ms-stepfunctions` | Máquina de estados que orquesta el flujo del pedido | (sin API propia) |

### Flujo del pedido (Step Functions)

```
RECIBIDO
   │
   ▼
COCCIÓN (Cocinero)
   │
   ▼
EMPAQUETADO (Despachador)
   │
   ├── si es "para llevar" ──► LISTO PARA RECOGER ──► ENTREGADO
   │
   └── si es delivery ──► EN REPARTO (Repartidor) ──► ENTREGADO
```

Cada etapa espera (`waitForTaskToken`) a que el trabajador con el rol
correspondiente presione "Completar" en el panel antes de avanzar. El pedido
se atiende por orden de llegada como **sugerencia visual** (el más antiguo se
muestra primero en la cola), no hay bloqueo forzado: cualquier trabajador
puede completar cualquier pedido de su cola.

### Modelo de datos (DynamoDB)

Todas las tablas usan claves con nombres semánticos (no `PK`/`SK` genéricos).

| Tabla | Partition key | Sort key | GSI | Notas |
|---|---|---|---|---|
| `MrSushiSedes` | `sedeId` | — | — | Las 8 sedes, sembradas con `ms-sedes/seed.js` |
| `MrSushiUsuarios` | `sedeId` | `email` | — | Trabajadores |
| `MrSushiUsuarioEmailLocks` | `email` | — | — | Garantiza email único de trabajador vía `TransactWriteItems` |
| `MrSushiClientes` | `clienteId` | `itemType` (`PERFIL` / `DIRECCION#{id}`) | — | Clientes, globales |
| `MrSushiClienteEmailLocks` | `email` | — | — | Garantiza email único de cliente |
| `MrSushiPedidos` | `sedeId` | `pedidoId` | `ClienteIndex` (clienteId+createdAt), `SedeCreatedIndex` (sedeId+createdAt) | Pedidos |
| `MrSushiContadores` | `sedeId` | `fecha` | — | Contador diario para el número de ticket amigable |
| `MrSushiFlujoTrabajo` | `pedidoId` | `step` | — | Registro de cada etapa del flujo |

### Aislamiento entre sedes

- Un trabajador solo ve y opera pedidos de **su propia sede** — el `sedeId` se
  lee siempre del JWT verificado, nunca de un parámetro que mande el cliente.
- Un pedido de delivery resuelve su sede automáticamente en el servidor:
  calcula la sede activa más cercana a las coordenadas del cliente y rechaza
  el pedido si ninguna sede cubre esa dirección (`coverageRadius`).
- Login de trabajadores y de clientes no requiere saber la sede de antemano:
  una tabla de "locks" por email resuelve a qué sede/cliente pertenece esa
  cuenta antes de verificar la contraseña.

## Frontends

### `frontend-trabajadores/` — Panel de personal

React 19 + Vite. Login/registro (con selector de sede), cola de pedidos por
rol, métricas del día ("Logros de hoy") y métricas por rol
(Cocinero/Despacho/Repartidor, con selector para el Admin).

Variables de entorno (`.env.production`):
```
VITE_AUTH_API_URL=...      # ms-autenticacion
VITE_PEDIDOS_API_URL=...   # ms-pedidos
VITE_FLUJO_API_URL=...     # ms-flujo-trabajo
VITE_SEDES_API_URL=...     # ms-sedes
```

### `mrsushi_clientes/` — Sitio público de clientes

HTML/CSS/JS estático, sin build. Menú, carrito, checkout, cuenta (Neki
Puntos, mis pedidos), selección de sede/dirección con geolocalización.

Configuración en `src/js/api-config.js` (variables `window.MR_SUSHI_*`).

## Desplegar

Requiere credenciales de AWS con permisos para desplegar vía Serverless
Framework (Lambda, DynamoDB, API Gateway, Step Functions, IAM role existente).

```bash
cd mrsushi-backend-main
npm run install:all
npm run deploy:all        # despliega los 6 servicios en orden
node ms-sedes/seed.js      # solo la primera vez: siembra las 8 sedes
```

Cada frontend se compila/empaqueta por separado y se sube manualmente a AWS
Amplify (Hosting → "Deploy without Git" con el `.zip` generado):

```bash
# frontend-trabajadores
cd frontend-trabajadores && npm install && npm run build
cd dist && zip -r ../mrsushi-trabajadores-amplify.zip . -x ".*"

# mrsushi_clientes (sin build, es estático)
cd mrsushi_clientes && zip -r mrsushi-clientes-amplify.zip . -x ".git/*" -x "*.zip"
```

## Limitaciones conocidas

- **Backend corre sobre AWS Academy Learner Lab**: es una cuenta educativa
  temporal. La app sigue funcionando normalmente entre sesiones del lab (no
  depende de que la sesión local esté activa), pero al terminar el curso la
  cuenta completa se recicla y todo se pierde — no está pensado como hosting
  permanente de un negocio real.
- El orden de atención (FIFO) es solo una sugerencia visual, no se fuerza.
- `ms-menu`, `ms-orders` y `ms-workflow` existieron en versiones anteriores y
  fueron retirados por no usarse (el menú se sirve como HTML estático).
