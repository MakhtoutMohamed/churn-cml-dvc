# MLOps de bout en bout utilisant Azure Blob au lieu de S3

Ce projet applique le flux du TP:
- Git pour versionner le code,
- DVC pour versionner `data/` et `models/`,
- Azure Blob Storage comme remote DVC,
- GitHub Actions pour CI/CD,
- CML pour publier un rapport automatique.

## 1) Prerequis

- Python 3.11
- Compte GitHub
- Compte Azure avec un `Storage Account`
- Un container Blob dedie pour DVC `dvc-storage`

## 2) Creer le stockage Azure

1. Azure Portal -> `Storage accounts` -> creer un compte
2. Dans ce compte, creer un container Blob prive `dvc-storage`
3. Recuperer soit:
- `Connection string` ou
- `Account name` + `Account key`

Remote DVC cible:

```bash
azure://dvc-storage/churn-cml-dvc
```

## 3) Initialiser le projet en local

```bash
python -m venv .venv
dans windows : .\.venv\Scripts\Activate.ps1 #sur linux . .venv/Scripts/activate
pip install -r requirements.txt
pip install "dvc[azure]==3.66.0"
```

Initialiser Git + DVC:

```bash
git init
dvc init
git add .dvc .gitignore
git commit -m "Init Git and DVC"
```

Configurer le remote Azure:

windows powershell : $env:AZURE_STORAGE_CONNECTION_STRING='DefaultEndpointsProtocol=https;AccountName=xxxxx;AccountKey=xxxxx==;EndpointSuffix=core.windows.net'
linux export AZURE_STORAGE_CONNECTION_STRING='DefaultEndpointsProtocol=https;AccountName=xxxxx;AccountKey=xxxxx==;EndpointSuffix=core.windows.net'

```bash
dvc remote add -d myremote azure://dvc-storage/churn-cml-dvc
dvc remote modify myremote connection_string "${AZURE_STORAGE_CONNECTION_STRING}"
```

- ne jamais versionner les credentials.

```bash
git rm --cached .dvc/config.local
echo ".dvc/config.local" >> .gitignore
git add .gitignore
git commit -m "Ignore local DVC credentials"
```

## 4) Tracker data et models avec DVC

```bash
python script.py
dvc add data models
git add data.dvc models.dvc dvc.lock
git commit -m "Track data and models with DVC"
dvc push
```

## 5) Configurer GitHub Secrets

Dans `Settings -> Secrets and variables -> Actions`, ajouter 'new repository secret':

-* `AZURE_STORAGE_CONNECTION_STRING`
-opt: `AZURE_STORAGE_ACCOUNT`
-opt: `AZURE_STORAGE_KEY`

`GITHUB_TOKEN` est fourni automatiquement par GitHub Actions

## 6) Pipeline CI/CD

Le workflow est dans:

- `.github/workflows/ci.yml`

Il fait:
1. installation Python + dependances,
2. installation `dvc[azure]`,
3. `dvc pull`,
4. entrainement (`python script.py`),
5. `dvc add` + commit/push des fichiers `.dvc`,
6. `dvc push` des artefacts vers Azure Blob,
7. publication du rapport CML (`metrics.txt` + `conf_matrix.png`).

=> pull request : https://github.com/MakhtoutMohamed/churn-cml-dvc/pull/1

```bash
dvc status
dvc pull
dvc push
dvc doctor
```

## autre alternative est MinIO