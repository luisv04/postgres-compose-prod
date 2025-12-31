# PostgreSQL Docker Compose - Producción

Servicio PostgreSQL listo para producción usando Docker Compose V2, configurado para uso interno en localhost.

## Descripción

Este servicio despliega PostgreSQL como base de datos interna de infraestructura, accesible únicamente desde `localhost` (127.0.0.1). Diseñado para ser consumido por servicios locales como n8n, Flowise, APIs backend, etc.

## Versión de PostgreSQL

Se utiliza **PostgreSQL 16** (imagen oficial `postgres:16`), versión estable y LTS recomendada para producción. La versión está fijada explícitamente para garantizar reproducibilidad y evitar cambios inesperados.

## Requisitos

- Docker Engine 20.10+
- Docker Compose V2 (`docker compose`, no `docker-compose`)
- Al menos 1GB de espacio libre para datos

## Configuración

1. Copia el archivo de ejemplo y configura tus variables:

```bash
cp .env.example .env
```

2. Edita `.env` y configura los valores:

```env
POSTGRES_DB=myapp_db
POSTGRES_USER=postgres
POSTGRES_PASSWORD=tu_password_seguro
POSTGRES_PORT=5432
```

**Importante:** Usa una contraseña fuerte en producción. El archivo `.env` está ignorado por git.

## Uso

### Levantar el servicio

```bash
docker compose up -d
```

### Verificar estado

```bash
docker compose ps
```

O verificar el healthcheck:

```bash
docker compose ps --format json | grep -i health
```

### Ver logs

```bash
docker compose logs -f postgres
```

### Detener el servicio

```bash
docker compose down
```

**Nota:** Los datos persisten en `./data/postgres/`. Para eliminar también los datos:

```bash
docker compose down -v
```

## Conexión

### Desde psql (local)

```bash
psql -h 127.0.0.1 -p ${POSTGRES_PORT:-5432} -U ${POSTGRES_USER} -d ${POSTGRES_DB}
```

O usando variables del `.env`:

```bash
source .env
psql -h 127.0.0.1 -p $POSTGRES_PORT -U $POSTGRES_USER -d $POSTGRES_DB
```

### Desde otros clientes

- **Host:** `127.0.0.1` o `localhost`
- **Puerto:** Valor de `POSTGRES_PORT` en `.env` (por defecto: `5432`)
- **Usuario:** Valor de `POSTGRES_USER` en `.env`
- **Contraseña:** Valor de `POSTGRES_PASSWORD` en `.env`
- **Base de datos:** Valor de `POSTGRES_DB` en `.env`

### Ejemplo de cadena de conexión

```
postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@127.0.0.1:${POSTGRES_PORT:-5432}/${POSTGRES_DB}
```

## Verificación

1. **Verificar que el contenedor está corriendo:**

```bash
docker compose ps
```

2. **Verificar conectividad:**

```bash
docker compose exec postgres pg_isready -U ${POSTGRES_USER}
```

3. **Verificar base de datos:**

```bash
docker compose exec postgres psql -U ${POSTGRES_USER} -d ${POSTGRES_DB} -c "SELECT version();"
```

## Inicialización personalizada

Para ejecutar scripts SQL o shell al inicializar la base de datos (solo en la primera creación), coloca archivos en el directorio `./init/`:

- Scripts `.sql` o `.sql.gz` se ejecutan automáticamente
- Scripts `.sh` deben ser ejecutables

Ejemplo:

```bash
mkdir -p init
echo "CREATE TABLE IF NOT EXISTS test (id SERIAL PRIMARY KEY);" > init/01-init.sql
docker compose up -d
```

**Nota:** Los scripts en `init/` solo se ejecutan cuando la base de datos se crea por primera vez.

## Estructura de directorios

```
postgres-compose-prod/
├── docker-compose.yml    # Configuración del servicio
├── .env.example          # Template de variables de entorno
├── .env                  # Variables de entorno (no versionado)
├── .gitignore           # Archivos ignorados por git
├── README.md            # Esta documentación
├── data/                # Datos persistentes (no versionado)
│   └── postgres/        # Directorio de datos de PostgreSQL
└── init/                # Scripts de inicialización (opcional)
    └── *.sql, *.sh      # Scripts ejecutados al crear la BD
```

## Características de producción

- **Healthcheck:** Monitoreo básico de salud del servicio
- **Restart policy:** `unless-stopped` para alta disponibilidad
- **Persistencia:** Datos almacenados en volumen local
- **Seguridad:** Puerto expuesto solo en localhost
- **Versión fija:** Imagen con versión explícita para reproducibilidad

## Troubleshooting

### El servicio no inicia

1. Verifica que el puerto configurado (por defecto 5432) no esté en uso:
```bash
netstat -an | grep ${POSTGRES_PORT:-5432}
```

2. Revisa los logs:
```bash
docker compose logs postgres
```

### Error de permisos en data/

Si hay problemas de permisos con el directorio `data/`:

```bash
sudo chown -R 999:999 ./data/postgres
```

El usuario `999` es el UID por defecto de PostgreSQL en el contenedor.

### Reiniciar desde cero

```bash
docker compose down -v
rm -rf data/
docker compose up -d
```

**Advertencia:** Esto elimina todos los datos.
