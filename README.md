# AAP Configuration as Code (CaC) Export Execution Environment

Execution Environment (EE) minimalista y optimizado para **Red Hat Ansible Automation Platform (AAP) 2.6**, diseñado para exportar y respaldar de forma automatizada cualquier objeto del clúster (Usuarios, Equipos, RBAC, Job Templates, Proyectos, Inventarios, EDA y Hub) en formato declarativo YAML usando la suite **`infra.aap_configuration_extended`** (`filetree_create`).

Imagen publicada y disponible en Quay.io:
quay.io/automationiberia/aap-export-ee:2.6.0

---

## 📋 Especificaciones Técnicas

| Componente | Versión / Detalle |
| :--- | :--- |
| **Plataforma Objetivo** | Red Hat Ansible Automation Platform (AAP) 2.6 |
| **Imagen Base** | `registry.redhat.io/ansible-automation-platform-26/ee-minimal-rhel9:latest` |
| **Ansible Core** | `ansible-core 2.16.x` |
| **Python Runtime** | `Python 3.12` |
| **Herramienta de Build** | `ansible-builder` v3 (`version: 3`) |
| **Motor de Contenedores** | Podman |

---

## 🛠️ Correcciones Técnicas Incorporadas (Hotfixes Integrados en el Build)

Este EE incluye en su definición dos parches automatizados necesarios para la interoperabilidad entre colecciones en AAP 2.6:

1. **Fix de Leading Slash en `infra.aap_configuration_extended`**:
   Se corrige una barra inicial en `roles/filetree_create/tasks/gateway_role_definitions.yml` que provocaba que el lookup `ansible.platform.gateway_api` generara una doble barra (`//api/gateway/v1/role_definitions/`), siendo rechazada por el proxy Envoy del Gateway.

2. **Fix de Manejo de Excepciones en `ansible.platform`**:
   Se añade `self.retryable = False` a la clase base `PlatformError` en `plugins/plugin_utils/platform/exceptions.py`. Esto evita que un fallo de autenticación HTTP 401 o conexión quede enmascarado por un error interno de Python (`AttributeError: 'PlatformError' object has no attribute 'retryable'`).

---

## 🚀 Requisitos Previos

Instalación de utilidades en el host de ejecución:

    dnf install -y podman git sqlite-libs
    dnf install -y ansible-builder ansible-navigator

Autenticación en el registro certificado de Red Hat (para la compilación):

    podman login registry.redhat.io

---

## 🔐 Configuración de Credenciales de Build

1. Copiar la plantilla:

    cp ansible.cfg.example ansible.cfg

2. Introducir el Token de Red Hat Automation Hub en `ansible.cfg`:

    [galaxy]
    server_list = automation_hub, release_galaxy

    [galaxy_server.automation_hub]
    url=https://console.redhat.com/api/automation-hub/content/published/
    auth_url=https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/token
    token=TU_OFFLINE_TOKEN_AQUI

    [galaxy_server.release_galaxy]
    url=https://galaxy.ansible.com/

---

## 🔨 Construcción del Execution Environment

Para reconstruir la imagen localmente:

    ansible-builder build -t localhost/aap-export-ee:2.6.0 --container-runtime podman -v 3

---

## 📦 Consumo Directo desde Quay.io

Si no deseas compilar localmente, puedes descargar directamente la imagen precompilada:

    podman pull quay.io/automationiberia/aap-export-ee:2.6.0

---

## 📖 Uso: Exportar RBAC (Usuarios, Equipos y Permisos)

1. Crear la carpeta de salida en el host:

    mkdir -p /root/aap-export-output

2. Copiar y ajustar el playbook de exportación:

    cp export-users-roles.yml.example export-users-roles.yml

3. Lanzar la extracción con `ansible-navigator`:

    ansible-navigator run export-users-roles.yml \
      -m stdout \
      --eei quay.io/automationiberia/aap-export-ee:2.6.0 \
      --pp missing \
      --eev /root/aap-export-output:/output_data:Z \
      --container-options="--network=host" \
      -e "controller_hostname=https://<TU_AAP_GATEWAY_FQDN>" \
      -e "controller_oauthtoken=<TU_OAUTH_TOKEN>" \
      -e "controller_validate_certs=false"

---

## 🗂️ Catálogo de Objetos que puede exportar este EE

Modificando la lista `input_tag` en el playbook, este EE puede extraer:

- **Identidad & RBAC (Gateway)**: `gateway_users`, `gateway_teams`, `gateway_organizations`, `gateway_role_definitions`, `gateway_role_user_assignments`, `gateway_role_team_assignments`
- **Automatización (Controller)**: `controller_job_templates`, `controller_workflow_job_templates`, `controller_projects`, `controller_inventories`, `controller_hosts`, `controller_groups`, `controller_credentials`, `controller_credential_types`, `controller_schedules`
- **Event-Driven Ansible (EDA)**: `eda_projects`, `eda_rulebook_activations`, `eda_decision_environments`, `eda_credentials`
- **Private Automation Hub (PAH)**: `hub_namespaces`, `hub_collections`, `hub_ee_repositories`, `hub_ee_registries`
- **Configuración Global**: `controller_settings`, `gateway_settings`

---

## 📄 Estructura del Repositorio

    ├── .gitignore                       # Exclusiones de Git
    ├── README.md                        # Documentación técnica
    ├── ansible.cfg.example              # Plantilla para Automation Hub
    ├── bindep.txt                       # Paquetes RPM de sistema y compilación
    ├── execution-environment.yml         # Definición del EE con hotfixes integrados
    ├── export-users-roles.yml.example   # Playbook de ejemplo para exportar RBAC
    ├── requirements.txt                 # Dependencias Python
    └── requirements.yml                 # Colecciones Ansible (CaC suite)
