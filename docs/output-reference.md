# Referencia de la salida del export RBAC

Qué genera `export-users-roles.yml` (rol `infra.aap_configuration_extended.filetree_create` **4.13.0**) y qué significa cada campo.

En [`examples/output/`](../examples/output/) hay un export de ejemplo completo con datos ficticios.

```
aap-export-output/
├── env_variables.yml                    # Plantilla de passwords a rellenar
├── gateway_organizations.yaml           # Organizaciones
├── gateway_users.yaml                   # Usuarios locales
├── gateway_teams.yaml                   # Equipos
├── gateway_role_definitions.yaml        # Roles personalizados (no los predefinidos)
├── gateway_role_user_assignments.yaml   # Quién (usuario) tiene qué rol sobre qué objeto
└── gateway_role_team_assignments.yaml   # Quién (equipo) tiene qué rol sobre qué objeto
```

## Modelo

```
Organización ──┬── Usuarios
               └── Equipos

Definición de rol  =  conjunto de permisos sobre un tipo de objeto
Asignación         =  (usuario | equipo) + definición de rol + objeto concreto
```

Todas las referencias entre ficheros son **por nombre**, no por ID, por lo que el export es portable entre instancias de AAP.

---

## gateway_organizations.yaml

Clave raíz: `gateway_organizations`

| Campo | Descripción |
| :--- | :--- |
| `name` | Nombre de la organización (clave única). |
| `description` | Solo aparece si no está vacía. Va envuelta en `{%- raw -%}` para que Ansible no interprete `{{ }}` dentro del texto al reimportar. |

```yaml
gateway_organizations:
  - name: "Finanzas"
    description: |-
      {%- raw -%}
      Automatizaciones del area de Finanzas
      {%- endraw -%}
```

> ⚠️ `infra.aap_configuration.dispatch` lee las organizaciones de **`aap_organizations`**, no de `gateway_organizations`. `import-users-roles.yml` ya hace esa conversión; si importas con otro playbook, tendrás que hacerla tú.

## gateway_users.yaml

Clave raíz: `aap_user_accounts`

| Campo | Descripción |
| :--- | :--- |
| `username` | Login del usuario (clave única). |
| `email`, `first_name`, `last_name` | Datos del perfil. Pueden ir vacíos. |
| `password` | La API nunca devuelve passwords, así que aquí se escribe la referencia `{{ vaulted_gateway_users_<username>_password }}`. En el nombre de la variable el username va en minúsculas y cualquier carácter que no sea `a-z0-9` se cambia por `_` (`ana.garcia` → `ana_garcia`). |
| `is_superuser` | `"True"` / `"False"`. Indica si es administrador de toda la plataforma. |
| `authenticators` | **IDs** de los autenticadores (Local, LDAP, SAML…) por los que entra el usuario. Son IDs, no nombres: comprueba que coinciden en el AAP de destino. |
| `authenticator_uid` | Identificador del usuario dentro de ese autenticador. |

```yaml
aap_user_accounts:
  - username: "ana.garcia"
    email: "ana.garcia@example.com"
    first_name: "Ana"
    last_name: "Garcia"
    password: "{{ vaulted_gateway_users_ana_garcia_password }}"
    is_superuser: "False"
    authenticators: [1]
    authenticator_uid: "ana.garcia"
```

## gateway_teams.yaml

Clave raíz: `aap_teams`

| Campo | Descripción |
| :--- | :--- |
| `name` | Nombre del equipo. Es único **dentro de su organización**. |
| `description` | Siempre presente, aunque esté vacía. |
| `organization` | Organización a la que pertenece. |

La pertenencia de usuarios a un equipo **no** se guarda aquí: es una asignación del rol `Team Member` en `gateway_role_user_assignments.yaml`.

## gateway_role_definitions.yaml

Clave raíz: `gateway_role_definitions`

Solo contiene los roles **personalizados**. Los predefinidos (`managed: true`, como *Organization Admin*, *Team Member* o *JobTemplate Execute*) y *Platform Auditor* no se exportan porque ya existen en cualquier AAP. Si sale vacío (`[]`), es que no hay roles personalizados.

| Campo | Descripción |
| :--- | :--- |
| `name` | Nombre del rol. |
| `description` | Descripción. Se omite si está vacía. |
| `permissions` | Lista de permisos `<servicio>.<accion>_<modelo>`, p. ej. `awx.execute_jobtemplate` o `shared.view_organization`. |
| `content_type` | Tipo de objeto al que se aplica el rol: `shared.organization`, `shared.team`, `awx.jobtemplate`, `awx.inventory`, `awx.project`, `awx.credential`, `eda.project`… |

## gateway_role_user_assignments.yaml

Clave raíz: `gateway_role_user_assignments`

| Campo | Descripción |
| :--- | :--- |
| `role_definition` | Nombre del rol asignado (predefinido o personalizado). |
| `object_ids` | Lista con el **nombre** del objeto sobre el que aplica el rol. No aparece en roles globales de plataforma, que no van sobre ningún objeto. |
| `user` | Username. |

```yaml
gateway_role_user_assignments:
  - role_definition: "Organization Admin"
    object_ids:
      - Finanzas
    user: "ana.garcia"
```

Suele ser el fichero más grande, porque contiene la mayor parte del RBAC, incluida la pertenencia a equipos (`Team Member`). Las asignaciones de roles `galaxy.*` (Automation Hub) no se exportan.

## gateway_role_team_assignments.yaml

Clave raíz: `gateway_role_team_assignments`

| Campo | Descripción |
| :--- | :--- |
| `role_definition` | Nombre del rol asignado. |
| `team` | Nombre del equipo. |
| `assignment_objects` | Lista de objetos con `name`, `type` y, para objetos que dependen de una organización (equipos, recursos `awx.*` y `eda.*`), también `organization`, para no confundir objetos con el mismo nombre en organizaciones distintas. |

Valores de `type`: `organizations`, `teams`, `awx.project`, `awx.inventory`, `awx.credential`, `awx.jobtemplate`, `awx.workflowjobtemplate`, `awx.executionenvironment`, `awx.instancegroup`, `awx.notificationtemplate`, `eda.project`, `eda.activation`, `eda.edacredential`, `eda.eventstream`, `eda.decisionenvironment`.

```yaml
gateway_role_team_assignments:
  - role_definition: "JobTemplate Launcher Readonly"
    team: "fin-operadores"
    assignment_objects:
      - name: Pagos - Cierre mensual
        type: awx.jobtemplate
        organization: Finanzas
```

> Las asignaciones a usuarios usan `object_ids` (lista de nombres) y las de equipos `assignment_objects` (lista de objetos). Es el formato que esperan los roles de importación de `infra.aap_configuration` 4.11.0.

## env_variables.yml

Plantilla con todas las variables `vaulted_*` que aparecen en el export, todas con el valor `CHANGE_ME`. Solo hace falta para **importar**:

```bash
vi env_variables.yml                # rellenar passwords del entorno destino
ansible-vault encrypt env_variables.yml
```

Usuarios de LDAP/SAML: la password local no se usa, pero la variable tiene que existir. Pon cualquier valor o excluye al usuario con `import_exclude_users`.

> 🔒 Una vez rellenado, no lo subas a git sin cifrar.

---

## Orden de importación

Las asignaciones referencian por nombre a todo lo demás, así que el orden importa:

1. `gateway_role_definitions`
2. `gateway_organizations`
3. `gateway_teams`
4. `gateway_users`
5. `gateway_role_team_assignments`
6. `gateway_role_user_assignments`

`import-users-roles.yml` (vía `infra.aap_configuration.dispatch`) aplica este orden automáticamente.

> En `--check` los pasos 5 y 6 pueden fallar si las organizaciones, equipos o usuarios aún no existen en el destino. En simulación no se crean, así que es normal.
