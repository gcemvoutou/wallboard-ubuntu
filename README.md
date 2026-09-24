# Wallboard de supervision CheckMK

## Contexte

Dans le cadre de mon alternance BTS SIO SISR au sein de la Direction des Systèmes d'Information d'une mairie, on m'a confié la transformation d'une TV inutilisée en panneau d'affichage permanent (« wallboard ») du dashboard CheckMK, pour rendre visible en continu l'état de la supervision des équipements réseau.

## Objectif

- Affichage 24h/24, 7j/7, sans aucune intervention humaine au quotidien
- Redémarrage et reprise entièrement autonomes en cas de coupure de courant ou de plantage
- Interface plein écran pure, sans barre d'adresse ni curseur visible

## Stack technique

| Élément | Choix |
|---|---|
| Matériel | Dell OptiPlex 3020M (Intel Core i3, 8 Go RAM) |
| Système | Ubuntu Desktop 24.04 LTS |
| Navigateur | Chromium (Snap), en mode kiosque |
| Résilience logicielle | Script watchdog maison (surveillance + relance auto) |
| Résilience matérielle | AC Recovery activé au BIOS |
| Compte de supervision | Compte CheckMK dédié, droits lecture seule |

## Ce que j'ai appris sur ce projet

- Diagnostiquer un blocage TLS causé par l'inspection SSL d'un pare-feu d'entreprise (import d'un certificat racine sur le système)
- Adapter une architecture initialement pensée pour Ubuntu Server (Xorg + Openbox minimal) vers Ubuntu Desktop (GNOME complet + autologin), suite à une contrainte imposée en cours de projet
- Configurer une machine en IP fixe en coordination avec l'administrateur réseau, et diagnostiquer un conflit d'adresses résiduel après la bascule DHCP → IP fixe
- Comprendre et résoudre un problème d'affichage lié à la négociation EDID entre le PC et l'écran (détaillé dans `PROCEDURE.md`)

## Résultat

[À COMPLÉTER — photo finale du wallboard installé au bureau]

## Documentation détaillée

Le déroulé technique complet (installation, kiosque, résilience, dépannage) est disponible dans [`PROCEDURE.md`](./PROCEDURE.md).

> Les adresses IP et noms de domaine présentés dans ce dépôt sont anonymisés (plages RFC 5737) — ce ne sont pas les valeurs réelles de l'infrastructure de production.
