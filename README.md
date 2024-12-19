# AWS Role Assumption and VPN Connection GitHub Action

This GitHub Action allows you to assume an AWS role, 
extract keys from AWS Secrets Manager, 
inject them as environment variables, and connect to VPN services like Tailscale and Pritunl.

<b><i>This action is managed by a private repository.</i></b>

## Inputs

| Input                         | Description                                                                          | Default | Required |
|-------------------------------|--------------------------------------------------------------------------------------|---------|----------|
| `python-version`              | The version of Python to be used.                                                    | `3.10`  | false    |
| `node-version`                | The version of Node to be used.                                                      | `20`    | false    |
| `install-docker`              | Whether to install Docker and Docker Compose.                                        | `false` | false    |
| `connect-to-vpn`              | The name of the VPN to connect to. Supported options are "pritunl", and "tailscale". | ""      | false    |
| `aws-role`                    | The AWS role to assume.                                                              | ""      | false    |
| `aws-region`                  | The AWS region for the assumed role.                                                 | ""      | false    |
| `aws-secrets`                 | Comma-separated list of AWS secrets to extract and inject as environment variables.  | ""      | false    |

## Assuming roles
In order to assume a role, you must have configured your GitHub Action to have AWS privileges.
You can do this relatively easily using Terraform:
```terraform
module "github-oidc" {
  source  = "terraform-module/github-oidc-provider/aws"
  version = "~> 1"

  create_oidc_provider = true
  create_oidc_role     = true

  repositories = ["${local.username}/*"]
  oidc_role_attach_policies = [
    aws_iam_policy.cicd.arn
  ]
}
```

## Extracting secrets
Potentially the most useful part of this action is extracting secrets 
and injecting them into the environment.
To extract a secret, a secret name must be passed to the action. This name 
can be either the human-readable name, or the ARN. In the case that you have chosen
to use the human-readable name, inserting a(n) * after the name of the secret
will search through all secrets to find the one with the corresponding prefix. 
This is useful in cases where you have created the secret with a randomly generated
suffix, and do not want to keep track of the suffix in your action.

The secret value itself in AWS Secretsmanager is called the SecretString, and 
it is frequently either a JSON, or a single value. For clarity's sake, I will 
refer to these two cases as a JSON key and a flat key respectively.

There are three relevant cases:

| Case                  | Example Usage          | Example SecretString Value | Resulting Env Vars                             |
|-----------------------|------------------------|----------------------------|------------------------------------------------|
| JSON key              | json_key_name*         | `{"key":"value"}`          | KEY=value                                      |
| JSON key with prefix  | prefix-=json_key_name* | `{"key":"value"}`          | PREFIX_KEY=value                               |
| Flat key with name    | name=flat_key_name     | `flat-key-value-as-secret` | NAME=flat-key-value-as-secret                  |
| Flat key without name | flat_key_name          | `flatkeyvalueassecret`     | ERROR: You need a name to map a flat value to. |

Remember that the "key_name" can be the name of the secret, a prefix of a secret, or an ARN.
inputs.aws-secrets can be passed as a comma-separated list, for multiple secrets to be injected.
See example usage below for a good example.

## Connecting to a VPN
Supported VPNs for connection are Tailscale and Pritunl. Connecting to these VPNs requires 
specific environment variables. These can either be configured as environment 
variables through GitHub, or can be injected via AWS secrets. 
The following environment variables are required:

### Pritunl 
* PRITUNL_PROFILE_FILE
* PRITUNL_PROFILE_PIN
* PRITUNL_PROFILE_SERVER

### Tailscale
* TAILSCALE_OAUTH_CLIENT_ID
* TAILSCALE_OAUTH_SECRET

## Usage

```yaml
name: Example Workflow

on: [push]

jobs:
  example-job:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup environment
        uses: kassett/setup-action@v3
        with:
          aws-secrets: "json-secret*,name=flat-secret"
          connect-to-vpn: "tailscale"
          install-docker: "true""
```