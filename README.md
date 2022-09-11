bitnami-openldap-arm64

TL;DR 

`bitnami/openldap` that runs on arm64-based CPUs such as Raspberry Pi.
---------------------------------------------------------------------------------------------------------

>Not affiliated with Bitnami.

>Use at own risk, no gurantes provided.

>⚠️ Don't use on production unless you know what are doing!


This image is based on `bitnami/openldap` [Docker Hub](https://hub.docker.com/r/bitnami/openldap/) | [Github](https://github.com/bitnami/containers/tree/main/bitnami/openldap)

As there is no official support yet for arm64 (see: https://github.com/bitnami/bitnami-docker-openldap/issues/18, https://github.com/bitnami/charts/issues/7305), I decided to make my own, and document the steps for others and for my future self :)

# This document covers:
- Compile (build) *OpenLDAP 2.6* for arm64
- Package the compiled binary 
- Modify `bitnami/openldap`s Dockerfile
- Deploy the modified image to a pi


# Contribute
Contributions are needed espicially given the vast options OpenLDAP offers
If you had an issue, and fixed it: please feel free to open a pull request, if you're lazy, just create an issue with label `solved-issue` describing what you needed to do in order to fix it/ mention any resources you used.

If you still facing issues, you can create an issue with label `help-needed` and hope for the best :D 
Use this as a last resort, search your favourite engine and read the manual instead.

> Note: I used my Pi to compile the source, so I didn't need extra setup for targetting arm64.


It's a good idea now to visit: https://www.openldap.org/doc/
Select the version you have, and navigate to the *Running configure* manual.. for 2.6: https://www.openldap.org/doc/admin26/install.html#Running%20configure

This guide uses OpenLDAP 2.6.3
If you're using a different version, you should check the steps here with the respective manual, and follow it along


## fs
This guide assumes:
- Source directory: `/src`, change as you like
- Output directory: `/opt/bitnami/openldap`, DON'T CHANGE

⚠️ Changing the output directory will cause a runtime error (more info below) todo: provide ref

## Install dependencies:

The following packages are needed to compile OpenLDAP:
```bash
apt install -y build-essential libsasl2-dev libltdl-dev libevent-dev libltdl7 openssl libssl-dev libcrack2-dev libwrap0-dev libevent-dev
```

I'm not 100% sure on which packages exactly are needed to compile OpenLDAP,
Also, you might need extra packages based on your module selection..
For example, if you want to authenticate with argon2, you'll need to install either `libargon2-dev` or `libsodium-dev`. 

To see what packages you might need, we'll need the source and use the `configure` script..


## Git the source:

[Official repos can be checked @ openldap.org](https://www.openldap.org/software/repo.html)
```bash

cd /src
git clone https://git.openldap.org/openldap/openldap
cd openldap

```

## Build OpenLDAP:

In order to build OpenLDAP, we'll need to:

### 1. Configure

From the source directory, run:

```bash
# Displays information about the available options/ modules

./configure -h

```

Read through the displayed options and take a note of what you need for your installation, for example, if you want to enable argon2 auth with `libsodium` as library, then you need to pass `--enable-argon2 --with-argon2=libsodium` to the `configure` script.

> Play around with `configure` and see if your args are correct by using the `--no-create` arg.. when added, the `configure` will just check against your system and displays errors without writing any config/ cache files.

#### Some important args MUST pass to `configure`:

- `--prefix=/opt/bitnami/openldap` sets the installation output to `/opt/bitnami/openldap` (Required, litral)
> ⚠️ --prefix=/opt/bitnami/openldap is mandatory for this setup, basically the whole software will be installed in `/opt/bitnami/openldap` to be packaged later, this path MUST match the installation path in the cotnainer, which is `/opt/bitnami/openldap` per bitnami image.
> In the configure source I also found that this prefix is being used at runtime to find the slapd.conf file, not setting this correctly will successfully build the image but will cause runtime error: `bind(8): errno=2 (No such file or directory)`

- `CPPFLAGS="-I/opt/bitnami/openldap/include" LDFLAGS="-L/opt/bitnami/openldap/lib -Wl,-rpath,/opt/bitnami/openldap/lib"` sets linker/ compiler flags to include the lib directory (Required, litral)
> ⚠️ Without this flag the binary will fail to locate lib files, and runtime errors such as `libldap-whatever.so.0: cannot open shared object file: No such file or directory` will occur.

- `--enable-modules` is requried if you want to enable modules (Conditional)


Note: We maybe should use the original `configure` args used to build the OpenLDAP binary in the bitnami image, but I can't find it..

The configure command I used for this image is:

```bash
./configure --prefix=/opt/bitnami/openldap CPPFLAGS="-I/opt/bitnami/openldap/include" LDFLAGS="-L/opt/bitnami/openldap/lib -Wl,-rpath,/opt/bitnami/openldap/lib" --enable-modules --enable-slapi --enable-dnssrv --enable-ldap --enable-mdb --enable-relay --enable-asyncmeta --enable-passwd --enable-null --enable-meta --enable-crypt --disable-cleartext --enable-valsort --enable-unique --enable-homedir --enable-accesslog --enable-dynlist --enable-dyngroup --enable-auditlog --enable-rwm --enable-ppolicy --enable-argon2 --with-argon2=libsodium
```
todo: Add SSL!
todo: Disable un-needed options!

The last message should be `Please "make depend" to build dependencies`, which indicates a successfull build cofiguration.. procceed to the next step

If `configure` failed, you need to review the errors, usually a module you specified and is not present in the machine, fixable by installing the missing package, consult the manual pages for the requirements in the *Prerequisite software* pages, select your version here: https://www.openldap.org/doc.


### 2. Build Dependencies

As the message suggests, run the command:
 ```bash
 make depend
 ```

Check for any errors/ warnings, you might need to adjust your `configure` command to accomodiate.. 

Once done, we're good to build OpenLDAP! 

#### 3. Compile

Simply:

```bash
make
```

Once done, you can test the compiled binaries with:
```bash
make test
```
> Check the README file in `/tests` for more about running the tests...

Don't worry about non-cofigured failing tests. 

Finally:
```bash
# Note the use of elevated prevligies
sudo make install

```
This will _install_ our binary and it's dependencies to the `--prefix=` folder we specified earlier with the `configure` command.. which should be `--prefix=/opt/bitnami/openldap`

#### 4. Export the package

Now, let's package OpenLDAP to use it in Bitnami's `bitnami/openldap` image.
For that, we need to match our built binaries paths with the original amd64 package, and provide it for the Dockerfile..

> To check the original package, download it from the originl Dockerfile [find the binary link here](https://github.com/bitnami/containers/blob/main/bitnami/openldap/2.6/debian-11/Dockerfile#L24)
> Unzip it and examine the folders..


##### 1. Match bitnami directories
First, let's `cd` to the output directory:

```bash
cd /opt/bitnami/openldap
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
(Files omitted for brevity)

opt
└── bitnami
    └── openldap
        ├── bin
        ├── etc
        │   ├── certs
        │   ├── ldap.conf
        │   └── schema
        ├── include
        ├── lib
        │   └── pkgconfig
        ├── libexec
        │   └── openldap
        ├── sbin
        ├── share
        │   └── slapd.ldif
        └── var
            └── run


We're ready to package it!


##### 2. Package

Change directory up so you are in the `/opt/bitnami` folder
Let's now put it all together with `tar`:

```bash
cd /opt/bitnami

# It's okay to choose your own package name
tar -cvzf openldap-2.6.3-linux-arm64.tar.gz openldap
```

This should output a single file (package), of which we will install in our `bitnami/openldap` image.

## Modify the Dockerfile

> This guide still assumes:
> - Source directory: `/src`, change as you like

In this step, we'll edit the Dockerfile of `bitnami/openldap` to use our own package.

> Note: this can be done on a different host, in this guide, I used a Windows machine to do it, although it can be done on the pi

First, let's _git_ the bitnami image source:
```bash
cd /src
git clone https://github.com/bitnami/containers.git
# Dive in 
cd containers/bitnami/openldap/2.6/debian-11
```
 
### Base image, `gosu`, and $PATH

Let's first apply some important chages to the Dockerfile:

(Chages are presented as diff)

- Change the base image to use arm64 version instead:
```diff
-FROM docker.io/bitnami/minideb:bullseye
+FROM docker.io/bitnami/minideb:latest-arm64
```

- Replace `gosu` with arm64 version:

[Find a matching version on the official releases page](https://github.com/tianon/gosu/releases)

In this guide, we'll use v1.14.0 as per the Docker file:

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
+   mv gosu /opt/bitnami --no-same-owner
```

- Modify the default environment variables to include `slapd` bin and the libraries:
```diff
-    PATH="/opt/bitnami/openldap/bin:/opt/bitnami/openldap/sbin:/opt/bitnami/common/bin:$PATH"
+    PATH="/opt/bitnami/openldap/bin:/opt/bitnami/openldap/sbin:/opt/bitnami/common/bin:/opt/bitnami/openldap/lib:/opt/bitnami/openldap/libexec:/opt/bitnami/openldap/libexec/openldap:$PATH"
```

- Optionally -for advanced use only- you can specify the UID:GID for the container user as follow:
```diff
-USER 1001
+USER 1001:995

+RUN chown 1001:995 /opt/bitnami -R
```

(Where `1001` is the UID and `995` is the GID)


### OpenLDAP package installation

Only single modification remains, which is to specify how the image will get the OpenLDAP binaries we built 
It's up to you on how to deliver it in the Dockerfile.. we'll cover two options:

#### 1. via COPY command

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
+ RUN mkdir -p /tmp/bitnami/pkg/cache/ && cd /tmp/bitnami/pkg/cache/ && \
+ mv /openldap-2.6.3-linux-arm64.tar.gz . && \
+ tar -zxf openldap-2.6.3-linux-arm64.tar.gz -C /opt/bitnami --no-same-owner && \
+ rm -rf openldap-2.6.3-linux-arm64.tar.gz
```
> Make sure to change `openldap-2.6.3-linux-arm64.tar.gz` to match your package name.


#### via HTTP server
- Choose a desired local server (or computer), that can open ports
- Create a directory in `/var/www/html` or any public dir 
- Place your packaged binary in the created dir
- Make sure the permissions are correct by giving anyone the ability to read
- Start your favourite http server in that directory, with external connections allowed, specifying the port
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
+   curl -SsLf http://[your-package-host-ip]:[port]/[pubfolder]/[somefolder..etc]/openldap-2.6.3-linux-arm64.tar.gz -O ; \
+   tar -zxf openldap-2.6.3-linux-arm64.tar.gz -C /opt/bitnami --no-same-owner && \
+   rm -rf openldap-2.6.3-linux-arm64.tar.gz
```
> Make sure to change `openldap-2.6.3-linux-arm64.tar.gz` to match your package name.

## Modify Makefile

```bash
cd /src/containers/bitnami/openldap/2.6/debian-11
# nano|vi|code|whatever Makefile

```


To make it simplir to build, make use of existing Makefile, add:

```diff
+ buildx:
+        docker buildx use mybuilder
+        docker buildx build --progress=plain --no-cache --rm --push --platform linux/arm64 -t $(NAME):$(VERSION) -t $(NAME):latest .
```

> Assuming you have Docker buildx `mybuilder`, you can simply create one by `docker buildx create -name mybuilder`
> Change `mybuilder` to suit your prefs.

> Added `--progress=plain` to see what errors might occur during scripts running
> Change to suit your prefs.

> Flag `--push` used to push the image to my registry.
> Cosult buildx docs on how to load the image after building instead of pushing.

> Flag `--platform linux/arm64` specifies the archs.
> Change to suit your needs.

> Modify `NAME` and `VERSION` to suit your needs.


## Build the image!

```bash
# Note the x
make buildx
```

Take notes of any errors/ warnings you might see, espicially in `RUN install_packages ..` and `RUN /opt/bitnami/scripts/openldap/postunpack.sh` commands...

> For any errors/ issues, check [#Contribute](Contribute)


## Use the image:

Now, you can the image tag instead of `bitnami/openldap:latest`, and pass the config you desired:


```bash
docker run -d --name openldap -p 1636:1636 -p 1389:1389 -e "LDAP_ROOT=dc=mydomain,dc=com" -e LDAP_CONFIG_ADMIN_ENABLED=true -e LDAP_USER_DC=users -e TZ=Asia/Riyadh -e LDAP_ADMIN_USERNAME=admin -e "LDAP_ADMIN_PASSWORD=some-strogn-pass" -e "LDAP_USERS=myuser" -e "LDAP_PASSWORDS=myuser-password" --mount type=bind,src=/openldap,dst=/bitnami/openldap/ docker.io/mghzawi/bitnami-openldap:latest
```
> Replace tag `docker.io/mghzawi/bitnami-openldap:latest` with the tag you chose earlier..

For more about the env vars you can pass to the container, connsult with the official Bitnami README at [Github](https://github.com/bitnami/containers/tree/main/bitnami/openldap), [Docker Hub](https://hub.docker.com/r/bitnami/openldap/)


check todos
.
...