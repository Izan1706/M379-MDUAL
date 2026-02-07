#!/bin/bash
# PROYECTO PERSONAL IZAN RODRIGUEZ
# -----------------------------------------------------------
#PORFAVOR ANTES DE EJECUTARLO LEER ESTO: Este es un script hecho por Izan Rodriguez #Garcia, se ha comprobado, pero porfavor leerlo y comprenderlo antes de usarlo.
#Se recomienda cambiar la contrasña (Contraseña por defecto P@ssw0rd), tambien se #recomienda revisar si su unidad en concreto es la /dev/sdb, por ultimo ESTE SCRIPT #FORMATEA EL DISCO QUE USARA PARA NAS, SI TIENE ALGO IMPORTANTE PONGALO A SALVO ANTES #DE EJECUTARLO" porfavor

# === CONFIGURACION (VARIABLES) ===
DISCO="/dev/sdb"
PUNTO_MONTAJE="/media/nas"
USUARIO="nasuser"
PASS="P@ssw0rd"
GRUPO="nasuser"

# --- COMPROBACIONES PREVIAS ---
echo "0. Comprobando sistema..."

# Comprobar Internet
wget -q --spider http://google.com
if [ $? -ne 0 ]; then
    echo "❌ ERROR: No hay conexión a Internet."
    exit 1
fi

# Esperar bloqueo de APT (Por si Ubuntu se está actualizando solo)
while fuser /var/lib/dpkg/lock-frontend >/dev/null 2>&1; do
    echo "⏳ Esperando a que terminen otras actualizaciones de Ubuntu (esto puede tardar)..."
    sleep 5
done

# Comprobamos que se ejecuta como root
if [ "$EUID" -ne 0 ]; then
  echo "❌ Por favor, ejecuta este script como root (sudo)."
  exit 1
fi

# --- INSTALACION SAMBA (MODIFICADO) ---
echo "3. Instalando Samba (puede tardar)..."

# Actualizamos lista de paquetes
apt-get update

# Instalamos SIN modo silencioso para ver errores si los hay
apt-get install -y samba

# VERIFICAMOS SI SE INSTALÓ BIEN
if [ $? -ne 0 ]; then
    echo "❌ ERROR CRÍTICO: Samba no se pudo instalar."
    echo "   Intenta ejecutar: sudo apt install samba -y manualmente."
    exit 1
fi

echo "--- INICIANDO INSTALACION DEL NAS ---"

# --- PASO 1: PREPARAR DISCO ---
echo "1. Preparando el disco $DISCO..."

# Desmontamos por si acaso
umount $DISCO 2>/dev/null

# Formateamos a EXT4 (Opción -F fuerza el formateo sin preguntar)
mkfs.ext4 -F $DISCO > /dev/null 2>&1
echo "   -> Disco formateado a EXT4."

# Creamos directorio
mkdir -p $PUNTO_MONTAJE

# Obtenemos UUID
UUID=$(blkid -s UUID -o value $DISCO)

# Añadimos a fstab solo si no existe ya para evitar duplicados
if ! grep -q "$UUID" /etc/fstab; then
    echo "UUID=$UUID $PUNTO_MONTAJE ext4 defaults 0 0" >> /etc/fstab
    echo "   -> Añadido a /etc/fstab para montaje automático."
else
    echo "   -> Ya estaba en fstab."
fi

# Montamos
mount -a

# --- PASO 2: GESTION DE USUARIOS ---
echo "2. Configurando usuario y permisos..."

# Crear usuario si no existe
if id "$USUARIO" &>/dev/null; then
    echo "   -> El usuario $USUARIO ya existe."
else
    useradd -M -s /sbin/nologin $USUARIO
    echo "   -> Usuario $USUARIO creado."
fi

# Asignar permisos
chown -R $USUARIO:$GRUPO $PUNTO_MONTAJE
chmod -R 770 $PUNTO_MONTAJE
echo "   -> Permisos asignados (770)."

# --- PASO 3: INSTALACION SAMBA ---
echo "3. Instalando y configurando Samba..."

# Actualizamos repositorios e instalamos (silencioso)
apt-get update -qq
apt-get install -y samba -qq > /dev/null 2>&1

# Configurar contraseña Samba
(echo "$PASS"; echo "$PASS") | smbpasswd -a -s $USUARIO
echo "   -> Contraseña de Samba establecida."

# Backup de configuración
if [ ! -f /etc/samba/smb.conf.bak ]; then
    mv /etc/samba/smb.conf /etc/samba/smb.conf.bak
fi

# Crear nueva configuración
cat > /etc/samba/smb.conf <<EOF
[global]
   workgroup = WORKGROUP
   server string = Ubuntu NAS Proyecto ASIX
   security = user
   map to guest = bad user
   dns proxy = no

[CarpetaPrivada]
   path = $PUNTO_MONTAJE
   browsable = yes
   writable = yes
   guest ok = no
   read only = no
   valid users = $USUARIO
   force user = $USUARIO
   create mask = 0770
   directory mask = 0770
EOF

# --- PASO 4: FINALIZAR ---
systemctl restart smbd
echo "---------------------------------------"
echo "✅ INSTALACIÓN COMPLETADA CON ÉXITO"
echo "---------------------------------------"
echo "Datos de acceso:"
echo "   IP Servidor: $(hostname -I | cut -d' ' -f1)"
echo "   Recurso:     CarpetaPrivada"
echo "   Usuario:     $USUARIO"
echo "   Contraseña:  $PASS"
echo "---------------------------------------"