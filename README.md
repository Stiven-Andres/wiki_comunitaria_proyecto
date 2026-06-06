# Wiki Comunitaria — DokuWiki

**Proyecto Final — Administración de Redes**
Universidad Católica de Colombia

-----

## Descripción

Despliegue automatizado de una **wiki comunitaria** basada en DokuWiki usando:

- **Ansible** para automatización completa
- **Docker** para contenedores reproducibles
- **AWS EC2** para el entorno cloud
- **OpenWrt / Linux local** para el entorno comunitario

-----

## Estructura del repositorio

```
wiki-comunitaria/
├── ansible/
│   ├── inventories/
│   │   ├── aws.ini              # IP de la instancia EC2
│   │   └── local.ini            # IP del servidor local
│   ├── group_vars/
│   │   └── all.yml              # Variables globales
│   ├── roles/
│   │   ├── docker/              # Rol: instala Docker
│   │   │   └── tasks/main.yml
│   │   └── dokuwiki/            # Rol: despliega DokuWiki
│   │       ├── tasks/main.yml
│   │       └── templates/docker-compose.yml.j2
│   ├── provision_aws.yml        # Crea EC2 + Security Group
│   └── deploy.yml               # Despliega DokuWiki
├── docker/
│   ├── docker-compose.yml       # Para pruebas locales rápidas
│   └── data/                    # Datos persistentes (gitignored)
├── diagrams/                    # Diagramas de arquitectura
└── README.md
```

-----

## Requisitos previos

### En tu máquina de control (donde corres Ansible)

```bash
# Instalar Ansible
pip install ansible

# Instalar colecciones necesarias
ansible-galaxy collection install amazon.aws
ansible-galaxy collection install community.docker

# Instalar dependencias Python para AWS
pip install boto3 botocore
```

### AWS

- Cuenta AWS activa (free tier funciona)
- AWS CLI configurado: `aws configure`
- Key Pair creado en AWS → guardado como `~/.ssh/wiki-key.pem`
- Permisos: `chmod 400 ~/.ssh/wiki-key.pem`

-----

## Fase 1 — Despliegue en AWS

### Paso 1: Configurar variables

Edita `ansible/group_vars/all.yml` y ajusta:

- `aws_region`: tu región de AWS
- `aws_key_name`: nombre de tu Key Pair en AWS (sin .pem)
- `aws_ami_id`: AMI de Ubuntu 22.04 para tu región

### Paso 2: Crear infraestructura

```bash
cd ansible/
ansible-playbook provision_aws.yml
```

Este playbook crea automáticamente:

- Security Group con puertos 22, 80, 443
- Instancia EC2 t2.micro con Ubuntu 22.04
- Actualiza el inventario `aws.ini` con la IP pública

### Paso 3: Desplegar DokuWiki

```bash
ansible-playbook deploy.yml -i inventories/aws.ini
```

Este playbook:

- Instala Docker y Docker Compose
- Despliega el contenedor de DokuWiki
- Verifica que el servicio esté disponible

### Paso 4: Acceder a la wiki

```
http://<IP_PUBLICA_EC2>
Primera configuración: http://<IP_PUBLICA_EC2>/install.php
```

-----

## Fase 2 — Despliegue Local

### Requisitos

- Servidor Linux (Ubuntu 22.04 recomendado)
- Acceso SSH configurado
- Red local: router OpenWrt o Cisco en el mismo segmento

### Paso 1: Configurar inventario local

Edita `ansible/inventories/local.ini`:

```ini
wiki-local ansible_host=<IP_SERVIDOR_LOCAL> ansible_user=ubuntu ...
```

### Paso 2: Desplegar

```bash
cd ansible/
ansible-playbook deploy.yml -i inventories/local.ini
```

### Paso 3: Acceder desde la red local

```
http://<IP_SERVIDOR_LOCAL>
```

Cualquier dispositivo en la misma red del router podrá acceder.

-----

## Prueba rápida con Docker (sin Ansible)

Para probar DokuWiki localmente antes de desplegar:

```bash
cd docker/
docker compose up -d
# Acceder en: http://localhost
```

-----

## Tecnologías utilizadas

|Tecnología         |Uso                                   |
|-------------------|--------------------------------------|
|Ansible            |Automatización completa del despliegue|
|Docker             |Contenedor de DokuWiki                |
|AWS EC2            |Servidor cloud (t2.micro free tier)   |
|AWS Security Groups|Firewall de la instancia EC2          |
|DokuWiki           |Motor de la wiki (sin base de datos)  |
|OpenWrt            |Router para red comunitaria local     |

-----

## Seguridad — Importante

**NUNCA subir al repositorio:**

- Archivos `.pem`
- Access Keys de AWS
- Contraseñas
- Tokens

Estos están excluidos en `.gitignore`.

-----

## Por qué DokuWiki para una red comunitaria

- **Sin base de datos**: funciona solo con archivos, ideal para servidores con recursos limitados
- **Liviana**: puede correr en hardware modesto (Raspberry Pi, etc.)
- **Offline-first**: funciona sin conectividad a internet una vez desplegada
- **Editable por la comunidad**: cualquier persona en la red local puede contribuir
- **Portable**: el volumen Docker contiene toda la información, fácil de respaldar

-----

## Comandos útiles

```bash
# Ver logs de DokuWiki
docker logs dokuwiki -f

# Reiniciar el servicio
docker restart dokuwiki

# Ver estado
docker ps

# Detener
docker compose down

# Actualizar imagen
docker compose pull && docker compose up -d
```
