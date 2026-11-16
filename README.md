# pipelines

The reusable GitHub Actions workflows every ClusterKeep repo calls. They run on the three self-hosted runners on the CI bastion, which hold the kubeconfig, tfvars and cluster-cli config that can't live in Git.

## Workflows

### `app-build.yml`

Called by each app's `build.yml` on every push.

| Branch | What happens |
|---|---|
| `development` | Tests, builds and pushes a `DEV-<timestamp>` image, then deploys it to dev with the app's OpenTofu `default` workspace |
| `preview` | The same with a `PREVIEW-<timestamp>` image and the `prv` workspace |
| `main` | Tests, builds and pushes a plain `<timestamp>` image without deploying it |

The tests and image build run inside the cluster through cluster-ci, so a failing test stops the image being pushed.

| Input | What it's for |
|---|---|
| `app` | Repo name, also the image and chart name |
| `namespace_suffix` | Suffix for the build's namespace label, defaults to `pub` |
| `namespace_override` | A fixed namespace label instead |

### `app-release.yml`

Called by each app's `release.yml`.

- Pushing a `vX.Y.Z` tag builds and pushes the image with that exact tag.
- Running the workflow by hand with a tag deploys that release to production. It runs the app's OpenTofu against the `prd` workspace with `prd.tfvars`, passing the tag as `image_tag`, so a first install gets the production settings and an old pinned tag can't roll production back.

| Input | What it's for |
|---|---|
| `app` | Repo name |
| `tag` | The release tag |
| `deploy` | Whether to deploy instead of build |
| `namespace_suffix`, `namespace_override` | As for `app-build.yml` |

### `tofu-deploy.yml`

Called by the infrastructure repos.

- Pushing a `vX.Y.Z` tag runs `tofu plan` for that tag so it can be reviewed.
- Running the workflow by hand with a tag and `apply` set runs `tofu apply`.

Nothing in the infrastructure repos is applied automatically.

| Input | What it's for |
|---|---|
| `repo` | Repo name, as listed in cluster-cli |
| `tag` | The tag to plan or apply |
| `apply` | Apply instead of plan |

## How a job runs

Every job starts by running `sync-repos.sh` on the bastion to update cluster-cli, the `charts` repo and the repo being built, then uses cluster-cli to build, run OpenTofu or deploy.

A new repo has to be cloned on the bastion before its first build, using its own read-only deploy key and SSH alias. The bastion's default key can't reach GitHub, so a first-time clone from the workflow fails.
