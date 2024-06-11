# GitOps and Docker Compose

This is a boilerplate repo to user docker compose with an GitOps approach.

Deployment of containers are done by Github Actions and selfhosted runners.
As docker isn't like k8s ans support tools like flux, this could be an alternative to manage your
docker compose in a git repository with automatic deployments on the docker host.

## Wiki

### Precommit

This repo uses [Pre-Commit](https://pre-commit.com/) to lint some things before committing and pushing to git.

1. [Install Pre-Commit](https://pre-commit.com/#install) according to there documentation on your system
1. run `precommit install` in the root of the git repository to install the pre-commit

### Add a new Application

- Create a folder for the application in `/services` and put docker-compose file and other configuration files in there

- Modify the config files in `/config`:

- Add the App to the `app-[host].yml` filter file for the corresponding host and make sure that the two application names are the same:

```yaml
application-name:
  - 'services/application-name/**'
```

- Add the App to the `system.yml` config file under the section of the corresponding host:

```yaml
host-1:
  - 'services/application-1/**'
  - 'services/application-2/**'

host-2:
  - 'services/application-1/**'
  - 'services/application-3/**'
'
```

- add the application to the `Services` section of the README.md file

### Add a new Docker-Host

- Follow the official documentation for [How to add a Runner](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/adding-self-hosted-runners#adding-a-self-hosted-runner-to-a-repository)
- In the configuration setup: add a tag to the runner similar to the hostname, e.g. `docker-prod-1`. This tag is used to que the applications to the right docker host
- to run the runner as a service, execute `./svc.sh install` and `./svc.sh start`
- add the label from the runner in `/.github/actionlint.yaml`

```yaml
self-hosted-runner:
  labels:
    - runner-label
```

- create a application config file in `/config` with the name `app-[hostname].yml`
- create a section in the system config `/config/system.yml`
- add the runner to the deployment pipeline in `./github/workflows/docker-deployment-prod.yml`.

#### Deployment-pipeline changes:

**change:**

- output-name: `<hostname>`

```yaml
check-changes-system:
    outputs:
      <hostname>: ${{ steps.changes-system.outputs.<hostname> }}
```

**change:**

- Job-Name: `<hostname>`
- outputs: app-`<hostname>`
- step-name: `changes-docker-<environment>`
- filters: config/`<config-file>`

```yaml
  check-changes-app-<hostname>:
    needs: [validate-yaml]
    runs-on: ubuntu-latest

    outputs:
      app-<hostname>: ${{ steps.changes-app.outputs.changes }}

    steps:
      - uses: actions/checkout@v4

      - name: changes-docker-<environment>
        uses: dorny/paths-filter@v3
        id: changes-app
        with:
          base: 'main'
          list-files: 'json'
          filters: config/<config-file>
```

**change:**

- Job-Name: `<hostname>`
- if: `<hostname>`
- runs-on: `<runner-label>`

```yaml
  update-repo-<hostname>:
    needs: [check-changes-system]
    if: ${{ needs.check-changes-system.outputs.<hostname> == 'true' }}
    runs-on: <runner-label>

    steps:
      - name: update-repository
        run: |
          cd ${{ vars.GIT_DOCKER_REPO }}
          git pull
          git checkout main
          git pull
```

**change:**

- Job-Name: `<environment>`
- needs: `<hostname>`
- if: `<hostname>`
- runs-on: `<runner-label>`
- environment: `<environment>`
- matrix-app: `<hostname>`

```yaml
  deploy-docker-<environment>:
    needs: [check-changes-system, check-changes-app-<hostname>, update-repo-<hostname>]
    if: ${{ needs.check-changes-system.outputs.<hostname> == 'true' }}
    runs-on: <runner-label>
    environment: <environment>
    strategy:
      matrix:
        app: ${{ fromJSON(needs.check-changes-app-<hostname>.outputs.app-<hostname>) }}

    steps:
      - name: deploy ${{ matrix.app }}
        run: |
          cd ${{ vars.GIT_DOCKER_REPO }}/services/${{ matrix.app }}
          docker compose pull
          docker compose up -d --remove-orphans
```

**change:**

- Job-Name: `<environment>`
- needs: `<hostname>`
- runs-on: `<runner-label>`
- step-name: `<hostname>`
- guid: `<newrelic_entity_guid>`

```yaml
  review-docker-<environment>:
    needs: [deploy-docker-<hostname>, check-changes-app-<hostname>]
    runs-on: <runner-label>
    steps:
      - name: Running Containers
        id: running-containers
        run: docker ps

      - name: Running Stacks
        id: running-stacks
        run: docker compose ls

      - name: Change Marker <hostname>
        uses: newrelic/deployment-marker-action@v2.3.0
        with:
          apiKey: ${{ secrets.NEW_RELIC_API_KEY }}
          guid: "<newrelic_entity_guid>"
          version: "${{ github.run_number }}"
          user: "${{ github.actor }}"
```
