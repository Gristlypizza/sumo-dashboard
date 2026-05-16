# Mesures DAX
> **Table** : _Mesures  
---

##  Dim_Region
- Colonne calculée IsGaijin pour calculer le nombre de lutteurs étrangers selon la colonne type du fichier Excel SumoRanking onglet shusshin importé
- si la colonne [Pays] = "Japon" alors [type] = "Préfécture" sinon [type] "Pays" (donc pays étranger)
- utilisée comme filtre dans les mesures ce qui évite de répeter "Pays" si besoin de changer la nomination
### IsGaijin
```
IsGaijin = Dim_Region[type] = "Pays"
```

## _Mesure Dossier Physique
> Médiane plutôt que moyenne en raison des écarts importants sur les 600+ lutteurs de rangs débutants à professionnels

### Poids Med
> Calcule la médiane de la taille
```
Poids Med = MEDIAN(Fact_Rikishi[weight])
```

### Taille Med
> Calcule la médiane du poids
```
Taille Med = MEDIAN(Fact_Rikishi[height])
```

### Poids Med Gaijin
> Calcule la médiane du poids pour les Gaijins
> Mesure séparée pour pouvoir être utilisée à part dans les cartes KPI
```
Poids Med Gaijin = 
CALCULATE(
    MEDIAN(Fact_Rikishi[weight]),
    Dim_Region[IsGaijin] = TRUE()
)
```

### Taille Med Gaijin
> Calcule la médiane de la taille pour les Gaijins
> Mesure séparée pour pouvoir être utilisée à part dans les cartes KPI
```
Taille Med Gaijin = 
CALCULATE(
    MEDIAN(Fact_Rikishi[height]),
    Dim_Region[IsGaijin] = TRUE()
)
```

### Ratio Moyen
```
Ratio Moyen = 
AVERAGEX(
    FILTER(Fact_Rikishi, Fact_Rikishi[height] <> 0 && Fact_Rikishi[height] <> BLANK()),
    DIVIDE(Fact_Rikishi[weight], Fact_Rikishi[height])
)
```

## _Mesure Dossier Quantités
### % gaijin 
> le pourcentage de lutteurs étrangers, basés sur la colonne calculée isGaijin présent dans la table de dimension Dim_Region
```
% gaijin = 
VAR Gaijin = CALCULATE(
    [Nb rikishi],
    Dim_Region[IsGaijin] = TRUE()
)
RETURN

DIVIDE( Gaijin, [Nb rikishi], 0)
```

### Nb de pays
> Nb de pays représentés par les lutteurs etrangers
```
Nb de pays = 
CALCULATE(
    DISTINCTCOUNT(Dim_Region[pays]),
    Dim_Region[IsGaijin] = TRUE()
)
```

### Nb gaijin
> Nb total de gaijin rikishi
```
Nb gaijin = 
CALCULATE(
    [Nb rikishi],
    Dim_Region[IsGaijin] = TRUE()
)
```

### Nb rikishi
> nombre total de rikishi au banzuke actuel
> DISTINCTCOUNT plutot que COUNTROWS pour prévenir doublon d'id
```
Nb rikishi = DISTINCTCOUNT(Fact_Rikishi[id])
```

## _Mesures Dossier Temporel
### Age Debut
```
Age Debut = 
AVERAGEX(
    Fact_Rikishi,
    INT(DATEDIFF(Fact_Rikishi[birthDate], Fact_Rikishi[StartDate], DAY) / 365.25)
)
```

### Age moyen
```
Age moyen = 
AVERAGEX(
    Fact_Rikishi,
    INT(DATEDIFF(Fact_Rikishi[birthDate], TODAY(), DAY) / 365.25)
)
```

### XP moyen
> calcul de l'anciennetée moyenne depuis le début de carrière
```
XP moyen = 
AVERAGEX(
    Fact_Rikishi,
    INT(DATEDIFF(Fact_Rikishi[StartDate], TODAY(), DAY) / 365.25)
)
```

### XP moyen Gaijin
```
XP moyen Gaijin = 
CALCULATE(
    AVERAGEX(
        Fact_Rikishi,
        INT(DATEDIFF(Fact_Rikishi[StartDate], TODAY(), DAY) / 365.25)
        ),
    Dim_Region[IsGaijin] = TRUE()
)
```

## Table du temps 
> relation de 1-* vers Fact_Rikishi sur la colonne [StartDate] (année début carrière sumo)
> marquée comme table de dates
```
Dim_Date = 
VAR DateDebut = DATE(1950, 1, 1)
VAR DateFin = DATE(2026, 12, 31)
RETURN
ADDCOLUMNS (
    CALENDAR(DateDebut, DateFin),
    "Année", YEAR([Date]),
    "Mois No", MONTH([Date]),
    "Mois Nom", FORMAT([Date], "MMMM"),
    "Trimestre", "T" & QUARTER([Date]),
    "AnnéeMois", FORMAT([Date], "YYYYMM")
)
```