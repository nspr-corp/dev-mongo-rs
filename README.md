# ReplicaSet de MongoDB para Desarrollo

Este conjunto de configuraciones de Docker está diseñado para facilitar el desarrollo local con MongoDB, configurando un ReplicaSet de un solo nodo que es necesario para operaciones de transacciones con Prisma.

## Dockerfile

Este Dockerfile crea una imagen de MongoDB y configura el servicio para ejecutarse como un ReplicaSet, también inicializa un usuario administrador.

```dockerfile
FROM mongo:4

# Iniciar MongoDB como un ReplicaSet de un solo nodo
ENTRYPOINT mongod --port $MONGO_REPLICA_PORT --replSet rs0 --bind_ip 0.0.0.0 && MONGODB_PID=$!; \
# Inicializar el ReplicaSet
INIT_REPL_CMD="rs.initiate({_id: 'rs0', members: [{_id: 0, host: '$MONGO_REPLICA_HOST:$MONGO_REPLICA_PORT' }]});"; \
# Crear usuario administrador
INIT_USER_CMD="db.getSiblingDB('admin').createUser({user: '$MONGO_INITDB_ROOT_USERNAME', pwd: '$MONGO_INITDB_ROOT_PASSWORD', roles: [{role: 'root', db: 'admin'}]});"; \
# Esperar a que el ReplicaSet esté listo y luego ejecutar los comandos de inicialización
until mongo admin --port $MONGO_REPLICA_PORT --eval "$INIT_REPL_CMD && $INIT_USER_CMD"; do sleep 1; done; \
# Mantener el contenedor en ejecución
echo "REPLICA SET ONLINE"; wait $MONGODB_PID;
```

## Docker Compose

El archivo `docker-compose.yml` se utiliza para orquestar el contenedor de MongoDB configurado como un ReplicaSet.

```yaml
version: "3.8"

services:
  rs:
    build: .
    container_name: "mongo-rs"
    restart: "always"
    ports:
      - "${MONGO_PORT}:${MONGO_REPLICA_PORT}"
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_USER}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_PASS}
      MONGO_INITDB_DATABASE: ${MONGO_DB}
      MONGO_REPLICA_HOST: ${MONGO_HOST}
      MONGO_REPLICA_PORT: ${MONGO_REPLICA_PORT}
    volumes:
      - ./data:/data/db
```

## Configuración de Variables de Entorno

Copia el archivo `.env.example` a un nuevo archivo llamado `.env` y actualiza las variables con tus valores específicos. Este archivo `.env` será utilizado por `docker-compose` para establecer las variables de entorno necesarias para tu contenedor de MongoDB.

```sh
CLIENT_NAME=CLIENT_NAME

MONGO_USER=${CLIENT_NAME}
MONGO_PASS=MONGO_PASS
MONGO_PORT=MONGO_PORT
MONGO_REPLICA_PORT=MONGO_REPLICA_PORT
MONGO_REPLICA_HOST=MONGO_REPLICA_HOST
MONGO_DB=MONGO_DB
```

Asegúrate de no subir tu archivo `.env` a tu repositorio, ya que puede contener información sensible. En su lugar, puedes subir `.env.example` sin valores reales para mostrar la estructura del archivo `.env`.

## Instrucciones de Comandos

Para levantar tu contenedor de MongoDB utilizando Docker Compose, ejecuta el siguiente comando:

- En **Windows**:

  ```sh
  docker-compose up -d
  ```

- En **Linux**:

  ```sh
  sudo docker compose up -d
  ```

## Configuración de Prisma

```sh
DATABASE_URL="mongodb://MONGO_USER:MONGO_PASS@MONGO_HOST:MONGO_PORT/MONGO_DB?authSource=admin&replicaSet=rs0&directConnection=true"
```

Reemplaza `MONGO_USER`, `MONGO_PASS`, `MONGO_HOST`, `MONGO_PORT`, y `MONGO_DB` con tus propias credenciales y datos de conexión. La opción directConnection=true es necesaria si no estás utilizando un servicio de descubrimiento de réplicas.
