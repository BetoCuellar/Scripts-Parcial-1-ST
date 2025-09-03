# Scripts-Parcial-1-ST
Repositorio creado para la entrega de scripts en la resolucion del parcial 1 ST

Presentado por:

Nicolas Cuellar Castrellón

Santiago Duque Valencia

--------------------------------------------------------------------------------
== PARTE 1 — Apache + PAM (en VM web) ==
--------------------------------------------------------------------------------
archivo: parte1-apache-pam/instalar-paquetes.sh

#!/usr/bin/env bash
set -euxo pipefail
sudo apt update
sudo apt install -y apache2 libapache2-mod-authnz-external pwauth
--------------------------------------------------------------------------------
archivo: parte1-apache-pam/configurar-apache-privado.sh

#!/usr/bin/env bash
set -euxo pipefail

# Directorio privado y contenido mínimo
sudo mkdir -p /archivos_privados
echo "Top secret (PAM)" | sudo tee /archivos_privados/secret.txt
echo "<h1>Hola, zona privada</h1>" | sudo tee /archivos_privados/index.html

# Config de publicación + autenticación PAM (pwauth)
sudo tee /etc/apache2/conf-available/privado.conf >/dev/null <<'EOF'
# Publicar /archivos_privados
Alias /archivos_privados /archivos_privados

# Autenticación externa via pwauth (PAM)
AddExternalAuth pwauth /usr/sbin/pwauth
SetExternalAuthMethod pwauth pipe

# Página 401 si cancelan
ErrorDocument 401 /error_401.html

<Directory "/archivos_privados">
Options -Indexes
AllowOverride None
Require all granted

AuthType Basic
AuthName "Zona privada — Autenticación PAM"
AuthBasicProvider external
AuthExternal pwauth

# Acepta cualquier usuario válido del sistema
Require valid-user
</Directory>
EOF

# Página 401 personalizada
sudo tee /var/www/html/error_401.html >/dev/null <<'EOF'
<!DOCTYPE html><html><head><title>401 - Acceso no autorizado</title></head>
<body><h1>Acceso cancelado o no autorizado</h1>
<p>Has cancelado el inicio de sesión o tus credenciales no son válidas.</p></body></html>
EOF

# Activar y recargar
sudo a2enmod authnz_external
sudo a2enconf privado
sudo systemctl reload apache2
--------------------------------------------------------------------------------
archivo: parte1-apache-pam/configurar-usuarios-denegados.sh

#!/usr/bin/env bash
set -euxo pipefail

# Lista de usuarios denegados
echo -e "usuario3\ncarlos" | sudo tee /etc/security/usuarios_denegados
sudo chown root:root /etc/security/usuarios_denegados
sudo chmod 0640 /etc/security/usuarios_denegados

# Regla PAM (si no existe) para negar usuarios de la lista
PAM_FILE="/etc/pam.d/pwauth"
if ! grep -q "pam_listfile.so.*usuarios_denegados" "$PAM_FILE"; then
sudo sed -i '1i auth required pam_listfile.so item=user sense=deny file=/etc/security/usuarios_denegados onerr=fail' "$PAM_FILE"
fi
--------------------------------------------------------------------------------
archivo: parte1-apache-pam/crear-usuarios-prueba.sh

#!/usr/bin/env bash
set -euxo pipefail

# Usuarios de prueba
for u in usuario1 usuario2 usuario3 carlos; do
id "$u" >/dev/null 2>&1 || sudo adduser --disabled-password --gecos "" "$u"
echo "$u:clave123" | sudo chpasswd
done

# (Opcional) evita expiraciones raras
sudo chage -M 99999 usuario1 usuario2 usuario3 carlos
--------------------------------------------------------------------------------
validación rápida (Parte 1)

# OK (permitido): usuario1
curl -i -u usuario1:clave123 http://localhost/archivos_privados/secret.txt | sed -n '1,10p'

# DENEGADO por PAM (lista): usuario3 / carlos
curl -i -u usuario3:clave123 http://localhost/archivos_privados/secret.txt | sed -n '1,10p'

# Cancelar -> 401 con página personalizada
curl -i http://localhost/archivos_privados/secret.txt | sed -n '1,10p'
--------------------------------------------------------------------------------
== PARTE 2 — DNS Maestro/Esclavo (BIND9)==

IPs: maestro 192.168.56.11, esclavo 192.168.56.12, web 192.168.56.10. Dominio empresa.local; zona inversa 56.168.192.in-addr.arpa
--------------------------------------------------------------------------------
archivo: parte2-dns-bind/setup_master.sh

#!/usr/bin/env bash
set -euxo pipefail
DOMAIN="empresa.local"
REV_ZONE="56.168.192.in-addr.arpa"   # para 192.168.56.0/24
MASTER_IP="192.168.56.11"
SLAVE_IP="192.168.56.12"
WWW_IP="192.168.56.10"
SERIAL="2025090301"

export DEBIAN_FRONTEND=noninteractive
apt-get update && apt-get install -y bind9 dnsutils

cat >/etc/bind/named.conf.options <<EOF
options {
directory "/var/cache/bind";
recursion yes;
allow-query { 192.168.56.0/24; localhost; };
listen-on { ${MASTER_IP}; 127.0.0.1; };
dnssec-validation no;
allow-transfer { ${SLAVE_IP}; };
notify yes;
also-notify { ${SLAVE_IP}; };
};
EOF

cat >/etc/bind/named.conf.local <<EOF
zone "${DOMAIN}" {
type master;
file "/etc/bind/db.${DOMAIN}";
allow-transfer { ${SLAVE_IP}; };
also-notify   { ${SLAVE_IP}; };
};
zone "${REV_ZONE}" {
type master;
file "/etc/bind/db.192.168.56";
allow-transfer { ${SLAVE_IP}; };
also-notify   { ${SLAVE_IP}; };
};
EOF

cat >/etc/bind/db.${DOMAIN} <<EOF
$TTL 86400
@   IN SOA maestro.${DOMAIN}. admin.${DOMAIN}. (
${SERIAL}
3600
900
1209600
86400 )
IN NS maestro.${DOMAIN}.
IN NS esclavo.${DOMAIN}.

maestro   IN A ${MASTER_IP}
esclavo   IN A ${SLAVE_IP}
www       IN A ${WWW_IP}
intranet  IN CNAME www
EOF

cat >/etc/bind/db.192.168.56 <<'EOF'
$TTL 86400
@   IN SOA maestro.empresa.local. admin.empresa.local. (
2025090301
3600
900
1209600
86400 )
IN NS maestro.empresa.local.
IN NS esclavo.empresa.local.

10  IN PTR www.empresa.local.
11  IN PTR maestro.empresa.local.
12  IN PTR esclavo.empresa.local.
EOF

named-checkconf
named-checkzone ${DOMAIN} /etc/bind/db.${DOMAIN}
named-checkzone ${REV_ZONE} /etc/bind/db.192.168.56

systemctl restart bind9
systemctl enable named || true

echo "[MASTER] Listo: ${DOMAIN} y ${REV_ZONE}."
--------------------------------------------------------------------------------
archivo: parte2-dns-bind/setup_slave.sh

#!/usr/bin/env bash
set -euxo pipefail
MASTER_IP="192.168.56.11"
DOMAIN="empresa.local"
REV_ZONE="56.168.192.in-addr.arpa"
SELF_IP="192.168.56.12"

export DEBIAN_FRONTEND=noninteractive
apt-get update && apt-get install -y bind9 dnsutils

cat >/etc/bind/named.conf.options <<EOF
options {
directory "/var/cache/bind";
recursion yes;
allow-query { 192.168.56.0/24; localhost; };
listen-on { ${SELF_IP}; 127.0.0.1; };
dnssec-validation no;
};
EOF

cat >/etc/bind/named.conf.local <<EOF
zone "${DOMAIN}" {
type slave;
file "/var/cache/bind/db.${DOMAIN}";
masters { ${MASTER_IP}; };
};
zone "${REV_ZONE}" {
type slave;
file "/var/cache/bind/db.192.168.56";
masters { ${MASTER_IP}; };
};
EOF

systemctl restart bind9
systemctl enable named || true
ls -l /var/cache/bind | egrep 'db.empresa.local|db.192.168.56' || true
echo "[SLAVE] Listo."
--------------------------------------------------------------------------------
archivo: parte2-dns-bind/test_from_web.sh (en VM web)

#!/usr/bin/env bash
set -euxo pipefail
SLAVE_DNS="192.168.56.12"

# Forzar DNS=esclavo con systemd-resolved
sudo sed -i "s/^#\?DNS=.*/DNS=${SLAVE_DNS}/" /etc/systemd/resolved.conf
sudo sed -i "s/^#\?Domains=.*/Domains=empresa.local/" /etc/systemd/resolved.conf
sudo systemctl restart systemd-resolved || true

echo "== A de www ==";       dig @${SLAVE_DNS} www.empresa.local +noall +answer
echo "== CNAME intranet =="; dig @${SLAVE_DNS} intranet.empresa.local CNAME +noall +answer
echo "== PTR 192.168.56.11 =="; dig @${SLAVE_DNS} -x 192.168.56.11 +noall +answer
--------------------------------------------------------------------------------
validación rápida (Parte 2)

# En MASTER
sudo named-checkconf
sudo named-checkzone empresa.local /etc/bind/db.empresa.local
sudo named-checkzone 56.168.192.in-addr.arpa /etc/bind/db.192.168.56

# En SLAVE (evidencia de transferencia)
ls -l /var/cache/bind | egrep 'db.empresa.local|db.192.168.56'
dig @192.168.56.11 AXFR empresa.local | head -n 25  # opcional

# En WEB (resolver contra el esclavo)
dig @192.168.56.12 www.empresa.local +noall +answer
dig @192.168.56.12 intranet.empresa.local CNAME +noall +answer
dig @192.168.56.12 -x 192.168.56.11 +noall +answer

# Tolerancia a fallos
# En MASTER:
sudo systemctl stop bind9 || sudo systemctl stop named
# En WEB:
dig @192.168.56.12 www.empresa.local +short
dig @192.168.56.12 -x 192.168.56.11 +short
# Reactivar
sudo systemctl start bind9 || sudo systemctl start named
--------------------------------------------------------------------------------
== PARTE 3 — ngrok v3 (en VM web)==

Instala v3, configura authtoken y publica el 80
--------------------------------------------------------------------------------
archivo: parte3-ngrok/instalar-ngrok.sh

#!/usr/bin/env bash
set -euxo pipefail
sudo apt update
sudo apt install -y unzip wget tar
sudo rm -f /usr/local/bin/ngrok
wget https://bin.equinox.io/c/bNyj1mQVY4c/ngrok-v3-stable-linux-amd64.tgz -O ngrok-v3.tgz
tar -xvzf ngrok-v3.tgz
sudo mv ngrok /usr/local/bin/
ngrok version
--------------------------------------------------------------------------------
archivo: parte3-ngrok/configurar-ngrok.sh

#!/usr/bin/env bash
set -euxo pipefail
TOKEN="<TU_TOKEN_NGROK>"
ngrok config add-authtoken "$TOKEN"
ngrok config check
--------------------------------------------------------------------------------
archivo: parte3-ngrok/pagina-prueba.sh

#!/usr/bin/env bash
set -euxo pipefail
sudo tee /var/www/html/pagina_personalizada.html >/dev/null <<'EOF'
<!DOCTYPE html>
<html>
<head><title>Página personalizada - Punto 3</title></head>
<body>
<h1>Bienvenido al Punto 3 del Parcial</h1>
<p>Esta es una página de prueba publicada con ngrok.</p>
</body>
</html>
EOF
--------------------------------------------------------------------------------
validación rápida (Parte 3)

ngrok http 80
# Abre desde otra red (móvil en datos):
# https://<algo>.ngrok-free.app/
# https://<algo>.ngrok-free.app/pagina_personalizada.html
--------------------------------------------------------------------------------
