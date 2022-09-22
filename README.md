# Use Amazon AWS SSM JSON Parameter as environment variables

This action will fetch a secret JSON parameter from the AWS SSM Parameter Store
and set all keys as environment variables. The parameter is expected to be a
JSON object with string values.

**The keys of the JSON object will be used as the environment variable names. This
could potentially overwrite existing environment variables.**

- [Usage](#usage)
  - [Fetching Parameters](#fetching-parameters) 
  - [Decrypt and exporting](#decrypt-and-exporting)
  - [Inputs](#inputs)
  - [Outputs](#outputs)
- [Examples](#examples)
  - [Storing the SSM parameters as JSON](#storing-the-ssm-parameters-as-json)
  - [Example Workflow](#example-workflow)

## Usage

Add the entry below to your workflow file. The `pgp_passphrase` should be a
GitHub secret and is only used by the action to encrypt the environment variables
as they are passed to other steps in the workflow.

### Fetching parameters

The example below will fetch the parameters from SSM and create an encrypted
string as an output for use in other jobs or steps.

```yaml
uses: accendero/aws-ssm-json-to-env@v1
id: <step id 1>
with:
  access_key_id: ${{ secrets.AWS_ACCESS_KEY_ID }}
  secret_access_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
  region: ${{ secrets.AWS_DEFAULT_REGION }}
  parameter_name: "/my-project/config/cicd"
  pgp_passphrase: ${{ secrets.PGP_PASSPHRASE }}
```

### Decrypt and exporting

The example below will decrypt a previously fetched SSM environment and export
its variables to the current job's environment.

```yaml
uses: accendero/aws-ssm-json-to-env@v1
id: <step id 2>
with:
  pgp_passphrase: ${{ secrets.PGP_PASSPHRASE }}
  decrypt: ${{ steps.<step id 1>.outputs.encrypted_environment }}
```

### Inputs

* `access_key_id` - AWS access key ID
* `secret_access_key` - AWS secret access key
* `region` - AWS region
* `parameter_name` - Name of the parameter to fetch
* `pgp_passphrase` - Passphrase to encrypt the environment variables
* **TODO**: `parameter_version` - Version of the parameter to fetch, latest is used by default
* `decrypt` - The value of an encrypted environment variables string. Passing this
  in will decrypt the variables and set then in the environment for the current
  job.
* `masks` - A comma separated list of environment variable names to mask in the
  logs. This is useful for sensitive information like passwords or API keys.

### Outputs

The only output for this action is an encrypted string of the environment variables.
* `encrypted_environment` - Encrypted string of the environment variables

## Examples

### Storing the SSM parameters as JSON

The SSM parameter should be a JSON object with string values. For example:

```json
{
    "MY_ENV_VAR": "my value",
    "MY_OTHER_ENV_VAR": "my other value"
}
```

### Example workflow

The workflow below will fetch the SSM parameter `/my-project/config/cicd` and
store the encrypted variables string as an output. The output will then be
decrypted and set in the environment for the `print_a_variable` job.

```yaml
name: Deploy to environment

on: push

jobs:
  setup:
    runs-on: ubuntu-latest
    outputs:
      encrypted_environment: ${{ steps.ssm.outputs.encrypted_environment }}
    steps:
      - uses: actions/checkout@v1
      - uses: accendero/aws-ssm-json-to-env@v1
        id: ssm
        with:
          access_key_id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          secret_access_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          region: ${{ secrets.AWS_DEFAULT_REGION }}
          parameter_name: "/my-project/config/cicd"
          pgp_passphrase: ${{ secrets.PGP_PASSPHRASE }}
          masks: "MY_OTHER_ENV_VAR"

  print_a_variable:
    runs-on: ubuntu-latest
    needs: setup
    steps:
      - uses: actions/checkout@v1
      - uses: accendero/aws-ssm-json-to-env@v1
        with:
           decrypt: ${{ needs.setup.outputs.encrypted_environment }}
           pgp_passphrase: ${{ secrets.PGP_PASSPHRASE }}
      - run: echo "${{ env.MY_ENV_VAR }}, ${{ env.MY_OTHER_ENV_VAR }}"
```      
