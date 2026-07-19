---
title: "À propos"
type: "page"
url: "about/"
layout: "about"
date: 2025-04-16
passions: "En dehors de la tech, je décroche en retenant ma respiration et en plongeant le plus profond possible dans la mer. Ça s'appelle l'<a href=\"https://www.aidainternational.org/Athletes/Profile-053c496c-cee5-4f1c-b5b4-96889aefa87c\">apnée</a>."
contact_intro: "Une question, une opportunité, ou juste envie d'échanger :"
email: "me@feki.dev"
products:
  - name: "issuance-gateway"
    url: "https://github.com/lemmebee/issuance-gateway"
    lang: "go"
    lang_label: "Go"
    desc: "Passerelle asynchrone d'émission de certificats en Go. Pool de workers, idempotence atomique, arrêt gracieux, livraison unique PKCS#12, injection de fautes intégrée."
  - name: "frfix"
    url: "https://github.com/lemmebee/frfix"
    lang: "python"
    lang_label: "Python"
    desc: "Démon de correction automatique du français en temps réel pour Linux/X11, conservateur par conception."
  - name: "ouioui"
    url: "https://github.com/lemmebee/ouioui"
    lang: "python"
    lang_label: "Python"
    desc: "Application full-stack d'apprentissage du français. Scraping Larousse, traduction DeepL, répétition espacée SM-2. FastAPI, HTMX, SQLite."
  - name: "lazyfy"
    url: "https://github.com/lemmebee/lazyfy"
    lang: "go"
    lang_label: "Go"
    desc: "TUI en Go créant des playlists Spotify à partir des morceaux du jour."
---

Je construis à grande échelle : les services, les pipelines, le cloud et les tests qui les protègent.

Huit ans entre le développement backend, l'infrastructure cloud et l'automatisation des tests. La plupart des ingénieurs couvrent l'un des trois. Maîtriser toute la chaîne, c'est supprimer les passages de relais qui ralentissent la livraison, et c'est ce qui m'amène sur les problèmes situés entre les équipes.

**Logiciel.** Chez Contentsquare, j'ai construit une plateforme d'analyse pour l'ingénierie : un pipeline de données agrégeant AWS, Prometheus, Jira, GitHub et TestRail, la couverture par merge livrée via une CLI CI, et plus de 30 vues de base de données derrière un tableau de bord réellement utilisé par les équipes. En dehors du travail, je livre en Go et en Python : une passerelle asynchrone d'émission de certificats avec pool de workers, idempotence atomique et injection de fautes intégrée ; un démon de correction automatique du français en temps réel pour Linux ; une application full-stack de répétition espacée sur FastAPI.

**Infrastructure.** Chez ORIS, j'ai réduit les coûts AWS de 30 à 50 % via consolidation multi-comptes, reconfiguration EKS/Karpenter, remplacement de la NAT Gateway (~90 % de trafic sortant en moins) et extinction hors heures ouvrées, et j'ai migré une flotte de microservices vers un workflow GitOps Helm/ArgoCD avec détection automatique de dérive. Chez Contentsquare, des déploiements Kubernetes basés sur Terraform pour plus de 15 microservices, avec un objectif de disponibilité de 99,95 %, absorbant des pics de trafic jusqu'à 300 %.

**Livraison.** J'ai refondu la CI/CD en templates GitLab réutilisables avec détection automatique des projets sur Node.js, Python, Java et Go, et déplacé les runners d'EC2 vers un groupe EKS spot auto-scalable pour exécuter les suites en parallèle sous charge.

**Tests.** Je viens de l'automatisation des tests et cela façonne encore ma façon de construire : des quality gates (SonarQube, Trivy) intégrés au pipeline plutôt que rajoutés après coup, des tests de contrat qui permettent à une flotte de microservices de se déployer indépendamment, des suites bout-en-bout parallélisées pour réduire le temps d'exécution d'environ 60 %.

L'essentiel de ce que j'apprends en chemin finit dans les articles de ce site.
