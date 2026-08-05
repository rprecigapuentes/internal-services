# internal-services

Ansible roles that deploy the team's internal services with Docker Compose. Ansible is agentless: `ansible-playbook` runs from your machine and configures the target over SSH. Pushing to GitLab deploys nothing.

## sonarqube

Deploys SonarQube Community Edition plus its PostgreSQL database as a Compose stack in `/opt/sonarqube`.

The Compose file is copied verbatim from `roles/sonarqube/files/compose.yml`, keeping Compose's own `${VAR:-default}` and `${VAR:?required}` interpolation intact. Only the environment is rendered, into `/opt/sonarqube/.env` (mode `0640`, root-owned), so the database password never lands in a world-readable file. Every task uses `ansible.builtin` modules only.

### Prerequisites

On the control machine:

```bash
cd ~/jalasoft/prog102/internal-services
. .ansible/bin/activate      # ansible-core 2.21.2
sudo apt install sshpass     # required by -k (password auth)
```

On the target host: SSH access, a user with sudo rights, and Docker Engine plus the Compose plugin already installed. The role checks for Docker and fails early with a clear message, but does not install it.

### Set the database password

`sonarqube_db_password` has no default on purpose. Copy the example file and set the real value:

```bash
cp group_vars/services.yml.example group_vars/services.yml
$EDITOR group_vars/services.yml
```

`group_vars/services.yml` is gitignored. Alternatively pass it at run time:

```bash
ansible-playbook deploy_sonarqube.yml -i inventory/hosts.yml -k -K -e sonarqube_db_password=...
```

Without it the play fails on the `Render .env template` task with an undefined-variable error, and never deploys a passwordless database.

### Run

```bash
ansible-playbook deploy_sonarqube.yml -i inventory/hosts.yml -k -K
```

`-k` prompts for the SSH password, `-K` for sudo. The play sets `become: true`, so every task runs as root.

### Variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `sonarqube_db_password` | none (required) | PostgreSQL password; set in `group_vars/services.yml` or via `-e` |
| `sonarqube_sonar_db_user` | `sonar` | PostgreSQL user |
| `sonarqube_sonar_db_name` | `sonarqube` | PostgreSQL database name |
| `sonarqube_sonar_web_port` | `9000` | Host port published for the SonarQube web UI |
| `sonarqube_sonar_stack_dir` | `/opt/sonarqube` | Directory holding `compose.yml` and `.env` on the target |

### Verify

```bash
ssh <user>@<host> 'sudo docker ps'                 # sonarqube_app and sonarqube_db (healthy)
ssh <user>@<host> 'sudo ls -l /opt/sonarqube/'     # compose.yml 0644, .env 0640
curl -s -o /dev/null -w "%{http_code}\n" http://<host>:9000   # expect 200
```

Then open `http://<host>:9000` and log in with `admin`/`admin`. The login page rendering is the proof that SonarQube reached its database: it refuses to start without one.

### Idempotency

Re-running the playbook is safe and reports honestly. `file`, `copy`, and `template` compare checksums before writing, and the deploy task is guarded: `docker compose up -d` runs only when the Compose file or the `.env` changed, or when either container is not running. A second consecutive run ends with `changed=0`.

### First run on a shared server

If a SonarQube stack was already deployed by hand on the target, bring it down before the first role run. `container_name` is fixed (`sonarqube_app`, `sonarqube_db`) and Docker refuses duplicate names, so the deploy fails; port 9000 would collide as well. A stack started from a different directory also uses different volume names, so the role's stack starts with an empty database and existing projects, tokens, and quality gates do not carry over.

```bash
ssh <user>@<host> 'cd <old-stack-dir> && sudo docker compose down'   # volumes stay on disk
```

### Troubleshooting: SonarQube crash-loops on first boot

If `sonarqube_app` restarts in a loop and its log shows Elasticsearch bootstrap errors, check the kernel setting on the host:

```bash
ssh <user>@<host> 'sysctl -n vm.max_map_count'   # the sonarqube image requires 524288
ssh <user>@<host> 'sudo sysctl -w vm.max_map_count=524288'
```

Then rerun the playbook. This is deliberately not automated in the role. WSL2 kernels commonly default to `65530`, so check it first when testing against a WSL target.
