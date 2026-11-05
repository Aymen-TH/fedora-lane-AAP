# Fedora Lane Automation Platform

## Overview

The Fedora Lane Automation project uses **Red Hat Ansible Automation Platform (AAP)** to automate AWS infrastructure provisioning, RHEL server configuration, and developer application onboarding.

Automation content is maintained in Git and executed through AAP Projects, Inventories, Credentials, Job Templates, and Workflow Job Templates.

The implementation covers:

- AWS network infrastructure provisioning.
- RHEL 10 EC2 instance provisioning and configuration.
- Dynamic AWS inventory discovery.
- PostgreSQL server configuration and database provisioning.
- Automated GitLab project creation for developers.
- Onboarding request validation and approval through AAP workflows.
- Centralized credential management in AAP.

## Architecture

### Global Platform Architecture

```
┌───────────────────────┐        ┌──────────────────────────────┐
│ Developer / Admin     │───────▶│ Red Hat Ansible Automation  │
│                       │        │ Platform (AAP)               │
└───────────────────────┘        └─────────────┬─────────────── ┘
                                               │
                                               │
│                                              │
                                               ▼
                         ┌─────────────────────────────────────┐
                         │ AAP Control Plane                   │
                         │                                     │
                         │ • Projects                          │
                         │ • Inventories                       │
                         │ • Credentials                       │
                         │ • Job Templates                     │
                         │ • Workflow Job Templates            │
                         └──────────────────┬──────────────────┘
                                            │
                      ┌─────────────────────┼──────────────────────┐
                      ▼                     ▼                      ▼
          ┌────────────────────┐ ┌────────────────────┐ ┌────────────────────┐
          │ AWS Infrastructure │ │ RHEL EC2 Instances │ │ Developer Services  │
          │                    │ │                    │ │                    │
          │ VPC / Subnet       │ │ RHEL 10 Config     │ │ GitLab Projects    │
          │ Security Groups    │ │ PostgreSQL Server  │ │ PostgreSQL DBs     │
          │ EC2 Provisioning   │ │                    │ │                    │
          └────────────────────┘ └────────────────────┘ └────────────────────┘
```

### Ansible Automation Platform

AAP provides centralized execution and orchestration for the automation.

The platform is configured with:

- **Projects:** Synchronize Ansible playbooks and roles from Git.
- **Inventories:** Discover and target AWS EC2 instances dynamically.
- **Credentials:** Store AWS and GitLab authentication securely.
- **Job Templates:** Execute infrastructure, configuration, cleanup, and application automation.
- **Workflow Job Templates:** Coordinate onboarding stages and approval.
- **Execution Environments:** Provide the Ansible runtime and required collections.

The configured Execution Environment is:

```
registry.redhat.io/ansible-automation-platform-27/ee-supported-rhel9
```

## Repository Structure

```
fedora-lane-AAP/
├── ansible.cfg
├── requirements.yml
├── README.md
│
├── inventories/
│   └── aws/
│       ├── aws_ec2.yml
│       └── postgres.aws_ec2.yml
│
├── group_vars/
│   └── all.yml
│
├── playbooks/
│
├── roles/
│   ├── aws_network/
│   ├── aws_provision/
│   ├── aws_cleanup/
│   ├── configure_rhel/
│   └── configure_postgresql/
│
└── templates/
```

The repository follows a role-based structure to separate AWS infrastructure operations, RHEL configuration, cleanup, and PostgreSQL management.

## AWS Infrastructure Provisioning

AWS infrastructure is provisioned through Ansible using the `amazon.aws` collection.

The infrastructure configuration includes:

- AWS region: `eu-north-1`
- VPC creation
- Subnet creation
- Network configuration
- Security group provisioning
- EC2 instance provisioning
- Resource tagging
- Infrastructure cleanup

### Network Configuration Example

The infrastructure variables are centralized in `group_vars/all.yml`.

```
---
# AWS configuration
aws_region: eu-north-1

# Network configuration
vpc_name: fedora-lane-vpc
vpc_cidr: 10.20.0.0/16

subnet_name: fedora-lane-public-subnet
subnet_cidr: 10.20.1.0/24

# Security group
security_group_name: fedora-lane-SG

ssh_port: 2222
http_port: 35443

# Public access
allowed_ssh_cidr: 0.0.0.0/0
allowed_http_cidr: 0.0.0.0/0


# Desired VM count
aws_instance_count: 2
```

### AWS Resource Tagging

Resources are tagged to identify their ownership, project, environment, and management source.

```
aws_resource_tags:
  Environment: "{{ deployment_environment }}"
  ManagedBy: "{{ managed_by }}"
  Owner: "{{ owner }}"
  Project: "{{ project_name }}"
  Name: postgres

deployment_environment: dev
managed_by: AAP
owner: fedora-lane
project_name: FedoraLane
```

### Infrastructure and RHEL Deployment Architecture

```
┌───────────────────────────────────┐
│ AAP Job Template                  │
│ Fedora Lane - AWS EC2 Provisioning│
└─────────────────┬─────────────────┘
                  ▼
┌───────────────────────────────────┐
│ AWS Network Provisioning          │
│                                   │
│ aws_network Role                  │
│                                   │
│ • VPC                             │
│ • Subnet                          │
│ • Security Groups                 │
└─────────────────┬─────────────────┘
                  ▼
┌───────────────────────────────────┐
│ AWS Compute Provisioning          │
│                                   │
│ aws_provision Role                │
│                                   │
│ • RHEL 10 EC2 Instances           │
│ • Resource Tags                   │
└─────────────────┬─────────────────┘
                  ▼
┌───────────────────────────────────┐
│ Dynamic AWS Inventory             │
│                                   │
│ amazon.aws.aws_ec2                │
│                                   │
│ • Discover running instances      │
│ • Filter by project tags          │
└─────────────────┬─────────────────┘
                  ▼
┌───────────────────────────────────┐
│ RHEL Configuration                │
│                                   │
│ configure_rhel Role               │
│                                   │
│ • Configure managed RHEL hosts    │
└─────────────────┬─────────────────┘
                  ▼
┌───────────────────────────────────┐
│ Configured RHEL Instances         │
└───────────────────────────────────┘
```

The provisioning playbook separates AWS API operations from configuration of the resulting instances.

```
---
- name: Provision Fedora Lane RHEL 10 EC2 Instances
  hosts: localhost
  connection: local
  gather_facts: false
  vars_files:
    - ../group_vars/all.yml
  roles:
    - role: aws_provision

- name: Configure Fedora Lane RHEL Instances
  hosts: all
  become: true
  gather_facts: true
  roles:
    - role: configure_rhel
```

The first play executes locally against AWS APIs. The second play targets the provisioned EC2 instances through Ansible.

### Dynamic AWS Inventory

AWS EC2 inventory is generated using the `amazon.aws.aws_ec2` inventory plugin.

Example: `inventories/aws/aws_ec2.yml`

```
---
plugin: amazon.aws.aws_ec2

regions:
  - eu-north-1

filters:
  instance-state-name: running
  "tag:Project": FedoraLane

hostnames:
  - ip-address

compose:
  ansible_host: public_ip_address
  ansible_user: "'ec2-user'"
  ansible_port: "2222"

keyed_groups:
  - key: tags.Environment
    prefix: env

  - key: tags.Project
    prefix: project
```

This inventory discovers running instances tagged with `Project: FedoraLane`, uses their public IP addresses as connection targets, and groups hosts based on their environment and project tags.

A dedicated inventory source is also configured for PostgreSQL instances.

Example: `inventories/aws/postgres.aws_ec2.yml`

```
---
plugin: amazon.aws.aws_ec2

regions:
  - eu-north-1

filters:
  instance-state-name: running
  "tag:Project": FedoraLane
  "tag:Name": postgres

hostnames:
  - ip-address

compose:
  ansible_host: public_ip_address
  ansible_user: "'ec2-user'"
  ansible_port: "22"
```

AAP inventory sources use the Fedora Lane AAP Project and the AWS credential. Inventory updates are enabled, with overwrite and update-on-launch configured.

### RHEL Configuration

The `configure_rhel` role is used to configure provisioned RHEL instances.

The configuration is executed through AAP against the discovered EC2 hosts, using privilege escalation where required.

The provisioning and configuration workflow is designed to separate infrastructure lifecycle operations from operating system configuration.

### AWS Cleanup

The `aws_cleanup` role provides automated cleanup of Fedora Lane AWS resources.

A dedicated AAP Job Template is configured:

```
Fedora Lane - AWS Cleanup
```

The cleanup automation handles the removal of project resources, including EC2 instances and associated networking resources.

## Credentials and Secrets Management

Credentials are managed centrally in **AAP Credentials** rather than hardcoded in playbooks, inventories, or Git.

### AWS Credential

AAP credential:

```
fedora-lane-aws
```

Credential type: Amazon Web Services

The credential provides AWS API authentication for infrastructure provisioning, network configuration, cleanup, and dynamic inventory discovery.

Typical credential fields:

```
Access Key ID:     <AWS_ACCESS_KEY_ID>
Secret Access Key: <AWS_SECRET_ACCESS_KEY>
```

The actual values are stored in AAP and are not committed to the repository.

### GitLab Credential

A GitLab credential is used by the onboarding automation to authenticate with the GitLab API.

The credential contains the GitLab access token required to create projects and perform the configured API operations.

Example:

```
GitLab Access Token: <GITLAB_ACCESS_TOKEN>
```

The token is managed in AAP and passed securely to the automation.

### SSH Authentication

EC2 instances are accessed through SSH using the configured Ansible connection user and SSH key.

Example connection settings:

```
ansible_user: ec2-user
ansible_port: 22
```

For the general Fedora Lane inventory, the configured SSH port is `2222`.

SSH private key material is stored in an AAP Machine Credential and is not included in the Git repository.

### Credential Handling Principles

- No AWS secret access keys are committed to Git.
- No GitLab access tokens are stored in playbook variables.
- SSH private keys are not stored in the repository.
- Credentials are attached to the relevant AAP Job Templates and Inventory Sources.
- Credential values are referenced through AAP at execution time.

## PostgreSQL Configuration and Database Provisioning

PostgreSQL is managed through Ansible and AAP.

The `configure_postgresql` role handles PostgreSQL configuration and database provisioning operations.

The automation includes:

- PostgreSQL database creation.
- PostgreSQL user creation.
- Database access configuration.
- Verification of the created database and user.

A dedicated PostgreSQL dynamic inventory discovers the database instance using its AWS tags.

Example PostgreSQL inventory filter:

```
filters:
  instance-state-name: running
  "tag:Project": FedoraLane
  "tag:Name": postgres
```

The database provisioning automation is integrated with the developer onboarding process.

## Developer Application Onboarding

A developer onboarding workflow is configured in AAP to automate the creation of a GitLab project and the associated PostgreSQL database resources.

The workflow uses survey inputs to collect the onboarding request details.

### Onboarding Inputs

Examples of onboarding variables include:

```
requester_username: cameron.shaw

onboarding_project_name: payment-api
onboarding_project_path: payment-api

onboarding_db_name: dev_database
onboarding_branch_name: main
```

These values represent example request data. Actual values are provided through the AAP Workflow Job Template survey.

The workflow validates the request before proceeding.

### Application Onboarding Workflow

```
┌──────────────────────────────────────┐
│ Developer submits onboarding request │
│                                      │
│ AAP Survey                           │
│ Requester / Project / Database       │
└──────────────────┬───────────────────┘
                   ▼
┌──────────────────────────────────────┐
│ Validate Onboarding Request          │
│                                      │
│ Required inputs and assertions       │
└──────────────────┬───────────────────┘
                   ▼
┌──────────────────────────────────────┐
│ AAP Approval Stage                   │
│                                      │
│ Request review and approval          │
└──────────────────┬───────────────────┘
                   ▼
┌──────────────────────────────────────┐
│ Create GitLab Project                │
│                                      │
│ GitLab REST API                      │
│ Private project / README              │
└──────────────────┬───────────────────┘
                   ▼
┌──────────────────────────────────────┐
│ Configure PostgreSQL Database        │
│                                      │
│ Create database and database user    │
│ Verify provisioned resources         │
└──────────────────┬───────────────────┘
                   ▼
┌──────────────────────────────────────┐
│ Application Resources Created        │
│                                      │
│ GitLab Project + PostgreSQL Database │
└──────────────────────────────────────┘
```

### Request Validation

The validation playbook checks the onboarding request variables before resource creation.

The validation stage uses Ansible assertions to ensure the required values are present and valid for the automation.

Example request:

```
requester_username: cameron.shaw
onboarding_project_name: payment-api
onboarding_project_path: payment-api
onboarding_db_name: dev_database
```

### AAP Approval

An approval stage is configured in the Workflow Job Template.

The workflow pauses for approval before proceeding to the GitLab project creation stage.

This allows the onboarding request to be reviewed before automated resources are created.

### GitLab Project Creation

The GitLab project creation automation uses the GitLab REST API.

The playbook:

- Creates the requested GitLab project.
- Uses the configured project name and path.
- Creates the project as private.
- Checks for an existing project to support idempotent execution.
- Initializes the project with a README.

Example onboarding request:

```
Requester: Cameron Shaw
Project: payment-api
Project Path: payment-api
Visibility: Private
Default Branch: main
```

### PostgreSQL Database Provisioning

After GitLab project creation, the PostgreSQL automation provisions the database resources associated with the onboarding request.

Example:

```
Application Project: payment-api
Database Name:       dev_database
Database User:       Derived from onboarding request
Database Target:     PostgreSQL instance
```

The PostgreSQL role creates the requested database and user and verifies the resulting resources.

Database credentials are handled through automation variables and credentials rather than hardcoded into the repository.

## AAP Job Templates and Workflows

The following AAP automation entry points have been configured:

| AAP Job Template / Workflow            | Purpose                                            |
| -------------------------------------- | -------------------------------------------------- |
| Fedora Lane - AWS Network Provisioning | Provision AWS networking resources                 |
| Fedora Lane - AWS EC2 Provisioning     | Provision and configure RHEL EC2 instances         |
| Fedora Lane - AWS Cleanup              | Clean up Fedora Lane AWS resources                 |
| PostgreSQL Configuration               | Configure PostgreSQL and manage database resources |
| Fedora Lane - Developer Onboarding     | Orchestrate the developer onboarding workflow      |

The Developer Onboarding Workflow Job Template coordinates request validation, approval, GitLab project creation, and PostgreSQL database provisioning.

## Idempotency and Automation Design

The automation is structured around reusable Ansible roles and declarative tasks.

The implementation uses:

- Role-based task organization.
- Centralized variables.
- AAP-managed credentials.
- Dynamic inventory discovery.
- Git-based source control.
- AAP Job Templates for repeatable execution.
- Workflow Job Templates for multi-stage orchestration.
- Existing-resource checks in GitLab project creation.
- Validation before resource provisioning.

This structure separates infrastructure provisioning, server configuration, database management, and application onboarding into independent automation components.

## Summary

The Fedora Lane Automation Platform brings AWS infrastructure provisioning, RHEL configuration, PostgreSQL management, and GitLab application onboarding under Ansible Automation Platform.

The implemented automation provides a centralized and repeatable workflow for:

- Provisioning AWS network resources.
- Creating and configuring RHEL EC2 instances.
- Discovering AWS resources through dynamic inventory.
- Managing PostgreSQL databases and users.
- Creating private GitLab projects through developer onboarding.
- Validating onboarding requests and coordinating approval.
- Managing cloud and GitLab authentication through AAP Credentials.
- Executing automation from Git-managed Ansible content.

Clarify the SSH port configurationSeparate implemented features from planned ones