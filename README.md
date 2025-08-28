# Ansible Playbook for Deploying Forgejo on Docker

This Ansible playbook deploys [Forgejo](https://forgejo.org/) on Docker with a PostgreSQL database. It supports various Linux distributions (e.g., Ubuntu, Debian, Fedora, CentOS) and includes web server detection to configure Apache, Nginx, or a reverse proxy setup with SSL support via Certbot.

## Prerequisites
- **Ansible**: Install on the control node (`pip install ansible` or via package manager).
- **SSH Access**: Ensure the target host is accessible via SSH with a user having `sudo` privileges.
- **DNS**: Configure a domain (e.g., `fj.yourdomain.com`) pointing to the target server's IP.
- **Docker**: The playbook installs Docker and Docker Compose if not already present.
- **Python**: Required for Ansible's Docker module (`pip install docker`).
- **Certbot**: The playbook installs Certbot for SSL certificate issuance.

## Directory Structure
```
ansibledeployforgejodocker/
├── README.md
├── hosts.ini
├── playbook.yml
├── roles/
│   ├── docker/
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   └── templates/
│   │       └── docker-compose.yml.j2
│   ├── forgejo/
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   ├── templates/
│   │   │   ├── app.ini.j2
│   │   │   ├── nginx.conf.j2
│   │   │   ├── fj-proxy.conf.j2
│   │   │   ├── fj.conf.j2
│   │   │   └── fj-le-ssl.conf.j2
│   │   └── handlers/
│   │       └── main.yml
```

## Setup Instructions
1. **Edit `hosts.ini`**:
   - Update the server details (`ansible_host`, `ansible_user`, `ansible_ssh_private_key_file`).
   - Set `domain_name` (e.g., `fj.yourdomain.com`).
   - Set a secure `db_password`.
   - Adjust `http_port`, `ssh_port`, `forgejo_uid`, `forgejo_gid`, and `forgejo_rootless` as needed.
   - Example:
     ```ini
     [forgejo_servers]
     your_server ansible_host=192.168.1.100 ansible_user=admin ansible_ssh_private_key_file=~/.ssh/id_rsa

     [forgejo_servers:vars]
     domain_name=fj.yourdomain.com
     forgejo_version=12
     db_type=postgres
     db_name=forgejo
     db_user=forgejo
     db_password=securepassword
     http_port=3000
     ssh_port=222
     forgejo_uid=1000
     forgejo_gid=1000
     forgejo_rootless=true
     ```

2. **Run the Playbook**:
   ```bash
   cd /home/toto/ansibledeployforgejodocker
   ansible-playbook -i hosts.ini playbook.yml
   ```

3. **Web Server Configuration**:
   - The playbook detects if Apache, Nginx, or both are installed:
     - **Apache**: Configures Forgejo with `fj.conf` (HTTP) and `fj-le-ssl.conf` (HTTPS) in `/etc/apache2/sites-available` (Debian) or `/etc/httpd/conf.d` (RedHat).
     - **Nginx**: Configures Forgejo with `/etc/nginx/conf.d/fj.conf` (HTTP/HTTPS).
     - **Both**: Applies both configurations, with Nginx as the primary reverse proxy.
     - **Neither**: Installs Nginx and Apache, with Apache on ports 8081 (HTTP) and 8083 (HTTPS), and Nginx as the reverse proxy in `/etc/nginx/conf.d/fj-proxy.conf`.
   - SSL is enabled via Certbot for Apache configurations (`fj-le-ssl.conf`).

4. **SSL Setup**:
   - The playbook installs Certbot and runs it to obtain an SSL certificate for `domain_name` using the webroot method.
   - After running the playbook, verify the SSL setup at `https://fj.yourdomain.com`.
   - Note: No Certbot renewal cron job is set up for subdomains. Manage renewals manually or via external tools.
   - Manual Certbot command (if needed):
     ```bash
     certbot certonly --webroot -w /var/lib/forgejo -d fj.yourdomain.com
     ```

5. **Access Forgejo**:
   - Open `https://fj.yourdomain.com` to complete the Forgejo onboarding process.
   - Test SSH: `ssh -F /dev/null git@<server_ip> -p 222`

## Notes
- **HTTPS**: The playbook configures SSL for Apache using Certbot. For Nginx HTTPS, manually update `nginx.conf.j2` or `fj-proxy.conf.j2` with SSL settings.
- **SELinux**: On SELinux-enabled systems (e.g., CentOS), check audit logs or set `setenforce 0` for testing.
- **Security**: Use Ansible Vault to encrypt `hosts.ini`:
  ```bash
  ansible-vault encrypt hosts.ini
  ansible-playbook -i hosts.ini playbook.yml --ask-vault-pass
  ```
- **.gitignore**: `hosts.ini` is ignored to prevent committing sensitive data.
- **Customization**:
  - Modify `roles/forgejo/templates/fj.conf.j2`, `fj-le-ssl.conf.j2`, `nginx.conf.j2`, or `fj-proxy.conf.j2` for custom configurations.
  - To use MySQL or SQLite, update `db_type` in `hosts.ini` and adjust `docker-compose.yml.j2`.

## Troubleshooting
- Check Docker logs: `docker logs forgejo` or `docker logs <postgres_container>`.
- Verify Nginx/Apache status: `systemctl status nginx` or `systemctl status apache2`.
- Check Certbot certificates: `certbot certificates`.
- Ensure DNS is set for `domain_name`.
