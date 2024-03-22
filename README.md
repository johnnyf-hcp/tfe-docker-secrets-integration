# Docker secrets integration with HashiCorp Terraform Enterprise

## Background

This is a guide to setting up Docker secrets integration with HashiCorp Terraform Enterprise (TFE) in external services mode.

Docker secrets can be created using the [Docker CLI](https://docs.docker.com/reference/cli/docker/secret/), allowing you to store sensitive data securely.

## Changes made to the original configuration

[compose-initial.yaml](compose-initial.yaml) contains the Docker compose configuration shared by Hashicorp on their [documentation](https://developer.hashicorp.com/terraform/enterprise/flexible-deployments/install/docker/install#external-services).

[compose-stack.yaml](compose-stack.yaml) contains the Docker compose configuration for the stack to be deployed on a Docker Swarm.

In order to use docker secrets in external mode, we have to first enable swarm mode:

```bash
docker swarm init
```

Then we can copy secrets to the server and create docker secrets

```bash
scp -i tfe-development.pem -r secrets ubuntu@<ec2-hostname>:~/
ssh -i tfe-development.pem ubuntu@<ec2-hostname>
cd secrets
docker secret create <secret-name> <filename>
docker secret ls # verify that secret is created
rm <filename> # remove file after creating secret
```

Now that swarm mode is enabled, we have to modify our compose file. See [`compose-stack.yaml`](compose-stack.yaml) for an example.

Notably,

- `name` top level element has to be removed as it is not allowed in swarm mode
  - Because `name` is removed, the variable `COMPOSE_PROJECT_NAME` is now empty, which causes issues as docker will complain about creating `_terraform-enterprise-cache` directory, because it starts with an underscore
  - We have to make modify the configuration `TFE_DISK_CACHE_VOLUME_NAME`

- `tmpfs` top level element has to be removed as it is not allowed in swarm mode. We instead create these tmpfs volumes in the `volumes` section
- attach `secrets` top level element to the compose file, and reference the secrets in the `tfe` service
- See [`compose-stack.yaml`](compose-stack.yaml) for the full example.
- Because we now use swarm mode, we operate with `docker stack` instead of `docker compose`

```bash
docker stack deploy --compose-file compose-stack.yaml tfe # deploy
docker service logs tfe_tfe # inspect logs, or use
docker ps # to get container id, then
docker logs <container-id> # inspect logs
docker stack rm tfe # shutdown
```
