**rkt-go-builder**

- install rkt
- create base os aci     : scripts/build-xenial.aci
- create base go builder : scripts/build-go-build-aci
- compile with the version of go installed in build-go-build-aci

The provided entrypoint script can be modified or replaced at runtime
to perform more complicated build strategies.

With a repo list like the following, you might run multiple builds.

```
    github.com/appc/goaci
    github.com/appc/acserver
    github.com/appc/acpush
    github.com/appc/spec
    github.com/coreos/flannel
    github.com/coreos/rkt
    github.com/davidwalter0/gpg-sign-tool
```



```
    DNS=your-dns-choice
    cat repolist|while read repo; do
      mkdir -p bins/${repo};
      rkt run --dns=${DNS} \
        --volume target,kind=host,source=${PWD}/bins/${repo}\
        --mount volume=target,target=/go/bin \
        aci/go-builder-go1.7.3-linux-amd64.aci \
        -- ${repo};
    done
```



---
**signing**

For testing to simplify key generation, image signing and
verification, the initial version uses bin/gpg-sign-tool a modified
version of the golang tool
(quickpgp)[https://github.com/thwarted/quickpgp]


- set the environment variables for (bin/gpg-sign-tool )[https://github.com/davidwalter0/bin/gpg-sign-tool]
  - [ ] KEY_USE_NAME    : the user name may be human readable - including spaces
  - [ ] KEY_USE_COMMENT : the key use comment
  - [ ] KEY_USE_EMAIL   : the full email address
- bin/gpg-sign-tool genkey aci-signing
- bin/gpg-sign-tool sign aci-file aci-signing.key.asc
- bin/gpg-sign-tool verify aci-file aci-signing.pub.asc

---
**rkt install**

Install rkt using the installer script if not already created.

* https://github.com/coreos/rkt

Install rkt from remote, requires wget or curl

```
    wget https://raw.githubusercontent.com/coreos/rkt/master/scripts/install-rkt.sh
    chmod +x install-rkt.sh
    sudo ./install-rkt.sh
```

```
... packages ...
+ gpg2 --trusted-key RKTs_KEY --verify-files rkt-v{version}.tar.gz.asc
gpg: Note: old default options file...

 You can remove it from your system anytime using: 
      dpkg -r rkt
```

---
**Bootstrap**

The example builder creates both a debian base aci image and golang
builder aci. Replace *ACI_REGISTRY* the registry name of your aci compatible registry


```
    sudo scripts/build-xenial-aci    ACI_REGISTRY
    sudo scripts/build-go-build-aci  ACI_REGISTRY
```


---
*Bug fix*

- There was a run time issue when executing with ubuntu xenial and
  systemd-231. The shared object file libsystemd-shared-231.so wasn't
  in the rkt aci, and including the file, didn't allow systemd-nspawn
  to run. Statically building and replacing the build environment's
  systemd-nspawn with the statically linked version fixes this glitch.

The following outlines the basic steps used to create a statically
linked systemd-nspawn for xenial.

Problem with creating runnable images with systemd-231, unable to find
the shared object from systemd-231


```
    mkdir -p /path/where/to/build/
    cd /path/where/to/build/
    apt-get build-dep systemd
    apt-get source systemd
    cd systemd-231

    # generate config vars for build
    ALLFLAGS=$(DEB_LDFLAGS_SET="$(dpkg-buildflags --get LDFLAGS) --static" dpkg-buildflags|while read line; do echo ${line%%=*}="'${line#*=}'"; done)

    echo export ALLFLAGS=${ALLFLAGS} > exports

    . exports

    # setting the command line and environment variables for static
    # build

    ./autogen.sh
    ./configure --prefix=/usr ${ALLFLAGS}

    make -j 2
    sudo mv $(which systemd-nspawn)  $(which systemd-nspawn).orig
    sudo cp systemd-nspawn /usr/bin/

```
