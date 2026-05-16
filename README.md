# 大相撲 — Sumo Dashboard
Power BI 大相撲 rikishi actifs au banzuke

## Introduction
- Dashboard Power BI analyse des rikishi actifs au dernier banzuke
- Données chargées API à partir du site sumo-api.com

## Pages
- **Banzuke** vue globale des lutteurs actifs
- **Gaijin** analyse des lutteurs étrangers
<!-- amélioration : page heya et rikishi dédiées pour recherche d'un lutteur en particulier -->

## Construction du DB
- Source : API REST sumo-api.com (JSON)
- Transformation : Power Query M
- Modélisation : schéma en étoile
- Calculs : DAX (AVERAGEX, CALCULATE, ALL, DIVIDE, MEDIAN, liens WEB ...)
- Outil : Power BI Desktop
- Images : https://www.irasutoya.com/

## Aperçu
### onglet Banzuke
<img src="assets/banzuke.png" width="800" alt="onglet Banzuke"/>
### onglet Gaijin
<img src="assets/gaijin.png" width="800" alt="onglet gaijin"/>

## Modèle de données
### Modèle en étoile
<img src="assets/etoile.png" width="800" alt="modèle en étoile"/>

### Dépendances (PowerQuery)
<img src="assets/dependances" width="800" alt="dependances"/>
