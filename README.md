# TP4 MLOps - Churn + DVC + S3 + GitHub Actions + CML

Ce projet implemente une boucle MLOps complete:
1. entrainement ML (`script.py`),
2. versioning des donnees/modeles avec DVC,
3. stockage distant sur S3,
4. CI/CD avec GitHub Actions,
5. publication automatique d'un rapport avec CML.

## 1. Prerequis

1. Un compte GitHub.
2. Un compte AWS (Free Tier suffit).
3. Python 3.11 recommande.
4. Git installe.

Dependances du projet dans `requirements.txt`:
1. `numpy>=2,<3`
2. `scikit-learn>=1.7,<1.8`
3. `imbalanced-learn>=0.14,<0.15`
4. `dvc[s3]==3.63.0`

## 2. Setup AWS (a faire une seule fois)

1. Creer un bucket S3 prive, par exemple `my-bucket-dvc-<votre-nom>` en `eu-west-3`.
2. Creer un utilisateur IAM technique, par exemple `dvc-ci-user`.
3. Attacher une policy minimale au prefixe `dvc-storage/*`.

Exemple de policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
     {
        "Sid": "AllowListBucket",
        "Effect": "Allow",
        "Action": ["s3:ListBucket"],
        "Resource": "arn:aws:s3:::my-bucket-dvc-xxx",
        "Condition": {
          "StringLike": {
             "s3:prefix": ["dvc-storage/*"]
          }
        }
     },
     {
        "Sid": "AllowBucketObjectsCRUD",
        "Effect": "Allow",
        "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
        "Resource": "arn:aws:s3:::my-bucket-dvc-xxx/dvc-storage/*"
     }
  ]
}
```

4. Recuperer `AWS_ACCESS_KEY_ID` et `AWS_SECRET_ACCESS_KEY`.

## 3. Setup local

### 3.1 Creer l'environnement Python

PowerShell:

```powershell
py -3.11 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
```

### 3.2 Initialiser Git

```powershell
git init
git branch -M main
git add .
git commit -m "Initial project setup"
```

### 3.3 Initialiser DVC

```powershell
dvc init
git add .dvc .gitignore
git commit -m "Init DVC"
```

### 3.4 Configurer le remote S3 DVC

```powershell
dvc remote add -d myremote s3://my-bucket-dvc-xxx/dvc-storage
dvc remote modify myremote region eu-west-3
git add .dvc/config
git commit -m "Configure DVC S3 remote"
```

### 3.5 Configurer les credentials AWS en local (non versionnes)

Option recommandee: variables d'environnement PowerShell.

```powershell
$env:AWS_ACCESS_KEY_ID="AKIA..."
$env:AWS_SECRET_ACCESS_KEY="..."
$env:AWS_DEFAULT_REGION="eu-west-3"
```

Ne jamais commiter les cles. `.dvc/config.local` est ignore.

### 3.6 Executer le training et versionner les artefacts

```powershell
python script.py
dvc add data models
git add data.dvc models.dvc .gitignore
git commit -m "Track data and models with DVC"
dvc push
```

## 4. Setup GitHub

### 4.1 Push du depot

```powershell
git remote add origin git@github.com:<votre-user>/churn-cml-dvc.git
git push -u origin main
```

### 4.2 Secrets GitHub Actions

Dans `Settings -> Secrets and variables -> Actions`, ajouter:
1. `AWS_ACCESS_KEY_ID`
2. `AWS_SECRET_ACCESS_KEY`
3. `AWS_DEFAULT_REGION`

`GITHUB_TOKEN` est fourni automatiquement.

## 5. Workflow CI/CD

Le workflow est dans `/.github/workflows/ci.yml`.

A chaque `push` ou `pull_request`, il fait:
1. setup Python 3.11,
2. installation des dependances,
3. `dvc pull`,
4. execution `python script.py`,
5. `dvc add data models`,
6. commit metadonnees DVC si changement,
7. `dvc push`,
8. publication CML (metrics + confusion matrix).

Note sans AWS:
1. si `dvc pull` ne recupere pas `data/dataset.csv`, la CI utilise automatiquement `seed_data/dataset.csv` pour permettre l'execution du training.
2. `dvc push` devient non bloquant tant que le remote AWS n'est pas accessible.
3. des que AWS est configure, le flux complet S3 reprend normalement.

## 6. Verification complete (checklist)

1. Local training:
    `python script.py` doit produire `metrics.txt` et `conf_matrix.png`.
2. DVC local:
    `dvc status` ne doit pas afficher d'erreur de config.
3. DVC remote:
    `dvc push` doit envoyer les objets vers `s3://.../dvc-storage`.
4. GitHub Actions:
    le job `churn-cml-dvc-ci` doit passer en vert.
5. CML:
    un commentaire automatique doit apparaitre sur le PR ou le commit.

## 7. Depannage rapide

1. `dvc: command not found`:
    reinstallez avec `pip install "dvc[s3]==3.63.0"` dans le bon environnement.
2. `No remote cache yet`:
    normal au premier run, faites un `dvc push` local initial.
3. Erreurs AWS auth:
    verifier les trois secrets GitHub et la region.
4. Erreurs permissions S3:
    verifier la policy IAM et le prefixe `dvc-storage/*`.
