# ===================Practica 1=========================== #
# Red-hat-modulo-7
comandos del modulo 7
############################################
# ========== SERVIDOR (RHEL 10) ==========
############################################

# Instala el servidor NFS y sus utilidades
sudo dnf install -y nfs-utils

# Crea la carpeta compartida y los 100 archivos
sudo mkdir /OS3
sudo touch /OS3/Adrian{1..100}.txt

# Verifica que son 100 archivos
ls /OS3 | wc -l

# Permisos: dueño "nobody" y acceso total para todos los usuarios
sudo chown nobody:nobody /OS3
sudo chmod 777 /OS3

# Define qué se comparte y con qué permisos (aquí va la IP del CLIENTE)
echo "/OS3 192.168.xx.xx(rw,sync)" | sudo tee -a /etc/exports

# Verifica el contenido del archivo
cat /etc/exports

# Aplica los exports y muestra lo compartido
sudo exportfs -rav
sudo exportfs -v

# Inicia NFS y lo habilita en cada arranque
sudo systemctl enable --now nfs-server
sudo systemctl status nfs-server

# Firewall: abre los servicios necesarios y recarga
sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --permanent --add-service=mountd
sudo firewall-cmd --permanent --add-service=rpc-bind
sudo firewall-cmd --reload

# SELinux: permite a NFS exportar con lectura y escritura
sudo setsebool -P nfs_export_all_rw on


############################################
# ========== CLIENTE LINUX ===============
############################################

# Instala el cliente NFS (en Ubuntu/Debian: sudo apt install -y nfs-common)
sudo dnf install -y nfs-utils

# Comprueba conexión con el servidor (aquí va la IP del SERVIDOR)
ping -c 4 192.168.xx.xx

# Crea el punto de montaje y monta la carpeta compartida (IP del SERVIDOR)
sudo mkdir -p /mnt/OS3
sudo mount -t nfs 192.168.xx.xx:/OS3 /mnt/OS3

# Evidencia: archivos visibles, cantidad y montaje activo
ls /mnt/OS3
ls /mnt/OS3 | wc -l
df -h | grep OS3

# Prueba de escritura desde el cliente
touch /mnt/OS3/prueba.txt

# Desmonta para configurar el montaje automático
sudo umount /mnt/OS3

# Añade la entrada a fstab (IP del SERVIDOR)
echo "192.168.xx.xx:/OS3  /mnt/OS3  nfs  defaults,_netdev  0  0" | sudo tee -a /etc/fstab

# Verifica fstab, recarga y prueba el montaje sin reiniciar
cat /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
df -h | grep OS3

# Reinicia para demostrar el montaje automático
sudo reboot


############################################
# ===== DESPUÉS DE REINICIAR (CLIENTE) ====
############################################

# Evidencia final: debe aparecer montado sin haber hecho nada
df -h | grep OS3
ls /mnt/OS3

############################################################
# PRÁCTICA 2: FILE SERVER SAMBA
# Contraseñas: adrian = Samba2026
############################################################

# ----- SERVIDOR RHEL -----
sudo dnf install -y samba samba-client policycoreutils-python-utils
sudo systemctl enable --now smb nmb
sudo firewall-cmd --permanent --add-service=samba
sudo firewall-cmd --reload

sudo groupadd sambausers
sudo useradd -M -s /sbin/nologin -G sambausers adrian
(echo "Samba2026"; echo "Samba2026") | sudo smbpasswd -s -a adrian

sudo mkdir -p /srv/samba/compartida
sudo chown -R root:sambausers /srv/samba/compartida
sudo chmod -R 2770 /srv/samba/compartida
sudo semanage fcontext -a -t samba_share_t "/srv/samba/compartida(/.*)?"
sudo restorecon -Rv /srv/samba/compartida

sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.bak
sudo tee /etc/samba/smb.conf > /dev/null <<'EOF'
[global]
    workgroup = WORKGROUP
    server string = Fileserver RHEL
    security = user
    map to guest = never

[compartida]
    path = /srv/samba/compartida
    browseable = yes
    writable = yes
    valid users = @sambausers
    force group = sambausers
    create mask = 0660
    directory mask = 2770
EOF
testparm
sudo systemctl restart smb nmb

sudo bash -c 'cd /srv/samba/compartida && touch adrian{1..100} && chown adrian:sambausers adrian* && chmod 660 adrian*'
sudo ls /srv/samba/compartida | wc -l

# ----- WINDOWS (CMD) -----
# ping 192.168.1.30
# net use Z: \\192.168.1.30\compartida /user:adrian
# (clave: Samba2026) Abrir Z:, editar adrian99, escribir: el zumzum de la carabela, guardar
# net use * /delete /y

# ----- VALIDAR EN RHEL -----
sudo cat /srv/samba/compartida/adrian99


############################################################
# PRÁCTICA 3: DOMINIO SO3.inet
# Contraseñas: Administrator = P@ssw0rd2026 / lanegracubana = 20261160
############################################################

# ----- SERVIDOR RHEL -----
sudo useradd -M -s /sbin/nologin lanegracubana
echo "lanegracubana:20261160" | sudo chpasswd
(echo "20261160"; echo "20261160") | sudo smbpasswd -s -a lanegracubana
sudo pdbedit -L

sudo systemctl disable --now smb nmb winbind
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.practica2
sudo dnf remove -y 'samba*' libwbclient libsmbclient
sudo rm -rf /var/lib/samba/*

sudo hostnamectl set-hostname dc1.so3.inet
echo "192.168.1.30 dc1.so3.inet dc1" | sudo tee -a /etc/hosts
sudo systemctl enable --now chronyd

sudo subscription-manager repos --enable codeready-builder-for-rhel-10-$(arch)-rpms
sudo wget -O /etc/pki/rpm-gpg/RPM-GPG-KEY-TISSAMBA-10 https://samba.tranquil.it/RPM-GPG-KEY-TISSAMBA-10
sudo rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-TISSAMBA-10
echo "[tis-samba]
name=tis-samba
baseurl=https://samba.tranquil.it/redhat10/samba-4.22/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-TISSAMBA-10
priority=1" | sudo tee /etc/yum.repos.d/tissamba.repo
sudo dnf clean all
sudo dnf install -y --allowerasing samba samba-dc samba-client krb5-workstation
which samba-tool

sudo rm -f /etc/samba/smb.conf
sudo rm -rf /var/lib/samba/private/*
sudo samba-tool domain provision --realm=SO3.INET --domain=SO3 --server-role=dc --dns-backend=SAMBA_INTERNAL --option="dns forwarder=8.8.8.8" --adminpass='P@ssw0rd2026'
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf

CON=$(nmcli -t -f NAME con show --active | head -1)
sudo nmcli con mod "$CON" ipv4.dns 192.168.1.30 ipv4.dns-search so3.inet
sudo nmcli con up "$CON"
sudo firewall-cmd --permanent --add-service={dns,kerberos,kpasswd,ldap,ldaps,samba,samba-dc}
sudo firewall-cmd --reload

sudo systemctl unmask samba
sudo systemctl enable --now samba

sudo samba-tool domain passwordsettings set --complexity=off --min-pwd-length=1 --min-pwd-age=0 --history-length=0
sudo samba-tool user create lanegracubana 20261160
sudo samba-tool user list

# ----- WINDOWS (PowerShell como administrador) -----
# Get-NetAdapter | Where Status -eq Up | ForEach { Set-DnsClientServerAddress -InterfaceIndex $_.ifIndex -ServerAddresses 192.168.1.30 }
# ping dc1.so3.inet
# Add-Computer -DomainName SO3.inet -Credential SO3\Administrator -Restart
# (clave Administrator: P@ssw0rd2026)
# Tras reiniciar: Otro usuario -> SO3\lanegracubana / 20261160
# En CMD: whoami   (debe mostrar so3\lanegracubana)
