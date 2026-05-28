# ansible-squid-proxy

Regional-HA Squid forward proxy on AWS Amazon Linux 2023, fronted by an
internal Network Load Balancer. Hardened to CIS Benchmark Level 1 via
`ansible-lockdown.amzn2023_cis`. Everything driven from variables in
`inventory/group_vars/`.

## Architecture

```
       ┌──────────────┐
       │ VPC clients  │  HTTP(S)_PROXY=http://<nlb-dns>:3128
       │ (10.0.0.0/8) │
       └──────┬───────┘
              │
   ┌──────────▼──────────┐
   │  Internal NLB :3128 │       (regional, multi-AZ)
   └─────┬──────────┬────┘
         │          │
   ┌─────▼──┐  ┌────▼───┐
   │ squid- │  │ squid- │     Amazon Linux 2023
   │ proxy- │  │ proxy- │     CIS L1 hardened
   │   01   │  │   02   │     Squid 3128/tcp
   │  AZ-a  │  │  AZ-b  │
   └────────┘  └────────┘
```

* **2× EC2** instances, one per AZ, in pre-existing private subnets.
* **NLB** (TCP) provides the stable "virtual IP" (DNS name with stable VPC ENIs).
* **NLB SG** allows ingress on 3128 from `allowed_client_cidrs` only.
* **Instance SG** allows 3128 *only* from the NLB SG (defence in depth).
* **IMDSv2** enforced. **EBS encryption** on. **No public IPs**.

## Repo layout

```
ansible.cfg
requirements.yml
inventory/
├── aws_ec2.yml                       # dynamic AWS inventory (tag-filtered)
└── group_vars/
    ├── all.yml                       # region, vpc, subnets, tags, CIDRs
    ├── tag_Role_squid_proxy.yml      # squid + NLB + EC2 + CIS knobs
    └── vault.yml.example             # placeholder for ansible-vault secrets
playbooks/
├── site.yml         # provision + configure (end-to-end)
├── provision.yml    # AWS infra + EC2 launches
├── configure.yml    # CIS hardening + squid config (existing hosts)
├── patch.yml        # rolling in-place `dnf update` (security patches)
└── replace.yml      # rolling immutable replacement (OS/AMI upgrades)
roles/
├── aws_infra/       # security groups, NLB, target group, listener
├── aws_ec2/         # AMI lookup + EC2 launch + target group registration
├── cis_hardening/   # wrapper around ansible-lockdown.amzn2023_cis
├── squid/           # install/configure squid + logrotate
└── lifecycle/       # deregister / drain / register / wait-healthy helpers
```

## Prerequisites

1. AWS credentials available to the control node (env vars, profile,
   or instance role).
2. A VPC with **at least two private subnets in different AZs**.
3. An IAM instance profile suitable for SSM Session Manager
   (`AmazonSSMManagedInstanceCore` policy). The profile name is set in
   `ec2_iam_instance_profile`.
4. Python 3 + `boto3`/`botocore` on the control node.

## First-time setup

```bash
# 1. Install collections + roles
ansible-galaxy install -r requirements.yml
ansible-galaxy collection install -r requirements.yml

# 2. Edit inventory/group_vars/all.yml
#    - vpc_id
#    - private_subnets[].id and .az
#    - allowed_client_cidrs
#    - admin_ssh_cidrs (or leave [] for SSM-only)

# 3. Edit inventory/group_vars/tag_Role_squid_proxy.yml
#    - squid_allowed_dst_domains (or set to [] to allow everything)
#    - ec2_instance_type, ec2_key_name
#    - squid_instances (one per AZ)

# 4. Provision + configure
ansible-playbook playbooks/site.yml
```

The final `provision.yml` task prints the NLB DNS name; that's your
"virtual IP" for clients:

```bash
export HTTP_PROXY=http://<nlb-dns>:3128
export HTTPS_PROXY=http://<nlb-dns>:3128
```

## Day-2 operations

### Routine security patching (in-place)

`patch.yml` rolls hosts one at a time: deregister → wait drain →
`dnf update` → reboot if needed → re-register → wait healthy → next.

```bash
ansible-playbook playbooks/patch.yml
```

### OS / AMI upgrade (immutable replacement)

For minor-version AL2023 bumps or anything that warrants a fresh box,
`replace.yml` re-launches each instance from the latest AMI:

```bash
# replace both instances, one at a time
ansible-playbook playbooks/replace.yml

# replace a single instance
ansible-playbook playbooks/replace.yml --limit squid-proxy-01
```

The dataflow per instance:

1. `community.aws.elb_target` deregisters the old target.
2. `pause` for `rolling_drain_wait_sec` (defaults to
   `nlb_deregistration_delay_sec + 30`).
3. `amazon.aws.ec2_instance state=absent` terminates the box.
4. `amazon.aws.ec2_instance state=running` launches a new one in the
   same AZ from the latest AL2023 AMI.
5. `ansible_host` is reset to the new private IP, connection reset, and
   the `cis_hardening` + `squid` roles run against the fresh host.
6. `community.aws.elb_target state=present` re-attaches it.
7. We poll `community.aws.elb_target_info` until `state=healthy` before
   moving to the next instance.

### Pushing config-only changes

If you've only tweaked `squid.conf.j2` or a Squid variable:

```bash
ansible-playbook playbooks/configure.yml --tags squid
```

## Variables you'll commonly touch

| Variable                       | Where                                  | Why |
|--------------------------------|----------------------------------------|-----|
| `aws_region`                   | `group_vars/all.yml`                   | AWS region |
| `vpc_id`, `private_subnets`    | `group_vars/all.yml`                   | Where to drop the NLB + EC2 |
| `allowed_client_cidrs`         | `group_vars/all.yml`                   | Who may reach the proxy (NLB SG + squid ACL) |
| `admin_ssh_cidrs`              | `group_vars/all.yml`                   | Who may SSH directly to the instances |
| `squid_instances`              | `group_vars/tag_Role_squid_proxy.yml`  | Names + AZ placement |
| `squid_allowed_dst_domains`    | `group_vars/tag_Role_squid_proxy.yml`  | Destination allowlist |
| `ec2_instance_type`            | `group_vars/tag_Role_squid_proxy.yml`  | Box size |
| `nlb_deregistration_delay_sec` | `group_vars/tag_Role_squid_proxy.yml`  | Connection drain on rolling ops |
| `rolling_health_wait_sec`      | `group_vars/tag_Role_squid_proxy.yml`  | Max wait for fresh instance health |
| `cis_level_2_server`           | `group_vars/tag_Role_squid_proxy.yml`  | Toggle CIS L2 (default off) |

## Suggestions / further work

* **Auto Scaling Group** instead of fixed instances — `min=max=2`,
  health-check-type=ELB, instance refresh would replace `replace.yml`
  with native AWS rolling updates. Cleaner but adds a layer.
* **Squid on systemd-resolved**: AL2023 ships `systemd-resolved`;
  consider tuning `dns_nameservers` in squid.conf if you have a private
  Route 53 zone.
* **CloudWatch Agent**: ship `access.log` / `cache.log` to CloudWatch
  Logs. Easy to add as a role; needs IAM perms on the instance profile.
* **Squid SSL bumping**: out of scope for IP-allowlist auth, but if you
  later need URL-level (not just host-level) filtering on HTTPS, add
  `ssl_bump` with a private CA — requires deploying the CA cert to
  client trust stores.
* **Health check upgrade**: TCP health check just verifies the port is
  open. For deeper checking, switch to an HTTP healthcheck against an
  endpoint that squid serves (e.g. a tiny sidecar) — but it costs a
  background process per instance.
* **Per-environment overrides**: split `inventory/` into
  `inventory/prod/` and `inventory/staging/` once you outgrow a single
  region.
