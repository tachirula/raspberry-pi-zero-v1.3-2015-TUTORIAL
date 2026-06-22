# raspberry-pi-zero-v1.3-2015-TUTORIAL
tutorial para hacerme acordar de como manejar una raspberry pi zero v1.3 del 2015 que no cuenta con conexion wifi ni bluetooth

# TUTORIAL
¡Excelente idea! Documentar el proceso completo es fundamental. Aquí tienes una guía **limpia y paso a paso** (omitiendo los errores) para conectar una **Raspberry Pi Zero v1.3 (2015) sin WiFi** a Internet a través del puerto USB J10, usando el **modo gadget Ethernet** y compartiendo la conexión de tu PC con Arch Linux.

---

## Guía definitiva: Raspberry Pi Zero v1.3 → Internet por USB (gadget Ethernet)

### Objetivo
- Conectar la Pi Zero al PC por USB (J10) y luego PWR IN (J1).
- Asignar IP fija a la Pi (`169.254.0.2`) y al PC (`169.254.0.1`).
- Habilitar SSH para acceder sin monitor ni teclado.
- Compartir la conexión a Internet del PC (WiFi) con la Pi.

---

## Requisitos previos
- PC con **Arch Linux** (o cualquier distribución Linux).
- 1 MicroSD de 16GB (mucha memoria puede generar problemas con la raspberry
evitar MicroSD que tengan mucha memoria)
- 2 Cable micro‑USB Essager cable **de datos de carga**
el cable para Xiaomi Realme Huawei OPPO Samsung Micro usb a USB-A funciona perfectamente.

---

##  Preparar la tarjeta microSD

### a) Descargar la imagen correcta (Bullseye, estable para Zero v1.3)
```bash
cd ~/Downloads
wget https://downloads.raspberrypi.org/raspios_lite_armhf/images/raspios_lite_armhf-2023-02-22/2023-02-21-raspios-bullseye-armhf-lite.img.xz
unxz 2023-02-21-raspios-bullseye-armhf-lite.img.xz
```

### b) Flashear la imagen en la SD (ajusta `/dev/sda` si es necesario)
```bash
sudo dd if=2023-02-21-raspios-bullseye-armhf-lite.img of=/dev/sda bs=4M status=progress conv=fsync
```

---

## Configurar el modo gadget USB y SSH

### a) Montar la partición boot
```bash
sudo mount /dev/sda1 /mnt/boot
```

### b) Activar el gadget Ethernet y serie en `config.txt`
```bash
sudo sed -i '1i dtoverlay=dwc2,dr_mode=peripheral' /mnt/boot/config.txt
```

### c) Cargar los módulos en `cmdline.txt`
```bash
sudo sed -i 's/rootwait/rootwait modules-load=dwc2,g_ether,g_serial/' /mnt/boot/cmdline.txt
```

### d) Habilitar SSH
```bash
sudo touch /mnt/boot/ssh
```

### e) (Opcional) Crear usuario `pi` con contraseña personalizada
```bash
# Genera el hash de tu contraseña (ej. "user-normal123")
openssl passwd -6
# Copia el hash y crea userconf.txt
echo "pi:TU_HASH_AQUI" | sudo tee /mnt/boot/userconf.txt
```

### f) Desmontar
```bash
sudo umount /mnt/boot
```

---

## Asignar IP fija a la Pi (en la partición raíz)

### a) Montar la partición raíz
```bash
sudo mount /dev/sda2 /mnt/root
```

### b) Configurar IP estática en `dhcpcd.conf`
```bash
echo "interface usb0
static ip_address=169.254.0.2/24
static routers=169.254.0.1
static domain_name_servers=8.8.8.8 1.1.1.1" | sudo tee -a /mnt/root/etc/dhcpcd.conf
```

### c) (Opcional) Usar `systemd-networkd` como alternativa robusta
```bash
sudo mkdir -p /mnt/root/etc/systemd/network
sudo tee /mnt/root/etc/systemd/network/10-usb0.network <<EOF
[Match]
Name=usb0

[Network]
Address=169.254.0.2/24
Gateway=169.254.0.1
EOF

sudo ln -sf /lib/systemd/system/systemd-networkd.service /mnt/root/etc/systemd/system/multi-user.target.wants/systemd-networkd.service
```

### d) Habilitar SSH y forzar IP al arranque (script `rc.local`)
```bash
sudo tee /mnt/root/etc/rc.local <<EOF
#!/bin/sh -e
ifconfig usb0 169.254.0.2 netmask 255.255.255.0 up
systemctl restart ssh
exit 0
EOF
sudo chmod +x /mnt/root/etc/rc.local
sudo ln -sf /lib/systemd/system/rc-local.service /mnt/root/etc/systemd/system/multi-user.target.wants/rc-local.service
```

### e) Desmontar y expulsar
```bash
sudo umount /mnt/root
sudo eject /dev/sda
```

---

## Encender la Pi y conectar por USB

1. Inserta la SD en la Pi.
2. Conecta el cable USB **de datos** al puerto **OTG** (J10) y al PC.
3. Alimenta la Pi desde el **PWR IN** con un cargador externo (2.5 A).
4. Espera ~2 minutos (el LED verde parpadea al arrancar).

---

## Configurar la IP en el PC (Arch Linux)

Cada vez que conectes la Pi, la interfaz USB aparecerá con un nombre como `enp0s20f0u3` o `enp0s20f0u5`. Para asignar la IP fija:

```bash
# Identifica la interfaz (busca la que empieza por enp0s20f0u)
ip a

# Asigna IP fija
sudo ip addr flush dev enp0s20f0u3   # usa el nombre correcto
sudo ip addr add 169.254.0.1/24 dev enp0s20f0u3
sudo ip link set enp0s20f0u3 up
```

---

## Probar conectividad y SSH

```bash
ping -c 4 169.254.0.2
ssh pi@169.254.0.2
```

Contraseña: la que configuraste (ej. `user-normal123`).

---

##  Dar salida a Internet a la Pi (NAT y enrutamiento)

La Pi tiene IP `169.254.0.2` pero no puede salir a Internet. Compartimos la conexión WiFi del PC:

### a) Activar forwarding de IP en el PC
```bash
sudo sysctl -w net.ipv4.ip_forward=1
# Para hacerlo permanente:
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.d/99-ipforward.conf
```

### b) Configurar NAT con `iptables`
```bash
sudo iptables -t nat -A POSTROUTING -o wlp0s20f3 -j MASQUERADE   # wlp0s20f3 = tu interfaz WiFi
sudo iptables -A FORWARD -i enp0s20f0u3 -o wlp0s20f3 -j ACCEPT
sudo iptables -A FORWARD -i wlp0s20f3 -o enp0s20f0u3 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

### c) Guardar las reglas (opcional)
```bash
sudo iptables-save | sudo tee /etc/iptables/iptables.rules
```

---

## Configurar la Pi para usar el PC como gateway

### Desde la Pi (por SSH):
```bash
sudo ip route add default via 169.254.0.1
```

### (Opcional) Hacerlo permanente:
```bash
echo "ip route add default via 169.254.0.1" | sudo tee -a /etc/rc.local
```

### Verificar DNS
```bash
cat /etc/resolv.conf   # Debe tener nameservers (8.8.8.8, etc.)
```

---

## ¡Internet funciona en la Pi!

Prueba:

```bash
ping -c 4 8.8.8.8
ping -c 4 google.com
sudo apt update
```

---

## Conexiones futuras

Cada vez que conectes la Pi por USB:

1. En el PC:
   ```bash
   sudo ip addr flush dev enp0s20f0u3
   sudo ip addr add 169.254.0.1/24 dev enp0s20f0u3
   sudo ip link set enp0s20f0u3 up
   ```
2. SSH:
   ```bash
   ssh pi@169.254.0.2
   ```
3. En la Pi, si la ruta por defecto no persiste:
   ```bash
   sudo ip route add default via 169.254.0.1
   ```

---

## Resumen de archivos clave

| Archivo en la Pi | Configuración |
|------------------|---------------|
| `/boot/config.txt` | `dtoverlay=dwc2,dr_mode=peripheral` |
| `/boot/cmdline.txt` | `... rootwait modules-load=dwc2,g_ether,g_serial` |
| `/boot/ssh` | (archivo vacío) |
| `/boot/userconf.txt` | `pi:$6$...` (hash de contraseña) |
| `/etc/dhcpcd.conf` | IP estática para `usb0` |
| `/etc/rc.local` | Asigna IP y reinicia SSH al arrancar |
| `/etc/systemd/network/10-usb0.network` | (alternativa) configuración de IP |

---

## Notas finales

- Esta configuración es **estable** y funciona con la imagen **Bullseye**.
- La Pi Zero v1.3 **no tiene WiFi**, pero con este método obtiene Internet a través del USB.
- Puedes instalar `fastfetch`, `tmux`, etc., usando `sudo apt install`.
- Si el PC se reinicia, las reglas `iptables` se pierden; vuelve a ejecutar el paso 7 o guarda las reglas.
- Algunas terminales pueden tener problemas, PI no las puede reconocer (como por ejemplo Kitty)
es recomendable ejecutar `export TERM=xterm-256color`, solo si se presenta problemas 

---

# Final

Si todo sale bien tendras una Raspberry Zero V1.3 con conectividad a internet mediante cables micro-USB a USB-A que funcionan como Ethernet y Cargador.
