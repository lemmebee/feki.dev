+++
date = '2026-07-14T10:00:00+02:00'
draft = false
title = 'Multi-Stack CDK Architecture for a Big-Scale Platform Across Dev, Stage, and Prod'
+++
**tl;dr:** One giant CloudFormation stack does not scale to a real platform. It hits resource limits, couples unrelated teams, and turns every deploy into a blast-radius gamble. This post lays out how to structure an AWS CDK app as a set of purpose-built stacks that communicate through typed references and SSM parameters, how to make one codebase serve dev, stage, and prod without copy-paste, and how to wire it all into a self-mutating pipeline that promotes changes environment by environment.

---
## Table of contents
- [Why Not One Big Stack](#why-not-one-big-stack)
- [Designing the Architecture First](#designing-the-architecture-first)
- [Splitting the Platform into Stacks](#splitting-the-platform-into-stacks)
- [One Codebase, Three Environments](#one-codebase-three-environments)
- [How Stacks Talk to Each Other](#how-stacks-talk-to-each-other)
- [Cross-Account and Cross-Region](#cross-account-and-cross-region)
- [The Deployment Pipeline](#the-deployment-pipeline)
- [Pitfalls](#pitfalls)
- [Wrap](#wrap)

## Why Not One Big Stack

A CDK app starts small. One stack, a VPC, a database, a couple of services. It works, so it grows. Six months later that single stack has two hundred resources, three teams committing to it, and a CloudFormation template pushing the 500-resource hard limit. Every one of those problems is structural, not incidental:

- **Resource limits are real.** CloudFormation caps a stack at 500 resources. A busy platform blows past that with load balancers, security groups, task definitions, and log groups alone.
- **The blast radius is the whole stack.** A bad change to a Lambda rolls back the VPC update sitting next to it. A stuck resource blocks every unrelated deploy behind it.
- **Deploys serialize.** One stack means one `cdk deploy` at a time. Teams queue behind each other instead of shipping in parallel.
- **Ownership blurs.** When networking, data, and application code all live in one file, nobody owns any of it, and every review needs everyone.

The fix is not fewer resources. It is drawing boundaries so that things that change together live together, and things that change on different cadences live apart.

## Designing the Architecture First

Before a line of CDK, decide the seams. The useful axis is *rate of change* combined with *ownership*. Foundational infrastructure (VPC, subnets, transit gateway) changes rarely and is owned by a platform team. Stateful resources (databases, buckets, queues) change occasionally and carry data you must never accidentally destroy. Stateless application services change many times a day and are owned by product teams.

Those three cadences map cleanly onto three layers:

```
  ┌──────────────────────────────────────────────────────────┐
  │  Layer 3: Application  (changes hourly, product teams)     │
  │  services, workers, APIs, front-end distributions          │
  └───────────────┬──────────────────────────────────────────┘
                  │ consumes: db endpoints, queue ARNs, VPC
  ┌───────────────┴──────────────────────────────────────────┐
  │  Layer 2: Data  (changes weekly, protected, data-bearing)  │
  │  RDS/Aurora, DynamoDB, S3, ElastiCache, SQS/SNS            │
  └───────────────┬──────────────────────────────────────────┘
                  │ consumes: VPC, subnets, security groups
  ┌───────────────┴──────────────────────────────────────────┐
  │  Layer 1: Foundation  (changes rarely, platform team)      │
  │  VPC, subnets, NAT, routing, shared IAM, KMS, DNS zones     │
  └──────────────────────────────────────────────────────────┘
```

Dependencies only ever point downward. Layer 3 depends on Layer 2 and Layer 1. Layer 1 depends on nothing. If you ever find yourself wanting a foundation stack to read something from an application stack, the boundary is wrong.

## Splitting the Platform into Stacks

In CDK, a **stack** is a unit of deployment and a **construct** is a unit of composition. The mistake is treating them as the same thing. You compose reusable constructs freely, then place them into a deliberate, small set of stacks.

A practical decomposition for a big platform:

```
platform-app (cdk.App)
├── NetworkStack        VPC, subnets, NAT, security groups, Route53 zones
├── SecurityStack       KMS keys, shared IAM roles, secrets
├── DataStack           Aurora cluster, DynamoDB tables, S3 buckets
├── MessagingStack      SQS queues, SNS topics, EventBridge bus
├── ServiceStack (×N)   one per bounded context: ECS/Lambda + ALB rules
└── EdgeStack           CloudFront, WAF, ACM certs (us-east-1)
```

Each `ServiceStack` is instantiated once per bounded context. `checkout`, `payments`, and `catalog` each get their own stack, their own deploy, their own owner, and their own blast radius. Adding a service adds a stack, not two hundred lines to an existing one.

Here is the shape of a stack that takes its dependencies as typed constructor props rather than reaching for them globally:

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

    // Grant instead of hand-writing IAM: the construct knows the least privilege.
    props.database.connections.allowDefaultPortFrom(service.service);
    props.eventBus.grantPutEventsTo(service.taskDefinition.taskRole);
  }
}
```

Passing `vpc`, `cluster`, and `database` as props is the single most important choice in the whole design. It makes the dependency explicit, type-checked at synth time, and lets CDK wire the cross-stack plumbing for you.

## One Codebase, Three Environments

Dev, stage, and prod are not three codebases. They are one codebase evaluated with three configs. The anti-pattern is `if (env === 'prod')` scattered through construct code. The pattern is a single typed config object resolved once at the top of the app and threaded down as props.

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

**Use separate AWS accounts per environment.** It is the only boundary AWS enforces for you. An account boundary makes it impossible for a dev experiment to touch prod data, gives you clean per-environment billing, and lets IAM policies stay simple because they never have to distinguish environments inside one account.

The app entry point loops over the config and stamps out one full set of stacks per environment. A wrapper stage keeps each environment's stacks grouped:

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

One loop, three environments, zero duplicated infrastructure code. Prod differs from dev only in the numbers in the config table, which is exactly where the difference should live and the only place a reviewer has to look.

## How Stacks Talk to Each Other

This is the crux. Within one CDK app and one account, you have two mechanisms, and choosing between them is the whole game.

**1. Direct construct references (preferred).** When you pass `network.vpc` into `DataStack`, and both stacks are in the same app, CDK notices the cross-stack reference and generates a CloudFormation `Export`/`Fn::ImportValue` automatically. You write plain TypeScript; CDK writes the plumbing.

```typescript
const network = new NetworkStack(this, 'Network', { config });
// network.vpc is an ec2.IVpc exposed as a public readonly field
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

This gives you compile-time safety and correct deploy ordering for free. CDK will not let you deploy `DataStack` before `NetworkStack` because the dependency is in the graph.

There is one sharp edge worth knowing: CloudFormation forbids deleting an exported value while another stack imports it. If you need to remove a cross-stack reference, do it in two deploys, or you get *"Export cannot be deleted as it is in use."* CDK's `exportValue()` helper exists to pin an export across that kind of refactor.

**2. SSM Parameter Store (for loose coupling).** When two stacks should not be tightly bound at synth time, the producer writes a value to SSM and the consumer reads it. This deliberately breaks the CloudFormation dependency, which is what you want across separately deployed apps or across teams.

```typescript
// Producer stack
new ssm.StringParameter(this, 'DbEndpointParam', {
  parameterName: `/platform/${config.name}/data/db-endpoint`,
  stringValue: this.cluster.clusterEndpoint.hostname,
});

// Consumer stack, deployed independently
const dbEndpoint = ssm.StringParameter.valueForStringParameter(
  this, `/platform/${config.name}/data/db-endpoint`,
);
```

The trade-off is explicit. Direct references give you type safety and ordering but couple deploys. SSM gives you independence but the contract is now a string path and a runtime lookup, with no compiler to catch a typo. **Rule of thumb:** direct references inside one app and one team's stacks; SSM (or a similar registry) at the seams between independently deployed apps.

Whichever you pick, keep the shared surface small. A stack should expose a handful of intentional outputs (a VPC, a cluster, a bus), not its entire internal resource graph. The public readonly fields on a stack *are* its API. Treat them like one.

## Cross-Account and Cross-Region

The automatic export/import trick works only inside a single account and region. A big platform crosses both boundaries, and two cases come up constantly:

**Cross-region (the CloudFront/ACM case).** CloudFront requires its ACM certificate in `us-east-1`, but your platform lives in `eu-west-1`. You cannot pass a construct reference across regions. SSM parameters do not read cross-region either. The clean answers are `crossRegionReferences: true` on the stacks (CDK provisions custom resources to shuttle the values), or provisioning the cert in a dedicated `us-east-1` stack and passing its ARN.

```typescript
const edge = new EdgeStack(this, 'Edge', {
  env: { account: config.account, region: 'us-east-1' },
  crossRegionReferences: true,
});
const web = new WebStack(this, 'Web', {
  env: { account: config.account, region: 'eu-west-1' },
  crossRegionReferences: true,
  certificate: edge.certificate, // CDK bridges the region gap
});
```

**Cross-account.** Here there is no free bridge at all. The producing account writes to SSM (or exposes a resource via a resource policy and a shared role), and the consuming account reads with an explicit cross-account IAM role. Do not try to make CloudFormation exports span accounts; they cannot. Model the contract as data (an SSM path, a well-known ARN convention) and grant the minimum cross-account access to read it.

The lesson repeats: the further apart two stacks sit (same app, same account, same region), the looser and more explicit their contract must be. In-app you get types; across accounts you get strings and IAM.

## The Deployment Pipeline

Manual `cdk deploy` does not survive contact with three environments and a team. CDK Pipelines gives you a self-mutating pipeline: the pipeline is itself defined in CDK, and when you change its definition, it updates itself before running your stages.

```typescript
const pipeline = new CodePipeline(this, 'Pipeline', {
  synth: new ShellStep('Synth', {
    input: CodePipelineSource.connection('org/platform', 'main', {
      connectionArn: config.codestarConnectionArn,
    }),
    commands: ['npm ci', 'npm run build', 'npx cdk synth'],
  }),
});

// Promote the exact same synthesized stacks, environment by environment.
pipeline.addStage(new PlatformStage(app, 'dev', environments.dev));

pipeline.addStage(new PlatformStage(app, 'stage', environments.stage), {
  pre: [new ShellStep('IntegrationTests', { commands: ['npm run test:integration'] })],
});

pipeline.addStage(new PlatformStage(app, 'prod', environments.prod), {
  pre: [new ManualApprovalStep('PromoteToProd')],
});
```

The important property is that the *same synthesized artifact* flows through dev, then stage, then prod. You are not re-synthesizing per environment and hoping they match. Dev proves the change deploys, stage runs integration tests against it, a human approves, and prod gets the byte-identical templates that already worked twice. Environment differences are entirely the config table, so promotion is boring by design, and boring is the goal for a prod deploy.

## Pitfalls

- **Do not split too finely.** A stack per resource is as bad as a stack per platform. Twelve tiny stacks with a web of exports between them is worse than three well-drawn ones. Split on ownership and rate of change, not on resource type.
- **Watch the export-delete trap.** Removing a cross-stack reference that is still imported fails the deploy. Refactor cross-stack contracts in two steps: stop importing, deploy, then stop exporting.
- **Never hardcode account IDs in construct code.** They belong in the config table, resolved once. A hardcoded ID is how a stage deploy lands in prod.
- **Keep stateful stacks separate and protected.** Databases and buckets belong in their own stack with `deletionProtection` and a `RETAIN` removal policy in prod. Never let a stateless service deploy be able to replace a database.
- **Env-agnostic stacks lie to you.** A stack without an explicit `env` synthesizes with `Fn` pseudo-parameters and hides real cross-region and AZ behavior. Always set `env` from config so synth reflects reality.
- **SSM contracts have no compiler.** If you couple via parameter paths, treat the path scheme (`/platform/<env>/<domain>/<key>`) as a versioned interface and change it deliberately.

## Wrap

The move from one big stack to a real platform is not about writing more CDK. It is about drawing three or four honest boundaries: foundation, data, and application, split by who owns them and how often they change. Inside those boundaries, direct construct references give you type-safe wiring for free. At the seams, SSM parameters and explicit IAM give you loose coupling on purpose. A single typed config table turns dev, stage, and prod into the same code with different numbers, and a self-mutating pipeline promotes one artifact through all three.

Get the seams right and the platform scales sideways: adding a service is adding a stack, onboarding a team is handing them ownership of a boundary, and a prod deploy is the same change that already shipped to dev and stage, promoted without drama.
