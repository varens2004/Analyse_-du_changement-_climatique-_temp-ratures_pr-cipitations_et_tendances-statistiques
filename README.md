# Analyse du changement climatique : températures, précipitations et tendances statistiques

> Projet de data analyse de bout en bout : évolution des températures, analyse des précipitations, comparaison de régions et détection de tendances statistiques, à partir de données publiques réelles.
> **Outils : Python (pandas, SciPy), statistiques (Mann-Kendall, pente de Sen, régression segmentée), SQL (SQLite), Power BI.**

![Réchauffement mondial](dashboards/dashboard_1.png)

## 1. Contexte et questions
Une collectivité, un gestionnaire d'eau ou un assureur doit savoir **si le climat local change réellement, à quelle vitesse, et où agir en priorité**. Le projet répond à quatre questions :

1. Comment les températures ont-elles évolué (monde et 7 villes australiennes) ?
2. Comment les précipitations ont-elles évolué (4 villes à longues séries) ?
3. Les régions se comportent-elles de la même façon ?
4. Quelles tendances sont **statistiquement fiables**, et lesquelles ne sont que du bruit ?

## 2. Données (toutes publiques)
| Jeu | Contenu | Lien |
|---|---|---|
| **Températures et pluies journalières** de villes australiennes (Bureau of Meteorology) | 7 villes, Tmax/Tmin depuis 1910, pluie depuis 1858 ; 528 278 + 179 273 lignes | <https://github.com/rfordatascience/tidytuesday/tree/main/data/2020/2020-01-07> (origine : <http://www.bom.gov.au/climate/data/>) |
| **Anomalies de température par zone** (NASA GISTEMP) | monde, hémisphères, 8 bandes de latitude, 1880-2022 | <https://github.com/rfordatascience/tidytuesday/tree/main/data/2023/2023-07-11> (origine : <https://data.giss.nasa.gov/gistemp/>) |
| **Anomalie mondiale annuelle** (NASA GISTEMP) | 1880-2025 | <https://github.com/datasets/global-temp> |
| **CO2 à Mauna Loa** (NOAA) | mensuel, 1958-2026 | <https://github.com/datasets/co2-ppm> (origine : <https://gml.noaa.gov/ccgg/trends/>) |

**Poids des données : ~36 Mo bruts réduits à moins de 1 Mo** après agrégation (journalier → mensuel → annuel). Les fichiers bruts sont téléchargés automatiquement par le script 01 (dossier ignoré par git) ; les tables nettoyées sont fournies dans `data/processed/`.

## 3. Structure du dépôt
```
├── python/
│   ├── 01_nettoyage_donnees.py     # téléchargement, audit, nettoyage, agrégation -> CSV + base SQLite
│   ├── 02_analyse_tendances.py     # tendances, tests, comparaisons, ruptures, CO2
│   └── 03_dashboards.py            # 3 tableaux de bord (PNG + PDF)
├── sql/requetes_sql.sql            # 16 requêtes (19 instructions) testées sur SQLite
├── powerbi/                        # CSV prêts à importer + guide (mesures DAX, pages)
├── dashboards/                     # dashboard_1..3.png et PDF
├── data/processed/                 # tables nettoyées (mensuel, annuel, monde, zones) et climat.db
└── outputs/                        # tables de résultats (CSV) et resultats_cles.json
```
```bash
pip install -r requirements.txt
python python/01_nettoyage_donnees.py
python python/02_analyse_tendances.py
python python/03_dashboards.py
sqlite3 data/processed/climat.db < sql/requetes_sql.sql
```

## 4. Nettoyage (journal : `outputs/tables/audit_qualite.csv`)
| Problème | Lignes | Décision |
|---|---:|---|
| Températures manquantes | 3 465 | Supprimées, **non imputées** (une valeur imputée lisserait artificiellement les tendances) |
| Pluies manquantes (NA) | 11 760 | Supprimées : **un NA n'est pas un 0 mm** |
| Cumuls de pluie sur plusieurs jours | 996 | Gardés pour les totaux, exclus des extrêmes journaliers |
| Doublons, Tmin > Tmax | 0 | Contrôlés |
| Années avec < 90 % de jours | 18 (température), 19 (pluie) | Conservées mais marquées invalides, **exclues des tendances** |
| Stations de pluie multiples (Brisbane) ou trop courtes (Canberra, 2008-2020) | – | Exclues des analyses de pluie |

Jeu final : **709 années-ville de température** (7 villes, 1910-2018 ; Brisbane depuis 1950) et **388 années-ville de pluie** (Sydney 1859-2019, Adelaide 1884-2014, Melbourne 1965-2019, Perth 1968-2019). Les températures s'arrêtent en mai 2019 : 2019 est donc exclue.

Indicateurs construits : anomalies vs normale 1961-1990 de chaque ville ; **jours chauds** et **nuits chaudes** (part des jours dont Tmax / Tmin dépasse le 90e centile 1961-1990 du même mois, soit 10 % attendus sans changement) ; jours ≥ 35 °C ; jours de pluie (≥ 1 mm), jours de forte pluie (≥ 25 mm), intensité, maximum journalier, plus longue séquence sèche.

## 5. Méthodes statistiques
- **Pente de Sen** avec intervalle de confiance à 95 % : robuste aux valeurs extrêmes, exprimée en unité par décennie.
- **Test de Mann-Kendall** (non paramétrique, correction des ex aequo) **et sa version pré-blanchie TFPW**, qui corrige l'autocorrélation. **Une tendance n'est déclarée significative que si les deux tests donnent p < 0,05.**
- **Régression segmentée** pour dater une accélération (indicatif) ; **test de Pettitt** pour les sauts de moyenne ; tests de Welch et de Mann-Whitney entre périodes.
- Les pentes de la régression linéaire (SQL, requête Q11) concordent avec celles de Python (écart ≤ 0,001 °C/décennie).

## 6. Résultats

### 6.1 Monde : un réchauffement qui s'accélère
- Anomalie moyenne 2016-2025 : **+1,01 °C** par rapport à 1951-1980. Record : 2024 (+1,28 °C) ; **les 10 années les plus chaudes sont toutes postérieures à 2010**.
- Pente de Sen : **+0,036 °C/décennie avant 1975** contre **+0,205 °C/décennie depuis 1976** (+0,258 depuis 2000). La régression segmentée place la cassure en **1975**. Sur 1880-2025 : +0,080 °C/décennie (IC 95 % 0,072 à 0,089).
- **Amplification arctique** (1976-2022) : la bande 64°N-90°N se réchauffe de **+0,64 °C/décennie** (IC 0,54-0,73), soit 4,4 fois la zone tropicale (+0,14), et 6 fois l'hémisphère sud (+0,11) ; l'hémisphère nord atteint +0,28.
- **CO2** : 316 ppm (1959) → **427 ppm** (2025) ; la hausse passe de +0,96 ppm/an (1959-75) à **+2,49 ppm/an** (2010-25). La corrélation avec la température mondiale est forte (r = 0,97 en niveaux ; ≈ +1,1 °C pour +100 ppm). Sur les variations d'une année à l'autre elle tombe à r = 0,32 (p = 0,009) : la variabilité naturelle (El Niño…) domine à court terme. **Deux séries qui montent ensemble ne prouvent pas la causalité à elles seules** ; cette relation empirique n'est pas une mesure de sensibilité climatique.

![Températures](dashboards/dashboard_2.png)

### 6.2 Températures : les 7 villes se réchauffent, mais pas au même rythme
Pente de Sen 1950-2018 (période commune), toutes **significatives** (Mann-Kendall et TFPW) :

| Ville | °C / décennie (IC 95 %) | Hausse 1961-90 → 1989-2018 | Jours chauds (%) | Nuits chaudes (%) | Jours ≥ 35 °C |
|---|---|---:|---|---|---|
| **Canberra** | **0,30** (0,24-0,36) | +0,84 °C | 9,7 → **15,9** | 9,6 → 13,0 | 4,8 → 7,6 |
| Perth | 0,23 (0,17-0,29) | +0,58 °C | 9,8 → 12,1 | 9,6 → **14,0** | 27,4 → 31,2 |
| Melbourne | 0,19 (0,15-0,23) | +0,42 °C | 9,6 → 11,8 | 9,7 → 11,8 | 7,8 → 9,4 |
| Adelaide | 0,19 (0,14-0,25) | +0,53 °C | 9,9 → 13,3 | 9,8 → 13,3 | 18,6 → **23,0** |
| Port Lincoln | 0,18 (0,14-0,23) | +0,40 °C | 9,6 → 13,0 | 9,6 → 10,7 | 7,3 → 8,3 |
| Brisbane | 0,15 (0,10-0,19) | +0,28 °C | 9,6 → 13,4 | 9,6 → 11,5 | 0,8 → 0,9 |
| Sydney | 0,14 (0,10-0,17) | +0,30 °C | 9,7 → 11,1 | 9,5 → **13,9** | 5,7 → 4,6 |

- **Hétérogénéité** : le rythme de Canberra (intérieur) est **2 fois** celui du monde sur la même période (+0,147 °C/décennie en 1950-2018) ; Sydney et Brisbane (côtes) en sont proches. Les hausses 1961-90 → 1989-2018 sont toutes significatives (Welch, p ≤ 0,006).
- **Accélération** : sur toute la série, Sydney et Adelaide se réchauffent de seulement +0,09 °C/décennie, contre +0,14 et +0,19 depuis 1950.
- **Jour ou nuit** : à Port Lincoln, Canberra et Adelaide les maximales montent plus vite que les minimales (0,29 vs 0,06 ; 0,36 vs 0,24 ; 0,24 vs 0,15) ; à **Brisbane et Sydney c'est l'inverse** (minimales : 0,20 et 0,16 ; maximales : 0,08 et 0,12) : le risque est surtout nocturne (Sydney : 14 % de nuits chaudes au lieu de 9,5 %).
- **Saisons** : 27 des 28 combinaisons ville × saison montrent une hausse significative ; les plus fortes sont le printemps (+0,38 °C/décennie) et l'été (+0,36) à Canberra, et le printemps à Adelaide (+0,30).
- Les anomalies annuelles des villes sont corrélées entre 0,34 et 0,91 : le signal régional est partagé, mais pas uniforme.

![Précipitations](dashboards/dashboard_3.png)

### 6.3 Précipitations : pas de signal uniforme
| Ville (période) | Tendance du cumul | Verdict | Autres signaux |
|---|---|---|---|
| **Melbourne** (1965-2019) | **−5,8 %/décennie** (−7,6 % sur 1968-2014) ; 1991-2014 vs 1968-90 : **−18 %** (p = 0,009) | **Baisse significative** | −4,7 jours de pluie/décennie ; automne **−12 %/décennie** (seule baisse saisonnière significative) ; saut de moyenne en 1999 (−21 %, p = 0,007) |
| **Adelaide** (1884-2014) | −1,1 %/décennie (−5,3 % sur 1968-2014) | Baisse significative, limite (p = 0,043 / 0,034) | −9,7 % entre 1968-90 et 1991-2014 (non significatif, p = 0,09) |
| **Perth** (1968-2019) | −2,7 %/décennie | Non significative | **−4,1 jours de pluie/décennie** et +0,23 mm par jour de pluie (significatifs) : moins de jours, pluies plus intenses |
| **Sydney** (1859-2019) | −0,1 %/décennie | **Aucune tendance** | variabilité forte (écart-type = 26 % de la moyenne) |

- **Aucune tendance significative** sur les extrêmes (jours ≥ 25 mm, maximum journalier, plus longue séquence sèche) dans les 4 villes.
- La variabilité d'une année à l'autre (18 à 26 % de la moyenne) est du même ordre que les changements : sur 50 ans, une baisse de 5 % par décennie ne ressort qu'à Melbourne.

## 7. Décisions que ce projet permet de prendre
| # | Décision | Preuve | Pour qui |
|---|---|---|---|
| 1 | **Revoir la référence climatique** : remplacer la normale 1961-1990 par 1989-2018 dans les seuils de dimensionnement (énergie, bâtiments, alertes) | Les 7 villes ont gagné de +0,28 à +0,84 °C, toutes significatives | Services techniques, assureurs |
| 2 | **Prioriser les plans canicule** : Canberra (jours chauds × 1,6), Adelaide et Perth (jours ≥ 35 °C : +4,4 et +3,8 par an) | Pentes de Sen les plus fortes, hausses des jours extrêmes | Santé publique, collectivités |
| 3 | **Traiter la chaleur nocturne à Sydney et Brisbane** (végétalisation, ventilation, rafraîchissement des logements) plutôt que la chaleur diurne | Minimales qui montent plus vite que les maximales ; Sydney : nuits chaudes × 1,5, jours ≥ 35 °C en baisse | Urbanisme, bailleurs |
| 4 | **Planifier la ressource en eau à Melbourne (et surveiller Adelaide)** : diversification, gestion de la demande | Seules baisses significatives et robustes (−18 % ; −5 à −8 %/décennie) | Gestionnaires d'eau |
| 5 | **Perth : adapter le stockage et le drainage, pas seulement la rareté** | Moins de jours de pluie (−4,1/décennie) et pluies plus intenses (+0,23 mm/jour) | Gestionnaires d'eau et d'assainissement |
| 6 | **Ne pas décréter de tendance de pluie à Sydney** : planifier sur la variabilité (CV 26 %) | Pente −0,1 %/décennie sur 160 ans, p = 0,84 | Tous |
| 7 | **Piloter avec des indicateurs annuels** (% jours chauds, % nuits chaudes, cumul de pluie) validés par Mann-Kendall + TFPW pour éviter les fausses alertes | Méthode du projet, reproductible avec les scripts | Observatoires locaux |

*Ce projet appuie des décisions d'**adaptation** locale ; il ne chiffre pas les politiques d'atténuation (émissions).*

## 8. Limites (à lire avant de publier)
1. **Séries brutes non vérifiées pour l'homogénéité** : déplacements de stations, changement d'instruments et urbanisation (Sydney, Observatory Hill) peuvent biaiser les tendances locales.
2. **Sept villes australiennes ne représentent pas le monde** : le volet mondial vient de jeux NASA/NOAA distincts ; aucune conclusion planétaire ne vient des villes.
3. **Précipitations** : seulement 4 villes ; **Melbourne n'a aucune donnée pour 1993-1998**, ce qui fragilise la datation de la baisse (les valeurs d'avant 1993 sont comparées à celles d'après 1998). Plusieurs stations ont des trous.
4. **Tests multiples** : 254 tendances testées ; avec un seuil de 5 %, environ 1 sur 20 peut sortir « significative » par hasard. Les cas limites (Adelaide, p = 0,043 ; Perth, intensité, p = 0,04) sont à confirmer avec des données récentes.
5. La **régression segmentée** et le **test de Pettitt** sont indicatifs : la recherche de la meilleure année de cassure rend leurs p-values optimistes, et Pettitt tend à désigner le milieu d'une série qui monte en continu.
6. Les données de température s'arrêtent en mai 2019 (jeu source) ; les années récentes, plus chaudes, ne sont donc pas incluses pour les villes.
7. Corrélation CO2-température : voir 6.1 ; ce n'est pas une preuve de causalité à elle seule.

## 9. Pistes d'amélioration
Ajouter des séries homogénéisées (réseau de référence du BoM) jusqu'en 2025, élargir à d'autres continents, ajouter une analyse d'extrêmes par valeurs extrêmes généralisées (GEV), et relier les tendances aux oscillations naturelles (ENSO, SAM).

## 10. Auteur
Espérant ODJO – étudiant ingénieur informatique (EILCO, Calais), recherche d'alternance en Data Analyst / Data Engineer.
