# Arquitectura, Documentación y Diagramas de Scripts

Este documento presenta el análisis técnico completo del repositorio **`ansible-sandbox`**, incluyendo diagramas de arquitectura, flujos de ejecución por playbook y rol, secuencia de interacción con los nodos gestionados, inventario detallado de archivos y estado de las mejoras aplicadas.

---

## 1. Arquitectura General del Repositorio

El repositorio está compuesto por **2 playbooks** en la raíz y **2 roles de Ansible** (`common` y `collectd`) orientados a aprovisionar nodos Linux definidos en el grupo de inventario `managed_nodes` con escalación de privilegios (`become: yes`).

```mermaid
flowchart TD
    INV["Inventario de Ansible\nGrupo: managed_nodes"] --> PB1["common_playbook.yml\nbecome: yes\nvars: install_package=true"]
    INV --> PB2["install_collectd_playbook.yml\nbecome: yes"]

    PB1 --> R1["Rol: common"]
    PB2 --> R2["Rol: collectd"]

    subgraph COMMON["Rol common"]
        C_DEF["defaults/main.yml\ninstall_package: false\nreboot_on_apparmor_disable: false"] -.-> C_MAIN
        C_MAIN["tasks/main.yml"] --> C_PKG["tasks/install_packages.yml\nInstala 8 paquetes base"]
        C_MAIN --> C_AA["tasks/apparmor.yml\nDetiene y deshabilita AppArmor"]
        C_AA -. "notify: Reboot" .-> C_HND["handlers/main.yml\nReboot (when: reboot_on_apparmor_disable)"]
    end

    subgraph COLLECTD["Rol collectd"]
        COL_DEF["defaults/main.yml\ncollectd_state: present\ncollectd_prometheus_port: 9103\ncollectd_plugins: [...]"] -.-> COL_MAIN
        COL_MAIN["tasks/main.yml"] -->|present| COL_INST["tasks/install_collectd.yml\napt update_cache (Debian) + package collectd"]
        COL_MAIN -->|present| COL_TPL["Despliegue de plantilla\ntemplates/template.j2"]
        COL_MAIN -->|present| COL_SVC["Asegura servicio collectd\nstate: started, enabled: true"]
        COL_MAIN -->|absent| COL_REM["tasks/remove_collectd.yml\nDetiene servicio, borra config y paquete"]

        COL_INST -. "notify" .-> COL_HND["handlers/main.yml\nRestart_collectd_service"]
        COL_TPL -. "notify" .-> COL_HND
    end

    R1 --> C_MAIN
    R2 --> COL_MAIN

    COL_TPL --> CFG["/etc/collectd/collectd.conf.d/prometheus.conf\nPuerto 9103 + 8 plugins de métricas"]
    CFG --> PROM["Prometheus Scraper\nHTTP :9103"]
```

---

## 2. Flujo de Ejecución: `common_playbook.yml` y Rol `common`

El playbook [`common_playbook.yml`](../common_playbook.yml) aplica el rol [`common`](../common/) sobrescribiendo `install_package: true` (su valor por defecto en `common/defaults/main.yml` es `false`).

```mermaid
flowchart TD
    START(["Inicio: common_playbook.yml\nhosts: managed_nodes, become: yes"]) --> VARS["Variables\ninstall_package = true\nreboot_on_apparmor_disable = false"]
    VARS --> MAIN["common/tasks/main.yml"]

    MAIN --> INC1["ansible.builtin.include_tasks:\ninstall_packages.yml"]
    INC1 --> CHK_PKG{"¿install_package == true?"}
    CHK_PKG -- "Sí" --> DO_PKG["ansible.builtin.package\nstate: present\ncurl, lsof, mc, nano, tar, unzip, vim, zip"]
    CHK_PKG -- "No (default: false)" --> SKIP_PKG["Omitir instalación de paquetes"]

    DO_PKG --> INC2["ansible.builtin.include_tasks:\napparmor.yml"]
    SKIP_PKG --> INC2

    INC2 --> CHK_AA["ansible.builtin.command\nsystemctl is-active apparmor\nregister: apparmor_status"]
    CHK_AA --> IS_ACT{"¿apparmor_status.stdout == 'active'?"}

    IS_ACT -- "No" --> END_OK(["Fin del playbook\nSin cambios en AppArmor"])
    IS_ACT -- "Sí" --> STOP_AA["ansible.builtin.systemd\nname: apparmor\nstate: stopped"]
    STOP_AA --> DIS_AA["ansible.builtin.systemd\nname: apparmor\nenabled: no\nnotify: Reboot"]

    DIS_AA --> HND["Handler: Reboot\ncommon/handlers/main.yml"]
    HND --> CHK_REB{"¿reboot_on_apparmor_disable | bool?"}
    CHK_REB -- "false (default)" --> END_OK
    CHK_REB -- "true" --> REBOOT["ansible.builtin.reboot\nreboot_timeout: 300"]
    REBOOT --> END_REB(["Fin del playbook tras reinicio"])
```

---

## 3. Flujo de Ejecución: `install_collectd_playbook.yml` y Rol `collectd`

El playbook [`install_collectd_playbook.yml`](../install_collectd_playbook.yml) instala `collectd`, despliega la configuración parametrizada del plugin `write_prometheus` junto con 8 plugins de métricas del sistema y habilita el servicio mediante `systemd`. Si se define `collectd_state: absent`, ejecuta la desinstalación controlada con `remove_collectd.yml`.

```mermaid
flowchart TD
    START(["Inicio: install_collectd_playbook.yml\nhosts: managed_nodes, become: yes"]) --> MAIN["collectd/tasks/main.yml"]
    MAIN --> STATE{"¿collectd_state?"}

    STATE -- "present (default)" --> INC1["ansible.builtin.include_tasks:\ninstall_collectd.yml"]
    INC1 --> APT["Si os_family == 'Debian':\nansible.builtin.apt update_cache"]
    APT --> PKG["ansible.builtin.package\nname: collectd, state: latest"]
    PKG -. "Si hay cambio: notify" .-> HND

    PKG --> TPL["ansible.builtin.template\nsrc: template.j2\ndest: /etc/collectd/collectd.conf.d/prometheus.conf\nroot:root 0644"]
    TPL -. "Si hay cambio: notify" .-> HND

    TPL --> SVC["ansible.builtin.systemd\nname: collectd\nstate: started, enabled: true"]
    SVC --> END_TASKS{"¿Alguna tarea notificó al handler?"}
    END_TASKS -- "No" --> FINISH(["Fin del playbook"])
    END_TASKS -- "Sí" --> HND["Handler: Restart_collectd_service\nansible.builtin.systemd\nname: collectd, state: restarted, enabled: yes"]
    HND --> FINISH

    STATE -- "absent" --> REM["ansible.builtin.include_tasks:\nremove_collectd.yml"]
    REM --> REM1["Detiene y deshabilita servicio collectd (si existe)"]
    REM1 --> REM2["Elimina /etc/collectd/collectd.conf.d/prometheus.conf"]
    REM2 --> REM3["Desinstala paquete collectd (state: absent)"]
    REM3 --> FINISH
```

---

## 4. Diagrama de Secuencia de Aprovisionamiento

Representación de la interacción temporal entre el nodo de control de Ansible, los nodos destino (`managed_nodes`) y un servidor de monitoreo Prometheus al ejecutar ambos playbooks en orden.

```mermaid
sequenceDiagram
    actor Admin as Operador / Ansible Control Node
    participant Node as Nodo Gestionado (managed_nodes)
    participant Systemd as systemd
    participant Collectd as Servicio collectd (:9103)
    participant Prom as Prometheus

    Note over Admin,Node: Fase 1: Ejecución de common_playbook.yml
    Admin->>Node: Instala paquetes base (curl, lsof, mc, nano, tar, unzip, vim, zip)
    Admin->>Systemd: systemctl is-active apparmor
    Systemd-->>Admin: stdout ("active" / "inactive")
    alt AppArmor está activo
        Admin->>Systemd: Detener servicio apparmor (state: stopped)
        Admin->>Systemd: Deshabilitar apparmor en arranque (enabled: no)
        opt Si reboot_on_apparmor_disable == true
            Admin->>Node: Handler Reboot (timeout 300s)
        end
    end

    Note over Admin,Collectd: Fase 2: Ejecución de install_collectd_playbook.yml
    Admin->>Node: Actualiza caché apt (si Debian) e instala/actualiza collectd
    Admin->>Node: Renderiza template.j2 en /etc/collectd/collectd.conf.d/prometheus.conf (0644)
    Admin->>Systemd: Inicia y habilita servicio collectd
    Systemd->>Collectd: Arranca daemon con plugins configurados
    opt Si cambió el paquete o la plantilla
        Admin->>Systemd: Handler Restart_collectd_service (state: restarted)
        Systemd->>Collectd: Reinicia servicio collectd
    end

    Note over Collectd,Prom: Fase 3: Operación / Monitoreo
    Prom->>Collectd: HTTP GET /metrics (puerto 9103)
    Collectd-->>Prom: Métricas del sistema en formato Prometheus
```

---

## 5. Inventario Detallado de Scripts y Archivos

| Archivo | Tipo | Propósito | Módulos Ansible | Variables / Handlers | Estado |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `common_playbook.yml` | Playbook | Ejecuta el rol `common` sobre `managed_nodes` con `become: yes` | — | Define `install_package: true` | Activo |
| `install_collectd_playbook.yml` | Playbook | Ejecuta el rol `collectd` sobre `managed_nodes` con `become: yes` | — | Sin variables adicionales | Activo |
| `common/defaults/main.yml` | Defaults | Define valores por defecto del rol `common` | — | `install_package: false`, `reboot_on_apparmor_disable: false` | Activo |
| `common/vars/main.yml` | Vars | Archivo de variables del rol `common` | — | Vacío | Plantilla |
| `common/tasks/main.yml` | Tasks | Punto de entrada del rol `common`; incluye `install_packages.yml` y `apparmor.yml` | `ansible.builtin.include_tasks` | — | Activo |
| `common/tasks/install_packages.yml` | Tasks | Instala 8 utilidades base (`curl`, `lsof`, `mc`, `nano`, `tar`, `unzip`, `vim`, `zip`) | `ansible.builtin.package` | Condición: `when: install_package` | Activo |
| `common/tasks/apparmor.yml` | Tasks | Verifica, detiene y deshabilita el servicio `apparmor` si está activo | `ansible.builtin.command`, `ansible.builtin.systemd` | Registra `apparmor_status`; notifica `Reboot` | Activo |
| `common/handlers/main.yml` | Handlers | Reinicia el nodo tras deshabilitar AppArmor (timeout 300s) | `ansible.builtin.reboot` | Condición: `when: reboot_on_apparmor_disable \| bool` | Activo |
| `common/tests/test.yml` | Test | Playbook de prueba autogenerado por `ansible-galaxy init` | — | Apunta a `localhost` como `root` | Plantilla |
| `collectd/defaults/main.yml` | Defaults | Define valores por defecto del rol `collectd` | — | `collectd_state`, `collectd_prometheus_port`, `collectd_plugins` | Activo |
| `collectd/vars/main.yml` | Vars | Archivo de variables del rol `collectd` | — | Vacío | Plantilla |
| `collectd/tasks/main.yml` | Tasks | Orquesta instalación/configuración (`present`) o desinstalación (`absent`) de `collectd` | `ansible.builtin.include_tasks`, `ansible.builtin.template`, `ansible.builtin.systemd` | Notifica `Restart_collectd_service` solo al cambiar plantilla/paquete | Activo |
| `collectd/tasks/install_collectd.yml` | Tasks | Actualiza caché en Debian e instala `collectd` | `ansible.builtin.apt`, `ansible.builtin.package` | Notifica `Restart_collectd_service` | Activo |
| `collectd/tasks/remove_collectd.yml` | Tasks | Detiene servicio, elimina configuración Prometheus y desinstala `collectd` | `ansible.builtin.command`, `ansible.builtin.systemd`, `ansible.builtin.file`, `ansible.builtin.package` | Ejecutado cuando `collectd_state == 'absent'` | Activo |
| `collectd/templates/template.j2` | Template | Configuración parametrizada de plugins de `collectd` (`write_prometheus` + lista `collectd_plugins`) | — | Usa `collectd_prometheus_port` y `collectd_plugins` | Activo |
| `collectd/handlers/main.yml` | Handlers | Reinicia y habilita el servicio `collectd` | `ansible.builtin.systemd` | Handler `Restart_collectd_service` | Activo |
| `collectd/tests/test.yml` | Test | Playbook de prueba autogenerado por `ansible-galaxy init` | — | Apunta a `localhost` como `root` | Plantilla |

---

## 6. Correcciones Aplicadas y Recomendaciones Pendientes

### Hallazgos corregidos en `agent-review`

1. **Handler `Reboot` (`common/handlers/main.yml` y `common/defaults/main.yml`):** Se reemplazó la variable indefinida `disable_selinux` por `reboot_on_apparmor_disable` (con valor por defecto `false`).
2. **Actualización de caché portable (`collectd/tasks/install_collectd.yml`):** Se removió `update_cache: yes` de `ansible.builtin.package` y se agregó una tarea condicional con `ansible.builtin.apt` para sistemas `os_family == 'Debian'`.
3. **Limpieza e implementación de tareas en `collectd/tasks/`:** Se eliminó el archivo sin uso `collectd/tasks/install_packages.yml` y se implementó `collectd/tasks/remove_collectd.yml`, controlado mediante `collectd_state: present | absent`.
4. **Eliminación de notificación redundante (`collectd/tasks/main.yml`):** Se retiró `notify: Restart_collectd_service` de la tarea que inicia el servicio `collectd`.
5. **Parametrización de `collectd/templates/template.j2`:** Se parametrizaron `collectd_prometheus_port` y `collectd_plugins` desde `collectd/defaults/main.yml`.
6. **Consistencia de FQCN:** Se actualizó `include_tasks` a `ansible.builtin.include_tasks` en `common/tasks/main.yml` y `collectd/tasks/main.yml`.

### Recomendaciones opcionales a futuro

- Añadir un inventario de ejemplo (`inventory.ini.example`) y un `ansible.cfg` base en la raíz del proyecto.
- Agregar pruebas automatizadas con `ansible-lint` o Molecule en un flujo de CI.
