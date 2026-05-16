# Requêtes Power Query M
===================================================
> **SRC_Rikishi** : source brute (table non chargée)
> **STG_Rikishi** : staging : nettoyage + jointures sur id (table non chargée)
> **Fact_Rikishi** : table finale chargée
> **Dimensions** : Les sources des 3 tables dimension sont chargées depuis un fichier Excel (3 onglets)
> **Sources locales** : Chemin à modifier dans la passe "Source" des dimension
===================================================

## SRC_Rikishi 
> **Source** : API REST sumo-api.com/api/rikishis
> **Rôle** : non chargée et base pour STG_Rikishi, backup de la table vierge sans modification

```
let
    Url = "https://www.sumo-api.com/api/rikishis",
    Réponse = Web.Contents(Url),
    Json = Json.Document(Réponse),
    Records = Json[records],
    Table = Table.FromList(Records, Splitter.SplitByNothing(), null, null, ExtraValues.Error),
    Étendu = Table.ExpandRecordColumn(Table, "Column1",
        {"id", "sumodbId", "nskId", "shikonaEn", "shikonaJp",
         "currentRank", "heya", "birthDate", "shusshin",
         "height", "weight", "debut", "updatedAt"})
in
    Étendu
```

## STG_Rikishi
> **Source** : API REST sumo-api.com/api/rikishis (duplication SRC)
> **Rôle** : nettoyage, transformation et jointures avec les 3 dimensions (Rank, Region, Heya)
> **Etapes** : 
    - * Filtrage des rikishi sans rang (currentRank null)
    - * Simplification du rang afin de pouvoir l'utiliser en tant que clef sur la table Dim_Rank (ex: "Maegashira 10 East" > "M10")
    - * Jointure gauche avec Dim_Rank sur la clé Rank
    - * Nettoyage du shusshin pour jointure avec Dim_Region
    - * Jointure avec Dim_Heya sur le nom de l'écurie

```
let
    Url = "https://www.sumo-api.com/api/rikishis",
    Réponse = Web.Contents(Url),
    Json = Json.Document(Réponse),
    Records = Json[records],
    Table = Table.FromList(Records, Splitter.SplitByNothing(), null, null, ExtraValues.Error),
    Étendu = Table.ExpandRecordColumn(Table, "Column1",
        {"id", "sumodbId", "nskId", "shikonaEn", "shikonaJp",
         "currentRank", "heya", "birthDate", "shusshin",
         "height", "weight", "debut", "updatedAt"}),

    // Filtrage des rikishi sans rang (10-05-2026 : 1 résultat)
    #"rank <> null" = Table.SelectRows(Étendu, each ([currentRank] <> null)),

    // Simplification du rang pour jointure avec Dim_Rank
    // Ex: "Maegashira 10 East" > "M10", "Juryo 3 West" > "J3"
    #"rank simplifié" = Table.AddColumn(#"rank <> null", "Rank", each
        let
            rank = [currentRank],
            clean = Text.Trim(Text.Replace(Text.Replace(rank, " East", ""), " West", "")),
            result = 
                if Text.StartsWith(clean, "Yokozuna")   then "Y"
                else if Text.StartsWith(clean, "Ozeki")      then "O"
                else if Text.StartsWith(clean, "Sekiwake")   then "S"
                else if Text.StartsWith(clean, "Komusubi")   then "K"
                else if Text.StartsWith(clean, "Maegashira ") then
                    "M" & Text.AfterDelimiter(clean, "Maegashira ")
                else if Text.StartsWith(clean, "Juryo ") then
                    "J" & Text.AfterDelimiter(clean, "Juryo ")
                else if Text.StartsWith(clean, "Makushita ") then
                    "Ms" & Text.AfterDelimiter(clean, "Makushita ")
                else if Text.StartsWith(clean, "Sandanme ") then
                    "Sd" & Text.AfterDelimiter(clean, "Sandanme ")
                else if Text.StartsWith(clean, "Jonidan ") then
                    "Jd" & Text.AfterDelimiter(clean, "Jonidan ")
                else if Text.StartsWith(clean, "Jonokuchi ") then
                    "Jk" & Text.AfterDelimiter(clean, "Jonokuchi ")
                else if Text.StartsWith(clean, "Maezumo") then "Mz"
                else clean
        in
            result),

    #"Type modifié" = Table.TransformColumnTypes(#"rank simplifié", {
        {"birthDate",    type datetime},
        {"updatedAt",    type datetime},
        {"id",           Int64.Type},
        {"sumodbId",     Int64.Type},
        {"nskId",        Int64.Type},
        {"shikonaEn",    type text},
        {"shikonaJp",    type text},
        {"currentRank",  type text},
        {"heya",         type text},
        {"shusshin",     type text},
        {"height",       Int64.Type},
        {"weight",       Int64.Type},
        {"debut",        type text},
        {"Rank",         type text}
    }),

    // Jointure avec Dim_Rank + développer l'ID
    #"fusion table Dim_Rank" = Table.NestedJoin(
        #"Type modifié", {"Rank"}, Dim_Rank, {"Rank"}, "Dim_Rank", JoinKind.LeftOuter),
    #"Dim_Rank développé" = Table.ExpandTableColumn(
        #"fusion table Dim_Rank", "Dim_Rank", {"RankID"}, {"RankID"}),

    // Nettoyage du shusshin (ne garder que la préfécture, suppression du niveau d'info Ville et des préfixes shi/ken)
    // Ex: "Toyama-ken, Toyama-shi" > "Toyama"
    #"nettoyage shusshin 1" = Table.AddColumn(
        #"Dim_Rank développé", "Texte avant le délimiteur",
        each Text.BeforeDelimiter([shusshin], "-"), type text),
    #"nettoyage shusshin 2" = Table.AddColumn(
        #"nettoyage shusshin 1", "Texte avant le délimiteur.1",
        each Text.BeforeDelimiter([Texte avant le délimiteur], ","), type text),

    // Jointure avec Dim_Region + développer l'ID
    #"fusion table Dim_Region" = Table.NestedJoin(
        #"nettoyage shusshin 2", {"Texte avant le délimiteur.1"},
        Dim_Region, {"region_clean"}, "Dim_Region", JoinKind.LeftOuter),
    #"Dim_Region développé" = Table.ExpandTableColumn(
        #"fusion table Dim_Region", "Dim_Region", {"RegionId"}, {"RegionId"}),

    // Conversion birthDate en Date simple (suppression de l'heure)
    #"format birthDate = Date" = Table.TransformColumns(
        #"Dim_Region développé", {{"birthDate", DateTime.Date, type date}}),

    // Jointure avec Dim_Heya + développer l'ID
    #"fusion table Dim_Heya" = Table.NestedJoin(
        #"format birthDate = Date", {"heya"}, Dim_Heya, {"Heya"}, "Dim_Heya", JoinKind.LeftOuter),
    #"Dim_Heya développé" = Table.ExpandTableColumn(
        #"fusion table Dim_Heya", "Dim_Heya", {"HeyaID"}, {"HeyaID"}),

    // Suppression colonnes inutiles + colonnes de jointure
    #"Colonnes supprimées" = Table.RemoveColumns(#"Dim_Heya développé", {
        "Texte avant le délimiteur.1",
        "Texte avant le délimiteur",
        "updatedAt",
        "heya",
        "shusshin",
        "Rank"
    })
in
    #"Colonnes supprimées"
```

## Fact_Rikishi
> 
- **Source** : STG_Rikishi (référence)
- **Rôle** : Table de faits finale chargée

```
let
    Source = STG_Rikishi
in
    Source
```

## Dim_Rank
> 
- **Source** : SumoRanking.xlsx Tableau1 (onglet rank)
- **Rôle** : classement des rangs avec hierarchie complète, division, statut, numéro du rang pour trier de Yokozuna à Jonokuchi

```
let
    Source = Excel.Workbook(File.Contents("D:\SumoRanking.xlsx"), null, true),
    Tableau1_Table = Source{[Item="Tableau1",Kind="Table"]}[Data],
    #"Type modifié" = Table.TransformColumnTypes(Tableau1_Table,{{"Rank", type text}, {"Rank Name", type text}, {"DivisionName", type text}, {"SpecialRankName", type text}, {"RankType", type text}, {"DivisionRank", Int64.Type}, {"RankingNumber", Int64.Type}, {"Statut professionnel", type text}, {"RankID", Int64.Type}})
in
    #"Type modifié"
```

## Dim_Heya
> **Source** : SumoRanking.xlsx - Tableau2 (onglet heya)
> **Rôle** : saisie manuelle d'infos complémentaires sur les écuries : ichimon, couleur mawashi HEX, année création

```
let
    Source = Excel.Workbook(File.Contents("D:\SumoRanking.xlsx"), null, true),
    Tableau2_Table = Source{[Item="Tableau2",Kind="Table"]}[Data],
    #"Type modifié" = Table.TransformColumnTypes(Tableau2_Table,{{"HeyaID", Int64.Type}, {"Heya", type text}, {"HeyaKanji", type text}, {"HeyaIchimon", type text}, {"CouleurMawashi", type text}, {"CouleurMawashiHex", type text}, {"DateCreation", Int64.Type}})
in
    #"Type modifié"
```

## Dim_Region
> **Source** : SumoRanking.xlsx - onglet shusshin
> **Rôle** : saisie manuelle d'infos complémentaires sur les régions (shusshin pays/ville d'origine du lutteur), lat/long, "type" pour différencier gaijin et japonais

```
let
    Source = Excel.Workbook(File.Contents("D:\SumoRanking.xlsx"), null, true),
    shusshin_Sheet = Source{[Item="shusshin",Kind="Sheet"]}[Data],
    #"En-têtes promus" = Table.PromoteHeaders(shusshin_Sheet, [PromoteAllScalars=true]),
    #"Type modifié" = Table.TransformColumnTypes(#"En-têtes promus",{{"region_clean", type text}, {"shusshin_original", type text}, {"nom_fr", type text}, {"pays", type text}, {"latitude", type number}, {"longitude", type number}, {"type", type text}}),
    #"Index ajouté" = Table.AddIndexColumn(#"Type modifié", "RegionId", 1, 1, Int64.Type)
in
    #"Index ajouté"
 ```