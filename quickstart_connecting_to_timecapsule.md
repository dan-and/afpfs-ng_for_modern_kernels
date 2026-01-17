# Quickstart: Connecting to Time Capsule with afpfs-ng

This guide shows how to compile and use afpfs-ng to connect to Apple Time Capsules on Linux.

## Required Packages

Install the build dependencies:

```bash
sudo apt-get update
sudo apt-get install build-essential libfuse-dev libreadline-dev libncurses-dev libgcrypt20-dev pkg-config
```

**Important**: `libgcrypt20-dev` is required for secure authentication methods (DHX2) that Time Capsules need.

## Compilation and Installation

```bash
# Clone or download afpfs-ng source
cd afpfs-ng/

# Configure with libgcrypt support
./configure

# Build
make

# Install
sudo make install

# Update library cache
sudo ldconfig
```

Verify installation:
```bash
afp_client status
```

Expected output shows secure UAMs:
```
AFPFS Version: 0.8.2
UAMs compiled in: Cleartxt Passwrd, No User Authent, Randnum Exchange, 2-Way Randnum Exchange, DHCAST128, DHX2
```

## Mounting Examples

### Basic Mount (DHX2 Authentication)

```bash
# Create mount point
mkdir -p /mnt/timecapsule
sudo chown $USER:$USER /mnt/timecapsule

# Mount Time Capsule volume
mount_afp "afp://username;AUTH=DHX2:password@timecapsule/TimeCapsule" /mnt/timecapsule

# Access files
ls /mnt/timecapsule
```

### Mount with Username Containing Spaces

```bash
# Use quotes around the entire URL
mount_afp "afp://John Doe;AUTH=DHX2:mypassword@timecapsule/TimeCapsule" /mnt/timecapsule
```

### Mount with Volume Name Containing Spaces

```bash
# Quote the entire URL when volume names have spaces
mount_afp "afp://username;AUTH=DHX2:password@timecapsule/My Shared Drive" /mnt/shared
```

### Mount Guest Volume (No Authentication)

```bash
# For public/guest access volumes
mount_afp "afp://;AUTH=No User Authent@timecapsule/Public" /mnt/public
```

## Backup and Data Transfer

### Complete Time Capsule Backup

Use rsync for efficient backups that preserve all file attributes and handle sparse files:

```bash
# Create backup directory
mkdir -p /path/to/backup

# Full backup preserving permissions, timestamps, and sparseness
rsync -avh --progress --delete --sparse --timeout=60 \
      --ignore-errors -E \
      /mnt/timecapsule/ \
      /path/to/backup/
```

### rsync Options Explained

- **`-a`** (archive): Preserves permissions, ownership, timestamps
- **`--sparse`**: Efficiently handles sparse files (important for Time Capsule disk images)
- **`--progress`**: Shows transfer progress for large files
- **`--delete`**: Removes files from backup that no longer exist on Time Capsule
- **`--timeout=60`**: Network timeout to handle connection issues
- **`-E`**: Preserves executability

### Sparse Files on Time Capsules

Time Capsules often contain sparse files (disk images with efficient zero storage). The `--sparse` option:
- Detects zero blocks during transfer
- Creates space-efficient sparse files on destination
- Saves bandwidth and storage space
- Essential for `.dmg`, `.sparseimage`, and VM disk files

### Backup Verification

```bash
# Compare file counts
find /mnt/timecapsule/ -type f | wc -l
find /path/to/backup/ -type f | wc -l

# Compare total sizes
du -sh /mnt/timecapsule/
du -sh /path/to/backup/

# Check sparse file efficiency
ls -lh /mnt/timecapsule/large-file.dmg  # Apparent size
du -h /mnt/timecapsule/large-file.dmg   # Actual disk usage
```

## Troubleshooting

### Permission Denied
If you get permission errors, mount as your regular user (not sudo):
```bash
sudo chown $USER:$USER /mnt/timecapsule
mount_afp "afp://user;AUTH=DHX2:pass@timecapsule/volume" /mnt/timecapsule
```

### UAM Not Available
If `afp_client status` doesn't show DHX2, reinstall with libgcrypt:
```bash
sudo apt-get install libgcrypt20-dev
cd afpfs-ng/
make clean
./configure
make
sudo make install
sudo ldconfig
```

### Connection Issues
- Verify Time Capsule hostname: `ping timecapsule`
- Check volume names on Time Capsule
- Ensure Time Capsule allows AFP connections

### Daemon Management
The afpfsd daemon runs per-user and is generally safe with multiple users:

**Check running daemons:**
```bash
ps aux | grep afpfsd
```

**Kill daemon if needed (replace PID with actual process ID):**
```bash
kill PID
# Or kill all instances
pkill afpfsd
```

**Safety notes:**
- Each user gets their own daemon instance (isolated by UID)
- Multiple daemons are safe but consume memory (~5-10MB each)
- Clean up stale socket files: `find /tmp -name "afp_server-*" -mtime +1 -delete`
- High user counts (>10) may impact system performance

## Notes

- Always use quotes around URLs containing spaces
- DHX2 is the recommended authentication method for modern Time Capsules
- Mount points must be owned by the mounting user
- The afpfsd daemon starts automatically when needed
