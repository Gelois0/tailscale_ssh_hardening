# Acceso remoto seguro con Tailscale y hardening SSH

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
  - [10.5 Respaldar, validar y aplicar cambios SSH](#105-respaldar-validar-y-aplicar-cambios-ssh)
- [11. Restringir SSH a Tailscale con firewall](#11-restringir-ssh-a-tailscale-con-firewall)
- [12. Fail2ban para protección adicional](#12-fail2ban-para-protección-adicional)
- [13. Access Controls: ACLs y grants](#13-access-controls-acls-y-grants)
- [14. Subnet Router](#14-subnet-router)
- [15. Exit Node](#15-exit-node)
- [16. Port knocking con `knockd`](#16-port-knocking-con-knockd)
- [17. Buenas prácticas operativas](#17-buenas-prácticas-operativas)
- [18. Troubleshooting](#18-troubleshooting)
- [19. Referencias oficiales](#19-referencias-oficiales)

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

Tailscale SSH y OpenSSH sobre Tailscale son modalidades distintas. Cuando está habilitado, Tailscale SSH atiende el tráfico al puerto 22 que llega a la dirección Tailscale del dispositivo en lugar del OpenSSH del sistema. La configuración de OpenSSH, como `authorized_keys` y `PasswordAuthentication`, y el jail de Fail2ban para OpenSSH no controlan automáticamente Tailscale SSH. La exposición pública de OpenSSH sigue siendo un asunto distinto que debes evaluar.

Tailscale SSH requiere tanto acceso de red como una regla `ssh` en la política del tailnet, incluidos los usuarios de destino permitidos. Revisa y prueba la política antes de habilitarlo.

Habilitar Tailscale SSH:

```bash
sudo tailscale set --ssh
```

Deshabilitarlo:

```bash
sudo tailscale set --ssh=false
```

Si prefieres OpenSSH tradicional con llaves, mantén Tailscale SSH deshabilitado y restringe OpenSSH mediante una política de firewall revisada. La sección 10 se aplica a OpenSSH, no automáticamente a Tailscale SSH.

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

> [!WARNING]
> Antes de forzar una reautenticación, confirma otra vía autenticada de administración o acceso independiente a la consola del proveedor. `--force-reauth` puede interrumpir el acceso si Tailscale es tu única ruta; no es un arreglo rutinario después de `tailscale status`.

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

Prueba una **conexión OpenSSH nueva** antes de desactivar el acceso por contraseña. Sustituye la dirección, el usuario y la ruta de llave del ejemplo. Desactiva las alternativas por contraseña e interacción y evita reutilizar conexiones multiplexadas; conserva la verificación de la identidad del servidor:

```bash
ssh -v -o ControlMaster=no -o ControlPath=none \
  -o PreferredAuthentications=publickey \
  -o PasswordAuthentication=no -o KbdInteractiveAuthentication=no \
  -o IdentitiesOnly=yes -i ~/.ssh/id_ed25519 usuario@100.87.13.44
```

Confirma que la salida detallada indica autenticación exitosa con `publickey`. La frase de paso de la llave privada no es la contraseña de la cuenta del servidor. Para autenticación desatendida, añade `-o BatchMode=yes` con una llave o un agente ya utilizable; una llave cifrada sin agente puede fallar en modo batch.

> [!IMPORTANT]
> Conserva la sesión actual hasta que funcione una conexión nueva. Una conexión atendida por Tailscale SSH no demuestra que funcione la autenticación por llave de OpenSSH.

---

### 10.4 Desactivar acceso por contraseña

Antes de editar, cumple los requisitos de respaldo y recuperación de la sección 10.5. Inspecciona `/etc/ssh/sshd_config` y su orden real de `Include`, incluidos los archivos de `/etc/ssh/sshd_config.d/`. OpenSSH normalmente utiliza el primer valor obtenido para cada opción: un archivo con nombre tardío no necesariamente sobrescribe un valor anterior. Considera los bloques `Match` aplicables; no añadas valores duplicados a ciegas.

Edita únicamente los archivos que controlan la conexión prevista:

```bash
sudo nano /etc/ssh/sshd_config
```

Valores previstos para OpenSSH solo con llaves:

```sshconfig
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
```

Opcional, si no necesitas forwarding X11:

```sshconfig
X11Forwarding no
```

Comprueba sintaxis y valores efectivos:

```bash
sudo sshd -t
sudo sshd -T | grep -E '^(passwordauthentication|kbdinteractiveauthentication|permitrootlogin|pubkeyauthentication) '
```

Para reglas `Match`, usa `sshd -T -C` con el usuario, la dirección de origen y el host reales de la conexión que pruebas. Las comprobaciones de sintaxis y configuración efectiva no demuestran que funcione el acceso remoto; prueba una conexión nueva después de aplicar el cambio.

---

### 10.5 Respaldar, validar y aplicar cambios SSH

1. Confirma una vía independiente de recuperación, como la consola del proveedor, y conserva la sesión SSH actual. No continúes si la ruta SSH que modificarás es la única vía de recuperación.
2. Antes de editar, respalda cada archivo que modificarás, incluidos los archivos de configuración incluidos, y registra las rutas exactas del original y su respaldo. Verifica que los respaldos sean legibles e iguales a los originales. Documenta cómo restaurarlos desde la consola independiente.
3. Verifica una conexión OpenSSH nueva solo con llave según la sección 10.3. Después de editar, valida la sintaxis con `sudo sshd -t` e inspecciona los valores efectivos según la sección 10.4. Detente ante errores o valores inesperados.
4. Aplica los cambios con el nombre real del servicio y el procedimiento de recarga de la distribución. Debian/Ubuntu suelen usar `sudo systemctl reload ssh`; otras distribuciones pueden usar `sudo systemctl reload sshd`. Si la recarga no está disponible, planifica el reinicio y la recuperación antes de ejecutarlos. No ejecutes ambos comandos de servicio a ciegas.
5. Comprueba el estado del servicio real y abre una conexión SSH **nueva** para confirmar el método de autenticación previsto. Cierra la sesión anterior solo después de que funcione la nueva.

Si falla la conexión nueva, usa la sesión conservada o la consola independiente para restaurar únicamente los archivos modificados desde los respaldos registrados, ejecuta `sshd -t` y recarga/reinicia el servicio correcto. Prueba otra conexión nueva antes de cerrar la vía de recuperación. Una configuración válida o un servicio activo no demuestran por sí solos que puedas iniciar sesión remotamente.

---

## 11. Restringir SSH a Tailscale con firewall

Este es un ejemplo de planificación, no una secuencia universal. Identifica el firewall activo, inspecciona todas las reglas en orden y la configuración IPv6, y revisa el firewall del proveedor para ambas familias de direcciones.

UFW aplica el criterio de primera coincidencia dentro de sus reglas: un permiso amplio anterior puede dejar SSH público abierto aunque después añadas `deny 22/tcp`. También importan las reglas before/after de UFW y las cadenas netfilter de Tailscale; `ufw status` por sí solo no muestra toda la política efectiva.

Antes de cambiar un firewall remoto:

- Verifica la recuperación por consola independiente y conserva una sesión funcional.
- Establece una segunda conexión SSH funcional por Tailscale.
- Respalda y verifica la configuración exacta del firewall que modificarás; registra la restauración y si UFW estaba activo.
- Inspecciona `sudo ufw status numbered`, `sudo ufw show raw` y las reglas activas de iptables/nftables según corresponda.
- Revisa las reglas del proveedor, incluido IPv6. Elimina o restringe solo los permisos conflictivos identificados; no uses `ufw reset` como atajo.

Para un conjunto de reglas de usuario UFW revisado, el orden previsto es permitir por `tailscale0` y después denegar el resto del tráfico TCP/22 antes de los permisos de entrada más amplios. Adáptalo a las reglas existentes y evita duplicarlas. Solo como ejemplo:

```bash
sudo ufw insert 1 allow in on tailscale0 to any port 22 proto tcp
sudo ufw insert 2 deny 22/tcp
sudo ufw status numbered
```

Activa UFW solo después de revisar el impacto sobre todos los servicios. Si existe IPv6, verifica que el filtrado IPv6 esté habilitado y que las reglas efectivas lo cubran. Tailscale instala sus propias reglas netfilter, por lo que no debes asumir que UFW controla todos los paquetes del tailnet.

Desde fuera del tailnet, comprueba que TCP/22 público está bloqueado en cada dirección pública IPv4/IPv6. Por separado, prueba una conexión SSH **nueva** por Tailscale. Conserva la recuperación por consola del proveedor. Si falla alguna prueba, detente y restaura la configuración registrada por la vía de recuperación; no cierres la sesión funcional.

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
> Este es un fragmento conceptual de política. Define `group:admins` y los propietarios de `tag:servers`. Los grants son aditivos: los permisos de los grants coincidentes se combinan y un grant limitado no sobrescribe otro amplio existente. Revisa toda la política. Tailscale SSH requiere además una regla `ssh` separada. Prueba conexiones previstas como permitidas y denegadas.
>
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

> [!WARNING]
> Se omite el ejemplo genérico UFW/knockd: un permiso añadido al final puede quedar después de una denegación SSH, y eliminar un permiso coincidente puede afectar una regla preexistente. Una integración específica debe insertar una excepción temporal limitada antes de la denegación y retirar únicamente la regla exacta que creó. Verifica orden, interfaz, reglas duplicadas, IPv4/IPv6 y recuperación en un entorno controlado. Hasta entonces, mantén knockd deshabilitado; no uses los comandos de reinicio/cliente siguientes como una secuencia de despliegue.


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

> [!WARNING]
> Antes de forzar una reautenticación, confirma otra vía autenticada de administración o acceso independiente a la consola del proveedor. `--force-reauth` puede interrumpir el acceso si Tailscale es tu única ruta; no es un arreglo rutinario después de `tailscale status`.

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


---

## 19. Referencias oficiales

- [Tailscale: instalación en Linux](https://tailscale.com/docs/install/linux)
- [Tailscale: descargar e instalar](https://tailscale.com/docs/install)
- [Tailscale: MagicDNS](https://tailscale.com/docs/features/magicdns)
- [Tailscale: Tailscale SSH](https://tailscale.com/docs/features/tailscale-ssh)
- [Tailscale: Subnet routers](https://tailscale.com/docs/subnets)
- [Tailscale: Exit nodes](https://tailscale.com/docs/features/exit-nodes)
- [Tailscale: Access controls](https://tailscale.com/kb/1018/acls)
- [Tailscale: Grants](https://tailscale.com/docs/reference/syntax/grants)
- [Tailscale: Netfilter modes](https://tailscale.com/docs/reference/netfilter-modes)
- [OpenSSH: ssh_config](https://man.openbsd.org/ssh_config)
- [OpenSSH: sshd_config](https://man.openbsd.org/sshd_config)
- [OpenSSH: sshd](https://man.openbsd.org/sshd.8)
- [UFW: manual](https://manpages.ubuntu.com/manpages/noble/man8/ufw.8.html)
- [Fail2ban: proyecto oficial](https://github.com/fail2ban/fail2ban)
- [Ubuntu manpage: knockd](https://manpages.ubuntu.com/manpages/focal/man1/knockd.1.html)

---
