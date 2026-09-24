# Wallboard de supervision CheckMK

## Contexte

Le but du projet est d’installer un wallboard permettant d’afficher en permanence notre logiciel de monitoring Checkmk dans le bureau du Desk, afin que la personne présente puisse surveiller facilement l’état des équipements et détecter rapidement d’éventuels problèmes.

## Objectif

- Affichage 24h/24, 7j/7, sans aucune intervention humaine au quotidien
- Redémarrage et reprise entièrement autonomes en cas de coupure de courant ou de plantage
- Interface plein écran pure, sans barre d'adresse ni curseur visible

## Stack technique

| Élément | Choix |
|---|---|
| Matériel | Dell OptiPlex 3040 |
| Système | Ubuntu Desktop 24.04 LTS |
| Navigateur | Chromium (Snap), en mode kiosque |
| Résilience logicielle | Script watchdog maison (surveillance + relance auto) |
| Résilience matérielle | AC Recovery activé au BIOS |
| Compte de supervision | Compte CheckMK dédié, droits lecture seule |

## Ce que j'ai appris sur ce projet

- Configurer une machine Linux sous Ubuntu Desktop LTS à partir d’un Dell OptiPlex pour en faire un wallboard dédié.
- Configurer Chromium en mode kiosque pour afficher automatiquement le dashboard Checkmk en plein écran au démarrage.
- Mettre en place un démarrage automatique et désactiver la veille et le verrouillage afin de maintenir l’affichage en permanence.
- Configurer un script watchdog .
- Configurer l’accès à distance en SSH et une adresse IP fixe pour administrer la machine.
- Configurer la machine pour redémarrer automatiquement après une coupure de courant grâce à l’AC Recovery du BIOS.
- Diagnostiquer différents problèmes d’affichage et de connexion afin d’assurer le fonctionnement continu du wallboard.

## Résultat

<img src="images/wallboard.jpeg" width="70%">

## Documentation détaillée

Le déroulé technique complet (installation, kiosque, résilience, dépannage) est disponible dans [`PROCEDURE.md`](./PROCEDURE.md).

> Les adresses IP et noms de domaine présentés dans ce dépôt sont anonymisés (plages RFC 5737) — ce ne sont pas les valeurs réelles de l'infrastructure de production.
