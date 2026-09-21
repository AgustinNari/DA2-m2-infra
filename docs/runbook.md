# Runbook de Lightsail DEV

Este runbook opera la instancia `m2-dev` en `sa-east-1`. El servicio sigue expuesto conscientemente por HTTP: Frontend/Nginx escucha en `80`, Backend sólo en `127.0.0.1:8080` y PostgreSQL únicamente en la red Docker. Flyway aplica migraciones al iniciar Backend; la última migración versionada actual es V41.

Todos los comandos de Compose se ejecutan desde:

```bash
cd /home/ubuntu/DA2/Infra/DA2-m2-infra/compose
```

El runtime usa dos archivos independientes:

- `.env`: configuración y secretos runtime. Es privado del servidor.
- `images.env`: referencias GHCR por SHA. No contiene secretos, pero se ignora para que el servidor conserve su estado desplegado.

Nunca mostrar, copiar a logs ni versionar `.env`.

## Estado

```bash
docker compose --env-file .env --env-file images.env -f compose.deploy.yml ps
```

## Logs

```bash
docker compose --env-file .env --env-file images.env -f compose.deploy.yml logs --tail=100 backend
docker compose --env-file .env --env-file images.env -f compose.deploy.yml logs --tail=100 frontend
docker compose --env-file .env --env-file images.env -f compose.deploy.yml logs --tail=100 postgres
```

## Health

```bash
curl --fail --connect-timeout 3 --max-time 10 http://127.0.0.1:8080/actuator/health
curl --fail --connect-timeout 3 --max-time 10 http://127.0.0.1/
curl --fail --connect-timeout 3 --max-time 10 http://127.0.0.1/api/catalog/categories
```

## Deploy manual por SHA

Usar siempre el SHA completo de 40 caracteres publicado por el workflow. El script toma un lock compartido durante hasta 600 segundos, prepara un `images.env` candidato, hace `pull` y `up -d --no-deps` sólo del servicio elegido y ejecuta los smoke tests. Sólo reemplaza atómicamente `images.env` cuando el despliegue queda sano.

```bash
cd /home/ubuntu/DA2/Infra/DA2-m2-infra
bash scripts/deploy-service.sh backend <FULL_SHA_BACKEND>
bash scripts/deploy-service.sh frontend <FULL_SHA_FRONTEND>
```

Backend y Frontend usan el mismo lock `/home/ubuntu/DA2/.deploy/m2-deploy.lock`. La concurrencia de GitHub evita interrupciones dentro de cada repositorio; `flock` agrega la exclusión mutua que GitHub no puede ofrecer entre repositorios distintos.

## Rollback

Si un deploy no supera los smoke tests, el script vuelve automáticamente a la imagen anterior registrada, repite `pull`, `up` y smoke, y finaliza con error para dejar visible que el SHA solicitado no se aplicó. Un rollback manual usa exactamente la misma interfaz:

```bash
cd /home/ubuntu/DA2/Infra/DA2-m2-infra
bash scripts/deploy-service.sh backend <FULL_SHA_BACKEND_ANTERIOR>
bash scripts/deploy-service.sh frontend <FULL_SHA_FRONTEND_ANTERIOR>
```

El script nunca ejecuta `down`, nunca reinicia PostgreSQL y nunca borra volúmenes.

## Runtime config

Editar `.env` directamente en el servidor sin imprimir su contenido. Mantener `FRONTEND_PORT=80` y `SIMULATOR_ENABLED=false`. Después de cambiar variables de un servicio, recrear sólo ese servicio:

```bash
docker compose --env-file .env --env-file images.env -f compose.deploy.yml up -d --no-deps backend
docker compose --env-file .env --env-file images.env -f compose.deploy.yml up -d --no-deps frontend
```

Para cambios de PostgreSQL se requiere una ventana operativa separada; no recrearlo como parte de un deploy de aplicación.

## GHCR privado entre owners

Las imágenes permanecen privadas:

- `ghcr.io/juanmaguida/backend-damii`
- `ghcr.io/santimussi/front-desarrollo-apps-2`

Elegir una única cuenta GitHub de lectura para Lightsail. Desde la configuración de cada package, otorgar a esa misma cuenta permiso **Read**; si el package hereda permisos del repositorio, la cuenta también necesita acceso de lectura a ese repositorio o se debe desactivar la herencia y conceder acceso explícito al package. Para publicar, cada package debe dar acceso **Write** al repositorio Actions que lo genera.

Crear manualmente para esa cuenta un PAT classic con sólo `read:packages` (y acceso SSO si el owner lo exige). No usar scopes `write:packages` ni `delete:packages` en el servidor. Iniciar sesión sin exponer el token en argumentos ni historial:

```bash
read -rsp 'GHCR token: ' GHCR_TOKEN && echo
printf '%s' "$GHCR_TOKEN" | docker login ghcr.io -u <GHCR_READER_USERNAME> --password-stdin
unset GHCR_TOKEN
```

No guardar el PAT en `.env` ni `images.env`. Docker conserva la credencial del usuario `ubuntu` en su configuración local; proteger esa cuenta y su home.

## Preparación inicial de `images.env`

```bash
cd /home/ubuntu/DA2/Infra/DA2-m2-infra/compose
cp images.env.example images.env
chmod 600 images.env
```

Editar ambos valores para que apunten a tags existentes `sha-<40_HEX>`. Validar sin levantar servicios:

```bash
docker compose --env-file .env --env-file images.env -f compose.deploy.yml config --quiet
docker compose --env-file .env --env-file images.env -f compose.deploy.yml pull backend frontend
```

## Configuración de GitHub CD

El proyecto no utiliza GitHub Environments. En Backend y Frontend, abrir `Settings → Secrets and variables → Actions` y mantener `CD_ENABLED` como Repository Variable en `false` o sin crear durante la preparación.

Configurar estas **Repository Variables** en cada uno de esos dos repositorios:

- `CD_ENABLED`: `false` durante la preparación; `true` sólo al habilitar CD.
- `LIGHTSAIL_HOST`: Static IPv4 o hostname verificado.
- `LIGHTSAIL_USER`: `ubuntu`.
- `LIGHTSAIL_DEPLOY_PATH`: `/home/ubuntu/DA2/Infra/DA2-m2-infra`.
- `LIGHTSAIL_KNOWN_HOSTS`: línea obtenida y verificada fuera del runner para el host de Lightsail.

Agregar como **Repository Secret** `LIGHTSAIL_SSH_PRIVATE_KEY`, usando una clave dedicada sin reutilizar una clave personal. El workflow exige `StrictHostKeyChecking=yes`; nunca sustituirlo por `no`.

Para habilitar CD, establecer `CD_ENABLED=true` en cada repositorio sólo después de probar deploy y rollback manuales. Si falta, está vacía o tiene cualquier otro valor, la publicación GHCR continúa y el job deploy se omite sin fallar el workflow.

Infra actualmente sólo valida Compose y Terraform: no tiene CD, no utiliza `CD_ENABLED` ni `LIGHTSAIL_*` y no requiere Secrets de deployment. Si en el futuro se implementa Infra CD, deberá adoptar la misma convención de Repository Variables (`CD_ENABLED`, `LIGHTSAIL_HOST`, `LIGHTSAIL_USER`, `LIGHTSAIL_DEPLOY_PATH`, `LIGHTSAIL_KNOWN_HOSTS`) y Repository Secret (`LIGHTSAIL_SSH_PRIVATE_KEY`), sin depender de GitHub Environments.

## Clave SSH dedicada y host key

En una estación administrativa:

```bash
ssh-keygen -t ed25519 -a 100 -N '' -f ./m2-github-actions -C github-actions-m2-dev
ssh-keyscan -H <LIGHTSAIL_HOST> > ./m2-known-hosts
ssh-keygen -lf ./m2-known-hosts
```

Comparar el fingerprint con la clave del servidor obtenida por un canal administrativo confiable. Instalar sólo la pública:

```bash
sed 's/^/restrict /' ./m2-github-actions.pub | ssh ubuntu@<LIGHTSAIL_HOST> 'umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys'
```

Guardar el contenido de la clave privada en `LIGHTSAIL_SSH_PRIVATE_KEY` y la línea verificada de `m2-known-hosts` en `LIGHTSAIL_KNOWN_HOSTS`. No versionar ninguno de esos archivos.

## Troubleshooting

- `denied` al hacer pull: verificar `docker login`, permiso Read en ambos packages y que el SHA exista.
- `Host key verification failed`: volver a verificar el fingerprint y actualizar `LIGHTSAIL_KNOWN_HOSTS`; no deshabilitar la comprobación.
- timeout del lock: comprobar si existe otro deploy activo. `flock` libera el lock al terminar el proceso; no borrar el archivo por reflejo.
- smoke de Backend falla: revisar los 100 logs de Backend y el estado de PostgreSQL, sin reiniciar la base.
- smoke vía `/api` falla pero Actuator responde: revisar Frontend/Nginx y la red Compose.
- rollback automático falla: conservar `images.env`, revisar logs limitados y ejecutar manualmente un SHA conocido.

## Emergencia: volver al deploy manual actual

Conservar `.env`, `images.env` y los volúmenes. Elegir SHAs previamente sanos y ejecutar el script manualmente para ambos servicios. Si el script no estuviera disponible pero `compose.deploy.yml` sí:

```bash
cd /home/ubuntu/DA2/Infra/DA2-m2-infra/compose
# Editar sólo BACKEND_IMAGE o FRONTEND_IMAGE en images.env con un SHA conocido.
docker compose --env-file .env --env-file images.env -f compose.deploy.yml pull backend frontend
docker compose --env-file .env --env-file images.env -f compose.deploy.yml up -d --no-deps backend frontend
```

Después ejecutar los tres checks de Health. No usar `docker compose down`, `down -v`, `prune` ni recrear PostgreSQL durante esta recuperación.
