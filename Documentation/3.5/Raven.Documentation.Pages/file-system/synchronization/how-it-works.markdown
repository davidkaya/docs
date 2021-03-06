#How file synchronization works?

A synchronization of a single file is a multi-step process. The synchronization starts when a file is modified and it is considered to be completed, that is when a remote file system confirms that the operation processed successfully. The synchronization is initialized by the file system where a file is changed - *source*. 
It pushes all data needed to synchronize the file to the remote file system called *destination*. One source file system can have multiple destinations. 

##Destinations

The list of the destination servers is kept as [the configuration item](../configurations) under the `Raven/Synchronization/Destinations` key. Adding a new destination file system
will automatically start to synchronize files there. If you want to stop syncing to some file system you can just drop it from the list or set `Enabled`
property to `false`. The easiest way to manipulate the destination servers is to use the studio.

{CODE-BLOCK:json}
{
    "Destinations": 
		[
			{
				"ServerUrl": "http://localhost:8080",
				"Username": null,
				"Password": null,
				"Domain": null,
				"ApiKey": null,
				"FileSystem": "Northwind",
				"Enabled": true
			}
		]
}
{CODE-BLOCK/}

##Available topologies

If you setup a file system to synchronize files in one direction to another Raven file system, they will work in the *master/slave* model.
Any changes made int the master will be propagated to the destination, but changes made on the slave server will not be reflected to the master.

RavenFS also supports the *master/master* synchronization model. Any file modifications on both file systems will be propagated between them.
Be aware that such configuration can result in conflicts if a file with a given name exists in both systems but has different versions ([read more about conflicts](./conflicts)).

##Synchronization process pipeline

Any file modification triggers a synchronization task. It goes through a list of active destinations and performs the following actions
for each of them:

1. It queries a remote file system for the `Etag` of the last synchronized file.
2. For each file that changed since last synchronization (where file `Etag` > `Etag` returned in (1)):
 * retrieve files metadata from the destination fs,
 * determine what kind of synchronization work has to be done based on the metadata that was just downloaded (see [Synchronization types](synchronization-types)),
 * add an appropriate synchronization work item to a pending synchronization queue.
3. It runs as many file synchronizations as it can concurrently (according to [`MaxNumberOfSynchronizationsPerDestination`](./configurations#ravensynchronizationconfig) setting).

{INFO: Synchronization types}
Depending on the determined synchronization work type, different data will be sent to a destination server to synchronize a file.
{INFO/}

{INFO: Synchronization limit}
There cannot be more than the [`MaxNumberOfSynchronizationsPerDestination`](./configurations#ravensynchronizationconfig) concurrent synchronization requests to a single
destination file system. However, to keep up as many request as possible, every finishing synchronization attempts to schedule the synchronization of the next file (if there is any pending one).
{INFO/}

{INFO: Etag order}
Files are sent to a destination file system in the `Etag` order.
{INFO/}

#File locking and synchronization timeout

When a destination file system is in the middle of the synchronization process of a given file, this file is inaccessible.
An attempt to modify it will result it  `PreconditionFailed (412)` response. Behind the scenes there is a locking mechanism which creates the
[`SyncingLock/[FILENAME]`](./configurations#syncinglockfilename) configuration when the synchronization starts and as long as it exists, 
the file is not allowed to be read or modified. The lock is removed at the end of the file synchronization process. 

In order to avoid potential deadlocks (e.g. when server restarts in the middle of the synchronization) we have a timeout value that specifies the max synchronization duration. You can control its value 
by setting the `SynchronizationLockTimeout` in the [`Raven/Synchronization/Config`](./configurations#ravensynchronizationconfig) configuration. If the synchronization duration reaches the timeout value, then an attempt to access the file
will automatically unlock it. By default the synchronization timeout is 10 minutes.

{INFO: Transactional locking}
Checking whether a file is already locked and a lock operation are made in the same transaction. There is no possibility that two servers
 will synchronize a file with the same name simultaneously. The lock taken by the first file system will prevent from syncing the second one.
{INFO/}

##Synchronization aborts

In contrast to a destination system, a source does not need to lock a file during its synchronization. You are able to read and modify it. 
However, if a file which synchronization is in progress is modified, then the synchronization will be aborted and retried with a new version.

##Handling failures and restarts

RavenFS has been designated to work with large files. File synchronizations can take quite a long time, especially if it synchronizes the first file version. 
Then it needs to transfer the entire file content to the destination nodes. It can happen any synchronization failure meanwhile caused by a network 
problem or a remote machine restart. Raven File System ensures that both files (source and remote versions) will eventually have the same name, content and metadata
even in the presence of a failure or temporary unavailability of the destination file system. In order to make sure that no file change is missed, the following mechanisms are used:

* `Etag` tracking - the successful synchronization stores `Etag` of just synchronized file (provided by the source) on the destination side.
If the source file system determines which files need to be synchronized, it asks the destination for `Etag` of the file last received from it.
Any modification of the file increases its `Etag` (`Etags` are sequential), so we just need to synchronize files that have `Etags` greater than the one returned by
the destination.

* confirmations - every synchronization must be confirmed. After a synchronization cycle, the source file system keeps info just about the synchronized files.
Next synchronization task run starts from asking a destination about status of the already accomplished synchronizations. If it answers that a file synchronization
has failed, the file will be added to the synchronization queue again.

* periodic runs - the synchronization task runs every 10 minutes.

## Related articles

- [Enabling Remote Differential Compression](../server/enabling-rdc)

