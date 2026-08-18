# 0007 — AWS in two shapes, not two clouds

**Status:** Accepted · 2026-08-18 · supersedes the Fly.io-then-AWS plan

## Context

The application has to be reachable at a URL. A portfolio project that only runs on
`localhost` is worth nothing, and free hosting tiers that sleep are worse than nothing —
Render spins a free service down after 15 minutes and takes about a minute to wake. A
recruiter clicking a CV link closes the tab in about eight seconds, and you never find out.

The original plan was Fly.io first, migrating to AWS after launch. The reasoning was speed:
Fly is an afternoon, AWS is a week.

That reasoning optimised for the wrong thing. AWS is what job listings ask for. Effort spent
learning a platform that gets thrown away is effort not spent on the thing this project
exists to demonstrate.

The obvious correction — "just build it on ECS properly" — has a cost problem that most
tutorials skip. Neither of the two components every ECS guide requires is free-tier eligible:

| Component | Cost |
|---|---|
| Application Load Balancer | ~$18/month |
| NAT Gateway | ~$32/month |

That is roughly **$50/month of networking before a single container runs**. The AWS free
tier is also no longer twelve months of free compute — it is credit-based, around $100 up
front and up to $200 over six months. The credits cover about four months of that shape,
and then it is $600/year for a side project.

## Decision

AWS from the first deploy. No Fly.io. But in **two shapes**, not one.

**S1 — a single EC2 instance.** `t4g.small` in a public subnet, Docker Compose on the box,
Caddy as a reverse proxy handling Let's Encrypt certificates automatically. RDS
`db.t4g.micro` reachable only from the instance security group. No load balancer, no NAT
gateway.

Live in a day or two, roughly $10–15/month, largely covered by credits. It still touches
VPC, security groups, EC2, IAM, ECR, Route 53 and CloudWatch — that is real AWS surface.

**S9 — ECS Fargate behind an ALB, with RDS in private subnets**, defined as
infrastructure-as-code. Stand it up, verify it, screenshot it, commit the stack and an
architecture document — then **tear it down**, leaving the S1 instance serving the live
demo.

## Alternatives rejected

**Fly.io first, AWS later.** The original plan. Fly is genuinely faster to reach and its
managed Postgres is pleasant. Rejected because it means learning a platform that gets
discarded, and because "I migrated from Fly to AWS" describes a migration nobody actually
runs. EC2-to-ECS is the migration companies really do.

**ECS Fargate from S1.** The architecture that reads best on a CV, built once instead of
twice. Rejected on two grounds: ~$50/month indefinitely, and one to two weeks of VPC, IAM
and task-definition work standing between the project and its first deploy. This milestone
exists to get something live in week one, and that risk is exactly how side projects die
undeployed.

**Render or Railway free tiers.** Free, and they sleep. Disqualifying — the demo has to be
instant or it does not count.

**A cheap VPS (Hetzner, DigitalOcean).** About €4/month, entirely adequate, and teaches more
about running a box than AWS does. Rejected because it puts nothing on the CV that anyone is
searching for.

## Consequences

**Good.** AWS appears from the first deploy. Something is live in week one. Monthly cost
stays near credits rather than near $50. The S9 migration is EC2-to-ECS, which is both more
relevant and more interesting than a cross-cloud move.

**Bad.** The live demo runs on a single instance, which is not a highly-available
architecture and should never be described as one. If that instance dies, the demo is down
until it is rebuilt — acceptable for a portfolio, and it must not be claimed as anything
more.

**Also bad.** Two deploy paths exist, and the S1 one is the one that keeps running. There is
a real risk the ECS work is treated as done once it is torn down and quietly rots. The
mitigation is that the stack is infrastructure-as-code and committed, so it can be raised
again from the repository rather than from memory.

**The teardown is deliberate.** "I built it on ECS Fargate behind an ALB — here is the stack
and the architecture doc; the live demo runs on a single instance because $50/month of load
balancer for a portfolio app is a bad trade" demonstrates both capability and cost
awareness. Most candidates who list AWS have never seen a bill.
