---
layout: post
title:  Bazel behind the scenes
date:   2021-06-02 +0800
tags:   [iOS, Bazel]
---

### Show Bazel logs

Turn on bazel logs, we could take a closer look at how Bazel works, and do some analysis.

Here are some options to turn on the logs.

- --disk_cache=\<a path\>:
A path to a directory where Bazel can read and write actions and action outputs. If the directory does not exist, it will be created.

- --execution_log_binary_file=\<a path\>: 
Log the executed spawns into this file as delimited Spawn protos.

- --execution_log_json_file=\<a path\>:
Log the executed spawns into this file as json representation of the delimited Spawn protos.


Here I recommend to use following settings.
```
bazel build //BazelDemo --disk_cache=./local_cache --execution_log_json_file=./json.log
```

In the Bazel doc [Remote Caching](https://docs.bazel.build/versions/master/remote-caching.html), 
there will be 2 types of remote cache data.
- The action cache, which is a map of action hashes to action result metadata.
- A content-addressable store (CAS) of output files.

So the folder will also have 2 sub-foldes, `ac` and `cas`.
![](/assets/2021/cache-folder.png)

Take a look at the cas files by command: `file local_cache/cas/*/*`, it will shows result as
![](/assets/2021/cas.png)

Let's look into the json.log for more detail.

```
{
    "path": "bazel-out/ios-x86_64-min10.0-applebin_ios-ios_x86_64-fastbuild-ST-7bf/bin/
             BazelDemo/Main_objs/Main/MainViewController.swift.o",
    "digest": {
      "hash": "56e6bf92857ae12933140664180b6eaeec8d98dd43565b8277ec0509e4cd7005",
      "sizeBytes": "18368",
      "hashFunctionName": "SHA-256"
    }
  }
```

`MainViewController.swift` is compiled into `MainViewController.swift.o`, and is placed in path `cas/56/56e6bf92857ae12933140664180b6eaeec8d98dd43565b8277ec0509e4cd7005`

We could use `file` command to check the type

```
cmd: file -b local_cache/cas/56/56e6bf92857ae12933140664180b6eaeec8d98dd43565b8277ec0509e4cd7005
out: Mach-O 64-bit object x86_64
```

### Xcode Build Process

From WWDC 2018 video, the xcode build process could roughly split into compiling, processing, linking.

![](/assets/2021/tasks.png)

The compiler compiles source code into Mach object file (Mach-O)
![](/assets/2021/compiler.png)

The linker links .o files together
![](/assets/2021/linker.png)

So theoretical speaking, since compiling and linking output are all accessible from remote cache, the build time now depens on the network speed now.

### Remote Cache vs Network Speed

We could do a simple math, to roughly estimate the remote cache size under different network speed.
Given a huge application, say the cache files are around 3GB.

| Speed     | Calculation            | Download time   |
| --------- | -----------------------| --------------- |
| 10   Mbps | 3 * 1024 / (10   / 8)  | 2457s   (41m)   |
| 30   Mbps | 3 * 1024 / (30   / 8)  | 819s    (13.65m)|
| 50   Mbps | 3 * 1024 / (50   / 8)  | 491s    (8.192m)|
| 100  Mbps | 3 * 1024 / (100  / 8)  | 245.76s (4m)    |
| 1000 Mbps | 3 * 1024 / (1000 / 8)  | 24.576s (0.4m)  |


Thus network speed has important impact on the cache strategy.

https://github.com/bazelbuild/bazel/pull/7512
> his PR adds the ability to enable disk and HTTP cache simultaneously. Specifically, if you set
> ```
> build --remote_http_cache=http://some.remote.cache
> build --disk_cache=/some/disk/cache
> ```
> Then Bazel will look for cached items in the disk cache first. If an item is not found in the disk cache, it will be looked up in the HTTP cache. If it is found there, it will be copied into the disk cache. On put, Bazel will store items in both the disk and the HTTP cache.

https://github.com/bazelbuild/bazel/issues/7664
> Suggestion
> Perhaps the HTTP cache should record the time it took to build an artefact (according to the client). This would give Bazel enough information to decide if it is better to build or fetch.