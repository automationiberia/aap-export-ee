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
| **Colecciones** | `infra.aap_configuration_extended` 4.13.0, `infra.aap_configuration` 4.11.0, `ansible.platform` 2.7.20260930, `ansible.controller` 4.8.12, `ansible.eda` 2.13.0, `ansible.hub` 1.1.2 |
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

    subscription-manager repos --enable ansible-automation-platform-2.6-for-rhel-9-x86_64-rpms
    dnf install -y podman git ansible-navigator ansible-builder   # ansible-builder solo para compilar

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

Lanza los comandos **desde la raíz del repositorio clonado**. El fichero `ansible-navigator.yml` del repo ya configura la imagen, el modo stdout, `--network=host`, la política de descarga y el paso de las variables `CONTROLLER_*` al contenedor. El token se pasa por variable de entorno para que no quede en el historial de bash.

    mkdir -p ~/aap-export-output
    export CONTROLLER_HOST=https://<TU_AAP_GATEWAY_FQDN>
    export CONTROLLER_VERIFY_SSL=false
    read -rsp "Token: " CONTROLLER_OAUTH_TOKEN; echo; export CONTROLLER_OAUTH_TOKEN

    ansible-navigator run export-users-roles.yml \
      --eev $HOME/aap-export-output:/output_data:Z

> La carpeta de salida tiene que ser accesible para el usuario que lanza el comando. Un usuario no root no puede usar `/root/...` (`PermissionError: [Errno 13]`).

> ⚠️ En los comandos de varias líneas, cada línea salvo la última debe terminar en `\` **sin espacios detrás**. Si no, bash ejecuta cada línea por separado y verás errores como `-m: command not found`.

---

## 📤 Qué se genera

    aap-export-output/
    ├── env_variables.yml                    # Plantilla de passwords (CHANGE_ME) para importar
    ├── gateway_organizations.yaml           # Organizaciones
    ├── gateway_users.yaml                   # Usuarios locales (password -> {{ vaulted_... }})
    ├── gateway_teams.yaml                   # Equipos y su organización
    ├── gateway_role_definitions.yaml        # Solo roles personalizados
    ├── gateway_role_user_assignments.yaml   # usuario + rol + objeto
    └── gateway_role_team_assignments.yaml   # equipo + rol + objeto

- Ejemplo completo con datos ficticios: [`examples/output/`](examples/output/)
- Explicación campo a campo: [`docs/output-reference.md`](docs/output-reference.md)

La API nunca devuelve passwords, así que en `gateway_users.yaml` se sustituyen por variables (`{{ vaulted_gateway_users_<usuario>_password }}`) y en `env_variables.yml` se genera la lista de esas variables para rellenarla al importar.

---

## 📥 Importar en otro AAP

`import-users-roles.yml` carga los ficheros del export y los aplica con `infra.aap_configuration.dispatch` en el orden correcto (role definitions → organizaciones → equipos → usuarios → asignaciones). Además:

- Convierte `gateway_organizations` a `aap_organizations`, que es lo que lee `dispatch`. Sin ese cambio, las organizaciones no se importarían y no habría ningún aviso.
- Excluye usuarios de `import_exclude_users` (por defecto `admin`, para no cambiar la password del admin del destino).
- Falla antes de tocar nada si falta alguna password o si sigue a `CHANGE_ME`.

Pasos:

    # 1. Rellenar las passwords del entorno destino
    vi ~/aap-export-output/env_variables.yml

    # 2. Apuntar al AAP de DESTINO
    export CONTROLLER_HOST=https://<AAP_DESTINO>
    export CONTROLLER_VERIFY_SSL=false
    read -rsp "Token: " CONTROLLER_OAUTH_TOKEN; echo; export CONTROLLER_OAUTH_TOKEN

    # 3. Probar primero en simulación (--check)
    ansible-navigator run import-users-roles.yml \
      --eev $HOME/aap-export-output:/output_data:Z \
      -e @/output_data/env_variables.yml \
      --check

    # 4. Aplicar (el mismo comando sin --check)
    ansible-navigator run import-users-roles.yml \
      --eev $HOME/aap-export-output:/output_data:Z \
      -e @/output_data/env_variables.yml

> En `--check` los pasos de asignaciones pueden fallar si las organizaciones, equipos o usuarios aún no existen en el destino, porque en simulación no se crean.

> 🔒 `env_variables.yml` contiene passwords en claro una vez relleno. Bórralo después de importar o cífralo con `ansible-vault encrypt` y añade `--ask-vault-pass` a los comandos.

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
    ├── ansible-navigator.yml            # Configuración por defecto de ansible-navigator
    ├── export-users-roles.yml           # Playbook para exportar RBAC
    ├── import-users-roles.yml           # Playbook para importar RBAC en otro AAP
    ├── docs/output-reference.md         # Qué contiene cada fichero exportado
    ├── examples/output/                 # Export de ejemplo (datos ficticios)
    ├── requirements.txt                 # Dependencias Python
    └── requirements.yml                 # Colecciones Ansible (versiones fijadas)
