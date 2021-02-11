# Running Kirby CMS on Object Storage via PHP Stream Wrappers

⚠️ I wrote this summary specifically for Kirby CMS and Google Cloud Storage – however the content should be applicable to other comparable products. Both, for other flat-file CMS and for other object storage providers.

## Problem
[Kirby CMS](https://github.com/getkirby/kirby) is a popular flat file content management system written in PHP. Instead of a database, it uses the file system to store content and media. This makes it rather easy to setup and deploy on local dev machines, servers and webspace products. It would be nice, if Kirby was optionally also able to store all its content and media files on object storage (like Amazon AWS S3, Google Cloud Storage or Microsoft Azure Storage) for persistence.

Kirby already has a mechanism called [Virtual Pages](https://getkirby.com/docs/guide/virtual-pages), which allows to get parts of your content from object storage, databases, etc. However, with virtual pages it is not possible to have your whole content and media folder on object storage, so this will not be covered here.

## Motivation / Why?
* Modern "serverless" infrastructure like "Platform as a Service" products (e.g. Google App Engine), often have no persistent file system storage available
* Storage limitations / very large content and media folders (Object storage can scale much easier, and can sometimes even be cheaper)

## Possible solutions
### File System Abstraction Layer in Kirby
A naive approach to tackle this, would be to add a **file system abstraction layer (possibly in the form of an interface) to Kirby**, which could have different implementations for filesystem alternatives like object storage or a proper relational database (which is a [highly requested feature in the Kirby community](https://kirby.nolt.io/22)). This is absolutely possible, but also requires a rather large modifications to Kirby, and will take some time to implement.

* 👍: Adaptable to other data layers (like a database)
* 👎: High effort

### Operating System Level Adapters
Another way to do this, are **operating system level adapters** to emulate a file system that is mapped to object storage (e.g. [GCS Fuse](https://cloud.google.com/storage/docs/gcs-fuse) or [JuiceFS](https://juicefs.com/))

* 👍: Turnkey solution, no code required
* 👎: Needs OS-level access/privileges – does not work for "Platform as a service" or simple webspace hosting

### Language Level Adapaters (PHP Stream Wrappers)
In this experiment I will focus on yet another approach, which combines many aspects of these two aforementioned solutions: **Adapter on the language level aka PHP Stream Wrappers**.

PHP offers a rather unknown solution to this problem right at the language level – it's called [Stream Wrappers](https://www.php.net/manual/en/wrappers.php). It allows us to make regular file system calls (e.g. `fopen()`, `fwrite()` or `file_get_contents()`) work with non-filesystem data layers – like an object storage or a database.

* 👍: No OS privileges needed, implemented on language level
* 👎: None (theoretically)

Thankfully, Object storage providers have already created stream wrapper implementations for PHP, which are ready to use ([Google Cloud Storage Wrapper](https://github.com/googleapis/google-cloud-php/tree/master/Storage#stream-wrapper), [Amazon S3 Wrapper](https://docs.aws.amazon.com/sdk-for-php/v3/developer-guide/s3-stream-wrapper.html))

## Problems while implementing Object Storage Stream Wrappers in Kirby
Theoretically, using Stream Wrappers to make Kirby work with object storage should only require a couple of lines of configuration/code – but unfortunately it's not that easy.

### Registering a Stream Wrapper and Configuring Roots
I'm using a Kirby starterkit for this experiment. After downloading/cloning the code, we need to install the Google Cloud Storage wrapper via `composer require google/cloud-storage` and register it in Kirbys `index.php`. To authenticate with the cloud storage, a service account with proper privileges (read and write) is needed. The keyfile for this service account needs to be added to the root directory.

Also, we need to adjust Kirby's root folders, so Kirby will use the proper, modified stream identifiers (`gs://`) for all file system calls internally instead of regular paths.

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

In a perfect world, we should be done now and Kirby would work perfectly with the Object storage. But, there are some problems at this stage:

### Incompatibilities between Stream Wrappers and "actual" file systems

Turns out, there are some differences between actual file systems and stream wrapped file systems, so not **every** file system method is supported. [Google lists all the unsupported methods](https://cloud.google.com/appengine/docs/standard/php/googlestorage/advanced#filesystem_functions_support_on_cloud_storage) as:

```
chgrp,chmod,chown,disk_free_space,disk_total_space,diskfreespace,fileatime,filectime,filegroup,fileinode,fileowner,flock,fputs,ftruncate,is_executable,is_link,is_writeable,lchgrp,lchown,link,linkinfo,pclose,popen,readlink,realpath,set_file_buffer,symlink,touch
```

⚠️ The reason why these are unsupported is not that the authors of the Stream Wrapper implementations at Amazon/Google/Microsoft are lazy, but rather that these methods **conceptually do not make sense in an object storage context. For example there is no such thing as a `lock` in object storage. Concurrent writes to the object storage will result in multiple revisions of the same file – no need for locking. Similarly the concepts of `symlink` or a relative path make no sense on object storage.**

Most of the unsupported methods are not used in Kirby anyway, so they can be ignored. But Kirby does use some of these, namely:
* `realpath()`
* `flock()`
* `ftruncate()`
* `symlink()`
* `unlink()`

All these calls will fail when called with a `gs://` identifier as parameter. So all the locations where these methods are used, need to be patched/modified in order to make Kirby work with stream wrappers. I've identified about 10 files within the Kirby core, where these methods are called, that would need to be adjusted. Sometimes this is as easy as adding another check whether the given parameter begins with `gs://` and sometimes it is a little more complicated.

Unfortunately the necessary modifications to these method calls can not be applied via a Kirby plugin (b/c there is no mechanism to hook into Kirbys internal file system calls). **Modification of Kirbys core source code is necessary for this to work.** But all of these problems will also arise, once Kirby decides to adopt any kind of file system abstraction layer (a database has many of the same conceptional drawbacks that object storag hase – e.g. no symlinks or locks). So these modifications might need to get tackled anyway, if Kirby wants to move to a more modular/adaptable file system approach in the future.

I tried to manually patch all unsupported file system methods and got a working Kirby starterkit as a result, even though this is very brittle. Stuff might break in areas that I haven't tested, yet. So I would not advise you to recreate this – this is just an experiment and not usable code. The modified code can be found in [kirby_object_storage.patch](kirby_object_storage.patch).

## It works! But it's really slow.

Now, after applying the listed changes to the Kirby core, you will have a Kirby instance that gets its content and media from Object Storage, but you will also notice **that it's incredibly slow (page load times between 30 and 60 seconds)**. I'm not entirely sure, what the reason for this is. Object storage is slow compared to a file system (typically a couple hundred ms to request text files from Object storage), but not THAT slow. Some more debugging is needed to find out, why it's so incredible slow. One of the reasons is a conceptual problem: Kirby still serves as a Proxy for your requests: Kirby still uses its own URL for assets, e.g. `http://kirby.test/media/pages/photography/sky/b13735b7f3-1611658788/coconut-milkyway.jpg`, not the public link for the object storage location. Unfortunately, we need to do it this way, so Kirby can still trigger thumbnail generation, when a thumb is requested. If we were to directly embed the storage URL, no thumbnails would be generated (we could probably fix this by [generating thumbnails on upload](https://github.com/bnomei/kirby3-janitor/wiki/Setup:-Thumbs-on-Upload), e.g. via the Kirby janitor plugin).

### Getting the panel to work
Unlike the rest of Kirby, the panel assets (JS and CSS files) are normally served statically from the filesystem. Kirby just uses the `$assetUrl` variable in the templates as a link to the static files. Since we're not serving them from the local filesystem anymore, but directly from the Object storage, we need to modify `$assetUrl` in `kirby\src\Cms\Panel.php` and adjust the `panel` link in `index.php`.

```diff
diff --git a/kirby/src/Cms/Panel.php b/kirby/src/Cms/Panel.php
index 4941a76..851f5de 100755
--- a/kirby/src/Cms/Panel.php
+++ b/kirby/src/Cms/Panel.php
@@ -114,7 +114,7 @@ class Panel
         $view = new View($kirby->root('kirby') . '/views/panel.php', [
             'kirby'     => $kirby,
             'config'    => $kirby->option('panel'),
-            'assetUrl'  => $kirby->url('media') . '/panel/' . $kirby->versionHash(),
+            'assetUrl'  => $kirby->url('panel') .'/'. $kirby->versionHash(),
             'customCss' => static::customCss($kirby),
             'icons'     => static::icons($kirby),
             'pluginCss' => $plugins->url('css'),
```
```php
$kirby = new Kirby([
  'roots' => [
    'media' => 'gs://gcs-kirby-test/media',
    'content' => 'gs://gcs-kirby-test/content'
  ],
  'urls' => [
    'panel' => 'https://storage.googleapis.com/gcs-kirby-test/media/panel'
  ]
]);
```

## Summary
Is it possible to get Kirbys content and media folders from object storage? Yes! Is it practical, yet? No! To make this process easier, Kirby needs to make some changes to its core, so we can change the implementation of unsupported file system calls for object storage. As soon as there is such a file system abstraction (like a PHP interface), it will be pretty easy to implement this functionality with Stream Wrappers via a Plugin and make it usable for everyone.



