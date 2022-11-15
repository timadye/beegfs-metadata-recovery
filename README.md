`beegfs-metadata-recovery` recovers files from a broken BeeGFS filesystem's metadata.
## Recovering files

This procedure was used to recover all 8TB of data from a 32TB BeeGFS filesystem. The system was configured with 6 storage servers (called `mercury006` to `mercury011`) with replicated metadata distributed on the same servers.

We developed these scripts from examination of the BeeGFS filesystem metadata of the particular system we needed to recover. We were not aware of any documentation for the BeeGFS metadata structures, so we may not cover all cases. The terminology used here probably isn't correct. We have not tested the procedure on any other system.

* One issue that is bound to come up with another system is the codes used to identify the order file data is written to different servers. The codes are stored in the file's inode xattr. We don't know how the codes are assigned to each server. You will have to change `server_decode` dict in `beegfs-metadata-recovery` accordingly. This maps the codes stored in the xattr data to the server name (actually the name of the subdirectory where each server's file data is stored).

Other possible issues:

* The program assumes that the top-level directory tag has the special name `root`, and is its own parent. If a different tag name is used, or if the top-level directory is missing, then files can't be placed in the directory tree. This could maybe be fixed by using a different directory as the root, but that would need a (small) change to the code.

* We workded out the format of the xattr blocks by guesswork, trial, and error. There were a number of different formats in the filesystem being examined. Fortunately, each format could be uniquely determined by the length of the xattr block. However, there may be other block formats than we observed on our system.

    The offsets of each part of the xattr block are defined in the `xattr_offset()` and `xattr_offset_inode()` routines. These offsets are mostly relative to the smallest block (not a very easy design, but not worth fixing now).

    The script carefully checks the data it uses, so you will see an error (probably many many errors) if the data format is not as expected:
    * "`file `F` bad tag`" lists unexpected format tags. These are usually of the form `XXX-XXXXXXXX-X`, where the `X`s are hexadecimal digits (`0`-`9`, `A`-`F`). There can be more or fewer digits in each part of the tag. Apart from a few special tags (most notably "`root`") listed in the `topdirs` variable, all tags have this form. If you see other forms (except maybe for a few others that could be added to `topdirs` to supress the warning), you probably have an unexpected xattr block format. The tags are required to make the connection between a directory and its parent, and between a directory entry and the file storage, so it is vital that the tags are extracted correctly.
    * "`file `F` bad server codes`" lists the server codes that are not recognised in `server_decode`. Maybe those need to be updated (see above). However, if there are many different code specified, then it probably means the codes are not where they are expected in the xattr block and something else is being read out instead.
    * "`file `F` bad mode`" indicates a bad file mode. This may not be too serious, as it means the permissions can't be set correctly with `chmod`. Symlinks won't be created as symlinks.
    * "`file `F` bad UID:GID`" indicates user/group ids that aren't in `/etc/passwd` or `/etc/groups`. Maybe that's normal, or it could indicate an unknown format xattr.
    * There are other consistency checks that could indicate a problem, either with the filesystem integrity or with the xattr block format. The `-x` option enables some additional checks. The `-S` option also checks the file sizes are as expected, but this will slow down the script. We observed a few files with slightly different sizes, probably due to incompletely updated metadata.
## Procedure

Here is the procedure we used.

1. On each metadata server (`mercury???`), extract metadata to shared disk:
    ```
    tar -C /beegfs --xattrs -zcf /opt/ppd/scratch/data2_recovery/mercury???/meta.tgz meta
    ```

2. On each storage server (`mercury???`), extract storage to shared disk:
    ```
    cp -a /beegfs/storage/chunks /opt/ppd/scratch/data2_recovery/mercury???/storage/chunks/
    ```

3. Extract the metadata files to a local disk - or anything that can keep xattr:
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
    * The final script (`copy.sh` in this example) uses the `beegfs-merge-chunks` script, executed from the same directory as this `beegfs-metadata-recovery`.
## Notes

* This script requires Python 3.6 or greater, as is default on CentOS7.

* The script is cluttered with lots of code and options that were useful while we investigated how to recover the data, but now are less useful. Still, if you need to make investigations of your own, the `-d` (list dirs), `-f` (list files), `-c` (list storage chunks), `-H` show hex dump of xattr block, and other options may be helpful.