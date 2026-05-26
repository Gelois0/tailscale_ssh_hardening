# Guía profesional: Tailscale + SSH seguro

> Instalación, configuración y operación de una red privada con Tailscale, reforzando el acceso SSH con llaves, firewall, Fail2ban y port knocking cuando aplique.


**Nivel recomendado:** principiante técnico / intermedio  
**Sistemas objetivo:** Linux, Windows, macOS, Android e iOS

---

## Índice

- [1. Objetivo de la guía](#1-objetivo-de-la-guía)
- [2. Arquitectura recomendada](#2-arquitectura-recomendada)
- [3. Requisitos previos](#3-requisitos-previos)
- [4. Conceptos clave](#4-conceptos-clave)
  - [4.1 Tailnet](#41-tailnet)
  - [4.2 IP de Tailscale `100.x.y.z`](#42-ip-de-tailscale-100xyz)
  - [4.3 MagicDNS](#43-magicdns)
  - [4.4 Subnet Router](#44-subnet-router)
  - [4.5 Exit Node](#45-exit-node)
  - [4.6 Tailscale SSH](#46-tailscale-ssh)
- [5. Instalación de Tailscale](#5-instalación-de-tailscale)
  - [5.1 Linux](#51-linux)
  - [5.2 Windows y macOS](#52-windows-y-macos)
  - [5.3 Android e iOS](#53-android-e-ios)
- [6. Autenticación de dispositivos](#6-autenticación-de-dispositivos)
- [7. Verificación de conectividad](#7-verificación-de-conectividad)
- [8. Conexión por SSH usando Tailscale](#8-conexión-por-ssh-usando-tailscale)
- [9. MagicDNS para conectarse por nombre](#9-magicdns-para-conectarse-por-nombre)
- [10. Hardening SSH recomendado](#10-hardening-ssh-recomendado)
  - [10.1 Crear una llave SSH](#101-crear-una-llave-ssh)
  - [10.2 Copiar la llave al servidor](#102-copiar-la-llave-al-servidor)
  - [10.3 Probar acceso por llave](#103-probar-acceso-por-llave)
  - [10.4 Desactivar acceso por contraseña](#104-desactivar-acceso-por-contraseña)
  - [10.5 Validar y reiniciar SSH](#105-validar-y-reiniciar-ssh)
- [11. Restringir SSH a Tailscale con firewall](#11-restringir-ssh-a-tailscale-con-firewall)
- [12. Fail2ban para protección adicional](#12-fail2ban-para-protección-adicional)
- [13. Access Controls: ACLs y grants](#13-access-controls-acls-y-grants)
- [14. Subnet Router](#14-subnet-router)
- [15. Exit Node](#15-exit-node)
- [16. Port knocking con `knockd`](#16-port-knocking-con-knockd)
- [17. Buenas prácticas operativas](#17-buenas-prácticas-operativas)
- [18. Troubleshooting](#18-troubleshooting)
- [19. Checklist final](#19-checklist-final)
- [20. Referencias oficiales](#20-referencias-oficiales)

---

## 1. Objetivo de la guía

Esta guía explica cómo crear una red privada entre varios dispositivos usando **Tailscale** y cómo reforzar el acceso remoto por **SSH**.

El objetivo principal es evitar exponer SSH directamente a Internet y usar Tailscale como red privada segura para administración remota.

La estrategia recomendada es:

1. Instalar Tailscale en todos los dispositivos.
2. Autenticar todos los equipos dentro del mismo tailnet.
3. Conectarse por IP privada de Tailscale o por MagicDNS.
4. Usar SSH con autenticación por llave.
5. Desactivar acceso SSH por contraseña.
6. Restringir SSH a la interfaz de Tailscale.
7. Añadir Fail2ban como capa adicional.
8. Usar port knocking solo si realmente se expone un puerto o se necesita una capa extra.

---

## 2. Arquitectura recomendada

```text
Laptop / PC
    |
    | Red privada Tailscale
    |
Servidor Linux ---- tailscale0 ---- SSH solo por Tailscale
    |
    └── Servicios internos: Docker, NAS, paneles, bases de datos, etc.
```

Modelo recomendado:

```text
Internet público
    ↓
Firewall bloquea SSH público
    ↓
Tailscale conecta por red privada
    ↓
SSH con llave, sin password login
```

> [!IMPORTANT]
> La opción más segura y simple es no exponer SSH a Internet. Si todos tus dispositivos usan Tailscale, puedes administrar servidores por la interfaz `tailscale0` sin abrir el puerto 22 públicamente.

---

## 3. Requisitos previos

Antes de empezar, necesitas:

- Una cuenta para iniciar sesión en Tailscale.
- Acceso administrativo al equipo Linux.
- Un usuario no-root para administrar el servidor.
- OpenSSH instalado en el servidor.
- Conexión a Internet para instalar paquetes.
- Acceso al panel de administración de Tailscale.

En Debian/Ubuntu puedes verificar SSH con:

```bash
systemctl status ssh
```

En algunas distribuciones el servicio se llama `sshd`:

```bash
systemctl status sshd
```

---

## 4. Conceptos clave

### 4.1 Tailnet

Un **tailnet** es tu red privada de Tailscale. Todos los dispositivos autenticados con la misma organización/cuenta forman parte de esa red privada.

Ejemplos de dispositivos dentro del mismo tailnet:

- Laptop personal.
- Servidor VPS.
- PC de escritorio.
- Teléfono Android/iOS.
- NAS.
- Raspberry Pi.

---

### 4.2 IP de Tailscale `100.x.y.z`

Cada dispositivo recibe una IP privada de Tailscale dentro del rango `100.64.0.0/10`.

Ejemplo:

```text
100.101.25.8    laptop-luis
100.87.13.44    servidor-vps
100.92.10.20    telefono
```

Estas IPs no son IPs públicas tradicionales. Funcionan dentro de tu tailnet y permiten conectar dispositivos como si estuvieran en una misma red privada.

Para ver la IP de Tailscale de un equipo:

```bash
tailscale ip -4
```

Para ver IPv4 e IPv6:

```bash
tailscale ip
```

---

### 4.3 MagicDNS

**MagicDNS** permite conectarte usando nombres en lugar de IPs.

Ejemplo:

```bash
ssh usuario@servidor-vps
```

En lugar de:

```bash
ssh usuario@100.87.13.44
```

MagicDNS se activa desde el panel de administración de Tailscale.

---

### 4.4 Subnet Router

Un **Subnet Router** permite acceder desde Tailscale a una red local completa detrás de un dispositivo.

Ejemplo:

```text
Laptop remota → Tailscale → Servidor casa → Red LAN 192.168.1.0/24
```

Sirve para acceder a recursos como:

- Impresoras.
- NAS.
- Cámaras internas.
- Routers.
- Servicios locales.

> [!NOTE]
> Un Subnet Router no enruta todo tu tráfico de Internet. Solo anuncia una o varias subredes específicas.

---

### 4.5 Exit Node

Un **Exit Node** permite enrutar todo el tráfico de Internet de un dispositivo a través de otro equipo dentro del tailnet.

Ejemplo:

```text
Laptop en cafetería → Tailscale → PC de casa → Internet
```

Casos de uso:

- Navegar desde redes públicas usando un punto de salida confiable.
- Usar la conexión de casa/oficina como salida.
- Proteger tráfico general cuando estás en una red Wi-Fi no confiable.

> [!WARNING]
> Subnet Router y Exit Node no son lo mismo.  
> El Subnet Router da acceso a redes internas específicas.  
> El Exit Node enruta todo el tráfico hacia Internet mediante otro dispositivo.

---

### 4.6 Tailscale SSH

Tailscale SSH permite controlar el acceso SSH mediante las políticas del tailnet.

No reemplaza necesariamente a OpenSSH tradicional, pero puede simplificar la administración de acceso entre usuarios y dispositivos dentro de Tailscale.

Para habilitarlo en una máquina:

```bash
sudo tailscale set --ssh
```

> [!NOTE]
> Si prefieres una configuración clásica y universal, puedes usar OpenSSH con llaves y restringirlo a la interfaz de Tailscale.

---

## 5. Instalación de Tailscale

### 5.1 Linux

En distribuciones Linux comunes:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Luego inicia la autenticación:

```bash
sudo tailscale up
```

El comando mostrará un enlace de autenticación. Ábrelo en el navegador e inicia sesión con la misma cuenta que usarás en tus otros dispositivos.

---

### 5.2 Windows y macOS

1. Descarga el instalador desde la página oficial de Tailscale.
2. Instala la aplicación.
3. Inicia sesión con la misma cuenta del tailnet.
4. Verifica que el dispositivo aparezca en el panel de administración.

---

### 5.3 Android e iOS

1. Instala Tailscale desde Play Store o App Store.
2. Inicia sesión con la misma cuenta.
3. Activa la conexión.
4. Verifica que el dispositivo aparezca como activo.

---

## 6. Autenticación de dispositivos

En Linux:

```bash
sudo tailscale up
```

Si el equipo ya estaba autenticado pero necesitas forzar una nueva autenticación:

```bash
sudo tailscale up --force-reauth
```

Para servidores que deben permanecer siempre conectados, revisa en el panel de Tailscale la opción de expiración de claves del dispositivo.

> [!WARNING]
> Desactivar la expiración de claves puede ser útil en servidores, pero reduce seguridad si el equipo se pierde o queda comprometido.

---

## 7. Verificación de conectividad

Ver estado de la red:

```bash
tailscale status
```

Ejemplo de salida:

```text
100.87.13.44    servidor-vps      usuario@   linux   active
100.101.25.8    laptop-luis       usuario@   linux   active
100.92.10.20    iphone-luis       usuario@   ios     active
```

Ver la IP de Tailscale del equipo actual:

```bash
tailscale ip -4
```

Probar conectividad contra otro equipo:

```bash
ping 100.87.13.44
```

---

## 8. Conexión por SSH usando Tailscale

Conectarse por IP de Tailscale:

```bash
ssh usuario@100.87.13.44
```

Conectarse por MagicDNS:

```bash
ssh usuario@servidor-vps
```

Ejemplo real:

```bash
ssh gelois@100.87.13.44
```

> [!TIP]
> Si SSH funciona por Tailscale, no necesitas abrir el puerto 22 al Internet público.

---

## 9. MagicDNS para conectarse por nombre

MagicDNS evita depender de IPs.

Ejemplo:

```bash
ssh usuario@mi-servidor
```

Recomendaciones:

- Usa nombres claros para cada equipo.
- Evita nombres genéricos como `server`, `test` o `desktop`.
- Usa nombres como `vps-cuenca`, `nas-casa`, `laptop-trabajo`.

---

## 10. Hardening SSH recomendado

### 10.1 Crear una llave SSH

En tu PC cliente:

```bash
ssh-keygen -t ed25519 -C "usuario@equipo"
```

Ruta recomendada:

```text
~/.ssh/id_ed25519
```

---

### 10.2 Copiar la llave al servidor

Usando la IP de Tailscale:

```bash
ssh-copy-id usuario@100.87.13.44
```

O usando MagicDNS:

```bash
ssh-copy-id usuario@servidor-vps
```

---

### 10.3 Probar acceso por llave

Antes de desactivar contraseñas, prueba una nueva sesión:

```bash
ssh usuario@100.87.13.44
```

> [!IMPORTANT]
> No cierres tu sesión SSH actual hasta confirmar que puedes entrar desde otra terminal usando la llave.

---

### 10.4 Desactivar acceso por contraseña

Edita la configuración de SSH:

```bash
sudo nano /etc/ssh/sshd_config
```

Asegura estos valores:

```sshconfig
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
```

Recomendación adicional:

```sshconfig
X11Forwarding no
```

---

### 10.5 Validar y reiniciar SSH

Valida la configuración:

```bash
sudo sshd -t
```

Si no hay errores, reinicia SSH.

En Debian/Ubuntu:

```bash
sudo systemctl restart ssh
```

En RHEL, Fedora, Arch u otras distribuciones:

```bash
sudo systemctl restart sshd
```

Comprueba el estado:

```bash
systemctl status ssh
```

O:

```bash
systemctl status sshd
```

---

## 11. Restringir SSH a Tailscale con firewall

Si usas UFW, puedes permitir SSH solo por la interfaz `tailscale0`.

Permitir SSH desde Tailscale:

```bash
sudo ufw allow in on tailscale0 to any port 22 proto tcp
```

Bloquear SSH público:

```bash
sudo ufw deny 22/tcp
```

Activar UFW si todavía no está activo:

```bash
sudo ufw enable
```

Ver reglas:

```bash
sudo ufw status numbered
```

> [!WARNING]
> Antes de activar o modificar firewall en un servidor remoto, mantén una sesión SSH abierta y prueba una segunda conexión por Tailscale.

---

## 12. Fail2ban para protección adicional

Fail2ban detecta intentos fallidos repetidos y puede bloquear IPs automáticamente.

> [!NOTE]
> Fail2ban es una capa adicional. No reemplaza la autenticación por llave ni desactivar login por contraseña.

Instalar en Debian/Ubuntu:

```bash
sudo apt update
sudo apt install -y fail2ban
```

Crear configuración local para SSH:

```bash
sudo nano /etc/fail2ban/jail.d/sshd.local
```

Contenido recomendado:

```ini
[sshd]
enabled = true
backend = systemd
port = ssh
filter = sshd
maxretry = 5
findtime = 10m
bantime = 1h
```

Reiniciar servicio:

```bash
sudo systemctl restart fail2ban
```

Ver estado general:

```bash
sudo fail2ban-client status
```

Ver estado de SSH:

```bash
sudo fail2ban-client status sshd
```

---

## 13. Access Controls: ACLs y grants

Los controles de acceso de Tailscale definen qué usuarios, grupos o dispositivos pueden comunicarse entre sí.

Usos comunes:

- Permitir que solo ciertos usuarios entren a servidores.
- Separar equipos personales y servidores.
- Limitar acceso por puertos.
- Controlar acceso SSH si usas Tailscale SSH.

Ejemplo conceptual:

```json
{
  "grants": [
    {
      "src": ["group:admins"],
      "dst": ["tag:servers"],
      "ip": ["tcp:22"]
    }
  ]
}
```

> [!NOTE]
> Las rutas definen por dónde viaja el tráfico. Los access controls definen si ese tráfico está permitido.

---

## 14. Subnet Router

Un Subnet Router permite entrar a una red LAN completa desde Tailscale.

Ejemplo: acceder a `192.168.1.0/24` desde fuera.

Habilitar forwarding en Linux:

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

Anunciar una ruta:

```bash
sudo tailscale set --advertise-routes=192.168.1.0/24
```

Luego aprueba la ruta desde el panel de administración de Tailscale.

Verifica rutas:

```bash
tailscale status
```

---

## 15. Exit Node

Un Exit Node enruta todo el tráfico de Internet de un cliente a través de otro dispositivo del tailnet.

Habilitar forwarding en Linux:

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

Anunciar el equipo como Exit Node:

```bash
sudo tailscale set --advertise-exit-node
```

Luego apruébalo desde el panel de administración.

Usar un Exit Node desde Linux:

```bash
sudo tailscale set --exit-node=nombre-del-exit-node
```

Dejar de usar Exit Node:

```bash
sudo tailscale set --exit-node=
```

> [!NOTE]
> Cada cliente decide si usa o no un Exit Node. No se aplica automáticamente a todos los dispositivos.

---

## 16. Port knocking con `knockd`

Port knocking permite mantener un puerto cerrado hasta que el cliente envía una secuencia específica de conexión a puertos definidos.

Ejemplo conceptual:

```text
Cliente golpea: 7000 → 8000 → 9000
Servidor abre temporalmente SSH para esa IP
Cliente conecta por SSH
Servidor vuelve a cerrar el acceso
```

### Cuándo tiene sentido

- Cuando SSH está expuesto a Internet y quieres una capa adicional.
- Cuando necesitas ocultar un puerto ante escaneos básicos.
- Cuando tienes una IP dinámica y no puedes limitar por IP fija.

### Cuándo no es necesario

- Si SSH solo escucha o solo está permitido por Tailscale.
- Si el puerto 22 no está expuesto públicamente.
- Si ya administras acceso con firewall, llaves y Tailscale.

Instalar en Debian/Ubuntu:

```bash
sudo apt update
sudo apt install -y knockd
```

Activar `knockd`:

```bash
sudo nano /etc/default/knockd
```

Configurar:

```ini
START_KNOCKD=1
KNOCKD_OPTS="-i eth0"
```

> [!WARNING]
> Cambia `eth0` por la interfaz pública real del servidor. Puedes verla con `ip addr`.

Editar configuración:

```bash
sudo nano /etc/knockd.conf
```

Ejemplo básico con UFW:

```ini
[options]
UseSyslog

[openSSH]
sequence = 7000,8000,9000
seq_timeout = 15
tcpflags = syn
command = /usr/sbin/ufw allow from %IP% to any port 22 proto tcp
cmd_timeout = 30
stop_command = /usr/sbin/ufw delete allow from %IP% to any port 22 proto tcp
```

Reiniciar:

```bash
sudo systemctl restart knockd
```

Desde el cliente:

```bash
knock IP_DEL_SERVIDOR 7000 8000 9000
ssh usuario@IP_DEL_SERVIDOR
```

> [!CAUTION]
> Port knocking puede bloquearte si configuras mal la interfaz, la secuencia, el firewall o el tiempo de apertura. Pruébalo primero en un entorno controlado.

---

## 17. Buenas prácticas operativas

- Mantén Tailscale actualizado.
- Usa usuarios no-root para administración.
- Usa llaves SSH Ed25519.
- Desactiva password login.
- Desactiva root login por SSH.
- No expongas SSH a Internet si puedes usar Tailscale.
- Restringe servicios internos por firewall.
- Usa MagicDNS para evitar errores con IPs.
- Revisa periódicamente dispositivos autorizados en el panel.
- Revoca dispositivos que ya no uses.
- Usa etiquetas y access controls si manejas varios servidores.
- Documenta cambios de firewall y SSH antes de aplicarlos.

---

## 18. Troubleshooting

### No aparece el dispositivo en Tailscale

Verifica estado:

```bash
tailscale status
```

Reautentica:

```bash
sudo tailscale up --force-reauth
```

---

### No sé cuál es mi IP de Tailscale

```bash
tailscale ip -4
```

---

### No puedo entrar por SSH

Comprueba conectividad:

```bash
ping 100.x.y.z
```

Prueba SSH con modo verbose:

```bash
ssh -vvv usuario@100.x.y.z
```

Revisa SSH en el servidor:

```bash
sudo systemctl status ssh
```

O:

```bash
sudo systemctl status sshd
```

---

### Cambié SSH y ahora no entra

Si todavía tienes una sesión abierta:

```bash
sudo sshd -t
```

Revisa logs:

```bash
sudo journalctl -u ssh -n 100 --no-pager
```

O:

```bash
sudo journalctl -u sshd -n 100 --no-pager
```

---

### Fail2ban no detecta SSH

Verifica el jail:

```bash
sudo fail2ban-client status sshd
```

Verifica logs:

```bash
sudo journalctl -u fail2ban -n 100 --no-pager
```

---

## 19. Checklist final

- [ ] Tailscale instalado en todos los dispositivos.
- [ ] Todos los dispositivos autenticados en el mismo tailnet.
- [ ] `tailscale status` muestra los equipos activos.
- [ ] `tailscale ip -4` devuelve una IP `100.x.y.z`.
- [ ] MagicDNS activado.
- [ ] SSH probado por IP de Tailscale.
- [ ] SSH probado por MagicDNS.
- [ ] Llave SSH Ed25519 creada.
- [ ] Llave copiada al servidor.
- [ ] Acceso por llave probado en una segunda terminal.
- [ ] `PasswordAuthentication no` configurado.
- [ ] `KbdInteractiveAuthentication no` configurado.
- [ ] `PermitRootLogin no` configurado.
- [ ] Configuración validada con `sudo sshd -t`.
- [ ] SSH reiniciado correctamente.
- [ ] Firewall permite SSH por `tailscale0`.
- [ ] SSH público bloqueado si no es necesario.
- [ ] Fail2ban instalado y activo.
- [ ] Jail `sshd` funcionando.
- [ ] Subnet Router configurado solo si se necesita acceder a una LAN.
- [ ] Exit Node configurado solo si se necesita enrutar Internet.
- [ ] Port knocking configurado solo si aplica.
- [ ] Dispositivos antiguos o no usados revocados del tailnet.

---

## 20. Referencias oficiales

- [Tailscale: instalación en Linux](https://tailscale.com/docs/install/linux)
- [Tailscale: MagicDNS](https://tailscale.com/docs/features/magicdns)
- [Tailscale: Tailscale SSH](https://tailscale.com/docs/features/tailscale-ssh)
- [Tailscale: Subnet routers](https://tailscale.com/docs/subnets)
- [Tailscale: Exit nodes](https://tailscale.com/docs/features/exit-nodes)
- [Tailscale: Access controls](https://tailscale.com/kb/1018/acls)
- [OpenSSH: sshd_config](https://man.openbsd.org/sshd_config)
- [Fail2ban: proyecto oficial](https://github.com/fail2ban/fail2ban)
- [Ubuntu manpage: knockd](https://manpages.ubuntu.com/manpages/focal/man1/knockd.1.html)

---

## Nota editorial para GitHub y HackMD

Esta guía usa títulos jerárquicos, bloques de código, advertencias y checklist en formato Markdown estándar.

Para HackMD puedes añadir al inicio:

```markdown
[TOC]
```

Para GitHub es preferible mantener el índice manual incluido en este documento.
