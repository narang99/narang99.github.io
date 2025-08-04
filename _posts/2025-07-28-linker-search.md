---
layout: post
title: Where's that shared library
subtitle: A slightly long guide on how MacOS and Linux linkers find your dependencies
tags: [linkers, python]
toc: true
private: true
---

Recently, while working at my previous company, I had gotten interested in packaging my python application, the goal was to create a self-contained directory which would be able to run our application on any machine (which won't have python installed as well).  
This turns out to be non-trivial when you have tons of imaging and AI libraries (C extensions essentially).  
I ended up diving reasonably deep into the linker rabbit-hole (specifically the MacOS and Linux linkers), the topic of this article.  

***NOTE:*** *There are references to python ecosystem in this article, but the crux is really about shared libraries and how the linker finds the dependencies your application expects. Being a python developer should not be a pre-requisite*

# The problem

Most of us would have seen errors like below (I got this error while running `import cv2` in python for an older version `opencv-python-headless==4.5.3.56`).   

```bash
ImportError: dlopen(
    /Users/hariomnarang/miniconda3/envs/linkers/lib/python3.9/site-packages/cv2/cv2.cpython-39-darwin.so, 0x0002
): 
Library not loaded: /opt/homebrew/opt/ffmpeg/lib/libavcodec.58.dylib
    Referenced from: <5066EC51-2BC1-37AE-9BDC-A1F0C65A405B> 
        /Users/hariomnarang/miniconda3/envs/linkers/lib/python3.9/site-packages/cv2/cv2.cpython-39-darwin.so
    Reason: tried: 
        '/opt/homebrew/opt/ffmpeg/lib/libavcodec.58.dylib' (no such file), 
        '/System/Volumes/Preboot/Cryptexes/OS/opt/homebrew/opt/ffmpeg/lib/libavcodec.58.dylib' (no such file), 
        '/opt/homebrew/opt/ffmpeg/lib/libavcodec.58.dylib' (no such file), 
        '/usr/local/lib/libavcodec.58.dylib' (no such file), 
        '/usr/lib/libavcodec.58.dylib' (no such file, not in dyld cache)
```

I've seen many people shoot arrows in the dark after this. We simply google these errors and try every random trick we can find. After reading this article, you would still do all of those things :), but you would know what you are doing, and might be able to go about the process in a structured way.   

The above error message is coming directly from the MacOS linker, it is saying that it wasn't able to find the dependency `libavcodec.58.dylib`, the library `cv2.cpython-39-darwin.so` has this as its dependency.  

# Basic anatomy of compiled code

At its core, an application is simply a file in a very specific format (conventionally, all application or shared library files are called **object files**, the format is the same, the OS uses the file metadata to differentiate between libraries and applications)     
Every OS has it's own format. Linux has [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format), MacOS has [Mach-O](https://en.wikipedia.org/wiki/Mach-O). These formats are ***huge***.  
When you run an application, the OS examines the contents of this file and figures out how to "run" the code **inside** this application file.  

The most basic way to run code is to load the contents of the executable in memory without any modifications and jump the instruction pointer to the start address of the machine code. The executable formats nowadays however, are slightly more complex. Apart from the raw code, there is a lot of metadata in these files which the OS uses. The metadata we are interested in is how these files define dependencies.  

## Dependencies of a Mach-O file

You can take a look at the metadata of a file using `otool` in MacOS. I'll inspect the first file in the stack trace above.  

```bash
otool -l cv2.cpython-39-darwin.so
```

This command prints the **Load Commands** in a MachO file. The MacOS linker `dyld` uses these commands to figure out how to prepare the MachO file for execution (*how to load it*).  

An example entry:
```text
Load command 7
     cmd LC_UUID
 cmdsize 24
    uuid 5066EC51-2BC1-37AE-9BDC-A1F0C65A405B
```

There are many different types of commands, which instruct the linker about some specific metadata. The command above is of type `LC_UUID`, which tells the linker to get the UUID of the object file from the property `uuid`.  

The command types we are interested in look like this
```text
Load command 12
          cmd LC_LOAD_DYLIB
      cmdsize 80
         name /opt/homebrew/opt/ffmpeg/lib/libavcodec.58.dylib (offset 24)
   time stamp 2 Thu Jan  1 05:30:02 1970
      current version 58.134.100
compatibility version 58.0.0
```

The load command above is of type `LC_LOAD_DYLIB`, the command tells the linker that the file at `/opt/homebrew/opt/ffmpeg/lib/libavcodec.58.dylib` is a direct dependency of this object file. The linker would load the dependency *if it exists at the path*.  
All the `LC_LOAD_DYLIB` commands for my object file:
```text
Load command 10
          cmd LC_LOAD_DYLIB
      cmdsize 56
         name /usr/lib/libSystem.B.dylib (offset 24)
   time stamp 2 Thu Jan  1 05:30:02 1970
      current version 1292.100.5
compatibility version 1.0.0
Load command 11
          cmd LC_LOAD_DYLIB
      cmdsize 88
         name /System/Library/Frameworks/AppKit.framework/Versions/C/AppKit (offset 24)
   time stamp 2 Thu Jan  1 05:30:02 1970
      current version 2022.44.149
compatibility version 45.0.0
Load command 12
          cmd LC_LOAD_DYLIB
      cmdsize 80
         name /opt/homebrew/opt/ffmpeg/lib/libavcodec.58.dylib (offset 24)
   time stamp 2 Thu Jan  1 05:30:02 1970
      current version 58.134.100
compatibility version 58.0.0
Load command 13
          cmd LC_LOAD_DYLIB
      cmdsize 80
         name /opt/homebrew/opt/ffmpeg/lib/libavformat.58.dylib (offset 24)
   time stamp 2 Thu Jan  1 05:30:02 1970
      current version 58.76.100
compatibility version 58.0.0
Load command 14
          cmd LC_LOAD_DYLIB
      cmdsize 72
         name /opt/homebrew/opt/ffmpeg/lib/libavutil.56.dylib (offset 24)
   time stamp 2 Thu Jan  1 05:30:02 1970
      current version 56.70.100
compatibility version 56.0.0

.....a lot of more load commands

Load command 29
      cmd LC_FUNCTION_STARTS
  cmdsize 16
  dataoff 21140744
 datasize 52768
Load command 30
      cmd LC_DATA_IN_CODE
  cmdsize 16
  dataoff 21193512
 datasize 0
Load command 31
      cmd LC_CODE_SIGNATURE
  cmdsize 16
  dataoff 26235040
 datasize 205128
Load command 32
          cmd LC_RPATH
      cmdsize 144
         path /private/var/folders/b7/g6qfbypj0tq32j5_trjh516r0000gn/T/pip-req-build-_pm4jjil/_skbuild/macosx-11.0-arm64-3.9/cmake-install/lib (offset 12)
Load command 33
          cmd LC_RPATH
      cmdsize 32
```

This is a lot to take in, let's focus on the the load command containing `libavcodec` as that's the error we got (linker could not find this specific library). Here's the load command  
```text
Load command 12
          cmd LC_LOAD_DYLIB
      cmdsize 80
         name /opt/homebrew/opt/ffmpeg/lib/libavcodec.58.dylib (offset 24)
   time stamp 2 Thu Jan  1 05:30:02 1970
      current version 58.134.100
compatibility version 58.0.0
```

`dyld` is trying to load the library `libavcodec.58.dylib` at `/opt/homebrew/opt/ffmpeg/lib/libavcodec.58.dylib`.  
> The output is very verbose. If you just a quick glance at the dependencies, use `otool -L` (capital `L`). All the idented paths are dependencies.  

Running `ls /opt/homebrew/opt/ffmpeg/lib/libavcodec.58.dylib`, I can see that it does not exist in my system. A quick google asks us to install `ffmpeg` using `brew install ffmpeg`.  
I still get the same stack trace, the file `/opt/homebrew/opt/ffmpeg/lib/libavcodec.58.dylib` still does not exist. The directory `/opt/homebrew/opt/ffmpeg/lib/` however, does exist. Running `ls` we see this.  

```bash
$ ls /opt/homebrew/opt/ffmpeg/lib/ | grep avcodec

libavcodec.61.19.101.dylib
libavcodec.61.dylib
libavcodec.a
libavcodec.dylib
```

So we have the wrong version here. If we look at the [download page](https://ffmpeg.org/download.html) of `ffmpeg`, we see that `libavcodec.58` is present in `ffmpeg 4.x` version. We can install it using `brew install ffmpeg@4`.  
This ***still*** does not fix the error. The problem is that the library now exists at `/opt/homebrew/opt/ffmpeg@4/lib/libavcodec.58.dylib` (note the `ffmpeg@4` in the path). The linker is however, looking at the path without `@4` suffix. `brew` has installed `ffmpeg 7.x` at the default location.  

We could technically uninstall `ffmpeg` and create a symlink at `/opt/homebrew/opt/ffmpeg` pointing `./ffmpeg@4`.  

*NOTE: This is dangerous, other applications using `ffmpeg@7` would start breaking*
```bash
brew uninstall ffmpeg
ln -s /opt/homebrew/opt/ffmpeg@4 /opt/homebrew/opt/ffmpeg
```
I get a numpy version mismatch error, but the problem is solved now. `opencv` is able to find its dependencies.  

### DYLD_LIBRARY_PATH

It is unreasonable to ask users to uninstall other library versions for this case though, what if some other application relies on `ffmpeg@7` to be the default installation? There is an escape hatch, the environment variable `DYLD_LIBRARY_PATH`. Its a colon-separated list of directories like `PATH`.  

If the dependency is specified by an absolute path `/opt/homebrew/opt/ffmpeg@4/lib/libavcodec.58.dylib`, `dyld` will search for the library using its leaf name (`libavcodec.58.dylib`) in all the directories in `DYLD_LIBRARY_PATH`, if the file exists in any of these directories, its loaded.  

The solution is to set `DYLD_LIBRARY_PATH` before starting python process  
```bash
# our startup file: run.sh 
export DYLD_LIBRARY_PATH="/opt/homebrew/opt/ffmpeg@4/lib/:$DYLD_LIBRARY_PATH"
python main.py
```

> Make sure you do not set `DYLD_LIBRARY_PATH` in your bashrc files, this would set it globally, it can affect other applications
> It's reasonably safe to use it like this in startup scripts.  

This solves our problem, `opencv` loads without uninstalling `ffmepg@7` system-wide.  

### absolute paths are a problem

This `opencv` installation is using absolute paths for its dependency load commands. This is the reason it is so painful for us as the end users to make this specific version work.   
It also makes shipping very annoying. We can't hope the end user would have some dependency installed at an exact location we want. It is also risky to ship the dependencies at those exact locations, as they might conflict with existing software in the user's machine (as `ffmpeg@4` conflicted with `ffmpeg@7` in my machine).  
The solution to provide libraries with all the dependencies contained in the distribution without conflicting with existing software is to use `rpaths`.  

*NOTE: You could use `DYLD_LIBRARY_PATH` to ship too (simply put all dependencies in one directory in your distribution, and set the environment variable in your startup script before calling your main program). This does not work though, I'll come to this problem later in the article*

### @loader_path, @executable_path

<img class="floating-right-picture" src="/assets/confused-psyduck.png">
Let's update opencv, and also remove ffmpeg :)

```bash
# remove dependencies
brew uninstall ffmpeg
brew uninstall ffmpeg@4


# update package
pip install opencv-python-headless opencv-python --upgrade
```

If you run the main program, the code would run without any problems! What happened?  

---
Let's look at the dependency of the shared library this version of `cv2` imports. It's at `site-packages/cv2/cv2.abi3.so`. Let's look at its dependencies.  

```bash
otool -L site-packages/cv2/cv2.abi3.so

# We see a lot of `/System/Library` paths
# these are dependencies which are provided by the kernel
# we are not interested in them for now
# I've only added the direct dependencies we care about

/Users/hariomnarang/miniconda3/envs/linkers/lib/python3.9/site-packages/cv2/cv2.abi3.so:
        @loader_path/.dylibs/libavif.16.3.0.dylib 
        @loader_path/.dylibs/libavformat.61.7.100.dylib 
        @loader_path/.dylibs/libavcodec.61.19.101.dylib 
        @loader_path/.dylibs/libswscale.8.3.100.dylib 
        @loader_path/.dylibs/libavutil.59.39.100.dylib 
        # .... other system dependency paths
```

These are now neither absolute not relative paths. They don't even look like actual paths (although the representation is very path-like). What's `@loader_path`?  

For every `LD_LOAD_DYLIB` command, `dyld` expands the path of each dependency using some rules. It provides three variables `@loader_path`, `@executable_path` and `@rpath`, which it dynamically resolves during runtime.  

`@loader_path` expands to the parent directory of the object file that `dyld` is analysing. `@executable_path` expands to the main executables parent directory path (In our case, this is the directory `python` resides in).  
<img class="floating-right-picture" src="/assets/mindblown-duck.png">

> This capability is **remarkable**. While compiling object files, we can point to dependencies relative to the current object file. This allows us to ship applications which would have all their dependencies packaged in the distribution.  

My `cv2.abi3.so` exists at `/<path-to-site-packages>/cv2/cv2.abi3.so`.  
`@loader_path` resolves to `/<path-to-site-packages>/cv2`.   
The dependency `@loader_path/.dylibs/libavcodec.61.19.101.dylib` resolves to `/<path-to-site-packages>/cv2/.dylibs/libavcodec.61.19.101.dylib`.  
Running `ls`, I can see that this file exists in my system.  

If you look closely at the opencv installation, it would have a directory `.dylibs` in the top level of the package. Running `ls` on it
```bash
$ ls /Users/hariomnarang/miniconda3/envs/linkers/lib/python3.9/site-packages/cv2/.dylibs

libSvtAv1Enc.3.0.2.dylib        libcjson.1.7.18.dylib           liblcms2.2.dylib                libsharpyuv.0.1.1.dylib         libvmaf.3.dylib
libX11.6.dylib                  libcrypto.3.dylib               liblzma.5.dylib                 libsnappy.1.2.2.dylib           libvorbis.0.dylib
libXau.6.dylib                  libdav1d.7.dylib                libmbedcrypto.3.6.3.dylib       libsodium.26.dylib              libvorbisenc.2.dylib
libXdmcp.6.dylib                libfontconfig.1.dylib           libmp3lame.0.dylib              libsoxr.0.1.2.dylib             libvpx.11.dylib
libaom.3.12.1.dylib             libfreetype.6.dylib             libnettle.8.10.dylib            libspeex.1.dylib                libwebp.7.1.10.dylib
libaribb24.0.dylib              libgmp.10.dylib                 libogg.0.dylib                  libsrt.1.5.4.dylib              libwebpmux.3.1.1.dylib
libavcodec.61.19.101.dylib      libgnutls.30.dylib              libopencore-amrnb.0.dylib       libssh.4.10.1.dylib             libx264.164.dylib
libavformat.61.7.100.dylib      libhogweed.6.10.dylib           libopencore-amrwb.0.dylib       libssl.3.dylib                  libx265.215.dylib
libavif.16.3.0.dylib            libhwy.1.2.0.dylib              libopenjp2.2.5.3.dylib          libswresample.5.3.100.dylib     libxcb.1.1.0.dylib
libavutil.59.39.100.dylib       libidn2.0.dylib                 libopus.0.dylib                 libswscale.8.3.100.dylib        libzmq.5.dylib
libbluray.2.dylib               libintl.8.dylib                 libp11-kit.0.dylib              libtasn1.6.dylib
libbrotlicommon.1.1.0.dylib     libjxl.0.11.1.dylib             libpng16.16.dylib               libtheoradec.1.dylib
libbrotlidec.1.1.0.dylib        libjxl_cms.0.11.1.dylib         librav1e.0.8.0.dylib            libtheoraenc.1.dylib
libbrotlienc.1.1.0.dylib        libjxl_threads.0.11.1.dylib     librist.4.dylib                 libunistring.5.dylib
```

The OpenCV authors have been kind to their users now, they are shipping all the dependencies in a subdirectory `.dylibs`.   
All files in `.dylibs` in turn also use `@loader_path` for **their own** dependencies.   

```bash
$ otool -L /Users/hariomnarang/miniconda3/envs/linkers/lib/python3.9/site-packages/cv2/.dylibs/libavcodec.61.19.101.dylib

# I've skipped a lot of output
/Users/hariomnarang/miniconda3/envs/linkers/lib/python3.9/site-packages/cv2/.dylibs/libavcodec.61.19.101.dylib:
        @loader_path/libvorbis.0.dylib 
        @loader_path/libvorbisenc.2.dylib 
        @loader_path/libwebp.7.1.10.dylib 
```

And these dependencies in turn do the same, each uses their own `@loader_path` expansion to correctly define its dependencies deterministically.  

### @rpath

There is one final variable that `dyld` expands, `@rpath`. An example helps.  

```bash
# a random dylib in my system
otool -l libmenu.dylib

Load command 12
          cmd LC_LOAD_DYLIB
         name @rpath/libncursesw.6.dylib

# only the relevant parts in the output
Load command 16
          cmd LC_RPATH
      cmdsize 32
         path @loader_path/

# only the relevant parts in the output
Load command 17
          cmd LC_RPATH
      cmdsize 32
         path /usr/local/lib
```

This dylib has a new load command type, `LC_RPATH`.  

`dyld` would expand `@rpath` in each `LC_LOAD_DYLIB` to the value in `LC_RPATH`. If there are multiple `LC_RPATH` commands, it would expand using each `RPATH` value, and return the first path that exists.  
You can use `@loader_path` and `@executable_path` in `LC_RPATH`.  
The command `@rpath/libncursesw.6.dylib` expands to `@loader_path/libncursesw.6.dylib` and `/usr/local/lib/libncursesw.6.dylib`, the linker loads the file that exists.  

`@rpath` provides an extra level of indirection. A pattern is to use `@rpath/<dep-name>.so` for each dependency, and set correct rpaths using `LC_RPATH`. Since we can add multiple `LC_RPATH` commands, it increases the search space the linker uses.  

### the search order

`dyld` approximately does these steps (in order) for searching, this is stripped down version.  

> `man dlopen` is the source of truth, I'm skipping Mac frameworks here.

- Search the leaf name in `DYLD_LIBRARY_PATH`
- If its not a path like component (simply specifying the library name in load command), search in the current directory
- Expand `@rpath, @executable_path, @loader_path` for each path-like dependency. If its a relative path, use the current directory to resolve.  

If a file exists at any step, it is loaded.  

## Comparison with Linux

The linux search order is slightly similar to MacOS. I'm going to briefly explain the similarities and differences for each step described in the previous section.  

Reading the dependencies of a file  
```bash
$ readelf -d <>
```