# Docker - Deep Interview Notes (Java / Spring Boot developer, 5 yrs)

> Scope: Docker Engine 24+ / 28.x, BuildKit (default builder), Compose v2 (`docker compose`, not `docker-compose`), OCI standards.
> Kubernetes is covered in a separate note.
>
> **About the outputs in this file:** when this note was written, the Docker Desktop client existed on the machine
> (`Client: Version 28.5.1, API 1.51, Context desktop-linux`) but the **daemon was NOT running**
> (`error during connect ... //./pipe/dockerDesktopLinuxEngine: The system cannot find the file specified`).
> So no command was executed against an engine. Every "output" block below is **illustrative** (typical shape of real output;
> IDs, sizes, timestamps will differ on your machine). Flags were only used when known to exist.

---

## Table of contents
1. 60-second mental model
2. Deep internals (container = namespaces + cgroups + layers; run trace; layer stack; build cache flow)
3. Dockerfile deep dive (instruction semantics, PID 1, cache, BuildKit)
4. Java-specific images and JVM in containers
5. Worked examples (Spring Boot layered multi-stage, React+nginx, full Compose stack)
6. Networking, storage, resources, logging, restart policies
7. Security
8. Registries, tags vs digests, supply chain
9. CI, buildx multi-arch
10. Debugging and exit codes
11. Windows / WSL2 section
12. Production war stories
13. Interview questions (46) with follow-ups and wrong answers
14. One-page cheat sheet

---

# 1. 60-second mental model

**Image** = a read-only, layered, content-addressed filesystem tarball set + a JSON config (default command, env, user, ports metadata).
**Container** = a normal Linux process (or process tree) started from that image, **fenced** by the kernel:
- *namespaces* decide what it can **see** (its own PIDs, network stack, mounts, hostname, users),
- *cgroups* decide how much it can **use** (CPU, memory, pids, IO),
- *capabilities + seccomp + LSM (AppArmor/SELinux)* decide what it can **do** (syscalls, privileged ops),
- an *overlay filesystem* gives it a private writable layer on top of the shared read-only image layers.

There is **no guest OS kernel**. Container = host kernel + process isolation. VM = hypervisor + full guest kernel.

**Analogy.** Image is a *shipping-container blueprint plus its frozen contents* (class file). Container is a *running unit on a ship* (object instance): many units share the ship's engine (kernel), each has its own sealed walls (namespaces), a fuel/weight quota (cgroups), and a scratch notebook that is thrown away when it is unloaded (writable layer). A VM is a separate small ship for each cargo.

```
Dockerfile --docker build--> Image (layers + config + manifest, sha256 digests)
                               |  docker push / pull
                          Registry (Docker Hub, ECR, GHCR, Harbor)
                               |  docker run
                          Container = process + namespaces + cgroups + overlay(upper writable dir)
```

Five sentences to say in an interview:
1. A container is a process, not a VM; it shares the host kernel.
2. Isolation comes from namespaces, resource limits from cgroups.
3. Images are immutable layered filesystems addressed by SHA-256 digests; containers add a thin writable layer.
4. Dockerfile order controls cache hits; multi-stage keeps build tools out of the runtime image.
5. For Java: exec-form ENTRYPOINT (so the JVM is PID 1 and receives SIGTERM), non-root user, `MaxRAMPercentage`, layered jar, pinned tags/digests.

---

# 2. Deep internals

## 2.1 What a container really is

### Linux namespaces (what the process can see)

| Namespace | Isolates | Effect in container |
|---|---|---|
| `pid` | process ID space | your app is PID 1 inside; cannot see host processes |
| `net` | interfaces, routing table, iptables, ports | own `eth0`, own `lo`, own port 8080 |
| `mnt` | mount table | own root filesystem (`pivot_root` into the merged overlay) |
| `uts` | hostname, domain | container id as hostname |
| `ipc` | SysV IPC, POSIX queues, shm | isolated shared memory (`--ipc` to share) |
| `user` | UID/GID mapping | root inside can be unprivileged outside (only with userns-remap / rootless) |
| `cgroup` | view of cgroup tree | container sees its own cgroup as `/` |
| `time` | boot/monotonic clock offsets | rarely used |

```
HOST kernel (single)
 |
 +-- PID ns A: [1 java] [37 sh]           <- container A's view
 |   NET ns A: eth0 172.17.0.2, lo
 |   MNT ns A: / = overlay merged dir
 |
 +-- PID ns B: [1 nginx] [9 worker]       <- container B's view
     NET ns B: eth0 172.17.0.3, lo
     MNT ns B: / = another overlay merged dir
 Host sees them as ordinary processes: e.g. host PID 41377 = container A's "PID 1".
```

Proof to show interviewers (Linux host or inside WSL2 distro):
```
$ docker run -d --name web nginx:alpine
$ docker top web                       # host-side PIDs
UID   PID    PPID   CMD
root  41377  41355  nginx: master process nginx -g daemon off;
$ docker exec web ps                   # container-side PIDs
PID   USER     COMMAND
    1 root     nginx: master process nginx -g daemon off;
```
(illustrative)

### cgroups (what it can use)
Control groups account and limit resources for a set of processes.
- **v1**: one hierarchy per controller (`/sys/fs/cgroup/memory/docker/<id>/memory.limit_in_bytes`).
- **v2** (unified, default on modern distros, Docker Desktop/WSL2 recent): single hierarchy, files `memory.max`, `memory.high`, `cpu.max`, `cpu.weight`, `pids.max`, `io.max`.
- `docker run --memory=512m` -> `memory.max = 536870912`. Over the limit and cannot reclaim -> kernel **OOM killer** kills a process in that cgroup (SIGKILL, container exit code **137**).
- `--cpus=1.5` -> `cpu.max = "150000 100000"` (CFS quota 150 ms per 100 ms period). It is a **throttle**, not pinning: JVM may still see all host cores unless container-aware (it is, see section 4).
- `--cpu-shares` / `cpu.weight` = relative priority only under contention.
- `--cpuset-cpus` pins cores.
- `--pids-limit` guards fork bombs.

Check which version: `docker info | grep -i "cgroup"` -> `Cgroup Driver: systemd`, `Cgroup Version: 2`.

### Capabilities, seccomp, LSM
- Root's superpowers are split into ~40 **capabilities** (`CAP_NET_ADMIN`, `CAP_SYS_ADMIN`, `CAP_CHOWN`...). Docker gives a container a **reduced default set** (about 14, e.g. CHOWN, DAC_OVERRIDE, SETUID, NET_BIND_SERVICE, KILL). `--cap-drop ALL --cap-add NET_BIND_SERVICE` is the least-privilege pattern.
- **seccomp**: default profile blocks ~40-50 dangerous syscalls (e.g. `mount`, `reboot`, `kexec_load`, `bpf`). `--security-opt seccomp=profile.json` to customize; `unconfined` = never in prod.
- **AppArmor/SELinux**: default `docker-default` profile on AppArmor hosts.
- `--privileged` = all capabilities, all devices, seccomp/AppArmor off. Effectively root on the host. Avoid.

### Not a VM
| | Container | VM |
|---|---|---|
| Kernel | shared host kernel | own guest kernel |
| Start | ms to seconds | tens of seconds |
| Size | MBs (image) | GBs (disk image) |
| Isolation | weaker (kernel shared, kernel exploit = escape) | stronger (hypervisor) |
| Density | hundreds per host | tens |
| OS | must be same kernel family (Linux containers need Linux kernel) | any guest OS |

**Windows/Mac:** Linux containers need a Linux kernel, so **Docker Desktop runs a Linux VM** (WSL2 utility VM on Windows with the WSL2 backend; a lightweight VM on macOS). `docker` CLI on Windows talks to the daemon inside that VM through a named pipe (`//./pipe/dockerDesktopLinuxEngine`), which is exactly the error seen when Docker Desktop is not started. (Separate feature: native *Windows containers* using Windows kernel, rare for Java shops.)

## 2.2 Trace: what happens on `docker run -d -p 8080:8080 --memory=512m myapp:1.0`

1. **CLI** (`docker`) sends REST call `POST /containers/create` then `/start` to **dockerd** over unix socket `/var/run/docker.sock` (Windows: named pipe).
2. dockerd resolves image `myapp:1.0`; if missing locally, **pulls** it: fetch manifest -> pick platform (amd64/arm64) from index -> download config + layer blobs by digest -> verify sha256 -> unpack into `/var/lib/docker/overlay2/<id>/diff`.
3. dockerd creates the **container's writable layer** (`upperdir`) + overlay mount config (`lowerdir` = image layers).
4. dockerd creates the **network endpoint**: veth pair, one end in container net namespace (`eth0`), other end attached to bridge `docker0` (or the user-defined bridge); assigns IP; programs **iptables DNAT** for `-p 8080:8080`.
5. dockerd asks **containerd** (gRPC) to create the container; containerd spawns a **containerd-shim** (keeps running even if dockerd/containerd restarts, holds stdio).
6. shim invokes **runc** (OCI runtime) with an OCI `config.json` (bundle).
7. runc: `clone()` with `CLONE_NEWPID|NEWNET|NEWNS|NEWUTS|NEWIPC` (+ user if enabled), joins the cgroup and writes limits (`memory.max`), mounts the overlay as root and `pivot_root`s into it, mounts `/proc`, `/sys`(ro), `/dev`, applies capability set, seccomp filter, AppArmor profile, sets UID from image `USER`, then `execve()`s the ENTRYPOINT+CMD. runc exits; the **shim** stays as the parent.
8. The app is now PID 1 in its namespace. dockerd returns the container ID. `docker ps` lists it.
9. On exit, shim reports exit code; dockerd stores it (`docker inspect --format '{{.State.ExitCode}}'`), applies restart policy; writable layer stays until `docker rm`.

```
docker CLI --REST--> dockerd --gRPC--> containerd --> containerd-shim --> runc --> [your process]
                        |                                                   (runc exits after exec)
                        +-- networking (bridge, iptables), volumes, image store, build (BuildKit)
```

## 2.3 Image internals

An image is 3 kinds of JSON/blob objects, all **content-addressable** (name = `sha256:` of bytes):

```
Image index (multi-arch, optional)   application/vnd.oci.image.index.v1+json
  +-- manifest for linux/amd64 -----> sha256:aaa...
  +-- manifest for linux/arm64 -----> sha256:bbb...

Manifest (per platform)              application/vnd.oci.image.manifest.v1+json (or docker v2 schema2)
  { "config": { "digest": "sha256:cfg..." },
    "layers": [ {"digest":"sha256:l1..","size":29000000},
                {"digest":"sha256:l2..","size":  120000},
                {"digest":"sha256:l3..","size":48000000} ] }

Config JSON (blob sha256:cfg...)
  { "architecture":"amd64","os":"linux",
    "config": { "Env":[...], "Entrypoint":[...], "Cmd":[...], "User":"app",
                "ExposedPorts":{"8080/tcp":{}}, "WorkingDir":"/app", "Healthcheck":{...}, "Labels":{...} },
    "rootfs": { "type":"layers", "diff_ids":["sha256:d1..","sha256:d2..","sha256:d3.."] },
    "history": [ {"created_by":"/bin/sh -c #(nop) ADD file ..."}, ... ] }
```
- **Layer blob digest** = sha256 of the *compressed* tar.gz (what registry stores). **diff_id** = sha256 of the *uncompressed* tar (identity in the local store). Image ID shown by `docker images` = digest of the **config JSON**.
- **Image digest** (`myapp@sha256:...`) = digest of the manifest (or index). Same bytes -> same digest, everywhere. A **tag** is a mutable pointer to a digest.
- Layers are **shared**: 20 images from `eclipse-temurin:21-jre` store the base layers once locally and in the registry; pulls skip layers already present ("Already exists").

### Union filesystem: overlay2 + copy-on-write

```
 merged/  (what the container sees as "/")
    ^  union of:
 upperdir (container writable layer)  <-- new/modified files, "whiteouts" for deletions
 lowerdir  layer 3 (RUN mvn ...)      read-only
 lowerdir  layer 2 (COPY app.jar)     read-only
 lowerdir  layer 1 (base OS + JRE)    read-only
 workdir   (overlayfs scratch)
```
- **Read**: file found by searching top to bottom; first hit wins.
- **Write to existing lower file**: **copy-up** the whole file into upperdir, modify the copy (first write slow for big files).
- **Delete**: a **whiteout** entry (char device 0/0 or `.wh.` file in the tar) is put in the upper layer hiding the lower file. **The bytes still exist in the lower layer** -> image size does not shrink. Hence `RUN rm -rf /tmp/x` in a *later* layer does not reduce size; delete in the *same* `RUN`, or use multi-stage.
- Path on Linux host: `/var/lib/docker/overlay2/<id>/{diff,merged,work,link,lower}`. On Docker Desktop that is inside the VM, not visible from Windows.
- Writable layer is **ephemeral** and slower than a volume (copy-on-write + overlay overhead). Databases / hot write paths -> volume.
- Modern Docker may use the **containerd image store** (snapshotters) as default on fresh Docker Desktop / Engine 29 installs; concepts are identical.

### Image vs container
| | Image | Container |
|---|---|---|
| State | immutable | + writable layer + runtime state |
| Identity | digest / tag | container ID / name |
| Analogy | class | object |
| Deleted with | `docker rmi` | `docker rm` (writable layer lost) |

### Inspect commands (illustrative)
```
$ docker history myapp:1.0
IMAGE          CREATED        CREATED BY                                      SIZE
a1b2c3d4e5f6   2 min ago      ENTRYPOINT ["java" "-jar" "app.jar"]            0B
<missing>      2 min ago      COPY /app/application/ ./ # buildkit            180kB
<missing>      2 min ago      COPY /app/dependencies/ ./ # buildkit           58.2MB
<missing>      3 min ago      USER app                                        0B
<missing>      3 weeks ago    /bin/sh -c #(nop)  ENV JAVA_HOME=/opt/java...   0B
<missing>      3 weeks ago    ... (base layers) ...                           190MB
```
`<missing>` just means those layers were built elsewhere (pulled), not an error.

```
$ docker inspect --format '{{json .Config.Entrypoint}} {{.Config.User}}' myapp:1.0
["java","org.springframework.boot.loader.launch.JarLauncher"] app
$ docker inspect --format '{{.State.ExitCode}} {{.State.OOMKilled}}' web
137 true
$ docker image inspect myapp:1.0 --format '{{.RootFS.Layers}}'
[sha256:d1... sha256:d2... ...]
```

### What makes images big (drivers)
1. Base image (ubuntu ~75MB, debian-slim ~75MB, alpine ~8MB, distroless ~2-20MB, JDK ~ 300-400MB vs JRE ~ 190-270MB).
2. Fat jar (100+ MB with dependencies).
3. Build tools left in runtime image (Maven, node_modules, `.git`).
4. Package manager caches (`apt` lists, pip cache) not cleaned in same `RUN`.
5. Deleting files in a later layer (whiteouts do not reclaim).
6. `COPY . .` dragging in `target/`, logs, `.git`.
Tools: `docker history`, `dive` (third-party layer explorer), `docker image ls`.

## 2.4 Build cache flow (BuildKit)

```
For each instruction, top to bottom:
   cache key = (parent layer digest) + (instruction text) + (for COPY/ADD: content checksum of source files)
        |
   key seen before? --yes--> reuse layer ("CACHED")
        |no
   execute -> new layer -> ALL later instructions are automatically cache-MISS (chain broken)
```
Rules to memorize:
1. Once one step misses, **every step after it is rebuilt**.
2. `COPY`/`ADD`: keyed by **file content checksums** (not mtime). Unchanged content -> hit.
3. `RUN`: keyed by the **command string only**. Docker does not know that `apt-get update` would return newer packages tomorrow -> stale cache (use `--no-cache` or bust with `ARG`).
4. `ARG` used in a later `RUN` invalidates from where it is first *used* when its value changes.
5. Changing an earlier `FROM` tag/digest invalidates everything.
6. BuildKit builds independent stages in **parallel** and skips unused stages.
7. `.dockerignore` reduces the build context sent to the builder and stops irrelevant files from busting `COPY . .`.

**Layer ordering to maximize hits (Java):**
```
1. FROM (rarely changes)
2. copy pom.xml            <- changes weekly
3. resolve dependencies    <- heavy, cached until pom.xml changes
4. copy src                <- changes every commit
5. compile/package
```
Upgrade: BuildKit **cache mounts** keep `~/.m2` between builds even when the pom changes (see 3.6).

---

# 3. Dockerfile deep dive

## 3.1 Instruction semantics

| Instruction | Semantics / gotchas |
|---|---|
| `# syntax=docker/dockerfile:1` | first line; opt into the current stable frontend (needed for newer features such as `--mount`, heredocs on older engines). |
| `FROM image[:tag][@digest] AS name` | starts a stage; ARGs before FROM only usable in FROM lines. Pin tags/digests. `FROM scratch` = empty. |
| `RUN` | executes at **build time**, creates a layer. Shell form `RUN cmd` -> `/bin/sh -c`; exec form `RUN ["a","b"]` no shell. Chain with `&&`, clean in same layer. |
| `COPY src dest` | copy from build context or `--from=stage`. Predictable. `--chown=user:group`, `--chmod=` supported. |
| `ADD` | COPY + auto-extract local tar archives + can fetch URLs (and git). Prefer COPY; use ADD only for local tar extract. |
| `CMD` | default arguments/command; **overridden** by anything after image name in `docker run`. Only the last CMD counts. |
| `ENTRYPOINT` | the executable; `docker run image args` **appends** args (override with `--entrypoint`). |
| `ENV k=v` | persisted in image config and visible at runtime AND in `docker history`/inspect. |
| `ARG k=v` | build-time only (not in runtime env), but **visible in `docker history`** -> never for secrets. |
| `WORKDIR` | sets cwd, creates it; use instead of `RUN cd`. |
| `USER name\|uid` | sets user for subsequent RUN and for the runtime. Prefer numeric UID for Kubernetes `runAsNonRoot` checks. |
| `EXPOSE 8080` | **documentation/metadata only**. Does NOT publish the port. Publishing = `-p` / `-P` (publish all EXPOSEd to random ports) / Compose `ports:`. |
| `VOLUME` | declares mount point; auto-creates an anonymous volume at run; can cause surprises (changes after VOLUME are discarded). Prefer explicit runtime volumes. |
| `HEALTHCHECK` | `--interval --timeout --start-period --retries`; container state becomes `healthy/unhealthy` (plain Docker only reports; Compose `depends_on` and Swarm use it; Kubernetes ignores it and uses probes). |
| `LABEL` | metadata, e.g. OCI labels `org.opencontainers.image.source`, `.revision`, `.version`. |
| `STOPSIGNAL` | signal sent by `docker stop` (default SIGTERM). |
| `SHELL` | change default shell for shell-form. |
| `ONBUILD` | trigger for child images; rare. |

### CMD vs ENTRYPOINT combinations
| ENTRYPOINT | CMD | `docker run img` runs | `docker run img X Y` runs |
|---|---|---|---|
| none | `["java","-jar","a.jar"]` | java -jar a.jar | X Y (CMD replaced) |
| `["java","-jar","a.jar"]` | none | java -jar a.jar | java -jar a.jar X Y |
| `["java","-jar","a.jar"]` | `["--spring.profiles.active=dev"]` | ... dev | java -jar a.jar X Y (CMD replaced) |
| shell form `java -jar a.jar` | any | `/bin/sh -c "java -jar a.jar"` **CMD/args ignored** | same |

Pattern: ENTRYPOINT = the fixed program, CMD = default overridable args.

### Shell form vs exec form and PID 1 (critical for Java)

```
Shell form   ENTRYPOINT java -jar app.jar
   -> runs  /bin/sh -c "java -jar app.jar"
   PID 1 = sh        PID 7 = java
   docker stop -> SIGTERM to PID 1 (sh). sh does NOT forward it (and PID 1 has no default
   signal disposition). java never sees SIGTERM -> waits 10 s -> SIGKILL (exit 137).
   No graceful shutdown: in-flight HTTP requests dropped, Kafka offsets/DB txns not closed, @PreDestroy never runs.

Exec form    ENTRYPOINT ["java","-jar","app.jar"]
   -> execve java directly. PID 1 = java.
   SIGTERM -> JVM shutdown hooks run -> Spring closes context (graceful shutdown) -> exit code 143 (128+15).
```
Additional PID 1 facts:
- The kernel does not apply default signal actions to PID 1 of a namespace: a signal with no installed handler is **ignored**. The JVM installs SIGTERM/SIGINT handlers, so it is fine as PID 1; naive apps (e.g. some shell scripts, Python without handlers) are not.
- PID 1 must **reap zombie** children. JVM doesn't need to unless it spawns processes. Use `docker run --init` (injects `tini`) or `init: true` in Compose to solve both signal forwarding and zombie reaping.
- If you need a wrapper script (`entrypoint.sh`), end it with `exec java ...` so java replaces the shell. Otherwise shell stays PID 1.
- Spring Boot: `server.shutdown=graceful` and `spring.lifecycle.timeout-per-shutdown-phase=25s`. Make sure `docker stop -t` (or `stop_grace_period` in Compose) >= that timeout.
- Env var expansion problem with exec form: `["java","-Xmx$MEM","-jar"...]` does NOT expand `$MEM` (no shell). Solutions: `JAVA_TOOL_OPTIONS` / `JAVA_OPTS` picked up with `sh -c 'exec java $JAVA_OPTS -jar app.jar'` (shell but with `exec`), or `JAVA_TOOL_OPTIONS` env which the JVM reads itself (prints `Picked up JAVA_TOOL_OPTIONS: ...` on stderr).

### ARG vs ENV
| | ARG | ENV |
|---|---|---|
| Lifetime | build only | build + runtime |
| Set by | `--build-arg K=V` | Dockerfile / `-e` / `--env-file` |
| In final image config | not in Env, but appears in `history` | yes |
| Secrets? | **no** | **no** |
Pattern: `ARG APP_VERSION=dev` then `ENV APP_VERSION=$APP_VERSION` if runtime needs it. `ARG` before first `FROM` is scoped only to FROM lines; re-declare `ARG X` inside a stage to use it.

### COPY vs ADD
Use COPY. ADD surprises: silently extracts tarballs, remote URL fetch has no checksum caching by default (`--checksum` exists in newer versions), implicit network dependency.

## 3.2 Multi-stage builds
```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build      # heavy stage: compilers, ~700MB, thrown away
...
FROM eclipse-temurin:21-jre AS runtime           # final stage: what ships
COPY --from=build /app/target/*.jar app.jar
```
- Only the **last stage** (or `--target`) becomes the image; earlier stages exist only in the build cache.
- Benefits: small images, no build tools/source/secret material in prod image, parallel stages, `--target test` for CI stages, reuse via `COPY --from=<external image>`.
- Build stage can double as a test stage: `FROM build AS test` + `RUN mvn test`.

## 3.3 .dockerignore
```
.git
.gitignore
target/
build/
node_modules/
*.log
.idea/
.vscode/
*.iml
.env
docker-compose*.yml
Dockerfile
**/*.md
```
Effects: smaller context (faster `docker build` upload to the builder, especially on Windows/WSL bind), avoids leaking `.env`/keys into `COPY . .`, avoids needless cache busting. Exception pattern: `!target/app.jar` if you build the jar outside.

## 3.4 BuildKit secrets (never bake secrets)
Secrets in `ARG`, `ENV`, or `COPY`'d files stay in layers forever (`docker history`, layer tarballs), even if you delete them later. Correct:
```dockerfile
# syntax=docker/dockerfile:1
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
RUN --mount=type=secret,id=mvn_settings,target=/root/.m2/settings.xml \
    --mount=type=cache,target=/root/.m2/repository \
    mvn -B -q dependency:go-offline
```
```
docker build --secret id=mvn_settings,src=$HOME/.m2/settings.xml -t myapp .
```
The secret is mounted as a tmpfs file only for that `RUN`, never written to a layer. SSH keys: `RUN --mount=type=ssh` + `docker build --ssh default`. Runtime secrets: environment from secret manager (Vault, AWS Secrets Manager, K8s Secret), Compose `secrets:` (mounted at `/run/secrets/<name>`, read via `spring.config.import=configtree:/run/secrets/`).

## 3.5 Multi-stage plus cache mount for Maven
```dockerfile
RUN --mount=type=cache,target=/root/.m2 mvn -B -DskipTests package
```
- `type=cache` directory persists across builds **on the builder** (not part of the image, not pushed to registry). Even if `pom.xml` changed, unchanged jars are not re-downloaded.
- In CI with ephemeral runners, persist with `--cache-to/--cache-from` (which caches layers, not mounts; mounts need runner cache tooling) or an internal Nexus/Artifactory mirror.
- Gradle: `--mount=type=cache,target=/home/gradle/.gradle`. npm: `--mount=type=cache,target=/root/.npm`.

## 3.6 Layer-order patterns
Bad:
```dockerfile
COPY . .
RUN mvn package          # any source edit re-downloads the world
```
Good:
```dockerfile
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package
```
Best: same plus cache mount and layered jar in the runtime stage (dependencies layer rarely changes, application layer ~200 KB changes every commit -> pushes/pulls of a few KB instead of 80 MB).

---

# 4. Java-specific images and JVM in containers

## 4.1 Base image choices

| Base | Size (approx.) | Pros | Cons / gotchas |
|---|---|---|---|
| `eclipse-temurin:21-jdk` | ~ 400 MB | compiler, jcmd, jstack, jmap, debugging | bigger attack surface; do not ship if only running |
| `eclipse-temurin:21-jre` | ~ 270 MB (Ubuntu) | runtime only, glibc, easy | contains a shell + package manager |
| `eclipse-temurin:21-jre-alpine` | ~ 150-200 MB | smaller | **musl libc** gotchas (below) |
| `gcr.io/distroless/java21-debian12` | ~ 220 MB | no shell/package manager, minimal CVEs, non-root variant (`:nonroot`) | cannot `exec sh`; use `:debug` tag (busybox) or ephemeral debug |
| `redhat/ubi9` + openjdk / `ubi9/openjdk-21-runtime` | ~ 250-400 MB | enterprise support, FIPS-ready, CVE SLAs | registry auth for some, bigger |
| jlink custom runtime on `debian-slim`/`alpine` | ~ 60-120 MB | only needed modules | maintain module list; Spring Boot needs `jdeps` care |
| GraalVM native on `distroless/base` or `scratch`-like | ~ 60-120 MB | ms startup, low RSS | see 4.4 |

(Sizes are indicative, they change per release.)

**JRE vs JDK:** Since Java 11 Oracle/Temurin still publish `-jre` variants. The JDK adds compiler/tools. You want `jcmd`/`jstack` in prod for debugging? Then either use JDK image (accept size) or debug via sidecar/`docker debug`/`kubectl debug` sharing PID namespace.

**Alpine (musl) gotchas:**
- Native libraries compiled for glibc fail: `Error loading shared library ld-linux-x86-64.so.2`, `UnsatisfiedLinkError` (Netty native epoll/kqueue transports and tcnative, Snappy, RocksDB, Oracle Instant Client, some JNI libs, Chromium/PhantomJS). Use `-jre` on Debian/Ubuntu instead.
- DNS resolver differences (musl historically lacked search-domain handling for some cases and TCP fallback details); odd behavior with `ndots` in K8s.
- `java.awt`/fonts (Apache POI, JasperReports, iText image rendering) -> `Fontconfig head is null` unless you `apk add fontconfig ttf-dejavu`.
- Performance: musl malloc can be slower; some perf-critical apps see differences.
- Debuggability: busybox tools; glibc-specific profilers do not work.
Rule of thumb: alpine only if you verified all native deps; otherwise `-jre` Ubuntu/Debian or distroless.

## 4.2 Timezone, locale, CA certificates
- **Timezone**: containers default to UTC. Set `-e TZ=Asia/Kolkata` (needs tzdata in image; slim images may lack it -> `apt-get install -y tzdata`) or `-Duser.timezone=Asia/Kolkata`. Best practice: store UTC, convert at the edges.
- **Locale/encoding**: JDK 18+ defaults `file.encoding` to UTF-8; older use `-Dfile.encoding=UTF-8` and `LANG=C.UTF-8`.
- **CA certificates** (corporate proxy / internal CA -> `PKIX path building failed`):
  ```dockerfile
  COPY corp-ca.crt /usr/local/share/ca-certificates/corp-ca.crt
  RUN update-ca-certificates            # Debian/Ubuntu; Temurin Ubuntu links cacerts to system store
  # or explicit JVM store
  RUN keytool -importcert -cacerts -storepass changeit -noprompt -alias corp -file /tmp/corp-ca.crt
  ```
  Or mount a truststore and pass `-Djavax.net.ssl.trustStore=/certs/truststore.jks -Djavax.net.ssl.trustStorePassword=...`. Distroless needs the cert added in an earlier stage or a custom truststore file.

## 4.3 JVM container flags
- **Container support** (`-XX:+UseContainerSupport`) is **on by default** (JDK 10+, backported 8u191+). JVM reads cgroup limits for memory and CPU. Cgroup **v2** support: JDK 15+ and backported in 8u372, 11.0.16, 17+. Old JDK 8 builds + cgroup v2 host = JVM sees host memory -> OOM-killed. Upgrade JDK.
- **Heap sizing**: default max heap = **25%** of container memory (`MaxRAMPercentage=25`) - often too small. Typical: `-XX:MaxRAMPercentage=70` to `75` (for small containers use 60-70; heap is not the only memory). Do not combine with hard `-Xmx` unless deliberate. `-XX:InitialRAMPercentage`, `-XX:MinRAMPercentage` (only matters for tiny <256 MB).
- **Container memory = heap + metaspace + thread stacks (1 MB each x threads) + code cache + direct buffers (Netty) + GC structures + native libs**. If heap 100% of limit -> OOMKilled (137) *without* a Java `OutOfMemoryError`. Distinguish: Java OOM -> exception + heap dump, exit code 1 or via `ExitOnOutOfMemoryError`; container OOM -> silent SIGKILL 137, `OOMKilled=true`.
- **CPU**: JVM computes `availableProcessors` from the CFS quota (`--cpus=2` -> 2). Below 2 CPUs and < ~1792 MB the JVM chooses **SerialGC** ("client class machine") instead of G1 - affects pauses; force with `-XX:+UseG1GC` if wanted. Override with `-XX:ActiveProcessorCount=N`. Thread pools sized by `availableProcessors()` (ForkJoin common pool, GC threads) follow.
- **Flags to remember**:
  ```
  -XX:MaxRAMPercentage=75.0
  -XX:+ExitOnOutOfMemoryError          # die so the orchestrator restarts a healthy replacement
  -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps   # mount a volume at /dumps, else dump is lost with the container
  -XX:+UseG1GC (or ZGC for latency)
  -Xss512k (if very many threads)
  -XX:NativeMemoryTracking=summary     # diagnose non-heap
  -Djava.security.egd=file:/dev/./urandom  # old JDKs; not needed on modern
  -XX:+PrintFlagsFinal | grep -E 'MaxHeapSize|UseContainerSupport'  # verify
  ```
- Verify inside container: `java -XX:+PrintFlagsFinal -version | grep -E "MaxHeapSize|ActiveProcessorCount|UseContainerSupport"`, or Actuator `jvm.memory.max`. `-Xlog:os+container=trace` prints what JVM detected.
- Pass flags without editing exec-form: `JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=75"`.

## 4.4 Alternatives for building Java images

### Spring Boot layered jar
Spring Boot repackages jar with `BOOT-INF/layers.idx` (enabled by default since 2.4). Layers: `dependencies`, `spring-boot-loader`, `snapshot-dependencies`, `application`.
```
java -Djarmode=layertools -jar app.jar list        # shows layer names
java -Djarmode=layertools -jar app.jar extract     # extracts folders (Boot 2.3 - 3.x; newer Boot 3.3+ prefers -Djarmode=tools, verify docs of your version)
```
Then `COPY --from=build` each folder separately so only the tiny `application` layer changes per commit. Launcher class: Boot 3.2+ `org.springframework.boot.loader.launch.JarLauncher`; earlier `org.springframework.boot.loader.JarLauncher`.

### Buildpacks
`mvn spring-boot:build-image -Dspring-boot.build-image.imageName=registry/shop/order:1.0` (Paketo builder; needs a running Docker daemon). Adds layered app, non-root user, JVM memory calculator (sets heap/metaspace/stack from container limit), reproducible, SBOM, no Dockerfile. Trade-off: less control, slower first build, bigger builder images. Native: `-Pnative` profile.

### Jib (Google)
`mvn compile jib:build` (push, **no Docker daemon needed**) or `jib:dockerBuild`. Builds layers directly (deps / resources / classes), reproducible timestamps, default distroless base, config in pom (`<from><image>`, `<to>`, `<container><jvmFlags>`, `<user>`). Excellent for CI without Docker-in-Docker. Limits: no arbitrary `RUN`.

### jlink custom runtime
```dockerfile
FROM eclipse-temurin:21-jdk AS jre-build
COPY app.jar /app.jar
RUN jdeps --ignore-missing-deps --print-module-deps --multi-release 21 /app.jar > /modules.txt
RUN jlink --add-modules $(cat /modules.txt),jdk.crypto.ec,jdk.unsupported \
          --strip-debug --no-man-pages --no-header-files --compress=zip-6 --output /javaruntime
FROM debian:bookworm-slim
COPY --from=jre-build /javaruntime /opt/java
ENV PATH="/opt/java/bin:$PATH"
```
Watch out: fat jar module detection is imperfect (missing `java.sql`, `jdk.crypto.ec` for TLS, `java.naming`, `jdk.unsupported`), so test TLS/JDBC.

### GraalVM native image (Spring Boot 3 AOT)
| Pro | Con |
|---|---|
| ~50 ms startup, small RSS, great for serverless/scale-to-zero | long build (minutes, GBs RAM) |
| no JVM/JIT warm-up | peak throughput usually lower than C2 JIT (PGO in Oracle GraalVM helps) |
| small attack surface | closed world: reflection/proxies/resources need hints; some libs unsupported |
| | no dynamic class loading, harder profiling (no jcmd/JFR parity), platform-specific binary |
Use when startup/footprint matters more than peak performance. Typical long-running monoliths: keep JVM.

## 4.5 Production hardening for Java containers
- Non-root: `USER 10001`. Ports < 1024 need `NET_BIND_SERVICE` -> use 8080.
- Read-only rootfs: `docker run --read-only --tmpfs /tmp` (Tomcat/Spring need writable `/tmp`; set `-Djava.io.tmpdir` if needed). Logs to stdout, not files.
- Drop capabilities: `--cap-drop ALL`; `--security-opt no-new-privileges`.
- No shell/package manager in final image (distroless) -> fewer CVEs.
- Pin base by digest for reproducibility: `FROM eclipse-temurin:21.0.5_11-jre@sha256:...`.
- Scan (Trivy/Grype) in CI; rebuild regularly for base image patches.
- Healthcheck/probes using Actuator `/actuator/health/liveness` & `/readiness`.
- Logs to stdout in JSON; no log files inside container.

---

# 5. Worked examples

## 5.1 Production Spring Boot Dockerfile (layered, multi-stage, cache mount, non-root)

```dockerfile
# syntax=docker/dockerfile:1.7
ARG JAVA_VERSION=21

############### Stage 1: build ###############
FROM maven:3.9-eclipse-temurin-${JAVA_VERSION} AS build
WORKDIR /workspace

# 1) dependency descriptors first -> cached until pom.xml changes
COPY pom.xml .
RUN --mount=type=cache,target=/root/.m2 \
    mvn -B -q dependency:go-offline

# 2) source last -> changes every commit
COPY src ./src
RUN --mount=type=cache,target=/root/.m2 \
    mvn -B -q package -DskipTests

# 3) split fat jar into layers
RUN java -Djarmode=layertools -jar target/*.jar extract --destination /layers

############### Stage 2: runtime ###############
FROM eclipse-temurin:${JAVA_VERSION}-jre AS runtime
LABEL org.opencontainers.image.source="https://github.com/acme/order-service"

# dedicated non-root user with fixed numeric UID
RUN groupadd --system --gid 10001 app && \
    useradd  --system --uid 10001 --gid app --no-create-home --shell /usr/sbin/nologin app

WORKDIR /app
# least-changing layers first
COPY --from=build --chown=10001:10001 /layers/dependencies/ ./
COPY --from=build --chown=10001:10001 /layers/spring-boot-loader/ ./
COPY --from=build --chown=10001:10001 /layers/snapshot-dependencies/ ./
COPY --from=build --chown=10001:10001 /layers/application/ ./

USER 10001:10001
ENV JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=75 -XX:+ExitOnOutOfMemoryError \
 -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps"
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health/liveness || exit 1

# exec form: JVM is PID 1 and receives SIGTERM directly
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

Line-by-line reasoning:
- `# syntax=` selects the BuildKit frontend so `--mount` works consistently.
- `ARG JAVA_VERSION` before `FROM`: parameterizes both stages; only usable in FROM lines (that is what we do).
- Maven builder image gives `mvn` + JDK; it is discarded and never shipped.
- `COPY pom.xml` then `go-offline`: this layer only rebuilds when `pom.xml` changes. Cache mount additionally keeps `~/.m2` between builds.
- `-DskipTests` here because tests run in a separate CI step/stage; do not skip silently in CI.
- `jarmode=layertools extract`: splits the jar into 4 directories.
- Runtime is the JRE image: no compiler, smaller.
- Fixed UID 10001 so file ownership on volumes and K8s `runAsNonRoot` are predictable. `--no-create-home` and `nologin` reduce attack surface.
- Layers copied from least to most volatile: dependencies (80 MB, stable) -> loader -> snapshot deps -> application (KBs). Registry push after a code change transfers only the last layer.
- `JAVA_TOOL_OPTIONS` avoids the exec-form env expansion problem and works with any launcher; JVM prints `Picked up JAVA_TOOL_OPTIONS`.
- `HeapDumpPath=/dumps`: mount a volume there in prod, or the dump vanishes with the container (and a container with 75% heap dump may fill the writable layer).
- `HEALTHCHECK` uses `wget`; confirm the tool exists in the base you use (distroless has none: skip HEALTHCHECK there and rely on orchestrator probes).
- `ENTRYPOINT` exec form + `JarLauncher` -> `java` is PID 1, SIGTERM works, and no `-jar` needed since layers are exploded (faster startup than a fat jar because no nested-jar unpacking).

Build and run (illustrative):
```
$ docker build -t shop/order-service:1.0.0 .
[+] Building 74.2s (16/16) FINISHED
 => [build 2/6] COPY pom.xml .                               0.0s
 => [build 3/6] RUN --mount=type=cache ... dependency:go-offline  48.1s
 => [build 4/6] COPY src ./src                               0.1s
 => [build 5/6] RUN --mount=type=cache ... package           17.9s
 => [runtime 4/8] COPY --from=build /layers/dependencies/    1.1s
$ (edit one .java file) docker build ...  -> 9.3s ; dependency-related steps show CACHED
$ docker run -d --name order --read-only --tmpfs /tmp --cap-drop ALL \
    --memory=768m --cpus=1 -p 8080:8080 -v dumps:/dumps shop/order-service:1.0.0
$ docker stop order          # exec-form -> graceful, returns quickly, exit 143
$ docker inspect --format '{{.State.ExitCode}}' order
143
```
(exit 143 is a *clean* SIGTERM shutdown of the JVM; Spring may also exit 0 depending on how the shutdown is completed. Either is fine; 137 would mean SIGKILL after timeout.)

## 5.2 React + nginx (multi-stage, SPA routing, non-root)

```dockerfile
# syntax=docker/dockerfile:1.7
############### build ###############
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm npm ci          # deterministic, uses lockfile
COPY . .
ARG VITE_API_URL=/api
ENV VITE_API_URL=$VITE_API_URL                          # baked into static bundle at build time (public info only!)
RUN npm run build                                       # output in /app/dist (CRA: build/)

############### runtime ###############
FROM nginxinc/nginx-unprivileged:1.27-alpine AS runtime
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://localhost:8080/ || exit 1
# base image's ENTRYPOINT/CMD already run nginx in foreground as non-root
```
`nginx.conf`:
```nginx
server {
  listen 8080;
  root /usr/share/nginx/html;
  index index.html;

  location / {
    try_files $uri /index.html;          # SPA client-side routing, otherwise 404 on refresh
  }
  location /assets/ {                    # hashed filenames -> cache aggressively
    add_header Cache-Control "public, max-age=31536000, immutable";
  }
  location = /index.html {
    add_header Cache-Control "no-cache"; # always revalidate the entry file
  }
  location /api/ {
    proxy_pass http://order-service:8080/;   # Compose service name via Docker DNS
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }
}
```
Points to state: env vars for a static React bundle are **build-time** (values are in JS shipped to browsers, never put secrets); `npm ci` vs `npm install`; final image has no Node (~50 MB vs ~1 GB); `nginx-unprivileged` listens on 8080 as non-root (stock `nginx` needs root/port 80). Runtime config trick: serve `/config.js` generated by entrypoint script from env.

## 5.3 Full Compose stack: Spring Boot + MySQL + Redis + RabbitMQ + Kafka (KRaft) + React/nginx

`compose.yaml` (Compose v2; the top-level `version:` key is obsolete and ignored):
```yaml
name: shop                                   # project name -> network shop_default, volume shop_mysql-data

x-logging: &default-logging                  # YAML anchor reused by services
  driver: json-file
  options: { max-size: "10m", max-file: "3" }   # prevents unbounded disk usage

services:
  mysql:
    image: mysql:8.4
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: shop
      MYSQL_USER: shop
      MYSQL_PASSWORD_FILE: /run/secrets/db_password
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/db_root_password
    secrets: [db_password, db_root_password]
    volumes:
      - mysql-data:/var/lib/mysql            # named volume: survives `down`, removed by `down -v`
      - ./db/init:/docker-entrypoint-initdb.d:ro   # *.sql run only on first init of an empty datadir
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h localhost -uroot -p$$(cat /run/secrets/db_root_password) --silent"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 30s
    logging: *default-logging

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: ["redis-server", "--appendonly", "yes"]
    volumes: [redis-data:/data]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      retries: 5
    logging: *default-logging

  rabbitmq:
    image: rabbitmq:3.13-management-alpine
    restart: unless-stopped
    environment:
      RABBITMQ_DEFAULT_USER: shop
      RABBITMQ_DEFAULT_PASS: shop-dev-only
    ports:
      - "127.0.0.1:15672:15672"              # management UI only on localhost
    volumes: [rabbit-data:/var/lib/rabbitmq]
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "-q", "ping"]
      interval: 15s
      timeout: 10s
      retries: 10
    logging: *default-logging

  kafka:                                      # single-node KRaft (no ZooKeeper), dev only
    image: apache/kafka:3.8.0
    restart: unless-stopped
    ports:
      - "127.0.0.1:9092:9092"                 # host-facing listener for tools on your laptop
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENERS: INTERNAL://:29092,EXTERNAL://:9092,CONTROLLER://:9093
      KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka:29092,EXTERNAL://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT,CONTROLLER:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
    volumes: [kafka-data:/var/lib/kafka/data]
    healthcheck:
      test: ["CMD-SHELL", "/opt/kafka/bin/kafka-broker-api-versions.sh --bootstrap-server localhost:9092 > /dev/null 2>&1"]
      interval: 15s
      timeout: 10s
      retries: 10
      start_period: 30s
    logging: *default-logging

  order-service:
    build:
      context: ./order-service
      args: { JAVA_VERSION: "21" }
    image: shop/order-service:dev
    restart: unless-stopped
    init: true                                # tini as PID 1 (belt and braces)
    stop_grace_period: 40s                    # > spring.lifecycle.timeout-per-shutdown-phase
    env_file: [./order-service/.env.dev]      # non-secret defaults
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/shop
      SPRING_DATASOURCE_USERNAME: shop
      SPRING_DATA_REDIS_HOST: redis
      SPRING_RABBITMQ_HOST: rabbitmq
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:29092     # INTERNAL listener, NOT localhost
      SPRING_CONFIG_IMPORT: optional:configtree:/run/secrets/
      SERVER_SHUTDOWN: graceful
    secrets: [db_password]
    depends_on:
      mysql:    { condition: service_healthy }
      redis:    { condition: service_healthy }
      rabbitmq: { condition: service_healthy }
      kafka:    { condition: service_healthy }
    read_only: true
    tmpfs: [/tmp]
    cap_drop: [ALL]
    security_opt: ["no-new-privileges:true"]
    mem_limit: 768m
    cpus: 1.0
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8080/actuator/health/readiness"]
      interval: 15s
      timeout: 3s
      retries: 5
      start_period: 60s
    logging: *default-logging

  frontend:
    build: ./frontend
    image: shop/frontend:dev
    ports:
      - "3000:8080"                           # host 3000 -> nginx 8080
    depends_on:
      order-service: { condition: service_healthy }
    logging: *default-logging

  adminer:                                    # optional tool, only with: docker compose --profile tools up
    image: adminer:4
    profiles: [tools]
    ports: ["127.0.0.1:8088:8080"]

volumes:
  mysql-data:
  redis-data:
  rabbit-data:
  kafka-data:

secrets:
  db_password:      { file: ./secrets/db_password.txt }
  db_root_password: { file: ./secrets/db_root_password.txt }
```
Notes:
- Services on the same user-defined network reach each other by **service name** through Docker's embedded DNS (127.0.0.11 inside containers). Only `frontend` and management ports are published to the host; DB/Redis/Kafka are reachable only inside the network (do not publish databases in production).
- No `networks:` key is used, so every service joins the project's single `default` network and can reach the others by service name. Careful: if you later give only some services an explicit `networks:` list, the others stay on `default` and can no longer resolve each other (classic Compose bug: `mysql` fails to resolve).
- **Kafka dual-listener** is the classic trap: containers must use the INTERNAL advertised address (`kafka:29092`); a laptop client uses `localhost:9092`. If only `localhost:9092` is advertised, other containers connect, get metadata pointing to `localhost` (themselves) and fail.
- `depends_on: condition: service_healthy` waits for the health check to pass **at startup only**; it does not restart the app if the DB dies later. The app must still retry (Spring Boot: HikariCP, `spring.rabbitmq` retry, Kafka clients reconnect). `service_started` = just started (default with short syntax), `service_completed_successfully` = one-shot init jobs (e.g. Flyway/migrations).
- `docker-entrypoint-initdb.d` scripts execute only when data dir is empty.
- MySQL `*_FILE` env variants are supported by the official image to read secrets from files.
- Commands: `docker compose up -d --build`, `docker compose ps`, `docker compose logs -f order-service`, `docker compose exec mysql mysql -ushop -p`, `docker compose down` (keeps volumes), `docker compose down -v` (deletes volumes/data), `docker compose config` (render merged file, validate), `docker compose --profile tools up -d`.
- **Scaling**: `docker compose up -d --scale order-service=3` works only if no fixed host port is published for that service (port conflict) - put a reverse proxy (nginx/Traefik) in front; Compose DNS returns all replica IPs (round-robin DNS). Compose is not an autoscaler.

### Override files and environments
```
compose.yaml                 # base, safe defaults
compose.override.yaml        # auto-merged for local dev (bind mounts, debug ports)
compose.prod.yaml            # explicit: docker compose -f compose.yaml -f compose.prod.yaml up -d
```
`compose.override.yaml` example:
```yaml
services:
  order-service:
    ports: ["5005:5005"]                 # remote debug
    environment:
      JAVA_TOOL_OPTIONS: "-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005"
```
Merge rules: scalars override, maps merge, lists (ports, volumes) are concatenated. Variable substitution from `.env` file: `${TAG:-dev}`, `${DB_PASS:?DB_PASS is required}`. `env_file` provides container env; `.env` provides Compose-file substitution (different purposes; common confusion). `docker compose watch` (with `develop.watch`) syncs/rebuilds on file changes for dev.

### Compose vs Kubernetes
| | Compose | Kubernetes |
|---|---|---|
| Scope | single host | cluster |
| Self-healing / rescheduling | restart policy only | full |
| Rolling update, autoscale, service discovery, config/secret objects, ingress | no / basic | yes |
| Use for | local dev, integration tests, small single-VM deployments | production at scale |
Tip: Testcontainers is often preferable to Compose for Java integration tests (per-test lifecycle, random ports).

---

# 6. Networking, storage, resources, logging, restart

## 6.1 Networking

Drivers:
| Driver | Behavior |
|---|---|
| `bridge` (default network `bridge`, interface `docker0`) | containers get IPs 172.17.0.0/16; **no DNS by name** on the *default* bridge (legacy `--link` only) |
| **user-defined bridge** (`docker network create shop`) | **automatic DNS by container name/alias** (embedded DNS 127.0.0.11), better isolation; Compose creates one per project |
| `host` | shares host's network stack; no NAT, no `-p` needed; Linux only fully (Docker Desktop support is limited/opt-in); port clashes with host |
| `none` | only loopback |
| `overlay` | multi-host virtual network (Swarm; VXLAN); K8s uses CNI plugins instead |
| `macvlan` | container gets own MAC/IP on the physical LAN |

Port publishing trace `-p 8080:80`:
```
Client -> host:8080
   -> iptables nat PREROUTING -> DOCKER chain: DNAT host:8080 -> 172.18.0.2:80
   -> bridge -> veth -> container eth0 :80
Localhost traffic on some setups goes through the userland `docker-proxy`.
```
- Format: `-p [hostIP:]hostPort:containerPort[/proto]`. `-p 8080:80` binds **0.0.0.0** (all interfaces) and **bypasses ufw/firewalld rules** on Linux because Docker inserts its own iptables rules -> use `-p 127.0.0.1:5432:5432` for local-only.
- `-P` publishes all EXPOSEd ports to random host ports; `docker port <c>` shows mappings.
- **Container -> container**: same user-defined network, by name: `jdbc:mysql://mysql:3306/shop` (container port, NOT the published host port).
- **Container -> host**: `host.docker.internal` (Docker Desktop Windows/Mac). On Linux Engine: `--add-host=host.docker.internal:host-gateway` (Compose: `extra_hosts: ["host.docker.internal:host-gateway"]`). `localhost` inside a container is the container itself - the #1 beginner mistake (`localhost:3306` for a DB on the host or another container fails).
- **Host -> container**: only via published ports (or container IP on Linux; not from Windows/Mac because the VM hides the bridge).
- DNS issues: containers inherit host resolvers; corporate VPN/DNS may break; override `--dns 8.8.8.8` or `daemon.json` `"dns": [...]`.
- Network debugging: `docker network ls/inspect`, run `nicolaka/netshoot` attached to the target's network: `docker run --rm -it --network container:app nicolaka/netshoot`.

## 6.2 Storage

| Type | Command | Managed by | Use |
|---|---|---|---|
| **Named volume** | `-v mysql-data:/var/lib/mysql` / `--mount type=volume,src=mysql-data,dst=...` | Docker (`/var/lib/docker/volumes/`), portable, drivers (NFS, cloud) | databases, persistent data. **Pre-populated from image content on first mount if empty** |
| **Bind mount** | `-v /host/path:/app/config:ro` / `--mount type=bind,...` | you (host path must exist with `--mount`; `-v` silently creates a root-owned dir) | dev source code, config |
| **tmpfs** | `--tmpfs /tmp` | RAM, Linux only | scratch, secrets you do not want on disk, read-only rootfs |
| Anonymous volume | `VOLUME` in Dockerfile / `-v /data` | Docker | accumulates orphans; `docker volume prune` (never blindly) |

- Container writable layer vanishes on `docker rm`; volumes persist independently. `docker compose down -v` deletes data.
- **Permissions / UID mismatch**: container app runs as UID 10001; a bind-mounted host directory is owned by your host UID 1000 (or root) -> `Permission denied` writing `/data`. Fixes: `chown 10001 host_dir`; run container with `--user $(id -u):$(id -g)`; use named volumes (Docker copies image dir ownership on first use); init container / entrypoint chown then drop privileges; with userns-remap remember shifted UIDs. On Windows/WSL2 bind mounts from `C:\` are translated by 9P/virtiofs and usually show root/777 with poor performance.
- **Backup a volume**:
  ```
  docker run --rm -v mysql-data:/data:ro -v ${PWD}:/backup alpine tar czf /backup/mysql-data.tgz -C /data .
  ```
  Better for DBs: logical dumps (`mysqldump`) as file-level copy of a running DB may be inconsistent.
- Data locations: `docker volume inspect mysql-data` -> Mountpoint.

## 6.3 Resource limits and OOM

```
docker run --memory=512m --memory-swap=512m --cpus=1.5 --pids-limit=200 myapp
```
- `--memory` hard limit. `--memory-swap` = memory + swap total; equal to `--memory` means **no swap**; unset means swap up to same size as memory (if host swap enabled); `-1` unlimited swap. Swapping a JVM is terrible for GC pauses -> disable swap for Java.
- `--memory-reservation` soft limit. `--oom-kill-disable` exists but dangerous.
- Over limit -> kernel kills process, container **exit code 137**, `docker inspect ... .State.OOMKilled == true`. `dmesg | grep -i oom` shows on Linux host ("Memory cgroup out of memory: Killed process ... (java)").
- 137 has other causes: `docker kill`, `docker stop` timeout (10 s default) - check the OOMKilled flag to tell them apart.
- `docker stats` live CPU/mem (mem usage includes page cache in some versions; compare against limit).
- Compose: `mem_limit`, `cpus`, or `deploy.resources.limits` (Compose v2 honors it).
- CPU throttling symptom: high latency, JVM thinks it has fewer cores; check `cpu.stat` `nr_throttled` in cgroup or metrics (`container_cpu_cfs_throttled_seconds_total` in K8s).

## 6.4 Logging
- Container stdout/stderr -> **logging driver**. Default `json-file`, stored at `/var/lib/docker/containers/<id>/<id>-json.log`, **no rotation by default** -> disk fills up (classic outage).
  ```
  docker run --log-driver json-file --log-opt max-size=10m --log-opt max-file=3 ...
  ```
  Global in `/etc/docker/daemon.json` (Docker Desktop: Settings -> Docker Engine):
  ```json
  { "log-driver": "json-file", "log-opts": { "max-size": "10m", "max-file": "3" } }
  ```
  (applies to newly created containers; restart daemon.) Alternatives: `local` driver (rotates by default, compact), `fluentd`, `gelf`, `syslog`, `awslogs`, `journald`. Note: `docker logs` may not work with remote drivers in older versions; dual logging in recent Engine keeps a local cache.
- Rules: app logs to **stdout** (12-factor), structured JSON, one line per event; no file appenders inside the container.
- `docker logs -f --tail 100 --since 10m <c>`; `-t` timestamps.

## 6.5 Restart policies
| Policy | Behavior |
|---|---|
| `no` (default) | never restart |
| `on-failure[:N]` | restart when exit code != 0 (max N times) |
| `always` | always restart, also after daemon restart, even if manually stopped once daemon restarts |
| `unless-stopped` | like always, but not if you explicitly stopped it |
Policies use exponential back-off (100 ms doubling to 1 min). Restart is not health-aware (unhealthy containers are not restarted by plain Docker; use orchestrator or autoheal tooling). Crash loops -> read `docker logs` of the last run.

## 6.6 `docker run` flag cheat sheet
```
-d                       detached                 -it            interactive TTY (shell)
--rm                     delete on exit           --name x       container name
-p 8080:80               publish port             -P             publish all EXPOSEd
-e K=V / --env-file f    env                      -v / --mount   storage
--network n              attach network           --add-host h:ip
--user 10001:10001       run as uid:gid           --read-only    read-only rootfs
--tmpfs /tmp             RAM dir                  --cap-drop ALL / --cap-add X
--security-opt no-new-privileges                  --privileged   (avoid)
--memory 512m --cpus 1   limits                   --pids-limit N
--restart unless-stopped restart policy           --init         tini as PID 1
--entrypoint x           override entrypoint      -w /dir        workdir
--health-cmd ...         runtime healthcheck      --stop-timeout 30 / -t on stop
--platform linux/amd64   force platform (emulation via QEMU/Rosetta if different arch)
--log-opt max-size=10m   log rotation             --pull always|missing|never
```

---

# 7. Container security

Threat model: a container escape needs a kernel/runtime bug or a misconfiguration you handed over. Reduce blast radius:

1. **Run as non-root** (`USER 10001`). Root in a container is UID 0 on host too (unless userns) - if it escapes or a bind mount is writable, it is real root.
2. **Drop capabilities**: `--cap-drop ALL` then add only what's needed. **`no-new-privileges`** blocks setuid escalation.
3. **Never `--privileged`**; avoid `--cap-add SYS_ADMIN`, `--pid=host`, `--net=host`, `--device`.
4. **Never mount `/var/run/docker.sock`** into a container: whoever controls it controls the Docker daemon = root on the host (can start a privileged container mounting `/`). CI "docker-in-docker" alternatives: rootless BuildKit, Kaniko, Jib, buildah, or a remote builder.
5. **Read-only root filesystem** + tmpfs for temp dirs. Read-only bind mounts (`:ro`) for config.
6. **User namespaces**: `userns-remap` in `daemon.json` maps container root to an unprivileged host UID range (drawbacks: volumes/bind-mount ownership, some features incompatible). **Rootless mode** (`dockerd-rootless-setuptool.sh install`): the daemon and containers run as an unprivileged user; limits: no privileged ports < 1024 by default, slower networking (slirp4netns/pasta), some storage-driver constraints. Podman is rootless by default.
7. **Minimal images** (distroless/slim), remove package managers, no build tools.
8. **Scan**: `trivy image --severity HIGH,CRITICAL --exit-code 1 myapp:1.0` (fail the CI on findings), `grype myapp:1.0`; also scan IaC and Dockerfiles (`trivy config`, hadolint). Scanners report vulnerable packages in the OS layers and language deps (jar), matching against CVE DBs; results depend on DB freshness and produce false positives (unreachable code) - triage with VEX/`.trivyignore` with expiry.
9. **SBOM**: list of components in the image: `syft myapp:1.0 -o spdx-json`, or `docker buildx build --sbom=true --provenance=true` attaching attestations. Lets you answer "are we affected by log4shell-like CVE X" fast.
10. **Signing**: `cosign sign` an image **by digest** (keyless OIDC via GitHub Actions or a key); admission policies (Kyverno, Sigstore policy-controller, Notary v2) verify at deploy. Protects against tampered/unknown images.
11. **Secrets**: not in images/ENV in Dockerfile/git. `docker inspect` exposes ENV to anyone with daemon access. Use secret managers/mounted files.
12. **Content trust / pinning**: pin by digest, use trusted registries, verify official images/verified publishers.
13. **Daemon**: keep Docker/kernel patched, TLS if exposing the API (never expose 2375 unauthenticated), limit `docker` group membership (= root).
14. seccomp/AppArmor default profiles stay on; custom profile for sensitive workloads. gVisor/Kata for stronger isolation.

---

# 8. Registries, tags, digests, supply chain

- **Registry** stores images by repository + tag/digest, implements OCI distribution spec (`/v2/`). Docker Hub, AWS ECR, GHCR, Google Artifact Registry, Azure ACR, Harbor/Nexus/Artifactory (self-hosted).
- Naming: `[registry-host[:port]/]namespace/repo[:tag|@sha256:digest]`; no host -> Docker Hub `docker.io/library/nginx`.
- **Tags are mutable**: `myapp:1.0` can be overwritten; `latest` is just a default tag name meaning nothing (not "newest", and `docker pull` without tag uses it). Never deploy `latest`: irreproducible, impossible to roll back, cached stale on nodes. 
- Tagging strategy: immutable, traceable: `1.4.2`, `1.4.2-3f9a1c7` (semver + git short SHA), and `sha-3f9a1c7`; optionally moving tags `1.4`, `1` for convenience in dev only. Enable **tag immutability** (ECR setting). Deploy by **digest** (`image: repo/app@sha256:...`) for exact reproducibility; keep a human tag for readability.
- **Docker Hub rate limits**: pull limits for anonymous/free accounts per IP/account per 6 hours (historically 100 anonymous / 200 authenticated; policy has changed over time - check current). CI farms behind NAT hit them -> `toomanyrequests`. Fixes: authenticate (`docker login`), pull-through cache/mirror (ECR pull-through cache, Harbor proxy, Artifactory), copy base images to your private registry, use registry mirror in `daemon.json`.
- **Auth**: `docker login registry`; credentials in `~/.docker/config.json` (use credential helpers: `docker-credential-desktop`, `ecr-login`, not plain base64 `auth`). ECR: `aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin <acct>.dkr.ecr.ap-south-1.amazonaws.com` (token valid 12 h). GHCR: PAT or `GITHUB_TOKEN`. Kubernetes uses `imagePullSecrets`.
- Cleanup: lifecycle policies (ECR) to expire untagged/old images.
- Push flow: `docker tag app:1.0 reg/team/app:1.0` -> `docker push` (uploads only missing layers by digest; cross-repo mounts avoid re-upload).

---

# 9. CI usage and buildx multi-arch

## 9.1 CI practices
- Build once, promote the **same digest** through dev -> staging -> prod (do not rebuild per environment; config via env).
- Tag with git SHA (+ semver on release); label with `org.opencontainers.image.revision=$GIT_SHA`.
- Use BuildKit cache across runs (runners are ephemeral):
  ```
  docker buildx build \
    --cache-from type=registry,ref=reg/shop/order:buildcache \
    --cache-to   type=registry,ref=reg/shop/order:buildcache,mode=max \
    -t reg/shop/order:sha-$GIT_SHA --push .
  # GitHub Actions: --cache-from type=gha --cache-to type=gha,mode=max
  ```
  `mode=max` also exports intermediate (builder-stage) layers; `min` only final image layers.
- Pipeline: lint Dockerfile (hadolint) -> unit tests -> build -> Trivy scan (gate) -> SBOM/provenance -> push -> cosign sign -> deploy by digest.
- Avoid docker-in-docker privileged runners where possible; if using DinD, isolate runners.
- Do not `docker system prune -a` on shared runners mid-build; schedule cleanup.

## 9.2 Multi-arch with buildx
Why: developers on Apple Silicon (arm64) vs servers (amd64), or AWS Graviton (arm64, cheaper).
```
docker buildx create --name multi --driver docker-container --use
docker buildx build --platform linux/amd64,linux/arm64 -t reg/shop/order:1.0.0 --push .
docker buildx imagetools inspect reg/shop/order:1.0.0     # shows the index with both manifests
```
- Result is an **image index** (manifest list); the pulling host picks its arch.
- Cross-building uses QEMU emulation (`docker run --privileged --rm tonistiivi/binfmt --install all`, slow: 5-20x) or native builders per arch, or **cross-compilation** in the Dockerfile: `FROM --platform=$BUILDPLATFORM ...` with `ARG TARGETARCH`. For Java, bytecode is portable - build the jar on the build platform once and only the JRE base differs per arch: `FROM --platform=$BUILDPLATFORM maven ... AS build` (no emulation for the slow Maven step), then `FROM eclipse-temurin:21-jre` (resolved per target platform).
- Multi-arch images cannot be `--load`ed into the classic local store in one go (loading only a single platform, unless containerd image store); `--push` is the standard.
- Error `exec format error` = binary arch does not match CPU (see 10.3).

---

# 10. Debugging

## 10.1 Everyday commands
```
docker ps -a                          # includes exited containers
docker logs -f --tail 200 <c>         # stdout/stderr
docker exec -it <c> sh                # shell in running container (bash if present)
docker inspect <c>                    # full JSON; use --format '{{...}}'
docker stats                          # live usage vs limits
docker top <c>                        # processes (host PIDs)
docker events --since 10m             # daemon events: die, oom, health_status, kill
docker diff <c>                       # files changed in writable layer
docker cp <c>:/app/x.log .            # copy files out
docker run --rm -it --entrypoint sh img   # shell in an image whose ENTRYPOINT crashes
docker system df                      # disk usage; -v for detail
docker system prune                   # removes stopped containers, dangling images, unused networks, build cache (NOT volumes unless --volumes). Read what it will delete first.
```
Debugging distroless / shell-less images:
- `docker debug <container>` (Docker Desktop feature, attaches a toolbox shell; availability depends on your subscription/version).
- Attach a helper container to the target's namespaces: `docker run --rm -it --pid=container:app --network=container:app busybox sh` (add `--cap-add SYS_PTRACE` for strace/jstack; filesystem via `/proc/1/root`).
- distroless `:debug` tag variants contain busybox (`docker run --entrypoint sh gcr.io/distroless/java21-debian12:debug`).
- Kubernetes equivalent: ephemeral containers (`kubectl debug -it pod --image=busybox --target=app`).
- JVM diagnosis from a JRE-only container: mount the JDK tools or use the JDK image as helper sharing the PID namespace, then `jcmd 1 Thread.print`. Java attach needs same UID and `/tmp` access.

## 10.2 Exit codes
| Code | Meaning |
|---|---|
| 0 | process ended successfully (for a server: unexpectedly finished, or PID 1 was a script that ended) |
| 1 | generic application error (Java uncaught exception, Spring startup failure) |
| 2 | shell misuse / wrong CLI usage |
| 125 | `docker run` itself failed (bad flag, daemon error) |
| 126 | command found but not executable (permission) |
| 127 | command not found (typo in ENTRYPOINT, missing binary) |
| 130 | SIGINT (Ctrl-C) |
| **137** | 128+9 SIGKILL: OOM killer, `docker kill`, or `docker stop` grace period expired |
| **139** | 128+11 SIGSEGV: segfault (native crash: JNI lib, musl/glibc mismatch, JVM bug -> `hs_err_pid.log`) |
| **143** | 128+15 SIGTERM: graceful stop (Java exits with 143 when killed by TERM) |
| 255 | exit status out of range / SSH-like failures |

## 10.3 Common failures: symptom -> cause -> fix
| Symptom | Cause | Fix |
|---|---|---|
| Container exits immediately, code 0 | PID 1 finished (e.g. `CMD ["bash"]` without TTY, shell script ended, app daemonized/forked to background, `nginx` without `daemon off;`) | run foreground process; `-it` for shells; `exec` in scripts |
| Exit 1 right after start | app crashed (bad config, DB not up) | `docker logs`; check depends_on health |
| `exec format error` | wrong CPU arch image (arm64 image on amd64 or vice versa), or script without shebang, or **CRLF line endings** in `entrypoint.sh` (`/bin/sh^M`) on Windows | `--platform`, buildx multi-arch; `dos2unix`, `.gitattributes` `*.sh text eol=lf` |
| `exec /entrypoint.sh: no such file or directory` | CRLF in shebang, or interpreter (`bash`) missing in alpine | fix line endings, use `sh` |
| `Bind for 0.0.0.0:8080 failed: port is already allocated` | host port in use by another container/process | `docker ps`, `netstat -ano | findstr :8080` (Windows), change host port |
| `ports are not available: ... forbidden by access permissions` (Windows) | port in Windows/Hyper-V excluded range | `netsh interface ipv4 show excludedportrange protocol=tcp`; pick another port |
| Can't resolve service name / `UnknownHostException: mysql` | on default bridge, different networks, typo, service not started | use user-defined network or Compose, same network, `docker network inspect` |
| App reaches `localhost:3306` refused | localhost = container itself | use service name or `host.docker.internal` |
| `permission denied` writing to volume | UID mismatch (7.2) | chown, `--user`, named volume |
| `no space left on device` | image/layer/volume/log accumulation; on Docker Desktop, VHDX full | `docker system df`, prune carefully, log rotation, compact VHDX |
| Slow builds / huge context | no `.dockerignore`, source in `/mnt/c` | add ignore; work inside WSL filesystem |
| `Error response from daemon: pull access denied` / `toomanyrequests` | not logged in / private repo / rate limit | `docker login`, mirror |
| `Cannot connect to the Docker daemon` | daemon not running (Docker Desktop not started), wrong context, no permission on socket | start Docker Desktop, `docker context ls`, add user to `docker` group |
| Health `unhealthy` | check command missing binary (curl absent) or start_period too short | `docker inspect --format '{{json .State.Health}}' c` shows last probe outputs |
| Old code runs after rebuild | cache, or tag not moved, or compose using old image | `docker compose up --build`, `--no-cache`, check image ID |

DNS check: `docker run --rm alpine nslookup google.com`; inside Compose `docker compose exec app getent hosts mysql`.

---

# 11. Windows and WSL2 section

Your setup (Windows 11 + Docker Desktop): 
```
Windows: docker.exe (CLI)  --named pipe //./pipe/dockerDesktopLinuxEngine-->  Docker Desktop backend
   |                                                                            |
   +-- context "desktop-linux"                                   WSL2 utility distro "docker-desktop"
                                                                   runs dockerd + containerd (Linux kernel)
                                                                   your containers live here
```
Facts:
- If you see `error during connect ... dockerDesktopLinuxEngine: The system cannot find the file specified` -> Docker Desktop is not running (start it, wait for "Engine running"). Verify: `docker version` shows both Client and Server; `docker context ls`; `wsl -l -v` shows `docker-desktop` Running.
- **Performance**: Files under `C:\...` mounted to Linux go through a cross-OS filesystem bridge (slow: Maven `target/`, `node_modules`, DB volumes). Keep projects in the WSL filesystem (`\\wsl$\Ubuntu\home\you\project`, open with `code .` from WSL) and run docker commands from that WSL shell (enable WSL integration in Docker Desktop settings). Named volumes live in the fast ext4 VHDX.
- **Resource limits for the VM** (`%UserProfile%\.wslconfig`): 
  ```
  [wsl2]
  memory=8GB
  processors=4
  swap=2GB
  ```
  then `wsl --shutdown`. Without limits WSL2 may hog memory. Java container limits are inside this VM budget: `--memory=4g` on a VM given 2 GB will OOM.
- **Disk**: `docker-desktop` VHDX grows and does not automatically shrink; after `docker system prune`, reclaim via Docker Desktop's disk cleanup/resource settings or `wsl --shutdown` + `Optimize-VHD` (Hyper-V module) / `diskpart compact`.
- **Line endings**: Git for Windows `core.autocrlf=true` turns `entrypoint.sh` into CRLF -> `exec format error` / `no such file`. Add `.gitattributes`: `*.sh text eol=lf`; `Dockerfile text eol=lf`.
- **Git Bash path mangling**: `docker run -v /c/data:/data` or `docker exec c ls /app` gets rewritten (`C:/Program Files/Git/app`). Fix with `MSYS_NO_PATHCONV=1` or double slash `//app`. PowerShell: use `${PWD}` for current dir: `-v ${PWD}:/app`; cmd: `%cd%`.
- Bind mount paths: `-v C:\dev\proj:/app` works in PowerShell; in WSL use `/home/you/proj`.
- **Ports & localhost**: published ports are reachable at `localhost:port` from Windows. Excluded port ranges (Hyper-V) can block e.g. 8080/50000 ranges - see 10.3.
- **host.docker.internal** resolves to the Windows host from containers (Desktop); use it for a DB/IDE service running natively on Windows.
- **Linux vs Windows containers mode**: tray -> "Switch to Windows containers". Default for Java stack is Linux containers; Windows containers use different images (`mcr.microsoft.com/windows/...`) and cannot run Linux images.
- **Docker Desktop licensing**: paid for larger companies (250+ employees or >$10M revenue) - alternative: Rancher Desktop, Podman Desktop, or Docker Engine installed directly in a WSL2 distro (`apt install docker-ce` there).
- **CPU arch**: amd64 machine pulling arm64-only image -> `exec format error`; `--platform linux/amd64` fix or rebuild.
- Antivirus real-time scanning of WSL/VHDX slows builds; exclude if company policy allows.
- Corporate proxy/certs: configure proxy in Docker Desktop settings (pulls) and pass `HTTP_PROXY/HTTPS_PROXY` as build args or via `~/.docker/config.json` `proxies`; add corporate CA into the image for `mvn`/`npm` inside build.

---

# 12. Production war stories (symptom -> diagnosis -> fix)

### WS1: Deployments take exactly 10 s to stop and requests get 502 during releases
- **Symptom:** rolling update drops in-flight requests; `docker stop` hangs ~10 s; exit code 137.
- **Diagnosis:** Dockerfile used `ENTRYPOINT java -jar app.jar` (shell form). `docker top` shows PID 1 `/bin/sh -c java ...`. `docker logs` shows no "Commencing graceful shutdown".
- **Fix:** exec form `ENTRYPOINT ["java","-jar","app.jar"]` (or `exec java` in script), `server.shutdown=graceful`, raise `stop_grace_period`/K8s `terminationGracePeriodSeconds` above shutdown timeout. Add readiness drain (preStop sleep) in K8s.

### WS2: Container restarts every few hours, no Java exception in logs
- **Symptom:** exit code 137, app restarts; heap graph looks fine.
- **Diagnosis:** `docker inspect` -> `OOMKilled: true`. Container limit 1 GB, `-Xmx900m` + metaspace + 300 threads + Netty direct buffers > 1 GB. Container OOM (kernel) not Java OOM.
- **Fix:** `MaxRAMPercentage=60-70`, size limit for total footprint, cap direct memory `-XX:MaxDirectMemorySize`, `-Xss`, enable NMT to see non-heap, alert on RSS/working set not just heap.

### WS3: Java 8 service constantly OOM-killed after moving to new hosts
- **Symptom:** JVM sizes heap = 25% of host RAM (64 GB host) ignoring container limit 2 GB.
- **Diagnosis:** old JDK 8u1xx without container support / cgroup v2 unaware; host upgraded to cgroup v2.
- **Fix:** upgrade to 8u372+/11.0.16+/17+/21 (Temurin), or explicit `-Xmx`. Verify with `-XX:+PrintFlagsFinal`.

### WS4: Disk 100% on the server, everything fails
- **Symptom:** `no space left on device`, DB writes fail.
- **Diagnosis:** `docker system df -v`, `du -sh /var/lib/docker/containers/*/*-json.log` -> a chatty service with 60 GB `json.log` (default driver no rotation). Plus dangling images/build cache from CI deploys.
- **Fix:** truncate (`truncate -s 0` the log), set `max-size/max-file` in daemon.json and recreate containers, log to stdout at sane levels, ship logs to a central system, scheduled `docker image prune` / `builder prune --filter until=...` (not on the DB volume host blindly).

### WS5: "Works locally, `exec format error` in cluster" (M1 laptop to amd64 nodes)
- **Diagnosis:** image built on Apple Silicon = arm64; nodes amd64.
- **Fix:** `docker buildx build --platform linux/amd64,linux/arm64 --push`; CI builds images, not laptops. Verify with `docker buildx imagetools inspect`.

### WS6: Secret leaked via image
- **Symptom:** Security scan finds AWS keys in registry image although Dockerfile did `RUN rm settings.xml`.
- **Diagnosis:** the `COPY settings.xml` layer still contains it; `docker history` / layer extraction reveal it; `ARG NEXUS_PASS` visible in history.
- **Fix:** rotate keys immediately; use `RUN --mount=type=secret`; multi-stage so secret-touching stage is not shipped (still prefer mounts); scan images for secrets (Trivy secret scanner, gitleaks); delete compromised image tags from registry.

### WS7: Build takes 12 min on every commit in CI
- **Diagnosis:** ephemeral runners have no cache; `COPY . .` before `mvn package`; big context (`.git`, `target`).
- **Fix:** reorder (pom first), add `.dockerignore`, `RUN --mount=type=cache`, `--cache-from/--cache-to type=registry` (or gha), Nexus proxy, layered jar. Build time 12 -> 2 min.

### WS8: App can't reach Kafka from another container but works from laptop
- **Diagnosis:** `KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092`; clients in other containers receive `localhost:9092` in metadata after bootstrap -> connect to themselves.
- **Fix:** separate INTERNAL (`kafka:29092`) and EXTERNAL (`localhost:9092`) listeners; container clients use INTERNAL.

### WS9: MySQL container "healthy" but app fails at startup with "Connection refused / Access denied"
- **Diagnosis:** `depends_on` short syntax only waits for container *start*; or healthcheck used `mysqladmin ping` before init scripts done; or data volume from old run has old password so `MYSQL_ROOT_PASSWORD` change had no effect (env vars only apply on first initialization).
- **Fix:** `condition: service_healthy` with realistic healthcheck; `docker compose down -v` in dev to re-init; app retry/backoff; separate migration job with `service_completed_successfully`.

### WS10: `Permission denied` on bind-mounted logs directory after moving to non-root
- **Diagnosis:** image now runs UID 10001; host dir owned by root.
- **Fix:** log to stdout; or chown dir to 10001; or named volume; K8s `fsGroup`.

### WS11: Corporate network `PKIX path building failed` inside container only
- **Diagnosis:** TLS-intercepting proxy CA present in laptop's Java but not in image's `cacerts`.
- **Fix:** import CA at build (`update-ca-certificates`/`keytool -cacerts`), or mount truststore + `-Djavax.net.ssl.trustStore`.

### WS12: Windows dev: builds and Maven tests 5x slower than on colleague's Linux
- **Diagnosis:** repo on `C:\` bind-mounted into containers through cross-OS file system; antivirus scanning.
- **Fix:** move repo into WSL2 ext4 home; run Docker from WSL; give WSL2 more memory via `.wslconfig`; use named volumes for `~/.m2` and `node_modules`.

### WS13: Latest tag pulled a different image on a new node; prod incident
- **Diagnosis:** `image: app:latest`, nodes cached different digests over time.
- **Fix:** immutable version tags plus digest pinning, ECR tag immutability, deployment manifests store digests, rollback by redeploying previous digest.

---

# 13. Interview questions

Legend: E = Easy, M = Medium, H = Hard. "Wrong" = typical incorrect answers to avoid.

## Fundamentals

**Q1 (E). What is Docker and what problem does it solve?**
A: A platform to package an application with its runtime and dependencies into an immutable image and run it as an isolated container consistently across environments. Solves "works on my machine", dependency conflicts, slow environment setup, and gives a uniform deployment artifact.
Follow-ups: How is it different from a VM? -> shared host kernel, process-level isolation, MBs and ms vs GBs and seconds. What does Docker not solve? -> kernel-level isolation strength, orchestration, persistent data management.
Wrong: "Docker is a lightweight virtual machine." / "Docker is a hypervisor."

**Q2 (E). Image vs container?**
A: Image = read-only template (layers + config); container = running (or stopped) instance with a writable layer and its own process/namespaces. Many containers can share one image. Deleting a container loses its writable layer, not the image.
Follow-up: where does the container write data? -> upperdir of overlay2; use volumes for persistence. What is `docker commit`? -> snapshot writable layer into an image; discouraged (non-reproducible), use Dockerfiles.
Wrong: "A container is a saved image."

**Q3 (E). Container vs VM?** 
A: See table in 2.1: VM has hypervisor + guest kernel; container shares host kernel, isolated by namespaces/cgroups; faster/lighter, weaker isolation; Linux containers on Windows/Mac run inside a Linux VM.
Follow-up: can a Linux container run on a Windows kernel directly? -> No; Docker Desktop uses WSL2 VM. Can a container kernel exploit affect others? -> yes, shared kernel; hence gVisor/Kata/seccomp.
Wrong: "Containers have their own OS." (they have their own userland/filesystem, not kernel)

**Q4 (E). Explain CMD vs ENTRYPOINT.**
A: ENTRYPOINT defines the executable, run args are appended; CMD provides default args/command, replaced by `docker run image args`. Combined: ENTRYPOINT fixed program + CMD default args. Override entrypoint with `--entrypoint`. Use exec form for both.
Follow-up: what if both in shell form? -> CMD ignored/becomes arguments to sh -c irrelevant; shell-form ENTRYPOINT ignores CMD and args. What if multiple CMD? -> last wins.
Wrong: "They are identical" / "CMD runs at build time."

**Q5 (E). COPY vs ADD?**
A: ADD = COPY + local tar auto-extraction + URL fetching. Prefer COPY for explicitness and predictability; use ADD for extracting a local archive. Both respect `.dockerignore` for context sources.
Follow-up: how to fetch and verify a remote file? -> `RUN curl ... && sha256sum -c` in one layer (or ADD --checksum).

**Q6 (E). What does EXPOSE do?**
A: Documents/metadata for which port the app listens on; does not publish. Publishing needs `-p host:container` or `-P`. Containers on the same user-defined network can reach each other's ports without EXPOSE.
Wrong: "EXPOSE opens the port to the host."

**Q7 (E). How do containers persist data?**
A: Volumes (named, managed by Docker), bind mounts (host path), tmpfs (RAM). Writable layer is ephemeral. DB data on named volume; back up with tar via helper container or logical dumps.
Follow-up: difference volume vs bind mount? -> Docker-managed and portable vs host-path coupled; bind mounts depend on host layout and UID; on Windows bind mounts slower.

**Q8 (E). Basic commands you use daily.** 
A: `build, run, ps -a, logs -f, exec -it, stop/rm, images, rmi, inspect, stats, cp, network ls, volume ls, compose up/down`. (Show fluency with formats `--format`.)

**Q9 (E). What is a Dockerfile layer and why care?**
A: Each filesystem-changing instruction (RUN, COPY, ADD) creates a layer; layers are cached and shared. Care for build speed (cache ordering), image size, and push/pull time. ENV/CMD/etc. only add metadata.
Follow-up: does `RUN rm bigfile` shrink the image? -> No (whiteout), must remove in the same RUN or multi-stage.

**Q10 (E). What is `docker-compose` used for?**
A: Define and run multi-container apps declaratively (services, networks, volumes) with one command; local dev, tests, small deployments. v2 CLI: `docker compose`.
Follow-up: does `depends_on` wait for readiness? -> only with `condition: service_healthy`.

## Internals

**Q11 (M). How does Docker isolate a container?**
A: Namespaces (pid, net, mnt, uts, ipc, user, cgroup) for visibility; cgroups for resources; capabilities, seccomp, AppArmor/SELinux for privileges; overlay filesystem for a private root. Say the runtime chain dockerd -> containerd -> shim -> runc.
Follow-ups: Which namespace gives each container its own IP? net. What does the user namespace add? UID remap so root in container != root on host (off by default). How do you see the container's process from the host? `docker top` / `ps` - it's a normal process.
Wrong: "Docker uses virtualization / hypervisor for isolation."

**Q12 (M). cgroups v1 vs v2 and how memory limits work.**
A: v1 separate hierarchy per controller; v2 unified, better accounting, `memory.max`, pressure stall info, is the default on modern distros. `--memory` sets the hard limit; exceeding and failing to reclaim -> OOM killer -> 137. `--cpus` uses CFS quota (throttle).
Follow-up: JVM impact? -> JVM must be cgroup-v2 aware (JDK 15+/8u372+/11.0.16+). Does `--cpus=1` pin to one core? -> No, it limits CPU time per period.

**Q13 (M). Walk through what happens on `docker run`.**
A: Use trace 2.2: CLI -> dockerd -> image pull/verify -> overlay setup -> network (veth, bridge, iptables) -> containerd -> shim -> runc (clone namespaces, cgroups, pivot_root, drop caps, seccomp) -> exec entrypoint -> PID 1.
Follow-up: what remains if dockerd restarts? -> shims keep containers alive (`live-restore` option makes this explicit).

**Q14 (M). Explain image layers, digests, and how image identity works.**
A: Layers = tar diffs, identified by sha256; manifest lists config + layer digests; image ID = config digest; image digest = manifest digest; tags are mutable pointers. Content addressing enables dedupe, integrity verification, and reproducible deploys by digest. Multi-arch = index -> per-platform manifests.
Follow-up: two different tags, same digest? -> same image. Why does digest change when you rebuild "the same" source? -> timestamps/nondeterminism in layers (reproducible builds need `SOURCE_DATE_EPOCH`, Jib does it by default).

**Q15 (M). How does overlay2 copy-on-write work? Why is deleting a file in a later layer not shrinking the image?**
A: Lower dirs read-only, upperdir writable, merged view; writes copy up entire file; deletes create whiteouts hiding lower entries; lower layer bytes remain -> image size is sum of layers.
Follow-up: performance implication? -> first write of large file expensive; heavy writes go to volumes. How does that relate to `docker diff`? -> shows upperdir changes.

**Q16 (M). Why does the PID 1 problem matter for Java?**
A: `docker stop` sends SIGTERM to PID 1. Shell-form ENTRYPOINT makes `/bin/sh` PID 1, which doesn't forward the signal -> JVM never runs shutdown hooks -> SIGKILL after 10 s (137), dropped requests/uncommitted state. Use exec form (or `exec java` in wrapper scripts) so JVM is PID 1; JVM has SIGTERM handler; or `--init/tini`. Set graceful shutdown in Spring and align timeouts.
Follow-up: why does PID 1 ignore signals without handlers? -> kernel protects init of a pid namespace from default signal actions. Zombie reaping? -> tini. Exit code after clean SIGTERM? -> typically 143.
Wrong: "Just use `CMD` instead of `ENTRYPOINT`" (form matters, not instruction).

**Q17 (M). How does the JVM behave in containers? What flags do you set?**
A: Container support on by default; JVM reads cgroup memory/CPU; default max heap 25% of limit so set `MaxRAMPercentage` (60-75); account for non-heap memory; `ExitOnOutOfMemoryError`, heap dump path on a volume; `ActiveProcessorCount` if needed; use current JDK for cgroup v2; SerialGC picked automatically on <2 CPU or <1792 MB.
Follow-ups: Why OOMKilled though heap < limit? -> non-heap (metaspace, threads, direct buffers, code cache). How to see what JVM detected? `-Xlog:os+container=info`, `PrintFlagsFinal`. Xmx vs percentage? -> percentage adapts to limit; Xmx fixed; don't mix.

## Dockerfile & build

**Q18 (M). Explain multi-stage builds and benefits.**
A: Multiple `FROM`s; artifacts copied via `COPY --from`; only final stage shipped. Result: small image without JDK/Maven/source/secret material; parallel stage execution in BuildKit; `--target` for test stages.
Follow-up: can a stage copy from an external image? -> yes `COPY --from=nginx:alpine /etc/nginx/nginx.conf .`. Do earlier stages appear in final image? -> No, but stay in build cache.

**Q19 (M). How does the build cache work; how do you order a Dockerfile for Java?**
A: Explain cache key (parent + instruction + file checksums), invalidation cascade, `RUN` keyed by string only. Order: base -> pom -> dependencies -> src -> package; cache mounts for `.m2`; layered jar so dependencies layer rarely changes; `.dockerignore`.
Follow-ups: How do you force refresh of `apt-get update` layer? -> `--no-cache`, `--pull`, combine update+install in one RUN, bump ARG. Why is `COPY . .` before `mvn package` bad? -> any change invalidates dependency download.

**Q20 (M). ARG vs ENV; how do you handle secrets at build time?**
A: ARG = build-time only (still visible in history), ENV persists into runtime. Secrets: BuildKit `--mount=type=secret` (tmpfs, not in layers); SSH mount for git; runtime secrets from orchestrator/secret manager, not baked. Deleting a secret in a later layer doesn't remove it.
Follow-up: how would you prove a secret is in the image? -> `docker history --no-trunc`, `docker save` and untar layers, `trivy image --scanners secret`.
Wrong: "Use ENV and rm the file after."

**Q21 (M). What is BuildKit and what does it add?**
A: The default build backend since Docker 23. Parallel stage execution, better caching, cache mounts, secret/ssh mounts, cache import/export (registry/gha), multi-platform via buildx, provenance/SBOM attestations, heredocs in Dockerfiles, garbage collection of cache.
Follow-up: `docker build` vs `docker buildx build`? -> buildx exposes extra builder features/multi-platform/drivers; `docker build` uses BuildKit by default now.

**Q22 (M). Shell form vs exec form for RUN/CMD/ENTRYPOINT.**
A: Shell form -> `/bin/sh -c "..."` (variable expansion, pipes, but shell PID 1 and signal problems, and ignores CMD args for ENTRYPOINT); exec form JSON array -> direct exec, no shell processing (no `$VAR` expansion, no `&&`, must use double quotes JSON).
Follow-up: how do you use env vars in exec form? -> `JAVA_TOOL_OPTIONS`, or `["sh","-c","exec java $JAVA_OPTS -jar app.jar"]`.

**Q23 (M). How do you shrink a Java image?**
A: JRE/distroless instead of JDK; multi-stage; layered jar (better for transfer size, not total size); jlink; remove docs/caches in same RUN; `.dockerignore`; avoid apt installs; consider GraalVM native; check with `dive`/`docker history`.
Follow-up: is smaller always better? -> Trade-offs: alpine musl issues, distroless debuggability, jlink maintenance. Layers vs size? -> layered jar improves cache/pull time.

**Q24 (M). Explain Spring Boot layered jars and why they help.**
A: `layers.idx` splits jar into dependencies / spring-boot-loader / snapshot-dependencies / application; extract with `-Djarmode=layertools`; COPY each as a layer; app changes touch only KB-sized layer; registry push/pull and deploy faster, cache hits for dependency layers. Extracted layout starts faster (no nested jar).
Follow-up: alternative? -> Jib or buildpacks do this automatically.

**Q25 (M). Buildpacks vs Jib vs Dockerfile?**
A: Dockerfile: full control, most common, needs daemon. Buildpacks (`spring-boot:build-image`): opinionated, secure defaults, memory calculator, SBOM, no Dockerfile, needs daemon. Jib: daemonless, reproducible, fast layering, no arbitrary RUN. Pick by team needs; Jib great for CI without Docker.

**Q26 (H). GraalVM native images in containers: when yes/no?**
A: Yes for fast start/scale-to-zero/low memory (serverless, CLI, sidecars). No when peak throughput/latency, dynamic reflection-heavy libs, fast builds, and standard JVM observability matter. Build needs lots of RAM/time; artifact is arch-specific (need per-arch builds); needs reachability metadata; smaller image on distroless/base with static or mostly-static linking.
Follow-up: how does AOT differ from JIT? -> closed-world at build time; no runtime bytecode/JIT profile. 

## Networking / storage / resources

**Q27 (M). Explain Docker networking modes.**
A: bridge (default; NAT, no name DNS on default network), user-defined bridge (DNS by name, isolation), host (shared stack), none, overlay (multi-host), macvlan. Port publishing = iptables DNAT; `EXPOSE` is docs.
Follow-ups: Why can't containers on default bridge resolve names? -> only user-defined networks have embedded DNS. How do two Compose services talk? -> service name + container port. How does host reach a container on Windows? -> published port at localhost.

**Q28 (M). Container needs to call a service on the developer's host. How?**
A: `host.docker.internal` on Desktop; `--add-host host.docker.internal:host-gateway` on Linux; ensure host service listens on 0.0.0.0/appropriate interface and firewall permits. Not `localhost`.
Wrong: "Use localhost."

**Q29 (M). Volume vs bind mount vs tmpfs; pitfalls?**
A: See 6.2. Pitfalls: UID mismatch, `-v` auto-creating root-owned dirs, anonymous volume leaks, Windows path/perf issues, `docker compose down -v` data loss, SELinux `:z/:Z` label option on RHEL hosts, config not applying because volume retained old state (MySQL password).

**Q30 (M). A container keeps restarting with exit 137; how do you diagnose?**
A: `docker inspect -f '{{.State.OOMKilled}}'`; if true -> memory limit (JVM footprint) - check `docker stats`, host `dmesg`; if false -> stop timeout or someone `kill`ed (shell-form PID 1, orchestrator liveness kill). `docker events` shows die/oom/kill events. Fix per cause.
Follow-up: difference from 143? -> TERM (graceful). 139? -> segfault, native lib / musl.

**Q31 (M). How do you limit resources; what about Java heap?**
A: `--memory`, `--memory-swap`, `--cpus`, `--pids-limit`; Compose `mem_limit`/`deploy.resources`. Heap must be below the limit with headroom; use `MaxRAMPercentage`; disable swap for JVM; watch CPU throttling.
Follow-up: `--memory-swap` semantics? -> total memory+swap; equals `--memory` = no swap.

**Q32 (M). Logging in Docker; how to prevent disk fill?**
A: Drivers; json-file default no rotation; set `max-size/max-file` globally in daemon.json or per service; `local` driver; log to stdout; central aggregation (ELK/Loki/CloudWatch).
Follow-up: does changing daemon.json affect existing containers? -> No, need recreation.

**Q33 (M). Restart policies: which for production single-host?**
A: `unless-stopped` (or `always`); `on-failure:5` for jobs. Not health-aware; orchestrator is better. Backoff. `restart` doesn't fix crash loops - inspect logs.

## Compose

**Q34 (M). `depends_on` vs healthcheck; how to make the app wait for MySQL?**
A: `depends_on` orders start; short form only waits for container start. Use `condition: service_healthy` + service healthcheck (`mysqladmin ping`), plus app-level retry (Hikari `initializationFailTimeout`, Spring retry). Migrations as one-shot service with `service_completed_successfully`. Wait scripts (wait-for-it) are legacy.
Follow-up: healthy DB later dies - does Compose restart app? -> no; app must reconnect.

**Q35 (M). How do you manage multiple environments with Compose?**
A: Base `compose.yaml` + override files (`-f`), `.env` substitution, `env_file`, profiles for optional services, `docker compose config` to inspect merged result. Secrets via `secrets:`. For real prod use orchestrator, not a dev-oriented file.
Follow-up: env_file vs .env? -> env_file goes into container env; `.env` feeds `${VAR}` substitution in the YAML.

**Q36 (M). How to scale a service in Compose? Limitations?**
A: `docker compose up --scale svc=3`; cannot have fixed host port; needs load balancer; DNS round-robin; no autoscaling; single host.

**Q37 (H). Kafka in Compose: why do clients fail after bootstrapping?**
A: Advertised listeners. Broker returns its advertised address in metadata; containers need internal name/port, host tools need localhost mapping -> two listeners (INTERNAL/EXTERNAL) with correct `advertised.listeners` and protocol map. KRaft mode: `process.roles`, `controller.quorum.voters`, node id, replication factors 1 for single node.
Follow-up: why not `network_mode: host`? -> loses isolation, Desktop limitations.

## Security

**Q38 (M). How do you secure containers?**
A: Non-root user, drop capabilities, no-new-privileges, read-only rootfs, no privileged, minimal image, patched base, scanning (Trivy/Grype) in CI, SBOM, image signing (cosign) + admission control, secrets not in images, limited resources, no docker.sock, seccomp/AppArmor default, rootless/userns-remap, network segmentation, don't publish DB ports.
Follow-up: what is wrong with adding your user to the `docker` group? -> equals root on host.

**Q39 (H). Why is mounting `/var/run/docker.sock` dangerous? Alternatives for CI builds?**
A: API access = create privileged container mounting host `/` -> full host takeover; also reveals all container envs/secrets. Alternatives: rootless BuildKit/Buildah/Kaniko, Jib (daemonless), remote BuildKit builders, ephemeral VM runners; if unavoidable (Traefik, Portainer) use a socket proxy with restricted API (read-only).

**Q40 (H). Rootless mode and user namespaces: what and trade-offs?**
A: userns-remap: daemon still root, container UIDs mapped to subuid range; rootless: daemon runs as normal user in a user namespace, so a daemon/runtime compromise doesn't give host root. Limits: ports < 1024, network performance (slirp4netns/pasta), volume ownership mapping, some drivers/features unavailable (overlay via fuse-overlayfs on old kernels, no `--net=host` semantics identical).
Follow-up: Podman comparison -> rootless by default, daemonless.

**Q41 (M). How do you handle image vulnerabilities in practice?**
A: Scan in CI, gate on HIGH/CRITICAL with fixes available, rebuild on base updates (scheduled), minimal base, ignore file with expiry and rationale (VEX), runtime scanning/inventory via SBOM, track jar deps too (Dependabot/OWASP). False positives are common: reachability matters.

## Registries / supply chain

**Q42 (E). Why not use `latest` in production?**
A: Mutable, ambiguous; different nodes may have different content; no rollback/traceability; surprise upgrades. Use immutable version+SHA tags or digests; enable registry tag immutability.
Follow-up: tag vs digest? -> digest content-addressed and immutable; tag human-friendly pointer. What about Kubernetes `imagePullPolicy`? -> with latest defaults to Always; with pinned tags IfNotPresent.

**Q43 (M). Docker Hub rate limit problems in CI; solutions?**
A: `toomanyrequests` from shared NAT IPs. Authenticate, use pull-through cache/mirror or private registry with mirrored base images, layer caching, minimize pulls. Provide ECR pull-through as example.

**Q44 (H). Build multi-arch images. Java-specific optimization?**
A: buildx with `--platform linux/amd64,linux/arm64 --push` producing an index; emulation via QEMU/binfmt slow; native builders faster. For Java build the jar once on `$BUILDPLATFORM` (`FROM --platform=$BUILDPLATFORM`), only runtime base varies per target -> no emulation of Maven. Native images require true per-arch builds.
Follow-up: how to verify? -> `docker buildx imagetools inspect`. Why did prod `exec format error`? -> arch mismatch.

## Troubleshooting / scenario

**Q45 (M). Container exits immediately with code 0. Why?**
A: PID 1 ended: script finished, server daemonized/forked, missing `daemon off`, `CMD ["bash"]` without `-it`, wrong command. Fix: foreground process, `docker logs`, run `docker run --rm -it --entrypoint sh` to explore. Containers live only as long as PID 1.
Wrong: "Docker is broken / restart it."

**Q46 (H). Production incident: app fine on laptop but in container returns 'Connection refused' to DB and `UnknownHostException`. Walk through your debugging.**
A: (1) `docker logs` for the exact exception; (2) verify names: are both on the same user-defined network (`docker network inspect`), Compose service name spelled right; (3) `docker exec app getent hosts mysql` / `nslookup`; (4) is DB actually ready (`docker ps` health, depends_on condition); (5) is app using `localhost` in `SPRING_DATASOURCE_URL` or the host-published port instead of container port; (6) check env override precedence (Spring relaxed binding of `SPRING_DATASOURCE_URL`, `.env` vs env_file); (7) `netshoot` container on the network to test `nc -zv mysql 3306`; (8) firewall/VPN DNS.
Follow-up: what changes if the DB is on the Windows host? -> `host.docker.internal`, DB bound to 0.0.0.0, Windows firewall.

## Extra graded deep-dives

**Q47 (H). Explain how `docker stop` works and how you would guarantee graceful shutdown in a Spring Boot + Kafka + JPA service.**
A: `docker stop` sends STOPSIGNAL (SIGTERM) to PID 1, waits `--time` (10 s default), then SIGKILL. Needs: exec-form/`--init`; `server.shutdown=graceful` (stops accepting, waits for active requests up to `spring.lifecycle.timeout-per-shutdown-phase`); Kafka listener containers stop and commit offsets on context close; connection pools closed; total shutdown < stop timeout (`stop_grace_period`, `terminationGracePeriodSeconds`). In K8s add readiness failure/preStop delay so LB stops routing first.
Follow-up: what if `@PreDestroy` never fires? -> SIGKILL happened (137), find out why signal not delivered.

**Q48 (H). Compare Docker, containerd, runc, Podman, and OCI.**
A: OCI = open specs: image-spec, runtime-spec, distribution-spec. runc = reference low-level OCI runtime (namespaces/cgroups). containerd = daemon managing images/snapshots/container lifecycle, calling runc via shims; used by Docker and Kubernetes (CRI plugin). Docker Engine = dockerd + CLI + build + networking + volumes on top of containerd. Kubernetes removed dockershim (1.24) and talks CRI to containerd/CRI-O; Docker-built images still work because of OCI. Podman = daemonless, rootless-default, Docker-compatible CLI, pods, systemd integration, uses crun/runc + buildah/skopeo ecosystem.
Follow-up: alternatives to runc? -> crun (C, faster), gVisor `runsc`, Kata (VM-backed).

**Q49 (H). Why might a build be slow even though only one line of source changed? Diagnose.**
A: Something early invalidated cache: `COPY . .` before dependency step, changing file in context (timestamps don't matter, contents do) like a generated file/`.git` folder/logs, `ARG BUILD_DATE` used early, unstable `FROM` tag pulled a new base, no `.dockerignore`, no cache in CI, cache mount not persisted. Use `docker build --progress=plain` to see `CACHED` steps, fix order.

**Q50 (H). Design the Docker strategy for a Java microservice platform (10 services) on AWS.**
A: Standard base image (internal hardened Temurin JRE, patched weekly), Dockerfile template or Jib/buildpacks via shared CI templates, layered jars, non-root, exec form, `MaxRAMPercentage`, actuator probes; CI: build -> test -> Trivy -> SBOM -> push to ECR with immutable tags `version-sha` -> cosign sign; multi-arch if Graviton; ECR lifecycle policies; deploy by digest to EKS/ECS; logs stdout JSON to CloudWatch/Loki; secrets from AWS Secrets Manager/SSM; local dev with Compose (profiles) or Testcontainers; Docker Hub avoided via ECR pull-through cache.

## Common wrong answers summary
| Topic | Wrong | Right |
|---|---|---|
| Container | "lightweight VM" | isolated process, shared kernel |
| EXPOSE | "publishes port" | metadata only |
| `depends_on` | "waits until service is ready" | waits for start unless `service_healthy` |
| `rm` in later layer | "reduces size" | whiteout, same-layer delete/multi-stage |
| ENV/ARG secrets | "ARG is safe" | visible in history |
| `latest` | "the newest version" | just a tag, mutable |
| localhost in container | "reaches host/DB" | itself |
| `--cpus=1` | "pins to core" | CFS quota |
| Volume delete | "compose down removes data" | only with `-v` |
| JVM in container | "always sees the container limit" | only with container-aware JDK; default heap = 25% |
| Exit 137 | "Java OutOfMemoryError" | SIGKILL; check `OOMKilled` |
| Shell-form ENTRYPOINT | "same as exec" | breaks signals/PID 1 |

---

# 14. One-page cheat sheet

```
CONCEPTS
 container = process + namespaces(pid net mnt uts ipc user cgroup) + cgroups + caps/seccomp + overlay2
 image = layers (sha256) + config + manifest; tag=mutable pointer; digest=immutable
 run chain: docker CLI -> dockerd -> containerd -> shim -> runc -> PID 1
 PID 1 = your process: exec form, tini (--init), graceful shutdown

IMAGES / BUILD
 docker build -t name:tag .            docker build --no-cache --pull --progress=plain .
 docker buildx build --platform linux/amd64,linux/arm64 -t reg/app:1.0 --push .
 docker history img | docker inspect img | docker image ls | docker rmi img
 docker save/load  | docker tag a b | docker push/pull
 Dockerfile order: FROM > deps descriptors > install deps > src > build > runtime stage
 RUN --mount=type=cache,target=/root/.m2 ...    RUN --mount=type=secret,id=x ...
 exec form: ENTRYPOINT ["java","org.springframework.boot.loader.launch.JarLauncher"]
 USER 10001 | EXPOSE = doc only | HEALTHCHECK | COPY over ADD | .dockerignore

JAVA
 -XX:MaxRAMPercentage=75 -XX:+ExitOnOutOfMemoryError -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps
 JAVA_TOOL_OPTIONS env; ActiveProcessorCount; JDK with cgroup v2 support (17+/21)
 JRE not JDK; distroless/temurin; alpine = musl risk; jlink; layertools; Jib; buildpacks; native = fast start, slow build

RUN
 docker run -d --name x -p 8080:8080 -e K=V -v vol:/data --network n --restart unless-stopped \
   --memory 512m --cpus 1 --read-only --tmpfs /tmp --cap-drop ALL --user 10001 --init img
 docker ps -a | logs -f | exec -it c sh | stop (SIGTERM, 10s, SIGKILL) | kill | rm -f | cp | diff | top | stats | events

NETWORK
 docker network create n ; user-defined bridge = DNS by name ; -p host:container (0.0.0.0 by default -> use 127.0.0.1:h:c)
 container->host: host.docker.internal ; localhost = itself ; container port not published port

STORAGE
 volumes (named) > bind mounts (dev) > tmpfs ; UID mismatch -> chown/--user ; backup via tar in helper container

COMPOSE v2
 docker compose up -d --build | down [-v] | ps | logs -f svc | exec svc sh | config | --profile p | -f a.yml -f b.yml
 depends_on: {db: {condition: service_healthy}} ; healthcheck ; env_file vs .env ; secrets ; init: true ; stop_grace_period

EXIT CODES
 0 finished | 1 app error | 125 docker failed | 126 not exec | 127 not found | 130 SIGINT
 137 SIGKILL (OOM / kill / stop timeout) | 139 SIGSEGV | 143 SIGTERM graceful

TROUBLESHOOT
 exec format error = arch or CRLF | port already allocated = host port busy | permission denied = UID
 no space = docker system df, log rotation | exits 0 = PID1 finished | unknown host = network/name
 daemon not running (Windows) = start Docker Desktop, docker context ls

SECURITY
 non-root, cap-drop ALL, no-new-privileges, read-only fs, no --privileged, no docker.sock, digest pin,
 Trivy/Grype scan, SBOM (syft), cosign sign, secrets not in image, rootless/userns

SUPPLY CHAIN/CI
 tags: 1.4.2-<gitsha>, never latest, immutable tags, deploy by digest, cache: --cache-from/--cache-to type=registry|gha
 Docker Hub rate limit -> login / mirror / private registry

WINDOWS/WSL2
 Docker Desktop = Linux VM (WSL2) ; repo inside WSL fs ; .wslconfig memory/processors ; *.sh eol=lf ;
 MSYS_NO_PATHCONV=1 in Git Bash ; excluded port ranges ; VHDX doesn't shrink automatically
```
