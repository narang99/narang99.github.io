---
layout: post
title: container_run_and_commit with rules_oci
subtitle: Fake the RUN Dockerfile instruction with bazel
tags: [bazel, docker]
---

If you have used bazel, chances are, you are using it to generate your docker images. There are two rules which enable this: [rules_docker](https://github.com/bazelbuild/rules_docker) and [rules_oci](https://github.com/bazel-contrib/rules_oci)   
`rules_docker` is no longer maintained, it does not support the Bazel 8 module system. Due to this, at qure.ai, we had to migrate our `rules_docker` implementations to `rules_oci`. A fundamental blocker is the lack of support of `container_run_and_commit` and friends (equivalent to `RUN` inside a `Dockerfile`). This blog post describes the rule implementation which was ported from `rules_docker` with minor patches (its a very basic and rough implementation, without any toolchains support).  
Note that the code in this blog is not exhaustive (it lacks cleanup and is closer to pseudocode). You can find the full code at [my github repo](https://github.com/narang99/oci_container_run_and_commit)

> A `RUN` instruction is fundamentally non-hermetic (it can have unintended side-effects [like making internet calls] which bazel has no way of knowing about). It does not fit well with bazel's philosophy of deterministic package building. This is the main reason `rules_oci` does not provide `container_run_and_commit`. You should always look at alternative solutions before using this rule :)   


Some assumptions about the reader:
- you have made docker images from bazel before
- have written some rules in starlark
- know that docker/OCI images are simply collection of tar files


# Usage
Lets first look at how you would use this rule (this is a contrived example, don't use this rule for something trivial like this)  
```python
oci_run_and_commit(
    name="nginx_with_conf_from_internet",
    base=":base_nginx",
    commands=[
        "curl https://github.com/my-cool-nginx-conf -o /etc/nginx/nginx.conf",
        "echo 'using my cool conf'"
    ],
    visibility=["//visibility:public"],
    entrypoint=["nginx"],
)
```
This rule expects a target of type `oci_image` as base, and a list of strings as commands.  
A new image would be created by committing the result of commands on the base image, the target type is `oci_image`. Arguments other than `name`, `base`, `commands` and `tars` are passed to the final `oci_image` target  

Approximately similar to creating a `Dockerfile` with the content below
```Dockerfile
# actual docker image referenced by :base_nginx
FROM nginx

RUN curl https://github.com/my-cool-nginx-conf -o /etc/nginx/nginx.conf && \
    echo 'using my cool conf'
```
And docker building this `Dockerfile`

# The top level rule macro implementation

How this works is:
1. load the `oci_image` target as a tarball which we can docker load
2. start a container from this image, run `commands`, commit the container and extract the last layer which we overlay on the base image
3. instantiate a new `oci_image` with `base=base` and `tars=[<last-extracted-layer>]`

```python
load(":run.bzl", "container_run_and_commit_layer")
load("@rules_oci//oci:defs.bzl", "oci_image", "oci_push", "oci_load")

def oci_run_and_commit(name, base, commands, **kwargs):
    ############################## step 1 ######################################################
    load_name = "{name}_base_load".format(name=name)
    # load the docker image, with a deterministic image name
    # note that invoking the target multiple times from different workspaces in the same machine is not concurrency safe
    oci_load(
        name = load_name,
        image = base,
        repo_tags = ["{load_name}:latest".format(load_name=load_name)],
    )
    # this provides a single tar for the image from the previous oci_load 
    # which can be easily loaded using `docker load` inside our implementation
    # reference: https://github.com/bazel-contrib/rules_oci/blob/main/docs/load.md#build-outputs
    tar_name = "{name}_base_tar".format(name=name)
    native.filegroup(
        name = tar_name,
        srcs = [":{load_name}".format(load_name=load_name)],
        output_group = "tarball",
    )


    ############################## step 2 ######################################################
    # load the docker tar `image`
    # run `commands`
    # commit the image
    # extract the last layer from the committed image (our layer)
    # set that layer as the build result
    commit_layer_name = "{name}_commit_layer".format(name=name)
    container_run_and_commit_layer(
        name = commit_layer_name,
        image = ":{tar_name}".format(tar_name=tar_name),
        commands = commands,
    )


    ############################## step 3 ######################################################
    # use the layer from the container_run_and_commit_layer as additional tars
    # done on top of the passed base image
    if "tars" in kwargs:
      fail("passing `tars` to kwargs is not valid, use other oci_image options")
    oci_image(
        name = name,
        base = base,
        tars = [":{commit_layer_name}".format(commit_layer_name=commit_layer_name)],
        **kwargs,
    )
```


# container_run_and_commit_layer

This rule will create a new OCI layer for you, consider it the equivalent layer of a `RUN` instruction in Dockerfile.  
In our top-level macro, we simply use this layer as a dependency in downstream `oci_image` rule  

How do we do this? The algorithm is pretty simple:
1. Bazel would pass us a tar of the base docker image. `docker load` this tar
2. Start a container, attach to it and run the commands passed by the user  
3. commit the container to generate a new image, dump the image using `docker save`
  - This basically creates a new docker image
  - The main difference from the base image would be that it would have **one more layer**, the layer containing the files resulting from our `commands`
4. read the tarball, find the location of the last layer, and extract it


A basic script would look like this
```bash
# commit_layer.sh

# BASE_IMAGE_TAR: passed by bazel
docker load -i "$BASE_IMAGE_TAR"
base_image_id=$(get_image_id "$BASE_IMAGE_TAR")

# USER_COMMANDS: passed by bazel
user_commands="sh -c '$USER_COMMANDS'"
# start the container and get its unique id
id=$(docker create "$base_image_id" "$user_commands")
# start and attach to the container
# this will run `<user-commands>` and wait for them to end
docker start -a "$id"

intermediate_image_name="bazel_intermediate_image:latest"
# commit the container to some intermediate image tag
docker commit "$id" "$intermediate_image_name"

saved_image_tar_loc="full_image.tar"
# save to a tar
docker save "$intermediate_image_name" -o "$saved_image_tar_loc"

# CONTAINER_LAYER_DEST: passed by bazel
extract_last_layer "$save_image_tar_loc" "$CONTAINER_LAYER_DEST"
```

Now we need to implement `get_image_id` and `extract_last_layer`   

## get_image_id and extract_last_layer

We need the image ID to actually run the container, `docker load` does not output it sadly.  
It is quite easy to do so if you understand how docker image tarballs look  

### docker tarball structure
Lets download `nginx:1.27.4`, save it and check out the tarball

```bash
docker pull nginx:1.27.4
docker save nginx:1.27.4 -o ./nginx.tar
tar -xvf nginx.tar
```

This is how it looks
```text
tree
.
├── blobs
│   └── sha256
│       ├── 22ca876b5b0cc85b878846c3e22eb3bcab375dd558fff64ee58c5348b8bbd12c
│       ├── 2457786386eb0fcdc50b0a7d52a1f19aa1a0e209bea76551d907d8b12e044e2c
│       ├── 2c9168b3c9a84851f91e03534dc4136951e9f581ab3ac8ee38b28b49ad57ba38
│       ├── 4384cff370110fa8863fd999f9aca2fa7a39d2fa4c2cd7e33650cdddcb7c668d
│       ├── 51c7337ac2b1f2febd4244289112a014d45edb5cbddde1ea26b1e0bb1982dcf8
│       ├── 5a5b8ab342444446d1de9d9c12f9b005efa1e7e1cf4d7ce390dabffb95bd4504
│       ├── 6d09d60c049b16fc12f4293535b7a1ef323bfde20838496d73669f56298b99b6
│       ├── 70a3ee4d4d38a5bb57ecd08c40c9a37d750d3f8f63591e97cdbf8277c3698e6f
│       ├── af48c25a7cd590191590ab81fb5de0541c90da4d87b1362030d52a31dd5656dd
│       ├── cb4262b4d29a1d258cd2cbe6df59bd95a2660700bdd6d73774209fe5a6ea4c4d
│       ├── d442fff664a9d339643a8719130f3c7902642ac109fea73e75cddeed76e8a154
│       ├── d4a08af0f696e966542052d30ed46209cc35c7159ddee06a02cd461b4d8ecc3c
│       ├── e530e4daca22b3f75c37b856380a8942995c4a1289ff29757414a9b44485919a
│       ├── e7baeb0871dab5b9adba53f721c6c500ef48bae7c4d2cb7e47f4c57bb9a8f3dd
│       ├── ee7940daa82405e7e5327c8fd3ca7ddcb7933ecc44bd15f2218670cd366b7950
│       └── f35cd1ab96625b73dbb9d6e58c50055c4405a4c4aa033bd34a8b2369c9be7c1d
├── index.json
├── manifest.json
├── oci-layout
└── repositories
```

The `manifest.json` file describes all the layers inside the docker image. It is a metadata store  
This is how it looks

```json
[
    {
        "Config": "blobs/sha256/2c9168b3c9a84851f91e03534dc4136951e9f581ab3ac8ee38b28b49ad57ba38",
        "RepoTags": [
            "nginx:1.27.4"
        ],
        "Layers": [
            "blobs/sha256/70a3ee4d4d38a5bb57ecd08c40c9a37d750d3f8f63591e97cdbf8277c3698e6f",
            "blobs/sha256/51c7337ac2b1f2febd4244289112a014d45edb5cbddde1ea26b1e0bb1982dcf8",
            "blobs/sha256/d442fff664a9d339643a8719130f3c7902642ac109fea73e75cddeed76e8a154",
            "blobs/sha256/2457786386eb0fcdc50b0a7d52a1f19aa1a0e209bea76551d907d8b12e044e2c",
            "blobs/sha256/5a5b8ab342444446d1de9d9c12f9b005efa1e7e1cf4d7ce390dabffb95bd4504",
            "blobs/sha256/f35cd1ab96625b73dbb9d6e58c50055c4405a4c4aa033bd34a8b2369c9be7c1d",
            "blobs/sha256/cb4262b4d29a1d258cd2cbe6df59bd95a2660700bdd6d73774209fe5a6ea4c4d"
        ],
        "LayerSources": {
            "sha256:2457786386eb0fcdc50b0a7d52a1f19aa1a0e209bea76551d907d8b12e044e2c": {
                "mediaType": "application/vnd.oci.image.layer.v1.tar",
                "size": 4608,
                "digest": "sha256:2457786386eb0fcdc50b0a7d52a1f19aa1a0e209bea76551d907d8b12e044e2c"
            },
            "sha256:51c7337ac2b1f2febd4244289112a014d45edb5cbddde1ea26b1e0bb1982dcf8": {
                "mediaType": "application/vnd.oci.image.layer.v1.tar",
                "size": 101223936,
                "digest": "sha256:51c7337ac2b1f2febd4244289112a014d45edb5cbddde1ea26b1e0bb1982dcf8"
            },
            "sha256:5a5b8ab342444446d1de9d9c12f9b005efa1e7e1cf4d7ce390dabffb95bd4504": {
                "mediaType": "application/vnd.oci.image.layer.v1.tar",
                "size": 2560,
                "digest": "sha256:5a5b8ab342444446d1de9d9c12f9b005efa1e7e1cf4d7ce390dabffb95bd4504"
            },
            "sha256:70a3ee4d4d38a5bb57ecd08c40c9a37d750d3f8f63591e97cdbf8277c3698e6f": {
                "mediaType": "application/vnd.oci.image.layer.v1.tar",
                "size": 100126720,
                "digest": "sha256:70a3ee4d4d38a5bb57ecd08c40c9a37d750d3f8f63591e97cdbf8277c3698e6f"
            },
            "sha256:cb4262b4d29a1d258cd2cbe6df59bd95a2660700bdd6d73774209fe5a6ea4c4d": {
                "mediaType": "application/vnd.oci.image.layer.v1.tar",
                "size": 7168,
                "digest": "sha256:cb4262b4d29a1d258cd2cbe6df59bd95a2660700bdd6d73774209fe5a6ea4c4d"
            },
            "sha256:d442fff664a9d339643a8719130f3c7902642ac109fea73e75cddeed76e8a154": {
                "mediaType": "application/vnd.oci.image.layer.v1.tar",
                "size": 3584,
                "digest": "sha256:d442fff664a9d339643a8719130f3c7902642ac109fea73e75cddeed76e8a154"
            },
            "sha256:f35cd1ab96625b73dbb9d6e58c50055c4405a4c4aa033bd34a8b2369c9be7c1d": {
                "mediaType": "application/vnd.oci.image.layer.v1.tar",
                "size": 5120,
                "digest": "sha256:f35cd1ab96625b73dbb9d6e58c50055c4405a4c4aa033bd34a8b2369c9be7c1d"
            }
        }
    }
]
```

#### extract_image_id
We can read this file, and get the image name from `manifest[0]["RepoTags"][0]`.  
A better option though, is to use the unique UUID of this image, accessed using `manifest[0]["Config"]`. Its value is `blobs/sha256/2c9168b3c9a84851f91e03534dc4136951e9f581ab3ac8ee38b28b49ad57ba38`, the docker image ID is the last part of this value (strip `blobs/sha256/`). We can use this image ID to reference this image using docker CLI  


```python
# extract_image_id.py
import sys
import json
import tarfile


def get_id(tar_path):
    tar = tarfile.open(tar_path, mode="r")
    manifest_file_handle = tar.extractfile("manifest.json")
    if manifest_file_handle is None:
        raise
    manifest = json.load(manifest_file_handle)
    return manifest[0]["Config"].split("/")[-1]

if __name__ == "__main__":
    print(get_id(sys.argv[1]))
```

#### extract_last_layer

`manifest[0]["Layers"]` is an array containing the name of all the layers in this image. The last string in this list is our committed layer's path inside the tar file (`blobs/sha256/cb4262b4d29a1d258cd2cbe6df59bd95a2660700bdd6d73774209fe5a6ea4c4d`, this path would be present in the `tree` output above)   

We simply need to extract that layer and copy it to a destination location  
```python
# extract_last_layer.py
import shutil
import sys
import json
import tarfile


def extract_last_layer(tar_path, dest_path):
    """Extracts the last layer of a docker image from its tarball.
    Args:
    tar_path: str path to the tarball
    dest_path: str path to the destination file
    """
    tar = tarfile.open(tar_path, mode="r")
    manifest_file_handle = tar.extractfile("manifest.json")
    if manifest_file_handle is None:
        raise
    manifest = json.load(manifest_file_handle)
    last_layer = manifest[0]["Layers"][-1]
    print("copying {} to {}".format(last_layer, dest_path))
    fp = tar.extractfile(last_layer)
    if fp is None:
        raise
    with open(dest_path, "wb") as f:
        shutil.copyfileobj(fp, f)

if __name__ == "__main__":
  extract_last_layer(sys.argv[1], sys.argv[2])
```

---

We now have most of the code that is needed to fake a `RUN` instruction using the system installed docker, a given base image and commands  
We'll stitch these scripts together in a bazel rule now  

## rule in starlark

Now we need to glue this code together using starlark  
In the main script `commit_layer.sh`, we need to inject three variables: `BASE_IMAGE_TAR, USER_COMMANDS, CONTAINER_LAYER_DEST`  
This is conventionally done in bazel by creating a template file, replace placeholders in that file with actual values, generate a new file and use that in downstream code    
The script also uses python files for extracting image-id and layers, we can pass those file locations similarly using placeholders  

The template file would have variable placeholders like below:
```bash
# commit_layer.sh.tpl
docker load -i %{BASE_IMAGE_TAR}
...
```


The rule implementation roughly looks like this
```python
def _commit_layer_impl(ctx):
    # name of the target
    name = ctx.attr.name
    # base image file location
    image = ctx.file.image
    commands = ctx.attr.commands

    # location where we save the extracted layer
    output_layer_tar = ctx.outputs.layer


    # Generate a shell script to execute the run statement and extract the layer
    script = ctx.actions.declare_file(name + ".build")
    ctx.actions.expand_template(
        template = ctx.file._run_tpl,
        output = script,
        substitutions = {
            # join the commands in a single command
            "%{commands}": "'{0}'".format(" && ".join(command_list)),
            # pass extract_id.py path
            "%{image_id_extractor_path}": ctx.executable._extract_image_id.path,
            # pass extract_last_layer.py path
            "%{image_last_layer_extractor_path}": ctx.executable._last_layer_extractor_tool.path,
            # base image tar path
            "%{image_tar}": image.path,
            # layer tar destination path
            "%{output_layer_tar}": output_layer_tar.path,
        },
        is_executable = True,
    )


    # now `script` contains our shell script, with all helpers and variables injected
    # run it now
    ctx.actions.run(
        outputs = [output_layer_tar],
        inputs = [image],
        executable = script,
        mnemonic = "RunAndCommitLayer",
        tools = [
            ctx.executable._extract_image_id,
            ctx.executable._last_layer_extractor_tool
        ],
        # this is needed to find the system installed docker
        use_default_shell_env = True,
    )

    return [
        DefaultInfo(files = depset([output_layer_tar])),
    ]
```

Rule definition and extra boilerplate
```python
container_run_and_commit_layer = rule(
    attrs = {
        "commands": attr.string_list(
            doc = "A list of commands to run (sequentially) in the container.",
            mandatory = True,
            allow_empty = False,
        ),
        "image": attr.label(
            doc = "The image to run the commands in.",
            mandatory = True,
            allow_single_file = True,
            cfg = "target",
        ),
        "_extract_image_id": attr.label(
            default = Label("//tools/container_run/utils:extract_image_id"),
            cfg = "exec",
            executable = True,
            allow_files = True,
        ),
        "_last_layer_extractor_tool": attr.label(
            default = Label("//tools/container_run/utils:extract_last_layer"),
            cfg = "exec",
            executable = True,
            allow_files = True,
        ),
        "_run_tpl": attr.label(
            default = Label("//tools/container_run/utils:commit_layer.sh.tpl"),
            allow_single_file = True,
        ),
    },
    doc = ("This rule runs a set of commands in a given image, waits" +
           "for the commands to finish, and then commits the" +
           "container state to a new layer."),
    executable = False,
    implementation = _commit_layer_impl,
)
```

# Wrapping up

That's it! After a reasonably long blog,  we have successfully implemented a basic version of `container_run_and_commit` for rules_oci. Its not the most robust implementation out there, but it works!

The code is good for basic use-cases and should be usable. I would still like to caution the user to only use this as a last resort. 