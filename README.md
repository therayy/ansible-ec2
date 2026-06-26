# Ansible EC2 Provisioning Lab

## Purpose

This lab demonstrates how Ansible can provision AWS EC2 instances, and why Terraform is usually still the better tool for infrastructure provisioning.

The main lesson is:

```text
Ansible can create infrastructure, but Terraform should usually own infrastructure lifecycle.
Ansible should usually own configuration.
```

In this lab, we tested two Ansible approaches:

1. Creating EC2 instances with `count: 1`
2. Creating EC2 instances with `exact_count: 1`

The goal was to see what happens when the same playbook runs more than once.

## Tools Used

* Ansible
* AWS EC2
* amazon.aws Ansible collection
* boto3 and botocore
* Ubuntu 24.04 AMI
* Nginx installed through EC2 user data

## Prerequisites

Install Python AWS dependencies:

```bash
pip install boto3 botocore
```

Install the Ansible AWS collection:

```bash
ansible-galaxy collection install amazon.aws
```

Configure AWS credentials:

```bash
aws configure
```

Or export an AWS profile and region:

```bash
export AWS_PROFILE=your-profile
export AWS_REGION=us-east-2
```

## AWS Region

This lab uses:

```text
us-east-2
```

Important note:

AWS key pairs are regional. A key pair that exists in `us-east-1` does not automatically exist in `us-east-2`.

If the playbook fails with this error:

```text
InvalidKeyPair.NotFound: The key pair 'ray-vault-key' does not exist
```

That means the key pair does not exist in the selected region.

Create the key pair in `us-east-2`:

```bash
mkdir -p ~/.ssh

aws ec2 create-key-pair \
  --region us-east-2 \
  --key-name ray-vault-key \
  --query 'KeyMaterial' \
  --output text > ~/.ssh/ray-vault-key-us-east-2.pem

chmod 400 ~/.ssh/ray-vault-key-us-east-2.pem
```

Verify the key exists:

```bash
aws ec2 describe-key-pairs \
  --region us-east-2 \
  --key-names ray-vault-key
```

## Files in This Lab

```text
ec2_count.yml
ec2_exact_count.yml
README.md
```

## What `ec2_count.yml` Does

The `ec2_count.yml` playbook launches an EC2 instance using:

```yaml
count: 1
```

This means:

```text
Launch one new EC2 instance.
```

When the playbook runs once, it creates one EC2 instance.

When the playbook runs again, it creates another EC2 instance.

So this is not safe if the goal is to enforce only one server.

## Run the Count Test

```bash
ansible-playbook ec2_count.yml
ansible-playbook ec2_count.yml
```

Expected result:

```text
First run creates one EC2 instance.
Second run creates another EC2 instance.
```

This proves that `count: 1` means launch one instance each time the task runs.

## What `ec2_exact_count.yml` Does

The `ec2_exact_count.yml` playbook uses:

```yaml
exact_count: 1
```

With filters:

```yaml
filters:
  "tag:Name": "{{ instance_name }}"
  "tag:ManagedBy": Ansible
  "tag:Lab": exact-count-test
  instance-state-name: running
```

This means:

```text
Make sure exactly one matching EC2 instance is running.
```

The filters are important because Ansible needs to know which instances count as matching instances.

The IP address should not be treated as the identity because the IP can change.

Instead, the playbook uses tags as the identity.

## Run the Exact Count Test

```bash
ansible-playbook ec2_exact_count.yml
ansible-playbook ec2_exact_count.yml
```

Expected result:

```text
First run creates one EC2 instance.
Second run does not create another matching EC2 instance.
```

This proves that `exact_count: 1` is closer to desired state behavior.

## Main Difference

| Option           | Meaning                                     | Second Run Behavior                       |
| ---------------- | ------------------------------------------- | ----------------------------------------- |
| `count: 1`       | Launch one instance                         | Creates another instance                  |
| `exact_count: 1` | Ensure exactly one matching instance exists | Does not create another matching instance |

## What We Learned

Ansible can provision EC2 instances.

However, how safe it is depends on how the playbook is written.

If the playbook uses `count: 1`, it can create duplicate infrastructure every time it runs.

If the playbook uses `exact_count: 1` with proper filters and tags, it can behave more like desired state.

This is why tagging matters.

For EC2 provisioning, the identity should be based on tags, filters, or instance IDs, not IP addresses.

## Why Terraform Is Still Better for Provisioning

Terraform is designed for infrastructure lifecycle management.

Terraform has:

* State file
* Plan and apply workflow
* Dependency graph
* Desired state model
* Better lifecycle management for create, update, and destroy

Ansible can create infrastructure, but Terraform is usually safer for provisioning because Terraform remembers what it created.

## Why Ansible Is Better for Configuration

Ansible is strong for configuration management.

Ansible is good for:

* Installing packages
* Starting services
* Managing files
* Applying system configuration
* Running Day 2 operations
* Rotating certificates
* Restarting services
* Handling OS-specific tasks

Example:

```yaml
- name: Install nginx
  ansible.builtin.package:
    name: nginx
    state: present
```

This is idempotent.

If Nginx is already installed, Ansible does not reinstall it unnecessarily.

## Idempotency Lesson

Idempotency means:

```text
If the system already matches the desired state, running the automation again should not make unnecessary changes.
```

Ansible modules are often idempotent when written correctly.

Example:

```yaml
state: present
```

Means:

```text
Make sure this thing exists.
If it already exists, do nothing.
```

But idempotency depends on the module and how the playbook is written.

For EC2:

```yaml
count: 1
```

Can create another server every time.

But:

```yaml
exact_count: 1
```

With proper filters can enforce one matching server.

## Terraform vs Ansible Positioning

The clean architecture is:

```text
Packer    = build golden images
Terraform = provision infrastructure
Ansible   = configure systems
```

Recommended pattern:

```text
Use Terraform to create EC2 instances.
Use Ansible to configure those EC2 instances.
Use Packer when you want to bake configuration into a golden image.
```

## Pets vs Cattle Lesson

For stateless servers, use a cattle model.

That means:

```text
Do not manually repair servers forever.
Replace them with clean servers from a known image.
```

If a stateless server needs a major change, such as OS patching or application upgrade:

```text
1. Update the image build
2. Create a new golden image
3. Update Terraform to use the new image
4. Replace the old instances
```

For stateful systems, such as databases, be more careful because they have persistent data.

A database server is often treated more like a pet.

But even then, the goal is to make the server as rebuildable as possible while protecting the data.

## Final Takeaway

Ansible can create EC2 instances.

But just because Ansible can provision infrastructure does not mean it should be the main tool for provisioning.

The better rule is:

```text
Terraform owns infrastructure.
Ansible owns configuration.
Packer owns images.
```

This lab proves why.

With `count: 1`, Ansible can create duplicate EC2 instances.

With `exact_count: 1` and proper filters, Ansible can enforce a desired count.

But Terraform handles this model more naturally because it has state, planning, and lifecycle management built in.
