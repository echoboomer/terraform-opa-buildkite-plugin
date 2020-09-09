# Buildkite Terraform Open Policy Agent Plugin

![GitHub release (latest by date)](https://img.shields.io/github/v/release/echoboomer/terraform-opa-buildkite-plugin) ![GitHub Release Date](https://img.shields.io/github/release-date/echoboomer/terraform-opa-buildkite-plugin) ![GitHub contributors](https://img.shields.io/github/contributors/echoboomer/terraform-opa-buildkite-plugin) ![GitHub issues](https://img.shields.io/github/issues/echoboomer/terraform-opa-buildkite-plugin)

Runs Open Policy Agent against Terraform plans.

## Prerequisites

#### Terraform Plan

The plugin requires a Terraform plan to evaluate. This can be accomplished by exporting a Terraform plan from another step.

Consider using the [Buildkite Terraform Plugin](https://github.com/echoboomer/terraform-buildkite-plugin) along with the [Buildkite Artifacts Plugin](https://github.com/buildkite-plugins/artifacts-buildkite-plugin) to accomplish this. The former exports a `json` version of the Terraform plan automatically.

The Terraform plan must be `json` formatted.

#### Open Policy Agent Constructs

Creating OPA policies is outside the scope of this plugin, but is detailed in [this](https://www.openpolicyagent.org/docs/latest/terraform/) article.

In general, you need three policy documents related to this process for the plugin to work:

- A Terraform `rego` file that defines the actual policy.
- A `resource_types.json` file that defines which resources to count in the evaluation.
- A `resource_weights.json` file that defines the weights assigned to Terraform resources to calculate a score.

The location of these files is customizable, but here are the defaults:

```bash
workdir (Buildkite Agent root working directory, correlates to root of your git repo)
  |__ terraform/
      |__ tests/
          |__ resource_types.json
          |__ resource_weights.json
          |__ terraform.rego
```

There are examples below on how you could configure these files.

#### Workflow

In general, this is the workflow the plugin follows:

- Ingest Terraform plan in `json` format.
- Use Terraform `rego` file to define policy.
- Levereage resource `types` and `weights` file to calculate a score versus a defined "blast radius".
- Output whether or not the proposed Terraform changes are in compliance.

## Example

Add the following to your `pipeline.yml`:

```yml
steps:
  - label: "Terraform Policy Evaluation"
    plugins:
      - echoboomer/terraform-opa#v1.0.0:
          terraform_plan: tfplan.json
```

You can customize other options based on your preferred configuration:

```yml
steps:
  - label: "Terraform Policy Evaluation"
    plugins:
      - echoboomer/terraform-opa#v1.0.0:
          fail_step: true
          policy_file: tf-this.rego
          resource_types_file: custom-types.json
          resource_weights_file: custom-weights.json
          terraform_plan: tfplan.json
          tests_dir: unit-tests
```

You can combine this plugin with the [Buildkite Terraform Plugin](https://github.com/echoboomer/terraform-buildkite-plugin) to run an OPA test inline in the same step as your Terraform plan:

```yml
steps:
  - label: ":terraform: Running Terraform"
    concurrency: 1
    concurrency_group: tf-repo
    plugins:
      - echoboomer/terraform#v1.2.18:
          apply_master: true
          init_args:
            - "-input=false"
            - "-backend-config=bucket=my_gcp_bucket"
            - "-backend-config=prefix=my-prefix"
            - "-backend-config=credentials=sa.json"
          image: mycustomtfimage
          skip_apply_no_diff: true
          version: mytag
      - echoboomer/terraform-opa#v1.0.0:
          fail_step: true
          terraform_plan: tfplan.json
          tests_dir: "tests"
```

With the provided configuration in this example, Terraform will run a plan and then immediately use the `tfplan.json` file that is available in the `terraform/` directory to run an OPA evaluation.

## Configuration

### `debug` (Not Required, boolean)

Provides some helpful information when attempting to troubleshoot issues with running the plugin. Defaults to `false`.

### `fail_step` (Not Required, boolean)

If this is provided and set to `true`, the Buildkite pipeline will fail if the provided Terraform plan does not meet the specified policy requirements. Without this option, the plugin only returns output regarding the assessment.

### `image` (Not Required, string)

The Docker image to run for Open Policy Agent. Defaults to `openpolicyagent/opa`. The `version` option specified below correlates with the `tag` option.

### `tests_dir` (Not Required, string)

The path of the directory in your Terraform repository containing the required files for running Open Policy Agent assessments against Terraform code. Since Buildkite agents typically operate from the root of a repository, this is in relation to that top level path. This defaults to `./terraform/tests`. You may override this as long as your files are available in the given location.

### `policy_file` (Not Required, string)

The path to the Terraform `.rego` file that Open Policy Agent uses to evaluate the Terraform plan. This must be available in your defined `tests_dir`. Defaults to `${tests_dir}/terraform.rego`.

### `resource_types_file` (Not Required, string)

The path to the `json` file that contains specifications for which Terraform resources types to evaluate. This must be available in your defined `tests_dir`. Defaults to `${tests_dir}/resource_types.json`.

### `resource_weights_file` (Not Required, string)

The path to the `json` file that contains specifications for weights assigned to Terraform resources. This must be available in your defined `tests_dir`. Defaults to `${tests_dir}/resource_weights.json`.

### `terraform_plan` (Required, string)

The path to the Terraform plan's `json` output. You need to provide this for the plugin to work. This is easily accomplished by using the `artifact` plugin and exporting it from your Terraform plan step and importing for the OPA step.

### `version` (Not Required, string)

The version tag for the Open Policy Agent Docker image. Defaults to `latest`.

## FAQ

- The official documentation for this process uses a single `rego` file and defines `types` and `weights` within it. Why do I have to specify two separate files for these values?
  - Defining the `types` and `weights` in their own files allows you to specify them independently. This way, you can specify your policy separately. This is handy when you're trying to define a large number of resources and weights.

## Sample Open Policy Agent Policies

#### Policy

```rego
package terraform.analysis

import data.resource_types as resource_import_types
import data.resource_weights as resource_import_weights
import input as tfplan

########################
# Parameters for Policy
########################

# Acceptable score for proposed changes.
blast_radius = 75

# Weights assigned for each operation on each resource-type.
weights := resource_import_weights["weights"]

# Consider exactly these resource types in calculations.
resource_types := resource_import_types["types"]

#########
# Policy
#########

# Authorization holds if score for the plan is acceptable
default authz = false
authz {
    score < blast_radius
}

# Compute the score for a Terraform plan as the weighted sum of deletions, creations, modifications
score = s {
    all := [ x |
            some resource_type
            crud := weights[resource_type];
            del := crud["delete"] * num_deletes[resource_type];
            new := crud["create"] * num_creates[resource_type];
            mod := crud["modify"] * num_modifies[resource_type];
            x := del + new + mod
    ]
    s := sum(all)
}

####################
# Terraform Library
####################

# list of all resources of a given type
resources[resource_type] = all {
    some resource_type
    resource_types[resource_type]
    all := [name |
        name:= tfplan.resource_changes[_]
        name.type == resource_type
    ]
}

# number of creations of resources of a given type
num_creates[resource_type] = num {
    some resource_type
    resource_types[resource_type]
    all := resources[resource_type]
    creates := [res |  res:= all[_]; res.change.actions[_] == "create"]
    num := count(creates)
}

# number of deletions of resources of a given type
num_deletes[resource_type] = num {
    some resource_type
    resource_types[resource_type]
    all := resources[resource_type]
    deletions := [res |  res:= all[_]; res.change.actions[_] == "delete"]
    num := count(deletions)
}

# number of modifications to resources of a given type
num_modifies[resource_type] = num {
    some resource_type
    resource_types[resource_type]
    all := resources[resource_type]
    modifies := [res |  res:= all[_]; res.change.actions[_] == "update"]
    num := count(modifies)
}
```

#### Resource Types

```json
{
	"types": {
		"google_compute_network": {},
		"google_compute_router": {},
		"google_compute_router_nat": {},
		"google_compute_subnetwork": {},
		"google_container_cluster": {},
		"google_container_node_pool": {},
		"google_kms_crypto_key": {},
		"google_kms_key_ring": {},
		"google_storage_bucket": {}
	}
}
```

#### Resource Weights

```json
{
	"weights": {
		"google_compute_network": {
			"delete": 100,
			"create": 30,
			"modify": 5
		},
		"google_compute_router": {
			"delete": 100,
			"create": 10,
			"modify": 5
		},
		"google_compute_router_nat": {
			"delete": 100,
			"create": 10,
			"modify": 5
		},
		"google_compute_subnetwork": {
			"delete": 100,
			"create": 10,
			"modify": 5
		},
		"google_container_cluster": {
			"delete": 100,
			"create": 20,
			"modify": 5
		},
		"google_container_node_pool": {
			"delete": 100,
			"create": 10,
			"modify": 5
		},
		"google_kms_crypto_key": {
			"delete": 100,
			"create": 10,
			"modify": 5
		},
		"google_kms_key_ring": {
			"delete": 100,
			"create": 10,
			"modify": 5
		},
		"google_storage_bucket": {
			"delete": 100,
			"create": 10,
			"modify": 5
		}
	}
}
```

## Developing

To run the linting tool:

```shell
docker-compose run --rm lint
```

## Contributing

1. Fork the repo.
2. Make your changes.
3. Make sure linting passes.
4. Commit and push your changes to your branch.
5. Open a pull request.
