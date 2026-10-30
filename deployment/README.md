# Helmfile deployment

This directory contains the OpenADSim-specific inputs and manual additions for the [openads-helm generator](https://github.com/openads-project/openads-helm/tree/main/utils/generate_helmfile_from_compose), which converts `docker-compose.yml` into a Helmfile setup.

## Layout

- `helmfile.yaml` is the entrypoint. It loads the add-ons and the generated services.
- `chart-map.yaml` contains chart references and conversion rules that cannot be derived from Compose.
- `addons/` contains manually maintained releases: a NetworkPolicy that isolates the middleware bridge per stack, deployed with the `carla` profile.
- `generated/` is the generator output. It is ignored by Git and must not be committed.

## CI

The [GitHub workflow](../.github/workflows/helmfile-oci.yml) and the [GitLab pipeline](../.gitlab-ci.yml) use the reusable templates from `openads-helm`. They generate `deployment/generated`, validate it with `helmfile`, and publish the `deployment` directory together with the root `.env` as OCI bundle `<registry>/<repository>/helmfile`, tagged with the branch or tag name and `sha-<commit>`. Pull requests publish under their source branch name.

The scheduled [cleanup workflow](../.github/workflows/ghcr-cleanup.yml) deletes bundles older than one month except `main` and `vX.Y.Z`. On GitLab, the container registry cleanup policy of the project applies the same rule.

## Local usage

Generate the Helmfile with a sibling checkout of `openads-helm`:

```sh
# openadsim$
git clone https://github.com/openads-project/openads-helm ../openads-helm
python3 ../openads-helm/utils/generate_helmfile_from_compose/scripts/generate_helmfile_from_compose.py \
  docker-compose.yml \
  --env-file .env \
  --chart-map deployment/chart-map.yaml \
  --output-dir deployment/generated
```

Load the runtime environment, then validate and deploy. `COMPOSE_PROFILES` selects the services as in Compose. `PROJECT_PATH` must be the absolute checkout path on the Kubernetes node, as it is used for `hostPath` volumes.

```sh
set -a; . ./.env; set +a
export PROJECT_PATH="$(pwd)"

helmfile --file deployment/helmfile.yaml template --concurrency 1
helmfile --file deployment/helmfile.yaml apply
```

## Optional settings

To run all services on one node of a multi-node cluster, e.g. the node with the checkout at `PROJECT_PATH`, export its hostname as `NODE`:

```sh
kubectl get nodes -L kubernetes.io/hostname
export NODE=worker-1
```

`DEPLOYMENT_PREFIX` separates several stacks in one namespace. References to other services of the stack are prefixed automatically, e.g. `CARLA_HOST: carla-server` becomes `sim1-carla-server`:

```sh
export DEPLOYMENT_PREFIX=sim1
```
