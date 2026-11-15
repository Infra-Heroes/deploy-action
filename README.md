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
| `heroctl-version` | no | `v1.4.0` | heroctl release |

The action first runs `heroctl validate` (problems in `hero.toml` show as annotations), then deploys and writes a summary to the job page.

Output `url`: the public URL of the app.

Without `image` the action builds the Docker image from your repository on the runner and pushes it to the Infra-Heroes registry; Docker is preinstalled on `ubuntu-latest`.
