+++
date = '2025-04-03T22:30:56+02:00'
draft = false
title = 'De rigide à robuste : refonte de nos templates CI pour un workflow moderne'
+++
**tl;dr :** Nos anciens templates CI/CD étaient répétitifs, rigides et difficiles à maintenir. Nous les avons reconstruits selon une approche modulaire et agnostique au langage, avec des composants réutilisables, une logique conditionnelle intelligente et un outillage automatisé (comme le versioning et le security scanning). Résultat : des pipelines plus rapides, plus cohérents, plus sûrs et plus faciles à maintenir, adaptables aux besoins variés des projets.

---
## Table des matières
- [Le défi : quand les templates deviennent des obstacles](#le-defi--quand-les-templates-deviennent-des-obstacles)
- [La transformation : introduction d'une architecture modulaire et agnostique au langage](#la-transformation--introduction-dune-architecture-modulaire-et-agnostique-au-langage)
 - [1. Conception modulaire avec `include`](#1-conception-modulaire-avec-include)
 - [2. Approche agnostique au langage et détection intelligente](#2-approche-agnostique-au-langage-et-detection-intelligente)
 - [3. Étapes de pipeline unifiées](#3-etapes-de-pipeline-unifiees)
 - [4. Capacités CI/CD avancées](#4-capacites-cicd-avancees)
- [Les bénéfices : un avenir plus efficace et plus sûr](#les-benefices--un-avenir-plus-efficace-et-plus-sur)
- [Conclusion](#conclusion)

Les pipelines d'intégration continue et de déploiement continu (CI/CD) sont la colonne vertébrale du développement logiciel moderne. Pourtant, les templates qui gouvernent ces pipelines deviennent souvent une source de frustration : rigides, répétitifs, difficiles à maintenir et incapables de suivre le rythme d'une diversité de stacks techniques. Nous avons rencontré exactement ces problèmes avec notre dépôt interne de templates CI. Voici comment nous avons transformé notre architecture CI, passant d'un casse-tête de maintenance à un atout flexible, modulaire et puissant.

## Le défi : quand les templates deviennent des obstacles

Notre configuration précédente de templates CI souffrait de plusieurs maux de croissance :

* **Duplication de code :** une logique similaire, comme la mise en place d'environnements pour un langage donné ou les étapes de déploiement, était copiée à travers de nombreux templates.
* **Flexibilité limitée :** personnaliser les pipelines impliquait souvent de dupliquer de larges portions de code de template juste pour apporter de petites modifications.
* **Modularité insuffisante :** un changement, par exemple sur la configuration d'un outil de scan, obligeait à traquer et mettre à jour plusieurs fichiers.
* **Hypothèses codées en dur :** les templates faisaient souvent des suppositions rigides sur la structure du projet ou le langage utilisé, ce qui les rendait difficiles à adapter. Exemple : un job de build pouvait ne fonctionner que si `requirements.txt` existait, échouant pour les projets Node.js.

Ces problèmes ralentissaient les cycles de développement et augmentaient le risque d'incohérences. Il était clair qu'une refonte majeure s'imposait.

## La transformation : introduction d'une architecture modulaire et agnostique au langage

Nous avons entrepris un effort de refactoring conséquent, guidé par des principes de modularité, de flexibilité et de maintenabilité. Voici les principales améliorations techniques :

### 1. Conception modulaire avec `include`

Nous avons largement exploité le mot-clé `include:` de GitLab CI. Au lieu d'un unique `.gitlab-ci.yml` monolithique, les projets incluent désormais les parties de template pertinentes. Nous avons créé un répertoire `common/` pour la logique partagée (par exemple les étapes de security scanning) et un répertoire `languages/` pour les configurations spécifiques (Node.js, Python).

*Exemple de structure :*

```yaml
# .gitlab-ci.yml in a user's project
include:
  - project: 'your-group/your-templates-repo'
    ref: main # Or a specific version tag
    file:
      - '/common/security.yml'
      - '/languages/python.yml' # Include if it's a Python project
      - '/pipeline-template/lambda.yml' # If deploying a Lambda

variables:
  PYTHON_APP_DIR: "src" # Tell the template where the code lives
```

### 2. Approche agnostique au langage et détection intelligente

La nouvelle architecture détecte intelligemment le langage et la structure du projet, souvent à l'aide de variables CI/CD prédéfinies (comme `PYTHON_APP_DIR` ou `NODE_APP_DIR`). Une logique conditionnelle (`rules:`) au sein des templates garantit que les jobs ne s'exécutent que lorsque c'est nécessaire.

*Exemple de règle conditionnelle (conceptuel) :*

```yaml
# From languages/python.yml
.python_default_rules: &python_default_rules
  rules:
    - if: '$PYTHON_APP_DIR && $PYTHON_APP_DIR != "."' # Run if Python dir is set
      exists:
        - "$PYTHON_APP_DIR/pyproject.toml" # And a key file exists
      when: always
    - when: never # Don't run otherwise

python-lint:
  <<: *python_default_rules # Apply the rule
  script:
    - echo "Running Python linting in $PYTHON_APP_DIR..."
    # - python lint commands...
```

### 3. Étapes de pipeline unifiées

Des étapes communes comme `install-dependencies` peuvent désormais avoir des implémentations spécifiques au langage, déclenchées de manière conditionnelle, qui alimentent les étapes partagées suivantes.

*Exemple d'étape unifiée (conceptuel) :*

```yaml
# From common/pipeline.yml
install-dependencies:
  stage: install
  script:
    - echo "All dependencies installed successfully"
  needs: # Optional dependencies based on detected language
    - job: python-install-dependencies # Runs if python rules match
      optional: true
    - job: node-install-dependencies # Runs if node rules match
      optional: true
  rules:
    # Rule to run if *any* language was detected
    - if: '$NODE_APP_DIR || $PYTHON_APP_DIR'
      when: always
```

### 4. Capacités CI/CD avancées

* **Versioning sémantique automatisé :** nous avons mis en place un système basé sur des conventions de messages de commit (par exemple en préfixant les commits par `#major`, `#minor`, `#patch`). L'automatisation CI détecte ces préfixes lors du merge vers la branche principale, calcule la prochaine version sémantique, crée un tag et génère une entrée de changelog.
* **Security scanning renforcé :** intégration de Trivy pour un scan complet des vulnérabilités des images de conteneurs et du système de fichiers. Les résultats peuvent être poussés automatiquement vers AWS Security Hub pour une visibilité et un suivi centralisés.
* **Support CDK multi-langage :** les templates de déploiement gèrent désormais mieux les projets AWS CDK écrits dans différents langages (comme TypeScript, Python), en exécutant les commandes `cdk` appropriées selon les fichiers de projet détectés (`package.json`, `requirements.txt`, etc.).
* **Caching amélioré :** mise en place de stratégies de caching plus intelligentes, en particulier pour les étapes de scan Kubernetes, afin d'accélérer l'exécution des pipelines.

## Les bénéfices : un avenir plus efficace et plus sûr

Cette architecture repensée a apporté des avantages significatifs :

* **Mises à jour simplifiées :** modifier une tâche commune (comme la mise à jour d'un scanner) ne nécessite de changer qu'un seul endroit central.
* **Développement plus rapide :** les développeurs configurent leurs pipelines plus vite en incluant les blocs pertinents et en définissant quelques variables.
* **Cohérence accrue :** la réduction de la duplication rend les builds et les déploiements plus fiables et plus prévisibles.
* **Sécurité renforcée :** un scan automatisé et intégré améliore la posture de sécurité globale.
* **Flexibilité accrue :** on intègre facilement de nouveaux langages ou on s'adapte à différentes structures de projet sans réécrire massivement les templates.

## Conclusion

Refactorer ses templates CI/CD est un investissement qui en vaut la peine. En adoptant la modularité (`include:`), la logique conditionnelle (`rules:`), l'automatisation (versioning, scanning) et une conception agnostique au langage, nous avons transformé nos templates internes d'un goulot d'étranglement en un puissant accélérateur. Si votre configuration CI/CD ressemble davantage à une collection de scripts qu'à un système cohérent, envisagez d'adopter ces principes pour un avenir plus maintenable, plus flexible et plus efficace.
