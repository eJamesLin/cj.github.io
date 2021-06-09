---
layout: post
title:  Bazel cache behind the scenes
date:   2021-06-09 +0800
tags:   [iOS, Bazel]
---

Bazel did a great help on speedup build by remote cache, let's see how it works behind the scenes.

### Show Bazel logs

Turn on bazel logs, we could take a closer look at how Bazel works, and do some analysis.

Here are some options to turn on the logs.

- --disk_cache=\<a path\>:
A path to a directory where Bazel can read and write actions and action outputs. If the directory does not exist, it will be created.

- --execution_log_binary_file=\<a path\>: 
Log the executed spawns into this file as delimited Spawn protos.

- --execution_log_json_file=\<a path\>:
Log the executed spawns into this file as json representation of the delimited Spawn protos.


Let's try the following command.
```
bazel build //BazelDemo --disk_cache=./local_cache --execution_log_json_file=./json.log
```

Based on [Bazel document](https://docs.bazel.build/versions/master/remote-caching.html), 
there are 2 types of cache data.
- The action cache, which is a map of action hashes to action result metadata.
- A content-addressable store (CAS) of output files.

So the folder will also have 2 sub-folders, `ac` and `cas`.
![](/assets/2021/cache-folder.png)

Take a look at the cas files by command: `file local_cache/cas/*/*`, it will shows result as
![](/assets/2021/cas.png)

Before dive right in to see what are these, let's read more about the build process.

### Xcode Build Process

From [2018 WWDC - Behind The Scenes of The Xcode Build Process](https://developer.apple.com/videos/play/wwdc2018/415/), the xcode build process could roughly split into compiling, processing, linking.

![](/assets/2021/tasks.png)

The compiler compiles source code into Mach object file (Mach-O)
![](/assets/2021/compiler.png)

The linker links .o files together as final executable output.
![](/assets/2021/linker.png)

### Cache and Bazel logs

Let's look into the json.log for more detail.

```
{
  "commandArgs": ["bazel-out/host/bin/external/build_bazel_rules_swift/tools/worker/worker", "swiftc", "@bazel-out/ios-x86_64-min10.0-applebin_ios-ios_x86_64-fastbuild-ST-7bf874b56ea0/bin/BazelDemo/BazelDemo_Main.swiftmodule-0.params"],
  "environmentVariables": [{
    "name": "APPLE_SDK_PLATFORM",
    "value": "iPhoneSimulator"
  }, {
    "name": "APPLE_SDK_VERSION_OVERRIDE",
    "value": "14.5"
  }, {
    "name": "XCODE_VERSION_OVERRIDE",
    "value": "12.5.0.12E262"
  }],
  "platform": {
    "properties": []
  },
  "inputs": [{
    "path": "BazelDemo/Main/MainViewController.swift",
    "digest": {
      "hash": "783a878a90944bd0f269119011f3861f67c4310fc885ad2879072e5f31e43f98",
      "sizeBytes": "303",
      "hashFunctionName": "SHA-256"
    }
  }
  ...
  ],
  "remotable": true,
  "cacheable": true,
  "timeoutMillis": "0",
  "progressMessage": "Compiling Swift module BazelDemo_Main",
  "mnemonic": "SwiftCompile",
  "actualOutputs": [{
    "path": "bazel-out/ios-x86_64-min10.0-applebin_ios-ios_x86_64-fastbuild-ST-7bf874b56ea0/bin/BazelDemo/Main_objs/Main/MainViewController.swift.o",
    "digest": {
      "hash": "520036179d8990f3bce6171ded1c78f4c6cb440b2b30b5cc3ff2d73a759cf176",
      "sizeBytes": "16232",
      "hashFunctionName": "SHA-256"
    }
  }],
  "runner": "remote cache hit",
  "remoteCacheHit": true,
  "status": "",
  "exitCode": 0
}
```

`MainViewController.swift` is compiled into `MainViewController.swift.o`, and is placed in path `cas/52/520036179d8990f3bce6171ded1c78f4c6cb440b2b30b5cc3ff2d73a759cf176`

We could use `file` command to check the type is indeed Mach-O.

```
cmd: file -b local_cache/cas/52/520036179d8990f3bce6171ded1c78f4c6cb440b2b30b5cc3ff2d73a759cf176
out: Mach-O 64-bit object x86_64
```

Also linking are shown in the log.

```
{
  "inputs": [
    {
      "path": "bazel-out/ios-x86_64-min10.0-applebin_ios-ios_x86_64-fastbuild-ST-7bf874b56ea0/bin/BazelDemo/libMain.a",
      "digest": {
        "hash": "887798d465951df90b0a55202e342a610f8dc5c787ef48597e77503bb5bb7a58",
        "sizeBytes": "73600",
        "hashFunctionName": "SHA-256"
      }
    },
    ...
  ],
  "listedOutputs": [
    "bazel-out/ios-x86_64-min10.0-applebin_ios-ios_x86_64-fastbuild-ST-7bf874b56ea0/bin/BazelDemo/BazelDemo_bin"
  ],
  "remotable": true,
  "cacheable": true,
  "progressMessage": "Linking BazelDemo/BazelDemo_bin",
  "actualOutputs": [
    {
      "path": "bazel-out/ios-x86_64-min10.0-applebin_ios-ios_x86_64-fastbuild-ST-7bf874b56ea0/bin/BazelDemo/BazelDemo_bin",
      "digest": {
        "hash": "edc029cc3263da7a4468f693fedafdc448aa6824eb8c45535c305b5c58cfe1b4",
        "sizeBytes": "91264",
        "hashFunctionName": "SHA-256"
      }
    }
  ],
  "runner": "darwin-sandbox",
  "remoteCacheHit": false,
}
```

The linker links compiled object into final executable output.

### Nginx local cache server

For easier observe what's happening on cache upload and download, we could host a local [nginx](https://docs.bazel.build/versions/master/remote-caching.html#nginx) server to act as a remote cache server.

```
brew install nginx-full --with-webdav
```

Modify the `/usr/local/etc/nginx/nginx.conf`, add the following configuration in server section. 
```
location /cache/ {
  # The path to the directory where nginx should store the cache contents.
  root /path/to/cache/dir;
  # Allow PUT
  dav_methods PUT;
  # Allow nginx to create the /ac and /cas subdirectories.
  create_full_put_path on;
  # The maximum size of a single file.
  client_max_body_size 1G;
  allow all;
}
```

Nginx commands are listed here.

```
start: nginx
quit: nginx -s quit
reload: nginx -s reload
```

Configure the `.bazelrc` with option `build --remote_cache=http://127.0.0.1:8080/cache`, then we could take a look about how the cache is uploaded and downloaded.

From the access log, we could see a series of GET and PUT request for `ac` and `cas`. 
The following shows the first time build, some `ac` is first not found, and then upload to remote cache.
Since it involves useless query and cache upload, the first time Bazel build will need longer time than normal build.
Usually this tough works will done by CI server.
```
127.0.0.1 "GET /cache/ac/3ab82a7f12a5eb6842c6f8bb05c43faa6993b14302f5076a823f4f3e356bd609 404 "bazel/4.1.0"
127.0.0.1 "PUT /cache/cas/e380a98f73ae0aaa75da463cbe01f1b2a62c44d3545b0e7d2f3eea2c5c58b91c 201 "bazel/4.1.0" 
127.0.0.1 "PUT /cache/cas/3ab82a7f12a5eb6842c6f8bb05c43faa6993b14302f5076a823f4f3e356bd609 201 "bazel/4.1.0" 
127.0.0.1 "PUT /cache/ac/3ab82a7f12a5eb6842c6f8bb05c43faa6993b14302f5076a823f4f3e356bd609 201 "bazel/4.1.0"
127.0.0.1 "GET /cache/ac/c248cf795732128f164e0662671490717a3139fdfd8541c05addb9bc956cb4d2 404 "bazel/4.1.0"
127.0.0.1 "PUT /cache/cas/a3216a3e18968b28e49e070b95e5990cd7cbd9d3891a894fac76feb6a86bfde1 201 "bazel/4.1.0" 
127.0.0.1 "PUT /cache/cas/b97438a55e4ca1c247254406e12e99efe565b15444a82832b5244121342aa0fe 201 "bazel/4.1.0" 
127.0.0.1 "PUT /cache/cas/520036179d8990f3bce6171ded1c78f4c6cb440b2b30b5cc3ff2d73a759cf176 201 "bazel/4.1.0" 
```

The following will be the case that, for a local build which use the speedup benefit from remote cache server, `ac` and `cas` could be downloaded.
Usually we have Bazel option `--noremote_upload_local_results` for the local build.
```
127.0.0.1 "GET /cache/ac/3ab82a7f12a5eb6842c6f8bb05c43faa6993b14302f5076a823f4f3e356bd609 200 "bazel/4.1.0" 
127.0.0.1 "GET /cache/cas/e380a98f73ae0aaa75da463cbe01f1b2a62c44d3545b0e7d2f3eea2c5c58b91c 200 "bazel/4.1.0"
127.0.0.1 "GET /cache/ac/c248cf795732128f164e0662671490717a3139fdfd8541c05addb9bc956cb4d2 200 "bazel/4.1.0" 
127.0.0.1 "GET /cache/cas/df5579bb0f6c552be91d85d560e4807e64e594a881f817a300d1d69554a8caa7 200 "bazel/4.1.0"
127.0.0.1 "GET /cache/cas/c5d4f12748a7e7ddb49219a73e8fda01d1938af7f2ce0612554212c35bf174c4 200"bazel/4.1.0" 
127.0.0.1 "GET /cache/cas/54f2001806addd837ad2d152f42af9847d47f337e14741c756074abe1bcdbd6f 200"bazel/4.1.0" 
127.0.0.1 "GET /cache/cas/b97438a55e4ca1c247254406e12e99efe565b15444a82832b5244121342aa0fe 200 "bazel/4.1.0"
127.0.0.1 "GET /cache/cas/520036179d8990f3bce6171ded1c78f4c6cb440b2b30b5cc3ff2d73a759cf176 200 "bazel/4.1.0"
127.0.0.1 "GET /cache/cas/c6ff3cbec15a2ee40cb9ccf8d97f26d416849dccffc837e95abaaab385d3b591 200 "bazel/4.1.0"
127.0.0.1 "GET /cache/cas/a3216a3e18968b28e49e070b95e5990cd7cbd9d3891a894fac76feb6a86bfde1 200 "bazel/4.1.0"
```

And if we take a look at the terminal log, the former one will have low remote cache hit rate.

While the latter have high hit rate as: `INFO: 264 processes: 216 remote cache hit, 47 internal, 1 local.`

To simulate different internet speed effect the Bazel cache, we could add access limit on Nginx. 

For example, limit the max request number to 10 request per second, add the following option on config file.
```
limit_req_zone $binary_remote_addr zone=one:10m rate=10r/s;
```
Then the http request to local cache server slow down, and we could see that Bazel build slow down as well.

### Remote Cache vs Download Speed

We could do a simple math, to roughly estimate the remote cache size under different download speed.
Given a huge application, say the sum of cache sizes is around 3GB.

| Speed     | Calculation            | Download time   |
| --------- | -----------------------| --------------- |
| 10   Mbps | 3 * 1024 / (10   / 8)  | 2457s   (41m)   |
| 20   Mbps | 3 * 1024 / (20   / 8)  | 1229s   (20.48m)|
| 30   Mbps | 3 * 1024 / (30   / 8)  | 819s    (13.65m)|
| 50   Mbps | 3 * 1024 / (50   / 8)  | 491s    (8.192m)|
| 100  Mbps | 3 * 1024 / (100  / 8)  | 245.76s (4m)    |

This is really a rough estimation, since the download process involves a series of GET request, the overhead will definitely takes longer time.

### Remote cache always help?

We could reference this [issue on github](https://github.com/bazelbuild/bazel/issues/7664). 
The reporter has a slow internet connection speed, and a high-end laptop.
So the pure local build is actually faster then build with remote cache.

Here I quote his suggestion.
> Perhaps the HTTP cache should record the time it took to build an artefact (according to the client). This would give Bazel enough information to decide if it is better to build or fetch.

Though there seems have some useful Bazel options suggested by this [pull request](https://github.com/bazelbuild/bazel/pull/7512).
> This PR adds the ability to enable disk and HTTP cache simultaneously. Specifically, if you set
> ```
> build --remote_http_cache=http://some.remote.cache
> build --disk_cache=/some/disk/cache
> ```
> Then Bazel will look for cached items in the disk cache first. If an item is not found in the disk cache, it will be looked up in the HTTP cache. If it is found there, it will be copied into the disk cache. On put, Bazel will store items in both the disk and the HTTP cache.

This helps, but not helping the dynamic strategy on local build or fetch cache, depends on computing power and internet condition.
I used to develop on slow internet condition with only mobile hotspot, thus I did experience that sometimes turn off remote cache is faster. 
Sincerely hope there are some improvements in the future :p
