# blobxfer Command-Line Usage
`blobxfer` operates using a command followed by options. Each
command will be detailed along with all options available.

### Quick Navigation
1. [Commands](#commands)
2. [Options](#options)
3. [Example Invocations](#examples)
4. [General Notes](#general-notes)

## <a name="commands"></a>Commands
### `download`
Downloads remote Azure paths, which may contain many resources, to the
local machine. This command requires at the minimum, the following options
if invoked without a YAML configuration file:

* Remote Azure Storage reference using one of two methods:
    * URL-based via `--storage-url` or the environment variable
      `BLOBXFER_STORAGE_URL`
    * Components
        * `--mode` specifies the source Azure Storage mode. This defaults
          to `auto` which will target Azure Blob storage (any blob type).
          To source from Azure File storage, set this option to `file`.
        * `--remote-path` for the source remote Azure path. This must have,
          at the minimum, a container or file share name.
        * `--storage-account` storage account for the source remote Azure path
          or the environment variable `BLOBXFER_STORAGE_ACCOUNT`
* Local path to download files to with `--local-path`
* An authentication option for the storage account is required. Please see
the Authentication sub-section below under Options.

### `upload`
Uploads local paths to a remote Azure path or set of remote Azure paths.
The local path may contain many resources on the local machine. This command
requires at the minimum, the following options if invoked without a YAML
configuration file:

* Remote Azure Storage reference using one of two methods:
    * URL-based via `--storage-url` or the environment variable
      `BLOBXFER_STORAGE_URL`
    * Components
        * `--mode` specifies the destination Azure Storage mode. This
          defaults to `auto` which will target Azure Blob storage (any blob
          type). To upload to Azure File storage, set this option to `file`.
        * `--remote-path` for the destination remote Azure path. This must
          have, at the minimum, a container or file share name.
        * `--storage-account` storage account for the destination remote
          Azure path or the environment variable `BLOBXFER_STORAGE_ACCOUNT`
* Local path to upload files from with `--local-path`
    * If piping from `stdin`, `--local-path` should be set to `-` as per
      convention.
* An authentication option for the storage account is required. Please see
the Authentication sub-section below under Options.

### `synccopy`
Synchronously copies remote paths (Azure or arbitrary URLs) to other remote
Azure paths. By default, copies occur on the Azure Storage servers. This
command requires at the minimum, the following options if invoked without
a YAML configuration file:

* Remote Azure Storage _source_ reference using one of two methods:
    * URL-based via `--storage-url` or the environment variable
      `BLOBXFER_STORAGE_URL`
    * Components
        * `--mode` specifies the source Azure Storage mode. This defaults
          to `auto` which will source from Azure Blob storage (any blob
          type). To source from Azure File storage, set this option to `file`.
        * `--remote-path` for the source remote path. If an Azure path this
          must have, at the minimum, a container or file share name. For
          an arbitrary URL, this must be a complete URL that starts with
          the proper protocol, e.g., `http://` or `https://`. Aribtrary URL
          support is limited to a single object.
        * `--storage-account` storage account for the source remote Azure path
          or the environment variable `BLOBXFER_STORAGE_ACCOUNT`
* Remote Azure Storage _destination_ reference using one of two methods:
    * URL-based via `--sync-copy-dest-storage-url` or the environment variable
      `BLOBXFER_SYNC_COPY_DEST_STORAGE_URL`
    * Components
        * `--sync-copy-dest-mode` for the destination mode. If `auto` is
          specified for this option, the destination mode will be set the
          same as the source `--mode`. These options need not be the same
          which allows for object-transform copy operations.
        * `--sync-copy-dest-remote-path` for the destination remote Azure
          path. This must have, at the minimum, a container or file share
          name.
        * `--sync-copy-dest-storage-account` storage account for the
          destination remote Azure path or the environment variable
          `BLOBXFER_SYNC_COPY_DEST_STORAGE_ACCOUNT`
* An authentication option for both storage accounts is required. Please
see the `Authentication` and `Connection` sub-section below under the
next section.

## <a name="options"></a>Options
### General
* `--config` specifies the YAML configuration file to use. This can be
optionally provided through an environment variable `BLOBXFER_CONFIG_FILE`.
* `--chunk-size-bytes` is the chunk size in bytes. For downloads, this
is the maximum length of data to transfer per request. For uploads, this
corresponds to one of block size for append and block blobs, page size for
page blobs, or file chunk for files. Only block blobs can have a block size
of up to 100MiB, all others have a maximum of 4MiB. Please see the
[performance considerations](98-performance-considerations.md) document
for important information regarding this option.
* `--dry-run` will not perform any actual download, upload or synccopy
operations and instead will log intent.
* `--enable-azure-storage-logger` enables the Azure Storage logger.
* `--file-attributes` or `--no-file-attributes` controls if POSIX file
attributes (mode and ownership) should be stored or restored. Note that to
restore uid/gid, `blobxfer` must be run as root or under sudo.
* `--file-md5` or `--no-file-md5` controls if the file MD5 should be computed.
* `--local-path` is the local resource path. Set to `-` if piping from
`stdin`.
* `--log-file` specifies the log file to write to. This must be specified
for a progress bar to be output to console.
* `--mode` is the operating mode. The default is `auto` but may be set to
`append`, `block`, `file`, or `page`. If specified with the `upload`
command, then all files will be uploaded as the specified `mode` type. If
`mode` is `auto` while uploading, then `.vhd` and `.vhdx` files are
uploaded automatically as page blobs while all other files are uploaded
as block blobs. If specified with `download`, then only remote entities
with that `mode` type are downloaded. Note that `file` should be specified
for the `mode` if interacting with Azure File shares.
* `--overwrite` or `--no-overwrite` controls clobber semantics at the
destination.
* `--progress-bar` or `--no-progress-bar` controls if a progress bar is
output to the console. `--log-file` must be specified for a progress bar
to be output.
* `-q` or `--quiet` enables quiet mode
* `--recursive` or `--no-recursive` controls if the source path should be
recursively uploaded or downloaded.
* `--remote-path` is a remote path. If an Azure path, this must contain the
Blob container or File share at the begining, e.g., `mycontainer/vdir`
* `--restore-file-lmt` will set the last access and modified times of a
downloaded file to the modified time set in Azure storage. This option can
only be applied to download operations.
* `--resume-file` specifies the resume database to write to or read from.
Resume files should be specific for a session.
* `--show-config` will show the configuration for the execution. Use caution
with this option as it will output secrets.
* `-h` or `--help` can be passed at every command level to receive context
sensitive help.
* `-v` or `--verbose` increases logging verbosity

### Authentication
`blobxfer` supports both Storage Account access keys and Shared Access
Signature (SAS) tokens. One type must be supplied with all commands in
order to successfully authenticate against Azure Storage. Note that if you
are utilizing `--storage-url` or `--sync-copy-dest-storage-url` and the
URL contains a SAS token, then you do not need to specify any of the
following options.

Authentication options are:

* `--sas` is a shared access signature (SAS) token. This can can be
optionally provided through an environment variable `BLOBXFER_SAS` instead.
* `--storage-account-key` is the storage account access key. This can be
optionally provided through an environment variable
`BLOBXFER_STORAGE_ACCOUNT_KEY` instead.
* `--sync-copy-dest-sas` is a shared access signature (SAS) token for the
destination Azure Storage account for the `synccopy` command. This can be
optionally provided through an environment variable
`BLOBXFER_SYNC_COPY_DEST_SAS` instead.
* `--sync-copy-dest-storage-account-key` specifies the destination Azure
Storage account key for the `synccopy` command. This can be optionally
provided through an environment variable
`BLOBXFER_SYNC_COPY_DEST_STORAGE_ACCOUNT_KEY` instead.

### Concurrency
Please see the [performance considerations](98-performance-considerations.md)
document for more information regarding concurrency options.

* `--crypto-processes` is the number of decryption offload processes to spawn.
`0` will in-line the decryption routine with the main thread.
* `--disk-threads` is the number of threads to create for disk I/O.
* `--max-single-object-concurrency` is the maximum number of concurrent
operations that can be applied to a single object during download.
* `--md5-processes` is the number of MD5 offload processes to spawn for
comparing files with `skip_on` `md5_match`.
* `--transfer-threads` is the number of threads to create for transferring
to/from Azure Storage.

### Connection
* `--connect-timeout` is the connect timeout value. This is the time allowed,
in seconds, to establish a connection with the server.
* `--endpoint` is the Azure Storage endpoint to connect to; the default is
Azure Public regions, or `core.windows.net`. Note that this is the base
endpoint name and not a full URL (as blobxfer deals with both Azure Blob
Storage and Files). You can use one of the following if you are not
connecting to an Azure Public region:
    * Azure China Cloud: `core.chinacloudapi.cn`
    * Azure Germany Cloud:  `core.cloudapi.de`
    * Azure US Government Cloud: `core.usgovcloudapi.net`
* `--max-retries` is the maximum number of retries for a request. Negative
values result in unlimited retries. The default is 1000.
* `--proxy-host` specifies the IP:port for an HTTP Proxy if required. This
can be optionally provided through an environment variable
`BLOBXFER_PROXY_HOST` instead.
* `--proxy-username` specifies a username for an HTTP Proxy if required. This
can be optionally provided through an environment variable
`BLOBXFER_PROXY_USERNAME` instead.
* `--proxy-password` specifies the password for the username for an HTTP
Proxy if required. This can be optionally provided through an environment
variable `BLOBXFER_PROXY_PASSWORD` instead.
* `--read-timeout` is the read timeout value. This is the time allowed,
in seconds, for the server to send a response.
* `--storage-account` specifies the storage account to use. This can be
optionally provided through an environment variable `BLOBXFER_STORAGE_ACCOUNT`
instead.
* `--storage-url` specifies the Azure Storage url to read entities from
or to write entities to. This parameter can be used in-lieu of
`--mode`, `--storage-account`, `--endpoint`, `--remote-path` and potentially
`--sas` if a SAS token is supplied as part of the URL.
* `--sync-copy-dest-storage-account` specifies the destination remote
Azure storage account for the `synccopy` command. This can be optionally
provided through an environment variable
`BLOBXFER_SYNC_COPY_DEST_STORAGE_ACCOUNT` instead.
* `--sync-copy-dest-mode` specifies the destination mode for `synccopy`
operations.
* `--sync-copy-dest-remote-path` specifies the destination remote Azure path
under the synchronous copy destination storage account.
* `--sync-copy-dest-storage-url` specifies the Azure Storage url to write
entities to as part of `synccopy` operations. This parameter can be used
in-lieu of `--sync-copy-dest-mode`, `--sync-copy-dest-storage-account`,
`--sync-copy-remote-path` and potentially `--sync-copy-dest-sas` if a SAS
token is supplied as part of the URL.
* `--timeout` is the timeout value, in seconds, applied to both connect
and read operations. This option is DEPRECATED. Please specify the connect
timeout as `--connect-timeout` and the read timeout as `--read-timeout`
separately.

### Encryption
* `--rsa-private-key` is the RSA private key in PEM format to use. This can
be provided for uploads but must be specified to decrypt encrypted remote
entities for downloads. This can be optionally provided through an environment
variable `BLOBXFER_RSA_PRIVATE_KEY`.
* `--rsa-private-key-passphrase` is the RSA private key passphrase. This can
be optionally provided through an environment variable
`BLOBXFER_RSA_PRIVATE_KEY_PASSPHRASE`.
* `--rsa-public-key` is the RSA public key in PEM format to use. This
can only be provided for uploads. This can be optionally provided through an
environment variable `BLOBXFER_RSA_PUBLIC_KEY`.

### Filtering
* `--exclude` is an exclude pattern to use; this can be specified multiple
times. Exclude patterns are applied after include patterns. If both an exclude
and an include pattern match a target, the target is excluded.
* `--include` is an include pattern to use; this can be specified multiple
times

### Skip On
* `--skip-on-filesize-match` will skip the transfer action if the filesizes
match between source and destination. This should not be specified for
encrypted files.
* `--skip-on-lmt-ge` will skip the transfer action:
    * On upload if the last modified time of the remote file is greater than
      or equal to the local file.
    * On download if the last modified time of the local file is greater than
      or equal to the remote file.
* `--skip-on-md5-match` will skip the transfer action if the MD5 hash match
between source and destination. This can be transparently used through
encrypted files that have been uploaded with `blobxfer`.

### Vectored IO
Please see the [Vectored IO](30-vectored-io.md) document for more information
regarding Vectored IO operations in `blobxfer`.

* `--distribution-mode` is the Vectored IO distribution mode
    * `disabled` which is default (no Vectored IO)
    * `replica` which will replicate source files to target destinations on
      upload. Note that replicating across multiple destinations will require
      a YAML configuration file.
    * `stripe` which will stripe source files to target destinations on
      upload. Note that striping across multiple destinations will require
      a YAML configuration file.
* `--stripe-chunk-size-bytes` is the stripe chunk width for stripe-based
Vectored IO operations

### Other
* `--access-tier` sets the
[access tier](https://docs.microsoft.com/azure/storage/blobs/storage-blob-storage-tiers)
of the object (block blobs only) for upload and synccopy operations. If this
is not specified, which is the default, then no access tier is actively set
for the object which then inherits from whatever the default access tier is
for the account. Note that the access tier is applied to all files
transferred for the blobxfer invocation. This can only be set for Blob
Storage and General Purpose V2 storage accounts. Valid values for this
option are `hot`, `cool`, or `archive`.
* `--delete` deletes extraneous files (including blob snapshots if the parent
is deleted) at the remote destination path on uploads and at the local
resource on downloads. This action occurs after all transfers have taken place,
similarly to rsync's delete after option. Note that this interacts with other
filters such as `--include` and `--exclude`.
* `--delete-only` only performs the delete operations as per `--delete` but
no transfer is invoked. This option must be specified with `--delete`.
* `--file-cache-control` sets the
[CacheControl](https://docs.microsoft.com/azure/cdn/cdn-manage-expiration-of-blob-content)
property on the destination entity on upload operations. Note that if an
existing file is not uploaded due to a skip option, but the cache control
setting differs, the target is not updated. To override this behavior,
either delete the target object or do not specify a skip condition.
* `--file-content-type` sets the `ContentType`
property on the destination entity on upload operations. By default, if this
option is not specified, the content type is guessed from the file. Note that
if an existing file is not uploaded due to a skip option, but the content type
setting differs, the target is not updated. To override this behavior,
either delete the target object or do not specify a skip condition.
* `--one-shot-bytes` controls the number of bytes to "one shot" a block
Blob upload. The maximum value that can be specified is 256MiB. This may
be useful when using account-level SAS keys and enforcing non-overwrite
behavior.
* `--rename` renames a single file to the target destination or source path.
This can only be used when transferring a single source file to a destination
and can be used with any command. This is automatically enabled when
using `stdin` as a source.
* `--server-side-copy` or `--no-server-side-copy` enables or disables
server side copies for synccopy operations. By default, this is enabled.
Only block blob destinations are supported for server side copies. If
the destination is not block blob, then this option must be disabled.
* `--stdin-as-page-blob-size` allows a page blob size to be set if known
beforehand when using `stdin` as a source and the destination is a page blob.
This value will automatically be page blob boundary aligned.
* `--strip-components N` will strip the leading `N` components from the
local file path on upload or remote file path on download. The default is `0`.

## <a name="examples"></a>Example Invocations
### `download` Examples
#### Download an Entire Encrypted Blob Container to Current Working Directory
```shell
blobxfer download --storage-account mystorageaccount --sas "mysastoken" --remote-path mycontainer --local-path . --rsa-private-key ~/myprivatekey.pem
```

#### Download an Entire File Share to Designated Path and Skip On Filesize Matches
```shell
blobxfer download --mode file --storage-account mystorageaccount --storage-account-key "myaccesskey" --remote-path myfileshare --local-path /my/path --skip-on-filesize-match
```

#### Download only Page Blobs in Blob Container Virtual Directory Non-recursively and Cleanup Local Path to Match Remote Path
```shell
blobxfer download --mode page --storage-account mystorageaccount --storage-account-key "myaccesskey" --remote-path mycontainer --local-path /my/pageblobs --no-recursive --delete
```

#### Resume Incomplete Downloads Matching an Include Pattern and Log to File and Restore POSIX File Attributes and Last Modified Time
```shell
blobxfer download --storage-account mystorageaccount --storage-account-key "myaccesskey" --remote-path mycontainer --local-path . --include '*.bin' --resume-file myresumefile.db --log-file blobxfer.log --file-attributes --restore-file-lmt
```

#### Download a Blob Snapshot
```shell
blobxfer download --storage-account mystorageaccount --sas "mysastoken" --remote-path "mycontainer/file.bin?snapshot=2017-04-20T02:12:49.0311708Z" --local-path .
```

#### Download an Entire File Share Snapshot
```shell
blobxfer download --mode file --storage-account mystorageaccount --sas "mysastoken" --remote-path "myshare?snapshot=2017-04-20T02:12:49.0311708Z" --local-path .
```

#### Download using a YAML Configuration File
```shell
blobxfer download --config myconfig.yaml
```

### `upload` Examples
#### Upload Current Working Directory as Encrypted Block Blobs Non-recursively
```shell
blobxfer upload --storage-account mystorageaccount --sas "mysastoken" --remote-path mycontainer --local-path . --no-recursive --rsa-public-key ~/mypubkey.pem
```

#### Upload Specific Path Recursively to a File Share, Store File MD5 and POSIX File Attributes to a File Share and Exclude Some Files
```shell
blobxfer upload --mode file --storage-account mystorageaccount --sas "mysastoken" --remote-path myfileshare --local-path . --file-md5 --file-attributes --exclude '*.bak'
```

#### Upload Single File with Resume and Striped Vectored IO into 512MiB Chunks
```shell
blobxfer upload --storage-account mystorageaccount --sas "mysastoken" --remote-path mycontainer --local-path /some/huge/file --resume-file hugefileresume.db --distribution-mode stripe --stripe-chunk-size-bytes 536870912
```

#### Upload Specific Path but Skip On Any MD5 Matches, Store File MD5 and Cleanup Remote Path to Match Local Path
```shell
blobxfer upload --storage-account mystorageaccount --sas "mysastoken" --remote-path mycontainer --local-path /my/path --file-md5 --skip-on-md5-match --delete
```

#### Upload From Piped `stdin`
```shell
curl -fSsL https://some.uri | blobxfer upload --storage-account mystorageaccount --sas "mysastoken" --remote-path mycontainer --local-path -
```

#### Upload using a YAML Configuration File
```shell
blobxfer upload --config myconfig.yaml
```

### `synccopy` Examples
#### Synchronously Copy an Entire Path Recursively to Another Storage Account
```shell
blobxfer synccopy --storage-account mystorageaccount --sas "mysastoken" --remote-path mysourcecontainer --sync-copy-dest-storage-account mydestaccount --sync-copy-dest-storage-account-key "mydestkey" --sync-copy-dest-remote-path mydestcontainer
```

#### Synchronously Copy an Arbitrary URL
```shell
blobxfer synccopy --remote-path "https://raw.githubusercontent.com/Azure/blobxfer/master/README.md" --sync-copy-dest-storage-account mydestaccount --sync-copy-dest-storage-account-key "mydestkey" --sync-copy-dest-remote-path mydestcontainer
```

#### Synchronously Copy using a YAML Configuration File
```shell
blobxfer synccopy --config myconfig.yaml
```

## <a name="general-notes"></a>General Notes
* `blobxfer` does not take any leases on blobs or containers. It is up to the
user to ensure that blobs are not modified while download/uploads are being
performed.
* No validation is performed regarding container and file naming and length
restrictions.
* `blobxfer` will attempt to download from blob storage as-is. If the source
filename is incompatible with the destination operating system, then failure
may result.
* When using SAS, the SAS key must have container- or share-level permissions
if performing recursive directory upload or container/file share download.
* If uploading via service-level SAS keys, the container or file share must
already be created in Azure storage prior to upload. Account-level SAS keys
with the signed resource type of `c` (i.e., container-level permission) is
required for to allow container or file share creation. Please see the
[limitations](99-current-limitations.md) doc for more considerations when
using SAS keys.
* When uploading files as page blobs, the content is page boundary
byte-aligned. The MD5 for the blob is computed using the final aligned data
if the source is not page boundary byte-aligned. This enables these page
blobs or files to be skipped during subsequent download or upload with the
appropriate `skip_on` option, respectively.
* Globbing of wildcards must be disabled by your shell (or properly quoted)
during invoking `blobxfer` such that include and exclude patterns can be
read verbatim without the shell expanding the wildcards.
