# Arquitectura, Documentación y Diagramas de Scripts

Este documento presenta el análisis técnico completo del repositorio **`ansible-sandbox`**, incluyendo diagramas de arquitectura, flujos de ejecución por playbook y rol, secuencia de interacción con los nodos gestionados, inventario detallado de archivos y observaciones técnicas detectadas durante la revisión.

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
        C_DEF["defaults/main.yml\ninstall_package: false"] -.-> C_MAIN
        C_MAIN["tasks/main.yml"] --> C_PKG["tasks/install_packages.yml\nInstala 8 paquetes base"]
        C_MAIN --> C_AA["tasks/apparmor.yml\nDetiene y deshabilita AppArmor"]
        C_AA -. "notify: Reboot" .-> C_HND["handlers/main.yml\nReboot (when: disable_selinux)"]
    end

    subgraph COLLECTD["Rol collectd"]
        COL_MAIN["tasks/main.yml"] --> COL_INST["tasks/install_collectd.yml\nInstala paquete collectd"]
        COL_MAIN --> COL_TPL["Despliegue de plantilla\ntemplates/template.j2"]
        COL_MAIN --> COL_SVC["Asegura servicio collectd\nstate: started, enabled: yes"]

        COL_INST -. "notify" .-> COL_HND["handlers/main.yml\nRestart_collectd_service"]
        COL_TPL -. "notify" .-> COL_HND
        COL_SVC -. "notify" .-> COL_HND

        COL_UNUSED1["tasks/install_packages.yml\n(No incluido en main.yml)"]
        COL_UNUSED2["tasks/remove_collectd.yml\n(Vacío / No incluido)"]
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
    START(["Inicio: common_playbook.yml\nhosts: managed_nodes, become: yes"]) --> VARS["Sobrescribe variable\ninstall_package = true"]
    VARS --> MAIN["common/tasks/main.yml"]

    MAIN --> INC1["include_tasks: install_packages.yml"]
    INC1 --> CHK_PKG{"¿install_package == true?"}
    CHK_PKG -- "Sí" --> DO_PKG["ansible.builtin.package\nstate: present\ncurl, lsof, mc, nano, tar, unzip, vim, zip"]
    CHK_PKG -- "No (default: false)" --> SKIP_PKG["Omitir instalación de paquetes"]

    DO_PKG --> INC2["include_tasks: apparmor.yml"]
    SKIP_PKG --> INC2

    INC2 --> CHK_AA["ansible.builtin.command\nsystemctl is-active apparmor\nregister: apparmor_status"]
    CHK_AA --> IS_ACT{"¿apparmor_status.stdout == 'active'?"}

    IS_ACT -- "No" --> END_OK(["Fin del playbook\nSin cambios en AppArmor"])
    IS_ACT -- "Sí" --> STOP_AA["ansible.builtin.systemd\nname: apparmor\nstate: stopped"]
    STOP_AA --> DIS_AA["ansible.builtin.systemd\nname: apparmor\nenabled: no\nnotify: Reboot"]

    DIS_AA --> HND["Handler: Reboot\ncommon/handlers/main.yml"]
    HND --> CHK_SEL{"Evaluación condición\nwhen: disable_selinux"}
    CHK_SEL -- "Variable indefinida" --> ERR["Error en tiempo de ejecución\n'disable_selinux' is undefined"]
    CHK_SEL -- "Si se define como true" --> REBOOT["ansible.builtin.reboot\nreboot_timeout: 300"]
    REBOOT --> END_REB(["Fin del playbook tras reinicio"])
```

---

## 3. Flujo de Ejecución: `install_collectd_playbook.yml` y Rol `collectd`

El playbook [`install_collectd_playbook.yml`](../install_collectd_playbook.yml) instala `collectd`, despliega la configuración del plugin `write_prometheus` junto con 8 plugins de métricas del sistema y habilita el servicio mediante `systemd`.

```mermaid
flowchart TD
    START(["Inicio: install_collectd_playbook.yml\nhosts: managed_nodes, become: yes"]) --> MAIN["collectd/tasks/main.yml"]

    MAIN --> INC1["include_tasks: install_collectd.yml"]
    INC1 --> PKG["ansible.builtin.package\nname: collectd\nstate: latest, update_cache: yes"]
    PKG -. "Si hay cambio: notify" .-> HND

    PKG --> TPL["ansible.builtin.template\nsrc: template.j2\ndest: /etc/collectd/collectd.conf.d/prometheus.conf\nroot:root 0644"]
    TPL -. "Si hay cambio: notify" .-> HND

    TPL --> SVC["ansible.builtin.systemd\nname: collectd\nstate: started, enabled: yes"]
    SVC -. "Si hay cambio: notify" .-> HND

    SVC --> END_TASKS{"¿Alguna tarea notificó al handler?"}
    END_TASKS -- "No" --> FINISH(["Fin del playbook"])
    END_TASKS -- "Sí" --> HND["Handler: Restart_collectd_service\nansible.builtin.systemd\nname: collectd, state: restarted, enabled: yes"]
    HND --> FINISH

    subgraph UNUSED["Archivos presentes pero NO incluidos en tasks/main.yml"]
        U1["install_packages.yml\nInstala paquete 'write_prometheus'"]
        U2["remove_collectd.yml\nArchivo vacío (0 bytes)"]
    end
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
        Note over Admin,Node: Notifica handler "Reboot" (condicionado a disable_selinux)
    end

    Note over Admin,Collectd: Fase 2: Ejecución de install_collectd_playbook.yml
    Admin->>Node: Instala/actualiza paquete collectd
    Admin->>Node: Copia template.j2 a /etc/collectd/collectd.conf.d/prometheus.conf (0644)
    Admin->>Systemd: Inicia y habilita servicio collectd
    Systemd->>Collectd: Arranca daemon con plugins (df, processes, protocols, swap, tcpconns, uptime, users, vmem, write_prometheus)
    opt Si hubo cambios en paquete, plantilla o servicio
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
| `common/defaults/main.yml` | Defaults | Define el comportamiento por defecto del rol `common` | — | `install_package: false` | Activo |
| `common/vars/main.yml` | Vars | Archivo de variables del rol `common` | — | Vacío | Plantilla |
| `common/tasks/main.yml` | Tasks | Punto de entrada del rol `common`; incluye `install_packages.yml` y `apparmor.yml` | `include_tasks` | — | Activo |
| `common/tasks/install_packages.yml` | Tasks | Instala 8 utilidades base (`curl`, `lsof`, `mc`, `nano`, `tar`, `unzip`, `vim`, `zip`) | `ansible.builtin.package` | Condición: `when: install_package` | Activo |
| `common/tasks/apparmor.yml` | Tasks | Verifica, detiene y deshabilita el servicio `apparmor` si está activo | `ansible.builtin.command`, `ansible.builtin.systemd` | Registra `apparmor_status`; notifica `Reboot` | Activo |
| `common/handlers/main.yml` | Handlers | Reinicia el nodo tras cambios de seguridad (timeout 300s) | `ansible.builtin.reboot` | Condición: `when: disable_selinux` | Con hallazgo |
| `common/tests/test.yml` | Test | Playbook de prueba autogenerado por `ansible-galaxy init` | — | Apunta a `localhost` como `root` | Plantilla |
| `collectd/defaults/main.yml` | Defaults | Archivo de valores por defecto del rol `collectd` | — | Vacío | Plantilla |
| `collectd/vars/main.yml` | Vars | Archivo de variables del rol `collectd` | — | Vacío | Plantilla |
| `collectd/tasks/main.yml` | Tasks | Incluye `install_collectd.yml`, despliega `template.j2` y asegura servicio `collectd` | `include_tasks`, `ansible.builtin.template`, `ansible.builtin.systemd` | Notifica `Restart_collectd_service` | Activo |
| `collectd/tasks/install_collectd.yml` | Tasks | Instala el paquete `collectd` en su última versión | `ansible.builtin.package` | Notifica `Restart_collectd_service` | Con hallazgo |
| `collectd/tasks/install_packages.yml` | Tasks | Intenta instalar el paquete `write_prometheus` | `ansible.builtin.package` | Notifica `Restart_collectd_service` | Sin incluir |
| `collectd/tasks/remove_collectd.yml` | Tasks | Reservado para tareas de desinstalación de `collectd` | — | — | Vacío / Sin incluir |
| `collectd/templates/template.j2` | Template | Configuración de plugins de `collectd` (`write_prometheus` en puerto `9103` + 8 plugins de sistema) | — | Sin variables Jinja2 | Activo |
| `collectd/handlers/main.yml` | Handlers | Reinicia y habilita el servicio `collectd` | `ansible.builtin.systemd` | Handler `Restart_collectd_service` | Activo |
| `collectd/tests/test.yml` | Test | Playbook de prueba autogenerado por `ansible-galaxy init` | — | Apunta a `localhost` como `root` | Plantilla |

---

## 6. Observaciones Técnicas y Hallazgos de Revisión

Durante el análisis de los scripts se identificaron los siguientes puntos a tener en cuenta antes de ejecutarlos en entornos productivos:

1. **Variable indefinida en el handler `Reboot` (`common/handlers/main.yml`)**
   - El handler evalúa `when: disable_selinux`, pero dicha variable no existe en `common/defaults/main.yml`, `common/vars/main.yml` ni en `common_playbook.yml`.
   - Si AppArmor está activo en el nodo destino, la tarea de deshabilitación notificará al handler `Reboot` y Ansible fallará con `'disable_selinux' is undefined`.
   - **Recomendación:** Definir una variable explícita (por ejemplo `reboot_on_apparmor_disable: false` en `common/defaults/main.yml`) y actualizar la condición del handler.

2. **Parámetro `update_cache: yes` en `ansible.builtin.package` (`collectd/tasks/install_collectd.yml` y `collectd/tasks/install_packages.yml`)**
   - `ansible.builtin.package` es un módulo agnóstico de distribución que delega en el gestor de paquetes subyacente y no documenta `update_cache` como parámetro estándar multiplataforma (es específico de `ansible.builtin.apt`). En sistemas basados en `dnf`/`yum` o con validación estricta puede generar errores.
   - **Recomendación:** Usar `ansible.builtin.apt` (si el objetivo es Debian/Ubuntu) o actualizar la caché en una tarea condicional según `ansible_facts['os_family']`.

3. **Archivos de tareas no utilizados en el rol `collectd`**
   - `collectd/tasks/install_packages.yml` no se incluye desde `collectd/tasks/main.yml`, y el nombre de paquete `write_prometheus` no suele coincidir con el nombre real en repositorios estándar (en Debian/Ubuntu viene incluido en `collectd`/`collectd-core`; en RHEL/CentOS suele llamarse `collectd-write_prometheus`).
   - `collectd/tasks/remove_collectd.yml` está vacío (0 bytes).
   - **Recomendación:** Implementar la lógica de desinstalación (por ejemplo controlada por tags o una variable de estado) o eliminar los archivos no utilizados para evitar confusión.

4. **Notificación redundante al handler en `collectd/tasks/main.yml`**
   - La tarea `Ensure collectd service is running and enabled` incluye `notify: Restart_collectd_service`. Si el servicio estaba detenido y se inicia en esa tarea, inmediatamente volverá a reiniciarse al final del play.
   - **Recomendación:** Mantener `notify: Restart_collectd_service` solo en las tareas de instalación de paquetes y despliegue de configuración (`template`).

5. **Plantilla `collectd/templates/template.j2` sin parametrizar**
   - Aunque utiliza extensión `.j2` y el módulo `ansible.builtin.template`, no contiene expresiones Jinja2: el puerto `9103` y la lista de plugins están codificados de forma fija.
   - **Recomendación:** Exponer variables en `collectd/defaults/main.yml` (p. ej. `collectd_prometheus_port: 9103` y `collectd_plugins: [...]`) para hacer reutilizable el rol.

6. **Ausencia de inventario raíz y `ansible.cfg`**
   - Ambos playbooks apuntan a `hosts: managed_nodes`, pero no existe un archivo de inventario en la raíz del proyecto ni un `ansible.cfg` que documente la configuración por defecto.
   - **Recomendación:** Añadir un archivo `inventory.example` (o `inventory/hosts.ini.example`) y un `ansible.cfg` base.

7. **Consistencia de FQCN (Fully Qualified Collection Names)**
   - Las tareas usan `ansible.builtin.package`, `ansible.builtin.systemd`, `ansible.builtin.template` y `ansible.builtin.command`, pero las inclusiones usan la forma corta `include_tasks` en lugar de `ansible.builtin.include_tasks`.
