# Jenkins Node Offline - Disk Space Issue

If your Jenkins node goes offline and shows a warning about low disk space, you can check which files are consuming space in the `/var/lib/jenkins` directory.

## Steps

1. Run the following command to identify the top space-consuming files and directories:

```bash
   sudo du -h /var/lib/jenkins | sort -h | tail -n 20
```

# Moving Jenkins Home to a Larger Disk Using a Symlink

## Scenario
- **Current Jenkins home:** `/var/lib/jenkins`  
- **Current mount point:** `/dev/mapper/ubuntu--vg-ubuntu--lv` (18 GB)  
- **Available larger disk:** `/dev/vg-data/lv-data` mounted on `/Data` (~49 GB free)  

We will move Jenkins data to `/Data/jenkins` and create a **symbolic link** from `/var/lib/jenkins` to the new location.  
This allows Jenkins to use more disk space without changing its configuration.

---

## Steps

### 1. Stop Jenkins Service
```bash
sudo systemctl stop jenkins
```

### 2. Create Jenkins Directory on the New Disk
```
sudo mkdir -p /Data/jenkins
```

### 3. Copy Existing Jenkins Data
Use rsync to preserve permissions, symlinks, and timestamps:
```
sudo rsync -av /var/lib/jenkins/ /Data/jenkins/
```

### 4. Backup Original Jenkins Directory
```
sudo mv /var/lib/jenkins /var/lib/jenkins.bak
```

### 5. Create a Symlink
Point /var/lib/jenkins to /Data/jenkins:
```
sudo ln -s /Data/jenkins /var/lib/jenkins
```

### 6. Fix Permissions
Ensure the Jenkins user owns the new directory:
```
sudo chown -R jenkins:jenkins /Data/jenkins
```

### 7. Start Jenkins Service
```
sudo systemctl start jenkins
```

### 8. Verify
Check the symlink:
```
ls -ld /var/lib/jenkins
```

Expected output:
```
lrwxrwxrwx 1 root root ... /var/lib/jenkins -> /Data/jenkins
```

Check which filesystem is used:
```
df -h /var/lib/jenkins
```

It should show /Data with the larger capacity.

### 9. Cleanup (Optional)
After confirming Jenkins works as expected:
```
sudo rm -rf /var/lib/jenkins.bak
```

## Rollback Procedure

If something goes wrong:

### 1. Stop Jenkins:
```
sudo systemctl stop jenkins
```

### 2. Remove the symlink:
```
sudo rm /var/lib/jenkins
```

### 3. Restore the backup:
```
sudo mv /var/lib/jenkins.bak /var/lib/jenkins
```

### 4. Start Jenkins:
```
sudo systemctl start jenkins
```


## Advantages of This Method

- No changes to /etc/fstab

- Easy to roll back

- Jenkins still uses the same path (/var/lib/jenkins)

