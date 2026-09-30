Servicio de Devoluciones

Microservicio REST para el registro de devoluciones de productos del sistema de tienda en línea, desarrollado como parte del proyecto de la materia Proyecto para el Desarrollo de Inteligencia de Negocios (Universidad Autónoma de Occidente, UAdeO).

Este servicio corresponde al módulo de Devoluciones y permite realizar las operaciones de:

Registrar la devolución de un producto comprado
Validar la venta, el producto, el cliente y el motivo de la devolución
Aceptar o rechazar la devolución según las reglas de negocio
Registrar el reembolso de las devoluciones aceptadas
Regresar al inventario los productos devueltos
Consultar el historial de devoluciones

El servicio expone una API REST utilizando HTTPS y JSON para permitir su integración con otros servicios del sistema. Incluye además una interfaz web sencilla para probarlo.

Tecnologías utilizadas
Capa	Tecnología
Servidor / API REST	Node.js + Express
Base de datos	SQLite (módulo integrado node:sqlite de Node.js)
Interfaz web	HTML, CSS y JavaScript (sin frameworks)
Formato de mensajes	JSON
Protocolo	HTTPS (en local corre sobre HTTP)
Arquitectura e integración con otros servicios
Navegador / otros servicios  --(HTTP + JSON)-->  API REST (server.js)  -->  Base de datos (tienda.db)

Los demás servicios deben comunicarse con Devoluciones mediante la API REST y no mediante consultas directas a la base de datos.

Estructura del proyecto
DEVOLUCIONES-SOA/
├── public/
│   └── index.html        Interfaz web (formulario, resultado e historial)
├── server.js             API REST y reglas de negocio
├── schema.sql            Estructura de la base de datos (tablas)
├── seed.sql              Datos de prueba (clientes, productos y ventas)
├── package.json          Dependencias y scripts
├── .gitignore            Archivos que no se suben al repositorio
├── README.md             Documentación
└── tienda.db             Base de datos (se genera sola al iniciar)
Requisitos
Node.js v22.5 o superior (incluye SQLite, no hay que instalar nada más)
Git
Un editor de código (por ejemplo, Visual Studio Code)

Para comprobar la versión de Node.js:

bash
node -v
Instalación

Clonar el repositorio:

bash
git clone https://github.com/RAMON777XD/DEVOLUCIONES-SOA.git

Entrar al proyecto:

bash
cd DEVOLUCIONES-SOA

Instalar las dependencias:

bash
npm install
Configuración

El proyecto no necesita variables de entorno obligatorias. De forma opcional, se puede cambiar el puerto (por defecto es 3000):

bash
# Windows (PowerShell)
$env:PORT=3001; npm start

# Linux / macOS
PORT=3001 npm start
Base de datos

El proyecto incluye dos scripts SQL:

schema.sql: crea las tablas.
seed.sql: carga datos de prueba (12 clientes, 15 productos y 14 ventas).

No hay que ejecutarlos a mano: al iniciar, el servidor crea tienda.db, las tablas y, si la base está vacía, carga los datos de prueba.

Tablas principales:

Tabla	Descripción
clientes	Datos de los clientes
productos	Catálogo con precio y existencias
ventas	Encabezado de la venta, estado de pago y referencia del pago
detalle_venta	Productos, cantidades y precios de cada venta
devoluciones_venta	Devoluciones registradas (aceptadas y rechazadas)
reembolsos	Reembolso emitido por cada devolución aceptada
devoluciones_compra	Devoluciones a proveedor (definida en el modelo; sin endpoint por ahora)

Para reiniciar la base de datos (se pierden las devoluciones registradas), detener el servidor, borrar tienda.db y volver a iniciar:

bash
# Windows (PowerShell)
Remove-Item tienda.db

# Linux / macOS
rm tienda.db
Ejecución
bash
npm start

Al iniciar debe aparecer:

Servicio de Devoluciones en http://localhost:3000

Abrir en el navegador http://localhost:3000.

Endpoints
Devoluciones
Método	Ruta	Descripción
POST	/api/devoluciones	Registra una devolución
GET	/api/devoluciones	Lista el historial de devoluciones
GET	/api/devoluciones/:id	Consulta una devolución
Consultas y datos de apoyo
Método	Ruta	Descripción
GET	/api/ventas/:id	Consulta una venta y sus productos disponibles para devolver
GET	/api/clientes	Lista los clientes
GET	/api/productos	Lista los productos
POST	/api/clientes	Agrega un cliente
POST	/api/productos	Agrega un producto
POST	/api/ventas	Registra una venta pagada y descuenta existencias
Registrar una devolución

POST /api/devoluciones

Datos de entrada

json
{
  "id_venta": 3,
  "id_producto": 1,
  "cantidad": 1,
  "motivo": "Llegó con defecto",
  "fecha_devolucion": "2026-09-29",
  "cliente": {
    "nombre": "Jesus Ramon Mata",
    "email": "04ramon54@gmail.com"
  }
}

Datos de salida (devolución aceptada, código 201)

json
{
  "numero_devolucion": 1,
  "estado": "aceptada",
  "mensaje": "Devolución aceptada. Reembolso procesado.",
  "fecha_procesamiento": "2026-09-29T21:19:41.000Z",
  "reembolso": {
    "monto": 450,
    "moneda": "MXN",
    "metodo": "tarjeta (pago PAY-1003)",
    "referencia": "RB-1790000000000-123",
    "estado": "procesado"
  }
}

Datos de salida (devolución rechazada, código 200)

json
{
  "numero_devolucion": 2,
  "estado": "rechazada",
  "mensaje": "Devolución rechazada: Fuera del plazo de devolución (30 días).",
  "fecha_procesamiento": "2026-09-29T21:20:10.000Z"
}
Reglas de negocio

Una devolución se acepta solo si cumple todas estas reglas. Si alguna falla, se registra como rechazada con el motivo:

La venta existe.
El correo del cliente coincide con el de la venta.
La venta está pagada.
El producto pertenece a la venta.
La fecha de devolución no es anterior a la de la venta.
La devolución se hace dentro de los 30 días posteriores a la compra.
La cantidad no supera lo comprado menos lo ya devuelto.

Cuando se acepta, en una sola transacción (o se guarda todo o no se guarda nada) el servicio:

guarda la devolución,
registra el reembolso con su referencia,
regresa los productos al inventario.

Nota: el envío del dinero está simulado en la función procesarReembolso de server.js. Para reembolsar dinero real se conecta una pasarela de pago (Stripe, Mercado Pago o PayPal) en ese punto.

Códigos HTTP

La API utiliza códigos HTTP para indicar el resultado de las operaciones:

Código	Significado
200	Consulta correcta o devolución rechazada por regla de negocio
201	Recurso creado (devolución aceptada, cliente, producto o venta)
400	Datos de entrada inválidos o incompletos
404	Recurso no encontrado
409	Conflicto (correo duplicado o existencias insuficientes)
502	Falló el procesamiento del reembolso; no se registró la devolución
Datos de prueba

Correo = el del cliente de la venta.

Venta	Correo	Resultado esperado
1	ana@correo.com	Aceptada
3	04ramon54@gmail.com	Aceptada
5	maria.torres@correo.com	Aceptada
12	roberto.chavez@correo.com	Aceptada (hace 29 días, último día del plazo)
2	carlos@correo.com	Rechazada (hace 60 días)
13	paola.mendoza@correo.com	Rechazada (hace 31 días)
4	ana@correo.com	Rechazada (venta sin pagar)
11	daniela.vega@correo.com	Rechazada (venta sin pagar)
Seguridad
Un servicio externo no necesita conocer la base de datos de Devoluciones; la comunicación se realiza mediante la API REST.
En producción, el servicio debe ejecutarse detrás de HTTPS (por ejemplo, con un proxy como Nginx o un hosting que incluya certificado).
No se suben al repositorio la base de datos (tienda.db) ni la carpeta node_modules.
Proyecto académico

Servicio desarrollado como parte del proyecto de la materia Proyecto para el Desarrollo de Inteligencia de Negocios, 7.º semestre, Universidad Autónoma de Occidente (UAdeO).

Autor: Jesús Ramon Mata

## Cómo correrlo
1. Instala Node.js (v18 o superior).
2. En esta carpeta: `npm install`
3. `npm start`
4. Abre http://localhost:3000

Se crea sola la base de datos `tienda.db` (SQLite) con datos de prueba.
Prueba: venta 1, correo `ana@correo.com` (dentro del plazo) · venta 2, `carlos@correo.com` (fuera de plazo -> rechazada).

## API (JSON)
- POST /api/devoluciones
  Entrada:
  {"id_venta":1,"id_producto":1,"cantidad":1,"motivo":"Llegó dañado","fecha_devolucion":"2026-09-29","cliente":{"nombre":"Ana López","email":"ana@correo.com"}}
  Salida:
  {"numero_devolucion":1,"estado":"aceptada","mensaje":"...","fecha_procesamiento":"...","reembolso":{"monto":450,"moneda":"MXN","metodo":"..."}}
- GET /api/devoluciones · GET /api/devoluciones/:id · GET /api/ventas/:id

## HTTPS
En local corre en HTTP. En producción ponlo detrás de un proxy con certificado (Nginx, Render, Railway, etc.) o usa `https.createServer` con tu certificado.

## Para fusionarlo con el proyecto del equipo
- Si ya tienen tablas de clientes/productos/ventas, borra esas del `schema.sql` y ajusta los nombres de columnas en las consultas de `server.js`.
- Si el front va en otro servidor, cambia la constante `API` en `public/index.html`.
