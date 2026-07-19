---
title: "About"
type: "page"
url: "about/"
layout: "about"
date: 2025-04-16
passions: "Outside the tech world, I disconnect by holding my breath and diving deep into the sea as far as I can."
contact_intro: "A question, an opportunity, or just to say hello:"
email: "me@feki.dev"
products:
  - name: "issuance-gateway"
    url: "https://github.com/lemmebee/issuance-gateway"
    lang: "go"
    lang_label: "Go"
    desc: "Async certificate issuance gateway in Go. Worker pool, atomic idempotency, graceful shutdown, one-time PKCS#12 delivery, built-in fault injection."
  - name: "frfix"
    url: "https://github.com/lemmebee/frfix"
    lang: "python"
    lang_label: "Python"
    desc: "System-wide real-time French autocorrect daemon for Linux/X11, conservative by design."
  - name: "ouioui"
    url: "https://github.com/lemmebee/ouioui"
    lang: "python"
    lang_label: "Python"
    desc: "Full-stack French learning app. Larousse scraping, DeepL translation, SM-2 spaced repetition. FastAPI, HTMX, SQLite."
  - name: "lazyfy"
    url: "https://github.com/lemmebee/lazyfy"
    lang: "go"
    lang_label: "Go"
    desc: "Go TUI building Spotify playlists from daily featured songs."
---

I build for scale: the services, the pipelines, the cloud, and the tests that guard them.

Eight years across backend engineering, cloud infrastructure, and test automation. Most engineers own one of those. Owning the chain is what removes the handoffs that slow delivery down, and it is why I end up on the problems that sit between teams.

**Software.** At Contentsquare I built an engineering insights platform: a data pipeline ingesting AWS, Prometheus, Jira, GitHub, and TestRail, per-merge coverage shipped through a CI CLI, and 30+ database views behind a dashboard teams actually used. Outside work I ship in Go and Python: an async certificate issuance gateway with a worker pool, atomic idempotency, and built-in fault injection; a real-time French autocorrect daemon for Linux; a full-stack spaced-repetition app on FastAPI.

**Infrastructure.** At ORIS I cut AWS costs 30-50% through multi-account consolidation, EKS/Karpenter reconfiguration, NAT Gateway replacement (~90% egress reduction), and off-hours shutdown, and moved a microservice fleet to Helm/ArgoCD GitOps with automated drift detection. At Contentsquare, Terraform-based Kubernetes deployments for 15+ microservices to a 99.95% availability target, absorbing traffic spikes up to 300%.

**Delivery.** I re-architected CI/CD into reusable GitLab templates with automatic project detection across Node.js, Python, Java, and Go, and moved runners from EC2 to an autoscaling EKS spot nodegroup so suites run in parallel under load.

**Testing.** I came up through test automation and it still shapes how I build: quality gates (SonarQube, Trivy) wired into the pipeline rather than bolted on, contract tests that let a microservice fleet deploy independently, end-to-end suites parallelized to cut execution time ~60%.

Most of what I learn doing this ends up in the posts on this site.
