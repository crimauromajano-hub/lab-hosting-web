# Configuración manual del servidor (fase 1)

Servidor Ubuntu Server en VirtualBox con un stack LEMP para WordPress, protegido con firewall, fail2ban y una superficie de ataque reducida. Esta primera fase se hizo **a mano** para entender cada pieza; en la fase 2 se automatiza con Ansible (ver `02-automatizacion-ansible.md`).

## Índice

1. [Firewall con ufw](#1-firewall-con-ufw)
2. [Stack LEMP](#2-stack-lemp)
3. [MariaDB para WordPress](#3-mariadb-para-wordpress)
4. [Nginx: server block](#4-nginx-server-block)
5. [fail2ban](#5-fail2ban)
6. [Reducir la superficie de ataque](#6-reducir-la-superficie-de-ataque)
7. [Conceptos clave](#7-conceptos-clave)

---

## 1. Firewall con ufw

### Cómo encajan las piezas

| Pieza | Qué es | Dónde actúa |
| --- | --- | --- |
| **Netfilter** | Firewall del kernel de Linux | Paquetes de red |
| **nftables / iptables** | Herramientas para configurar Netfilter | Reglas |
| **ufw** | Interfaz sencilla sobre nftables/iptables | Reglas |
| **fail2ban** | Lee logs y ordena al firewall bloquear IPs | Logs y, de forma indirecta, reglas |

### Reglas básicas

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80
sudo ufw allow 443
sudo ufw enable
```

Se permite SSH **antes** de activar el firewall para no quedarse sin acceso a la máquina.

### Comprobar el resultado

```bash
sudo ufw status verbose
```

### Gestionar reglas

```bash
sudo ufw status numbered          # listar reglas numeradas
sudo ufw allow 8000/tcp           # añadir una regla (solo TCP)
sudo ufw delete allow 8000/tcp    # borrar la regla
```

Si no se indica `/tcp`, ufw abre el puerto para TCP **y** UDP.

### ufw frente a nft

Hay dos formas de revisar una regla:

| Comando | Qué muestra | Cuándo usarlo |
| --- | --- | --- |
| `sudo ufw status numbered` | Las reglas tal como se configuraron en ufw, numeradas | Día a día: ver qué se ha permitido y borrar por número |
| `sudo nft list ruleset` | **Todas** las reglas que aplica el kernel, en sintaxis moderna | Verificar lo que se aplica de verdad y depurar |

Por ejemplo, para ver cómo se ha traducido la regla del puerto 8000:

```bash
sudo nft list ruleset | grep 8000
```

---

## 2. Stack LEMP

**LEMP** es el conjunto de programas que sirve una web dinámica:

- **L, Linux:** el sistema operativo donde corre todo (Ubuntu Server).
- **E, Nginx** (se pronuncia «engine-x», de ahí la E): el servidor web. Recibe las peticiones HTTP/HTTPS del navegador y entrega la página.
- **M, MariaDB** (o MySQL): la base de datos. WordPress guarda aquí entradas, usuarios y ajustes.
- **P, PHP:** el lenguaje de WordPress. Se ejecuta con **PHP-FPM**, un servicio aparte al que Nginx pasa los archivos `.php` para que los procese.

```bash
sudo apt install nginx mariadb-server php-fpm php-mysql
```

---

## 3. MariaDB para WordPress

```bash
sudo mysql
```

### Crear la base de datos

```sql
CREATE DATABASE wordpress CHARACTER SET utf8mb4;
```

`CHARACTER SET utf8mb4` define el juego de caracteres de la base de datos. `utf8mb4` es el más completo: admite cualquier carácter Unicode (acentos, ñ, emojis) sin errores al guardar.

### Crear un usuario con el mínimo privilegio

```sql
CREATE USER 'wpuser'@'localhost' IDENTIFIED BY '<CONTRASEÑA_FUERTE>';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wpuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

- `GRANT ALL PRIVILEGES ON wordpress.*` concede todos los permisos (leer, escribir, modificar…) **solo sobre la base de datos `wordpress`**.
- Es mínimo privilegio porque el `.*` limita los permisos a esa base de datos: si alguien compromete a `wpuser`, solo puede dañar esa base de datos y no el resto del servidor.
- `'wpuser'@'localhost'` restringe además el origen: el usuario solo puede conectarse desde la propia máquina.
- `FLUSH PRIVILEGES` ordena a MariaDB recargar en memoria las tablas de permisos.

> La contraseña real no se publica en el repositorio.

### Comprobar que MariaDB escucha solo en local

```bash
sudo ss -tulpn | grep 3306
```

`ss` **muestra** qué procesos están escuchando. Mi primera idea fue usar `nc -lvnp 3306`, pero ese comando hace otra cosa: **abre un listener propio** en el puerto, no comprueba el que ya existe (y fallaría si MariaDB ya lo está usando). Para verificar un servicio, `ss`; para abrir un puerto de pruebas, `nc`.

---

## 4. Nginx: server block

Un **server block** es el archivo que describe un sitio web en Nginx: en qué puerto escucha, con qué nombre, desde qué carpeta sirve y cómo trata cada tipo de petición.

```bash
sudo tee /etc/nginx/sites-available/lab.local > /dev/null <<'EOF'
server {
    listen 80;
    server_name lab.local;
    root /var/www/lab.local;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.5-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }
}
EOF
```

Qué hace cada bloque:

- `location /`: intenta servir el archivo o carpeta pedidos y, si no existen, pasa la petición a `index.php` (necesario para los enlaces permanentes de WordPress).
- `location ~ \.php$`: envía los archivos `.php` a PHP-FPM a través de un **socket Unix**. La versión del socket debe coincidir con la de PHP instalada.
- `location ~ /\.ht { deny all; }`: **bloquea el acceso a los archivos que empiezan por `.ht`** (`.htaccess`, `.htpasswd`), restos de Apache que pueden contener configuración o contraseñas. Es una protección preventiva. La barra invertida escapa el punto: en una expresión regular el punto suelto significa «cualquier carácter», y así se busca un punto literal.

Activar el sitio y comprobar la configuración:

```bash
sudo ln -s /etc/nginx/sites-available/lab.local /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## 5. fail2ban

fail2ban lee los logs de un servicio, detecta intentos fallidos y ordena al firewall bloquear la IP que los provoca. Funciona como un sistema de prevención de intrusiones basado en logs.

Su configuración se organiza en **jails**: reglas de vigilancia para un servicio concreto (qué log leer, qué patrones cuentan como fallo y qué hacer con la IP que los provoca).

### Qué archivo modificar

- `/etc/fail2ban/jail.conf`: configuración **original** del paquete. Define los valores por defecto y todos los jails disponibles (casi todos desactivados). No se edita.
- `/etc/fail2ban/jail.local`: archivo **propio**. fail2ban lo lee después de `jail.conf` y lo que se escribe aquí sobrescribe lo anterior; solo hace falta indicar lo que se quiere cambiar.

### Configuración (`jail.local`)

```ini
[DEFAULT]
bantime  = 10m
findtime = 10m
maxretry = 5
ignoreip = 127.0.0.1/8 ::1 10.0.2.2

[sshd]
enabled = true
mode    = aggressive
```

| Parámetro | Significado |
| --- | --- |
| `maxretry` | Fallos permitidos antes del baneo |
| `findtime` | Ventana de tiempo en la que se cuentan esos fallos |
| `bantime` | Duración del baneo |
| `ignoreip` | IPs que nunca se banean (aquí, localhost y la puerta de enlace NAT de VirtualBox) |
| `mode = aggressive` | Cuenta también las conexiones rechazadas por clave, no solo las contraseñas incorrectas. Útil si la contraseña de SSH ya está desactivada |

### Comprobar

```bash
sudo fail2ban-client status sshd
```

---

## 6. Reducir la superficie de ataque

El firewall protege del exterior, pero el interior sigue siendo vulnerable si alguien consigue entrar o saltarse esa protección. Por eso se aplica **seguridad por capas**: además del firewall, se desactivan los servicios que el stack no usa.

### 6.1 Ver qué hay activo

```bash
ss -tulpn                                                    # puertos TCP/UDP abiertos y su proceso
systemctl list-units --type=service --state=running          # servicios en marcha
systemctl list-unit-files --type=service --state=enabled     # servicios que arrancan con el sistema
ss -xlnp                                                     # sockets Unix locales
```

| Pregunta | Herramienta |
| --- | --- |
| ¿Qué puede alcanzar alguien por la red? | `ss -tulpn` |
| ¿Qué corre como servicio, aunque no escuche en la red? | `systemctl list-units` |
| ¿Qué escucha en sockets locales (Unix)? | `ss -xlnp` |

`ss -tulpn` incluye procesos que **no son servicios de systemd**, por eso conviene usar las dos herramientas.

### 6.2 Comprobar dependencias antes de desactivar nada

`systemctl` es el gestor del sistema y de los servicios en Linux moderno. Antes de quitar algo hay que saber si otro servicio lo necesita. Ejemplo con nginx:

```bash
systemctl list-dependencies --reverse nginx   # qué servicios dependen de nginx
systemctl list-dependencies nginx             # qué necesita nginx para funcionar
```

### 6.3 Simular antes de eliminar

```bash
apt-get -s purge nginx
```

- `-s` (*simulate*): calcula todo pero **no cambia nada**, por eso no hace falta `sudo`.
- `purge`: desinstala el paquete y **borra también su configuración**. `remove`, en cambio, quita el programa y deja la configuración en `/etc`.

Si la simulación es correcta, se elimina de verdad:

```bash
sudo apt-get purge nginx
```

---

## 7. Conceptos clave

**Socket.** Extremo de una comunicación entre procesos. Un socket de red (TCP/UDP) queda identificado por una IP, un puerto y un protocolo de transporte, y conecta un proceso con otro, en la misma máquina o en otra. Un socket **Unix** (como el de PHP-FPM) comunica procesos de una misma máquina y aparece como un archivo en el sistema. Un pipe es parecido, pero solo conecta procesos locales. Un programa que está «escuchando» ha creado su socket y espera a que alguien se conecte.
