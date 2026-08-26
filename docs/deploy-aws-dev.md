# Deploy manual en AWS DEV

Este procedimiento se ejecutará cuando AWS Support habilite un tamaño adecuado de Amazon Lightsail. No se debe crear infraestructura hasta que el tamaño quede confirmado.

## 1. Crear la instancia

En Amazon Lightsail, crear una instancia DEV con:

- región `sa-east-1`;
- Ubuntu 24.04 LTS;
- plan General Purpose;
- red Dual-stack;
- tamaño o bundle pendiente de la resolución del caso con AWS Support.

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

Obtener las revisiones versionadas de Backend, Frontend e Infra y conservar una estructura compatible con las rutas relativas del Compose:

```text
/opt/m2/
├── Backend/
│   └── Backend-DAMII/
├── Frontend/
│   └── front-desarrollo-apps-2/
└── Infra/
    └── DA2-m2-infra/
```

Ingresar al directorio del Compose:

```bash
cd /opt/m2/Infra/DA2-m2-infra/compose
```

## 4. Configurar DEV

Crear el archivo local a partir del ejemplo:

```bash
cp .env.example .env
```

Configurar en `.env`:

```dotenv
FRONTEND_PORT=80
```

Reemplazar `POSTGRES_PASSWORD` por una contraseña exclusiva y robusta para DEV. No versionar `.env`, contraseñas ni otras credenciales.

## 5. Validar y desplegar

```bash
docker compose config
docker compose build
docker compose up -d
docker compose ps
docker compose logs --tail=100
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
curl http://localhost:8080/actuator/health
```

Desde un navegador o equipo externo, reemplazando `STATIC_IP` por la Static IP asignada:

```text
http://STATIC_IP
http://STATIC_IP/api/categories
```

La primera URL debe servir React mediante Nginx. La segunda debe atravesar el proxy `/api` hacia Backend; una respuesta `401` también confirma que la solicitud alcanzó Spring cuando el endpoint está protegido.

## Consideraciones

- PostgreSQL debe permanecer únicamente en la red Docker y nunca publicar `5432`.
- Backend escucha en `8080` para el host, pero ese puerto no debe habilitarse en el firewall público de Lightsail.
- HTTPS y el dominio se incorporarán después.
- GHCR y CD se incorporarán después de validar este despliegue manual.
- Terraform se preparará después de comprender y validar manualmente la infraestructura.
- No ejecutar `docker compose down -v` salvo que se quiera eliminar deliberadamente la base de datos persistente.
