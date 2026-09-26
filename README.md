# 🖥️ Automatisation du Wallboard CheckMK (Dell OptiPlex / Ubuntu 24.04 LTS)

Documentation technique et fichiers de configuration pour le projet de wallboard de supervision CheckMK déployé au centre de services. Ce projet a été entièrement modernisé dans le cadre de mon BTS SIO (option SISR) pour remplacer un ancien script de surveillance (*watchdog*) instable par une architecture native, propre et robuste basée sur **systemd**.

> ℹ️ Les adresses IP présentées dans ce dépôt sont anonymisées (plage RFC 5737 — `203.0.113.0/24`) et ne correspondent pas aux valeurs réelles de l'infrastructure de production.

---

## 📋 Présentation du Projet

* **Objectif :** Afficher en continu et de manière autonome l'état de supervision des équipements réseau sur un écran mural.
* **Matériel :** PC Dell OptiPlex (Nom d'hôte : `wallboard-desk`, IP : `203.0.113.38`).
* **Système d'exploitation :** Ubuntu Desktop 24.04 LTS.
* **Évolution architecturale :** Migration d'un lancement par fichier `.desktop` et d'un script de *watchdog* externe en arrière-plan vers un **service utilisateur systemd** associé au protocole graphique **X11** pour fiabiliser la connexion automatique.

### Cahier des charges

- Affichage continu du dashboard CheckMK, sans intervention quotidienne
- Redémarrage et reprise autonomes après coupure de courant ou plantage
- Mode kiosque strict : plein écran, sans éléments d'interface
- Accès SSH pour la maintenance à distance
- Système d'exploitation : Ubuntu Desktop LTS, pour rester distinct du serveur CheckMK existant (lui-même sous Ubuntu Server)

---

## ⚙️ Stack Technique

- **Gestionnaire de connexion :** GDM3 configuré avec `AutomaticLogin` et désactivation de Wayland (`WaylandEnable = false`) pour contourner les invites de trousseau de clés interactives.
- **Supervision des processus :** Service utilisateur systemd (`chromium-wallboard.service`) intégrant une relance automatique (`Restart=always`) et une limite stricte de consommation mémoire (`MemoryMax=1.5G`) pour éviter toute saturation de la RAM.
- **Navigateur Kiosk :** Chromium exécuté en mode plein écran autonome (`--kiosk`, `--no-first-run`, `--disable-infobars`) pointé vers le serveur de supervision local.
- **Compte de supervision :** Compte CheckMK dédié, en lecture seule, session maintenue active durablement.

---

## 📂 Structure du Dépôt

```text
├── README.md               # Présentation du projet et architecture
├── procedure.md            # Guide technique et de dépannage complet
├── scripts/
│   └── chromium-wallboard.sh           # Script principal de lancement du navigateur
└── systemd/
    └── chromium-wallboard.service      # Fichier de service utilisateur systemd
```

---

## 🚀 Guide de Déploiement Rapide

1. **Copier les fichiers** sur la machine cible dans le répertoire de l'utilisateur `checkmk-kiosk`.
2. **Configurer l'authentification automatique** dans `/etc/gdm3/custom.conf`.
3. **Activer le service Systemd utilisateur :**
   ```bash
   systemctl --user enable chromium-wallboard.service
   systemctl --user start chromium-wallboard.service
   ```
4. **Redémarrer le poste** pour valider le démarrage à froid et l'affichage automatique du kiosque.

*Pour consulter la procédure détaillée pas à pas, référez-vous au fichier [procedure.md](./procedure.md).*

---

## 🎓 Ce que j'ai appris sur ce projet

- Diagnostiquer un blocage TLS causé par l'inspection SSL d'un pare-feu d'entreprise, et corriger l'import d'un certificat racine (y compris l'étape souvent oubliée : redémarrer `snapd` pour qu'il prenne en compte le nouveau certificat)
- Adapter une première ébauche de procédure (pensée pour Ubuntu Server, Xorg + Openbox minimal) vers le véritable besoin exprimé au cahier des charges : Ubuntu Desktop LTS avec GNOME complet et autologin
- Diagnostiquer un bug de détection de processus (`pgrep -x "chromium"` ne correspondant jamais, le binaire s'exécutant en réalité sous le nom `chrome`) ayant provoqué une relance en boucle et une saturation mémoire — corrigé en migrant vers un service systemd natif avec limite mémoire explicite
- Configurer une machine en IP fixe en coordination avec l'administrateur réseau, et diagnostiquer un conflit d'adresses résiduel après la bascule DHCP → IP fixe

## 🖼️ Résultat

[À COMPLÉTER — photo finale du wallboard installé au bureau]
