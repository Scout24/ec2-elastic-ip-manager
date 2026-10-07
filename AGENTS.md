# ec2-elastic-ip-manager

## Source of truth

These instructions apply to any contributor or coding agent. Read configuration from the owning code when needed; do not copy schedules, versions, resource identifiers, defaults, or pipeline behavior into this file. The links below identify where to look, not a snapshot of the values there. Verify deployed state separately when the task depends on it.

## Read for the task

- For event handling and association behavior, inspect [manager.py](src/elastic_ip_manager/manager.py), `handler` and `Manager`.
- For pool matching and tag keys, inspect [instance discovery](src/elastic_ip_manager/ec2_instance.py) and [EIP discovery](src/elastic_ip_manager/eip.py). Trace consumer values through [shared-stack.yaml](../fizz-service-shared/src/main/shared-stack.yaml) and tag propagation through [cluster.yaml](../infinity-cluster/templates/cluster.yaml).
- For reconciliation cadence, event filters, runtime, permissions, and alarms, read the corresponding resources in [Lambda template](cloudformation/elastic-ip-manager.yaml).
- For metric names, namespace, dimensions, and emission conditions, inspect `put_cloudwatch_metric` and its callers in [manager.py](src/elastic_ip_manager/manager.py); compare with the alarm definitions in the Lambda template.
- For artifact destinations, ZIP naming, build prerequisites, and upload/deploy steps, inspect [Makefile](Makefile), [Makefile.mk](Makefile.mk), and [artifact-bucket template](cloudformation/artifact-bucket.yaml).
- For consumer deployment integration, inspect [shared deployment script](../fizz-service-shared/deploy/elastic-ip-manager.sh) and its [pipeline caller](../fizz-service-shared/Jenkinsfile). Determine branch selection and deployment conditions from those files.
- For pool resource definitions, inspect [pool template](cloudformation/elastic-ip-pool.yaml). Establish live pool capacity and ownership from the target environment rather than inferring them from a template.

## Working rules

- For upstream reconciliation, read `fizz-mods.md` for context and verify that Scout24-specific changes survive against the implementation above.
- Preserve pool matching, monitoring, and artifact contracts unless the task explicitly changes them. Coordinate affected consumers.
- Keep IAM changes limited to required implementation calls. Compare the handler with the template's permission declarations.
- Keep automatic pool growth outside association logic unless the task explicitly changes that responsibility.

## Validation and completion

Read the build/test targets in [Makefile](Makefile) and [Makefile.mk](Makefile.mk) before selecting commands. Derive interpreter requirements and deployed runtime from their respective build and template settings.

Inspect [test_manager.py](tests/test_manager.py), especially resource discovery and association calls, before executing tests. Determine the affected pool and AWS side effects from the code and use only the task's authorized environment. Inspect release targets for repository mutations and uploads before running them.

For documentation-only changes, check the diff and source links. Report preserved upstream modifications, monitoring impact, checks performed, deployment targets, and rollback artifact. Investigate consumer networking when diagnosing connectivity; report unavailable evidence and permission failures explicitly.
