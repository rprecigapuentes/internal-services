# internal-services

Ansible roles that deploy the team's internal services with Docker Compose. 

## sonarqube

Deploys SonarQube Community Edition plus its PostgreSQL database as a Compose stack in `/opt/sonarqube`. Every task uses `ansible.builtin` modules only.

### Set the database password

`sonarqube_db_password` has no default on purpose. Copy the example file and set the real value:

```bash
cp group_vars/services.yml.example group_vars/services.yml
```

### Run

```bash
ansible-playbook deploy_sonarqube.yml -i inventory/hosts.yml -k -K
```

`-k` prompts for the SSH password, `-K` for sudo. 

### Variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `sonarqube_db_password` | none (required) | PostgreSQL password; set in `group_vars/services.yml` |
| `sonarqube_sonar_db_user` | `sonar` | PostgreSQL user |
| `sonarqube_sonar_db_name` | `sonarqube` | PostgreSQL database name |
| `sonarqube_sonar_web_port` | `9000` | Host port published for the SonarQube web UI |
| `sonarqube_sonar_stack_dir` | `/opt/sonarqube` | Directory holding `compose.yml` and `.env` on the target |