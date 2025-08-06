# Jenkins Node Offline - Disk Space Issue

If your Jenkins node goes offline and shows a warning about low disk space, you can check which files are consuming space in the `/var/lib/jenkins` directory.

## Steps

1. Run the following command to identify the top space-consuming files and directories:

   ```bash
   sudo du -h /var/lib/jenkins | sort -h | tail -n 20
