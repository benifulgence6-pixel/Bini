#!/bin/bash

clear
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "      🛡️ Beni Ano le King - Version Clé + Durée 🛡️"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""

# ==================================================
# Vérifications système EN PREMIER (avant de demander
# la clé, pour ne pas la redemander deux fois)
# ==================================================
if [ "$EUID" -ne 0 ]; then
    echo "🔐 Ce script doit être exécuté en root, relance avec sudo..."
    exec sudo bash "$0" "$@"
fi

if [ ! -f /etc/os-release ]; then
    echo "❌ Impossible de détecter le système (/etc/os-release introuvable)."
    exit 1
fi

source /etc/os-release
if [ "$ID" != "ubuntu" ]; then
    echo "❌ Ce script est conçu uniquement pour Ubuntu."
    exit 1
fi

echo "✔ Système Ubuntu détecté, exécution en root confirmée."
echo ""

# ==================================================
# 🔑 VÉRIFICATION DE LA CLÉ VIA L'API CENTRALE
# (empêche la réutilisation de la même clé sur un autre serveur)
# ==================================================
LICENSE_API_URL="http://benifulgenc1.vps.webdock.cloud:8000/api/claim"
LICENSE_API_SECRET="CHANGE_MOI_AVEC_UN_SECRET_ALEATOIRE"  # doit correspondre au secret du service

BASE="/etc/benivps"
# ==================================================

mkdir -p "$BASE"

# Demande de la clé
if [ -z "${INSTALL_KEY:-}" ]; then
    read -p "🔑 Entre ta clé d'installation : " INSTALL_KEY
fi

INSTALL_KEY=$(echo "$INSTALL_KEY" | tr -d ' \r\n')

SERVER_IP_TMP=$(curl -s ifconfig.me)
SERVER_NAME_TMP=$(hostname)

API_RESPONSE=$(curl -s -X POST "$LICENSE_API_URL" \
    -H "Content-Type: application/json" \
    -H "X-API-Secret: $LICENSE_API_SECRET" \
    -d "{\"key\":\"$INSTALL_KEY\",\"server_ip\":\"$SERVER_IP_TMP\",\"server_name\":\"$SERVER_NAME_TMP\"}")

if [ -z "$API_RESPONSE" ]; then
    echo ""
    echo "❌ Impossible de contacter le serveur de licence. Vérifie ta connexion internet."
    exit 1
fi

IS_VALID=$(echo "$API_RESPONSE" | grep -o '"valid": *true')

if [ -z "$IS_VALID" ]; then
    REASON=$(echo "$API_RESPONSE" | grep -o '"reason": *"[^"]*"' | cut -d'"' -f4)
    echo ""
    case "$REASON" in
        already_used) echo "❌ Cette clé a déjà été utilisée sur un autre serveur." ;;
        invalid_key)  echo "❌ Clé invalide." ;;
        unauthorized) echo "❌ Erreur d'autorisation avec le serveur de licence." ;;
        *)            echo "❌ Clé refusée (raison : ${REASON:-inconnue})." ;;
    esac
    exit 1
fi

DURATION=$(echo "$API_RESPONSE" | grep -o '"duration_days": *[0-9]*' | grep -o '[0-9]*')
INSTALL_DATE=$(echo "$API_RESPONSE" | grep -o '"install_date": *[0-9]*' | grep -o '[0-9]*')
EXPIRE_DATE=$(echo "$API_RESPONSE" | grep -o '"expire_date": *[0-9]*' | grep -o '[0-9]*')

echo "✅ Clé valide (durée : $DURATION jours)"
sleep 1

# Installation des paquets
echo "📦 Installation des paquets de base..."
apt update -y
apt install -y curl wget git unzip zip tar sudo nano cron net-tools dnsutils lsof screen jq bc socat openssl ca-certificates openssh-server

systemctl enable ssh
systemctl restart ssh

clear
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "        CONFIGURATION DU SERVEUR"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

read -p "📝 Nom du serveur : " SERVER_NAME
read -p "🌐 Domaine (optionnel) : " SERVER_DOMAIN

SERVER_IP=$(curl -s ifconfig.me)
if [ -z "$SERVER_IP" ]; then
    echo "⚠️  Impossible de récupérer l'IP publique (pas de connexion internet ?)."
    SERVER_IP="INCONNUE"
fi

mkdir -p "$BASE"/{protocolos,usuarios,sistema,logs,herramientas}

# Configuration + licence
cat > "$BASE/config.conf" <<EOF
SERVER_NAME="$SERVER_NAME"
SERVER_DOMAIN="$SERVER_DOMAIN"
SERVER_IP="$SERVER_IP"

CLOUDFLARE_STATUS="OFF"
SSL_TUNNEL="ON"
DOMAIN_IP_MATCH="NO"
PROXY_STATUS="UNKNOWN"

AUTO_START=ON
OPTIMIZAR=ON

OPENSSH=ON
SYSTEMDNS=ON
WEBSOCKET=ON
ZIPVPN=ON
DROPBEAR=ON
SSL=ON
BADVPN=ON
UDP_CUSTOM=ON
SLOWDNS=ON
V2RAY=ON
XRAY=ON
OPENVPN=OFF
SQUID=OFF
TROJAN=OFF
SHADOWSOCKS=OFF
SOCKS5=OFF
WEBMIN=OFF
FAIL2BAN=ON
BBR=ON

# Licence
LICENSE_KEY="$INSTALL_KEY"
LICENSE_INSTALL_DATE="$INSTALL_DATE"
LICENSE_EXPIRE_DATE="$EXPIRE_DATE"
LICENSE_DURATION="$DURATION"
EOF

# La clé est déjà marquée comme utilisée côté API (dès l'appel /api/claim),
# donc plus besoin de fichier local ici.

echo "✅ Configuration terminée."

# Clone de TON FORK
echo "📥 Téléchargement depuis ton dépôt..."
cd /root || exit 1
rm -rf /tmp/multi-script

git clone https://github.com/benifulgence6-pixel/multi-script.git /tmp/multi-script || {
    echo "❌ Erreur de clonage."
    exit 1
}

cp -a /tmp/multi-script/. "$BASE/"
chmod -R +x "$BASE"
rm -rf /tmp/multi-script

# ==================================================
# Création du lanceur menu avec vérification de licence
# ==================================================
cat > /usr/local/bin/xmenu <<'INNEREOF'
#!/bin/bash

BASE="/etc/benivps"
CONFIG="$BASE/config.conf"

if [ ! -f "$CONFIG" ]; then
    echo "❌ Configuration introuvable. Réinstalle le script."
    exit 1
fi

source "$CONFIG"

if [ -z "$LICENSE_EXPIRE_DATE" ]; then
    echo "❌ Licence introuvable."
    exit 1
fi

NOW=$(date +%s)

if [ "$NOW" -gt "$LICENSE_EXPIRE_DATE" ]; then
    clear
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo "          ❌ LICENCE EXPIRÉE"
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo ""
    echo "La durée d'utilisation de ce panel est terminée."
    echo ""
    echo "Clé utilisée     : $LICENSE_KEY"
    echo "Durée allouée    : $LICENSE_DURATION jours"
    echo ""
    echo "Contacte le vendeur pour renouveler."
    echo ""
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    exit 1
fi

REMAINING=$(( (LICENSE_EXPIRE_DATE - NOW) / 86400 ))

if [ "$REMAINING" -le 3 ]; then
    echo ""
    echo "⚠️  Attention : il reste seulement $REMAINING jour(s) de licence."
    sleep 2
fi

exec bash "$BASE/menu.sh"
INNEREOF

chmod +x /usr/local/bin/xmenu

clear
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "        ✅ INSTALLATION TERMINÉE"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""
echo "📝 Serveur   : $SERVER_NAME"
echo "🌐 Domaine   : ${SERVER_DOMAIN:-Aucun}"
echo "🔗 IP        : $SERVER_IP"
echo "🔑 Clé       : $INSTALL_KEY"
echo "⏳ Durée     : $DURATION jours"
echo ""
echo "💡 Commande pour ouvrir le menu :"
echo "   xmenu"
echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
