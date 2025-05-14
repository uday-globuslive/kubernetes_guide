# Docker Storage

Docker provides several options for storing data, each with different use cases, performance characteristics, and persistence mechanisms.

## Storage Drivers

Docker uses storage drivers to manage the contents of the image layers and the writable container layer. The specific storage driver used depends on your operating system and configuration:

- **overlay2**: The preferred storage driver for all Linux distributions that support it
- **aufs**: Used on Ubuntu and Debian systems (legacy)
- **devicemapper**: Used on CentOS and RHEL (legacy)
- **btrfs**: Available when using Btrfs as the filesystem
- **zfs**: Available when using ZFS as the filesystem
- **vfs**: Very slow but works on any filesystem

To check which storage driver you're using:

```bash
docker info | grep "Storage Driver"
```

## Container Filesystem

Each container has its own filesystem layers:

1. **Read-only image layers**: The layers that make up the base image
2. **Thin writable container layer**: Where all changes are stored

When a container is deleted, all data written to the container's writable layer is lost.

## Docker Volume Types

### Bind Mounts

Bind mounts map a host file or directory to a container file or directory.

```bash
docker run -v /host/path:/container/path nginx
```

Characteristics:
- Host location can be anywhere on the filesystem
- Performance depends on the host filesystem
- No Docker CLI commands for direct management

### Named Volumes

Named volumes are managed by Docker and stored in a part of the host filesystem that's managed by Docker.

```bash
# Create a volume
docker volume create my-data

# Use the volume
docker run -v my-data:/container/path nginx
```

Characteristics:
- Docker manages the storage location on the host
- Can be backed up, restored, and migrated more easily
- Can be managed using Docker CLI commands

```bash
docker volume ls
docker volume inspect my-data
docker volume rm my-data
```

### tmpfs Mounts

tmpfs mounts are stored in the host's memory only and never written to the host's filesystem.

```bash
docker run --tmpfs /container/path nginx
```

Characteristics:
- Useful for storing sensitive data 
- Data is lost when the container stops
- Can improve performance for non-persistent state data

## Volume Drivers

Docker supports volume drivers to enable volume storage on remote hosts or cloud providers:

- **local**: Default driver for local storage
- **azure**: For Azure File Storage
- **convoy**: For block storage
- **gce**: For Google Compute Engine persistent disks
- **flocker**: For multi-host portable volumes
- **portworx**: For distributed storage
- **vmware-vsphere**: For vSphere virtual storage

```bash
# Create a volume with a specific driver
docker volume create --driver aws-efs my-efs-volume
```

## Best Practices

1. **Use named volumes for persistence**: More portable and easier to manage than bind mounts

2. **Use volume for databases**: Store database data in volumes for persistence

   ```bash
   docker run -v mysql-data:/var/lib/mysql mysql
   ```

3. **Use read-only bind mounts for configuration**: When you need to provide configuration to a container

   ```bash
   docker run -v /host/config.json:/app/config.json:ro nginx
   ```

4. **Use tmpfs for sensitive information**: For secrets that shouldn't be persisted

   ```bash
   docker run --tmpfs /run/secrets nginx
   ```

5. **Backup volumes regularly**: Develop a consistent backup strategy

   ```bash
   docker run --rm -v my-volume:/source -v $(pwd):/backup alpine tar -czvf /backup/my-volume.tar.gz -C /source .
   ```

6. **Consider volume plugins for production**: For cloud native deployments, use appropriate cloud storage drivers

## Docker Compose with Volumes

Docker Compose makes it easy to define and share volumes between services:

```yaml
version: '3'
services:
  db:
    image: postgres
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d:ro
  
  web:
    image: nginx
    volumes:
      - ./website:/usr/share/nginx/html:ro
      - ./nginx.conf:/etc/nginx/nginx.conf:ro

volumes:
  postgres-data:
```

## Cleanup and Maintenance

Regular maintenance of volumes is important:

```bash
# Remove unused volumes
docker volume prune

# Remove specific volume
docker volume rm my-volume

# Remove all unused Docker objects (including volumes)
docker system prune --volumes
```

## Troubleshooting Docker Storage

1. **Storage space issues**: 
   - Check available space: `df -h /var/lib/docker`
   - Identify large volumes: `docker system df -v`

2. **Permission problems**:
   - Check volume ownership: `ls -la $(docker volume inspect my-volume --format '{{ .Mountpoint }}')`
   - Run container with appropriate user or adjust permissions

3. **Performance issues**:
   - Consider using volume drivers optimized for performance
   - Monitor I/O performance: `docker stats`

4. **Data not persisting**:
   - Verify volume is correctly mounted
   - Check if you're using the correct paths inside the container

By understanding Docker's storage options and applying these best practices, you can ensure your containerized applications maintain data persistence, security, and performance.