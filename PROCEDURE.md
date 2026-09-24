# Procédure — Wallboard de supervision CheckMK

> ℹ️ Les adresses IP, domaines et identifiants présents dans ce document sont anonymisés (plages RFC 5737 — `203.0.113.0/24`, `198.51.100.0/24`) et ne correspondent pas aux valeurs réelles de l'infrastructure.

## Sommaire

1. [Cahier des charges](#1-cahier-des-charges)
2. [Préparation de la clé USB](#2-préparation-de-la-clé-usb)
3. [Effacement du disque](#3-effacement-du-disque)
4. [Installation d'Ubuntu Desktop](#4-installation-dubuntu-desktop)
5. [Certificat racine d'entreprise](#5-certificat-racine-dentreprise)
6. [Installation de Chromium et mode kiosque](#6-installation-de-chromium-et-mode-kiosque)
7. [Résilience logicielle et matérielle](#7-résilience-logicielle-et-matérielle)
8. [Compte de supervision dédié](#8-compte-de-supervision-dédié)
9. [Passage en IP fixe](#9-passage-en-ip-fixe)
10. [Problème d'affichage résolu](#10-problème-daffichage-résolu)
11. [Recette finale](#11-recette-finale)

---

## 1. Cahier des charges

- Affichage continu du dashboard CheckMK sur une TV, sans intervention quotidienne
- Redémarrage et reprise autonomes après coupure de courant ou plantage
- Mode kiosque strict : plein écran, pas de barre d'adresse, pas de curseur visible
- Accès SSH pour la maintenance à distance

> [!NOTE]
 > ** Choix du système d’exploitation : **
> 
 > Nous avons choisi Ubuntu Desktop 24.04 LTS pour le Dell OptiPlex.
 > Ce choix s’explique principalement par le fait que la version Desktop est plus simple à configurer pour un poste destiné à afficher une interface graphique avec Chromium. La version LTS (Long Term Support) permet également  > de bénéficier de mises à jour et d’un support sur une longue durée, ce qui est adapté à une machine destinée à fonctionner en continu.

---

## 2. Préparation de la clé USB

```bash
lsblk
sudo dd if=ubuntu-24.04-desktop-amd64.iso of=/dev/sdX bs=4M status=progress oflag=sync
```

---

## 3. Effacement du disque

Le poste ayant déjà été utilisé par un précédent agent, un effacement du disque a été réalisé avant installation, depuis l'environnement live (« Essayer Ubuntu ») :

```bash
lsblk
sudo wipefs -a /dev/sda
```

> ⚠️ Un effacement de ce type ne peut pas se faire depuis un système actuellement démarré sur ce même disque — il faut d'abord booter sur un support externe (ici, la clé USB en mode « Essayer »).

---

## 4. Installation d'Ubuntu Desktop

Installation standard, avec deux points de vigilance :
- Coche **« Se connecter automatiquement »** à la création du compte — condition nécessaire à l'autologin, indispensable pour un poste sans surveillance
- **Coche « Install OpenSSH server »** si l'option est proposée, ou installe-le après coup (section 8)

---

## 5. Certificat racine d'entreprise

Le pare-feu de l'organisation effectue une inspection SSL (interception TLS) sur le trafic sortant, ce qui bloque par défaut les installations passant par HTTPS (notamment le Snap Store). Sans le certificat racine de l'entreprise, l'installation de Chromium échoue avec :

```
error: cannot install "chromium": Post "https://api.snapcraft.io/v2/snaps/refresh": tls:
       failed to verify certificate: x509: certificate signed by unknown authority
```

**Résolution :**

```bash
sudo cp certificate.crt /usr/local/share/ca-certificates/rootca.crt
sudo update-ca-certificates
sudo systemctl restart snapd
```

> 💡 Le redémarrage de `snapd` est l'étape souvent oubliée : `update-ca-certificates` met à jour la liste système, mais le service déjà démarré garde l'ancienne liste en mémoire tant qu'il n'est pas relancé.

**Vérification :**
```bash
curl -v https://api.snapcraft.io 2>&1 | grep -i issuer
```

---

## 6. Installation de Chromium et mode kiosque

```bash
sudo snap install chromium
```

**Script de lancement** (`/usr/bin/chromium-wallboard.sh`) :
```bash
#!/bin/bash
URL="http://203.0.113.21/monitoring"

unclutter -idle 0.1 -root &

chromium \
  --kiosk \
  --noerrdialogs \
  --disable-infobars \
  --disable-session-crashed-bubble \
  --disable-translate \
  --no-first-run \
  --start-fullscreen \
  --overscroll-history-navigation=0 \
  --check-for-update-interval=31536000 \
  "$URL"
```

**Lancement automatique** via un fichier `.desktop` dans `~/.config/autostart/`, exécuté à chaque ouverture de session GNOME (autologin).

**Connexion persistante au compte de supervision** : plutôt que de préremplir un formulaire de connexion (méthode retirée des versions récentes de CheckMK pour des raisons de sécurité), une connexion manuelle unique est effectuée, dont la session reste active durablement — Chromium ne tournant pas en navigation privée, son profil est conservé entre les redémarrages.

---

## 7. Résilience logicielle et matérielle

**Watchdog** — surveille et relance Chromium en cas de plantage :
```bash
#!/bin/bash
while true; do
  if ! pgrep -x "chromium" > /dev/null; then
    /usr/bin/chromium-wallboard.sh &
  fi
  sleep 15
done
```

> ⚠️ **Bug rencontré avec cette approche** : `pgrep -x "chromium"` ne trouve jamais de correspondance, car le processus Chromium s'exécute en réalité sous le nom `chrome` (particularité historique du projet, valable même pour la version Google Chrome). Résultat : le watchdog considérait Chromium comme absent en permanence et en relançait une nouvelle instance toutes les 15 secondes, provoquant un rafraîchissement de page en boucle. **Corrigé en migrant vers un service systemd utilisateur** (section suivante), qui surveille directement le processus qu'il a lui-même lancé plutôt que de chercher un nom de processus par pattern.

**Anti-veille** (GNOME) :
```bash
gsettings set org.gnome.desktop.session idle-delay 0
gsettings set org.gnome.desktop.screensaver lock-enabled false
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'
gsettings set org.gnome.desktop.screensaver idle-activation-enabled false
gsettings set org.gnome.desktop.notifications show-banners false
gsettings set org.gnome.settings-daemon.plugins.power power-button-action 'nothing'
```

**Redémarrage automatique après coupure de courant** : option **AC Recovery** activée au BIOS (`Power Management`), sur `Power On`.

### Migration vers un service systemd utilisateur

Le couple autostart `.desktop` + watchdog maison présentait deux limites : un délai perceptible avant l'affichage (attente de l'initialisation complète de GNOME) et le bug de détection décrit ci-dessus. Migration vers un service systemd utilisateur, plus rapide et plus fiable :

```bash
mkdir -p ~/.config/systemd/user
tee ~/.config/systemd/user/chromium-wallboard.service > /dev/null << 'EOF'
[Unit]
Description=Wallboard Chromium kiosque
After=graphical-session.target
PartOf=graphical-session.target

[Service]
ExecStart=/usr/bin/chromium-wallboard.sh
Restart=always
RestartSec=2

[Install]
WantedBy=graphical-session.target
EOF

systemctl --user enable chromium-wallboard.service
systemctl --user daemon-reload
```

**Avantages :** démarrage dès que la session graphique est prête (`graphical-session.target`), sans attendre le chargement complet de GNOME (panneau, extensions) ; relance native (`Restart=always`) basée sur l'état réel du processus lancé, plutôt que sur une recherche de nom de processus par pattern.

---

## 8. Compte de supervision dédié

Un compte CheckMK dédié (`kiosque`) a été créé avec un rôle en **lecture seule** — pas de droits d'administration, pour un poste affiché en continu dans un lieu de passage. Le délai d'inactivité de session est désactivé **uniquement pour ce compte** (réglage disponible par utilisateur dans CheckMK), sans toucher au délai global appliqué aux autres comptes.

---

## 9. Passage en IP fixe

```bash
sudo nmcli connection modify <nom-connexion> \
  ipv4.addresses 198.51.100.38/16 \
  ipv4.gateway 198.51.100.254 \
  ipv4.dns 198.51.100.254 \
  ipv4.method manual
sudo nmcli connection up <nom-connexion>
```

**Point de vigilance rencontré** : après bascule DHCP → IP fixe, l'ancienne adresse DHCP est restée active en parallèle (`inet ... secondary dynamic`), créant une ambiguïté réseau. Nettoyage :
```bash
sudo ip addr del <ancienne-ip>/16 dev <interface>
```

---

## 10. Problème d'affichage résolu

**Symptôme :** au premier allumage, l'image n'occupait pas toute la surface de l'écran (bande noire sur un côté), alors qu'un poste identique, sur le même écran, affichait correctement en plein écran.

**Diagnostic :** la connexion se faisant en VGA (signal analogique), la détection de la résolution native de l'écran repose sur la lecture des données **EDID** (Extended Display Identification Data — les informations que l'écran transmet au PC sur ses capacités, dont sa résolution native). Sur certaines configurations, un **problème de timing au démarrage** empêche cette lecture de se faire correctement : le système d'exploitation termine son démarrage avant que l'écran n'ait fini d'annoncer ses capacités, et retombe alors sur une résolution de secours non optimale.

**Résolution :** un simple cycle **extinction/rallumage de l'écran** (pas du PC) suffit à résoudre le problème. Éteindre puis rallumer l'écran déclenche un nouveau signal de détection (*hot-plug detect*), qui force une nouvelle lecture des données EDID par la carte graphique — cette fois sans contrainte de timing avec le démarrage du système. GNOME applique alors automatiquement la résolution native correctement détectée.

> 💡 Ce comportement est plus fréquent en VGA qu'en HDMI, la détection *hot-plug* étant historiquement moins fiable sur ce standard analogique plus ancien.

**Solution pérenne (sans intervention manuelle à chaque coupure de courant) :** plutôt que de compter sur une détection EDID correcte à chaque démarrage, la résolution est forcée explicitement dans le script de lancement du kiosque, via `xrandr` :

```bash
xrandr --output VGA-1 --mode 1920x1080
```

Cette ligne est ajoutée en tête de `chromium-wallboard.sh`, avant le lancement de Chromium — elle s'exécute donc à chaque ouverture de session (y compris après un redémarrage suite à coupure de courant), et applique la résolution native indépendamment du bon déroulement ou non de la négociation EDID automatique.

---

## 11. Recette finale

- [x] Démarrage entièrement autonome, sans intervention
- [x] Affichage plein écran du dashboard, sans éléments d'interface visibles
- [x] Reprise automatique après coupure d'alimentation (AC Recovery)
- [x] Relance automatique en cas de plantage du navigateur (watchdog)
- [x] Aucune mise en veille après plusieurs heures d'affichage continu
- [x] Accès SSH fonctionnel pour la maintenance à distance
- [x] Compte de supervision à droits limités, session persistante
