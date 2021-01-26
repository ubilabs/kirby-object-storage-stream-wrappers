# Running Kirby CMS on Object Storage via PHP Stream Wrappers

## Problem
[Kirby CMS](https://github.com/getkirby/kirby) is a popular flat file content management system written in PHP. Instead of a database, it uses the file system to store content and media. This makes it rather easy to setup and deploy on local dev machines, servers and webspace products. It would be nice, if Kirby was optionally also able to store all its content and media files on Object Storage (like Amazon S3 or Google Cloud Storage) for persistence.

## Motivation / Why?
* Modern "serverless" infrastructure like "Platform as a Service" products (e.g. Google App Engine), often have no persistent file system storage available
* Storage limitations / very large content and media files

## Possible solutions
### File System Abstraction Layer in Kirby
A naive approach to tackle this, would be to add a **file system abstraction layer (possibly in the form of an interface) to Kirby**, which could have different implementations for filesystem alternatives like a proper database (which is a [highly requested feature in the Kirby community](https://kirby.nolt.io/22)) or object storage. This is absolutely possible, but also requires a rather large modifications to Kirby, and will take some time to implement.

* 👍: Adaptable to other implementations (like a database)
* 👎: High effort

### Operating System Level Adapters
Another way to do this, are **operating system level adapters** to emulate a file system that is mapped to object storage (e.g. [GCS Fuse](https://cloud.google.com/storage/docs/gcs-fuse) or [JuiceFS](https://juicefs.com/))

* 👍: Turnkey solution, no code required
* 👎: Needs OS-level access/privileges – does not work for "Platform as a service" products

### Language Level Adapaters (PHP Stream Wrappers)
In this experiment I will focus on yet another approach, which combines many aspects of these two aforementioned solutions: **Adapter on the language level aka PHP Stream Wrappers**.

PHP offers a solution to this problem right at the language level – it's called [Stream Wrappers](https://www.php.net/manual/en/wrappers.php). It allows us to make regular file system calls (e.g. `fopen()`, `fwrite()` or `file_get_contents()`) work with non-filesystem data layers – like an Object storage.

* 👍: No OS privileges needed, runs on language level
* 👎: None (theoretically)

Thankfully, Object storage providers have already created stream wrapper implementations for PHP, which are ready to use ([Google Cloud Storage Wrapper](https://github.com/googleapis/google-cloud-php/tree/master/Storage#stream-wrapper), [Amazon S3 Wrapper](https://docs.aws.amazon.com/sdk-for-php/v3/developer-guide/s3-stream-wrapper.html))
