# bitnami-openldap for arm64


This guide explains how to compile OpenLDAP on arm64, and include it in a `bitnami/openldap` image build, that runs on arm64-based CPUs such as Raspberry Pi 4.

---


>Not affiliated with Bitnami.

>Use at your own risk, no guarantee provided.

>⚠️ Don't use on production unless you know what you are doing!


---


`bitnami/openldap` [Docker Hub](https://hub.docker.com/r/bitnami/openldap/) | [Github](https://github.com/bitnami/containers/tree/main/bitnami/openldap)

As there is no official support yet for arm64 (see: https://github.com/bitnami/bitnami-docker-openldap/issues/18, https://github.com/bitnami/charts/issues/7305), I decided to make my own, and document the steps for others and for my future self :)

 
# This document covers:
- Compile (build) *OpenLDAP 2.6* for arm64
- Package the compiled binary 
- Modify `bitnami/openldap`s Dockerfile
- Deploy the modified image to a pi


# Table of contents
- You're here
- [Tested hardware](#tested-hardware)
- [Contribute](#contribute)
- [Start](#start)
  * [1. Install dependencies](#1-install-dependencies)
  * [2. Git the source:](#2-git-the-source)
  * [3. Build OpenLDAP:](#3-build-openldap)
    + [3.1. Configure](#31-configure)
    + [3.2. Build Dependencies](#32-build-dependencies)
    + [3.3. Compile](#33-compile)
    + [3.4. Export the package](#34-export-the-package)
  * [4. Modify the Dockerfile](#4-modify-the-dockerfile)
    + [4.1. Git the bitnami image source](#41-git-the-bitnami-image-source)
    + [4.2. Base image, `gosu`, and `$PATH`](#42-base-image-gosu-and-path)
    + [4.3. OpenLDAP package installation](#43-openldap-package-installation)
      - [4.3.1. via `COPY` command](#431-via-copy-command)
      - [4.3.2. via HTTP server](#432-via-http-server)
  * [5. Modify libopenldap.sh script](#5-modify-libopenldapsh-script)
  * [6. Build the image](#6-build-the-image)
  * [7. Use the image](#7-use-the-image)


---

# Tested hardware:
- Raspberry Pi 4 (RPI OS Lite x64)

# Contribute
Contributions are needed especially given the vast options OpenLDAP offers

If you had an issue, and fixed it: please feel free to open a pull request, if you're lazy, just create an issue with label `enhancement` describing what you needed to do in order to fix it/ mention any resources you used.

If you still facing issues, you can create an issue with label `help wanted` and hope for the best :D 

Use this as a last resort, search your favorite engine and read the manual instead.

---

# Start

> Note: I used my Pi to compile the source, so I didn't need extra setup for targeting arm64.


It's a good idea to visit: https://www.openldap.org/doc

Select the version you have, and navigate to the *Running configure* manual.. for 2.6: https://www.openldap.org/doc/admin26/install.html#Running%20configure

This guide uses OpenLDAP 2.6.3 - If you're using a different version, you should check the steps here with the respective manual, and follow it along


This guide assumes:
- Source directory: `/src`, change as you like
- Output directory: `/opt/bitnami/openldap`, DON'T CHANGE

> ⚠️
> 
> Changing the output directory will cause a runtime error
>
> Mentioned in [3.1.1 Configure](#311-some-important-args-must-pass-to-configure)

## 1. Install dependencies:

The following packages are needed to compile OpenLDAP:
```bash
apt install -y build-essential libsasl2-dev libltdl-dev libevent-dev libltdl7 openssl libssl-dev libcrack2-dev libwrap0-dev libevent-dev
```

I'm not 100% sure on which packages exactly are needed to compile OpenLDAP,

Also, you might need extra packages based on your module/ overlay selection..

For example, if you want to authenticate with argon2, you'll need to install either `libargon2-dev` or `libsodium-dev`. 


To see what packages you might need, we'll need the source and use the `configure` script..


## 2. Git the source:

[Official repos can be checked @ openldap.org](https://www.openldap.org/software/repo.html)
```bash

cd /src
git clone https://git.openldap.org/openldap/openldap
cd openldap

```

## 3. Build OpenLDAP:


In order to build OpenLDAP, we'll need to:



### 3.1. Configure

From the source directory, run:


```bash
# Displays information about the available options/ modules

./configure -h

```

Read through the displayed options and take a note of what you need for your installation.

For example, if you want to enable argon2 auth with `libsodium` as library, then you need to pass `--enable-argon2 --with-argon2=libsodium` to the `configure` script.


> Play around with `configure` and see if your args are correct by using the `--no-create` arg.. when added, the `configure` will just check against your system and displays errors without writing any config/ cache files.
>
> You can always use `make clean` to remove your choices and start fresh.



#### 3.1.1 Some important args MUST pass to `configure`:

- `--prefix=/opt/bitnami/openldap` sets the installation output to `/opt/bitnami/openldap` (Required, literal)
> ⚠️ --prefix=/opt/bitnami/openldap is mandatory for this setup, basically the whole software will be installed in `/opt/bitnami/openldap` to be packaged later, this path MUST match the installation path in the container, which is `/opt/bitnami/openldap` per bitnami image.
>
> In the `configure` source I also found that this prefix is being used at runtime to find the slapd.conf file, not setting this correctly will successfully build the image but will cause runtime error: `bind(8): errno=2 (No such file or directory)`

- `CPPFLAGS="-I/opt/bitnami/openldap/include" LDFLAGS="-L/opt/bitnami/openldap/lib -Wl,-rpath,/opt/bitnami/openldap/lib"` sets linker/ compiler flags to include the lib directory (Required, literal)
> ⚠️ Without this flag the binary will fail to locate lib files, and runtime errors such as `libldap-whatever.so.0: cannot open shared object file: No such file or directory` will occur.

- `--enable-modules` is required if you want to enable modules (Conditional)


Note: We maybe should use the original `configure` args used to build the OpenLDAP binary in the bitnami image, but I can't find it..

Use this command as a base command, append your modules/ overlays accordingly:

```bash
./configure --prefix=/opt/bitnami/openldap CPPFLAGS="-I/opt/bitnami/openldap/include" LDFLAGS="-L/opt/bitnami/openldap/lib -Wl,-rpath,/opt/bitnami/openldap/lib" --enable-modules --enable-slapi --enable-ldap --enable-mdb --with-tls=openssl --with-cyrus-sasl
```

The configure command I used for testing is:

```bash
./configure --prefix=/opt/bitnami/openldap CPPFLAGS="-I/opt/bitnami/openldap/include" LDFLAGS="-L/opt/bitnami/openldap/lib -Wl,-rpath,/opt/bitnami/openldap/lib" --enable-modules --enable-slapi --with-tls=openssl --enable-dnssrv --enable-ldap --enable-mdb --enable-relay --enable-asyncmeta --enable-passwd --enable-null --enable-meta --enable-crypt --disable-cleartext --enable-valsort --enable-unique --enable-homedir --enable-accesslog --enable-dynlist --enable-dyngroup --enable-auditlog --enable-rwm --enable-ppolicy --enable-argon2 --with-argon2=libsodium --with-cyrus-sasl
``` 


The last message should be `Please "make depend" to build dependencies`, which indicates a successful build configuration.. proceed to the next step


If `configure` failed, you need to review the errors, usually a module you specified and is not present in the machine, fixable by installing the missing package.

Consult the manual pages for the requirements in the *Prerequisite software* section, select your version here: https://www.openldap.org/doc.


### 3.2. Build Dependencies

As the message suggests, run the command:
 ```bash
 make depend
 ```

Check for any errors/ warnings, you might need to adjust your `configure` command to accommodate.. 

Once done, we're good to build OpenLDAP! 


#### 3.3. Compile

Simply:

```bash
make
```

Once done, you can test the compiled binaries with:
```bash
make test
```
> Check the README file in `/tests` for more about running the tests...

Don't worry about non-configured failing tests. 


Finally, install:

```bash
# Note the use of elevated privileges
sudo make install

```
This will _install_ our binary and it's dependencies to the `--prefix=` folder we specified earlier with the `configure` command.. which should be `--prefix=/opt/bitnami/openldap`


#### 3.4. Export the package

In order to package OpenLDAP to use it in Bitnami's `bitnami/openldap` image, we need to match our built binaries paths with the original amd64 package, and provide it for the Dockerfile..

> (Optional) To check the original package, download it from the original Dockerfile [find the binary link here](https://github.com/bitnami/containers/blob/main/bitnami/openldap/2.6/debian-11/Dockerfile#L24)
>
> Unzip it and examine the folders..


##### 3.4.1. Match Bitnami's image directories
First, let's `cd` to the output directory:

```bash

cd /opt/bitnami/openldap

# If you're not running as root, you might need to change the folder permissions,
# change user:group to match your systems.
sudo chown user:group * -R
```

You should see a list of directories such as `bin`, `etc` etc... :D 

- Move `ldap.conf` from `./etc/openldap` to `./etc/`
```bash
mv ./etc/openldap/ldap.conf ./etc/
```
- Move `schema` directory from `./etc/openldap` to `./etc/`
```bash
mv ./etc/openldap/schema ./etc/
```
- Remove `./etc/openldap` directory and it's content
```bash
rm -r ./etc/openldap
```
- Add `certs` folder to `etc`:
```bash
mkdir ./etc/certs
```
- Copy `slapd.ldif` to `./share` from either:
  - [this repo](https://github.com/mmgrt/bitnami-openldap-arm64/blob/main/slapd.ldif)
  - Download the amd64 binary from the original Dockerfile [find the binary link here](https://github.com/bitnami/containers/blob/main/bitnami/openldap/2.6/debian-11/Dockerfile#L24), unzip it, and inspect `/files/openldap/share/` you'll find the `slapd.ldif` in question
```bash
nano ./share/slapd.ldif
```
- Create `slapd` directory in `./var/run`
```bash
mkdir ./var/run/slapd
```
Your directory tree should look something like this:

```
# Files omitted for brevity

opt
└── bitnami
    └── openldap
        ├── bin
        ├── etc
        │   ├── certs
        │   ├── ldap.conf
        │   └── schema
        ├── include
        ├── lib
        │   └── pkgconfig
        ├── libexec
        │   └── openldap
        ├── sbin
        ├── share
        │   └── slapd.ldif
        └── var
            └── run
```

We're ready to package it!


##### 3.4.2. Package

Let's now put it all together with `tar`:

```bash
# Change directory up so you are in the `/opt/bitnami` folder
# cd ..
# alternatively:
cd /opt/bitnami

# It's okay to choose your own package name
tar -cvzf openldap-2.6.3-linux-arm64.tar.gz openldap
```

This should output a single file (package), of which we will install in our `bitnami/openldap` image.


## 4. Modify the Dockerfile

> This guide still assumes:
> - Source directory: `/src`
>
> change as you like

In this step, we'll edit the Dockerfile of `bitnami/openldap` to use our own package.

> Note: this can be done on a different host, in this guide, I used a Windows machine to do it, although it can be done on the pi
>
> ⚠️ If you used a Windows machine to modify and build the image, make sure your `git` uses `lf` 
> 
> Append line: `*.txt text eol=lf` to `.gitattributes` file
>
> Or globally:
>
> ```bash
> git config --global core.eol lf
> git config --global core.autocrlf input
> ```
> <br>


### 4.1 Git the bitnami image source:

```bash
cd /src
git clone https://github.com/bitnami/containers.git
# Dive in, choose version
cd containers/bitnami/openldap/2.6/debian-11
```
 
### 4.2 Base image, `gosu`, and `$PATH`

Let's first apply some important changes to the Dockerfile:

<sub>Changes are presented as diff</sub> 

- Change the base image to use arm64 version instead:
```diff
-FROM docker.io/bitnami/minideb:bullseye
+FROM docker.io/bitnami/minideb:latest-arm64
```

- Replace `gosu` with arm64 version:

  [Find a matching version on the official releases page](https://github.com/tianon/gosu/releases)

  In this guide, we'll use v1.14.0 as per the Dockerfile:

```diff
- RUN mkdir -p /tmp/bitnami/pkg/cache/ && cd /tmp/bitnami/pkg/cache/ && \
-    if [ ! -f gosu-1.14.0-154-linux-${OS_ARCH}-debian-11.tar.gz ]; then \
-      curl -SsLf https://downloads.bitnami.com/files/stacksmith/gosu-1.14.0-154-linux-${OS_ARCH}-debian-11.tar.gz -O ; \
-      curl -SsLf https://downloads.bitnami.com/files/stacksmith/gosu-1.14.0-154-linux-${OS_ARCH}-debian-11.tar.gz.sha256 -O ; \
-    fi && \
-    sha256sum -c gosu-1.14.0-154-linux-${OS_ARCH}-debian-11.tar.gz.sha256 && \
-    tar -zxf gosu-1.14.0-154-linux-${OS_ARCH}-debian-11.tar.gz -C /opt/bitnami --strip-components=2 --no-same-owner --wildcards '*/files' && \
-    rm -rf gosu-1.14.0-154-linux-${OS_ARCH}-debian-11.tar.gz
+ RUN mkdir -p /tmp/bitnami/pkg/cache/ && cd /tmp/bitnami/pkg/cache/ && \
+   curl -SsLf https://github.com/tianon/gosu/releases/download/1.14/gosu-arm64 > gosu && \
+   mv gosu /opt/bitnami
```

> https://github.com/tianon/gosu/releases/download/1.14/gosu-arm64.asc is available as well.


- Modify the default environment variables to include `slapd` bin and the libraries:
```diff
ENV APP_VERSION="2.6.3" \
    BITNAMI_APP_NAME="openldap" \
-   PATH="/opt/bitnami/openldap/bin:/opt/bitnami/openldap/sbin:/opt/bitnami/common/bin:$PATH"
+   PATH="/opt/bitnami/openldap/bin:/opt/bitnami/openldap/sbin:/opt/bitnami/common/bin:/opt/bitnami/openldap/lib:/opt/bitnami/openldap/libexec:/opt/bitnami/openldap/libexec/openldap:$PATH"
```

- Optionally -for advanced use only- you can specify the UID:GID for the container user as follow:
```diff
EXPOSE 1389 1636
+RUN chown 1001:995 /opt/bitnami -R
-USER 1001
+USER 1001:995
```
> ⚠️ 
> 
> Editing the group id (`gid`) requires extra change in the setup script.
> 
> Mentioned in [5. Modify libopenldap.sh script](#when-changing-gid)

<sub>Where `1001` is the UID and `995` is the GID<sub>


### 4.3. OpenLDAP package installation

Only single modification remains, which is how the image will get the OpenLDAP binaries we built 

It's up to you on how to deliver it in the Dockerfile.. we'll cover two options, pick one:

#### 4.3.1. via `COPY` command

- First, lets copy the package we built into the `/src/containers/bitnami/openldap/2.6/debian-11` next to the Dockerfile:
```bash
cp /opt/bitnami/openldap-2.6.3-linux-arm64.tar.gz /src/containers/bitnami/openldap/2.6/debian-11
# scp can be used to transfer files between machines with ssh
# scp /opt/bitnami/openldap-2.6.3-linux-arm64.tar.gz username@host:/dir/on/host
# or the way around
# scp username@host:/opt/bitnami/openldap-2.6.3-linux-arm64.tar.gz dir/on/host
```

- Apply Dockerfile changes:

```diff
- RUN mkdir -p /tmp/bitnami/pkg/cache/ && cd /tmp/bitnami/pkg/cache/ && \
- if [ ! -f openldap-2.5.13-5-linux-${OS_ARCH}-debian-11.tar.gz ]; then \
-   curl -SsLf https://downloads.bitnami.com/files/stacksmith/openldap-2.5.13-5-linux-${OS_ARCH}-debian-11.tar.gz -O ; \
-   curl -SsLf https://downloads.bitnami.com/files/stacksmith/openldap-2.5.13-5-linux-${OS_ARCH}-debian-11.tar.gz.sha256 -O ; \
- fi && \
- sha256sum -c openldap-2.5.13-5-linux-${OS_ARCH}-debian-11.tar.gz.sha256 && \
- tar -zxf openldap-2.5.13-5-linux-${OS_ARCH}-debian-11.tar.gz -C /opt/bitnami --strip-components=2 --no-same-owner --wildcards '*/files' && \
- rm -rf openldap-2.5.13-5-linux-${OS_ARCH}-debian-11.tar.gz

+ COPY openldap-2.6.3-linux-arm64.tar.gz /
+ RUN mkdir /opt/bitnami/openldap && mkdir -p /tmp/bitnami/pkg/cache/ && cd /tmp/bitnami/pkg/cache/ && \
+ mv /openldap-2.6.3-linux-arm64.tar.gz . && \
+ tar -zxf openldap-2.6.3-linux-arm64.tar.gz -C /opt/bitnami --no-same-owner --wildcards '*/*' && \
+ rm -rf openldap-2.6.3-linux-arm64.tar.gz
```
> Make sure to change `openldap-2.6.3-linux-arm64.tar.gz` to match your package name.


#### 4.3.2 via HTTP server
- Choose a desired local server (or computer), that can open ports
- Create a directory in `/var/www/html` or any public dir 
- Place your packaged binary in the created dir
- Make sure the permissions are correct by giving anyone the ability to read
- Start your favorite http server in that directory, with external connections allowed, specifying the port
- Take a note of the local IP address and the port
> todo: details missing

- Apply Dockerfile changes:

```diff
- RUN mkdir -p /tmp/bitnami/pkg/cache/ && cd /tmp/bitnami/pkg/cache/ && \
- if [ ! -f openldap-2.5.13-5-linux-${OS_ARCH}-debian-11.tar.gz ]; then \
-   curl -SsLf https://downloads.bitnami.com/files/stacksmith/openldap-2.5.13-5-linux-${OS_ARCH}-debian-11.tar.gz -O ; \
-   curl -SsLf https://downloads.bitnami.com/files/stacksmith/openldap-2.5.13-5-linux-${OS_ARCH}-debian-11.tar.gz.sha256 -O ; \
- fi && \
- sha256sum -c openldap-2.5.13-5-linux-${OS_ARCH}-debian-11.tar.gz.sha256 && \
- tar -zxf openldap-2.5.13-5-linux-${OS_ARCH}-debian-11.tar.gz -C /opt/bitnami --strip-components=2 --no-same-owner --wildcards '*/files' && \
- rm -rf openldap-2.5.13-5-linux-${OS_ARCH}-debian-11.tar.gz

+ RUN mkdir -p /tmp/bitnami/pkg/cache/ && cd /tmp/bitnami/pkg/cache/ && \
+   curl -SsLf http://[your-package-host-ip]:[port]/public_folder/openldap-2.6.3-linux-arm64.tar.gz -O ; \
+   tar -zxf openldap-2.6.3-linux-arm64.tar.gz -C /opt/bitnami --no-same-owner && \
+   rm -rf openldap-2.6.3-linux-arm64.tar.gz
```
> Make sure to change `openldap-2.6.3-linux-arm64.tar.gz` to match your package name.
>
> Also, change the package path (ip, port, folder etc..) to match your server ip.
>
> `localhost` won't work since this will be called from inside the container.



## 5. Modify libopenldap.sh script

We need to make sure that our libs and `slapd` bin paths are exported while setup, this can be done by modifying `libopenldap.sh` file:

```bash
cd /src/containers/bitnami/openldap/2.6/debian-11/rootfs/opt/bitnami/scripts
# nano|vi|code|whatever libopenldap.sh

```

Make the following changes within `ldap_env` function:

```diff
ldap_env() {
     cat << "EOF"
# Paths
...
+export LDAP_LIB_DIR="${LDAP_BASE_DIR}/lib:${LDAP_BASE_DIR}/libexec:${LDAP_BASE_DIR}/libexec/openldap"
...
-export PATH="${LDAP_BIN_DIR}:${LDAP_SBIN_DIR}:$PATH"
+export PATH="${LDAP_LIB_DIR}:${LDAP_BIN_DIR}:${LDAP_SBIN_DIR}:$PATH"
```

> Note: for debugging, you can also export `BITNAMI_DEBUG=true` in this file, and use `LDAP_LOGLEVEL=-1` env in the `docker run` command 

### When changing `gid`:

> The following change is required ONLY if you're using a different `gid` for the container:

```diff
ldap_create_online_configuration() {
    info "Creating LDAP online configuration"

-! am_i_root && replace_in_file "${LDAP_SHARE_DIR}/slapd.ldif" "uidNumber=0" "uidNumber=$(id -u)"
+! am_i_root && replace_in_file "${LDAP_SHARE_DIR}/slapd.ldif" "uidNumber=0" "uidNumber=$(id -u)" && replace_in_file "${LDAP_SHARE_DIR}/slapd.ldif" "gidNumber=0" "gidNumber=$(id -g)"
```


## 6. Build the image!


```bash
docker buildx use mybuilder
docker buildx build --progress=plain --no-cache --rm --push --platform linux/arm64 -t $(NAME):$(VERSION) -t $(NAME):latest .

```

> Assuming you have Docker buildx `mybuilder`, you can simply create one by `docker buildx create -name mybuilder`
>
> Change `mybuilder` to suit your preferences.
>
>
> Added `--progress=plain` to see what errors might occur during scripts running
>
> Change to suit your preferences.
>
>
> Flag `--push` used to push the image to my registry.
>
> Consult buildx docs on how to load the image after building instead of pushing.
>
>
> Flag `--platform linux/arm64` specifies the architecture.
>
> Change to suit your needs.
>
>
> Modify `NAME` and `VERSION` to suit your needs.
>


Take notes of any errors/ warnings you might see, especially in `RUN install_packages ..` and `RUN postunpack.sh` commands..


## 7. Use the image:

Now, you can the use the newly built image tag instead of `bitnami/openldap:latest`, and pass the config you desire, for example:


```bash
docker run -d --name openldap -p 1636:1636 -p 1389:1389 -e "LDAP_ROOT=dc=mydomain,dc=com" -e LDAP_CONFIG_ADMIN_ENABLED=true -e LDAP_USER_DC=users -e TZ=Asia/Riyadh -e LDAP_ADMIN_USERNAME=admin -e "LDAP_ADMIN_PASSWORD=some-strong-pass" -e "LDAP_USERS=myuser" -e "LDAP_PASSWORDS=myuser-password" --mount type=bind,src=/openldap,dst=/bitnami/openldap/ docker.io/mghzawi/bitnami-openldap:latest
```
> Replace tag `docker.io/mghzawi/bitnami-openldap:latest` with the tag you chose earlier..

For more about the env vars you can pass to the container, consult with the official Bitnami README at [Github](https://github.com/bitnami/containers/tree/main/bitnami/openldap), [Docker Hub](https://hub.docker.com/r/bitnami/openldap/)

 

---
---