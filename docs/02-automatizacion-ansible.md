# Automatización con Ansible (fase 2)

En la fase 1 el servidor se configuró a mano. Aquí se automatiza con **Ansible**: la instalación de paquetes y la configuración del firewall pasan a describirse en un archivo y se aplican de forma repetible.

## Cómo funciona

- Todo lo que debe tener la VM se describe en un **playbook**, un archivo YAML.
- Ansible **no necesita ningún agente** instalado en la VM: se conecta por **SSH**.
- Al conectarse, compara el estado real de la VM con el del playbook y solo cambia lo que falta. Por eso ejecutarlo varias veces da el mismo resultado (*idempotencia*): si no hay nada que cambiar, devuelve `changed=0`.

## Playbook `site.yml`

```yaml
---
- name: Configuración base del servidor lab
  hosts: lab
  become: true

  tasks:
    - name: Instalar paquetes del stack
      ansible.builtin.apt:
        name:
          - nginx
          - mariadb-server
          - php-fpm
          - php-mysql
          - ufw
          - fail2ban
        state: present
        update_cache: true

    - name: Permitir SSH, HTTP y HTTPS (solo TCP)
      community.general.ufw:
        rule: allow
        port: "{{ item }}"
        proto: tcp
      loop: ["22", "80", "443"]

    - name: Activar el firewall con entrada denegada por defecto
      community.general.ufw:
        state: enabled
        default: deny
        direction: incoming
```

### Qué hace cada parte

| Elemento | Función |
| --- | --- |
| `name` (play) | Etiqueta descriptiva del playbook |
| `hosts: lab` | Grupo de máquinas sobre las que se ejecuta (definido en el inventario de Ansible) |
| `become: true` | Usa privilegios de `sudo` |
| `ansible.builtin.apt` con `state: present` | Comprueba que los paquetes de la lista estén instalados y, si no, los instala |
| `community.general.ufw` con `loop` | Abre los puertos 22, 80 y 443 (solo TCP), uno por iteración |
| `state: enabled` + `default: deny` | Activa el firewall y deniega por defecto todo el tráfico entrante que no esté permitido |

Igual que en la configuración manual, SSH se permite antes de activar el firewall.

## Ejecución

Validar la sintaxis del YAML:

```bash
ansible-playbook site.yml --syntax-check
```

Simular qué cambiaría, sin tocar la VM:

```bash
ansible-playbook site.yml --check --diff -K
```

- `--check`: simula sin cambiar nada (equivale al `-s` de `apt-get`).
- `--diff`: muestra qué cambiaría.
- `-K`: pide la contraseña de `sudo` del usuario remoto.

Ejecutarlo de verdad (se quita `--check`):

```bash
ansible-playbook site.yml --diff -K
```

## Handlers

Un **handler** en Ansible es una tarea especial que solo se ejecuta cuando otra tarea la avisa con `notify` y esa tarea ha **cambiado** algo. Se usa, por ejemplo, para recargar un servicio solo si su configuración se ha modificado:

```yaml
tasks:
  - name: Copiar configuración de nginx
    ansible.builtin.template:
      src: lab.local.j2
      dest: /etc/nginx/sites-available/lab.local
    notify: Recargar nginx

handlers:
  - name: Recargar nginx
    ansible.builtin.service:
      name: nginx
      state: reloaded
```

El término es más general en programación: un *event handler* es el código que gestiona un evento (por ejemplo, un clic de ratón en una interfaz). Ansible aplica la misma idea: el «evento» es que una tarea haya cambiado algo.
