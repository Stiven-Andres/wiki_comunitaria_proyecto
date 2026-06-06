# Wiki Comunitaria — DokuWiki
**Proyecto Final — Automatización y Despliegue de Servicios en Cloud y Redes Comunitarias**  
**Asignatura:** Administración de Redes  
**Universidad Católica de Colombia**

---

## Descripción

Este proyecto implementa una **wiki comunitaria** basada en DokuWiki, desplegada de forma completamente automatizada usando Ansible y Docker sobre una instancia EC2 en AWS. El objetivo es demostrar cómo una organización comunitaria puede disponer de servicios digitales reproducibles, escalables y de bajo costo operativo, tanto en la nube como en entornos locales.

DokuWiki fue elegido por ser liviano, no requerir base de datos y funcionar perfectamente en entornos con recursos limitados, lo que lo hace ideal para redes comunitarias.

---

## Tecnologías utilizadas

| Tecnología | Uso en el proyecto |
|---|---|
| Ansible | Automatización completa del despliegue |
| Docker | Contenedor donde corre DokuWiki |
| DokuWiki | Motor de la wiki comunitaria |
| AWS EC2 | Servidor cloud (Ubuntu/Debian, free tier) |
| AWS Security Groups | Control de acceso a puertos de la instancia |
| Linux (Debian) | Sistema operativo base del servidor |

---

## Arquitectura

```
                        FASE 1 — CLOUD
┌─────────────────────────────────────────────────────┐
│                                                     │
│   Internet → EC2 AWS (3.87.126.183)                 │
│                  │                                  │
│              Puerto 80                              │
│                  │                                  │
│         Contenedor Docker                           │
│                  │                                  │
│            DokuWiki:latest                          │
│                                                     │
└─────────────────────────────────────────────────────┘

                       FASE 2 — LOCAL
┌─────────────────────────────────────────────────────┐
│                                                     │
│   Red local → Servidor Linux                        │
│                  │                                  │
│              Puerto 80                              │
│                  │                                  │
│         Contenedor Docker                           │
│                  │                                  │
│            DokuWiki:latest                          │
│                                                     │
└─────────────────────────────────────────────────────┘
```

El mismo playbook de Ansible despliega DokuWiki en ambos entornos. La única diferencia es el inventario que se le pasa como parámetro.

---

## Estructura del repositorio

```
wiki_comunitaria_proyecto/
├── ansible/
│   ├── inventories/
│   │   └── wiki.ini               # IP del servidor destino
│   ├── group_vars/
│   │   └── all.yml                # Variables globales del proyecto
│   ├── roles/
│   │   ├── docker/
│   │   │   └── tasks/
│   │   │       └── main.yml       # Instala Docker automáticamente
│   │   └── dokuwiki/
│   │       ├── tasks/
│   │       │   └── main.yml       # Despliega el contenedor DokuWiki
│   │       └── templates/
│   │           └── docker-compose.yml.j2  # Template del compose
│   ├── provision_aws.yml          # Crea EC2 y Security Group en AWS
│   └── deploy.yml                 # Playbook principal de despliegue
├── docker/
│   └── docker-compose.yml         # Para pruebas locales rápidas
├── .gitignore
└── README.md
```

---

## Requisitos previos

En la máquina desde donde se ejecuta Ansible:

```bash
# Instalar Ansible
sudo apt install ansible -y

# Instalar colecciones necesarias
ansible-galaxy collection install community.docker
ansible-galaxy collection install amazon.aws  # Solo si se usa AWS real

# Para AWS real: configurar credenciales
aws configure
```

---

## Instrucciones de despliegue

### Clonar el repositorio

```bash
git clone https://github.com/<usuario>/<repo>.git
cd wiki_comunitaria_proyecto
```

### Fase 1 — Cloud (AWS)

El playbook `provision_aws.yml` automatiza la creación de la infraestructura en AWS:
- Crea el Security Group con puertos 22, 80 y 443
- Lanza una instancia EC2 t2.micro con Ubuntu
- Actualiza el inventario automáticamente con la IP pública

```bash
# Crear infraestructura (requiere AWS CLI configurado)
ansible-playbook ansible/provision_aws.yml

# Desplegar DokuWiki
ansible-playbook ansible/deploy.yml -i ansible/inventories/wiki.ini
```

### Fase 2 — Entorno local

Se usa el mismo playbook, apuntando al servidor local:

```bash
# Editar inventario con la IP del servidor local
nano ansible/inventories/wiki.ini

# Desplegar (idéntico a la Fase 1)
ansible-playbook ansible/deploy.yml -i ansible/inventories/wiki.ini
```

El servicio queda accesible desde cualquier dispositivo en la misma red local.

---

## Flujo de automatización

Cuando se ejecuta `deploy.yml`, Ansible realiza automáticamente los siguientes pasos:

1. **Conexión SSH** al servidor destino
2. **Rol `docker`**: actualiza apt, instala Docker, lo inicia y habilita como servicio
3. **Rol `dokuwiki`**:
   - Crea los directorios `/opt/wiki-comunitaria` y `/opt/wiki-comunitaria/data`
   - Genera el `docker-compose.yml` desde el template Jinja2 con las variables del proyecto
   - Ejecuta `docker compose up -d` para levantar el contenedor
   - Verifica que el puerto 80 esté respondiendo
4. **Confirmación**: muestra el estado del contenedor y la URL de acceso

Todo el proceso es **idempotente**: si se ejecuta de nuevo, Ansible detecta qué ya está en el estado deseado y no repite cambios innecesarios.

---

## Evidencia de funcionamiento

**Resultado del playbook:**
```
PLAY RECAP
wiki-server: ok=16   changed=5   unreachable=0   failed=0   skipped=0
```

**Contenedor corriendo:**
```
CONTAINER ID   IMAGE                         STATUS          PORTS
4a24e3c7d004   linuxserver/dokuwiki:latest   Up 11 seconds   0.0.0.0:80->80/tcp
```

**Acceso público:**
- URL: http://3.87.126.183
- Primera configuración: http://3.87.126.183/install.php

---

## Componente de redes comunitarias

### ¿Qué utilidad tiene DokuWiki en una comunidad?

Una wiki comunitaria permite que los miembros de una comunidad documenten conocimiento colectivo: manuales, guías, recursos educativos, información local y acuerdos comunitarios, todo desde un navegador web sin necesidad de instalar nada en los dispositivos de los usuarios.

### ¿Cómo funcionaría con conectividad limitada?

DokuWiki no requiere conexión a internet para funcionar una vez desplegado. Al estar corriendo en un servidor local conectado a un router OpenWrt, todos los dispositivos de la red local (computadores, celulares, tablets) pueden acceder a la wiki sin depender de internet. Esto es clave en zonas rurales o comunidades con conectividad limitada.

### Integración con OpenWrt

El router OpenWrt actuaría como el punto de acceso de la red comunitaria. El servidor con DokuWiki se conecta al router y obtiene una IP local (por ejemplo `192.168.1.100`). Desde cualquier dispositivo conectado al mismo router, se accede a la wiki en `http://192.168.1.100`. Opcionalmente, OpenWrt puede configurarse para resolver un nombre de dominio local como `wiki.local` apuntando a esa IP.

### Ventajas de una red local autónoma

- **Sin dependencia de internet**: el servicio funciona aunque no haya conectividad externa
- **Bajo costo**: un servidor modesto (incluso una Raspberry Pi) es suficiente
- **Control comunitario**: la comunidad administra su propia infraestructura
- **Privacidad**: los datos no salen de la red local

### Retos para escalar

- Sincronización de contenido entre múltiples nodos en una red mesh
- Gestión de usuarios y permisos a medida que crece la comunidad
- Respaldos automáticos del volumen de datos de DokuWiki

---

## Por qué Docker

| Ventaja | Descripción |
|---|---|
| **Portabilidad** | El mismo contenedor corre igual en AWS, en un servidor local o en una Raspberry Pi |
| **Reproducibilidad** | Cualquier miembro del equipo puede levantar el servicio con un solo comando |
| **Aislamiento** | DokuWiki corre en su propio entorno sin afectar el sistema operativo base |
| **Facilidad de actualización** | `docker compose pull && docker compose up -d` actualiza el servicio |

---

## Seguridad — Importante

**NUNCA subir al repositorio:**
- Archivos `.pem`
- Access Keys de AWS
- Contraseñas o tokens

Estos archivos están excluidos mediante `.gitignore`.

---

## Integrantes del grupo

| Nombre |
|---|
| Stiven Andres Paloma |
| Jhon Alexander Parra Olarte |

---

## Conclusiones

- La automatización con Ansible elimina la posibilidad de errores manuales y hace el despliegue reproducible en cualquier entorno.
- Docker garantiza que DokuWiki se comporte igual independientemente del servidor donde corra.
- DokuWiki es una solución ideal para redes comunitarias por su ligereza, su funcionamiento sin base de datos y su capacidad de operar completamente offline.
- El uso de roles en Ansible permite separar responsabilidades: un rol instala Docker y otro despliega la aplicación, facilitando el mantenimiento y la reutilización.
- Infrastructure as Code permite que toda la infraestructura esté versionada en Git, lo que facilita la colaboración y el seguimiento de cambios.

