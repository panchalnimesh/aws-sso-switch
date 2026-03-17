# aws-sso-switch

A lightweight CLI tool that dynamically discovers all your AWS SSO accounts and roles, lets you pick one via `fzf`, and exports temporary credentials directly into your shell — no AWS profile pre-configuration required.

---

## Features

- Discovers all accessible accounts and roles from your active SSO token
- Interactive fuzzy search via `fzf`
- Exports temporary credentials as environment variables
- Always saves selected credentials as a named AWS profile (account name by default)
- Supports multiple SSO sessions / orgs
- Pre-filter by keyword for fast switching

---

## Requirements

- Python 3.7+
- [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
- [fzf](https://github.com/junegunn/fzf)

```bash
brew install awscli fzf
```

---

## Installation

```bash
# Copy the script to your local bin
cp aws-sso-switch ~/.local/bin/aws-sso-switch
chmod +x ~/.local/bin/aws-sso-switch

# Make sure ~/.local/bin is in your PATH (add to ~/.zshrc or ~/.bashrc)
export PATH="$HOME/.local/bin:$PATH"
```

### Shell wrapper (recommended)

Add this to your `~/.zshrc` (or `~/.bashrc`) so credentials are exported into the current shell:

```bash
function sw() {
  eval "$(aws-sso-switch "$@")"
}
```

Then reload:

```bash
source ~/.zshrc
```

---

## AWS SSO Configuration

Add your SSO sessions to `~/.aws/config`:

```ini
[sso-session prod]
sso_start_url = https://your-sso-portal.awsapps.com/start/
sso_region = ap-southeast-1
sso_registration_scopes = sso:account:access

[sso-session staging]
sso_start_url = https://your-staging-sso-portal.awsapps.com/start/
sso_region = ap-southeast-1
sso_registration_scopes = sso:account:access
```

Login to get a token:

```bash
aws sso login --sso-session prod
```

---

## Usage

```bash
# Interactive account/role picker — always saves a profile named after the account
sw

# Pre-filter by keyword
sw -f prod
sw -f faber

# Use a specific SSO session
sw -s prod
sw -s staging

# List all accessible accounts/roles without switching
aws-sso-switch -l

# Save selected credentials under a custom profile name
sw -p my-prod-admin

# Combine flags
sw -s prod -f faber
sw -s prod -p my-prod-admin

# Show current AWS context
aws-whoami   # if installed, or: aws sts get-caller-identity
```

---

## Named AWS Profile (automatic)

Every `sw` invocation saves credentials to `~/.aws/credentials` and `~/.aws/config` as a named profile, in addition to exporting them as environment variables.

**Default** — profile name is derived from the AWS account name:
```bash
sw
# Selects e.g. "faber" account → saves profile named "faber"
```

**Custom name** — override with `-p`:
```bash
sw -p my-prod-admin
# Saves as "my-prod-admin" regardless of account name
```

After saving, `AWS_PROFILE` is also exported, so any tool that respects `AWS_PROFILE` will use it automatically:

```bash
sw
aws s3 ls                    # uses account-name profile via AWS_PROFILE
aws --profile faber s3 ls    # explicit profile reference
```

### What gets written

**`~/.aws/credentials`:**
```ini
[my-prod-admin]
aws_access_key_id = ASIA...
aws_secret_access_key = ...
aws_session_token = ...
```

**`~/.aws/config`:**
```ini
[profile my-prod-admin]
region = ap-southeast-1
output = json
x_account_name = My Account
x_account_id = 123456789012
x_role_name = AdminRole
```

> **Note:** SSO temporary credentials expire (typically ~1 hour). Re-run `sw` after expiry to refresh.

---

## Environment Variables Exported

After switching, the following variables are set in your shell:

| Variable | Description |
|---|---|
| `AWS_ACCESS_KEY_ID` | Temporary access key |
| `AWS_SECRET_ACCESS_KEY` | Temporary secret key |
| `AWS_SESSION_TOKEN` | Temporary session token |
| `AWS_DEFAULT_REGION` | Region from SSO session |
| `AWS_ACCOUNT_ID` | Selected account ID |
| `AWS_ACCOUNT_NAME` | Selected account name |
| `AWS_ROLE_NAME` | Selected role name |
| `AWS_PROFILE` | Profile name (account name by default, or custom via `-p`) |

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `No valid SSO token found` | `aws sso login --sso-session prod` |
| Token expired mid-session | Re-run `aws sso login`, then `sw` again |
| No accounts listed | Wrong SSO session — use `sw -s <session>` |
| `fzf: command not found` | `brew install fzf` |
| `sw: command not found` | `source ~/.zshrc` or open a new terminal |
| `aws-sso-switch: command not found` | Check `~/.local/bin` is in your `$PATH` |

---

## How It Works

1. Reads `~/.aws/sso/cache/*.json` to find a valid (non-expired) access token
2. Calls `aws sso list-accounts` → `aws sso list-account-roles` to discover all accessible accounts/roles
3. Presents them in `fzf` for interactive selection
4. Calls `aws sso get-role-credentials` for the selected account/role
5. Emits `export` statements — the `sw` wrapper runs them via `eval`
6. Writes credentials to `~/.aws/credentials` and `~/.aws/config` under a profile named after the account (use `-p name` to override)
