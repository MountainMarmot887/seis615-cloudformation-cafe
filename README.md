# CloudFormation Assignment Agent (CloudShell)

A human-approved AWS CloudFormation controller for the SEIS 615 lab. It uses the AWS CLI identity already available in CloudShell and does not store AWS credentials or call an AI API.

## Safety model

- Region is restricted to `us-east-1`.
- Expected account defaults to `058015011786`; the agent stops if the active identity differs.
- `doctor`, `inspect`, `validate`, `status`, and `cleanup-plan` do not create or delete resources.
- `plan` creates a CloudFormation change set but does not execute it.
- `execute` requires the exact phrase `EXECUTE <stack> <change-set>`.
- `cleanup` is separate and requires `--confirm` plus `DELETE STACK <stack>`.
- The agent only manages CloudFormation stacks. It does not automatically delete snapshots, retained resources, or resources created outside the stack.

## 1. Upload and unzip

Upload `cloudformation-assignment-agent.zip` into AWS CloudShell, then run:

```bash
unzip -o cloudformation-assignment-agent.zip
cd cloudformation-assignment-agent
chmod +x agent/agent.py
```

## 2. Add the assignment templates

Copy your actual files into `templates/`:

- `lab-network.yaml`
- `lab-application.yaml`
- `lab-application2.yaml`

Do not deploy the two application starter templates unchanged. One allows SSH from `0.0.0.0/0`, and both define a 100-GB EBS volume with `DeletionPolicy: Snapshot`.

Keep the assignment PDF in the project folder for the review stage.

## 3. Check the environment (read-only)

```bash
python3 agent/agent.py doctor
python3 agent/agent.py inspect
```

The default expected account is the AWS account used in the earlier lab. If your intended account is different, change `expected_account_id` in `config/agent_config.json` only after verifying the account in the AWS console.

## 4. Validate templates

```bash
python3 agent/agent.py validate templates/lab-network.yaml
python3 agent/agent.py validate templates/lab-application.yaml
python3 agent/agent.py validate templates/lab-application2.yaml
```

Validation calls AWS CloudFormation `validate-template`; it does not deploy anything.

## 5. Create a change set (plan only)

Network stack first:

```bash
python3 agent/agent.py plan --stack lab-network --template templates/lab-network.yaml
```

The command prints proposed changes and saves full change-set details in `evidence/`. It does not deploy.

After reviewing the network stack and confirming the exports match the application imports, create the application change set:

```bash
python3 agent/agent.py plan --stack lab-application --template templates/lab-application.yaml --parameter NetworkStackName=lab-network
```

Pass additional parameters with repeated `--parameter ParameterKey=ParameterValue`. If a template creates IAM resources, inspect it and only then pass the required capability, for example `--capability CAPABILITY_NAMED_IAM`.

## 6. Execute only after reviewing

Use the exact change-set name printed by `plan`:

```bash
python3 agent/agent.py execute --stack lab-network --change-set CHANGE_SET_NAME
```

The agent displays the changes and requires the exact approval phrase. Repeat for the application stack only after the network stack reaches `CREATE_COMPLETE` and exports are available.

## 7. Status and evidence

```bash
python3 agent/agent.py status --stack lab-network
python3 agent/agent.py status --stack lab-application
```

Logs are stored in `logs/agent.jsonl`; change-set details are stored in `evidence/`.

## 8. Cleanup

First inspect:

```bash
python3 agent/agent.py cleanup-plan --stack lab-application
python3 agent/agent.py cleanup-plan --stack lab-network
```

Delete the application stack before the network stack because the application imports network exports:

```bash
python3 agent/agent.py cleanup --stack lab-application --confirm
python3 agent/agent.py cleanup --stack lab-network --confirm
```

Cleanup is intentionally not automatic. Before deletion, check for `DeletionPolicy: Retain`, `DeletionPolicy: Snapshot`, snapshots, S3 objects, and resources outside the stack. A successful stack deletion does not prove the account has zero ongoing charges.

## Deployment results

For the SEIS 615 elasticity and availability assignment, the deployed
CloudFormation stacks are `cafe-network` and `cafe-application`.
Both reached `CREATE_COMPLETE`.

The network stack provisions a two-AZ VPC, two public subnets, two private
subnets, two NAT gateways, and a bastion host with restricted SSH access.
The application stack provisions an internet-facing Application Load
Balancer, a launch template, an Auto Scaling Group, and a private Multi-AZ
MariaDB RDS database.

Verification showed the load balancer returned HTTP 200. The Auto Scaling
Group scaled from two to four healthy instances during a CPU workload test.
The newly provisioned database has not yet been populated with the original
Café schema and data, so the database-backed menu was not fully verified.

The controller is a deterministic Python/AWS CLI tool with human approval
for change-set execution. ChatGPT was used as a reasoning and development
assistant; the controller is not an autonomous LLM agent.
