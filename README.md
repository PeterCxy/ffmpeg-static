FFmpeg (mostly) static build for Jellyfin for better music streaming
==================

The main difference between this and jellyfin-ffmpeg is that it replaces the default "aac" encoder with "libfdk\_aac". The default aac encoder in ffmpeg produces suboptimal results on many samples (especially ones with a lot of higher-frequency components, like Anime songs), resulting in a lot of very noticeable distortion.

You can run build script in this repository, which produces a static `ffmpeg` binary. It can then be placed on a Jellyfin server, which can be instructed to use the new ffmpeg binary by setting `FFmpeg path` in Admin Dashboard -> Playback.

This build is mostly static, but with the following library dynamically linked:

```
[peter@peter-pc ffmpeg-static]$ ldd out/ffmpeg
        linux-vdso.so.1 (0x00007ffd983f4000)
        libdl.so.2 => /usr/lib/libdl.so.2 (0x00007f42a23db000)
        libpthread.so.0 => /usr/lib/libpthread.so.0 (0x00007f42a23b9000)
        libm.so.6 => /usr/lib/libm.so.6 (0x00007f42a2273000)
        libmvec.so.1 => /usr/lib/libmvec.so.1 (0x00007f42a2247000)
        libc.so.6 => /usr/lib/libc.so.6 (0x00007f42a2081000)
        /lib64/ld-linux-x86-64.so.2 => /usr/lib64/ld-linux-x86-64.so.2 (0x00007f42a56b3000)
        libva.so.2 => /usr/lib/libva.so.2 (0x00007f42a205c000)
        libva-drm.so.2 => /usr/lib/libva-drm.so.2 (0x00007f42a2055000)
        libdrm.so.2 => /usr/lib/libdrm.so.2 (0x00007f42a2040000)
```

This is because in order to support `VA-API`, the library `libva` must be dynamically linked, and thus we cannot link shared dependencies like `libc`, `libm` either. A lot of hacking is done to make the build system actually work with such strange configuration:

1. `-Wl,-Bstatic` is used at the very beginning of LD flags (`--extra-ldexeflags`) instead of `-static` to statically link most of the libraries; this option allows us to later use `-Wl,-Bdynamic` to link some other libraries dynamically.
2. The `--static` option of `pkg-config` is no longer usable because that will pull in things like `libm` as static dependencies; instead, everything must be handled manually (i.e. all libraries that are in `--static` output but not in non-static must be added manually to `--extra-libs`, while those that cannot be linked statically must be placed after `-Wl,-Bdynamic`).
3. A patch in `configure` is needed to exclude `-lva -lva-drm` from its automatically-generated LD flags; this is because the `-Wl,-Bdynamic` only occurs at the end of the argument list (at least this is how options in `--extra-libs` behave on my machine), and since libva can only be linked dynamically, we need to manually place it after that flag.

FFmpeg static build
===================

*STATUS*: community-supported

Three scripts to make a static build of ffmpeg with all the latest codecs (webm + h264).

Just follow the instructions below. Once you have the build dependencies,
run ./build.sh, wait and you should get the ffmpeg binary in target/bin

Build dependencies
------------------

    # Debian & Ubuntu
    $ apt-get install build-essential curl tar libass-dev libtheora-dev libvorbis-dev libtool cmake automake autoconf

    # OS X
    # 1. install XCode
    # 2. install XCode command line tools
    # 3. install homebrew
    # brew install openssl frei0r sdl2

Build & "install"
-----------------

    $ ./build.sh [-j <jobs>] [-B] [-d]
    # ... wait ...
    # binaries can be found in ./target/bin/

Ubuntu users can download dependencies and and install in one command:

    $ sudo ./build-ubuntu.sh

If you have built ffmpeg before with `build.sh`, the default behaviour is to keep the previous configuration. If you would like to reconfigure and rebuild all packages, use the `-B` flag. `-d` flag will only download and unpack the dependencies but not build.

NOTE: If you're going to use the h264 presets, make sure to copy them along the binaries. For ease, you can put them in your home folder like this:

    $ mkdir ~/.ffmpeg
    $ cp ./target/share/ffmpeg/*.ffpreset ~/.ffmpeg


Build in docker
---------------

    $ docker build -t ffmpeg-static .
    $ docker run -it ffmpeg-static
    $ ./build.sh [-j <jobs>] [-B] [-d]

The binaries will be created in `/ffmpeg-static/bin` directory.
Method of getting them out of the Docker container is up to you.
`/ffmpeg-static` is a Docker volume.

Debug
-----

On the top-level of the project, run:

    $ . env.source

You can then enter the source folders and make the compilation yourself

    $ cd build/ffmpeg-*
    $ ./configure --prefix=$TARGET_DIR #...
    # ...

Remaining links
---------------

I'm not sure it's a good idea to statically link those, but it probably
means the executable won't work across distributions or even across releases.

    # On Ubuntu 10.04:
    $ ldd ./target/bin/ffmpeg
    not a dynamic executable

    # on OSX 10.6.4:
    $ otool -L ffmpeg
    ffmpeg:
        /usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 125.2.0)

Community, bugs and reports
---------------------------

This repository is community-supported. If you make a useful PR then you will
be added as a contributor to the repo. All changes are assumed to be licensed
under the same license as the project (ISC).

As a contributor you can do whatever you want. Help maintain the scripts,
upgrade dependencies and merge other people's PRs. Just be responsible and
make an issue if you want to introduce bigger changes so we can discuss them
beforehand.

### TODO

 * Add some tests to check that video output is correctly generated
   this would help upgrading the package without too much work
 * OSX's xvidcore does not detect yasm correctly
 * remove remaining libs (?)

Related projects
----------------

* FFmpeg Static Builds - http://johnvansickle.com/ffmpeg/

License
-------

This project is licensed under the ISC. See the [LICENSE](LICENSE) file for
the legalities.

