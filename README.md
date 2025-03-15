# Guía de Porting de Ubuntu Touch para Redmi Note 10S (MTK Helio G95)

## Requisitos previos

- Redmi Note 10S con bootloader desbloqueado
- Termux instalado en el dispositivo o un teléfono secundario
- Al menos 15GB de espacio libre
- Conexión a internet estable
- Conocimientos básicos de línea de comandos
- Paciencia (el proceso puede llevar horas)

## 1. Preparar el entorno en Termux

### 1.1 Instalar paquetes necesarios

```bash
# Actualizar repositorios
pkg update -y && pkg upgrade -y

# Instalar paquetes esenciales
pkg install -y git wget curl python python-pip openssh proot-distro android-tools

# Instalar Ubuntu en proot-distro
proot-distro install ubuntu
```

### 1.2 Configurar el entorno Ubuntu

```bash
# Entrar al entorno Ubuntu
proot-distro login ubuntu

# Actualizar e instalar dependencias
apt update && apt upgrade -y
apt install -y sudo git wget curl python3 python3-pip adb fastboot repo android-tools-adb android-tools-fastboot build-essential unzip zip tar gzip zstd rsync

# Configurar variables de entorno
echo 'export PATH=$PATH:/usr/bin:/usr/sbin' >> ~/.bashrc
source ~/.bashrc
```

## 2. Obtener el código fuente de Halium/UBports

Halium es la capa que permite a Ubuntu Touch ejecutarse en dispositivos Android.

```bash
# Crear directorio de trabajo
mkdir -p ~/ubports/halium
cd ~/ubports/halium

# Inicializar el repositorio
repo init -u https://github.com/Halium/android.git -b halium-9.0
repo sync -c -j$(nproc --all)
```

## 3. Obtener archivos específicos del dispositivo

Para el Redmi Note 10S (código "rosemary" para MTK Helio G95) necesitarás:

```bash
# Clonar repositorios específicos para MTK Helio G95
git clone https://github.com/LineageOS/android_device_xiaomi_rosemary.git device/xiaomi/rosemary
git clone https://github.com/LineageOS/android_device_xiaomi_mt6785-common.git device/xiaomi/mt6785-common
git clone https://github.com/LineageOS/android_kernel_xiaomi_mt6785.git kernel/xiaomi/mt6785
git clone https://github.com/LineageOS/android_vendor_xiaomi_rosemary.git vendor/xiaomi/rosemary
```

## 4. Preparar archivos de configuración Halium

### 4.1 Crear manifest para Halium

Crea un archivo `deviceinfo` para tu dispositivo:

```bash
mkdir -p ~/ubports/devices/halium/xiaomi/rosemary
cd ~/ubports/devices/halium/xiaomi/rosemary

cat > deviceinfo << EOF
deviceinfo_name="Xiaomi Redmi Note 10S"
deviceinfo_manufacturer="Xiaomi"
deviceinfo_codename="rosemary"
deviceinfo_year="2021"
deviceinfo_arch="arm64"
deviceinfo_kernel_source="kernel/xiaomi/mt6785"
deviceinfo_kernel_config="rosemary_defconfig"
deviceinfo_kernel_cmdline="bootopt=64S3,32N2,64N2"
deviceinfo_kernel_image="Image.gz-dtb"
deviceinfo_kernel_has_dtb="true"
deviceinfo_dtb="mediatek/mt6785"
deviceinfo_modules_initfs=""
deviceinfo_external_storage="true"
deviceinfo_screen_width="1080"
deviceinfo_screen_height="2400"
deviceinfo_dev_touchscreen="/dev/input/event2"
deviceinfo_dev_touchscreen_calibration=""
deviceinfo_dev_keyboard=""
EOF
```

### 4.2 Crear archivo de configuración Halium

```bash
cat > halium.mk << EOF
$(call inherit-product, vendor/halium/config/common.mk)
$(call inherit-product, device/xiaomi/rosemary/device.mk)

PRODUCT_NAME := halium_rosemary
PRODUCT_DEVICE := rosemary
PRODUCT_BRAND := Xiaomi
PRODUCT_MODEL := Redmi Note 10S
PRODUCT_MANUFACTURER := Xiaomi
EOF
```

## 5. Compilar Halium

```bash
cd ~/ubports/halium

# Preparar ambiente de compilación
source build/envsetup.sh
breakfast rosemary
export USE_CCACHE=1
export CCACHE_EXEC=/usr/bin/ccache

# Compilar
mka hybris-hal
```

## 6. Compilar rootfs de Ubuntu Touch

```bash
# Clonar herramientas de UBports
cd ~/ubports
git clone https://github.com/ubports/ubports-installer.git
cd ubports-installer

# Instalar dependencias
npm install

# Descargar rootfs para arm64
./bin/ubports-installer download \
    --device=generic_arm64 \
    --channels=16.04/stable \
    --path=~/ubports/rootfs
```

## 7. Instalar Ubuntu Touch en el dispositivo

### 7.1 Preparar el dispositivo

1. Reinicia el dispositivo en modo fastboot:
   ```bash
   adb reboot bootloader
   ```

2. Flashear Halium:
   ```bash
   fastboot flash boot ~/ubports/halium/out/target/product/rosemary/boot.img
   fastboot flash system ~/ubports/halium/out/target/product/rosemary/system.img
   ```

### 7.2 Instalar rootfs

```bash
# Montar el sistema
fastboot boot ~/ubports/halium/out/target/product/rosemary/boot.img
adb wait-for-device

# Desempaquetar rootfs
adb push ~/ubports/rootfs/ubuntu.tar.gz /data/
adb shell "mkdir -p /data/ubuntu-rootfs"
adb shell "tar xf /data/ubuntu.tar.gz -C /data/ubuntu-rootfs"

# Configurar chroot
adb shell "mkdir -p /data/ubuntu-rootfs/android"
adb shell "mkdir -p /data/ubuntu-rootfs/userdata"
```

### 7.3 Configurar Ubuntu Touch

```bash
# Entrar al chroot de Ubuntu
adb shell "chroot /data/ubuntu-rootfs /bin/bash"

# Configurar el sistema
cat > /etc/init/lxc-android-config.override << EOF
# Don't start session before unity-system-compositor is ready
pre-start script
    if [ "$UPSTART_EVENTS" = "started" ]; then
        wait-for-state ubuntu-touch-session
    fi
end script
EOF

# Configurar repositorios
echo "deb http://repo.ubports.com xenial main" > /etc/apt/sources.list.d/ubports.list
apt update && apt upgrade -y

# Instalar paquetes necesarios para Ubuntu Touch
apt install -y ubuntu-touch-session unity8 unity8-common ubuntu-system-settings
```

## 8. Finalizar la instalación

```bash
# Reiniciar el dispositivo
adb reboot
```

## 9. Depuración y resolución de problemas

Si enfrentas problemas, puedes verificar logs:

```bash
# Obtener logs del sistema
adb shell logcat

# Ver logs específicos de Ubuntu Touch
adb shell "chroot /data/ubuntu-rootfs journalctl -f"
```

### Problemas comunes y soluciones:

1. **Pantalla negra después del boot**:
   - Verifica los drivers de pantalla y los ficheros DTB

2. **Sin conexión WiFi/Bluetooth**:
   - Asegúrate de que los firmware estén correctamente instalados

3. **Problemas de hardware específico**:
   - Consulta los logs para identificar qué componente está fallando

## Recursos adicionales

- [Documentación oficial de UBports](https://docs.ubports.com/)
- [Foro de Halium](https://forums.halium.org/)
- [Grupo de Telegram para porting de Ubuntu Touch](https://t.me/UBports_Porting)

## Notas específicas para MTK Helio G95

El MediaTek Helio G95 puede requerir parches específicos para el kernel. Consulta los repositorios de LineageOS y otras ROMs personalizadas para encontrar soluciones a problemas específicos de este SoC.
