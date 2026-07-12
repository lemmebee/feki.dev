+++
date = '2025-08-28T21:14:07+02:00'
draft = false
title = 'Migration de PostgreSQL RDS depuis plusieurs comptes AWS vers une instance unique'
+++
# Table des matières
- [Le scénario](#le-scenario)
- [Notre architecture](#notre-architecture)
- [Prérequis](#prerequis)
- [La stratégie de migration : pourquoi `pg_dump` et `pg_restore` ?](#la-strategie-de-migration-pourquoi-pg_dump-et-pg_restore)
- [Guide de migration étape par étape](#guide-de-migration-etape-par-etape)
  - [Phase 1 : préparation du compte cible (Non-Prod)](#phase-1-preparation-du-compte-cible-non-prod)
  - [Phase 2 : préparation des comptes sources (A et B)](#phase-2-preparation-des-comptes-sources-a-et-b)
  - [Phase 3 : exécution du script de migration](#phase-3-execution-du-script-de-migration)
  - [Phase 4 : validation post-migration](#phase-4-validation-post-migration)
- [Considérations finales](#considerations-finales)

À mesure que les organisations grandissent, il est fréquent que les environnements de développement et de test se retrouvent cloisonnés dans des comptes AWS distincts. Si cette approche est excellente pour l'isolation de la sécurité et de la facturation, elle peut entraîner une charge de gestion et des coûts accrus. Une stratégie courante pour rationaliser les opérations consiste à consolider ces bases de données non productives au sein d'une seule instance RDS partagée, hébergée dans un compte central « Non-Prod ».

Dans ce guide, nous allons détailler une méthode robuste, pilotée par script, pour migrer plusieurs instances PostgreSQL RDS provenant de différents comptes AWS sources vers une unique instance RDS cible. Nous utiliserons un hôte de migration EC2 dédié ainsi que les outils PostgreSQL standard (`pg_dump` et `pg_restore`) afin de garantir un processus fiable et reproductible.

### Le scénario

Imaginez que vous disposez de deux comptes AWS sources et d'un compte cible :
*   **Compte source A** : contient une instance RDS avec une base de données pour une application (`app_db_a`).
*   **Compte source B** : contient une instance RDS avec une base de données pour une autre application ou un autre environnement (`app_db_b`).
*   **Compte cible Non-Prod** : là où nous voulons héberger une seule instance RDS plus grande contenant à la fois `app_db_a` et `app_db_b` en tant que bases de données logiques distinctes.

### Notre architecture

Nous réaliserons la migration depuis une instance EC2 centrale située dans le compte cible. Cette instance assumera des rôles IAM dans les comptes sources afin d'obtenir les accès nécessaires pour récupérer les sauvegardes des bases de données.



## Prérequis

Avant de commencer, assurez-vous d'avoir les éléments suivants en place :

1.  **Permissions IAM** : vous avez besoin d'un accès administrateur dans les trois comptes AWS pour gérer les rôles IAM, les policies, EC2, RDS et les paramètres VPC.
2.  **Réseau** :
    *   **VPC Peering** : vous devez établir et accepter une connexion de peering VPC entre le VPC cible et chaque VPC source.
    *   **Tables de routage** : mettez à jour les tables de routage de tous les VPC afin d'acheminer correctement le trafic à travers les connexions de peering.
    *   **Security groups** : ce point est critique.
        *   **Security groups des RDS sources** : ils doivent autoriser le trafic entrant sur le port PostgreSQL `5432` depuis l'**adresse IP privée** de l'hôte de migration EC2.
        *   **Security group du RDS cible** : il doit autoriser le trafic entrant sur le port `5432` depuis l'hôte de migration EC2.
3.  **Gestion des identifiants** : vous disposez des identifiants master pour toutes les instances RDS. **Il est recommandé, en matière de sécurité, de stocker ces identifiants dans AWS Secrets Manager** dans leurs comptes respectifs, et ce guide partira du principe que vous le faites.
4.  **Outils** : une instance EC2 sous Linux avec l'**AWS CLI** et les **outils client PostgreSQL** installés.

## La stratégie de migration : pourquoi `pg_dump` et `pg_restore` ?

Pour cette tâche, nous optons pour une méthode de sauvegarde et de restauration logique.

*   **`pg_dump`** : crée une sauvegarde cohérente et portable d'une seule base de données, au format texte ou binaire.
*   **`pg_restore`** : restaure une base de données à partir d'une archive créée par `pg_dump`.

Cette approche est idéale car elle est :
*   **Flexible** : elle gère facilement les migrations entre différentes versions de PostgreSQL.
*   **Granulaire** : elle nous permet de faire un dump d'une base depuis un serveur et de la restaurer en tant que base logique différente sur un autre, ce qui correspond exactement à notre besoin.
*   **Fiable** : c'est la méthode standard et éprouvée pour les migrations PostgreSQL.

**Remarque sur l'indisponibilité** : cette méthode nécessite une brève interruption de service des applications sources afin de garantir que le dump de la base constitue un instantané cohérent. Planifiez votre fenêtre de migration en conséquence.

---

## Guide de migration étape par étape

### Phase 1 : préparation du compte cible (Non-Prod)

1.  **Lancer l'hôte de migration EC2**
    *   Lancez une instance EC2 Amazon Linux 2 (par exemple `t3.medium`) dans le même VPC que votre instance RDS cible.
    *   Attachez-lui un rôle IAM, que nous créerons à l'étape suivante.

2.  **Créer un rôle IAM pour l'hôte EC2 (`MigrationHostRole`)**
    Ce rôle permettra à notre instance EC2 d'assumer des rôles dans les comptes sources. Créez une policy nommée `AssumeCrossAccountRolePolicy` et attachez-la au rôle.

    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": "sts:AssumeRole",
                "Resource": [
                    "arn:aws:iam::SOURCE_A_ACCOUNT_ID:role/RdsCrossAccountAccessRole",
                    "arn:aws:iam::SOURCE_B_ACCOUNT_ID:role/RdsCrossAccountAccessRole"
                ]
            }
        ]
    }
    ```

3.  **Préparer l'instance RDS cible**
    *   Provisionnez votre instance PostgreSQL RDS consolidée si elle n'existe pas déjà.
    *   Connectez-vous à l'instance avec un client comme `psql` et créez les bases de données vides qui recevront les données migrées.
    ```sql
    -- Connect as the master user
    psql -h <your-target-rds-endpoint> -U <master_user> -d postgres

    -- Run these commands
    CREATE DATABASE app_db_a;
    CREATE DATABASE app_db_b;
    ```

### Phase 2 : préparation des comptes sources (A et B)

***Effectuez ces étapes dans chaque compte source.***

1.  **Créer le rôle IAM cross-account (`RdsCrossAccountAccessRole`)**
    C'est le rôle que notre hôte EC2 va assumer.
    *   **Nom du rôle** : utilisez un nom cohérent, par exemple `RdsCrossAccountAccessRole`.
    *   **Relation de confiance** : configurez le rôle pour qu'il fasse confiance au `MigrationHostRole` de votre compte cible.

    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {
                    "AWS": "arn:aws:iam::TARGET_ACCOUNT_ID:role/MigrationHostRole"
                },
                "Action": "sts:AssumeRole"
            }
        ]
    }
    ```

2.  **Attacher une policy de permissions**
    Créez et attachez à ce rôle une policy qui accorde les permissions nécessaires pour lire les détails RDS et récupérer le mot de passe de la base depuis Secrets Manager.

    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": "rds:DescribeDBInstances",
                "Resource": "*"
            },
            {
                "Effect": "Allow",
                "Action": "secretsmanager:GetSecretValue",
                "Resource": "arn:aws:secretsmanager:REGION:ACCOUNT_ID:secret:YOUR_RDS_SECRET_NAME-*"
            }
        ]
    }
    ```

### Phase 3 : exécution du script de migration

Place au moment fort ! Connectez-vous à votre hôte de migration EC2 et suivez ces étapes.

1.  **Installer les outils**
    ```bash
    sudo yum update -y
    sudo yum install -y postgresql15 jq  # Use the client version matching your DB
    ```

2.  **Le script d'automatisation**
    Créez un fichier nommé `migrate_dbs.sh` et collez-y le script ci-dessous. Ce script automatise l'assomption des rôles, la récupération des identifiants, le dump de la base source et sa restauration vers la cible.

    **Avant de l'exécuter, mettez soigneusement à jour toutes les variables de substitution dans la section de configuration.**

    ```bash
    #!/bin/bash

    # Exit immediately if a command exits with a non-zero status.
    set -e

    # --- Configuration: UPDATE THESE VARIABLES ---
    # Target Account Details
    TARGET_RDS_IDENTIFIER="your-target-rds-instance-id"
    TARGET_SECRET_ARN="arn:aws:secretsmanager:us-east-1:TARGET_ACCOUNT_ID:secret/your-target-secret"
    TARGET_REGION="us-east-1"

    # Source A Details
    SOURCE_A_ACCOUNT_ID="111111111111"
    SOURCE_A_ROLE_ARN="arn:aws:iam::${SOURCE_A_ACCOUNT_ID}:role/RdsCrossAccountAccessRole"
    SOURCE_A_RDS_IDENTIFIER="source-a-rds-instance-id"
    SOURCE_A_SECRET_ARN="arn:aws:secretsmanager:us-east-1:${SOURCE_A_ACCOUNT_ID}:secret/your-source-a-secret"
    SOURCE_A_DB_NAME="source_database_a" # The actual DB name in the source RDS
    TARGET_A_DB_NAME="app_db_a"          # The new DB name in the target RDS
    SOURCE_A_REGION="us-east-1"

    # Source B Details
    SOURCE_B_ACCOUNT_ID="222222222222"
    SOURCE_B_ROLE_ARN="arn:aws:iam::${SOURCE_B_ACCOUNT_ID}:role/RdsCrossAccountAccessRole"
    SOURCE_B_RDS_IDENTIFIER="source-b-rds-instance-id"
    SOURCE_B_SECRET_ARN="arn:aws:secretsmanager:us-east-1:${SOURCE_B_ACCOUNT_ID}:secret/your-source-b-secret"
    SOURCE_B_DB_NAME="source_database_b" # The actual DB name in the source RDS
    TARGET_B_DB_NAME="app_db_b"          # The new DB name in the target RDS
    SOURCE_B_REGION="us-east-1"
    # --- End Configuration ---

    # --- Reusable Migration Function ---
    migrate_source() {
        local NICKNAME=$1; local ROLE_ARN=$2; local RDS_IDENTIFIER=$3;
        local SECRET_ARN=$4; local SOURCE_DB=$5; local TARGET_DB=$6;
        local REGION=$7; local DUMP_FILE="/tmp/${TARGET_DB}.dump"

        echo "--- Starting Migration for: ${NICKNAME} ---"

        # 1. Assume Role in the source account
        echo "--> Assuming role in source account..."
        CREDS=$(aws sts assume-role --role-arn "${ROLE_ARN}" --role-session-name "RDSCrossAccountMigration" --output json)
        export AWS_ACCESS_KEY_ID=$(echo "${CREDS}" | jq -r .Credentials.AccessKeyId)
        export AWS_SECRET_ACCESS_KEY=$(echo "${CREDS}" | jq -r .Credentials.SecretAccessKey)
        export AWS_SESSION_TOKEN=$(echo "${CREDS}" | jq -r .Credentials.SessionToken)

        # 2. Get Source RDS Connection Details
        echo "--> Fetching source RDS details for ${RDS_IDENTIFIER}..."
        SOURCE_ENDPOINT=$(aws rds describe-db-instances --db-instance-identifier "${RDS_IDENTIFIER}" --region "${REGION}" | jq -r .DBInstances[0].Endpoint.Address)
        SOURCE_SECRET=$(aws secretsmanager get-secret-value --secret-id "${SECRET_ARN}" --region "${REGION}" | jq -r .SecretString)
        SOURCE_USER=$(echo "${SOURCE_SECRET}" | jq -r .username)
        export PGPASSWORD=$(echo "${SOURCE_SECRET}" | jq -r .password)
        if [ -z "$SOURCE_ENDPOINT" ]; then echo "Error: Failed to get source endpoint."; exit 1; fi

        # 3. Dump the source database
        echo "--> Dumping database '${SOURCE_DB}'..."
        pg_dump -Fc -v -h "${SOURCE_ENDPOINT}" -U "${SOURCE_USER}" -d "${SOURCE_DB}" -f "${DUMP_FILE}"
        echo "--> Dump complete: ${DUMP_FILE}"

        # Unset temporary AWS credentials
        unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN

        # 4. Get Target RDS Connection Details
        echo "--> Fetching target RDS details..."
        TARGET_ENDPOINT=$(aws rds describe-db-instances --db-instance-identifier "${TARGET_RDS_IDENTIFIER}" --region "${TARGET_REGION}" | jq -r .DBInstances[0].Endpoint.Address)
        TARGET_SECRET=$(aws secretsmanager get-secret-value --secret-id "${TARGET_SECRET_ARN}" --region "${TARGET_REGION}" | jq -r .SecretString)
        TARGET_USER=$(echo "${TARGET_SECRET}" | jq -r .username)
        export PGPASSWORD=$(echo "${TARGET_SECRET}" | jq -r .password)
        if [ -z "$TARGET_ENDPOINT" ]; then echo "Error: Failed to get target endpoint."; exit 1; fi

        # 5. Restore to the target database
        echo "--> Restoring dump to database '${TARGET_DB}'..."
        pg_restore -v -h "${TARGET_ENDPOINT}" -U "${TARGET_USER}" -d "${TARGET_DB}" "${DUMP_FILE}"
        echo "--> Restore complete."

        # 6. Cleanup
        rm "${DUMP_FILE}"
        unset PGPASSWORD
        echo "--- Migration for ${NICKNAME} finished successfully! ---"
    }

    # --- Main Execution ---
    migrate_source "Source A" "$SOURCE_A_ROLE_ARN" "$SOURCE_A_RDS_IDENTIFIER" "$SOURCE_A_SECRET_ARN" "$SOURCE_A_DB_NAME" "$TARGET_A_DB_NAME" "$SOURCE_A_REGION"
    migrate_source "Source B" "$SOURCE_B_ROLE_ARN" "$SOURCE_B_RDS_IDENTIFIER" "$SOURCE_B_SECRET_ARN" "$SOURCE_B_DB_NAME" "$TARGET_B_DB_NAME" "$SOURCE_B_REGION"

    echo "ALL MIGRATIONS COMPLETED."
    ```
3.  **Exécuter le script**
    ```bash
    chmod +x migrate_dbs.sh
    ./migrate_dbs.sh
    ```

### Phase 4 : validation post-migration

Vérifiez toujours votre travail.
1.  **Se connecter au RDS cible** : utilisez `psql` pour vous connecter à votre nouvelle instance consolidée.
2.  **Vérifier les bases de données** : listez toutes les bases pour vous assurer que `app_db_a` et `app_db_b` sont présentes (`\l`).
3.  **Inspecter les données** : connectez-vous à chaque nouvelle base (`\c app_db_a`) et inspectez les tables (`\dt`). Exécutez quelques requêtes `SELECT count(*)` sur les tables importantes pour vérifier que les données ont bien été transférées.
4.  **Vérifier la propriété et les permissions** : `pg_dump` préserve les informations de propriété. Si les rôles utilisateurs d'origine n'existent pas sur le RDS cible, les objets appartiendront à l'utilisateur master. Vous devrez peut-être créer les rôles nécessaires et accorder les permissions (`GRANT`) manuellement après la migration.

## Considérations finales

*   **Plan de rollback** : votre chemin de rollback le plus sûr reste les instances RDS sources d'origine. Ne les décommissionnez pas tant que la nouvelle instance consolidée n'a pas été entièrement testée et validée par vos équipes applicatives.
*   **Bases de données volumineuses** : pour des bases extrêmement volumineuses (plusieurs centaines de gigaoctets ou plus), cette méthode `pg_dump` peut s'avérer lente et exiger un espace disque conséquent sur l'hôte de migration. Dans ces cas, envisagez d'utiliser AWS Database Migration Service (DMS).
*   **Nettoyage** : une fois la migration terminée et vérifiée, pensez à arrêter l'hôte de migration EC2 pour réduire les coûts, et à supprimer les rôles et policies IAM temporaires s'ils ne sont plus nécessaires.

Félicitations ! Vous avez consolidé vos instances RDS avec succès, simplifiant ainsi votre environnement non productif et réduisant potentiellement vos coûts opérationnels.
