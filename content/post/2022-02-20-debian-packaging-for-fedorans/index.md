+++
title = "Debian Packaging for Fedorans"
author = ["Bryce Carson"]
date = 2020-02-21
lastmod = 2022-02-21T19:03:17-07:00
tags = ["Debian", "Fedora", "Packaging"]
categories = ["Linux"]
draft = false
weight = 2001
+++

## Preamble {#preamble}

I'm the package maintainer for [SLiM:
Selection on Linked Mutations](https://www.messerlab.org/slim) on Fedora-based, Debian-based,
SUSE-based Linux distributions. I created and have maintained the RPM
package (which targets the Fedora-family and openSUSE) for more than a
year now, and I have also created and maintained the shell script which
builds and installs SLiM for Debian-based platforms. The script is a
little rigid, and it doesn't have any safety included: it will overwrite
binaries from the defunct SLiM login manager (which I'm okay with, but
some particular might not be).


## Packaging Utility-software {#packaging-utility-software}

There is a lot of software that is necessary to package other software.
You'll need the essentials, literally `build-essentials` on Debian, and
other packages to get started. Configure the utility software as well,
as there are few defaults when it comes to packaging in Debian. The next
is obtaining sources, and creating the Debian source control: the set of
sources that you'll create which _control_ the building of the upstream
sources into a Debian package.

It's not alike `rpmbuild` and a .spec file, with the utility software
associated with applying patches. Debian packaging is a monstrous beast
that invokes a number of helpers, wrappers, and other utilities that are
unique to workflows, eras in Debian Developer preferences, and the
Debian Policy Manual. RPM packaging isn't simple either, there is a lot
of policy and it is thankfully well-written and expository, but the
Debian documentation can be a bit opaque and there isn't an exhaustive
manual about packaging itself: the documentation seems to assume that
you're attending DebConf or a user-group meeting and have the manual at
hand and a Debian Developer within earshot to answer questions when a
confusion arises. One of the documents is literally a set of slides that
is provided without an accompanying talk. One of the slides is "let's do
an example" where a user-group would then work on an example, have
questions answered, confusion evaporated, and then move on to the next
topic. That's not available to you if you're learning online by
yourself.

Enough about that.


## Fedora {#fedora}

Fedora includes a lot of software. We don't pride ourselves on the
number of packages we include, but we've still got a lot. That includes
the `apt` package. APT on Fedora?
[Why
is APT in the Fedora repositories?](https://docs.fedoraproject.org/en-US/quick-docs/dnf-vs-apt/#_why_is_apt_in_the_fedora_repositories) Fedora users would like to package
software for Debian users!

-   APT is required for `pbuilder`, which is like `rpmbuild`: it creates a
    chroot and builds the software in isolation from the workstation
    environment.


### Configuring APT {#configuring-apt}

APT will reference `/etc/apt/trusted.gpg.d`, but this is not easily
configured in Fedora 35 still. The APT package in Fedora should suggest
the `debian-keyring` package and use that to configure the sources,
taking (by default) from Sid. Fedora is fast. Debian is slow. We'll use
the unstable sources, because if you're packaging for Debian, on Fedora,
you should be targeting the next Debian release anyways.

```text
[brycecarson@fedora 2022-02-20-debian-packaging-for-fedorans]$ sudo apt update
Get:1 http://deb.debian.org/debian buster InRelease [122 kB]
Err:1 http://deb.debian.org/debian buster InRelease
  The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 648ACFD622F3D138 NO_PUBKEY 0E98404D386FA1D9 NO_PUBKEY DCC9EFBF77E11517
Reading package lists... Done
W: GPG error: http://deb.debian.org/debian buster InRelease: The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 648ACFD622F3D138 NO_PUBKEY 0E98404D386FA1D9 NO_PUBKEY DCC9EFBF77E11517
E: The repository 'http://deb.debian.org/debian buster InRelease' is not signed.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
N: See apt-secure(8) manpage for repository creation and user configuration details.
```

`pbuilder` won't function correctly without being able to download
packages from Debian or Ubuntu for the chroot, so we need therefore a
proper configuration.

I'll use an Ubuntu-specific jammySources.list because I had significant
trouble with Debian sources and their GPG signatures. The difficulty I
had with Debian GPG signatures is the main reason I'm writing this: to
help other Fedorans package for Debian without needing a virtual machine
(if you're already using `pbuilder` you may as well have a chroot on
your workstation instead!).

<div class="html">

&lt;!-- jammySources.list : switch.ca is in Edmonton with 4 Gbps connection. --&gt;

</div>

```text
deb https://mirrors.switch.ca/ubuntu/ jammy main
# Re-compiling Ubuntu software is not what we're doing, we only need a cache of
# binary packages for the chroot to work with, and only if it is a chroot of
# Jammy. APT just needs _something_ to work with or it will complain. Leave the
# deb-src disabled, we don't need to query it which will slow things down.
# deb-src https://mirrors.switch.ca/ubuntu/ jammy main
```

I deleted the default .list from my _etc/apt_ because the mirror will be
slower than the one close to me (which as a relatively fast connection;
the closest is not _always_ the fastest, but a fast and close mirror
will be generally faster than the furthest and fastest mirror you could
choose).

I also deleted all the trusted keys in `/etc/apt/trusted.gpg.d` because
I was having problems with them anyways!

```text
[brycecarson@fedora 2022-02-20-debian-packaging-for-fedorans]$ sudo apt update
Get:1 https://mirrors.switch.ca/ubuntu jammy InRelease [270 kB]
Err:1 https://mirrors.switch.ca/ubuntu jammy InRelease
  The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 871920D1991BC93C
Reading package lists... Done
W: GPG error: https://mirrors.switch.ca/ubuntu jammy InRelease: The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 871920D1991BC93C
E: The repository 'https://mirrors.switch.ca/ubuntu jammy InRelease' is not signed.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
N: See apt-secure(8) manpage for repository creation and user configuration details.
```

The GPG error is what I struggled with for an hour earlier today.
Following every guide I could and reading manual pages did not help.
Obviously Stack Overflow didn't help either. Fedora doesn't have
documentation here either, because as
[Why
is APT in the Fedora repositories?](https://docs.fedoraproject.org/en-US/quick-docs/dnf-vs-apt/#_why_is_apt_in_the_fedora_repositories) explains, `apt` was actually
APT-RPM, which is unmaintained, buggy, etc. and was dropped from Fedora
and now we actually include the real APT for packagers in the Fedora
repositories since Fedora 32.

If we begin reading the Debian SecureApt documentation
[here](https://wiki.debian.org/SecureApt#Signed_Release_files), we'll
encounter this section:

> ```text
> W: GPG error: http://ftp.us.debian.org testing Release: The following signatures
> couldn't be verified because the public key is not available: NO_PUBKEY 010908312D230C5F
> ```

That's the same error we see when we run `sudo apt update` for the first
time, and what will cause `pbuilder` to fail if we try to package some
software before all the utilities are properly configured; any Debian
Developer will know to configure their toolchain. Any... surely.

Here is something _really_ important, I think, that I didn't see
mentioned anywhere on Stack Overflow answers:

> Note that the second half of the long number is the key id of the key
> that apt doesn't know about, in this case that's 2D230C5F.

The _second half_ of the long number is the key ID? That might've been
helpful when following the other guides referring to
`gpg --receive-keys [ e.g. 2D230C5F [...]]` (after reading the GPG
manual you'll know to use "the --keyserver in `dirmngr.conf` instead" of
`--keyserver name`, because that's deprecated, of course and not
mentioned in any recent guides).

---

Perhaps I got ahead of myself. Reading Debian documentation I went and
found the `Release.gpg` file from the mirror I'm using
([Switch Inc.](switch.ca) is hosting it) and installed that to
`/etc/apt/trusted.gpg.d/ca.switch.Release.gpg`. APT still failed with
the same error. Alright, I got a GPG key, put it in the directory, but
APT is still unhappy. What do we do?

Let's note that the
[SecureApt
documentation](https://wiki.debian.org/SecureApt#How_to_find_and_add_a_key) from Debian was last modified in 2022-01, but that
modification only changed the wording in the introduction. The previous
edit was two years prior, in 2020-11. This means the official
documentation hasn't been updated to reflect the deprecation of
`apt-key`.

````text
man apt-key
````

> NAME apt-key - Deprecated APT key management utility


### Why is `apt-key` deprecated? And what do we do about? {#why-is-apt-key-deprecated-and-what-do-we-do-about}

[Askeli
answered this](https://askubuntu.com/questions/1286545/what-commands-exactly-should-replace-the-deprecated-apt-key) rhetorical question on AskUbuntu. The deprecation is
motivated by the chain of trust; just as forensic evidence has a chain
of custody, there is a chain of signatures for ultimate trust in
cryptography.

We can ultimately trust the key from our mirror to sign _all_ packages
from any repository, but that's called "cross-signing" and isn't the
most secure. We should instead modify the list file:

<div class="html">

&lt;!-- jammySources.list : switch.ca is in Edmonton with 4 Gbps connection. --&gt;

</div>

<div class="html">

&lt;!-- Proper use of the Release.gpg to solve the apt-key deprecation. --&gt;

</div>

````text
deb [signed-by=/etc/apt/trusted.gpg.d/ca.switch.Release.gpg] https://mirrors.switch.ca/ubuntu/ jammy main
# Re-compiling Ubuntu software is not what we're doing, we only need a cache of
# binary packages for the chroot to work with, and only if it is a chroot of
# Jammy. APT just needs _something_ to work with or it will complain. Leave the
# deb-src disabled, we don't need to query it which will slow things down.
# deb-src [signed-by=/etc/apt/trusted.gpg.d/ca.switch.Release.gpg] https://mirrors.switch.ca/ubuntu/ jammy main
````

For _my_ comfort-level, I'm fine keeping the GPG key in the trusted
directory since I obtained the key directly from the Switch Inc. website
myself, Switch Inc. is an officially-listed Ubuntu mirror, and I have
deleted the default keyring and have no other source lists.

I ultimately trust Switch Inc. That might be a bad idea. Oh well. I'm
not _installing_ software from them, just building scientific software
for other people in a chroot. It'd be one hell of an exploit. I'd like
to see them try.

Following Askeli's answer, I've edited the .list but I get a different
error now:

> ````text
> [brycecarson@fedora 2022-02-20-debian-packaging-for-fedorans]$ sudo apt update
> E: Invalid value set for option Signed-By regarding source https://mirrors.switch.ca/ubuntu/ jammy (not a fingerprint)
> E: The list of sources could not be read.
> ````

Let's replace the file path with the "fingerprint", the key ID (if these
two are the same thing), and try again. Nope, `991BC93C` is _not_ a
fingerprint. Just the key ID. That's... helpful?

The
[GNUPG
Manual § 4.4 Examples](https://www.gnupg.org/documentation/manuals/gnupg/GPG-Examples.html) shows us `gpg -fingerprint user_ID`, and
presumably the user_ID parameter is the same as the key ID that Debian
documentation refers to (recall "the second half of the long number"?).

`````text
gpg -fingerprint 991BC93C
`````

> [brycecarson@fedora Downloads]$ gpg --fingerprint 991BC93C pub rsa4096
> 2018-09-17 [SC] F6EC B376 2474 EDA9 D21B 7022 8719 20D1 991B C93C uid
> [ unknown] Ubuntu Archive Automatic Signing Key (2018)
> [ftpmaster@ubuntu.com](mailto:ftpmaster@ubuntu.com)

Alright... it is pretty-printed so we need to see how to type this in
the jammySources.list file we created earlier. This
[old, http:// webpage](http://irtfweb.ifa.hawaii.edu/~lockhart/gpg/)
from a Hawaiian educational server says:

> To generate a short list of numbers that you can use via an
> alternative method to verify a public key, use:
> `gpg --fingerprint > fingerprint` This creates the file fingerprint
> with your fingerprint info.

Alright, a fingerprint **_file_**! Great. Creating a .fingerprint file and
changing the extension in the signed-by parameter in the list does _not_
fix the issue. Fun!

It's important to note that removing the [signed-by=/path/to/key.gpg]
from the list and importing the public key with

`````text
gpg --receive-key 991BC93C | sudo apt-key add -
`````

does not work either. The same error concerning "The following
signatures couldn't be verified because the public key is not available:
NO_PUBKEY 871920D1991BC93C" is given.


### Whatever are we to do? {#whatever-are-we-to-do}

We somewhat give up, and we use the deprecated method after re-reading
the Debian documentation and find that we're a half-idiot and were using
the wrong command when piping to `apt-key`:

`````text
[brycecarson@fedora ~]$ gpg --keyserver keyserver.ubuntu.com --receive-keys 991BC93C
gpg: key 871920D1991BC93C: "Ubuntu Archive Automatic Signing Key (2018) <ftpmaster@ubuntu.com>" not changed
gpg: Total number processed: 1
gpg:              unchanged: 1


[brycecarson@fedora ~]$ gpg -a --export 991BC93C | sudo apt-key add -
Warning: apt-key is deprecated. Manage keyring files in trusted.gpg.d instead (see apt-key(8)).
OK
W: The key(s) in the keyring /etc/apt/trusted.gpg.d/ca.switch.Release.gpg are ignored as the file has an unsupported filetype.


[brycecarson@fedora ~]$ sudo apt update
Get:1 https://mirrors.switch.ca/ubuntu jammy InRelease [270 kB]
Get:2 https://mirrors.switch.ca/ubuntu jammy/main amd64 Packages [1,407 kB]
Get:3 https://mirrors.switch.ca/ubuntu jammy/main Translation-en [514 kB]
Fetched 1,922 kB in 2s (866 kB/s)
Reading package lists... Done
Building dependency tree... Done
All packages are up to date.
W: https://mirrors.switch.ca/ubuntu/dists/jammy/InRelease: The key(s) in the keyring /etc/apt/trusted.gpg.d/ca.switch.Release.gpg are ignored as the file has an unsupported filetype.
`````

Okay! We've _finally_ managed to get APT to update its cache, and now we
should be able to use `pbuilder` on Fedora, right?


## Right? {#right}

No. Nothing has changed. We cry.

`````text
[brycecarson@fedora ~]$ sudo pbuilder create --debootstrapopts "--keyring=/usr/share/keyrings/debian-archive-keyring.gpg" --distribution jammy --architecture amd64 --mirror "deb https://mirrors.switch.ca/ubuntu jammy main"
[sudo] password for brycecarson:
dpkg-query: no packages found matching systemd
W: cgroups are not available on the host, not using them.
I: Distribution is jammy.
I: Current time: Mon Feb 21 01:54:36 MST 2022
I: pbuilder-time-stamp: 1645433676
I: Building the build environment
I: running debootstrap
/usr/sbin/debootstrap
E: /var/cache/pbuilder/aptcache/: No such directory
E: debootstrap failed
E: debootstrap.log not present
W: Aborting with an error
`````

Let's try turning caching off entirely, as per the
[man-page](https://manpages.debian.org/testing/pbuilder/pbuilderrc.5.en.html#FORMAT).

`````text
# NOT RUN
sed -i 's/^(APTCACHE=).*$/""/' ~/.pbuilderrc
sudo sed -i 's/^(APTCACHE=).*$/""/' /root/.pbuilderrc
# NOT RUN
`````

Wow... that worked. What sadness for my time.