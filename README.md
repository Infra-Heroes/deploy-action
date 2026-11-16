# Deploy to Infra-Heroes

Deploys your app to [Infra-Heroes](https://infra-heroes.de) from a GitHub workflow with [heroctl](https://codeberg.org/InfraHeroes/heroctl).

1. Create a deploy token and store it as the repository secret `HERO_TOKEN`:
   ```bash
   heroctl tokens create --name github --scope deploy
   ```
2. Add `hero.toml` to your repository and create the project once (`heroctl projects create my-project`).
3. Add the workflow:

```yaml
name: Deploy to Infra-Heroes
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: infra-heroes/deploy-action@v1
        with:
          token: ${{ secrets.HERO_TOKEN }}
          project: my-project
```

| Input | Required | Default | |
|---|---|---|---|
| `token` | yes | | Deploy token |
| `project` | yes | | Project name |
| `image` | no | | Deploy this existing image instead of building (public or hero registry) |
| `working-directory` | no | `.` | Directory with `hero.toml` |
| `preview` | no | `false` | Deploy pull requests as their own app, see below |
| `github-token` | no | `github.token` | Comments the preview URL on the pull request (only with `preview`) |
| `heroctl-version` | no | `v1.8.0` | heroctl release |

The action first runs `heroctl validate` (problems in `hero.toml` show as annotations), then deploys and writes a summary to the job page.

Output `url`: the public URL of the app.

Without `image` the action builds the Docker image from your repository on the runner and pushes it to the Infra-Heroes registry; Docker is preinstalled on `ubuntu-latest`.

## Preview every pull request

With `preview: true` each pull request is deployed as its own app, `<app>-pr-<number>`, the action comments the address on the pull request, and the preview is deleted when the pull request is closed.

```yaml
name: Preview
on:
  pull_request:
    types: [opened, synchronize, reopened, closed]
permissions:
  contents: read
  pull-requests: write
jobs:
  preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        if: github.event.action != 'closed'
      - uses: infra-heroes/deploy-action@v1
        with:
          token: ${{ secrets.HERO_TOKEN }}
          project: my-project
          preview: true
```

What a preview is, and what it is not:

- It is your app under another name next to the real one. It gets **no custom domains and no volumes** (a volume belongs to one deployment and stays with the real app).
- A `[[databases]]` entry in `hero.toml` gives the preview its **own, empty database**, deleted together with the preview. Seed it from your app if it needs data.
- It **scales to zero**, so it only runs while somebody opens it; the first request after a pause is slow.
- It uses the project's secrets and the project's quota like any other app.
- **Pull requests from forks are skipped.** A fork's code would run with your deploy token, so the action does not deploy it (it says so in the job log).
- Closing the pull request deletes the preview. If the pull request cannot be checked out any more, the action reads `hero.toml` from the base branch (`contents: read`).

### Woodpecker and Forgejo

The action is GitHub only. On Woodpecker (Forgejo, Gitea, Codeberg) call heroctl yourself; the commands are what the action runs:

```yaml
# .woodpecker/preview.yml
when:
  event: pull_request

steps:
  preview:
    image: alpine
    environment:
      HERO_TOKEN:
        from_secret: hero_token
    commands:
      - apk add --no-cache curl docker-cli
      - curl -fsSL https://infra-heroes.de/install.sh | sh
      - heroctl deploy --project my-project --yes --token "$HERO_TOKEN" --preview "pr-$CI_COMMIT_PULL_REQUEST"
```

Delete a preview with `heroctl preview delete pr-<number> --project my-project --token "$HERO_TOKEN"`. Woodpecker has no pipeline event for a closed pull request, so on Woodpecker previews are deleted by hand or by a scheduled cleanup job.
