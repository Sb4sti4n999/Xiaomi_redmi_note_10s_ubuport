# Guía completa para crear un port de Ubuntu Touch en Redmi Note 10S (Helio G95)

A continuación te presento una guía detallada, paso por paso, para iniciar el proceso de porting de Ubuntu Touch en tu dispositivo Xiaomi Redmi Note 10S con procesador MediaTek Helio G95, trabajando desde un sistema Ubuntu.

## Preparación del entorno de desarrollo

### 1. Configura tu sistema Ubuntu
```bash
# Actualiza el sistema
sudo apt update && sudo apt upgrade -y

# Instala dependencias básicas
sudo apt install git curl wget adb fastboot android-tools-adb android-tools-fastboot

# Instala dependencias para compilación
sudo apt install build-essential ccache lzop bison gperf cmake ninja-build zip unzip zlib1g-dev libncurses5-dev lib32ncurses5-dev libsdl1.2-dev libxml2-utils xsltproc bc flex g++-multilib gcc-multilib libc6-dev-i386 lib32z1-dev liblz4-tool pngcrush schedtool python3 python3-pip
```

### 2. Configura el repositorio UBports
```bash
# Añade el repositorio de UBports
sudo add-apt-repository ppa:ubports-developers/ppa
sudo apt update

# Instala herramientas de UBports
sudo apt install ubports-tools phablet-tools
```

### 3. Crea la estructura de directorios
```bash
# Crea el directorio principal
mkdir -p ~/halium-build/
cd ~/halium-build/

# Prepara el entorno repo
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo
export PATH=~/bin:$PATH
```

## Preparación del dispositivo

### 1. Obtén información detallada del dispositivo
```bash
# Asegúrate de tener el dispositivo conectado y el modo de depuración USB activado
adb devices

# Obtén información clave del dispositivo
adb shell getprop > ~/halium-build/device_props.txt
adb shell cat /proc/cpuinfo > ~/halium-build/cpuinfo.txt
adb shell cat /proc/meminfo > ~/halium-build/meminfo.txt
adb shell cat /proc/cmdline > ~/halium-build/cmdline.txt
```

### 2. Extrae información de particiones
```bash
# Obtén la tabla de particiones
adb shell cat /proc/partitions > ~/halium-build/partitions.txt
adb shell ls -la /dev/block/platform/*/by-name/ > ~/halium-build/partition_layout.txt

# Obtén información del bootloader
adb reboot bootloader
fastboot getvar all > ~/halium-build/bootloader_vars.txt
fastboot reboot
```

## Configuración de Halium

### 1. Inicializa el repositorio Halium
```bash
# Crea el directorio para Halium
mkdir -p ~/halium/
cd ~/halium/

# Inicializa repo
repo init -u https://github.com/Halium/android -b halium-11.0 --depth=1
```

### 2. Crea manifiestos locales para tu dispositivo
```bash
# Crea directorio para manifiestos locales
mkdir -p .repo/local_manifests/

# Crea el archivo rosemary.xml (nombre en clave del Redmi Note 10S)
nano .repo/local_manifests/rosemary.xml
```

Contenido para `rosemary.xml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
  <!-- Repositorios específicos del dispositivo -->
  <project path="device/xiaomi/rosemary" name="LineageOS/android_device_xiaomi_rosemary" remote="github" revision="lineage-18.1" />
  <project path="vendor/xiaomi/rosemary" name="TheMuppets/proprietary_vendor_xiaomi_rosemary" remote="github" revision="lineage-18.1" />
  <project path="kernel/xiaomi/rosemary" name="mt6785-common/android_kernel_xiaomi_mt6785" remote="github" revision="lineage-18.1" />
  
  <!-- Repositorios específicos para MediaTek -->
  <project path="device/mediatek/sepolicy_vndr" name="LineageOS/android_device_mediatek_sepolicy_vndr" remote="github" revision="lineage-18.1" />
  <project path="hardware/mediatek" name="LineageOS/android_hardware_mediatek" remote="github" revision="lineage-18.1" />
</manifest>
```

### 3. Sincroniza los repositorios
```bash
# Sincroniza todos los repositorios (puede tomar varias horas)
repo sync -c -j$(nproc --all)
```

## Adaptación para el dispositivo

### 1. Configura el entorno de compilación para Halium
```bash
# Entra en el directorio Halium
cd ~/halium/

# Configura el entorno
source build/envsetup.sh

# Configura ccache para acelerar compilaciones futuras
export USE_CCACHE=1
export CCACHE_EXEC=/usr/bin/ccache
ccache -M 50G

# Prepara la configuración para tu dispositivo
breakfast rosemary
```

### 2. Crea los archivos de configuración de Halium
```bash
# Crea el archivo de configuración para Halium
cd ~/halium/device/xiaomi/rosemary/
touch halium.mk

# Edita el archivo
nano halium.mk
```

Contenido para `halium.mk`:
```makefile
# Inherits
$(call inherit-product, $(SRC_TARGET_DIR)/product/full_base_telephony.mk)
$(call inherit-product, $(SRC_TARGET_DIR)/product/halium.mk)
$(call inherit-product, device/xiaomi/rosemary/device.mk)

# Device identifiers
PRODUCT_DEVICE := rosemary
PRODUCT_NAME := halium_rosemary
PRODUCT_BRAND := Xiaomi
PRODUCT_MODEL := Redmi Note 10S
PRODUCT_MANUFACTURER := Xiaomi

# Overrides
PRODUCT_BUILD_PROP_OVERRIDES += \
    TARGET_DEVICE=rosemary \
    PRODUCT_NAME=rosemary
```

### 3. Adapta la configuración del kernel para Halium
```bash
# Entra en el directorio del kernel
cd ~/halium/kernel/xiaomi/rosemary/

# Crea una configuración específica para Halium
cp arch/arm64/configs/rosemary_defconfig arch/arm64/configs/halium_rosemary_defconfig

# Edita la configuración
nano arch/arm64/configs/halium_rosemary_defconfig
```

Añade estas líneas al final de la configuración:
```
CONFIG_AUDIT=y
CONFIG_SECURITY_SELINUX=y
CONFIG_SECURITY_SELINUX_BOOTPARAM=y
CONFIG_SECURITY_SELINUX_BOOTPARAM_VALUE=0
CONFIG_SECURITY_SELINUX_DISABLE=y
CONFIG_IKCONFIG=y
CONFIG_IKCONFIG_PROC=y
CONFIG_NAMESPACES=y
CONFIG_FHANDLE=y
CONFIG_CGROUP_FREEZER=y
CONFIG_CPUSETS=y
CONFIG_CGROUP_DEVICE=y
CONFIG_MEMCG=y
CONFIG_MEMCG_SWAP=y
CONFIG_MEMCG_KMEM=y
CONFIG_BLK_CGROUP=y
CONFIG_NET_CLS_CGROUP=y
CONFIG_CGROUP_NET_PRIO=y
CONFIG_CGROUP_PERF=y
CONFIG_SCHED_AUTOGROUP=y
CONFIG_CFS_BANDWIDTH=y
CONFIG_RT_GROUP_SCHED=y
CONFIG_SECCOMP=y
CONFIG_SECCOMP_FILTER=y
CONFIG_CHECKPOINT_RESTORE=y
CONFIG_PROC_DEVICETREE=y
CONFIG_PROC_CHILDREN=y
CONFIG_UTS_NS=y
CONFIG_IPC_NS=y
CONFIG_PID_NS=y
CONFIG_NET_NS=y
CONFIG_USER_NS=y
```

## Compilación del sistema

### 1. Compila el kernel y el sistema
```bash
# Vuelve al directorio principal
cd ~/halium/

# Compila el sistema
make -j$(nproc --all) halium-boot
make -j$(nproc --all) systemimage
```

### 2. Prepara los archivos de instalación
```bash
# Crea un directorio para los archivos de instalación
mkdir -p ~/halium-install
cd ~/halium-install

# Copia los archivos compilados
cp ~/halium/out/target/product/rosemary/halium-boot.img ./
cp ~/halium/out/target/product/rosemary/system.img ./
```

## Preparación específica para MediaTek

### 1. Configura herramientas específicas para MediaTek
```bash
# Instala herramientas específicas para MediaTek
cd ~/
git clone https://github.com/bkerler/mtkclient
cd mtkclient
pip3 install -r requirements.txt
python3 setup.py install
```

### 2. Realiza una copia de seguridad completa
```bash
# Usa SP Flash Tool para realizar un backup completo
# (Esta es una herramienta gráfica, sigue las instrucciones en pantalla)
sudo apt install libpng12-0
wget https://spflashtool.com/download/SP_Flash_Tool_v5.1924_Linux.zip
unzip SP_Flash_Tool_v5.1924_Linux.zip
cd SP_Flash_Tool_Linux
chmod +x flash_tool
./flash_tool
```

### 3. Extrae y adapta firmware específico para MediaTek
```bash
# Crea directorios para firmware
mkdir -p ~/halium-build/firmware/{wifi,bt,gps}

# Extrae firmware WiFi
adb pull /vendor/firmware/wifi ~/halium-build/firmware/wifi/

# Extrae firmware Bluetooth
adb pull /vendor/firmware/bt ~/halium-build/firmware/bt/

# Extrae ficheros de calibración
adb pull /vendor/etc/calibration ~/halium-build/firmware/
```

## Instalación de Ubuntu Touch

### 1. Prepara el dispositivo para la instalación
```bash
# Reinicia en modo bootloader
adb reboot bootloader

# Flashea la imagen del kernel Halium
fastboot flash boot ~/halium-install/halium-boot.img

# En algunos dispositivos MediaTek puede necesitar:
fastboot flash boot_a ~/halium-install/halium-boot.img
fastboot flash boot_b ~/halium-install/halium-boot.img
```

### 2. Instala Ubuntu Touch
```bash
# Para dispositivos MediaTek, es aconsejable usar la herramienta de Halium
cd ~/
git clone https://gitlab.com/JBBgameich/halium-install.git
cd halium-install
pip3 install .

# Instala Ubuntu Touch
sudo halium-install -p ut ~/halium-install/system.img ~/halium-install/halium-boot.img
```

### 3. Configura el sistema
```bash
# Reinicia en modo normal
fastboot reboot

# Espera a que el dispositivo arranque y se conecte
adb wait-for-device

# Configura acceso SSH para depuración
adb shell
sudo passwd phablet  # Establece una contraseña para el usuario phablet
exit

# Redirige el puerto SSH
adb forward tcp:2222 tcp:22

# Conéctate por SSH
ssh phablet@localhost -p 2222
```

## Depuración y finalización

### 1. Verifica los servicios fundamentales
```bash
# Dentro del dispositivo (vía SSH)
systemctl status lightdm.service
systemctl status unity8.service
```

### 2. Configura servicios específicos para MediaTek
```bash
# Crea configuración para sensores
sudo mkdir -p /etc/ubuntu-touch-session.d/
sudo nano /etc/ubuntu-touch-session.d/android.conf
```

Añade estas líneas:
```
ANDROID_CONFIG=(
    persist.sys.usb.config=mtp,adb
    ro.telephony.sim.count=2
    ro.mediatek.platform=MT6785
    ro.hardware=mt6785
)
```

### 3. Soluciona problemas comunes de MediaTek
```bash
# Verifica servicios de telefonía
sudo systemctl status ofono.service
sudo systemctl status repowerd.service

# Verifica controladores de cámara
ls -la /dev/video*

# Verifica estado WiFi
sudo systemctl status network-manager.service
sudo nmcli device
```

## Post-instalación y optimizaciones

### 1. Configura servicios de Ubuntu Touch
```bash
# Reconfigura servicios principales
sudo mount -o remount,rw /
sudo apt update
sudo apt install ubuntu-touch-session-mir

# Configura sensores para MediaTek
sudo nano /etc/init/mtkservices.conf
```

Contenido para `mtkservices.conf`:
```
description "MediaTek services"
start on started lightdm

task
script
    echo "performance" > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
    echo "performance" > /sys/devices/system/cpu/cpu1/cpufreq/scaling_governor
    echo "performance" > /sys/devices/system/cpu/cpu2/cpufreq/scaling_governor
    echo "performance" > /sys/devices/system/cpu/cpu3/cpufreq/scaling_governor
end script
```

### 2. Habilita actualizaciones OTA
```bash
# Configura el canal de actualización
sudo system-image-cli --switch ubports-touch/16.04/devel

# Habilita actualizaciones
sudo touch /userdata/.writable_image
sudo touch /userdata/.adb_onlock
```

Esta guía abarca todos los pasos principales para iniciar el desarrollo de un port de Ubuntu Touch para el Redmi Note 10S. Ten en cuenta que el proceso puede requerir ajustes específicos según las particularidades de tu dispositivo y la versión exacta del firmware. El desarrollo de un port es un proceso iterativo que puede requerir varias semanas o meses de trabajo, especialmente para dispositivos con chipsets MediaTek que suelen tener menos soporte en la comunidad de desarrollo.