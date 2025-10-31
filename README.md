# Multimedia Home Lab Suite

An Ansible-powered home lab deployer that sets up a complete multimedia environment using Docker containers. Features include media streaming (Plex), download automation (Sonarr, Radarr, etc.), and media editing tools, all configured with secure access and SMB/CIFS (NAS) storage integration. This multimedia suite extends the functionality of the Home Lab deployment available at https://github.com/cyph3ramfm/home_lab_setup

## Features

- **Media Streaming**: Plex Media Server with hardware transcoding support
- **Download Automation**:
  - Sonarr (TV series)
  - Radarr (movies)
  - Bazarr (subtitles)
  - Jackett (torrent indexer)
  - Transmission (download client)
  - Gluetun (VPN container)
- **Media Editing**:
  - Handbrake (video transcoding)
  - GIMP (image editing)
- **Dashboard**: Homepage for service monitoring and quick access
- **Storage**: SMB/CIFS (NAS) integration for media and container persistence

## Prerequisites

- SMB-capable NAS accessible from the target host(s) (shares must be exported and reachable)
- [Home Lab and pre-requisites deployed](https://github.com/cyph3ramfm/home_lab_setup)

## Quick Start

1. **Clone the repository**:
   ```bash
   git clone <repository-url>
   cd multimedia_suite
   ```

2. **Set up secrets**:
   ```bash
   # Create a new vault password file (only once)
   echo "your-secure-password" > ~/.vault_pass.txt
   chmod 600 ~/.vault_pass.txt
   
   # Create vault.yml from template
   cp group_vars/vault.yml.template group_vars/vault.yml
   ansible-vault encrypt group_vars/vault.yml --vault-password-file ~/.vault_pass.txt
   
   # Edit encrypted vault.yml
   ansible-vault edit group_vars/vault.yml --vault-password-file ~/.vault_pass.txt
   ```

3. **Configure settings**:
   - Edit `group_vars/main.yml` to set NAS/SMB paths (e.g. `nas_server`, `smb_multimedia`, `mount_point_multimedia`), domain prefixes, and enable/disable features
   - Update `inventory/hosts` with your target host(s)

4. **Deploy the stack**:
   ```bash
    # Deploy with vault password and sudo privileges
    ansible-playbook deploy_multimedia_suite_playbook.yml -i inventory/hosts \\
       --vault-password-file ~/.vault_pass.txt --ask-become-pass
   
    # For testing/preview (shows what would change)
    ansible-playbook deploy_multimedia_suite_playbook.yml -i inventory/hosts \\
       --vault-password-file ~/.vault_pass.txt --ask-become-pass --check
   
    # Alternative: prompt for vault password (and become password)   
    ansible-playbook -i inventory/hosts \
      --ask-vault-pass \
      --ask-become \
      deploy_multimedia_suite_playbook.yml
   ```

## Managing Secrets with ansible-vault

This project uses `ansible-vault` to securely store sensitive information in `group_vars/vault.yml`. The template file `vault.yml.template` shows which variables need to be set.

### Working with vault.yml

1. **Creating the vault file**:
   ```bash
   # Copy template
   cp group_vars/vault.yml.template group_vars/vault.yml
   
   # Encrypt the file
   ansible-vault encrypt group_vars/vault.yml
   ```

2. **Editing encrypted values**:
   ```bash
   # Edit directly
   ansible-vault edit group_vars/vault.yml
   
   # View contents
   ansible-vault view group_vars/vault.yml
   ```

3. **Required secret values**:
   - `vault_device_name`: Unique name for this deployment
   - `vault_domain`: Base domain for service URLs
   - `vault_gluetun_wireguard_key`: If using WireGuard VPN
   - `vault_plex_claim_token`: For Plex server registration

### Security Best Practices

#### Vault Management
- Never commit `vault.yml` to version control (included in .gitignore)
- Keep vault password in a protected file (e.g., `~/.vault_pass.txt` with 600 permissions)
- Update `vault.yml.template` when adding new secret variables
- Use descriptive comments in template for required formats/sources

#### Container Security
- Set custom PUID/PGID values in service configurations (default: 1000:1000)
- Change default credentials for all services (e.g., GIMP, Transmission)
- Use strong passwords for all service admin interfaces
- Consider implementing additional authentication for exposed services

#### Network Security
- Use hostnames instead of IP addresses where possible
- Implement proper firewall rules for exposed ports
- Use HTTPS with valid certificates (configured through Traefik)
- Restrict access to admin interfaces with IP whitelisting
- Configure VPN (Gluetun) with strong authentication

## Troubleshooting

### Common Issues

1. **SMB/CIFS (NAS) Mount Failures**
    ```
    Error: Failed to mount SMB/CIFS share
    ```
    - Check the NAS is reachable (e.g. `ping <nas_server>`)
    - List available SMB shares using `smbclient -L //<nas_server>` (install `smbclient` if missing)
    - Verify the SMB share path exists on the NAS and the account you plan to use has appropriate permissions
    - Ensure the host has CIFS utilities installed: `apt install cifs-utils` (Ubuntu/Debian)
    - Example `/etc/fstab` entry (use a credentials file, do NOT store passwords directly in fstab):

       ```text
       //nas_server/multimedia /mnt/multimedia cifs credentials=/root/.smbcredentials,iocharset=utf8,vers=3.0 0 0
       ```

    - Create a credentials file `/root/.smbcredentials` with:

       ```text
       username=your_username
       password=your_password
       domain=YOUR_DOMAIN  # optional
       ```

       Set restrictive permissions on the credentials file: `chmod 600 /root/.smbcredentials`.

2. **Docker Network Issues**
   ```
   Error: network <name> not found
   ```
   - Create missing networks manually:
     ```bash
     docker network create home_lab
     docker network create proxy
     ```

3. **Container Permission Errors**
   ```
   Error: permission denied while trying to connect to the Docker daemon socket
   ```
   - Add user to docker group: `usermod -aG docker $USER`
   - Re-login or run: `newgrp docker`

4. **Vault Access Issues**
   ```
   ERROR! Attempting to decrypt but no vault secrets found
   ```
   - Ensure vault password file exists and is readable
   - Check if vault.yml is encrypted: should start with `$ANSIBLE_VAULT`
   - Try re-encrypting: `ansible-vault encrypt group_vars/vault.yml`

5. **Redeployment Issues**
   ```
   ERROR! Running playbook doesn't update existing containers
   ```
   - Please ensure that all linked containers have been stopped and removed.
   - e.g. To re-deploy any container inside the multimedia_download role, ensure that gluetun, bazarr, jackett, radarr, sonarr, transmission, lidarr, readarr and mylar3 have been stopped and removed.

### Debug Mode

Enable debug_mode in `group_vars/main.yml` to:
- Preview rendered templates
- See detailed error messages
- Inspect variable values

```yaml
debug_mode: true  # in group_vars/main.yml
```

### Health Checks

1. **Check SMB Mounts**:
   ```bash
   # Check mounted filesystems for configured mount points
   df -h | grep -E "$(grep mount_point_ group_vars/main.yml | cut -d: -f2)"
   # List SMB shares available on the NAS (may require credentials)
   smbclient -L //<your-nas-host> -N || true
   ```

2. **Verify Docker Networks**:
   ```bash
   docker network ls | grep -E 'home_lab|proxy'
   ```

3. **Container Status**:
   ```bash
   docker ps --format 'table {{.Names}}\t{{.Status}}'
   ```

### Getting Help

1. Check container logs:
   ```bash
   docker logs <container_name>
   ```

2. Review Ansible output with increased verbosity:
   ```bash
   ansible-playbook deploy_multimedia_suite_playbook.yml -vvv
   ```

3. Verify service connectivity:
   ```bash
   curl -I http://<service>.<your-domain>
   ```

## Role Dependencies

- `smb_multimedia`: Required by all media-related roles
-- `multimedia_download`: Depends on working SMB/CIFS mounts
-- `multimedia_streaming`: Expects media folders from SMB/CIFS mounts
-- `multimedia_editing`: Uses SMB/CIFS mounts for input/output
- `dashboards`: Independent but enhanced by other services

## Contributing

1. Fork the repository
2. Create your feature branch: `git checkout -b feature/my-feature`
3. Follow existing patterns for:
   - Role organization
   - Variable naming
   - Template structure
4. Update `vault.yml.template` if adding secrets
5. Submit a Pull Request
