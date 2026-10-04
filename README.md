# CRM ORM/ODM Lab

API REST de un CRM básico que combina un ORM (Sequelize + PostgreSQL) y un ODM (Mongoose + MongoDB).

## Stack

- Node.js 22, Express 5, CommonJS
- Sequelize + PostgreSQL 16 (`User`, `Company`, `Contact`)
- Mongoose + MongoDB 7 (`Activity`)
- Jest + Supertest
- GitHub Codespaces, Dev Containers, Docker Compose
- Supervisor (`npm run dev`)

## Arquitectura

```text
GitHub Codespace
│
├── app       Node.js 22  ──┬── Sequelize ──> postgres (PostgreSQL)
│                           └── Mongoose  ──> mongo    (MongoDB)
├── postgres
└── mongo
```

La aplicación se conecta por nombre de servicio (`postgres`, `mongo`). Las credenciales de desarrollo llegan como variables de entorno definidas en `.devcontainer/docker-compose.yml` (ver `.env.example`).

## Iniciar el Codespace

1. En GitHub: **Code → Codespaces → Create codespace on main**.
2. Espera a que se levanten los tres servicios (`app`, `postgres`, `mongo`). `postCreateCommand` ejecuta `npm install`.

## Instalar dependencias

```bash
npm install
```

## Seed y reset

```bash
npm run seed    # inserta datos deterministas (3 users, 4 companies, 8 contacts, 10 activities)
npm run reset   # elimina y recrea tablas/base de datos y vuelve a sembrar
```

## Iniciar la API

```bash
npm start       # node ./bin/www
npm run dev     # supervisor ./bin/www
```

Servidor en el puerto `3000` (variable `PORT`).

## Pruebas

```bash
npm test
```

Cada suite restablece PostgreSQL y MongoDB antes de ejecutarse y cierra las conexiones al terminar.

## Endpoints

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/health` | Health check |
| GET | `/users` | Listar usuarios |
| GET | `/users/:id` | Obtener usuario |
| POST | `/users` | Crear usuario |
| PUT | `/users/:id` | Actualizar usuario |
| DELETE | `/users/:id` | Eliminar usuario |
| GET | `/companies` | Listar compañías (`?industry=`) |
| GET | `/companies/:id` | Obtener compañía |
| POST | `/companies` | Crear compañía |
| PUT | `/companies/:id` | Actualizar compañía |
| DELETE | `/companies/:id` | Eliminar compañía |
| GET | `/contacts` | Listar contactos |
| GET | `/contacts/:id` | Obtener contacto |
| POST | `/contacts` | Crear contacto |
| PUT | `/contacts/:id` | Actualizar contacto |
| DELETE | `/contacts/:id` | Eliminar contacto |
| GET | `/activities` | Listar actividades (`?type=`) |
| GET | `/activities/:id` | Obtener actividad |
| POST | `/activities` | Crear actividad |
| PUT | `/activities/:id` | Actualizar actividad |
| DELETE | `/activities/:id` | Eliminar actividad |

Los errores se devuelven como JSON: `{ "error": "Contact not found" }`.

## Respuestas
**1. Dos motores.**  
Activity es buena candidata para MongoDB porque su campo `metadata` puede tener estructuras diferentes según sea una llamada, correo o reunión.  
Company y Contact funcionan bien en PostgreSQL porque tienen relaciones claras, como la llave foránea `companyId`, y requieren consistencia entre registros.

**2. ORM vs ODM.**  
Un ORM convierte modelos de objetos en tablas y consultas de una base relacional; en este proyecto es Sequelize con PostgreSQL.  
Un ODM trabaja con documentos de una base no relacional; aquí es Mongoose con MongoDB. Una diferencia es que Sequelize usa tablas y relaciones, mientras Mongoose usa documentos con estructuras más flexibles de usar.

**3. Configuración por variables de entorno.**  
Las credenciales se definen en `.devcontainer/docker-compose.yml` y se consultan mediante variables de entorno, en lugar de escribirlas dentro de archivos JavaScript.  
La app usa `DB_HOST=postgres` y `MONGODB_URI=mongodb://mongo:27017/crm`; no usan `localhost` porque cada servicio corre en un contenedor distinto y se comunica por el nombre del servicio de Docker.

**4. Asociaciones.**  
En `models/sequelize/index.js`, una Company puede tener muchos Contact y cada Contact pertenece a una Company.  
La llave foránea es `companyId`, vive en la tabla de contactos y señala a su compañía. `as: 'contacts'` nos permite obtener los contactos relacionados mediante `include`.

**5. Eager loading.**  
Si primero traigo la compañía y después los contactos, tendría que hacer dos consultas separadas.  
Con `include` se obtienen juntos como parte de la misma consulta lógica y se evita hacer consultas adicionales; es mejor usarla cuando se necesitan varias compañías con sus contactos.

**6. Instancia vs consulta.**  
En `controllers/contacts.js`, buscar el contacto y llamar `contact.update()` nos permite comprobar antes si existe y responder con el objeto actualizado.  
`Model.update()` puede ser útil para actualizar varios registros de una vez, pero normalmente devuelve la cantidad de filas afectadas y no el registro completo.

**7. Esquema flexible.**  
En `models/mongoose/activity.js`, `metadata` usa `mongoose.Schema.Types.Mixed`.  
Esto permite guardar objetos distintos para CALL, EMAIL y MEETING sin cambiar el esquema. La desventaja es que hay menos validación automática sobre los campos internos y sus tipos.

**8. Sin ref.**  
`contactId` y `userId` son números que pertenecen a PostgreSQL, mientras Activity está en MongoDB; por eso Mongoose no puede usar `ref` ni `populate` entre ambas bases de datos.  
Como consecuencia, si se elimina un User o Contact en PostgreSQL, una actividad puede conservar un id que ya no existe y quedar sin integridad referencial automática.

**9. Documento actualizado.**  
Antes de corregir el reto 08, `findByIdAndUpdate()` devolvía el documento anterior porque ese es su comportamiento predeterminado.  
Agregué `new: true` para obtener la versión actualizada y `runValidators: true` para validar los cambios según el esquema de Activity.

**10. Pruebas de comportamiento.**  
Probar la respuesta de la API verifica lo que realmente recibe quien usa el endpoint: estado, datos y errores.  
Esto permite cambiar la implementación interna, por ejemplo de `findAll()` a otra consulta válida, sin romper las pruebas mientras el comportamiento siga siendo correcto.

**11. Repetibilidad.**  
En `tests/setup.js`, antes de cada suite se conectan Sequelize y Mongoose y se restablecen los datos semilla conocidos.  
Al terminar, se cierran ambas conexiones. Así cada prueba inicia con los mismos datos y `npm test` produce resultados repetibles.

**12. Mi experiencia.**  
El reto que me resultó más difícil fue el 08, porque la actualización sí se guardaba, pero la respuesta mostraba los datos anteriores.  
El fallo de Jest donde esperaba la descripción nueva y recibía la anterior me ayudó a identificar el problema. Lo resolví revisando `findByIdAndUpdate()` y agregando `new: true` y `runValidators: true`.

## Evidencia
<img width="490" height="349" alt="image" src="https://github.com/user-attachments/assets/dc726515-0a03-41dd-9fe2-4dd3812e3950" />

