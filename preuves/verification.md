# Verification du projet

## 1. Jeu initial de quatre articles

**Commande :**

```powershell
docker volume create stock-resultats
docker run --rm -v "${PWD}/entrees:/entrees:ro" -v stock-resultats:/resultats analyse-stock:v1
```

**Sortie :**

```text
References : 4
Unites : 16
Valeur totale : 670.00 EUR
References en stock faible : 3
References en rupture : 1
```

**Explication :** Le programme lit les quatre articles de `entrees/stock.csv`, calcule les statistiques et écrit le rapport dans le volume Docker. Les 16 unités valent au total 670,00 EUR.

## 2. CSV modifie sans reconstruction

**Modification de `entrees/stock.csv` pour le test :**

```bash
@"
article,quantite,prix_unitaire
Clavier,5,25.00
Souris,2,15.00
Ecran,1,180.00
Cable,0,8.00
"@ | Set-Content .\entrees\stock.csv -Encoding utf8
```

**Commande (sans `docker build`) :**

```powershell
docker run --rm -v "${PWD}/entrees:/entrees:ro" -v stock-resultats:/resultats analyse-stock:v1
```

**Sortie :**

```text
References : 4
Unites : 8
Valeur totale : 335.00 EUR
References en stock faible : 3
References en rupture : 1
```

**Explication :** La modification du fichier du PC est prise en compte sans reconstruire l'image, car les entrées sont montées depuis l'hôte.

## 3. Tentative d'ecriture dans entrees en lecture seule

**Commande :**

```powershell
docker run --rm --entrypoint /bin/sh -v "${PWD}/entrees:/entrees:ro" analyse-stock:v1 -c "echo test > /entrees/test.txt"
Test-Path .\entrees\test.txt
```

**Sortie :**

```text
/bin/sh: 1: cannot create /entrees/test.txt: Read-only file system

False
```

**Explication :** Le montage `:ro` interdit l'écriture dans `entrees`. Le fichier `test.txt` ne doit pas être créé.

## 4. Rapport relu depuis un autre conteneur avec le meme volume nomme

**Commande :**

```powershell
docker run --rm --entrypoint /bin/cat -v stock-resultats:/resultats:ro analyse-stock:v1 /resultats/bilan-stock.txt
```

**Sortie (ces valeurs a cause du test 2):**

```text
References : 4
Unites : 8
Valeur totale : 335.00 EUR
References en stock faible : 3
References en rupture : 1
```

**Explication :** Un nouveau conteneur peut lire le rapport produit précédemment, car le volume nommé conserve les fichiers après la suppression du premier conteneur.

## 5. Entree absente, vide ou invalide

Le Dockerfile utilise `CMD` : pour passer `--entree`, on appelle explicitement Python et le script après le nom de l'image.

**Cas A — Fichier absent :**

```powershell
docker run --rm -v "${PWD}/entrees:/entrees:ro" -v stock-resultats:/resultats analyse-stock:v1 python3 /app/stock.py --entree /entrees/inexistant.csv
```

**Sortie :**

```text
Erreur : [Errno 2] No such file or directory: '/entrees/inexistant.csv'
```

**Cas B — CSV vide :**

```powershell
New-Item -ItemType File -Force -Path .\entrees\vide.csv
docker run --rm -v "${PWD}/entrees:/entrees:ro" -v stock-resultats:/resultats analyse-stock:v1 python3 /app/stock.py --entree /entrees/vide.csv
```

**Sortie :**

```text
Erreur : Colonnes : article,quantite,prix_unitaire
```

**Cas C — CSV invalide :**

```powershell
Set-Content .\entrees\invalide.csv "article,quantite,prix_unitaire`nClavier,-2,25.00"
docker run --rm -v "${PWD}/entrees:/entrees:ro" -v stock-resultats:/resultats analyse-stock:v1 python3 /app/stock.py --entree /entrees/invalide.csv
```

**Sortie :**

```text
Erreur : Nom requis, quantite et prix non negatifs.
```

**Explication :** Le programme refuse un fichier absent, un fichier sans colonnes et un article avec une quantité négative. Ces erreurs sont affichées sur la sortie d'erreur.

**Nettoyage après les tests :**

```powershell
Remove-Item .\entrees\vide.csv, .\entrees\invalide.csv
```

Remettre ensuite les quatre articles et leurs quantités initiales dans `entrees/stock.csv`.

```bash
@"
article,quantite,prix_unitaire
Clavier,10,25.00
Souris,4,15.00
Ecran,2,180.00
Cable,0,8.00
"@ | Set-Content .\entrees\stock.csv -Encoding utf8
```
