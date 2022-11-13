`beegfs-metadata-recovery` recovers files from a broken beegfs filesystem's metadata.

## Recovering files

```
# Start with:
#     /opt/ppd/scratch/data2_recovery/mercury???/meta.tgz
#     /opt/ppd/scratch/data2_recovery/mercury???/storage/chunks/...

# Extract the metadata files to a local disk, so they keep their xattr:
cd /scratch/adye/data2_recovery
mkdir $(cd /opt/ppd/scratch/data2_recovery; ls -1d mercury???)
for d in mercury???; do cp -pi /opt/ppd/scratch/data2_recovery/$d/meta.tgz $d/; done
for d in mercury???; do (set -x; cd $d; tar --xattrs -zxf meta.tgz); done

# Creates directories:
#     mercury00?/meta/dentries
#     mercury00?/meta/buddymir/dentries
#     mercury00?/meta/inodes
#     mercury00?/meta/buddymir/inodes

# Create stor.txt with the list of storage chunks:
cd /opt/ppd/scratch/data2_recovery
find mercury???/storage/chunks -type f > /scratch/adye/data2_recovery/stor.txt

# Create meta.txt with all the metadata from the meta directories:
cd /scratch/adye/data2_recovery
beegfs-metadata-recovery -w -o /scratch/adye/data2_recovery/meta.txt /scratch/adye/data2_recovery
# With -x checks xattr metadata matches between different server's instances of the same data

# Create copy.sh:
cd /mercury/data2/restore
beegfs-metadata-recovery -o copy.sh -D restore -G /opt/ppd/scratch/data2_recovery /scratch/adye/data2_recovery
# With -O includes chown commands, which requires root.
# With -S checks storage file sizes (slower).
# With -x checks xattr metadata matches between dentries and file inodes (more RAM?)

# Run copy.sh:
./copy.sh
```

## Notes

* This script requires Python 3.6 or greater, as is default on CentOS7.

* The final script (`copy.sh` in this example) uses the `beegfs-merge-chunks` script,
executed from the same directory as this `beegfs-metadata-recovery`.

* If recovering a different filesystem from what I used, check the `server_decode`
and (just to supress some error messages) `topdirs` in `beegfs-metadata-recovery`.
