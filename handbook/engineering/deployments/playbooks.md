# Playbooks for deployments

- [Playbooks for deployments](#playbooks-for-deployments)
    - [Sourcegraph.com](#sourcegraphcom)
        - [Deploying to sourcegraph.com](#deploying-to-sourcegraphcom)
        - [Rolling back sourcegraph.com](#rolling-back-sourcegraphcom)
        - [Invalidating all user sessions](#invalidating-all-user-sessions)
        - [Accessing sourcegraph.com database](#accessing-sourcegraphcom-database)
            - [Via the CLI](#via-the-cli)
            - [Via BigQuery (for read-only operations)](#via-bigquery-for-read-only-operations)
        - [Restarting docs.sourcegraph.com](#restarting-about-sourcegraph-com-and-docs-sourcegraph-com)
    - [k8s.sgdev.org](#k8ssgdevorg)
        - [Manage users in k8s.sgdev.org](#manage-users-in-k8ssgdevorg)
    - [Cloudflare Configuration](#cloudflare-configuration)

## Sourcegraph.com

To learn more about this deployment, see [instances](./instances.md#sourcegraph-com).

### Deploying to sourcegraph.com

Every commit to the `release` branch (the default branch) on [deploy-sourcegraph-dot-com](https://github.com/sourcegraph/deploy-sourcegraph-dot-com) deploys the Kubernetes YAML in this repository to our dot-com cluster [in CI](https://buildkite.com/sourcegraph/deploy-sourcegraph-dot-com/builds?branch=release) (i.e. if CI is green then the latest config in the `release` branch is deployed).

Deploys on sourcegraph.com are currently [handled by Renovate](#renovate). The [Renovate dashboard](https://app.renovatebot.com/dashboard#github/sourcegraph/deploy-sourcegraph-dot-com) shows logs for previous runs and allows you to predict when the next run will happen.

If you want to expedite a deploy, you can manually create and merge a PR that updates the Docker image tags in [deploy-sourcegraph-dot-com](https://github.com/sourcegraph/deploy-sourcegraph-dot-com). You can find the desired Docker image tags by looking at the output of the Docker build step in [CI on sourcegraph/sourcegraph `main` branch](https://buildkite.com/sourcegraph/sourcegraph/builds?branch=main) or by looking at [Docker Hub](https://hub.docker.com/u/sourcegraph/).


### Rolling back sourcegraph.com

To roll back soucegraph.com, push a new commit to the `release` branch in [deploy-sourcegraph-dot-com](https://github.com/sourcegraph/deploy-sourcegraph-dot-com) that reverts the image tags and configuration to the desired state.

```sh
# Ensure that you're up-to-date
git checkout release
git pull

# Rollback the release branch to $COMMIT
# See https://stackoverflow.com/a/21718540 if you want more context
git revert --no-commit $COMMIT..HEAD

# Push the revert commit back to the release branch
git commit
git push origin release
```

[Buildkite](https://buildkite.com/sourcegraph/deploy-sourcegraph-dot-com/) will deploy the working commit to sourcegraph.com.

🚨 You also need to disable auto-deploys to prevent Renovate from automatically merging in image digest updates so that the site doesn't roll-forward.

1. Go to [renovate.json](https://github.com/sourcegraph/deploy-sourcegraph-dot-com/blob/release/renovate.json) and remove the `"extends:["default:automergeDigest"]` entry for the "Sourcegraph Docker images" group ([example](https://github.com/sourcegraph/deploy-sourcegraph-dot-com/commit/0eb16fd9e3ddfcf3a3c75ccdda0e7eddabf19c7a)).
1. Once you have fixed the issue in the `main` branch of [sourcegraph/sourcegraph](https://github.com/sourcegraph/sourcegraph), re-enable auto-deploys by reverting your change to [renovate.json](https://github.com/sourcegraph/deploy-sourcegraph-dot-com/blob/release/renovate.json) from step 1.
1. Go to [open pull requests](https://github.com/sourcegraph/deploy-sourcegraph-dot-com/pulls/app%2Frenovate) and ensure there no pending renovate PR's without an automerge label. If there are, apply the automerge label to the PR's to ensure the `release` branch is continuously updated.

### Backing up & restoring a Cloud SQL instance (production databases)

Before any potentially risky operation you should ensure the databases have recent ( < 1 hour) backups. We currently have daily backups enabled.

You can create a backup of a Cloud SQL instance via `gcloud sql backups create --instance=${instance_name} --project=sourcegraph-dev`

To restore a Cloud SQL instance to a previous revision you can use `gcloud sql backups restore $BACKUP_ID --restore-instance=${instance_name}`

You can also perform these commands from the [Google Cloud SQL UI](https://console.cloud.google.com/sql/instances?project=sourcegraph-dev)

🚨 You should notify the #dev-ops channel if an situation arises when a restore my be required. It should also be filed in our ops-incident log.

### Invalidating all user sessions

If all user sessions need to be invalidated, you can run this on the `frontend` database to force all users to log in again.

```
UPDATE users SET invalidated_sessions_at=now(), updated_at=now();
```

### Accessing sourcegraph.com database

#### Via the CLI

Sourcegraph.com utilizes an external HA database. You will need to connect to it directly. The easiest way to do this is through the `gcloud` cli.

To connect to the production database:

```
  gcloud beta sql connect sg-cloud-732a936743 --user=sg -d sg --project sourcegraph-dev
```

However, if you want to use any other SQL client, you'll have to run the [`cloud_sql_proxy`](https://cloud.google.com/sql/docs/postgres/connect-admin-proxy#install) utility, which authenticates with you local `gcloud` credentials automatically.

```
  cloud_sql_proxy -instances=sourcegraph-dev:us-central1:sg-cloud-732a936743=tcp:5555
```

Once the proxy connects successfully, you can use any client to connect to the local `5555` port (you can choose any other port you want).

The password of the sg user is in our shared 1Password under [Google Cloud SQL](https://team-sourcegraph.1password.com/vaults/dnrhbauihkhjs5ag6vszsme45a/allitems/svfiw4vcbxhhbobpl442olyebu)

#### Via BigQuery (for read-only operations)

You can also query the production database via BigQuery as an external data source.

See an [example query](https://console.cloud.google.com/bigquery?sq=527047051561:bfa7c7e57f884d209f261d15e4610229) to get started.

**Note**: This method only permits read-only access

### Restarting about.sourcegraph.com and docs.sourcegraph.com

To restart the services powering about.sourcegraph.com and docs.sourcegraph.com:

1. List the active pods with `kubectl get pods`. This should produce a list that includes items like the following:

    ```
    NAME                                     READY   STATUS      RESTARTS   AGE
    about-sourcegraph-com-74f96c659b-t6lqp   1/1     Running     0          7m49s
    docs-sourcegraph-com-5f97dd5db-7wrxm     1/1     Running     0          7m49s 
    ```

    The exact names will differ, but the general format will be the same.
1. Delete the pods, being careful to copy the exact names from the output in the previous step. For example, with that output, you would run:

    ```
    kubectl delete pod about-sourcegraph-com-74f96c659b-t6lqp docs-sourcegraph-com-5f97dd5db-7wrxm
    ```
1. Wait a moment, and check https://about.sourcegraph.com/ and https://docs.sourcegraph.com/.

## k8s.sgdev.org

To learn more about this deployment, see [instances](./instances.md#k8s-sgdev-org).

### Manage users in k8s.sgdev.org

To create an account on [k8s.sgdev.org](https://k8s.sgdev.org), log in with your Sourcegraph Google account via OpenID Connect.

To promote a user to site admin (required to make configuration changes), use the admin user credentials available in 1password (titled `k8s.sgdev.org admin user`) to log in to [k8s.sgdev.org](https://k8s.sgdev.org), and go to the [users page](https://k8s.sgdev.org/site-admin/users) to promote the desired user.

## Cloudflare Configuration

For documentation on how to configure Cloudflare's WAF and rate limiter, see the [security documentation](./security.md#cloudflare).
