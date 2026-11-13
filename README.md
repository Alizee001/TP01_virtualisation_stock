# Analyse de stock avec Docker

## Auteurs

Alizée Chambon — travail individuel.

## Construction

Dans PowerShell, à la racine du projet :

```powershell
docker build -t analyse-stock:v1 .
```

Le `Dockerfile` utilise `ubuntu:24.04`, installe `python3`, définit `/app` comme dossier de travail, copie uniquement `stock.py` et lance `python3 stock.py` par défaut. Les données d'entrée ne sont pas intégrées à l'image : elles sont fournies par un montage au lancement.

Le programme calcule le nombre de références, le nombre d'unités, la valeur totale du stock, le nombre de références en stock faible (quantité de 1 à 10) et le nombre de références en rupture (quantité nulle). Il affiche le bilan et l'enregistre dans `/resultats/bilan-stock.txt`.

Le lecteur CSV vérifie les trois colonnes attendues (`article,quantite,prix_unitaire`) dans cet ordre, la présence d'un nom, des quantités entières et non négatives, des prix décimaux finis et non négatifs et la présence d'au moins un article. Une entrée incorrecte provoque un message d'erreur.

## Execution avec montages

Créer le dossier local de résultats :

```powershell
New-Item -ItemType Directory -Force -Path .\resultats
```

Exécuter le programme avec les dossiers du PC :

```powershell
docker run --rm -v "${PWD}/entrees:/entrees:ro" -v "${PWD}/resultats:/resultats" analyse-stock:v1
```

- `${PWD}/entrees` est le dossier du PC contenant `stock.csv` ; `/entrees` est son emplacement dans le conteneur. `:ro` impose la lecture seule.
- `${PWD}/resultats` est le dossier du PC où retrouver le rapport ; `/resultats` est son emplacement dans le conteneur.
- `--rm` supprime le conteneur après exécution, pas les fichiers du PC.

Pour conserver le rapport dans un volume Docker nommé :

```powershell
docker volume create stock-resultats
docker run --rm -v "${PWD}/entrees:/entrees:ro" -v stock-resultats:/resultats analyse-stock:v1
```

Le volume `stock-resultats` persiste après la fin du conteneur. Son rapport peut être relu depuis un autre conteneur :

```powershell
docker run --rm --entrypoint /bin/cat -v stock-resultats:/resultats:ro analyse-stock:v1 /resultats/bilan-stock.txt
```

## Tests et resultats

Les commandes, les sorties et les explications des cinq tests sont dans [preuves/verification.md](preuves/verification.md).


```text
References : 4
Unites : 16
Valeur totale : 670.00 EUR
References en stock faible : 3
References en rupture : 1
```

Le même texte est affiché dans le terminal et enregistré dans `bilan-stock.txt`.

## Reproduction apres git clone

Sur un PC avec Docker et Git, dans PowerShell :

```powershell
git clone URL_DU_DEPOT
cd TP01_virtualisation_stock
New-Item -ItemType Directory -Force -Path .\resultats
docker build -t analyse-stock:v1 .
docker run --rm -v "${PWD}/entrees:/entrees:ro" -v "${PWD}/resultats:/resultats" analyse-stock:v1
Get-Content .\resultats\bilan-stock.txt
```

Le dossier `resultats` est créé localement après le clonage ; le fichier `bilan-stock.txt` est produit par le programme.
