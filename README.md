# 大相撲 — Sumo Dashboard
Power BI 大相撲 rikishi actifs au banzuke

## Introduction
Dashboard Power BI analyse des rikishi actifs au dernier banzuke
Données chargées API à partir du site sumo-api.com

## Pages
- **Banzuke** — vue globale des lutteurs actifs
- **外国人力士** — analyse des lutteurs étrangers
<!-- amélioration : page heya et rikishi dédiées pour recherche d'un lutteur en particulier -->

## Construction du DB
- Source : API REST sumo-api.com (JSON)
- Transformation : Power Query M
- Modélisation : schéma en étoile
- Calculs : DAX (AVERAGEX, CALCULATE, ALL, DIVIDE, MEDIAN, liens WEB ...)
- Outil : Power BI Desktop
- Images : https://www.irasutoya.com/

## Modèle de données
image modele etoile

## Aperçu
images onglets
