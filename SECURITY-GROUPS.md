# RoboShop security groups

`security-groups.yaml` creates only the 11 service groups using
the [RoboShop README](https://github.com/daws-92s/roboshop-documentation#security-groups).
`roboshop-common` must already exist; this playbook does not create or modify it.
Keep shared SSH access configured there. Attach both the common group and the
appropriate service group to every service instance, as in `roboshop.yaml`.

Launch your Ansible controller with the existing common group first,
then run this playbook before provisioning the service instances.

## Where this runs

The playbook is set up for the Stockholm practice account:

| Setting | Value |
|---|---|
| Region | `eu-north-1` (the account is limited to this region) |
| VPC | `vpc-09dc55d870bfce035` (default VPC) |
| Common group | `roboshop-common` (`sg-0166c73a51017be2d`) |

Both region and VPC are play variables, so `-e region=...` or `-e vpc_id=...` overrides them.
The EC2 playbook doesn't specify a subnet, so instances land in the default VPC. For a custom VPC,
pass its `vpc_id` here and set `vpc_subnet_id` in the EC2 creation task to a subnet in that VPC.

## Prerequisites

On the Ansible controller (RHEL 9, `ansible-core 2.14`):

```sh
sudo dnf install -y python3-pip
pip3 install --user boto3 botocore
ansible-galaxy collection install 'amazon.aws:>=7.0.0,<8.0.0'   # 8.x+ needs ansible-core 2.15+
```

AWS access comes from the instance role `roboshop-ansible-role` (EC2 + Route 53 permissions)
attached to the controller. No access keys on the server.

## Run

```sh
ansible-playbook security-groups.yaml
```

## How it works

The first loop creates all service groups without removing existing rules.
The second loop applies the full rule list, so group references already exist.
`group_name` identifies the source group allowed to connect to the destination
group on the specified port. Dispatch has no service inbound rules; its SSH
access comes from the common group. Outbound IPv4 traffic is allowed for package
downloads and service connections.

The final pass uses `purge_rules: true`: existing inbound rules not listed here
are removed. Ensure the common group is attached to existing instances before
running, since service-specific SSH rules will be removed.

**Note:** these per-service rules only restrict traffic if `roboshop-common` allows
just port 22. If `roboshop-common` allows all traffic between its members, every
server can reach every port on every other server anyway.

## RabbitMQ management (optional)

Disabled by default. To allow TCP 15672 from your own IP, append:

```sh
-e 'rabbitmq_management_enabled=true admin_cidr=YOUR_PUBLIC_IP/32'
```

This only opens the port; the RabbitMQ management plugin must also be enabled.

The security group playbook always creates all 11 service groups. The EC2
playbook's `instances` variable separately controls which servers are created.
