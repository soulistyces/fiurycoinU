# How to build Fiurycoin

## Requirements:

- Ubuntu 22 LTS or WSL2 running Ubuntu 22 LTS
- Setup Docker Container (additional setup for WSL2: [https://docs.docker.com/desktop/windows/wsl/])
- Be sure to keep Docker online!

## Build process:

Create a clean folder where building takes place. Copy over all necessary files.

```bash
$ mkdir builds && cd builds
$ git clone https://github.com/FiurycoinIO/FiuryCoin.git
$ cp FiuryCoin/contrib/gitian-build.py .
```

In the same folder create a file with name `0001-Disable-apt-cacher.patch` and fill as shown below.

```bash
$ touch 0001-Disable-apt-cacher.patch
$ nano 0001-Disable-apt-cacher.patch
# fill with the text below. once filled, ctrl + o to save, ctrl + c to close.
```
```patch
From b8d662f6b721b0f4992ca234d5db312f3564b08a Mon Sep 17 00:00:00 2001
From: David Burkett <davidburkett38@gmail.com>
Date: Fri, 19 Nov 2021 10:58:23 -0500
Subject: [PATCH] Disable apt cacher

---
 bin/make-base-vm | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)

diff --git a/bin/make-base-vm b/bin/make-base-vm
index 30e4fbf..be0b168 100755
--- a/bin/make-base-vm
+++ b/bin/make-base-vm
@@ -198,7 +198,7 @@ if [ $DOCKER = "1" ]; then
 FROM $base_image
 
 ENV DEBIAN_FRONTEND=noninteractive
-RUN echo 'Acquire::http { Proxy "$MIRROR_BASE"; };' > /etc/apt/apt.conf.d/50cacher
+# RUN echo 'Acquire::http { Proxy "$MIRROR_BASE"; };' > /etc/apt/apt.conf.d/50cacher
 RUN apt-get update && apt-get --no-install-recommends -y install $addpkg
 
 RUN useradd -ms /bin/bash -U $DISTRO
-- 
2.25.1
```

Now run the setup command. REPLACE `NAME` with the builder's name and `BUILD_VERSION` with git tag version, without the `v` prefix.

```bash
$ ./gitian-build.py --setup NAME BUILD_VERSION --docker --disable-apt-cacher
```

Now you may run the build command. Make sure to replace `NAME` & `BUILD_VERSION`. Also replace `THREADS` with number of available CPU threads, and `MEMORY` with available RAM in MB.

```bash
$ ./gitian-build.py --docker --build NAME BUILD_VERSION -j THREADS -m MEMORY --no-commit
```

----------------------

### Build Script features

There are three different build modes, `--build`, `--sign` or `--buildsign`.

Building for specific target OS can be specified by parameter `-o` where `m` is macOS, `w` is Windows and `l` is Linux.

`-j` to specific number of build threads (there is a known issue where too many threads may cause build failures).

`-m` to specify amount of memory to allocate in MB.

**Additional help?**
```bash
$ ./gitian-build.py --help
```

----------------------

## Windows Codesigning

```bash
$ mkdir signwin
$ cp FiuryCoin/contrib/windeploy/detached-sig-create.sh signwin
$ cp gitian-builder/inputs/fiurycoin-{VERSION}-win-unsigned.tar.gz signwin # replace {VERSION} with build version without v 
$ cd signwin
$ tar xf fiurycoin-{VERSION}-win-unsigned.tar.gz
$ openssl pkcs12 -in PATH_TO_P12_CERT -nodes -out private.key -nocerts
# enter p12 key password
$ ./detached-sig-create.sh -key private.key
# if succeeded continue
$ cd ..
$ mv signwin/signature-win.tar.gz fiurycoin-detached-sigs
$ cd fiurycoin-detached-sigs
$ rm -rf win
$ tar xf signature-win.tar.gz
$ rm signature-win.tar.gz
$ cd ..
```

If you've codesigned macOS and generated detached sigs, please commit and push. Otherwise continue with instructions for macOS Codesigning on a Mac. 

------------------------

## macOS Codesigning

Once you have built a macOS unsigned builds, a new `fiurycoin-osx-unsigned.tar.gz` file will be created in `gitian-builder/inputs`. Copy this file over to a macOS Device. I would reccomend sha256sum-ming the file before and after sending, to review integrity of the file.

Please make sure to have setup the codesigning Mac as a developer system, with xcode & xcode command line tools enabled. Please ensure that the  macOS codesigning certificate is installed on the system.

On macOS codesigning system:

```bash
# copy fiurycoin-osx-unsigned.tar.gz to a new temporary folder
# cd into temporary folder, then proceed
$ tar xf fiurycoin-{VERSION}-osx-unsigned.tar.gz # replace {VERSION} with version number with no v
$ rm fiurycoin-{VERSION}-osx-unsigned.tar.gz
$ ./detached-sig-create.sh -s "Developer ID Application: Fiurycoin Consulting, LLC (4H4D2P83CX)"
```

Now a manual deterministic disk image (dmg) creation is required.

First time setup for codesigner, requires creation of app-specific-password via Apple ID website.
Once password is obtained, save it to the macOS Keychain for future reference:

```bash
$ xcrun altool -u "<apple-id-email>" -p "<app-specific-password>" --store-password-in-keychain-item "<apple-id-notarisation-app-specific-password>"
# If <team-id-shortcode> is unknown for team accounts with multiple organisations, query:
$ xcrun altool --list-providers -u "<apple-id-email>" -p "@keychain:<apple-id-notarisation-app-specific-password>"
```

Notarize the disk image:

```bash
$ zip -r Fiurycoin-Qt.app.zip dist/Fiurycoin-Qt.app
$ xcrun altool --notarize-app --primary-bundle-id "org.fiurycoin.Fiurycoin-Qt" -u "<apple-id-email>" -p "@keychain:<apple-id-notarisation-app-specific-password>" --asc-provider <team-id-shortcode> -t osx -f Fiurycoin-Qt.app.zip
# The notarization takes a few minutes. Check the status:
$ xcrun altool --notarization-info <request-uuid> -u "<apple-id-email>" -p "@keychain:<apple-id-notarisation-app-specific-password>" --asc-provider <team-id-shortcode>
# If notarization fails, query log with uuid:
$ xcrun altool --notarization-info <request-uuid> -u "<apple-id-email>" -p "@keychain:<apple-id-notarisation-app-specific-password>" --asc-provider <team-id-shortcode>
```

Staple the notarization ticket onto the application

```bash
$ xcrun stapler staple dist/Fiurycoin-Qt.app
```

Verify if app is sucessfully notarized:
```bash
$ codesign -vvvv dist/Fiurycoin-Qt.app 
  dist/Fiurycoin-Qt.app: valid on disk
  dist/Fiurycoin-Qt.app: satisfies its Designated Requirement
$ spctl -a -vvvv dist/Fiurycoin-Qt.app
  dist/Fiurycoin-Qt.app: accepted
  source=Notarized Developer ID
  origin=Developer ID Application: Fiurycoin Consulting, LLC (4H4D2P83CX)
```

Copy `signature-osx.tar.gz` & `dist/Fiurycoin-Qt.app/Contents/CodeResources` back to build system, inside of the `fiurycoin-detached-sigs` repo. This repo should be in the same folder `gitian-builder` is located in. Again verify sha256sum before and after copy. In the example below, both files were moved to a temporary folder called signmac, however this may be different for you.

```bash
$ mv signmac/signature-osx.tar.gz fiurycoin-detached-sigs
$ rm -rf osx
$ cd fiurycoin-detached-sigs
$ tar xf signature-osx.tar.gz
$ rm signature-osx.tar.gz
$ mv signmac/CodeResources fiurycoin-detached-sigs/osx/dist/Fiurycoin-Qt.app/Contents
$ cd ..
```

------------------------

## Building Signed Binaries

Once you have built and git push-ed both Windows and macOS detached sigs (see previous two sections), you are ready to finally build signed binaries for distribution. Make sure to create a tag in the `fiurycoin-detached-sigs` with the build version as the tag name (e.g. `v2.1.0`). This time with a `v` prefix.

Now back to building :)

```bash
$ ./gitian-build.py --docker --sign NAME BUILD_VERSION -j THREADS -m MEMORY --no-commit
# push gitian sigs
$ cd gitian.sigs.fiurycoin
$ git add .
$ git commit -S -m "{NAME'}s v{BUILD_VERSION} win & osx gitian sigs"
$ git push
```

Now all you need to do is upload your binaries, and include a copy of the sha256sum files.
For extra security, you may also provide signed asc files, signed with a trustworthy gpg key. 