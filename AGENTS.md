# AGENTS.md — ec2-elastic-ip-manager

Operational instructions for Claude Code, Codex, and other autonomous agents working in this repo. This file covers only what's specific to `ec2-elastic-ip-manager`; for the broader FiZZ picture, use the wiki and public docs linked below.

## What this repo is

A Lambda that **associates Elastic IPs from a tagged pool onto tagged EC2 instances** as they reach `running`, and releases them on termination. Scout24 uses it to give every FiZZ build/deploy agent a stable outbound public IP so that per-tenant Jenkins ALB SGs only need to allow-list a bounded set.

**This is a fork of `binxio/ec2-elastic-ip-manager`** with Scout24-specific modifications. The deltas are documented in `fizz-mods.md` — read it before merging anything from upstream.

### How FiZZ uses it

- A pool of EIPs is allocated once in each AWS account and all tagged `elastic-ip-manager-pool = fizz-agent-eip-pool`.
- Each FiZZ agent ASG has `elastic-ip-manager-pool = fizz-agent-eip-pool` propagated at launch (set via `DynamicElasticIpTag` on the cluster stack).
- When an agent instance reaches `running`, this Lambda picks a free EIP with the matching tag and associates it; on termination, the EIP returns to the pool.
- Pool size is a hard ceiling on concurrent FiZZ agents. **Growing the pool is manual** (AWS Support case for the EIP quota + a one-off tagging script). This repo does not provide IaC for the pool itself.
- Prod: 48 EIPs in account `279671539266`. Playground (`s24-platform-dev`, account `528053483448`): **no pool** — agents there get auto-assigned public IPs that change on every replace.

## Required local reading before edits

1. **FiZZ wiki** — https://wiki.scout24.com/spaces/ap/pages/26911/fizz — how the EIP pool fits into FiZZ.
2. `fizz-mods.md` — **required reading**. Exact deltas from upstream `binxio`: FiZZ-specific custom metric, the two CloudWatch alarms, and Makefile adjustments.
3. `README.md` — upstream deployment story (partially out of date for Scout24; cross-check with `fizz-mods.md`).

## Repo map

| Path | What's there |
|---|---|
| `src/elastic_ip_manager/manager.py` | Lambda handler. Scout24-modified: emits the pool-remaining custom metric. |
| `cloudformation/elastic-ip-manager.yaml` | Lambda + CloudWatch alarms. Scout24-modified: the two FiZZ alarms (pool-almost-empty, association-failure). |
| `cloudformation/artifact-bucket.yaml` | New in Scout24 fork — our own artifact bucket for the Lambda ZIP. |
| `Makefile`, `Makefile.mk`, `.make-release-support` | Build + deploy. Scout24-modified for the artifact bucket and build-number versioning. |
| `tests/` | Unit tests. |
| `Dockerfile.lambda` | Lambda build container. |
| `fizz-mods.md` | **Required reading before any upstream merge.** |

## Load-bearing constraints

### Do not merge upstream `binxio` changes blindly

- Upstream uses a public artifact bucket; we use our own via `cloudformation/artifact-bucket.yaml`. Replacing our Makefile artefact flow with upstream's breaks our deploy.
- Upstream's `manager.py` does not emit the pool-remaining metric; our two CloudWatch alarms depend on it. A re-sync that drops the metric blinds the on-call to EIP exhaustion.
- Follow the `fizz-mods.md` checklist on every upstream reconciliation.

### Scaling behaviour

- The Lambda is triggered by EC2 instance state-change events in **every region/account where it is deployed**. Scope the deployment to the FiZZ accounts; a stray deploy elsewhere will start taking EIPs from any tagged pool it finds.
- Tag `elastic-ip-manager-pool` value **must match** between the pool EIPs and the ASG `PropagateAtLaunch` tag. A typo is silent — the Lambda just skips the instance.

### Pool management is out-of-band

- There is no IaC for the pool size. If a PR here claims to "grow the EIP pool", it is doing something else — the pool itself is managed by:
  1. A VPC EIP quota increase (AWS Support case against the FiZZ account).
  2. `aws ec2 allocate-address` runs that tag the new EIPs with `elastic-ip-manager-pool=fizz-agent-eip-pool`.
- Both are manual by owner choice; don't propose adding them to this Lambda.

## Branches that matter

- `master` → build + publish the Lambda ZIP to the artifact bucket, re-deploy the Lambda stack in each FiZZ account.
- Upstream `binxio/master` is **not a merge target** — treat it as reference. Reconcile deliberate deltas through `fizz-mods.md`.

## Change conventions

### Alarm or metric changes

- Any change to the pool-remaining metric or the two Scout24 alarms needs:
  - A note in the PR explaining which Datadog/CloudWatch dashboards consume it (several on-call runbooks reference these alarm names).
  - Updating any downstream dashboard JSON that keys off the metric name.

### Lambda IAM

- The Lambda needs `ec2:DescribeInstances`, `ec2:DescribeAddresses`, `ec2:AssociateAddress`, `ec2:DisassociateAddress`, and `cloudwatch:PutMetricData`. Keep the policy tight — the Lambda runs with cross-account visibility of EIPs, which is more than it needs for most of its logic.

## PR conventions

A good PR body here answers:

1. **Upstream reconciliation?** — yes/no. If yes, which upstream commit range, and which Scout24 mods were preserved vs re-applied.
2. **Alarm/metric impact** — "no change", or exact list of affected metric names and alarm ARNs.
3. **Deploy plan** — which account(s), and in what order (prod `279671539266` is last).
4. **Rollback** — redeploy the previous Lambda ZIP from the artifact bucket; EIP state in-flight at the moment of the swap is not affected.

## When blocked

- Lambda associates an EIP but the ALB SG still blocks the host → this is **not** this Lambda's problem. The ALB SG allow-list is maintained in the consumer stack (`fizz-service-shared` for prod, DP-909 tracks playground). Point the reporter at the correct repo.
- Pool-remaining metric drops to zero in prod → page the FiZZ owner; the fix is a new AWS Support case for EIP quota, not a code change here.
- Deployment fails on the artifact bucket → check `cloudformation/artifact-bucket.yaml` is already stood up in the target account; it's a one-time per-account bootstrap.

## Attribution

- Commits by Claude Opus end with `Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>`.
- PR descriptions by Claude Code end with `🤖 Generated with [Claude Code](https://claude.com/claude-code)`.
- Codex follows its own attribution convention.
