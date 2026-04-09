# GitLab VM Migration: XenServer → VMware

## Overview
This project documents the full migration of a GitLab instance from an older XenServer environment to a VMware environment. The source GitLab server had been down for ~5 months before the migration, making this a complex process requiring careful planning, selective data transfer, and troubleshooting.

The goal was to migrate GitLab with all repositories, configuration, database, and web GUI fully functional while navigating storage limitations and network restrictions.

---

## Migration Challenges & Issues Encountered

1. **Network restrictions prevented simple copy operations**  
   - The source network was highly secured, so standard copy methods didn’t work.  
   - Solution: We used `scp` for secure file transfers between servers.

2. **Disk attachment and VM export issues**  
   - Attempted to attach the XenServer disk directly to VMware → failed due to older hypervisor versions and incompatibilities.  
   - Attempted to export VM image → failed due to insufficient storage (~12GB free, unable to expand).  
   - Solution: Focused on transferring only critical files and configs.

3. **Service and GUI recovery**  
   - GitLab web GUI was down for months. Needed database info, secret keys, configs, and repositories to restore it.  
   - Solution: Compiled all necessary files into a tarball, transferred them to VMware, and restored GitLab services.

4. **Repository transfer issues**  
   - Initial transfer of repositories to new VM resulted in incomplete/missing repos.  
   - Solution: Removed failed repos from VMware VM and re-transferred them from XenServer.

5. **Database migration and sanitization**  
   - Needed to sanitize commands and ensure proper database restoration.  
   - Solution: Used `gitlab-rake` and `gitlab-rails` production commands to reset and reconfigure the database.

---

## Migration Workflow (Step-by-Step)

### 1. Prepare source VM
```bash
# Stop GitLab services
gitlab-ctl stop

# Clean temporary files and verify backups
tar -cvzf gitlab_backup.tar.gz /etc/gitlab /var/opt/gitlab /var/opt/gitlab/backups
```

### 2. Mount disk for helper VM (if GitLab console is down)
```bash
# Mount GitLab disk to a temporary helper VM
mount /dev/sdx /mnt/gitlab_disk
```

### 3. Compile critical files
- Secret keys  
- GitLab configuration files  
- Repositories  
- Database dumps (sanitized)  

```bash
# Create tarball with all critical files
tar -cvzf gitlab_migration_bundle.tar.gz /mnt/gitlab_disk/etc/gitlab \
/mnt/gitlab_disk/var/opt/gitlab/backups \
/mnt/gitlab_disk/var/opt/gitlab/repositories
```

### 4. Transfer files securely to VMware
```bash
scp gitlab_migration_bundle.tar.gz user@vmware-server:/tmp
```

### 5. Extract and restore on VMware VM
```bash
ssh user@vmware-server
tar -xvzf /tmp/gitlab_migration_bundle.tar.gz -C /tmp/gitlab_restore
```

### 6. Restore GitLab services
```bash
# Reconfigure GitLab
gitlab-ctl reconfigure

# Reset and restore database
gitlab-rake gitlab:db:reset RAILS_ENV=production
gitlab-rake gitlab:db:setup RAILS_ENV=production

# Start GitLab
gitlab-ctl start

# Check status
gitlab-ctl status
```

### 7. Re-transfer repositories if needed
```bash
# If repositories were incomplete:
scp -r /mnt/gitlab_disk/var/opt/gitlab/repositories user@vmware-server:/var/opt/gitlab/repositories
```

### 8. Post-migration validation
- Access GitLab web GUI → confirm functionality  
- Test CI/CD pipelines → confirm operational  
- Verify repository integrity and backups

---

## Lessons Learned & Best Practices

- Always plan for **storage limitations** before attempting full VM exports.  
- **Secure, selective file transfers** are effective when network restrictions prevent normal copy methods.  
- Helper VMs can be a lifesaver when main VMs are down for extended periods.  
- Re-transfer repositories carefully—always validate integrity before final cutover.  
- Document every step, especially when working with legacy environments.  

---

## Security Note
- All hostnames, IPs, and sensitive paths have been sanitized for this public repository.  
- Commands are generalized to demonstrate process and skills without exposing real data.  

---

## Future Improvements
- Automate migrations with scripts for repeatability  
- Include detailed backup and rollback procedures  
- Add flow diagrams for VM and repository transfer processes
