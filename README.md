# Use Amazon AWS SSM JSON Parameter as environment variables

This action will fetch a secret JSON parameter from the AWS SSM Parameter Store
and set all keys as environment variables. The parameter is expected to be a
JSON object with string values.

**The keys of the JSON object will be used as the environment variable names. This
could potentially overwrite existing environment variables.**

- [Usage](#usage)
  - [Fetching Parameters](#fetching-parameters)
  - [Fetching Parameters with OIDC](#fetching-parameters-with-oidc)
  - [Decrypt and exporting](#decrypt-and-exporting)
  - [Accessing parameters directly](#accessing-parameters-directly)
  - [Inputs](#inputs)
  - [Outputs](#outputs)
- [Examples](#examples)
  - [Storing the SSM parameters as JSON](#storing-the-ssm-parameters-as-json)
  - [Example Workflow](#example-workflow)
  - [Example Workflow with OIDC](#example-workflow-with-oidc)
- [Development](#development)
  - [Running Tests Locally](#running-tests-locally)

## Usage

Add the entry below to your workflow file. The `pgp_passphrase` should be a
GitHub secret and is only used by the action to encrypt the environment variables
as they are passed to other steps in the workflow.

### Fetching parameters

The example below will fetch the parameters from SSM and create an encrypted
string as an output for use in other jobs or steps.

```yaml
uses: accendero/aws-ssm-json-to-env@v2.1
id: <step id 1>
with:
  access_key_id: ${{ secrets.AWS_ACCESS_KEY_ID }}
  secret_access_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
  region: ${{ secrets.AWS_DEFAULT_REGION }}
  parameter_name: "/my-project/config/cicd"
  pgp_passphrase: ${{ secrets.PGP_PASSPHRASE }}
```

### Fetching parameters with OIDC

If you're using OIDC authentication (via `aws-actions/configure-aws-credentials`),
you can omit the `access_key_id` and `secret_access_key` inputs. The action will
use the existing AWS session credentials.

```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/my-github-actions-role
    aws-region: us-east-1

- uses: accendero/aws-ssm-json-to-env@v2.1
  id: <step id 1>
  with:
    region: us-east-1
    parameter_name: "/my-project/config/cicd"
    pgp_passphrase: ${{ secrets.PGP_PASSPHRASE }}
```

### Decrypt and exporting

The example below will decrypt a previously fetched SSM environment and export
its variables to the current job's environment.

```yaml
uses: accendero/aws-ssm-json-to-env@v2.1
id: <step id 2>
with:
  pgp_passphrase: ${{ secrets.PGP_PASSPHRASE }}
  decrypt: ${{ steps.<step id 1>.outputs.encrypted_environment }}
```

### Accessing parameters directly

The `matrix` output exposes all fetched parameters as a JSON object, keyed by
parameter name. This lets you read individual values in the same job without
going through the encrypt/decrypt cycle, or forward them to other jobs.

```yaml
# Within the same job
- uses: accendero/aws-ssm-json-to-env@v2.1
  id: ssm
  with:
    region: ${{ secrets.AWS_DEFAULT_REGION }}
    parameter_name: "/my-project/config/cicd"
    pgp_passphrase: ${{ secrets.PGP_PASSPHRASE }}

- run: echo "${{ fromJSON(steps.ssm.outputs.matrix).MY_ENV_VAR }}"
```

To pass parameters to a downstream job, expose `matrix` as a job output:

```yaml
jobs:
  setup:
    runs-on: ubuntu-latest
    outputs:
      params: ${{ steps.ssm.outputs.matrix }}
    steps:
      - uses: accendero/aws-ssm-json-to-env@v2.1
        id: ssm
        with:
          region: ${{ secrets.AWS_DEFAULT_REGION }}
          parameter_name: "/my-project/config/cicd"
          pgp_passphrase: ${{ secrets.PGP_PASSPHRASE }}

  deploy:
    needs: setup
    runs-on: ubuntu-latest
    steps:
      - run: echo "${{ fromJSON(needs.setup.outputs.params).MY_ENV_VAR }}"
```

### Inputs

* `access_key_id` - AWS access key ID (optional if using OIDC or existing session)
* `secret_access_key` - AWS secret access key (optional if using OIDC or existing session)
* `region` - AWS region (required when fetching parameters)
* `parameter_name` - Name of the parameter to fetch (required when fetching parameters)
* `scope` - Which store to pull from: `SSM` (default) or `Secrets`
* `pgp_passphrase` - Passphrase to encrypt the environment variables
* `decrypt` - The value of an encrypted environment variables string. Passing this
  in will decrypt the variables and set them in the environment for the current
  job.

### Outputs

* `encrypted_environment` - Encrypted string of the environment variables, for use with `decrypt`
* `matrix` - JSON object of all fetched parameters, keyed by parameter name. Access individual values with `fromJSON(steps.<id>.outputs.matrix).KEY_NAME`

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
      encrypted_secrets: ${{ steps.secrets.outputs.encrypted_environment }}
    steps:
      - uses: actions/checkout@v6
      - uses: accendero/aws-ssm-json-to-env@v2.1
        id: ssm
        with:
          access_key_id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          secret_access_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          region: ${{ secrets.AWS_DEFAULT_REGION }}
          parameter_name: "/my-project/config/cicd"
          pgp_passphrase: ${{ secrets.PGP_PASSPHRASE }}
      - uses: accendero/aws-ssm-json-to-env@v2.1
        id: secrets
        with:
          access_key_id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          secret_access_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          region: ${{ secrets.AWS_DEFAULT_REGION }}
          parameter_name: "/my-project/config/cicd"
          pgp_passphrase: ${{ secrets.PGP_PASSPHRASE }}
          scope: Secrets

  print_a_variable:
    runs-on: ubuntu-latest
    needs: setup
    steps:
      - uses: actions/checkout@v6
      - uses: accendero/aws-ssm-json-to-env@v2.1
        with:
           decrypt: ${{ needs.setup.outputs.encrypted_environment }}
           pgp_passphrase: ${{ secrets.PGP_PASSPHRASE }}
      - run: echo "${{ env.MY_ENV_VAR }}, ${{ env.MY_OTHER_ENV_VAR }}"
      - uses: accendero/aws-ssm-json-to-env@v2.1
        with:
          decrypt: ${{ needs.setup.outputs.encrypted_secrets }}
          pgp_passphrase: ${{ secrets.PGP_PASSPHRASE }}
      - run: echo "${{ env.MY_SECRET_VAR }}, ${{ env.MY_OTHER_SECRET_VAR }}"

```

### Example workflow with OIDC

This workflow uses OIDC authentication instead of static access keys. You'll need
to configure an IAM role with a trust policy for GitHub Actions OIDC.

```yaml
name: Deploy to environment

on: push

permissions:
  id-token: write
  contents: read

jobs:
  setup:
    runs-on: ubuntu-latest
    outputs:
      encrypted_environment: ${{ steps.ssm.outputs.encrypted_environment }}
    steps:
      - uses: actions/checkout@v6
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/my-github-actions-role
          aws-region: us-east-1
      - uses: accendero/aws-ssm-json-to-env@v2.1
        id: ssm
        with:
          region: us-east-1
          parameter_name: "/my-project/config/cicd"
          pgp_passphrase: ${{ secrets.PGP_PASSPHRASE }}

  print_a_variable:
    runs-on: ubuntu-latest
    needs: setup
    steps:
      - uses: actions/checkout@v6
      - uses: accendero/aws-ssm-json-to-env@v2.1
        with:
           decrypt: ${{ needs.setup.outputs.encrypted_environment }}
           pgp_passphrase: ${{ secrets.PGP_PASSPHRASE }}
      - run: echo "${{ env.MY_ENV_VAR }}, ${{ env.MY_OTHER_ENV_VAR }}"
```

## Development

### Running tests locally

This project uses [act](https://github.com/nektos/act) to run GitHub Actions locally.

#### Install act

If you follow the linux instructions below, it may install to the local directory you are running the installation
command from, and the resulting binary will be in `./bin/act`. You can move this to a directory in your PATH or add the 
local directory to your PATH to run `act` from anywhere.

Additionally, act requires docker to run the GitHub Actions locally, so make sure you have Docker installed and 
running on your machine.

```bash
# macOS
brew install act

# Linux
curl -s https://raw.githubusercontent.com/nektos/act/master/install.sh | sudo bash

# Other methods: https://github.com/nektos/act#installation
```

#### Configure act

Run the setup script to generate `.actrc` with the correct Docker socket path for your system:

```bash
./setup-act.sh
```

This will auto-detect your Docker socket location (works with Docker Desktop and standard installations).

#### Run tests

The test suite includes validation tests and decrypt tests that don't require AWS
credentials, plus an integration test that requires real AWS access.

```bash
# Run validation tests (no AWS credentials needed)
act -j test-validation

# Run decrypt tests (no AWS credentials needed)
act -j test-decrypt

# Run matrix output tests (no AWS credentials needed)
act -j test-matrix-output

# Run all non-integration tests
act pull_request
```

#### Run integration tests with AWS

To run the SSM integration tests, you need AWS credentials and a test parameter.

1. Copy the secrets example file:
   ```bash
   cp .secrets.example .secrets
   ```

2. Edit `.secrets` with your AWS credentials and test parameter:
   ```
   AWS_ACCESS_KEY_ID=your-access-key-id
   AWS_SECRET_ACCESS_KEY=your-secret-access-key
   AWS_REGION=us-east-1
   SSM_PARAMETER_NAME=/test/parameters
   ```

3. Run the integration test:
   ```bash
   act -j test-ssm-fetch --secret-file .secrets
   ```

Alternatively, pass secrets directly:
```bash
act -j test-ssm-fetch \
  -s AWS_ACCESS_KEY_ID=xxx \
  -s AWS_SECRET_ACCESS_KEY=xxx \
  -s AWS_REGION=us-east-1 \
  -s SSM_PARAMETER_NAME=/your/test/param
```      
