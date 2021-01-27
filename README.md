# Running Kirby CMS on Object Storage via PHP Stream Wrappers

## Problem
[Kirby CMS](https://github.com/getkirby/kirby) is a popular flat file content management system written in PHP. Instead of a database, it uses the file system to store content and media. This makes it rather easy to setup and deploy on local dev machines, servers and webspace products. It would be nice, if Kirby was optionally also able to store all its content and media files on Object Storage (like Amazons AWS S3, Google Cloud Storage or Microsoft Azure Storage) for persistence.

Kirby already has a mechanism called [Virtual Pages](https://getkirby.com/docs/guide/virtual-pages), which makes it possible to get portions of your content from object storage, databases, etc. However, with Virtual Pages it is not possible to have your whole content and media folder on object storage.

## Motivation / Why?
* Modern "serverless" infrastructure like "Platform as a Service" products (e.g. Google App Engine), often have no persistent file system storage available
* Storage limitations / very large content and media folders (Object storage can scale much easier, and can sometimes be cheaper)

## Possible solutions
### File System Abstraction Layer in Kirby
A naive approach to tackle this, would be to add a **file system abstraction layer (possibly in the form of an interface) to Kirby**, which could have different implementations for filesystem alternatives like a proper database (which is a [highly requested feature in the Kirby community](https://kirby.nolt.io/22)) or object storage. This is absolutely possible, but also requires a rather large modifications to Kirby, and will take some time to implement.

* 👍: Adaptable to other data layers (like a database)
* 👎: High effort

### Operating System Level Adapters
Another way to do this, are **operating system level adapters** to emulate a file system that is mapped to object storage (e.g. [GCS Fuse](https://cloud.google.com/storage/docs/gcs-fuse) or [JuiceFS](https://juicefs.com/))

* 👍: Turnkey solution, no code required
* 👎: Needs OS-level access/privileges – does not work for "Platform as a service" products

### Language Level Adapaters (PHP Stream Wrappers)
In this experiment I will focus on yet another approach, which combines many aspects of these two aforementioned solutions: **Adapter on the language level aka PHP Stream Wrappers**.

PHP offers a rather unknown solution to this problem right at the language level – it's called [Stream Wrappers](https://www.php.net/manual/en/wrappers.php). It allows us to make regular file system calls (e.g. `fopen()`, `fwrite()` or `file_get_contents()`) work with non-filesystem data layers – like an object storage.

* 👍: No OS privileges needed, runs on language level
* 👎: None (theoretically)

Thankfully, Object storage providers have already created stream wrapper implementations for PHP, which are ready to use ([Google Cloud Storage Wrapper](https://github.com/googleapis/google-cloud-php/tree/master/Storage#stream-wrapper), [Amazon S3 Wrapper](https://docs.aws.amazon.com/sdk-for-php/v3/developer-guide/s3-stream-wrapper.html))

## Problems while implementing Object Storage Stream Wrappers in Kirby
Theoretically, using Stream Wrappers to make Kirby work with object storage should only require a couple of lines of configuration/code – however it's not that easy.

### Registering a Stream Wrapper and Configuring Roots
After installing the Google Cloud Storage wrapper via `composer require google/cloud-storage` it can be registered in Kirbys `index.php`. To authenticate with the cloud storage, a service account with proper privileges (read and write) is needed. The keyfile for this service account needs to be added to the main directory.


Also, we need to adjust Kirby's root folders, so Kirby will use the proper, modified stream identifiers (`gs://`) for all file system calls internally.

```php
<?php

use Google\Cloud\Storage\StorageClient;

require __DIR__ . '/kirby/bootstrap.php';

function register_stream_wrapper()
{
  $client = new StorageClient([
    'keyFile' => json_decode(file_get_contents('./keyfile.json'), true)
  ]);
  $client->registerStreamWrapper();
}

register_stream_wrapper();

$kirby = new Kirby([
  'roots' => [
    'media' => 'gs://gcs-kirby-test/media',
    'content' => 'gs://gcs-kirby-test/content'
  ]
]);

echo $kirby->render();
```

In a perfect world, this would be it and Kirby should now work with the Object storage. But, there are some problems at this stage:

### Incompatibilities between Stream Wrappers and "actual" file systems

Unfortunately, there are some differences between actual file systems and stream wrapped file systems, so not **every** file system method is supported. [Google lists the unsupported methods](https://cloud.google.com/appengine/docs/standard/php/googlestorage/advanced#filesystem_functions_support_on_cloud_storage), which are:

```
chgrp,chmod,chown,disk_free_space,disk_total_space,diskfreespace,fileatime,filectime,filegroup,fileinode,fileowner,flock,fputs,ftruncate,is_executable,is_link,is_writeable,lchgrp,lchown,link,linkinfo,pclose,popen,readlink,realpath,set_file_buffer,symlink,touch
```

The reason why these are unsupported is not that the authors of the Stream Wrapper implementations at Amazon/Google/Microsoft are lazy, but rather that these methods conceptually do not make sense in an object storage context (e.g. there is no such thing as a `symlink` or `lock` or relative paths in object storage).

Most of these methods are not used in Kirby anyway, so they can be ignored. But Kirby does use some of these, namely:
* `realpath()`
* `flock()`
* `ftruncate()`
* `symlink()`
* `unlink()`

All these calls will fail when called with a `gs://` identifier as parameter. So all the locations where these methods are used, need to be patched/modified to make Kirby work with stream wrappers. I've identified about 10 files within the Kirby core, where these methods are called, that would need to be adjusted. Sometimes this is as easy as adding another check whether the given parameter begins with `gs://` and sometimes it is a little more complicated.



