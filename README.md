# whoc
A container image that extracts the underlying container runtime and sends it to a remote server.
Poke at the underlying container runtime of your favorite CSP container platform!

- [WhoC at Defcon 29 Cloud Village](https://www.youtube.com/watch?v=DF0qoCsHKT4)
- [Azurescape](https://unit42.paloaltonetworks.com/azure-container-instances/) - whoc-powered research, the first cross-account container takeover in the public cloud (70,000$ bounty)

## How does it work?
As shown by runc [CVE-2019-5736](https://unit42.paloaltonetworks.com/breaking-docker-via-runc-explaining-cve-2019-5736/), traditional Linux container runtimes expose themselves to the containers they're running through `/proc/self/exe`. `whoc` uses this link to read the container runtime executing it.

### Dynamic Mode
This is `whoc` default mode that works against dynamically linked container runtimes.

1. The `whoc` image entrypoint is set to `/proc/self/exe`, and the image's dynamic linker (`ld.so`) is replaced with `upload_runtime`.
2. Once the image is run, the container runtime re-executes itself inside the container.
3. Given the runtime is dynamically linked, the kernel loads our fake dynamic linker (`upload_runtime`) to the runtime process and passes execution to it. 
4. `upload_runtime` reads the runtime binary through `/proc/self/exe` and sends it to the configured remote server.

![alt text](https://github.com/twistlock/whoc/blob/master/images/whoc_dynamic.png?raw=true "whoc dynamic mode")


### Wait-For-Exec Mode
For statically linked container runtimes, `whoc` comes in another flavor: `whoc:waitforexec`.

1. `upload_runtime` is the image entrypoint, and runs as the `whoc` container PID 1.
2. The user is expected to exec into the `whoc` container and invoke a file pointing to `/proc/self/exe` (e.g. `docker exec whoc_ctr /proc/self/exe`).
3. Once the exec occurs, the container runtime re-executes itself inside the container.
4. `upload_runtime` reads the runtime binary through `/proc/$runtime-pid/exe` and sends it to the configured remote server.

![alt text](https://github.com/twistlock/whoc/blob/master/images/whoc_waitforexec.png?raw=true "whoc wait-for-exec mode")

## Try Locally
You'll need `docker` and `python3` installed. Clone the repository:
```console
$ git clone git@github.com:twistlock/whoc.git
```

Set up a file server to receive the extracted container runtime:
```console
$ cd whoc
$ mkdir -p stash && cd stash
$ ln -s ../util/fileserver.py fileserver 
$ ./fileserver
```
From another shell, run the `whoc` image in your container environment of choice, for example Docker:
```console
$ cd whoc
$ docker build -f Dockerfile_dynamic -t whoc:latest src  # or ./util/build.sh
$ docker run --rm -it --net=host whoc:latest 127.0.0.1  # or ./util/run_local.sh
```
See that the file server received the container runtime. Since we run `whoc` under vanilla Docker, the received container runtime should be [runc](https://github.com/opencontainers/runc). 

*`--net=host` is only used in local tests so that the `whoc` container could easily reach the fileserver on the host via `127.0.0.1`.*


## Help
Help for `whoc`'s main binary, `upload_runtime`:
```
Usage: upload_runtime [options] <server_ip>

Options:
 -p, --port                 Port of remote server, defaults to 8080
 -e, --exec                 Wait-for-exec mode for static container runtimes, waits until an exec to the container occurred
 -b, --exec-bin             In exec mode, overrides the default binary created for the exec, default is /bin/enter
 -a, --exec-extra-argument  In exec mode, pass an additional argument to the runtime so it won't exit quickly
 -r, --exec-readdir-proc    In exec mode, instead of guessing the runtime pid (which gives whoc one shot of catching the runtime),
                            find the runtime by searching for new processes under '/proc'
```
