# Infraestructura M2

Este repositorio contiene la configuración de infraestructura para levantar el módulo completo en entornos local y DEV. Los repositorios deben conservar esta estructura relativa:

```text
Backend/
Frontend/
Infra/
```

El Compose canónico de `Infra/compose` levanta Frontend (React + Nginx), Backend y PostgreSQL. El repositorio Backend también posee un Compose propio para desarrollar únicamente Backend + PostgreSQL sin clonar Infra. Son alternativas válidas y no conviene ejecutarlas simultáneamente con sus puertos predeterminados, ya que pueden colisionar, especialmente en el puerto `8080`.

## Requisitos

- Docker
- Docker Compose plugin

## Configuración local

Desde el repositorio Infra:

```bash
cd compose
cp .env.example .env
```

Editá `.env` y reemplazá la contraseña de ejemplo por una contraseña local. El archivo `.env` real está ignorado por Git y no debe versionarse.

## Comandos habituales

Ejecutar desde `Infra/compose`:

```bash
# Levantar
docker compose up -d

# Reconstruir y levantar
docker compose up -d --build

# Ver estado
docker compose ps

# Ver logs
docker compose logs -f

# Detener sin borrar datos
docker compose down
```

No ejecutes `docker compose down -v` salvo que quieras eliminar deliberadamente el volumen y los datos de PostgreSQL.

## Puertos locales

- Frontend: `http://localhost:8081` por defecto; se puede cambiar con `FRONTEND_PORT`.
- Backend: `http://localhost:8080`.
- PostgreSQL: `5432` únicamente dentro de la red Docker, sin publicación al host.

```text
Browser
   |
   v
Frontend / Nginx
   |-- /    -> React
   `-- /api
          |
          v
       Backend :8080
          |
          v
       PostgreSQL :5432
```

El procedimiento previsto para AWS está documentado en [`docs/deploy-aws-dev.md`](docs/deploy-aws-dev.md).
