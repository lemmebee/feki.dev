+++
date = '2026-07-12T10:00:00+02:00'
draft = false
title = "Tester les contrats d'une flotte de microservices sans ralentir la livraison"
+++
**tl;dr :** Les tests end-to-end sur une flotte de microservices sont lents, instables et forcent les équipes à déployer de manière synchronisée. Les tests de contrat pilotés par le consommateur remplacent le goulot d'étranglement du staging partagé par des vérifications rapides et isolées que chaque service exécute de son côté. Cet article présente une architecture construite autour d'un Pact Broker et d'une barrière `can-i-deploy`, et montre comment l'intégrer à la CI pour protéger les intégrations sans bloquer le moindre pipeline.

---
## Table des matières
- [Le problème : la confiance d'intégration ne passe pas à l'échelle](#le-problème--la-confiance-dintégration-ne-passe-pas-à-léchelle)
- [Pourquoi pas davantage de tests end-to-end](#pourquoi-pas-davantage-de-tests-end-to-end)
- [Les contrats pilotés par le consommateur en une minute](#les-contrats-pilotés-par-le-consommateur-en-une-minute)
- [L'architecture](#larchitecture)
- [Côté consommateur](#côté-consommateur)
- [Côté fournisseur](#côté-fournisseur)
- [La barrière de déploiement : can-i-deploy](#la-barrière-de-déploiement--can-i-deploy)
- [Pourquoi cela reste rapide](#pourquoi-cela-reste-rapide)
- [Versioning et environnements](#versioning-et-environnements)
- [Déploiement progressif, sans big bang](#déploiement-progressif-sans-big-bang)
- [Pièges](#pièges)
- [Conclusion](#conclusion)

## Le problème : la confiance d'intégration ne passe pas à l'échelle

Un service isolé est facile à tester. Une flotte de quarante services qui s'appellent les uns les autres via HTTP ne l'est pas. La question que chaque équipe se pose sans cesse est simple : *si je modifie cet endpoint, quel déploiement vais-je casser ?*

La réponse habituelle est un environnement de staging partagé et une suite de tests end-to-end. Cela fonctionne un temps. Puis cela s'arrête. Le staging s'écarte de la production, le déploiement cassé d'une équipe bloque les tests de toutes les autres, la suite prend quarante minutes et la moitié des échecs sont instables. La livraison ralentit jusqu'à la vitesse du test le plus lent et le plus instable de l'environnement partagé. C'est exactement la taxe que vous cherchiez à éviter en adoptant les microservices.

## Pourquoi pas davantage de tests end-to-end

Les tests end-to-end sont précieux, mais ce sont les mauvais outils pour obtenir la *confiance d'intégration au moment du changement*. Ils sont :

- **Lents.** Ils démarrent de vrais services, de vraies bases de données, de vrais sauts réseau.
- **Instables.** Chaque dépendance est une occasion d'échouer pour des raisons sans rapport avec votre changement.
- **Couplés.** Ils exigent que tous les participants soient déployés ensemble, imposant donc des releases synchronisées.
- **Tardifs.** Ils s'exécutent après le merge, dans un environnement partagé, loin du développeur qui a fait le changement.

Vous voulez tout de même conserver une fine couche de tests end-to-end pour les parcours utilisateurs critiques. Mais vous ne voulez pas qu'ils se dressent entre une modification d'une ligne et la production.

## Les contrats pilotés par le consommateur en une minute

Les tests de contrat coupent le test d'intégration en deux, de sorte que les deux services n'ont jamais besoin d'être actifs en même temps.

- Le **consommateur** (l'appelant) écrit un test décrivant précisément les requêtes qu'il émet et les réponses qu'il attend. L'exécution de ce test contre un mock local produit un **contrat** : un fichier JSON d'exemples concrets de requêtes/réponses.
- Le **fournisseur** (l'appelé) rejoue ce contrat contre son vrai code et prouve qu'il peut satisfaire chaque attente.

Aucun des deux tests n'a besoin que l'autre service tourne. Le consommateur teste contre un mock, le fournisseur teste contre un contrat enregistré. Les deux moitiés ne se rencontrent que sous la forme d'un fichier, arbitré entre elles. [Pact](https://pact.io) en est le standard de facto, et un **Pact Broker** est le service qui stocke les contrats et suit quelles versions sont compatibles.

## L'architecture

```
   ┌─────────────────┐        publishes pact         ┌──────────────────┐
   │  Consumer CI     │  ───────────────────────────> │                  │
   │  (checkout-svc)  │                                │                  │
   │                  │  <── can-i-deploy? ──────────  │   Pact Broker    │
   └─────────────────┘                                │   (PactFlow or   │
                                                       │    self-hosted)  │
   ┌─────────────────┐   webhook: verify this pact     │                  │
   │  Provider CI     │  <───────────────────────────  │   - contracts    │
   │  (payments-svc)  │                                │   - versions     │
   │                  │  ── publishes verification ──> │   - env tags     │
   └─────────────────┘                                └──────────────────┘
             │                                                  │
             │              can-i-deploy? <─────────────────────┘
             ▼
        deploy to prod
```

Trois éléments mobiles, et le Broker est la seule pièce d'infrastructure partagée :

1. **Le Broker** stocke chaque contrat, chaque résultat de vérification de fournisseur, et un tag indiquant dans quel environnement chaque version de service est déployée. C'est la source unique de vérité pour répondre à : « la version X de A est-elle compatible avec ce qui tourne en production de B ? »
2. **Les pipelines consommateurs** publient les contrats à chaque build et interrogent le Broker avec `can-i-deploy` avant de livrer.
3. **Les pipelines fournisseurs** vérifient les contrats, soit sur leurs propres commits, soit déclenchés par un **webhook** du Broker lorsqu'un nouveau contrat apparaît.

La propriété clé : il n'y a aucun environnement d'exécution partagé sur le chemin critique. Le Broker échange des fichiers et des verdicts, pas du trafic en direct.

## Côté consommateur

Le test consommateur déclare l'interaction et génère le pact. Le voici avec `pact-python` :

```python
import atexit
import requests
from pact import Consumer, Provider

pact = Consumer("checkout-svc").has_pact_with(
    Provider("payments-svc"), pact_dir="./pacts"
)
pact.start_service()
atexit.register(pact.stop_service)


def test_authorize_payment():
    expected = {"status": "authorized", "reference": "PAY-123"}

    (
        pact
        .given("a customer with a valid card")
        .upon_receiving("an authorization request")
        .with_request(
            method="POST",
            path="/v1/authorizations",
            body={"amount": 4200, "currency": "EUR"},
        )
        .will_respond_with(200, body=expected)
    )

    with pact:
        resp = requests.post(
            f"{pact.uri}/v1/authorizations",
            json={"amount": 4200, "currency": "EUR"},
        )

    assert resp.json()["status"] == "authorized"
```

Cela s'exécute en quelques millisecondes contre un mock local. Aucun `payments-svc` à l'horizon. En cas de succès, il écrit `pacts/checkout-svc-payments-svc.json`, que le pipeline publie sur le Broker, estampillé avec le SHA git du consommateur :

```bash
pact-broker publish ./pacts \
  --consumer-app-version "$CI_COMMIT_SHA" \
  --branch "$CI_COMMIT_REF_NAME" \
  --broker-base-url "$PACT_BROKER_URL" \
  --broker-token "$PACT_BROKER_TOKEN"
```

## Côté fournisseur

Le fournisseur vérifie le contrat contre sa véritable application. Le Broker lui indique quels contrats récupérer, si bien que le fournisseur n'a pas besoin de connaître ses consommateurs à l'avance :

```python
from pact import Verifier

verifier = Verifier(
    provider="payments-svc",
    provider_base_url="http://localhost:8080",
)

success, _ = verifier.verify_with_broker(
    broker_url=PACT_BROKER_URL,
    broker_token=PACT_BROKER_TOKEN,
    publish_version=CI_COMMIT_SHA,
    publish_verification_results=True,
    provider_states_setup_url="http://localhost:8080/_pact/provider_states",
)
assert success
```

Le seul vrai travail côté fournisseur, ce sont les **provider states** : un endpoint réservé aux tests qui place le service dans la précondition qu'un contrat suppose (`"a customer with a valid card"`), généralement en peuplant une base de données de test. C'est là toute la couture d'intégration, et elle vit dans le propre dépôt du fournisseur, là où est sa place.

## La barrière de déploiement : can-i-deploy

C'est ici que les tests de contrat sont rentabilisés. Avant que le moindre service ne se déploie, il pose une seule question au Broker : *compte tenu de tout ce qui tourne déjà en production, ma version peut-elle être livrée sans danger ?*

```bash
pact-broker can-i-deploy \
  --pacticipant checkout-svc \
  --version "$CI_COMMIT_SHA" \
  --to-environment production \
  --broker-base-url "$PACT_BROKER_URL" \
  --broker-token "$PACT_BROKER_TOKEN"
```

Le Broker vérifie sa matrice de contrats et de résultats de vérification, et sort avec un code non nul si cette version a le moindre contrat non satisfait vis-à-vis de ce qui est actuellement en `production`. Vert signifie que chaque consommateur avec lequel il dialogue, et chaque fournisseur qui dépend de lui, sont vérifiés compatibles. Rouge signifie que le déploiement casserait quelqu'un, et il nomme qui.

Aucun environnement n'a été démarré pour répondre à cette question. C'est une simple requête en base sur des faits enregistrés.

## Pourquoi cela reste rapide

Tout l'intérêt est que la livraison ne ralentisse pas. Elle reste rapide parce que :

- **Aucun environnement partagé sur le chemin critique.** Consommateur et fournisseur sont testés en isolation, en parallèle, dans leurs propres pipelines. Le build rouge d'une équipe ne bloque jamais les tests d'une autre.
- **Les vérifications s'exécutent là où se trouve le changement.** Un test de contrat échoue dans la MR du développeur en quelques secondes, pas dans un run end-to-end nocturne que personne ne lit.
- **La vérification côté fournisseur est découplée via des webhooks.** Lorsqu'un consommateur publie un nouveau contrat, un webhook du Broker déclenche la vérification du seul fournisseur concerné. Rien d'autre dans la flotte ne se reconstruit.
- **La barrière de déploiement est une requête, pas une exécution de tests.** `can-i-deploy` lit des résultats enregistrés. Elle ajoute des secondes, pas des minutes.
- **Seul ce qui a changé est vérifié.** Les fournisseurs vérifient les contrats qui les référencent réellement, pas la suite entière de la flotte.

Les parties lentes, instables et synchronisées sont précisément celles que cette architecture supprime.

## Versioning et environnements

Deux conventions rendent la matrice digne de confiance :

- **Versionnez chaque pacticipant par son SHA git.** Le Broker raisonne sur des versions immuables : `--consumer-app-version` et `--publish-version` doivent donc valoir `$CI_COMMIT_SHA`, jamais un tag flottant.
- **Enregistrez les environnements avec `record-deployment`.** Lorsqu'un service atteint réellement la production, informez-en le Broker :

```bash
pact-broker record-deployment \
  --pacticipant payments-svc \
  --version "$CI_COMMIT_SHA" \
  --environment production
```

Désormais, `can-i-deploy --to-environment production` compare avec la réalité, pas avec une supposition. Le Broker connaît toujours l'ensemble exact des versions en service dans chaque environnement, ce qui rend la réponse de compatibilité correcte plutôt qu'optimiste.

## Déploiement progressif, sans big bang

Vous ne convertissez pas quarante services en un sprint. Commencez là où ça fait mal :

1. **Choisissez une intégration bruyante.** La paire de services dont les échecs en staging partagé coûtent le plus cher. Un consommateur, un fournisseur.
2. **Mettez en place le Broker.** PactFlow si vous le voulez managé, ou le Pact Broker open source sur votre propre cluster. C'est un petit service stateful adossé à un stockage Postgres, qui s'intègre proprement dans un workflow Helm/ArgoCD.
3. **Rendez `can-i-deploy` consultatif d'abord.** Exécutez-le dans les pipelines mais sans bloquer dessus. Laissez les équipes le voir attraper de vraies ruptures pendant deux semaines et bâtir la confiance.
4. **Basculez-le en mode bloquant** sur cette première paire, puis étendez vers l'extérieur une intégration à la fois. Chaque nouvelle paire s'ajoute sans jamais toucher aux autres.

Parce que chaque vérification est isolée, l'adoption est incrémentale par construction. Il n'y a aucun interrupteur global à actionner pour toute la flotte.

## Pièges

- **Les contrats ne sont pas des tests de schéma.** Limitez-les aux champs et comportements que le consommateur utilise réellement. Trop spécifier transforme chaque refactoring du fournisseur en faux échec.
- **Les provider states sont du code.** Traitez l'endpoint de setup comme du code de production, pas comme un ajout de test après coup, sinon la vérification devient instable pour de mauvaises raisons.
- **Les tests de contrat ne sont pas des tests end-to-end.** Ils prouvent que deux services s'accordent sur un message. Ils ne prouvent pas qu'un flux métier fonctionne à travers cinq d'entre eux. Conservez une fine couche end-to-end pour les parcours qui comptent le plus.
- **Oublier `record-deployment`.** Si le Broker ne sait pas ce qui est réellement en production, `can-i-deploy` répond à la mauvaise question.

## Conclusion

Le mode de défaillance d'une flotte qui grandit n'est pas le manque de tests, c'est une confiance d'intégration qui n'existe qu'à l'intérieur d'un unique environnement partagé et lent. Les contrats pilotés par le consommateur déplacent cette confiance dans le pipeline propre à chaque service, et une barrière `can-i-deploy` transforme la question « est-ce que ça va casser quelqu'un ? » en une requête rapide et sans surprise plutôt qu'en un pari de quarante minutes.

L'architecture est modeste : un Broker au milieu, les contrats qui affluent des consommateurs, les vérifications qui affluent des fournisseurs, et une barrière de déploiement qui lit la matrice. Cela suffit à laisser chaque équipe livrer à son propre rythme tout en dormant sur ses deux oreilles. La qualité intégrée dès le départ, pas rajoutée juste avant la release.
