# Vérificateur de Licences FFSU

Outil automatisé pour vérifier les licences sportives FFSU (Fédération Française du Sport Universitaire). Développé dans le cadre du **Challenge Centrale Lyon 2026** pour traiter les inscriptions de centaines de sportifs efficacement.

## Contexte

En tant que co-responsable du Challenge Centrale Lyon (3 000 participants), la vérification manuelle des licences FFSU était une tâche chronophage. Ce script automatise l'ensemble du processus : connexion au portail FFSU, recherche de chaque sportif par nom, correspondance avec le numéro de licence, et export d'un rapport de statuts.

## Fonctionnalités

- Connexion automatique au portail de gestion FFSU
- Traitement par lot depuis un fichier CSV
- Correspondance par nom + numéro de licence
- Gestion des cas particuliers : nom introuvable, résultats ambigus, numéro de licence manquant
- Export des résultats dans un nouveau CSV avec les statuts

## Stack technique

- Python 3.10+
- [Playwright](https://playwright.dev/python/) – automatisation du navigateur
- [Pandas](https://pandas.pydata.org/) – traitement CSV

## Installation

```bash
pip install playwright pandas
playwright install chromium
```

## Utilisation

1. Place ta liste de sportifs dans `FFSU.csv` avec au minimum ces colonnes :

| Nom | Numero |
|-----|--------|
| Dupont | 0123456 |
| Martin | Aucune license |

2. Configure le script (en haut de `main.py`) :

```python
USERNAME = "ton_identifiant_ffsu"
PASSWORD = "ton_mot_de_passe"
SEP = ","      # séparateur CSV ("," ou ";")
NUM_LEN = 7    # longueur du numéro de licence (complété par des zéros)
```

3. Lance le script :

```bash
python FFSU.py
```

4. Les résultats sont écrits dans `FFSU_resultats.csv` :

| Nom | Numero | Statut | NumeroTrouve |
|-----|--------|--------|--------------|
| Dupont | 0123456 | Active | 0123456 |
| Martin | Aucune license | Aucune Active (nom trouvé) | |

## Valeurs de statut

| Statut | Signification |
|--------|---------------|
| `Active` | Licence valide trouvée et correspondante |
| `Non active` | Licence trouvée mais non active |
| `Nom introuvable` | Nom absent de la base FFSU |
| `Numéro non trouvé (mais nom trouvé)` | Nom trouvé mais le numéro de licence ne correspond pas |
| `Ambigu (plusieurs Active)` | Plusieurs licences actives trouvées pour ce nom |
| `Aucune Active (nom trouvé)` | Nom trouvé mais aucune licence active |

## Remarques

- Le navigateur s'ouvre en mode visible (`headless=False`) — ne pas interagir avec lui pendant l'exécution
- La vitesse réseau influence le temps de traitement ; le script attend le chargement complet de chaque page
- Les identifiants sont écrits en dur pour simplifier l'usage — préférer des variables d'environnement en production
