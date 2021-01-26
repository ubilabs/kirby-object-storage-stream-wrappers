# Running Kirby CMS on Object Storage via PHP Stream Wrappers

## Problem
Kirby CMS is a popular flat file content management system written in PHP. Instead of a database, it uses the file system to store content and media. This makes it rather easy to setup and deploy on local dev machines, servers and webspace products. It would be nice, if Kirby was optionally also able to store all its content and media files on Object Storage (like Amazon S3 or Google Cloud Storage) for persistence.

## Motivation / Why?
* Modern "serverless" infrastructure like "Platform as a Service" products (e.g. Google App Engine), often have no persistent file system storage available
* Storage limitations / very large content and media files

## Possible solutions
A naive approach to tackle this, would be to add a **file system abstraction layer (possibly in the form of an interface) to Kirby**, which could have different implementations for filesystem alternatives like a proper database or object storage. This is still possible, but also requires a rather large modification of Kirby, and will take a lot of time.

* 👍: Adaptable to other implementations (like a database)
* 👎: High effort

Another way to do this, are **operating system level adapters** to emulate a file system for object storage (e.g. [GCS Fuse](https://cloud.google.com/storage/docs/gcs-fuse) or [JuiceFS](https://juicefs.com/))

* 👍: Turnkey solution, no code required
* 👎: Needs OS-level access/privileges – does not work for "Platform as a service" products


In this post I will focus on yet another approach, which is be somewhere between these two aforementioned solutions: **Adapter on the language level aka PHP Stream Wrappers**.

PHP offers a solution to this problem at the language level – it's called Stream Wrappers, and it is a solution to make file system calls (e.g. `fopen()`, `fwrite()` or `file_get_contents()`) work with non-filesystem data layers – like an Object storage.

* 👍: No OS privileges needed, runs on language level
* 👎: None (theoretically)
