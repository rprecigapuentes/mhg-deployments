# mhg-deployments

Ansible role that deploys the Monster Hunter Guild API with a blue/green strategy behind nginx. Only `ansible.builtin` modules are used.

## deploy_backend

Deploys the API plus its MySQL database as a Docker Compose stack in `/opt/mhg`. Two colours, `blue` and `green`, run side by side with independently pinned image tags. nginx serves the public port and proxies to whichever colour is active, so a new version is started on the idle colour, verified, and only then given traffic. The previous colour keeps running for rollback.

### Prerequisites

On the target host:

- Docker Engine and the Docker Compose plugin.
- The SSH user in the `docker` group.

On the control node:

- A virtualenv with `ansible-core`.
- A GitLab deploy token for `monster-hunter-guild` with the `read_registry` scope.

### Set the secrets

Secrets are read from the control node's environment, never from a file in this repository.

```bash
cp secrets.env.example secrets.env    # gitignored; fill in the real values
set -a; source secrets.env; set +a
```

Required: `MHG_REGISTRY_USER`, `MHG_REGISTRY_TOKEN`, `MHG_DB_ROOT_PASSWORD`. The role fails on the first task if any is empty.

### Run

```bash
ansible-playbook -i inventory/hosts.yml deploy_backend_dev.yml 
```

### Deploy a new version

Deploying is two edits in `deploy_backend_dev.yml` and one run.

1. Find the live colour:

   ```bash
   docker exec mhg-lb grep server /etc/nginx/nginx.conf
   ```

2. Set the **idle** colour's version to the new tag and make it active. If blue is live:

   ```yaml
   deploy_backend_active_env: green
   deploy_backend_version_green: "v1.3.0"
   ```

3. Run the playbook. It pulls the tag, starts green beside blue, polls green until it answers `200`, and only then moves the nginx upstream. If the check never passes the play fails before nginx is touched and blue keeps serving.

Image tags carry a `v` prefix: the pipeline publishes `<image>:v$VERSION`, so `v1.2.0`, not `1.2.0`.

### Roll back

Set `deploy_backend_active_env` back to the previous colour and run again. That colour never stopped, so the rollback is a single nginx reload.

### Variables

Set per deploy, in `deploy_backend_dev.yml`:

| Variable | Purpose |
| --- | --- |
| `deploy_backend_active_env` | Colour that receives traffic after this run, `blue` or `green` |
| `deploy_backend_version_blue` | Image tag for the blue container |
| `deploy_backend_version_green` | Image tag for the green container |

Role defaults, in `roles/deploy_backend/defaults/main.yml`:

| Variable | Default | Purpose |
| --- | --- | --- |
| `deploy_backend_dir` | `/opt/mhg` | Directory holding `compose.yml`, `.env`, and `nginx.conf` on the target |
| `deploy_backend_image_name` | `registry.gitlab.com/jalau-bootcamps/bc-lt-at-fs-05/monster-hunter-guild` | Image repository, without tag |
| `deploy_backend_db_name` | `monster_hunter_guild` | MySQL database name |
| `deploy_backend_public_port` | `5000` | Host port served by nginx |
| `deploy_backend_port_blue` | `3001` | Loopback-only host port for the blue container |
| `deploy_backend_port_green` | `3002` | Loopback-only host port for the green container |
| `deploy_backend_health_path` | `/guilds` | Path polled on the new colour before the switch. It reaches the database, so a `200` proves the API, the MySQL connection, and the migrated schema |
| `deploy_backend_health_retries` | `30` | Attempts before the play fails |
| `deploy_backend_health_delay` | `3` | Seconds between attempts |
| `deploy_backend_registry_host` | `registry.gitlab.com` | Container registry |
| `deploy_backend_registry_user` | from `MHG_REGISTRY_USER` | Registry user |
| `deploy_backend_registry_token` | from `MHG_REGISTRY_TOKEN` | Registry token |
| `deploy_backend_db_root_password` | from `MHG_DB_ROOT_PASSWORD` | MySQL root password |
