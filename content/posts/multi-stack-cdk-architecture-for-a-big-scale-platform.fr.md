+++
date = '2026-07-14T10:00:00+02:00'
draft = false
title = "Architecture CDK multi-stacks pour une plateforme à grande échelle sur dev, stage et prod"
+++
**tl;dr :** Une seule pile CloudFormation géante ne passe pas à l'échelle d'une vraie plateforme. Elle atteint les limites de ressources, couple des équipes qui n'ont rien à voir et transforme chaque déploiement en pari sur le rayon d'impact. Cet article explique comment structurer une application AWS CDK en un ensemble de stacks dédiées qui communiquent via des références typées et des paramètres SSM, comment faire servir dev, stage et prod par une seule base de code sans copier-coller, et comment relier le tout à un pipeline auto-mutant qui promeut les changements environnement par environnement.

---
## Table des matières
- [Pourquoi pas une seule grosse pile](#pourquoi-pas-une-seule-grosse-pile)
- [Concevoir l'architecture d'abord](#concevoir-larchitecture-dabord)
- [Découper la plateforme en stacks](#découper-la-plateforme-en-stacks)
- [Une base de code, trois environnements](#une-base-de-code-trois-environnements)
- [Comment les stacks se parlent](#comment-les-stacks-se-parlent)
- [Cross-account et cross-region](#cross-account-et-cross-region)
- [Le pipeline de déploiement](#le-pipeline-de-déploiement)
- [Pièges](#pièges)
- [Conclusion](#conclusion)

## Pourquoi pas une seule grosse pile

Une application CDK démarre petite. Une stack, un VPC, une base de données, deux services. Ça marche, donc ça grossit. Six mois plus tard, cette unique stack compte deux cents ressources, trois équipes qui y committent et un template CloudFormation qui flirte avec la limite dure de 500 ressources. Chacun de ces problèmes est structurel, pas accidentel :

- **Les limites de ressources sont réelles.** CloudFormation plafonne une stack à 500 ressources. Une plateforme active dépasse ce seuil rien qu'avec les load balancers, les security groups, les task definitions et les log groups.
- **Le rayon d'impact, c'est toute la stack.** Un mauvais changement sur une Lambda annule la mise à jour du VPC posée juste à côté. Une ressource bloquée fige tous les déploiements sans rapport qui attendent derrière.
- **Les déploiements se sérialisent.** Une seule stack, c'est un seul `cdk deploy` à la fois. Les équipes font la queue au lieu de livrer en parallèle.
- **La propriété devient floue.** Quand réseau, données et code applicatif vivent dans un même fichier, personne ne possède rien, et chaque revue implique tout le monde.

La solution n'est pas moins de ressources. C'est de tracer des frontières pour que ce qui change ensemble vive ensemble, et que ce qui change à des rythmes différents vive à part.

## Concevoir l'architecture d'abord

Avant la moindre ligne de CDK, décidez des coutures. L'axe utile est le *rythme de changement* combiné à la *propriété*. L'infrastructure fondamentale (VPC, sous-réseaux, transit gateway) change rarement et appartient à une équipe plateforme. Les ressources avec état (bases de données, buckets, files) changent occasionnellement et portent des données que vous ne devez jamais détruire par accident. Les services applicatifs sans état changent plusieurs fois par jour et appartiennent aux équipes produit.

Ces trois rythmes se projettent proprement sur trois couches :

```
  ┌──────────────────────────────────────────────────────────┐
  │  Couche 3 : Application  (change à l'heure, équipes produit)│
  │  services, workers, APIs, distributions front-end          │
  └───────────────┬──────────────────────────────────────────┘
                  │ consomme : endpoints db, ARNs de files, VPC
  ┌───────────────┴──────────────────────────────────────────┐
  │  Couche 2 : Données  (change chaque semaine, protégée)     │
  │  RDS/Aurora, DynamoDB, S3, ElastiCache, SQS/SNS            │
  └───────────────┬──────────────────────────────────────────┘
                  │ consomme : VPC, sous-réseaux, security groups
  ┌───────────────┴──────────────────────────────────────────┐
  │  Couche 1 : Fondation  (change rarement, équipe plateforme)│
  │  VPC, sous-réseaux, NAT, routage, IAM partagé, KMS, DNS    │
  └──────────────────────────────────────────────────────────┘
```

Les dépendances ne pointent jamais que vers le bas. La couche 3 dépend de la couche 2 et de la couche 1. La couche 1 ne dépend de rien. Si vous vous surprenez à vouloir qu'une stack de fondation lise quelque chose dans une stack applicative, la frontière est mauvaise.

## Découper la plateforme en stacks

En CDK, une **stack** est une unité de déploiement et un **construct** est une unité de composition. L'erreur est de les confondre. On compose librement des constructs réutilisables, puis on les place dans un ensemble délibéré et restreint de stacks.

Un découpage concret pour une grosse plateforme :

```
platform-app (cdk.App)
├── NetworkStack        VPC, sous-réseaux, NAT, security groups, zones Route53
├── SecurityStack       clés KMS, rôles IAM partagés, secrets
├── DataStack           cluster Aurora, tables DynamoDB, buckets S3
├── MessagingStack      files SQS, topics SNS, bus EventBridge
├── ServiceStack (×N)   une par bounded context : ECS/Lambda + règles ALB
└── EdgeStack           CloudFront, WAF, certs ACM (us-east-1)
```

Chaque `ServiceStack` est instanciée une fois par bounded context. `checkout`, `payments` et `catalog` ont chacun leur stack, leur déploiement, leur propriétaire et leur rayon d'impact. Ajouter un service ajoute une stack, pas deux cents lignes à une stack existante.

Voici la forme d'une stack qui prend ses dépendances comme props de constructeur typées plutôt que d'aller les chercher globalement :

```typescript
interface ServiceStackProps extends cdk.StackProps {
  vpc: ec2.IVpc;
  cluster: ecs.ICluster;
  database: rds.IDatabaseCluster;
  eventBus: events.IEventBus;
  config: EnvConfig;
}

export class ServiceStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props: ServiceStackProps) {
    super(scope, id, props);

    const service = new ApplicationLoadBalancedFargateService(this, 'Svc', {
      cluster: props.cluster,
      desiredCount: props.config.service.desiredCount,
      taskImageOptions: {
        image: ecs.ContainerImage.fromRegistry(props.config.service.image),
        environment: {
          DB_ENDPOINT: props.database.clusterEndpoint.hostname,
          EVENT_BUS: props.eventBus.eventBusName,
        },
      },
    });

    // grant plutôt que d'écrire l'IAM à la main : le construct connaît le moindre privilège.
    props.database.connections.allowDefaultPortFrom(service.service);
    props.eventBus.grantPutEventsTo(service.taskDefinition.taskRole);
  }
}
```

Passer `vpc`, `cluster` et `database` en props est le choix le plus important de toute la conception. Il rend la dépendance explicite, vérifiée au type à la synthèse, et laisse CDK câbler la plomberie inter-stacks à votre place.

## Une base de code, trois environnements

Dev, stage et prod ne sont pas trois bases de code. C'est une seule base de code évaluée avec trois configs. L'anti-pattern, c'est des `if (env === 'prod')` éparpillés dans le code des constructs. Le pattern, c'est un unique objet de config typé, résolu une seule fois en haut de l'application et propagé vers le bas en props.

```typescript
export interface EnvConfig {
  readonly name: 'dev' | 'stage' | 'prod';
  readonly account: string;
  readonly region: string;
  readonly vpcMaxAzs: number;
  readonly database: { instances: number; instanceClass: string; deletionProtection: boolean };
  readonly service: { desiredCount: number; image: string };
}

const environments: Record<string, EnvConfig> = {
  dev: {
    name: 'dev', account: '111111111111', region: 'eu-west-1',
    vpcMaxAzs: 2,
    database: { instances: 1, instanceClass: 'serverless', deletionProtection: false },
    service: { desiredCount: 1, image: 'checkout:dev' },
  },
  stage: {
    name: 'stage', account: '222222222222', region: 'eu-west-1',
    vpcMaxAzs: 2,
    database: { instances: 1, instanceClass: 't4g.medium', deletionProtection: true },
    service: { desiredCount: 2, image: 'checkout:stage' },
  },
  prod: {
    name: 'prod', account: '333333333333', region: 'eu-west-1',
    vpcMaxAzs: 3,
    database: { instances: 2, instanceClass: 'r6g.xlarge', deletionProtection: true },
    service: { desiredCount: 6, image: 'checkout:1.4.2' },
  },
};
```

**Utilisez des comptes AWS séparés par environnement.** C'est la seule frontière qu'AWS applique pour vous. Une frontière de compte rend impossible qu'une expérimentation dev touche des données de prod, donne une facturation propre par environnement, et laisse les politiques IAM simples parce qu'elles n'ont jamais à distinguer les environnements au sein d'un même compte.

Le point d'entrée de l'application boucle sur la config et estampe un jeu complet de stacks par environnement. Une stage enveloppante regroupe les stacks de chaque environnement :

```typescript
const app = new cdk.App();

class PlatformStage extends cdk.Stage {
  constructor(scope: Construct, id: string, config: EnvConfig) {
    super(scope, id, { env: { account: config.account, region: config.region } });

    const network = new NetworkStack(this, 'Network', { config });
    const data = new DataStack(this, 'Data', { config, vpc: network.vpc });
    const messaging = new MessagingStack(this, 'Messaging', { config });

    new ServiceStack(this, 'Checkout', {
      config, vpc: network.vpc, cluster: network.cluster,
      database: data.cluster, eventBus: messaging.bus,
    });
  }
}

for (const config of Object.values(environments)) {
  new PlatformStage(app, config.name, config);
}
```

Une boucle, trois environnements, zéro code d'infrastructure dupliqué. La prod ne diffère du dev que par les nombres dans la table de config, ce qui est exactement là où la différence doit vivre et le seul endroit qu'un relecteur doit regarder.

## Comment les stacks se parlent

C'est le point clé. Au sein d'une même application CDK et d'un même compte, vous avez deux mécanismes, et choisir entre eux est tout l'enjeu.

**1. Références directes de constructs (à préférer).** Quand vous passez `network.vpc` dans `DataStack`, et que les deux stacks sont dans la même application, CDK repère la référence inter-stacks et génère automatiquement un `Export`/`Fn::ImportValue` CloudFormation. Vous écrivez du TypeScript ordinaire ; CDK écrit la plomberie.

```typescript
const network = new NetworkStack(this, 'Network', { config });
// network.vpc est un ec2.IVpc exposé comme champ public readonly
const data = new DataStack(this, 'Data', { config, vpc: network.vpc });
```

```typescript
export class NetworkStack extends cdk.Stack {
  public readonly vpc: ec2.Vpc;
  public readonly cluster: ecs.Cluster;

  constructor(scope: Construct, id: string, props: NetworkStackProps) {
    super(scope, id, props);
    this.vpc = new ec2.Vpc(this, 'Vpc', { maxAzs: props.config.vpcMaxAzs });
    this.cluster = new ecs.Cluster(this, 'Cluster', { vpc: this.vpc });
  }
}
```

Cela vous donne gratuitement la sûreté à la compilation et le bon ordre de déploiement. CDK ne vous laissera pas déployer `DataStack` avant `NetworkStack` parce que la dépendance est dans le graphe.

Il y a une arête tranchante à connaître : CloudFormation interdit de supprimer une valeur exportée tant qu'une autre stack l'importe. Si vous devez retirer une référence inter-stacks, faites-le en deux déploiements, sinon vous obtenez *« Export cannot be deleted as it is in use. »* Le helper `exportValue()` de CDK existe pour épingler un export à travers ce genre de refactorisation.

**2. SSM Parameter Store (pour un couplage lâche).** Quand deux stacks ne doivent pas être liées étroitement à la synthèse, le producteur écrit une valeur dans SSM et le consommateur la lit. Cela casse délibérément la dépendance CloudFormation, ce qui est justement voulu entre applications déployées séparément ou entre équipes.

```typescript
// Stack productrice
new ssm.StringParameter(this, 'DbEndpointParam', {
  parameterName: `/platform/${config.name}/data/db-endpoint`,
  stringValue: this.cluster.clusterEndpoint.hostname,
});

// Stack consommatrice, déployée indépendamment
const dbEndpoint = ssm.StringParameter.valueForStringParameter(
  this, `/platform/${config.name}/data/db-endpoint`,
);
```

Le compromis est explicite. Les références directes donnent la sûreté de type et l'ordre, mais couplent les déploiements. SSM donne l'indépendance, mais le contrat est désormais un chemin sous forme de chaîne et une lecture au runtime, sans compilateur pour attraper une faute de frappe. **Règle empirique :** références directes au sein d'une même application et des stacks d'une même équipe ; SSM (ou un registre équivalent) aux coutures entre applications déployées indépendamment.

Quel que soit votre choix, gardez la surface partagée petite. Une stack doit exposer une poignée de sorties intentionnelles (un VPC, un cluster, un bus), pas tout son graphe interne de ressources. Les champs public readonly d'une stack *sont* son API. Traitez-les comme telle.

## Cross-account et cross-region

L'astuce automatique export/import ne marche qu'au sein d'un même compte et d'une même région. Une grosse plateforme franchit les deux frontières, et deux cas reviennent sans cesse :

**Cross-region (le cas CloudFront/ACM).** CloudFront exige son certificat ACM en `us-east-1`, mais votre plateforme vit en `eu-west-1`. Vous ne pouvez pas passer une référence de construct à travers les régions. Les paramètres SSM ne se lisent pas non plus cross-region. Les réponses propres sont `crossRegionReferences: true` sur les stacks (CDK provisionne des custom resources pour faire transiter les valeurs), ou provisionner le certificat dans une stack `us-east-1` dédiée et passer son ARN.

```typescript
const edge = new EdgeStack(this, 'Edge', {
  env: { account: config.account, region: 'us-east-1' },
  crossRegionReferences: true,
});
const web = new WebStack(this, 'Web', {
  env: { account: config.account, region: 'eu-west-1' },
  crossRegionReferences: true,
  certificate: edge.certificate, // CDK comble l'écart de région
});
```

**Cross-account.** Là, aucun pont gratuit. Le compte producteur écrit dans SSM (ou expose une ressource via une resource policy et un rôle partagé), et le compte consommateur lit avec un rôle IAM cross-account explicite. N'essayez pas de faire franchir les comptes aux exports CloudFormation ; ils ne le peuvent pas. Modélisez le contrat comme de la donnée (un chemin SSM, une convention d'ARN bien connue) et accordez l'accès cross-account minimal pour le lire.

La leçon se répète : plus deux stacks sont éloignées (même app, même compte, même région), plus leur contrat doit être lâche et explicite. En intra-app vous avez les types ; entre comptes vous avez des chaînes et de l'IAM.

## Le pipeline de déploiement

Le `cdk deploy` manuel ne survit pas au contact de trois environnements et d'une équipe. CDK Pipelines vous donne un pipeline auto-mutant : le pipeline est lui-même défini en CDK, et quand vous changez sa définition, il se met à jour lui-même avant de lancer vos stages.

```typescript
const pipeline = new CodePipeline(this, 'Pipeline', {
  synth: new ShellStep('Synth', {
    input: CodePipelineSource.connection('org/platform', 'main', {
      connectionArn: config.codestarConnectionArn,
    }),
    commands: ['npm ci', 'npm run build', 'npx cdk synth'],
  }),
});

// Promeut les mêmes stacks synthétisées, environnement par environnement.
pipeline.addStage(new PlatformStage(app, 'dev', environments.dev));

pipeline.addStage(new PlatformStage(app, 'stage', environments.stage), {
  pre: [new ShellStep('IntegrationTests', { commands: ['npm run test:integration'] })],
});

pipeline.addStage(new PlatformStage(app, 'prod', environments.prod), {
  pre: [new ManualApprovalStep('PromoteToProd')],
});
```

La propriété importante, c'est que le *même artefact synthétisé* traverse dev, puis stage, puis prod. Vous ne re-synthétisez pas par environnement en espérant qu'ils correspondent. Dev prouve que le changement se déploie, stage lance les tests d'intégration contre lui, un humain approuve, et prod reçoit les templates octet pour octet identiques qui ont déjà fonctionné deux fois. Les différences d'environnement se réduisent entièrement à la table de config, donc la promotion est ennuyeuse par conception, et l'ennuyeux est le but pour un déploiement de prod.

## Pièges

- **Ne découpez pas trop finement.** Une stack par ressource est aussi mauvaise qu'une stack par plateforme. Douze stacks minuscules avec une toile d'exports entre elles est pire que trois bien tracées. Découpez sur la propriété et le rythme de changement, pas sur le type de ressource.
- **Attention au piège de suppression d'export.** Retirer une référence inter-stacks encore importée fait échouer le déploiement. Refactorez les contrats inter-stacks en deux temps : arrêtez d'importer, déployez, puis arrêtez d'exporter.
- **Ne codez jamais en dur les IDs de compte dans le code des constructs.** Ils appartiennent à la table de config, résolus une seule fois. Un ID en dur, c'est comme ça qu'un déploiement de stage atterrit en prod.
- **Gardez les stacks avec état séparées et protégées.** Bases de données et buckets appartiennent à leur propre stack avec `deletionProtection` et une removal policy `RETAIN` en prod. Ne laissez jamais un déploiement de service sans état pouvoir remplacer une base de données.
- **Les stacks agnostiques d'environnement vous mentent.** Une stack sans `env` explicite se synthétise avec des pseudo-paramètres `Fn` et masque le vrai comportement cross-region et AZ. Réglez toujours `env` depuis la config pour que la synthèse reflète la réalité.
- **Les contrats SSM n'ont pas de compilateur.** Si vous couplez via des chemins de paramètres, traitez le schéma de chemin (`/platform/<env>/<domaine>/<clé>`) comme une interface versionnée et changez-le délibérément.

## Conclusion

Le passage d'une seule grosse stack à une vraie plateforme n'est pas une question d'écrire plus de CDK. C'est de tracer trois ou quatre frontières honnêtes : fondation, données et application, découpées selon qui les possède et à quelle fréquence elles changent. À l'intérieur de ces frontières, les références directes de constructs vous donnent gratuitement un câblage sûr au type. Aux coutures, les paramètres SSM et l'IAM explicite vous donnent un couplage lâche à dessein. Une unique table de config typée transforme dev, stage et prod en le même code avec des nombres différents, et un pipeline auto-mutant promeut un seul artefact à travers les trois.

Tracez bien les coutures et la plateforme passe à l'échelle latéralement : ajouter un service, c'est ajouter une stack ; intégrer une équipe, c'est lui confier la propriété d'une frontière ; et un déploiement de prod, c'est le même changement déjà livré en dev et stage, promu sans drame.
