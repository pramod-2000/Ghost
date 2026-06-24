# Production deployment

The `Production build and deploy` workflow implements:

`GitHub pull request → CodeRabbit → merge to main → Docker build → GitHub Container Registry → EC2 → health check → summary → Prometheus/Grafana`

CodeRabbit runs as a GitHub App, not inside a GitHub-hosted runner. Install the CodeRabbit app for this repository, then protect `main` and require its review check before merge. The committed `.coderabbit.yaml` configures its reviews.

## One-time setup

Use an Ubuntu 22.04 or 24.04 EC2 instance. Allow inbound SSH (22) only from trusted addresses and HTTP (80) publicly. Ports 3000 and 9090 bind to localhost by default and should not be opened in the security group.

Add these GitHub **Actions secrets**:

- `EC2_SSH_PRIVATE_KEY` (paste the **complete contents** of the EC2 `.pem` file, including the `BEGIN` and `END` lines)
- `EC2_USER` (optional; defaults to `ubuntu`)

GitHub **Actions variables**:

- `EC2_HOST` (required public IP or DNS name; the workflow automatically records its SSH host key)
- `IMAGE_NAME` (defaults to `ghost-custom`)
- `GHOST_URL` (optional custom domain; defaults to `http://EC2_HOST`)

Create a protected GitHub environment named `production`. Pushing or merging to `main` then builds the custom Ghost distribution, publishes immutable `sha-...` and `latest` tags to GitHub Container Registry (`ghcr.io`) using the workflow's automatic `GITHUB_TOKEN`, installs Docker on EC2 when needed, deploys the exact SHA image, checks health, and writes the result to the workflow summary.

GitHub Secrets cannot receive a file upload. Open the PEM file as text and paste all of it into `EC2_SSH_PRIVATE_KEY`. The workflow writes it to a permission-protected temporary SSH key on the runner. The EC2 security group must permit inbound TCP port 22 from the GitHub-hosted runner; the selected EC2 user must be able to run `sudo` without an interactive password.

Persistent named volumes retain Ghost content, MySQL, Prometheus, and Grafana data across deployments. Back up `ghost-production_ghost-content` and `ghost-production_mysql-data` independently; deployment is not a backup.

The first deployment generates random MySQL and Grafana passwords directly on EC2 and stores them in `/opt/ghost/.env` with mode `600`. Later deployments preserve those values. To retrieve the initial Grafana password from your own terminal, run `ssh ubuntu@EC2_HOST 'grep ^GRAFANA_ADMIN_PASSWORD= /opt/ghost/.env'`.

## Monitoring access

Keep monitoring private and tunnel it over SSH:

```bash
ssh -L 3000:127.0.0.1:3000 -L 9090:127.0.0.1:9090 ubuntu@EC2_HOST
```

Open Grafana at `http://localhost:3000` and Prometheus at `http://localhost:9090`. Grafana starts with a provisioned Prometheus source and the **Ghost Production Overview** dashboard.

To validate the compose model locally without starting containers:

```bash
cp deployment.env.example .env
docker compose -f compose.production.yaml config --quiet
```
