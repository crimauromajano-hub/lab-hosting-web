# lab-hosting-web

Laboratorio de hosting web en una máquina virtual Ubuntu Server (VirtualBox): un stack **LEMP** para WordPress, protegido con firewall, fail2ban y una superficie de ataque reducida, primero configurado a mano y después automatizado con **Ansible**.

El objetivo es aprender cómo se monta y se asegura un servidor web de principio a fin, y dejar documentado el porqué de cada decisión, no solo los comandos.

## Qué incluye

- **Stack LEMP:** Linux, Nginx, MariaDB y PHP-FPM.
- **Firewall con ufw:** entrada denegada por defecto y solo SSH, HTTP y HTTPS permitidos, verificado también con `nft`.
- **fail2ban:** protección del servicio SSH contra intentos repetidos de acceso.
- **Mínimo privilegio en la base de datos:** un usuario de MariaDB con permisos únicamente sobre la base de datos de WordPress y solo desde `localhost`.
- **Server block de Nginx** con bloqueo de archivos `.ht*` y PHP servido por socket Unix.
- **Reducción de la superficie de ataque:** revisión de puertos y servicios activos, comprobación de dependencias y simulación antes de desinstalar.
- **Automatización con Ansible:** instalación de paquetes y configuración del firewall mediante un playbook, ejecutable de forma repetible.

## Documentación

| Fase | Documento | Contenido |
| --- | --- | --- |
| 1 | [Configuración manual](docs/01-configuracion-manual.md) | Firewall, LEMP, MariaDB, Nginx, fail2ban y reducción de servicios, paso a paso y con la explicación de cada comando |
| 2 | [Automatización con Ansible](docs/02-automatizacion-ansible.md) | Playbook `site.yml`, simulación con `--check`, ejecución y handlers |

## Cómo está planteado

1. **Hacerlo a mano primero**, para entender qué hace cada pieza y cómo se comprueba que funciona.
2. **Automatizarlo después**, para que el mismo resultado se pueda reproducir en otra máquina sin repetir los pasos.

## Notas

- Las contraseñas y otros secretos no se incluyen en el repositorio; en los ejemplos aparecen como marcadores (`<CONTRASEÑA_FUERTE>`).
- Es un laboratorio local: el dominio `lab.local` y las IP que aparecen corresponden a la red NAT de VirtualBox.
