# Contribution constraints

- Preserve Scout24-specific monitoring and artifact contracts during upstream reconciliation. Read [fizz-mods.md](fizz-mods.md), “FiZZ related modifications”, to identify the fork changes that must survive; verify them against the current implementation.
- Keep automatic pool growth outside association logic unless the task explicitly changes that responsibility. Limit IAM additions to calls the implementation requires.
- When changing pool matching, inspect discovery in [ec2_instance.py](src/elastic_ip_manager/ec2_instance.py) and [eip.py](src/elastic_ip_manager/eip.py), then check consumer tag propagation in [cluster.yaml](../infinity-cluster/templates/cluster.yaml), `AutoScalingGroup`. Coordinate affected consumers.
- For monitoring changes, compare `put_cloudwatch_metric` and its callers in [manager.py](src/elastic_ip_manager/manager.py) with the alarms in [elastic-ip-manager.yaml](cloudformation/elastic-ip-manager.yaml); preserve agreement on dimensions and emission conditions.
- For artifact changes, trace the shared deployment caller in [elastic-ip-manager.sh](../fizz-service-shared/deploy/elastic-ip-manager.sh) before altering ZIP naming or destinations. A sibling checkout alone does not establish which code the caller deploys.

# Validation

Select build and test checks from [Makefile](Makefile) and its included [Makefile.mk](Makefile.mk). Before running tests, inspect resource discovery and association calls in [test_manager.py](tests/test_manager.py); establish the target pool, AWS side effects, and task authorization. Before release targets, inspect their repository mutations and uploads in `Makefile.mk`.

For documentation edits, check links and the diff. For deployment work, report monitoring impact, affected consumers, deployment target, and available rollback artifact. Diagnose consumer networking before attributing connectivity failures to address association.
