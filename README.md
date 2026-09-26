# SProject — API de Reservas para Restaurante

API REST para gestionar reservas de mesas en un restaurante. Permite registrar
clientes, consultar la disponibilidad de mesas por fecha y turno, y crear,
modificar o cancelar reservas, validando que no haya choques de horario ni
grupos más grandes de lo que la mesa admite.

## Enunciado

> Un cliente puede hacer una reserva para una mesa en un turno específico.
> La API debe validar que la mesa esté disponible en la fecha y turno
> solicitados. Implementar GET, POST, PUT, DELETE.

## Stack técnico

| Componente | Tecnología |
|---|---|
| Lenguaje | Java 17 |
| Framework | Spring Boot |
| Persistencia | Spring Data JPA / Hibernate |
| Base de datos | PostgreSQL (Amazon RDS) |
| Build | Maven |
| Contenedor | Docker |
| Reverse proxy | Nginx |

## Modelo de dominio

Tres entidades relacionadas entre sí:


Customer (1) ──< Reservation >── (1) RestaurantTable


- *Customer*: cliente que hace la reserva. Nombre, email (único) y teléfono opcional.
- *RestaurantTable*: mesa del restaurante. Número y capacidad.
- *Reservation*: une a un cliente con una mesa en una fecha y un turno (LUNCH o DINNER). Guarda cuántas personas asistirán.

Al arrancar la aplicación por primera vez, se siembran 6 mesas de ejemplo con
capacidades variadas (2, 2, 4, 4, 6 y 8 personas).

## Arquitectura

El proyecto sigue una organización inspirada en Domain-Driven Design, con
cuatro capas dentro del paquete reservation:


com.sandcz.sproject.reservation
├── domain            Entidades, interfaces de repositorio y excepciones de negocio.
│                      No depende de Spring Data ni de ninguna otra capa.
├── application        Casos de uso (ReservationService, CustomerService).
│                      Aquí viven las reglas de negocio.
├── infrastructure     Implementación de los repositorios con Spring Data JPA,
│                      y el sembrado inicial de datos (DataSeeder).
└── api                Controllers REST, DTOs de entrada/salida y el manejador
                       global de excepciones.


La regla de dependencias es api → application → domain, e infrastructure
implementa las interfaces que declara domain. El dominio no sabe nada de
HTTP ni de JPA.

## Reglas de negocio

- *Disponibilidad*: no puede haber dos reservas para la misma mesa, en la
  misma fecha y el mismo turno. Se valida en el servicio (mensaje claro, 409)
  y además está protegida por una restricción UNIQUE en la base de datos,
  por si dos peticiones llegan al mismo tiempo.
- *Capacidad*: el tamaño del grupo no puede superar la capacidad de la
  mesa elegida (400 si se excede).
- *Email único de cliente*: no se permiten dos clientes con el mismo email,
  normalizado a minúsculas antes de guardar.
- *Borrado protegido*: no se puede eliminar un cliente que tiene reservas
  activas; primero hay que cancelarlas.
- *Fecha de la reserva*: debe ser hoy o una fecha futura.

## Endpoints

### Clientes

| Método | Ruta | Descripción |
|---|---|---|
| POST | /customers | Crear cliente |
| GET | /customers | Listar todos los clientes |
| GET | /customers/{id} | Ver un cliente |
| PUT | /customers/{id} | Modificar un cliente |
| DELETE | /customers/{id} | Eliminar un cliente (falla si tiene reservas) |
| GET | /customers/{id}/reservations | Historial de reservas del cliente |

### Mesas

| Método | Ruta | Descripción |
|---|---|---|
| GET | /tables | Listar todas las mesas |
| GET | /tables/available?date=&shift=&partySize= | Mesas libres en una fecha y turno (partySize es opcional) |

### Reservas

| Método | Ruta | Descripción |
|---|---|---|
| POST | /reservations | Crear una reserva |
| GET | /reservations?date=&shift= | Listar reservas (filtros opcionales) |
| GET | /reservations/{id} | Ver una reserva |
| PUT | /reservations/{id} | Modificar una reserva |
| DELETE | /reservations/{id} | Cancelar una reserva |

### Salud

| Método | Ruta | Descripción |
|---|---|---|
| GET | /actuator/health | Estado de la aplicación y su conexión a la base |

## Ejemplos

*Crear un cliente*
bash
curl -X POST http://localhost:8080/customers \
  -H "Content-Type: application/json" \
  -d '{"name":"Ana Perez","email":"ana@mail.com","phone":"7777-0000"}'


*Crear una reserva*
bash
curl -X POST http://localhost:8080/reservations \
  -H "Content-Type: application/json" \
  -d '{"customerId":1,"tableId":1,"partySize":2,"date":"2026-10-15","shift":"DINNER"}'


*Consultar mesas disponibles*
bash
curl "http://localhost:8080/tables/available?date=2026-10-15&shift=DINNER&partySize=4"


### Errores

Los errores se devuelven como ProblemDetail (RFC 7807):

json
{
  "detail": "La mesa 1 ya esta reservada el 2026-10-15 en el turno DINNER",
  "instance": "/reservations",
  "status": 409,
  "title": "Conflict"
}


| Código | Cuándo ocurre |
|---|---|
| 400 | Body inválido (validaciones de campo) o regla de negocio con datos imposibles (capacidad excedida) |
| 404 | El cliente, la mesa o la reserva referenciados no existen |
| 409 | Conflicto: mesa ya reservada, email duplicado, o cliente con reservas al intentar borrarlo |

## Configuración

La aplicación se configura por variables de entorno, con valores por defecto
para desarrollo local en application.properties:

| Variable | Descripción | Default |
|---|---|---|
| SERVER_PORT | Puerto HTTP | 8080 |
| DB_HOST | Host de PostgreSQL | localhost |
| DB_PORT | Puerto de PostgreSQL | 5432 |
| DB_NAME | Nombre de la base de datos | sproject |
| DB_USER | Usuario de la base de datos | postgres |
| DB_PASSWORD | Contraseña de la base de datos | postgres |
| DDL_AUTO | Estrategia de Hibernate para el esquema | update |

La base de datos debe existir de antemano; Hibernate crea las tablas pero no
la base en sí.

## Ejecutar en local

*Requisitos*: JDK 17, Maven, un PostgreSQL accesible.

bash
# Levantar un Postgres local de prueba
docker run -d --name postgres-dev \
  -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=sproject \
  -p 5432:5432 postgres:16

# Ejecutar la aplicación
./mvnw spring-boot:run


## Construir la imagen Docker

bash
docker build -t sproject .
docker run -d --name backend -p 8080:8080 \
  -e DB_HOST=tu-rds.amazonaws.com \
  -e DB_USER=... -e DB_PASSWORD=... \
  sproject


## Infraestructura

El backend se despliega en una instancia EC2 dentro de una subnet privada,
detrás de un Nginx en una instancia EC2 pública que actúa como reverse proxy
y balanceador. La base de datos vive en una instancia de Amazon RDS
(PostgreSQL), separada del resto del tráfico por su propio security group,
que solo acepta conexiones desde la instancia del backend.

## Frontend

La API no impone un frontend específico. Para consumirla desde una app en
otro origen durante el desarrollo (por ejemplo con Vite), se recomienda
configurar un proxy que reenvíe las peticiones al backend y evite problemas
de CORS. En producción, lo más simple es servir el frontend y la API bajo el
mismo dominio a través de Nginx.