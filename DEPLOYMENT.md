# Production deployment

The `Production security, build and deploy` workflow implements:

`GitHub pull request → Trivy → CodeRabbit → merge to main → Docker build → Docker Hub → EC2 → health check → summary → Prometheus/Grafana`

CodeRabbit runs as a GitHub App, not inside a GitHub-hosted runner. Install the CodeRabbit app for this repository, then protect `main` and require both **Trivy repository scan** and the CodeRabbit review check before merge. The committed `.coderabbit.yaml` configures its reviews.

## One-time setup

Use an Ubuntu 22.04 or 24.04 EC2 instance. Allow inbound SSH (22) only from trusted addresses and HTTP (80) publicly. Ports 3000 and 9090 bind to localhost by default and should not be opened in the security group.

Add these GitHub **Actions secrets**:

- `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` (create an access token in Docker Hub; do not use your account password)
- `EC2_HOST` (public IP or DNS name)
- `EC2_SSH_PRIVATE_KEY` (paste the **complete contents** of the EC2 `.pem` file, including the `BEGIN` and `END` lines)
- `EC2_KNOWN_HOSTS` (the verified output of `ssh-keyscan -H your-host`)
- `GHOST_URL` (for example `http://blog.example.com` or the EC2 public URL)
- `MYSQL_ROOT_PASSWORD` and `MYSQL_PASSWORD`
- `GRAFANA_ADMIN_PASSWORD`

Optional GitHub **Actions variables**:

- `IMAGE_NAME` (defaults to `ghost-custom`)
- `EC2_USER` (defaults to `ubuntu`)

Create a protected GitHub environment named `production`. Pushing or merging to `main` then builds the custom Ghost distribution, publishes immutable `sha-...` and `latest` Docker Hub tags, installs Docker on EC2 when needed, deploys the exact SHA image, checks health, and writes the result to the workflow summary.

GitHub Secrets cannot receive a file upload. Open the PEM file as text and paste all of it into `EC2_SSH_PRIVATE_KEY`. The workflow writes it to a permission-protected temporary SSH key on the runner. The EC2 security group must permit inbound TCP port 22 from the GitHub-hosted runner; the selected EC2 user must be able to run `sudo` without an interactive password.

Persistent named volumes retain Ghost content, MySQL, Prometheus, and Grafana data across deployments. Back up `ghost-production_ghost-content` and `ghost-production_mysql-data` independently; deployment is not a backup.

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
