# Setup GitHub App

A GitHub Action to generate a GitHub App installation token and automatically configure Git identity for verified bot commits.

## Features

- 🔑 **GitHub App Token Generation**: Generates an installation access token using GitHub App credentials.
- 🤖 **Verified Bot Git Identity**: Automatically resolves the App bot user ID and configures Git (`user.name` and `user.email`) for verified commit attribution.
- 🔄 **Trigger Downstream Workflows**: Actions performed using GitHub App tokens can trigger other GitHub Actions workflows (unlike the default `GITHUB_TOKEN`).
- ⚡ **Lightweight Composite Action**: No extra build step or runtime dependencies required; runs directly with standard GitHub Actions runners.
- ⚙️ **Configurable**: Easily toggle automated Git identity setup on or off.

---

## Usage

### 1. Basic Usage (Generate Token & Configure Git Identity)

```yaml
name: CI / Automated Commit

on:
  push:
    branches: [main]

jobs:
  commit-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Setup GitHub App
        id: app
        uses: abhisin98/avivox-action-setup-github-app@v1
        with:
          app-client-id: ${{ secrets.APP_CLIENT_ID }}
          app-private-key: ${{ secrets.APP_PRIVATE_KEY }}

      # Checkout can happen after this action. Git identity is configured globally.
      - name: Checkout Repository
        uses: actions/checkout@v4
        with:
          token: ${{ steps.app.outputs.app-token }}

      - name: Make Changes and Commit
        run: |
          echo "Automated update: $(date)" >> build.log
          git add build.log
          git commit -m "chore: automated build update [skip ci]"
          git push
```

### 2. Token-Only Usage (Disable Git Identity Setup)

If you only need the token for API calls or other actions and don't need Git identity configured:

```yaml
steps:
  - name: Setup GitHub App Token
    id: app
    uses: abhisin98/avivox-action-setup-github-app@v1
    with:
      app-client-id: ${{ secrets.APP_CLIENT_ID }}
      app-private-key: ${{ secrets.APP_PRIVATE_KEY }}
      auto-setup-git-identity: false

  - name: Call GitHub API
    run: |
      gh api repos/${{ github.repository }}/dispatches \
        -f event_type='trigger-deploy'
    env:
      GH_TOKEN: ${{ steps.app.outputs.app-token }}
```

---

## Inputs

| Input | Description | Required | Default |
| --- | --- | --- | --- |
| `app-client-id` | The Client ID (or App ID) of your GitHub App. | **Yes** | — |
| `app-private-key` | The private key of your GitHub App in PEM format. | **Yes** | — |
| `auto-setup-git-identity` | If `true`, automatically configures git `user.name` and `user.email` for the GitHub App bot. | No | `true` |

---

## Outputs

| Output | Description |
| --- | --- |
| `app-token` | The generated GitHub App installation access token. |
| `app-slug` | The slug (name) of the GitHub App. |
| `user-id` | The numeric user ID of the GitHub App bot. |

---

## How It Works

1. **Token Generation**: Uses [`actions/create-github-app-token@v3`](https://github.com/actions/create-github-app-token) to generate an installation token from the provided App credentials.
2. **Bot Identity Resolution**: Fetches the numeric user ID of `<app-slug>[bot]` from the GitHub API using GitHub CLI (`gh`).
3. **Git Configuration**: If `auto-setup-git-identity` is `true`, configures global Git settings on the runner:
   - `user.name`: `<app-slug>[bot]`
   - `user.email`: `<user-id>+<app-slug>[bot]@users.noreply.github.com`

The configuration is global so the action can run before `actions/checkout` without failing with `fatal: not in a git directory`. It also applies to repositories checked out later in the same job.

This email structure matches GitHub's standard noreply email format for GitHub Apps, ensuring commits display the verified `bot` badge in GitHub's web interface.

---

## Prerequisites & Setup

1. **Create a GitHub App**:
   - Go to your Organization/Account Settings → **Developer Settings** → **GitHub Apps** → **New GitHub App**.
   - Set the necessary repository and organization permissions (e.g., Contents: Read & write, Pull requests: Read & write).
   - Generate and download a **Private Key** (PEM format).
   - Note down the **Client ID** (or **App ID**).
   - Install the GitHub App on your target repository or organization.

2. **Add Repository Secrets**:
   - `APP_CLIENT_ID`: The Client ID / App ID of your GitHub App.
   - `APP_PRIVATE_KEY`: The entire content of the downloaded `.pem` private key file.

---

## License

[MIT](LICENSE)