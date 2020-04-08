---
title: Cloud Snaps
keywords: portworx, container, Kubernetes, storage, Docker, k8s, flexvol, pv, persistent disk, snapshots, stork, clones, cloud, cloudsnap
description: Learn to take a cloud snapshot of a Portworx volume using pxctl and use that snapshot
weight: 3
---

## Multi-Cloud Backup and Recovery of PX Volumes

This document outlines how PX volumes can be backed up to different cloud provider's object storage including any S3-compatible object storage. If a user wishes to restore any of the backups, they can restore the volume from that point in the timeline. This enables administrators running persistent container workloads on-prem or in the cloud to safely backup their mission critical database volumes to cloud storage and restore them on-demand, enabling a seamless DR integration for their important business application data.

### Supported Cloud Providers

Portworx PX-Enterprise supports the following cloud providerss
1. Amazon S3 and any S3-compatible Object Storage
2. Azure Blob Storage
3. Google Cloud Storage

### Backing up a PX Volume to cloud storage

The first backup uploaded to the cloud is a full backup. After that, subsequent backups are incremental.
After 6 incremental backups, every 7th backup is a full backup.

### Restoring a PX Volume from cloud storage

Any PX Volume backup can be restored to a PX Volume in the cluster. The restored volume inherits the attributes such as file system, size and block size from the backup. Replication level and aggregation level of the restored volume defaults to 1 irrespective of the replication and aggregation level of the volume that was backed up. Users can increase replication or aggregation level level once the restore is complete on the restored volume.

### Performing Cloud Backups of a PX Volume

Performing cloud backups of a PX Volume is available via `pxctl cloudsnap` command. This command has the following operations available for the full lifecycle management of cloud backups.

```text
# opt/pwx/bin/pxctl cloudsnap --help
NAME:
   pxctl cloudsnap - Backup and restore snapshots to/from cloud

USAGE:
   pxctl cloudsnap command [command options] [arguments...]

COMMANDS:
    backup, b         Backup a snapshot to cloud
    restore, r        Restore volume to a cloud snapshot
    list, l           List snapshot in cloud
    status, s         Report status of active backups/restores
    history, h        Show history of cloudsnap operations
    stop, st          stop an active backup/restore
    schedules, sched  Manage schedules for cloud-snaps
    catalog, t        Display catalog for the backup in cloud
    delete, d         Delete a cloudsnap from the objectstore. This is not reversible.

OPTIONS:
   --help, -h  show help
```

#### Set the required cloud credentials ####

For this, we will use `pxctl credentials create` command. These cloud credentials are stored in an external secret store. Before you use the command to create credentials, ensure that you have [configured a secret provider of your choice](/key-management).

```text
# pxctl credentials create

NAME:
   pxctl credentials create - Create a credential for cloud-snap

USAGE:
   pxctl credentials create [command options] [arguments...]

OPTIONS:
   --provider value                            Object store provider type [s3, azure, google]
   --s3-access-key value
   --s3-secret-key value
   --s3-region value
   --s3-endpoint value                         Endpoint of the S3 server, in host:port format
   --s3-disable-ssl
   --azure-account-name value
   --azure-account-key value
   --google-project-id value
   --google-json-key-file value
   --encryption-passphrase value,
   --enc value  Passphrase to be used for encrypting data in the cloudsnaps
```

For Azure:

```text
# pxctl credentials create --provider azure --azure-account-name portworxtest --azure-account-key zbJSSpOOWENBGHSY12ZLERJJV
```

For AWS:

By default, Portworx creates a bucket (ID same as cluster UUID) to upload cloudsnaps. With Portworx version 1.5.0 onwards,uploading to a pre-created bucket by a user is supported. Thus the AWS credential provided to Portworx should either have the capability to create a bucket or the bucket provided to Portworx at minimum must have the permissions mentioned below. If you prefer that a user specified bucket be used for cloudsnaps, specify the bucket id with `--bucket` option while creating the credentials.

With user specified bucket (applicable only from 1.5.0 onwards):
```text
pxctl credentials create mycreds --provider=s3 --s3-disable-ssl --s3-region=us-east-1 --s3-access-key=<S3-ACCESS_KEY> --s3-secret-key=<S3-SECRET_KEY> --s3-endpoint=mys3-enpoint.com --disable-path-style --bucket=mybucket
```
User created/specified bucket at minimum must have following permissions: Replace `<bucket-name>` with name of your user-provided bucket.
```json
{
     "Version": "2012-10-17",
     "Statement": [
			{
				"Sid": "VisualEditor0",
				"Effect": "Allow",
				"Action": [
					"s3:ListAllMyBuckets",
					"s3:GetBucketLocation"
				],
				"Resource": "*"
			},
			{
				"Sid": "VisualEditor1",
				"Effect": "Allow",
				"Action": "s3:*",
				"Resource": [
					"arn:aws:s3:::<bucket-name>",
					"arn:aws:s3:::<bucket-name/*"
				]
			}
		]
 }
```

Without user specified bucket:
```text
# pxctl credentials create --provider s3  --s3-access-key AKIAJ7CDD7XGRWVZ7A --s3-secret-key mbJKlOWER4512ONMlwSzXHYA --s3-region us-east-1 --s3-endpoint s3.amazonaws.com
```

#### Google Cloud

1. Make sure the user or service account used by Portworx has the following roles:

   * Editor
   * Storage
   * Object Admin
   * Storage Object Viewer

    For more information about roles and permissions within GCP, see the [Granting, changing, and revoking access to resources](https://cloud.google.com/iam/docs/granting-changing-revoking-access) section of the GCP documentation.

2. Enter the `pxctl credentials create` command  specifying:

   * The `provider` flag with the name of the provider (`google`)
   * The `--google-project-id` flag with your Google project ID
   * The `--google-json-key-file` flag with the name of the JSON file containing your key
   * The name of your cloud credentials

    Example:

    ```text
    pxctl credentials create --provider google --google-project-id px-test --google-json-key-file px-test.json my-google-cred
    ```

#### Configure credentials

`pxctl credentials create` enables the user to configure the credentials for each supported cloud provider.

An additional encryption key can also be provided for each credential. If provided, all the data being backed up to the cloud will be encrypted using this key. The same key needs to be provided when configuring the credentials for restore to be able to decrypt the data succesfuly.

These credentials can only be created once and cannot be modified. In order to maintain security, once configured, the secret parts of the credentials will not be displayed.

#### List the credentials to verify ####

Use `pxctl credentials list` to verify the credentials supplied.

```text
# pxctl credentials list

S3 Credentials
UUID                                         REGION            ENDPOINT                ACCESS KEY            SSL ENABLED        ENCRYPTION
5c69ca53-6d21-4086-85f0-fb423327b024        us-east-1        s3.amazonaws.com        AKIAJ7CDD7XGRWVZ7A        true           false

Azure Credentials
UUID                                        ACCOUNT NAME        ENCRYPTION
c0e559a7-8d96-4f28-9556-7d01b2e4df33        portworxtest        false

Google Credentials
UUID						PROJECT ID     ENCRYPTION
8bd266b5-da9f-4114-84a2-309bbb3838c6		px-test        false

```

`pxctl credentials list`  only displays non-secret values of the credentials. Secrets are neither stored locally nor displayed.  These credentials will be stored as part of the secret endpoint given for PX for persisting authentication across reboots. Please refer to `pxctl secrets` help for more information.

#### Perform Cloud Backup ####

The actual backup of the PX Volume is done via the `pxctl cloudsnap backup` command

```text
# pxctl cloudsnap backup

NAME:
   pxctl cloudsnap backup - Backup a snapshot to cloud

USAGE:
   pxctl cloudsnap backup [command options] [arguments...]

OPTIONS:
   --volume value, -v value       source volume
   --full, -f                     force a full backup
   --cred-uuid value, --cr value  Cloud credentials ID to be used for the backup

```

This command is used to backup a single volume to the cloud provider using the specified credentials.
This command decides whether to take a full or incremental backup depending on the existing backups for the volume.
If it is the first backup for the volume it takes a full backup of the volume. If its not the first backup, it takes an incremental backup from the previous full/incremental backup.

```text
# pxctl cloudsnap backup volume1 --cred-uuid 82998914-5245-4739-a218-3b0b06160332
```

Users can force the full backup any time by giving the --full option.
If only one credential is configured on the cluster, then the cred-uuid option may be skipped on the command line.

Here are a few steps to perform cloud backups successfully

* List all the available volumes to choose the volume to backup

```text
# pxctl volume list
ID			NAME	SIZE	HA	SHARED	ENCRYPTED	IO_PRIORITY	SCALE	STATUS
538316104266867971	NewVol	4 GiB	1	no	no		LOW		1	up - attached on 70.0.9.73
980081626967128253	evol	2 GiB	1	no	no		LOW		1	up - detached
```

```output
pxctl cloudsnap list --help
List snapshot in cloud

Usage:
  pxctl cloudsnap list [flags]

Aliases:
  list, l

Flags:
  -m, --migration             Optional, lists migration related cloudbackups
  -a, --all                   List cloud backups of all clusters in cloud
  -s, --src string            Optional source volume to list cloud backups
      --cred-id string        Cloud credentials ID to be used for the backup
  -c, --cluster string        Optional cluster id to list cloud backups. Current cluster-id is default
  -t, --status string         Optional backup status(failed. aborted, stopped) to list cloud backups; Defaults to Done
      --label pairs           Optional list of comma-separated name=value pairs to match with cloudsnap metadata
  -i, --cloudsnap-id string   Optional cloudsnap id to list(lists a single entry)
  -x, --max uint              Optional number to limit display of backups in each page
  -h, --help                  help for list

Global Flags:
      --ca string        path to root certificate for ssl usage
      --cert string      path to client certificate for ssl usage
      --color            output with color coding
      --config string    config file (default is $HOME/.pxctl.yaml)
      --context string   context name that overrides the current auth context
  -j, --json             output in json
      --key string       path to client key for ssl usage
      --raw              raw CLI output for instrumentation
      --ssl              ssl enabled for portworx
```

### Inspect a cloud snapshot

The `pxctl cloudsnap list` command displays all the cloud snapshots for a given credential, source volume, or type of cloud snapshot. To view more details about a particular cloud snapshot, you must specify the `-i` flag with the ID of the cloud snapshot you want to inspect.

Example:

1. Start by listing your cloud snapshots with:

      ```text
      pxctl cloudsnap list
      ```

      ```output
      SOURCEVOLUME            SOURCEVOLUMEID            CLOUD-SNAP-ID                                        CREATED-TIME                TYPE        STATUS
      agg-cs_journal_1        10769800556491614        fe431d7d-0b42-4a4b-9496-f3e9050d0f68/10769800556491614-673132711323933325        Thu, 24 Oct 2019 19:02:08 UTC        Manual        Done
      agg-cs_0            365276421799434338        fe431d7d-0b42-4a4b-9496-f3e9050d0f68/365276421799434338-461608030527675278        Thu, 24 Oct 2019 19:02:47 UTC        Manual        Done
```


2. To inspect the first cloud snapshot (`fe431d7d-0b42-4a4b-9496-f3e9050d0f68/10769800556491614-673132711323933325`) and print the output in JSON format, enter the following command:

      ```text
      pxctl -j cloudsnap list --cloudsnap-id fe431d7d-0b42-4a4b-9496-f3e9050d0f68/10769800556491614-673132711323933325
      ```

      ```output
      [
      {
        "ID": "fe431d7d-0b42-4a4b-9496-f3e9050d0f68/10769800556491614-673132711323933325",
        "SrcVolumeID": "10769800556491614",
        "SrcVolumeName": "agg-cs_journal_1",
        "Timestamp": "2019-10-24T19:02:08Z",
        "Metadata": {
        "cloudsnapType": "Manual",
        "compression": "lz77",
        "sizeBytes": "2152751104",
        "starttime": "Thu, 24 Oct 2019 19:02:08 UTC",
        "status": "Done",
        "updatetime": "Thu, 24 Oct 2019 19:06:12 UTC",
        "version": "V2.00",
        "volume": "{\"DevSpec\":{\"size\":137438953472,\"format\":2,\"block_size\":4096,\"ha_level\":1,\"cos\":3,\"volume_labels\":{\"best_effort_location_provisioning\":\"true\",\"name\":\"vContainer\"},\"replica_set\":{},\"aggregation_level\":1,\"scale\":1,\"journal\":true,\"queue_depth\":128,\"force_unsupported_fs_type\":true,\"io_strategy\":{}},\"UsedSize\":0,\"PoolId\":0,\"ClusterId\":\"PX-INT-C0-BVT-MN-NS-BRANCH_476_24_Oct_19_04_49_UTC\",\"PublicSecretData\":null,\"Labels\":null}",
        "volumename": "agg-cs_journal_1"
        },
        "Status": "Done"
      }
      ```

### Perform cloud backup of a group of volumes

Portworx 2.0.3 and higher supports backing up multiple volumes to cloud at the same consistency point. To see the available command line options, run:

```text
# pxctl cloudsnap credentials list

Azure Credentials
UUID						ACCOUNT NAME		ENCRYPTION
ef092623-f9ba-4697-aeb5-0d5d6d9b5742		portworxtest		false
```

Authenticate the nodes where the storage for volume to be backed up is provisioned.

* Login to the secrets database to use encryption in-flight

```text
# pxctl secrets kvdb login
Successful Login to Secrets Endpoint!
```

* Now issue the backup command

Note that in this particular example,  since only one credential is configured, there is no need to specify the credentials on the command line

```text
# pxctl cloudsnap backup NewVol
Cloudsnap backup started successfully
```

* Watch the status of the backup

```text
# pxctl cloudsnap status
SOURCEVOLUME		STATE		BYTES-PROCESSED	TIME-ELAPSED	COMPLETED			ERROR
538316104266867971	Backup-Active	62914560	20.620429615s
980081626967128253	Backup-Done	68383234	4.522017785s	Sat, 08 Apr 2017 05:09:54 UTC
```

Once the volume is backed up to the cloud successfully, listing the remote cloudsnaps will display the backup that just completed.

* List the backups in cloud

```text
# pxctl cloudsnap list
SOURCEVOLUME	CLOUD-SNAP-ID					CREATED-TIME			STATUS
evol		pqr9-cl1/980081626967128253-941778877687318172	Sat, 08 Apr 2017 05:09:49 UTC	Done
NewVol		pqr9-cl1/538316104266867971-807625803401928868	Sat, 08 Apr 2017 05:17:21 UTC	Done
```

#### Restore from a Cloud Backup ####

Use `pxctl cloudsnap restore` to restore from a cloud backup.

Here is the command syntax.

```text
# pxctl cloudsnap restore

NAME:
   pxctl cloudsnap restore - Restore volume to a cloud snapshot

USAGE:
   pxctl cloudsnap restore [command options] [arguments...]

OPTIONS:
   --snap value, -s value         Cloud-snap id
   --node value, -n value         Optional node ID for provisioning restore volume storage
   --cred-uuid value, --cr value  Cloud credentials ID to be used for the restore

```

This command is used to restore a successful backup from cloud. It requires the cloudsnap ID which can be used to restore and credentials for the cloud storage provider or the object storage. Restore happens on any node where storage can be provisioned. In this release restored volume will have a replication factor of 1. The restored volume can be updated to different replication factors using `pxctl volume ha-update` command.

The command usage is as follows.
```text
# pxctl cloudsnap restore --snap cs30/669945798649540757-864783518531595119 --cr 82998914-5245-4739-a218-3b0b06160332
```

Upon successful start of the command it returns the volume id created to restore the cloud snap
If the command fails to succeed, it shows the failure reason.

The restored volume will not be attached or mounted automatically.


* Use `pxctl cloudsnap list` to list the available backups.

`pxctl cloudsnap list` helps enumerate the list of available backups in the cloud. This command assumes that you have all the credentials setup properly. If the credentials are not setup, then the backups available in those clouds won't be listed by this command.

```text
# pxctl cloudsnap list
SOURCEVOLUME 	CLOUD-SNAP-ID					CREATED-TIME			STATUS
dvol		pqr9-cl1/520877607140844016-50466873928636534	Fri, 07 Apr 2017 20:22:43 UTC	Done
NewVol		pqr9-cl1/538316104266867971-807625803401928868	Sat, 08 Apr 2017 05:17:21 UTC	Done
```

* Choose one of them to restore

```text
# pxctl cloudsnap restore -s pqr9-cl1/538316104266867971-807625803401928868
Cloudsnap restore started successfully: 622390253290820715
```
`pxctl cloudsnap status` gives the status of the restore processes as well.

```text
# pxctl cloudsnap status
SOURCEVOLUME		STATE		BYTES-PROCESSED	TIME-ELAPSED	COMPLETED			ERROR
622390253290820715	Restore-Active	99614720	10.144539084s
980081626967128253	Backup-Done	68383234	4.522017785s	Sat, 08 Apr 2017 05:09:54 UTC
538316104266867971	Backup-Done	1979809411	2m39.761333366s	Sat, 08 Apr 2017 05:20:01 UTC
```

#### Deleting a Cloud Backup ###

{{<info>}}
**Note:**<br/> This is only supported from PX version 1.4 onwards
{{</info>}}

You can delete backups from the cloud using the `/opt/pwx/bin/pxctl cloudsnap delete` command. The command will mark a cloudsnap for deletion and a job will take care of deleting objects associated with these backups from the objectstore.

Only cloudsnaps which do not have any dependant cloudsnaps (ie incrementals) can be deleted. If there are dependant cloudsnaps then the command will throw an error with the list of cloudsnaps that need to be deleted first.

For example to delete the backup `pqr9-cl1/538316104266867971-807625803401928868`:

```text
# pxctl cloudsnap delete --snap pqr9-cl1/538316104266867971-807625803401928868
Cloudsnap deleted successfully
# pxctl cloudsnap list
SOURCEVOLUME 	CLOUD-SNAP-ID					CREATED-TIME			STATUS
dvol		pqr9-cl1/520877607140844016-50466873928636534	Fri, 07 Apr 2017 20:22:43 UTC	Done
```

## Cloud operations

Help for specific cloudsnap commands can be found by running the following command

Note: All cloudsnap operations requires secrets login to configured endpoint with/without encryption. Please refer pxctl secrets cmd help.

Also, to see how to configure cloud provider credentials, click the link below.

[Credentials](/reference/cli/credentials)

**pxctl cloudsnap –help**

```text
/opt/pwx/bin/pxctl cloudsnap --help
NAME:
   pxctl cloudsnap - Backup and restore snapshots to/from cloud

USAGE:
   pxctl cloudsnap command [command options] [arguments...]

COMMANDS:
     backup, b         Backup a snapshot to cloud
     restore, r        Restore volume to a cloud snapshot
     list, l           List snapshot in cloud
     status, s         Report status of active backups/restores
     history, h        Show history of cloudsnap operations
     stop, st          stop an active backup/restore
     schedules, sched  Manage schedules for cloud-snaps
     catalog, t        Display catalog for the backup in cloud
     delete, d         Delete a cloudsnap from the objectstore. This is not reversible.

OPTIONS:
   --help, -h  show help
```

**pxctl cloudsnap backup**

`pxctl cloudsnap backup` command is used to backup a single volume to the configured cloud provider through credential command line. If it will be the first backup for the volume a full backup of the volume is generated. If it is not the first backup, it only generates an incremental backup from the previous full/incremental backup. If a single cloud provider credential is created then there is no need to specify the credentials on the command line.

```text
/opt/pwx/bin/pxctl cloudsnap backup vol1
Cloudsnap backup started successfully
```

If multiple cloud providers credentials are created then need to specify the credential to use for backup on command line

```text
/opt/pwx/bin/pxctl cloudsnap backup vol1 --cred-uuid ffffffff-ffff-ffff-1111-ffffffffffff
Cloudsnap backup started successfully
```

Note: All cloudsnap backups and restores can be monitored through CloudSnap status command which is described in following sections

**pxctl cloudsnap restore**

`pxctl cloudsnap restore` command is used to restore a successful backup from cloud. \(Use cloudsnap list command to get the cloudsnap Id\). It requires cloudsnap Id \(to be restored\) and credentials. Restore happens on any node in the cluster where storage can be provisioned. In this release, restored volume will be of replication factor 1. This volume can be updated to different repl factors using volume ha-update command.

```text
sudo /opt/pwx/bin/pxctl cloudsnap restore --snap gossip12/181112018587037740-545317760526242886
Cloudsnap restore started successfully: 315244422215869148
```

Note: All cloudsnap backups and restores can be monitored through CloudSnap status command which is described in following sections

**pxctl cloudsnap status**

`pxctl cloudsnap status` can be used to check the status of cloudsnap operations

```text
/opt/pwx/bin/pxctl cloudsnap status
SOURCEVOLUME		   STATE		      BYTES-PROCESSED	TIME-ELAPSED		COMPLETED			            ERROR
1040525385624900824	Restore-Done	11753581193	      8m32.231744596s	Wed, 05 Apr 2017 06:57:08 UTC
1137394071301823388	Backup-Done	   11753581193	      1m46.023734966s	Wed, 05 Apr 2017 05:03:42 UTC
13292162184271348	   Backup-Done	   27206221391	      4m25.740022954s	Wed, 05 Apr 2017 22:39:41 UTC
454969905909227504	Backup-Active	91944386560	      4h8m19.283242837s
827276927130532677	Restore-Failed	0									                                       Failed to authenticate creds ID
```

**pxctl cloudsnap list**

`pxctl cloudsnap list` is used to list all the cloud snapshots

```text
/opt/pwx/bin/pxctl cloudsnap list --cred-uuid ffffffff-ffff-ffff-1111-ffffffffffff --all
SOURCEVOLUME 			CLOUD-SNAP-ID									CREATED-TIME				STATUS
vol1			gossip12/181112018587037740-545317760526242886		Sun, 09 Apr 2017 14:35:28 UTC		Done
```

Filtering on cluster ID or volume ID is available and can be done as follows:

```text
/opt/pwx/bin/pxctl cloudsnap list --cred-uuid ffffffff-ffff-ffff-1111-ffffffffffff --src vol1
SOURCEVOLUME 		CLOUD-SNAP-ID					CREATED-TIME				STATUS
vol1			1137394071301823388-283948499973931602		Wed, 05 Apr 2017 04:50:35 UTC		Done
vol1			1137394071301823388-674319852060841900		Wed, 05 Apr 2017 05:01:56 UTC		Done

/opt/pwx/bin/pxctl cloudsnap list --cred-uuid ffffffff-ffff-ffff-1111-ffffffffffff --cluster cs25
SOURCEVOLUME 		CLOUD-SNAP-ID					CREATED-TIME				STATUS
vol1			1137394071301823388-283948499973931602		Wed, 05 Apr 2017 04:50:35 UTC		Done
vol1			1137394071301823388-674319852060841900		Wed, 05 Apr 2017 05:01:56 UTC		Done
volshared1	13292162184271348-457364119636591866		Wed, 05 Apr 2017 22:35:16 UTC		Done
```

**pxctl cloudsnap delete**

`pxctl cloudsnap delete` can be used to delete cloudsnaps. This is not reversible and will cause the backups to be permanently deleted from the cloud, so use with care.

```text
/opt/pwx/bin/pxctl cloudsnap  delete --snap gossip12/181112018587037740-545317760526242886
Cloudsnap deleted successfully
```
