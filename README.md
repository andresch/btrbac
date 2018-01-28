Btrbac (alpha)
=====

Dockerized backups for [Btrfs](https://btrfs.wiki.kernel.org/index.php/Main_Page) subvolumes

What is Btrbac?
-----

Btrbac is a dockerized backup script for Btrfs subvolumes. It provides protection on three levels:

1) __Btrfs snapshots__
    
    It creates readonly snapshots of a subvolume. Those
	allow you access previous versions of the subvolume straight from the file system.

2) __Btrfs backups on local disks__
	
	In case you loose your Btrfs snapshots or filesystem for whatever reason (filesystem corruption, accidential formatting or deletion of snapshots) you'll have incremental backups based on [brtfs send](https://btrfs.wiki.kernel.org/index.php/Manpage/btrfs-send). If stored on other (potentially non-btrfs) filesystems, you can easily reconstruct all snapshots via [brtfs receive](https://btrfs.wiki.kernel.org/index.php/Manpage/btrfs-receive) once you have again a working Btrfs filesystem

3) __Backups on remote storage__
	
	To prevent you from data loss in case all disks in your machine get damaged, Btrbac allows you to transfer all your backups to a remote storage. This functionaly is build upon [rclone](https://rclone.org/) and by that supports out of the box a huge variety of storage options and protocols.

Usage
-----

You get all three levels of protection for a given subvolume with a single command:

	docker run --rm \
	-v <path to your subvolume>:/subvolume \
	-v <path to your backup directory>:/backup \
	-v <path to your rclone configuration>:/rclone.conf \ 
	--privileged \
	andresch/btrbac \
	-p <prefix for all artifacts>
	-r <name of your rconf remote>

As result you will find

- a snapshot of your subvolume in `<subvolume>/.snapshots/<prefix>-snapshot-<timestamp>`
- a backup (either incremental or full) of the subvolume in directory `<backup-dir>/<prefix>-backup-<timestamp>-<type>`
- a copy of the backup directory on the remote storage

### Btrbac Mountpoints

Btrbac uses three docker mountpoints which you bind to absolute paths on your machine:

* `/subvolume`
	
	The path of the Btrfs subvolume you want to backup

* `/backup`
	
	The directory where you want to store your backups.
    To prevent data loss in case of filesystem corruption or hardware failure this directory should reside on a different disk or at least outside of the Btrfs filesystem that contains the subvolume.

* `/rclone.conf`

    The file that contains the configurations for the rclone remotes. Only required when using option `-r`

### Btrbac Options

You have a few options to influence Brtbac's behaviour

* `-p <prefix>`
    A prefix that is used for the names of snapshots and backup archives.
    **Default:** `subvolume`

* `-x <max-backup-file-size>`
    Allows to split the backup stream into chunks of a given maximal size. This might be helpful when the upload to the remote storage goes over slow or unreliable connections. Btrbac retries the upload of missing files even after connectity issues. Having split the backup into multiple smaller files reduces the amount of data that has to be retransmitted. Accepts all values of [split's -b option](http://man7.org/linux/man-pages/man1/split.1.html)

* `-n <snapshot-subvolume>`
    Path of a subvolume where to place snapshots. The path must be relative to <subvolume> and belong
    to the same btrfs filesystem. When necessary the subvolume will get created.
    **Default:** `.snapshots`

* `-r <rclone-remote>`
    The name of a remote location that is configured in the rclone.conf
    **Warning!** When missing the backup will not get copied to any remote storage

* `-f`
    Forces a full backup, even if incremental backup would be possible

* `-d` 
    Enables debug messages

* `-h`
    Prints usage information

Some options of the Btrbac script are implicitly set to the mount points of the docker environment and usually not not used in combination with `docker run`:

* `-s <subvolume>`
    Path of the subvolume to backup
	Always set to docker mount point `/subvolume` 
* `-b <backup-dir>`
    Path where to store the backup stream
	Always set to docker mount point `/backup`
    **Note!** If all you want are snapshots (better think twice!) you can disable backup generation by adding `-b ""` to the docker command
* `-c <rclone.conf>`
    Path to rclone config files as expected by rclone --config
	Always set to docker mount point `/rclone.conf`

### Preparing [rclone](https://rclone.org/)

[rclone](https://rclone.org/) requires a configuration file where it stores all necessary information to access the remote storage. This configuration file gets created with the [rclone config](https://rclone.org/commands/rclone_config/) command.

You find the location of the configuration file with  the help of `rclone --help` where you search for the default value of the option `--config`. Then bind the absolute path of that file to the docker mount point `/rclone.conf`.

Depending on the remote storage system you intent to use, `rclone` might open a browser that allows you to log into the storage system and grant rclone access. If you only have terminal access to the machine where you want to use Btrbac, please check [rclone's remote setup options](https://rclone.org/remote_setup/)

Status
-----

Btrbac is still in early stage. While already running on the maintainer's machine to protect against dataloss, it still lacks [some important features](https://github.com/andresch/btrbac/issues?q=is%3Aissue+is%3Aopen+label%3Aenhancement) before it can leave alpha stage.

At the same time the underlying tools (esp. btrfs and rclone) that Btrbac just wraps nicely are production grade and usage of Btrbac, even in this early stage, should already give you decent protection from data loss.

