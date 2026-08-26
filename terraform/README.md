# Terraform AWS DEV

Esta carpeta contiene la base mínima para administrar posteriormente el entorno DEV en Amazon Lightsail: instancia, Static IP, asociación y puertos públicos. No incluye PROD ni otros servicios AWS.

## Estructura

- `versions.tf`: versiones requeridas de Terraform y del provider AWS.
- `providers.tf`: configuración regional del provider.
- `variables.tf`: parámetros del entorno DEV.
- `main.tf`: recursos Lightsail.
- `outputs.tf`: nombre de instancia, Static IP y región.
- `terraform.tfvars.example`: valores públicos de referencia.

## Configuración

Copiar el ejemplo a un archivo local ignorado por Git:

```bash
cp terraform.tfvars.example dev.tfvars
```

Antes de ejecutar un plan, reemplazar el CIDR de documentación para SSH y confirmar que el blueprint, bundle y Availability Zone continúan disponibles en la región elegida. No guardar credenciales AWS en archivos `.tf` o `.tfvars`.

## Ciclo previsto

```bash
terraform init
terraform fmt
terraform validate
terraform plan -var-file=dev.tfvars
terraform apply -var-file=dev.tfvars
```

Actualmente **no se debe ejecutar `terraform apply`**. Primero deben resolverse la restricción de tamaño de Lightsail con AWS Support y validarse el deploy manual DEV documentado en `../docs/deploy-aws-dev.md`.

El state es local. `.terraform/`, los archivos `terraform.tfstate*` y los `.tfvars` locales están ignorados y no deben versionarse. Cuando la infraestructura deje de ser experimental deberá evaluarse un backend remoto antes del uso colaborativo.
