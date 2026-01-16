# Import Tofu output Github Actions composite

## Description

This action composite allows to import OpenTofu variables from envs to .tfvars.json file.

The original idea for this was found on <https://github.com/orgs/community/discussions/25225#discussioncomment-6776295> discussion and belongs to [rdhar](https://github.com/rdhar).

## Inputs

```yml
inputs:
  workdir:
    required: true
    description: Tofu working directory
```

## Example usage

NOTE: this workflow works the best with `marcinbator/export-tofu-output@v1.0.0` (<https://github.com/marcinbator/export-tofu-output>) which encodes OpenTofu outputs to Base64 and exports them to single GITHUB_OUTPUT variable (as shown in [Full example](#full-example) below).

IMPORTANT: Base64 encoded outputs from previous job must be passed to this composite using `env`. You can pass multiple outputs (e.g. from multiple jobs), they will be all put together into one .tfvars.json file. Every env variable with outputs need to have `_outputs` suffix, as shown in the example.

Make sure every Tofu output has unique name to avoid errors during Tofu execution.

```yml
- name: Extract TF variables
  uses: marcinbator/import-tofu-output@v1.0.0 # here
  with:
    workdir: ./db
  env:
    rds_outputs: ${{ needs.rds.outputs.tf_outputs }} #outputs needs to be passed in env variable with `_outputs` suffix.
```

## Full example

```yml
jobs:
  rds: # this job exports outputs to GITHUB_OUTPUT using marcinbator/export-tofu-output@v1.0.0. Outputs are always exported to `tf_outputs` output.
    runs-on: ubuntu-latest
    outputs:
      tf_outputs: ${{ steps.output.outputs.tf_outputs }}
    defaults:
      run: working-directory:./rds

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup OpenTofu
        uses: opentofu/setup-opentofu@v1
        with:
          tofu_version: 1.11.2

      - name: TF init
        run: tofu init

      - name: TF Apply
        run: tofu apply -auto-approve -input=false

      - id: output
        name: Output TF vars
        uses: marcinbator/tofu-output@v1.0.0 # here
        with:
          workdir: ./rds

  db: # this job uses outputs exported in previous job
    runs-on: ubuntu-latest
    needs: rds
    defaults:
      run:
        working-directory: ./db

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup OpenTofu
        uses: opentofu/setup-opentofu@v1
        with:
          tofu_version: 1.11.2

      - name: Extract TF variables
        uses: marcinbator/import-tofu-output@v1.0.0 # here
        with:
          workdir: ./db
        env:
          rds_outputs: ${{ needs.rds.outputs.tf_outputs }} #outputs needs to be passed in env variable with `_outputs` suffix.

      - name: TF init
        run: tofu init

      - name: TF Apply
        run: tofu apply -auto-approve -input=false
```
