# Explicit `AssociatePublicIpAddress` on all EC2 network interfaces

## Problem

AWS organizational policies (SCPs, Config rules) can require that every
`ec2:RunInstances` call includes an explicit `AssociatePublicIpAddress` value in
its `NetworkInterfaces` block. When the field is **omitted**, EC2 falls back to
the subnet's `MapPublicIpOnLaunch` setting, which policy scanners treat as
non-deterministic and reject.

ParallelCluster omits `AssociatePublicIpAddress` in several places, causing
cluster creation to fail in policy-restricted accounts.

## Changes implemented

### Change 1 — Compute nodes: default `AssignPublicIp` to `false`

**File:** `cli/src/pcluster/templates/queues_stack.py`

When the user omits `AssignPublicIp` from queue networking config, the value is
`None`, and CDK omits the property from the CloudFormation template. Now defaults
to `False` when not specified:

```python
associate_public_ip_address=queue.networking.assign_public_ip if queue.networking.assign_public_ip is not None else False,
```

### Change 2a — Add `AssignPublicIp` to head node config

**File:** `cli/src/pcluster/schemas/cluster_schema.py`
- Added `assign_public_ip = fields.Bool(...)` to `HeadNodeNetworkingSchema`

**File:** `cli/src/pcluster/config/cluster_config.py`
- Added `assign_public_ip` parameter to `HeadNodeNetworking.__init__`

This allows users to specify `AssignPublicIp: false` under `HeadNode > Networking`
in the cluster YAML. The value is used by the dry-run validator (Change 3).

### Change 2b — Head node secondary interfaces: explicit `false`

**File:** `cli/src/pcluster/templates/cluster_stack.py`

Secondary network card interfaces (multi-NIC instance types) now explicitly set
`associate_public_ip_address=False`.

### Change 3 — Dry-run validator: always set `AssociatePublicIpAddress`

**File:** `cli/src/pcluster/validators/cluster_validators.py`

`_build_launch_network_interfaces` now sets `AssociatePublicIpAddress` on every
interface: primary gets the `use_public_ips` value, secondary interfaces get
`False`. Previously the field was only set when `True` and multi-NIC.

`HeadNodeLaunchTemplateValidator` now passes `use_public_ips=bool(head_node.networking.assign_public_ip)`
to use the new config value.

## Head node primary interface — known limitation

The head node primary interface uses a **pre-created ENI** (`AWS::EC2::NetworkInterface`)
referenced by `NetworkInterfaceId` in the launch template. AWS API does not allow
`AssociatePublicIpAddress` alongside `NetworkInterfaceId` — they are mutually exclusive.

**Why the ENI can't be removed:**

The ENI serves as a **circular dependency breaker** in CloudFormation:

```
HeadNodeENI (created first, gets private IP)
    ↓
ComputeFleet launch templates (user data references ENI's private IP)
    ↓
HeadNode instance (DependsOn ComputeFleet, attaches ENI)
```

- Compute fleet user data needs the head node's private IP
- Head node must be created AFTER compute fleet (its bootstrap reads fleet config)
- Without the ENI as intermediary, `ComputeFleet → HeadNode` (via GetAtt PrivateIp)
  and `HeadNode → ComputeFleet` (via DependsOn) would be a circular dependency

**Impact:** The head node with pre-created ENI is inherently safe — a pre-existing
ENI attached at launch does not auto-assign public IPs. Public IP only comes from
an explicit EIP. However, strict SCPs that deny when `AssociatePublicIpAddress` is
absent from ANY network interface in the request will still block this.

**Workaround for strict SCPs:** Add a condition to the SCP that excludes requests
where `NetworkInterfaceId` is present (since public IP behavior is deterministic
for pre-existing ENIs):

```json
{
  "Condition": {
    "Null": {
      "ec2:NetworkInterfaceId": "true"
    }
  }
}
```

This makes the deny only apply when creating NEW interfaces (no `NetworkInterfaceId`),
not when attaching existing ones.

## Files changed

| File | Change |
|------|--------|
| `cli/src/pcluster/templates/queues_stack.py` | Default `associate_public_ip_address` to `False` when `None` |
| `cli/src/pcluster/schemas/cluster_schema.py` | Add `assign_public_ip` to `HeadNodeNetworkingSchema` |
| `cli/src/pcluster/config/cluster_config.py` | Add `assign_public_ip` to `HeadNodeNetworking` |
| `cli/src/pcluster/templates/cluster_stack.py` | Add `associate_public_ip_address=False` on head node secondary interfaces |
| `cli/src/pcluster/validators/cluster_validators.py` | Always set `AssociatePublicIpAddress` on all dry-run interfaces |

## Example YAML config

```yaml
HeadNode:
  InstanceType: t3a.micro
  Networking:
    SubnetId: subnet-0cd7b93140ec7c0d2
    AssignPublicIp: false          # NEW — used by dry-run validator
    ElasticIp: false
    SecurityGroups:
      - sg-00998dfa6aabe1fc9
      - sg-0dd367c16e8c16e24
  Ssh:
    KeyName: artem

Scheduling:
  SlurmQueues:
    - Name: myqueue
      Networking:
        SubnetIds:
          - subnet-xxx
        AssignPublicIp: false      # Already supported; now defaults to false when omitted
```
