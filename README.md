`beegfs-metadata-recovery` recovers files from a broken beegfs filesystem's metadata.

## Recovering files

1. On each metadata server (`mercury???`), extract metadata to shared disk:
    ```
    tar -C /beegfs --xattrs -zcf /opt/ppd/scratch/data2_recovery/mercury???/meta.tgz meta
    ```

2. On each storage server (mercury???), extract storage to shared disk:
    ```
    cp -a /beegfs/storage/chunks /opt/ppd/scratch/data2_recovery/mercury???/storage/chunks/
    ```

3. Extract the metadata files to a local disk, so they keep their xattr:
    ```
    cd /scratch/adye/data2_recovery
    mkdir $(cd /opt/ppd/scratch/data2_recovery; ls -1d mercury???)
    for d in mercury???; do cp -pi /opt/ppd/scratch/data2_recovery/$d/meta.tgz $d/; done
    for d in mercury???; do (set -x; cd $d; tar --xattrs -zxf meta.tgz); done
    ```
    * Creates directories:
        ```
        mercury???/meta/dentries
        mercury???/meta/buddymir/dentries
        mercury???/meta/inodes
        mercury???/meta/buddymir/inodes
        ```

4. Create `stor.txt` with the list of storage chunks:
    ```
    cd /opt/ppd/scratch/data2_recovery
    find mercury???/storage/chunks -type f > /scratch/adye/data2_recovery/stor.txt
    ```

5. Create `meta.txt` with all the metadata from the `meta` directories:
    ```
    cd /scratch/adye/data2_recovery
    beegfs-metadata-recovery -w -o /scratch/adye/data2_recovery/meta.txt /scratch/adye/data2_recovery
    ```
    * With `-x` checks xattr metadata matches between different server's instances of the same data

6. Create `copy.sh`:
    ```
    cd /mercury/data2/restore
    beegfs-metadata-recovery -o copy.sh -D restore -G /opt/ppd/scratch/data2_recovery /scratch/adye/data2_recovery
    ```
    * With `-O` includes chown commands, which requires root.
    * With `-S` checks storage file sizes (slower).
    * With `-x` checks xattr metadata matches between dentries and file inodes (more RAM?)

7. Run copy.sh:
    ```
    ./copy.sh
    ```

## Notes

* This script requires Python 3.6 or greater, as is default on CentOS7.

* The final script (`copy.sh` in this example) uses the `beegfs-merge-chunks` script,
executed from the same directory as this `beegfs-metadata-recovery`.

* If recovering a different filesystem from what I used, check the `server_decode`
and (just to supress some error messages) `topdirs` in `beegfs-metadata-recovery`.
