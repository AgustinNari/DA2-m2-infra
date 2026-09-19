# Infraestructura M2

Este repositorio contiene la configuración de infraestructura para levantar el mismo código de la rama `dev` tanto localmente como en AWS Lightsail DEV. Las diferencias entre entornos se resuelven con el `.env` no versionado. Los repositorios deben conservar esta estructura relativa:

```text
DA2/
├── Backend/
│   └── Backend-DAMII/
├── Frontend/
│   └── front-desarrollo-apps-2/
└── Infra/
    └── DA2-m2-infra/
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

Editá `.env` y reemplazá todos los placeholders de passwords por valores locales. El archivo `.env` real está ignorado por Git y no debe versionarse. El simulador queda apagado por defecto; un desarrollador puede habilitarlo explícitamente con `SIMULATOR_ENABLED=true`.

Variables por entorno (sin versionar valores reales):

```dotenv
# LOCAL
POSTGRES_DB=m2
POSTGRES_USER=m2_app
POSTGRES_PASSWORD=<PASSWORD_LOCAL>
FRONTEND_PORT=8081
MOCK_CITIZEN_PASSWORD=<PASSWORD_LOCAL_CITIZEN>
MOCK_AGENT_PASSWORD=<PASSWORD_LOCAL_AGENT>
MOCK_AREA_RESPONSIBLE_PASSWORD=<PASSWORD_LOCAL_AREA>
MOCK_ADMIN_PASSWORD=<PASSWORD_LOCAL_ADMIN>
SIMULATOR_ENABLED=false # usar true sólo cuando el desarrollador lo necesite
ATTACHMENT_STORAGE_ROOT=/var/lib/m2/attachments
```

En Lightsail se usan las mismas variables, con passwords DEV distintos y robustos,
`FRONTEND_PORT=80` y `SIMULATOR_ENABLED=false`.

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

No ejecutes `docker compose down -v` salvo que quieras eliminar deliberadamente los volúmenes y los datos de PostgreSQL y adjuntos.

## Puertos locales

- Frontend: `http://localhost:8081` por defecto; se puede cambiar con `FRONTEND_PORT`.
- Backend: `http://127.0.0.1:8080`, accesible sólo desde la misma máquina.
- PostgreSQL: `5432` únicamente dentro de la red Docker, sin publicación al host.
- Adjuntos: volumen persistente `attachment_data`, montado en `/var/lib/m2/attachments`.

En Lightsail se usa el mismo Compose con `FRONTEND_PORT=80`. El Backend conserva el acceso local por `127.0.0.1:8080`, pero públicamente sólo se alcanza a través de `http://STATIC_IP/api/...`.

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

## Alcance funcional pendiente

Este bloque garantiza build, arranque, migraciones y proxy, pero no completa los endpoints
funcionales que el Frontend espera y el Backend aún no implementa. Esas divergencias no
impiden desplegar el stack; algunas pantallas pueden recibir `401`, `404` o `405` hasta que
se resuelvan en tareas funcionales posteriores.
