+++
date = '2025-04-25T20:30:56+02:00'
draft = false
title = "Maîtriser GitLab CI/CD : sélection dynamique d'environnement avec les pipelines enfants"
+++
## Table des matières
- [Le problème](#le-probleme)
- [Le défi principal](#le-defi-principal)
- [Les approches infructueuses](#les-approches-infructueuses)
- [La solution avec les pipelines enfants](#la-solution-avec-les-pipelines-enfants)
- [Comment fonctionne la solution](#comment-fonctionne-la-solution)
- [Principaux avantages](#principaux-avantages)
- [Pièges et leçons apprises](#pieges-et-lecons-apprises)
- [Conclusion](#conclusion)

Dans le développement logiciel moderne, les pipelines CI/CD sont essentiels pour automatiser les processus de test et de déploiement. Cependant, lorsque votre stratégie de test implique plusieurs environnements et une planification complexe, la configuration du pipeline peut vite devenir un casse-tête. Dans cet article, je vais expliquer comment j'ai résolu un problème GitLab CI/CD particulièrement épineux, impliquant des variables spécifiques à chaque environnement et une sélection dynamique d'environnement.

## Le problème

Je devais mettre en place une stratégie de tests planifiés répondant aux exigences suivantes :

1. Les tests doivent s'exécuter selon un cycle de 2 semaines à travers différents environnements :
   - Semaine 1 : du mercredi au vendredi, exécution sur l'environnement develop
   - Semaine 2 : du lundi au mercredi, exécution sur l'environnement develop
   - Semaine 2 : jeudi, exécution sur l'environnement staging
   - Autres jours : aucun test

2. Chaque environnement (develop, staging) possédait son propre jeu d'identifiants et de valeurs de configuration, stockés sous forme de variables GitLab CI/CD avec une portée (scope) d'environnement.

3. Les tests devaient accéder à ces variables spécifiques à chaque environnement pour s'exécuter correctement.

L'implémentation initiale ressemblait à ceci :

```yaml
tests-scheduled:
  extends: .tests
  environment:
    name: placeholder
  rules:
    - if: '($CI_PIPELINE_SOURCE == "schedule" && $SCHEDULER_JOB == "testing")'
  script:
    - | #shell
      # Get current day of week (1-7, Monday is 1)
      DAY_OF_WEEK=$(date +%u)
      
      # Determine which week in the 2-week cycle
      WEEK_NUMBER=$(date +%V)
      CYCLE_WEEK=$((WEEK_NUMBER % 2))
      
      # Apply 2-week cycle logic
      if [[ $CYCLE_WEEK == 0 && $DAY_OF_WEEK -ge 3 && $DAY_OF_WEEK -le 5 ]]; then
        # Week 1: Wednesday-Friday -> develop
        export TARGET_ENV="develop"
      elif [[ $CYCLE_WEEK == 1 && $DAY_OF_WEEK == 4 ]]; then
        # Week 2: Thursday -> staging
        export TARGET_ENV="staging"
      elif [[ $CYCLE_WEEK == 1 && $DAY_OF_WEEK -ge 1 && $DAY_OF_WEEK -le 3 ]]; then
        # Week 2: Monday-Wednesday -> develop
        export TARGET_ENV="develop"
      else
        echo "Today doesn't match any scheduled testing pattern. Skipping tests."
        exit 0
      fi
      
      echo "Running tests for environment: $TARGET_ENV"
      npm run test:remote
```

## Le défi principal

En exécutant ce job, je me suis heurté à un problème fondamental : les variables spécifiques à l'environnement (comme les identifiants) n'étaient pas accessibles au job. Pourquoi ? Parce que GitLab CI/CD ne rend les variables à portée d'environnement disponibles que lorsque l'environnement du job correspond au scope de la variable.

Mon job avait `environment: name: placeholder`, mais les variables étaient rattachées aux environnements « develop » et « staging ». Même si je calculais correctement `TARGET_ENV`, GitLab n'injectait pas les bonnes variables dans le contexte d'exécution.

## Les approches infructueuses

J'ai tenté plusieurs approches qui n'ont pas fonctionné :

1. **Définition dynamique de l'environnement** : tenter de modifier l'environnement en cours de job via la variable `CI_ENVIRONMENT_NAME` ne fonctionnait pas correctement.

2. **Création d'un fichier .env** : rechercher un fichier `.env` inexistant censé contenir comme par magie nos variables d'environnement s'est soldé par un échec.

3. **Codage en dur des valeurs** : cela aurait représenté un risque de sécurité et un cauchemar de maintenance.

## La solution avec les pipelines enfants

La solution que j'ai mise en place s'appuie sur la fonctionnalité de pipeline enfant de GitLab pour créer dynamiquement un pipeline doté du bon contexte d'environnement. Voici l'approche à deux jobs que j'ai conçue :

```yaml
# First job: determines environment and creates child pipeline YAML
determine-environment:
  stage: prepare
  rules:
    - if: '($CI_PIPELINE_SOURCE == "schedule" && $SCHEDULER_JOB == "testing")'
  artifacts:
    paths:
      - child-pipeline.yml
  script:
    - | #shell
      # Get current day of week (1-7, Monday is 1)
      DAY_OF_WEEK=$(date +%u)
      
      # Determine which week in the 2-week cycle
      WEEK_NUMBER=$(date +%V)
      CYCLE_WEEK=$((WEEK_NUMBER % 2))
      
      # Apply 2-week cycle logic
      if [[ $CYCLE_WEEK == 0 && $DAY_OF_WEEK -ge 3 && $DAY_OF_WEEK -le 5 ]]; then
        # Week 1: Wednesday-Friday -> develop
        export TARGET_ENV="develop"
      elif [[ $CYCLE_WEEK == 1 && $DAY_OF_WEEK == 4 ]]; then
        # Week 2: Thursday -> staging
        export TARGET_ENV="staging"
      elif [[ $CYCLE_WEEK == 1 && $DAY_OF_WEEK -ge 1 && $DAY_OF_WEEK -le 3 ]]; then
        # Week 2: Monday-Wednesday -> develop
        export TARGET_ENV="develop"
      else
        echo "Today doesn't match any scheduled testing pattern. Skipping tests."
        # Create a placeholder file to satisfy the artifacts requirement
        touch child-pipeline.yml
        # Exit with error code to mark job as failed and stop the pipeline
        exit 1
      fi
      
      echo "Creating child pipeline for environment: $TARGET_ENV"
      
      # Create child pipeline with proper environment configuration
      cat > child-pipeline.yml << EOF
      image: node:latest
      
      stages:
        - test
        - reporting
      
      run-tests:
        stage: test
        environment:
          name: ${TARGET_ENV}
        before_script:
          - npm ci
        after_script:
          - echo "Completed tests"
        parallel: 3
        artifacts:
          when: always
          expire_in: 2 days
          paths:
            - test-results
        script:
          - echo "Running tests in ${TARGET_ENV} environment" 
          - npm run test:remote
      
      generate-report:
        stage: reporting
        before_script:
          - npm ci
        script:
          - npm run generate-report
          - npm run notify-team
        artifacts:
          when: always
          expire_in: 7 days
          paths:
            - test-report
      EOF

# Second job: triggers the child pipeline
tests-scheduled:
  stage: test
  needs:
    - determine-environment
  rules:
    - if: '($CI_PIPELINE_SOURCE == "schedule" && $SCHEDULER_JOB == "testing")'
  trigger:
    include:
      - artifact: child-pipeline.yml
        job: determine-environment
```

## Comment fonctionne la solution

1. **Séparation des jobs** : j'ai divisé la logique en deux jobs :
   - `determine-environment` : décide quel environnement tester en fonction de la date
   - `tests-scheduled` : déclenche un pipeline enfant avec la configuration issue du premier job

2. **Génération dynamique du YAML** : le premier job crée une configuration YAML complète pour le pipeline enfant, comprenant :
   - L'image Docker
   - La configuration du job
   - Le bon paramètre `environment: name: ${TARGET_ENV}`
   - Les commandes d'exécution des tests

3. **Déclenchement du pipeline enfant** : le second job récupère ce YAML généré et l'exécute en tant que pipeline enfant.

4. **Variables à portée d'environnement** : parce que le job du pipeline enfant porte le bon nom d'environnement, GitLab injecte automatiquement toutes les variables spécifiques à cet environnement.

5. **Gestion correcte des échecs** : si les tests ne doivent pas s'exécuter un jour donné, je fais explicitement échouer le premier job (via `exit 1`), ce qui empêche le pipeline enfant de démarrer.

## Principaux avantages

Cette approche offre plusieurs bénéfices significatifs :

1. **Isolation des variables d'environnement** : les variables de chaque environnement sont correctement cloisonnées et isolées.

2. **Tests conscients de la planification** : la logique du cycle de 2 semaines détermine précisément quand et où les tests doivent s'exécuter.

3. **Structure de pipeline propre** : le pipeline parent gère la logique de planification, tandis que le pipeline enfant gère l'exécution des tests.

4. **Aucun problème d'héritage de templates** : le pipeline enfant dispose d'une configuration complète et autonome, sans dépendre des templates du parent.

5. **Rapport d'échec clair** : lorsque les tests ne doivent pas s'exécuter un jour donné, le pipeline échoue de manière nette et précoce.

## Pièges et leçons apprises

Pendant l'implémentation, j'ai rencontré quelques subtilités :

1. **Conflit entre `script` et `trigger`** : GitLab n'autorise pas la présence simultanée d'un `script` et d'un `trigger` dans le même job, d'où la nécessité de deux jobs distincts.

2. **Accès aux templates depuis le pipeline enfant** : les pipelines enfants n'ont pas accès aux templates du pipeline parent (comme `.tests`), il a donc fallu y inclure toute la configuration directement.

3. **Échappement des variables** : lors de la création du YAML du pipeline enfant, les variables d'environnement doivent être correctement échappées (à l'aide de barres obliques inverses) pour éviter une expansion prématurée.

4. **Gestion des artifacts** : créez toujours le fichier d'artifact (même vide) pour éviter les erreurs « artifact not found » dans les jobs en aval.

## Conclusion

La sélection dynamique d'environnement à l'aide de pipelines enfants est une technique puissante pour les workflows CI/CD complexes. Elle permet de prendre des décisions à l'exécution sur le lieu et le moment des tests, tout en préservant une isolation correcte des environnements et un cloisonnement propre des variables.

Cette approche est particulièrement précieuse lorsque :

- Vous disposez de plusieurs environnements avec des configurations différentes
- Votre logique de planification des tests est complexe
- Vous devez garantir un accès correct aux identifiants spécifiques à chaque environnement
- Vous souhaitez éviter d'exécuter des tests inutiles

En tirant parti de la fonctionnalité de pipeline enfant de GitLab, vous pouvez créer des workflows de test véritablement dynamiques et sensibles à l'environnement, qui s'adaptent à votre planning et à vos exigences tout en maintenant une sécurité et une isolation appropriées.
