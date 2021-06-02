---
layout: post
title:  Bazel behind the scenes
date:   2021-06-02 +0800
tags:   [iOS, Bazel]
---

To take a closer look at how Bazel works, we could view the log.

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

From WWDC, the xcode build process could roughly split into compiling, processing, linking.

![](/assets/2021/tasks.png)

The compiler compiles source code into Mach object file (Mach-O)
![](/assets/2021/compiler.png)

The linker links .o files together
![](/assets/2021/linker.png)

So theoretical speaking, since compiling and linking output are all accessible from remote cache, the build time now depens on the network speed now.
