# Terraform GitHub Actions

This is a suite of terraform related GitHub Actions that can be used together to build effective Infrastructure as Code workflows.

## Actions

See the documentation for the available actions:
- [dflook/terraform-plan](terraform-plan)
- [dflook/terraform-apply](terraform-apply)
- [dflook/terraform-output](terraform-output)
- [dflook/terraform-remote-state](terraform-remote-state)
- [dflook/terraform-validate](terraform-validate)
- [dflook/terraform-fmt-check](terraform-fmt-check)
- [dflook/terraform-fmt](terraform-fmt)
- [dflook/terraform-check](terraform-check)
- [dflook/terraform-new-workspace](terraform-new-workspace)
- [dflook/terraform-destroy-workspace](terraform-destroy-workspace)
- [dflook/terraform-destroy](terraform-destroy)
- [dflook/terraform-version](terraform-version)

## Examples

Here are some examples of how these actions can be used together.

### Terraform plan PR approval

Terraform plans typically need to be reviewed by a human before being applied.
Fortunately, GitHub has a well established method for requiring human reviews of changes - a Pull Request.

We can use PRs to safely automate infrastructure changes.
You can make GitHub enforce this using branch protection, see the [dflook/terraform-apply](terraform-apply) action for details.

In this example we use two workflows:

#### plan.yaml
This workflow runs on changes to a PR branch. It generates a terraform plan and attaches it to the PR as a comment.
```yaml
name: Create terraform plan

on:
  pull_request

jobs:
  plan:
    runs-on: ubuntu-latest
    name: Create a plan for an example terraform configuration
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: terraform plan
        uses: dflook/terraform-plan@v1
        with:
          path: my-terraform-config
```

#### apply.yaml
This workflow runs when the PR is merged into the master branch, and applies the planned changes.
```yaml
name: Apply terraform plan

on:
  push:
    branches:
      - master

jobs:
  apply:
    runs-on: ubuntu-latest
    name: Apply terraform plan
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: terraform apply
        uses: dflook/terraform-apply@v1
        with:
          path: my-terraform-config
```

### Linting
This workflow runs on every push to non-master branches and checks the terraform configuration is valid.  
For extra strictness, we check the files are in the canonical format.

This can be used to check for correctness before merging.

#### lint.yaml
```yaml
name: Lint

on:
  push:
    branches:
      - !master

jobs:
  validate:
    runs-on: ubuntu-latest
    name: Validate terraform configuration
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: terraform validate
        uses: dflook/terraform-validate@v1
        with:
          path: my-terraform-config

  fmt-check:
    runs-on: ubuntu-latest
    name: Check formatting of terraform files
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: terraform fmt
        uses: dflook/terraform-fmt-check@v1
        with:
          path: my-terraform-config
```

### Checking for drift

This workflow runs every morning and checks that the state of your infrastructure matches the configuration.

This can be used to detect manual or misapplied changes before they become a problem.  
If there are any unexpected changes, the workflow will fail.

#### drift.yaml
```yaml
name: Check for infrastructure drift

on:
  schedule:
    - cron:  "0 8 * * *"

jobs:
  check_drift:
    runs-on: ubuntu-latest
    name: Check for drift of example terraform configuration
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: Check for drift
        uses: dflook/terraform-check@v1
        with:
          path: my-terraform-config
```

### Scheduled infrastructure updates
There may be times when you expect terraform to plan updates without any changes to your terraform configuration files.
Your configuration could be consuming secrets from elsewhere, or renewing certificates every few months.

This example workflow runs every morning and applies any outstanding changes to those specific resources.

#### rotate-certs.yaml
```yaml
name: Rotate TLS certificates

on:
  schedule:
    - cron:  "0 8 * * *"

jobs:
  check_drift:
    runs-on: ubuntu-latest
    name: Check for drift of example terraform configuration
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: Rotate certs
        uses: dflook/terraform-apply@v1
        with:
          path: my-terraform-config
          auto_approve: true
          target: acme_certificate.certificate,kubernetes_secret.certificate
```

### Automatically fixing formatting
Perhaps you don't want to spend engineer time making formatting changes. This workflow will automatically create or update a PR that fixes any terraform formatting issues.

#### fmt.yaml
```yaml
name: Check terraform file formatting

on:
  push:
    branch: master 

jobs:
  format:
    runs-on: ubuntu-latest
    name: Check terraform file are formatted correctly
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: terraform fmt
        uses: dflook/terraform-fmt@v1
        with:
          path: my-terraform-config
          
      - name: Create Pull Request
        uses: peter-evans/create-pull-request@v2
        with:
          commit-message: terraform fmt
          title: Reformat terraform files
          body: Update terraform files to canonical format using `terraform fmt`
          branch: automated-terraform-fmt
```

### Ephemeral test environments
Testing of software changes often requires some supporting infrastructure, like databases, DNS records, compute environments etc.
We can use these actions to create dedicated resources for each PR which is used to run tests.

There are two workflows:

#### integration-test.yaml
This workflow runs with every change to a PR.

It deploys the testing infrastructure using a terraform workspace dedicated to this branch, then runs integration tests against the new infrastructure.

```yaml
name: Run integration tests

on:
  pull_request

jobs:
  check_format:
    runs-on: ubuntu-latest
    name: Check terraform file are formatted correctly
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: Use branch workspace
        uses: dflook/terraform-new-workspace@v1
        with:
          path: my-terraform-config
          workspace: ${{ github.head_ref }}
          
      - name: Deploy test infrastrucutre
        uses: dflook/terraform-apply@v1
        with:
          path: my-terraform-config
          workspace: ${{ github.head_ref }}
          auto_approve: true
          
      - name: Get the terraform outputs
        uses: dflook/terraform-output@v1
        id: test-infra
        with:
          path: my-terraform-config
          workspace: ${{ github.head_ref }}

      - name: Run tests
        run: |
          ./run-tests.sh --endpoint "${{ steps.test-infra.outputs.url }}"
```

#### integration-test-cleanup.yaml
This workflow runs when a PR is closed and destroys any testing infrastructure that is no longer needed.
```yaml
name: Destroy testing workspace

on:
  pull_request:
    types: [closed] 

jobs:
  check_format:
    runs-on: ubuntu-latest
    name: Check terraform file are formatted correctly
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: terraform destroy
        uses: dflook/terraform-destroy-workspace@v1
        with:
          path: my-terraform-config
          workspace: ${{ github.head_ref }}
```

## What if I don't use GitHub Actions?
This is an adaption of OVO Energy's [`ovotech/terraform`](https://github.com/ovotech/circleci-orbs/tree/master/terraform) CircleCI orb. If you use CircleCI, check it out.
