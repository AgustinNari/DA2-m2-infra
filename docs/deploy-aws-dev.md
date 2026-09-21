# Deploy manual en AWS DEV

Este documento conserva el alta base de la instancia Amazon Lightsail ya habilitada. El despliegue vigente consume imágenes privadas GHCR mediante `compose/compose.deploy.yml`; el procedimiento operativo completo está en [`runbook.md`](runbook.md).

## 1. Crear la instancia

La instancia DEV objetivo tiene:

- región `sa-east-1`;
- Ubuntu 24.04 LTS;
- plan General Purpose;
- red Dual-stack;
- 2 GB RAM;
- 2 vCPU;
- 60 GB SSD.

Crear una Static IP y asociarla a la instancia. En los firewalls IPv4 e IPv6 de Lightsail permitir solamente:

- TCP `22` para SSH, preferentemente limitado a la IP administrativa;
- TCP `80` para HTTP;
- no abrir `5432`;
- no abrir `8080`.

## 2. Preparar Ubuntu y Docker

Conectarse mediante SSH y actualizar el sistema:

```bash
sudo apt update
sudo apt upgrade -y
```

Instalar Docker Engine y el plugin de Docker Compose siguiendo el repositorio oficial de Docker para Ubuntu 24.04. Los paquetes requeridos son:

```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

La configuración previa del repositorio `apt` está documentada en [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/).

Verificar la instalación:

```bash
sudo systemctl status docker
sudo docker version
sudo docker compose version
```

Los siguientes comandos asumen que el usuario tiene permisos para ejecutar Docker. De lo contrario, anteponer `sudo`.

## 3. Copiar el proyecto

Desde el equipo que contiene los tres repos en `dev`, crear los archivos preservando los nombres de los directorios raíz:

```bash
git -C Backend/Backend-DAMII archive --format=tar --prefix=Backend-DAMII/ --output=backend.tar dev
git -C Frontend/front-desarrollo-apps-2 archive --format=tar --prefix=front-desarrollo-apps-2/ --output=frontend.tar dev
git -C Infra/DA2-m2-infra archive --format=tar --prefix=DA2-m2-infra/ --output=infra.tar dev
```

Copiar los tres archivos a la VM y extraerlos:

```bash
mkdir -p ~/DA2/Backend ~/DA2/Frontend ~/DA2/Infra
tar -xf backend.tar  -C ~/DA2/Backend
tar -xf frontend.tar -C ~/DA2/Frontend
tar -xf infra.tar    -C ~/DA2/Infra
```

El resultado debe ser:

```text
~/DA2/
├── Backend/
│   └── Backend-DAMII/
├── Frontend/
│   └── front-desarrollo-apps-2/
└── Infra/
    └── DA2-m2-infra/
```

Ingresar al directorio del Compose:

```bash
cd ~/DA2/Infra/DA2-m2-infra/compose
```

## 4. Configurar DEV

Crear el archivo local a partir del ejemplo:

```bash
cp .env.example .env
```

Configurar en `.env`:

```dotenv
POSTGRES_DB=m2
POSTGRES_USER=m2_app
POSTGRES_PASSWORD=<CONTRASEÑA_DEV_EXCLUSIVA>
FRONTEND_PORT=80
MOCK_CITIZEN_PASSWORD=<CONTRASEÑA_MOCK_CITIZEN_EXCLUSIVA>
MOCK_AGENT_PASSWORD=<CONTRASEÑA_MOCK_AGENT_EXCLUSIVA>
MOCK_AREA_RESPONSIBLE_PASSWORD=<CONTRASEÑA_MOCK_AREA_EXCLUSIVA>
MOCK_ADMIN_PASSWORD=<CONTRASEÑA_MOCK_ADMIN_EXCLUSIVA>
SIMULATOR_ENABLED=false
ATTACHMENT_STORAGE_ROOT=/var/lib/m2/attachments
```

Reemplazar todos los placeholders por contraseñas exclusivas y robustas. No reutilizar los valores del `.env.example`. No versionar `.env`, contraseñas ni otras credenciales. El simulador debe permanecer deshabilitado en Lightsail.

## 5. Validar y desplegar por imagen

```bash
cp images.env.example images.env
# Reemplazar ambos tags de ejemplo por SHAs que ya existan en GHCR.
docker compose --env-file .env --env-file images.env -f compose.deploy.yml config --quiet
bash ../scripts/deploy-service.sh backend <FULL_SHA_BACKEND>
bash ../scripts/deploy-service.sh frontend <FULL_SHA_FRONTEND>
```

Si un servicio presenta problemas, consultar sus logs:

```bash
docker compose logs -f frontend
docker compose logs -f backend
docker compose logs -f postgres
```

## 6. Comprobaciones

Desde la instancia, comprobar el Backend sin exponer `8080` públicamente:

```bash
curl --fail http://127.0.0.1:8080/actuator/health
```

Desde un navegador o equipo externo, reemplazando `STATIC_IP` por la Static IP asignada:

```text
http://STATIC_IP
http://STATIC_IP/api/catalog/categories
```

La primera URL debe servir React mediante Nginx. La segunda debe atravesar el proxy `/api` y devolver el catálogo público desde Backend.

## Consideraciones

- PostgreSQL debe permanecer únicamente en la red Docker y nunca publicar `5432`.
- Backend se enlaza sólo a `127.0.0.1:8080`; el acceso público ocurre exclusivamente mediante Nginx bajo `/api`.
- PostgreSQL y los adjuntos usan volúmenes persistentes. `docker compose down` los conserva.
- Los tres servicios usan `restart: unless-stopped` para recuperarse después de reiniciar Docker o la VM.
- HTTPS y el dominio se incorporarán después.
- GHCR y CD están preparados, pero CD permanece deshabilitado hasta configurar manualmente GitHub y establecer `CD_ENABLED=true`.
- La base Terraform ya existe, pero la infraestructura creada manualmente no debe administrarse con `apply` hasta definir e importar correctamente su state.
- No ejecutar `docker compose down -v` salvo que se quieran eliminar deliberadamente la base de datos y los adjuntos persistentes.
