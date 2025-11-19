
# Guía Completa: Crear, Configurar y Administrar una VM en Oracle Cloud (Always Free)

Guía en formato Markdown para:

- Crear infraestructura de red en Oracle Cloud (OCI).
- Crear una VM Always Free con Ubuntu.
- Conectarse vía SSH desde Windows.
- Configurar Apache + DNS + SSL (Let’s Encrypt).
- Usar herramientas de red (tcpdump, tc).
- Mejorar seguridad básica.
- Entender qué imágenes se pueden usar en Always Free.

---

## 1. Arquitectura general por capas

### Capa 1: Infraestructura en OCI
- VCN (Virtual Cloud Network)
- Subnet pública
- Subnet privada (opcional)
- Internet Gateway
- NAT Gateway (para subnets privadas)
- Route Table
- Security List o Network Security Groups (NSG)
- VNIC (interfaz de red virtual de la VM)
- IP pública y privada

### Capa 2: Sistema operativo
- Ubuntu Server (o Ubuntu Minimal)
- Servicio SSH
- Firewall interno (ufw opcional)
- Paquetes y herramientas base

### Capa 3: Servicios
- Apache HTTP Server
- Certbot (Let’s Encrypt) para SSL
- Herramientas de red (tcpdump, iproute2/tc)
- Docker (opcional para backend)

### Capa 4: DNS y capa externa
- Cloudflare (u otro DNS)
- Registros A / CNAME
- Configuración SSL (Full / Full strict)
- Caché, propagación DNS

---

## 2. Crear la red (VCN) correctamente

1. Ir en la consola OCI a:
   Networking → Virtual Cloud Networks → Create VCN

2. Elegir la opción:
   VCN with Internet Connectivity

Esto crea automáticamente:
- 1 VCN
- 1 Subnet pública
- 1 Subnet privada
- Internet Gateway (IGW)
- NAT Gateway
- Route Table
- Security List por defecto

### 2.1 Verificaciones mínimas

En la Route Table asociada a la Subnet pública, debe existir una regla:

- Destination CIDR: 0.0.0.0/0  
- Target: Internet Gateway

En la Security List asociada a la Subnet pública, debe permitirse (ingress):

- TCP 22 (SSH)
- TCP 80 (HTTP)
- TCP 443 (HTTPS)

Si no están, agregarlos.

---

## 3. Crear la VM Always Free

Ir a:
Compute → Instances → Create instance

### 3.1 Elegir la imagen (Image)

Depende del shape (tipo de CPU):

#### Para x86 (shape VM.Standard.E2.1.Micro)
Imágenes típicas Always Free:

- Canonical Ubuntu 22.04
- Canonical Ubuntu 22.04 Minimal
- Canonical Ubuntu 24.04
- Canonical Ubuntu 24.04 Minimal
- Oracle Linux 8 / 9

Recomendado para empezar:
- Canonical Ubuntu 22.04 (o la Minimal si querés algo más liviano).

#### Para ARM (shape VM.Standard.A1.Flex)
Solo imágenes aarch64:

- Canonical Ubuntu 22.04 Minimal aarch64
- Canonical Ubuntu 24.04 Minimal aarch64
- Oracle Linux aarch64

No hay instalación “desde ISO”: siempre se parte de una imagen cloud.

### 3.2 Elegir el shape (Always Free)

Opciones más usadas:

- VM.Standard.E2.1.Micro (x86, Always Free)
  - 1 OCPU compartido
  - 1 GB RAM

- VM.Standard.A1.Flex (ARM, Always Free si hay capacidad)
  - Hasta 4 OCPUs y 24 GB RAM dentro del Free Tier
  - A veces saturado en ciertas regiones

Para evitar errores de capacidad, E2.1.Micro suele ser más estable.

### 3.3 Configuración de red de la VM

En la sección Networking al crear la instancia:

- Virtual cloud network: elegir la VCN creada.
- Subnet: elegir la public subnet.
- Assign public IPv4 address: habilitado (Yes / Automatically assign).

De esta forma la VM obtiene:
- Una IP privada dentro de la VCN.
- Una IP pública accesible desde Internet.

### 3.4 SSH Keys

En la parte de SSH keys:

- Seleccionar “Generate SSH key pair”.
- Descargar:
  - Archivo .key (clave privada).
  - Archivo .pub (clave pública).

Guardar la clave privada en tu PC (no compartirla).

Crear la instancia (Create).

---

## 4. Reservar la IP pública

Si no reservás la IP, puede cambiar al reiniciar la VM.

1. Ir a:
   Networking → IP Management → Public IPs

2. Buscar la IP asignada a tu VM.

3. Seleccionarla y hacer:
   Reserve

Queda marcada como:
- Reserved public IP
- Asociada a la VNIC de tu instancia

Usá esa IP fija en tu DNS (Cloudflare u otro).

---

## 5. Conectarse por SSH desde Windows

Supongamos que descargaste la clave privada como:
- C:\Users\TuUsuario\Desktop\oracle.key

### 5.1 Ajustar permisos (opcional pero recomendable)

Abrir PowerShell:

    cd C:\Users\TuUsuario\Desktop
    icacls oracle.key /inheritance:r /grant:r "$env:USERNAME:R"

### 5.2 Conexión SSH

Si tu usuario en la VM es ubuntu (caso Ubuntu):

    ssh -i .\oracle.key ubuntu@IP_PUBLICA

Aceptar el fingerprint la primera vez (escribir “yes” cuando lo pida).

Si la conexión falla:
- Verificar que la VM esté en estado Running.
- Confirmar que la IP es la pública reservada.
- Verificar en Security List que el puerto 22 está permitido.

---

## 6. Primeros pasos dentro de Ubuntu

Actualizar paquetes:

    sudo apt update && sudo apt upgrade -y

Ver versión de Ubuntu:

    lsb_release -a

Ver interfaces de red:

    ip a

Normalmente la interfaz principal tendrá nombre tipo:
- ens3
- enp0s3
- eth0 (depende de la imagen)

---

## 7. Instalar y configurar Apache

Instalar Apache:

    sudo apt install apache2 -y

Habilitar módulos recomendados:

    sudo a2enmod ssl headers rewrite
    sudo systemctl restart apache2

Ver estado del servicio:

    sudo systemctl status apache2

Probar desde tu navegador en tu PC:

- http://IP_PUBLICA

Deberías ver la página por defecto de Apache en Ubuntu.

---

## 8. Configurar DNS en Cloudflare

Suponiendo que tenés un dominio propio.

1. Apuntar un registro A:

   - Nombre: api.tu-dominio.com
   - Tipo: A
   - Contenido: IP Pública reservada en Oracle
   - Proxy: ON (naranja) o OFF (gris), según tu caso.

2. Ajustar SSL/TLS:

   - Recomendado: Full (strict)
   - Debes tener certificado válido en tu servidor (vía Certbot).

3. Mientras emitís certificados con Certbot, podés activar:
   - Development Mode para evitar caché.

---

## 9. Instalar Certbot y emitir certificados SSL

Instalar Certbot (para Apache):

    sudo apt install certbot python3-certbot-apache -y

Emitir certificado para tu dominio:

    sudo certbot --apache -d api.tu-dominio.com

Certbot hace automáticamente:

- Crea y configura un VirtualHost SSL.
- Crea el archivo 000-default-le-ssl.conf.
- (Opcional) Activa redirección HTTP → HTTPS.
- Configura renovación automática con systemd/cron.

Si usás Cloudflare en modo proxy, asegurate de usar SSL Full (strict).

---

## 10. Crear un archivo grande para pruebas de red

Para generar tráfico HTTP utilizable en herramientas como Wireshark o tcpdump:

    sudo truncate -s 20M /var/www/html/archivo.bin
    ls -lh /var/www/html/

Desde tu navegador:

- http://IP_PUBLICA/archivo.bin

Esto genera una descarga grande para capturar tráfico.

---

## 11. Herramientas de red: tcpdump y tc

Instalar herramientas:

    sudo apt install tcpdump iproute2 -y

### 11.1 Capturar tráfico con tcpdump

Identificar interfaz principal (ej. ens3), luego:

    sudo tcpdump -i ens3 port 80 -w captura-http.pcap

Detener con Ctrl + C.

Copiar archivo .pcap a tu PC (PowerShell):

    scp -i .\oracle.key ubuntu@IP_PUBLICA:/home/ubuntu/captura-http.pcap .

Abrir captura-http.pcap con Wireshark en tu computadora.

### 11.2 Simular problemas de red con tc

Ejemplo: agregar 200 ms de latencia a toda la salida de ens3:

    sudo tc qdisc add dev ens3 root netem delay 200ms

Ver configuración:

    sudo tc qdisc show dev ens3

Agregar pérdida de paquetes (ej. 5%):

    sudo tc qdisc change dev ens3 root netem loss 5%

Revertir y dejar la interfaz normal:

    sudo tc qdisc del dev ens3 root

---

## 12. Seguridad recomendada

### 12.1 Deshabilitar autenticación por contraseña en SSH

Editar configuración de SSH:

    sudo nano /etc/ssh/sshd_config

Buscar o agregar:

    PasswordAuthentication no

Guardar, cerrar y reiniciar SSH:

    sudo systemctl restart ssh

Asegurate de que tu clave SSH funcione antes de desactivar PasswordAuthentication.

### 12.2 Configurar UFW (opcional)

Si decidís usar firewall interno:

    sudo ufw allow 22
    sudo ufw allow 80
    sudo ufw allow 443
    sudo ufw enable

Ver estado:

    sudo ufw status verbose

### 12.3 Instalar Fail2ban

    sudo apt install fail2ban -y

Fail2ban ayuda a bloquear intentos de fuerza bruta en servicios como SSH.

---

## 13. Logs y debugging

### 13.1 Logs de Apache

Ver últimos logs del servicio:

    sudo journalctl -u apache2 -n 100 --no-pager

Error log:

    sudo tail -f /var/log/apache2/error.log

Access log:

    sudo tail -f /var/log/apache2/access.log

### 13.2 Ver puertos abiertos

    sudo lsof -i -P -n | grep LISTEN

### 13.3 Probar HTTP/HTTPS localmente

Dentro de la VM:

    curl -I http://127.0.0.1
    curl -I https://api.tu-dominio.com --resolve api.tu-dominio.com:443:IP_PUBLICA

Desde tu PC:

    curl -I https://api.tu-dominio.com

---

## 14. Imágenes compatibles con Always Free

### 14.1 Para x86 (VM.Standard.E2.1.Micro)

Típicamente disponibles:

- Canonical Ubuntu 22.04
- Canonical Ubuntu 22.04 Minimal
- Canonical Ubuntu 24.04
- Canonical Ubuntu 24.04 Minimal
- Oracle Linux 8 / 9

Diferencias clave:

- Imágenes “normales” (sin la palabra Minimal) → más paquetes preinstalados.
- Imágenes Minimal → solo lo esencial, menos consumo y arranque más rápido.

### 14.2 Para ARM (VM.Standard.A1.Flex)

Imágenes aarch64:

- Canonical Ubuntu 22.04 Minimal aarch64
- Canonical Ubuntu 24.04 Minimal aarch64
- Oracle Linux aarch64

Puntos importantes:

- No hay “Ubuntu Server” ARM como ISO instalable; todas son cloud images.
- No se puede “crear VM sin imagen” ni bootear desde ISO en OCI.
- No existe una conversión real de “Ubuntu Minimal” a “Ubuntu Server completo”.
  Se pueden instalar paquetes extra, pero la base sigue siendo Minimal.

Ejemplo para agregar algunos metapaquetes en Ubuntu Minimal (no es conversión 1:1):

    sudo apt install ubuntu-standard ubuntu-server-minimal

---

## 15. Snapshots y backups

Para proteger tu entorno antes de cambios grandes:

1. Ir a:
   Block Storage → Boot Volumes

2. Seleccionar el boot volume asociado a tu instancia.

3. Crear Snapshot:
   Create Snapshot

Luego podés crear una nueva VM desde ese snapshot si algo sale mal.

---

## 16. Buenas prácticas generales

- Usar siempre:
  - Subnet pública para servidores accesibles desde Internet.
  - IP pública reservada para no romper DNS.
- Mantener el sistema actualizado:
  
      sudo apt update && sudo apt upgrade -y

- Monitorear logs de Apache y del sistema.
- Usar Cloudflare con SSL “Full (strict)” si el servidor tiene certificado válido.
- Desactivar servicios y módulos Apache que no se usen.
- Usar NSG en lugar de Security List si administrás varias VMs y querés reglas más ordenadas.
- Documentar:
  - IPs
  - Nombres de VCN, Subnets
  - Dominios y registros DNS
  - Comandos especiales de configuración (tc, firewalls, etc.)

---

## 17. Resumen rápido

1. Crear VCN con Internet Connectivity y subnet pública.
2. Crear VM Always Free (E2.1.Micro o A1.Flex) con Ubuntu.
3. Asignar y reservar IP pública.
4. Conectarse por SSH desde Windows usando la clave privada.
5. Actualizar Ubuntu e instalar Apache.
6. Configurar DNS en Cloudflare apuntando a la IP pública.
7. Instalar Certbot y emitir certificados SSL.
8. Crear archivo grande para pruebas de red.
9. Instalar tcpdump y tc para capturar tráfico y simular problemas de red.
10. Aplicar seguridad básica (SSH sin contraseña, UFW, Fail2ban).
11. Crear snapshots antes de cambios importantes.

Esta guía en Markdown se puede guardar directamente como README.md y versionar en tu repositorio o carpeta de documentación.
