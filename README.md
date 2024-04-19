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

- Modify the deployment pipeline in `/.github/workflows/docker-deployment-prod.yml`:

- Add the App to the output variables to check for changes:

```yaml
jobs:
    check-changes:
        outputs:
            application: ${{ steps.changes-prod.outputs.application }}
```

- Add the App to the filter list:

```yaml
filters: |
    application:
        - 'services/application/**'
```

- add deployment block in the section from the docker host where the app should be deployed, e.g.: `deploy-docker-prod-1`

```yaml
- name: deploy-application
if: needs.check-changes.outputs.application == 'true'
run: |
    cd ${{ vars.GIT_DOCKER_REPO }}/services/application
    docker compose pull
    docker compose up -d --remove-orphans
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

- add the runner to the deployment pipeline in `./github/workflows/docker-deployment-prod.yml`.

deployment-job:

**change:**

- name: `deploy-<docker-host-name>`
- runs-on: `<runner-label>`

```yaml
deploy-<docker-host-name>:
    needs: [validate-yaml, check-changes]
    runs-on: <runner-label>
    steps:
      - name: update-repository
        run: |
          cd ${{ vars.GIT_DOCKER_REPO }}
          git pull
          git checkout main
          git pull
```

review-job:

**change:**

- name: `review-<docker-host-name>`
- needs: `deploy-<docker-host-name>`
- runs-on: `<runner-label>`

```yaml
review-<docker-host-name>:
    needs: [deploy-<docker-host-name>]
    runs-on: <runner-label>
    steps:
      - name: Running Containers
        run: docker ps
      - name: Running Stacks
        run: docker compose ls
```
