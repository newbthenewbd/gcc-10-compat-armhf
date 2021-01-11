# gcc-10-compat workaround

Maintained at: <https://salsa.debian.org/rpavlik/gcc-10-compat>

This package exists to fix the buster->bullseye upgrade path, apparently broken
by the libgcc1 (and friends) rename (to libgcc-s1, etc) and removal of
transitional packages.

The packages from this repo are empty except for the documentation and copyright
files: they are only additional "edges" in the apt dependency graph, needed to
allow upgrading from Buster to a new enough snapshot of Bullseye that has the
libgcc renames. They can be removed once the `dist-upgrade`/`full-upgrade` is
complete, and `deborphan` should be able to detect this automatically (they are
in section "oldlibs")

I have only tried this for x86_64, and I left out the packages that only applied
to e.g. MIPS and HPPA. This should really be fixed in the gcc-10 package,
hopefully that will happen before the Bullseye release.

## About the problem

There are a number of bugs filed for this - the one I have submitted this info
to is <https://bugs.debian.org/972820>.

This is an apparent bug with the gcc-10 packages in Debian: they renamed several
packages and removed the transitional packages after Buster and before the
release of Bullseye. The following packages were renamed (to match stated
convention/debian policy more closely, I think):

- libgcc1 -> libgcc-s1
- lib64gcc1 -> lib64gcc-s1 (on 32 bit architectures only)
- lib32gcc1 -> lib32gcc-s1 (on 64 bit architectures only)
- libx32gcc1 -> libx32gcc-s1 (on x86_64 only)
- a few others on less common arches

The newly-renamed packages have a `Provides: libgcc1` (etc.) property, but it
appears that apt is preferring a real (old) libgcc1 package over a virtual
package that is only "provide"d by the newer package, since it's trying to not
break your system. I am not familiar with the details of the internals of
dependency resolution and installation order, but that's how I understand the
problem anyway. This repo provides a "real" libgcc1, etc. package to get apt
over that hurdle.

In general you know you've hit this bug when you see this on trying to do a
dist-upgrade from buster to bullseye (or equivalent in derived distros):

```
The following packages have unmet dependencies:
 libc6-dev : Breaks: libgcc-8-dev (< 8.4.0-2~) but 8.3.0-6 is to be installed
E: Error, pkgProblemResolver::Resolve generated breaks, this may be caused by held packages.
```

In some related cases, apt-get will just suggest removal of a number of desired
packages that still exist in Bullseye, instead of failing like this. I'm not
sure why. Aptitude similarly recommends removing what feels like a large number
of desired packages to fix it.

A commonly-suggested "solution" is `apt-get install gcc-8-base`, which often
results in this suggested-mass-removal. Unfortunately, this is the "accepted
answer" to most users without the experience to make the missing transitional
packages. One reference I found noted to do that *before* starting the upgrade,
which perhaps is enough hint to apt to find a solution in gcc-10? That suggested
"solution" doesn't resolve my test case (below), so perhaps there was a previous
bug resolvable in this way, with similar symptoms.

I have a pretty minimal reproduction/test case at
<https://salsa.debian.org/rpavlik/gcc-upgrade-testcase>

## Instructions

I've pushed this package to the [OpenSuSE Build Service OBS instance][obs], so
there's a repo you can add which will let you finish upgrading without the
unnecessary removals that other tools might suggest. Once the dist-upgrade is
complete, please remove the repo and key from your machine: it's no longer
needed, and it's good to keep the list of repos you use short.

[obs]: https://build.opensuse.org/project/show/home:rpavlik:bullseye-fix

**Before doing this**, in case it's needed by the devs working on fixing it in
Debian, I'd suggest making a copy of `/var/lib/dpkg/status` somewhere. It can be
very useful in reproducing this or other upgrade bugs.

To use, run these as a normal user with sudo permission:

```sh
echo "deb http://download.opensuse.org/repositories/home:/rpavlik:/bullseye-fix/Debian_Testing/ ./" | sudo tee /etc/apt/sources.list.d/bullseye-upgrade-fix.list
curl http://download.opensuse.org/repositories/home:/rpavlik:/bullseye-fix/Debian_Testing/Release.key | sudo tee /etc/apt/trusted.gpg.d/bullseye-upgrade-fix.asc
sudo apt update
```

After running that, you'll have transitional packages available named libgcc1,
etc. that depend on their renamed equivalents in bullseye. This will let your
`apt-get dist-upgrade` or `apt full-upgrade` complete, or at least get past the
problem described above: you'll see

- libgcc-s1 in the list of new packages to install
- libgcc1 in the list of packages that will be upgraded (it's being upgraded to
  this transitional package)

Once the dist-upgrade completes successfully and you've rebooted, you can remove
the repo and key you added above:

```sh
sudo rm /etc/apt/sources.list.d/bullseye-upgrade-fix.list
sudo rm /etc/apt/trusted.gpg.d/bullseye-upgrade-fix.asc
```

If you want to uninstall these empty transitional packages too, I would suggest
letting deborphan do it for you.

Hope this helps!

## References to this problem

- Debian BTS
  - <https://bugs.debian.org/972820>
  - <https://bugs.debian.org/964477>
  - Probably related: <https://bugs.debian.org/972936> and its archived relative
    <https://bugs.debian.org/946285>
  - This might have been the earlier relative of the bug, referenced in various
    places: <https://bugs.debian.org/954826>
- In derivatives
  - <https://forums.kali.org/showthread.php?49704-Troubleshooting-Kali-apt-get-upgrade>
  - <https://forum.sparkylinux.org/index.php?topic=5323.0> - reference is made
    to a "bug in gcc-8 already fixed in unstable", but the symptoms match the
    still-existing bug.
- Elsewhere - Note that the list below includes issues filed in incorrect places
  (e.g. the raspberry pi kernel issue tracker), so don't necessarily add to any
  of these reports.
  - <https://unix.stackexchange.com/questions/592657>
  - <https://unix.stackexchange.com/questions/602827>
  - <https://superuser.com/questions/1577742>
  - <https://github.com/raspberrypi/linux/issues/3812>
  - <https://www.reddit.com/r/debian/comments/h878ci/fullupgrade_to_debian_testing_fails_due_to/>
  - <https://www.reddit.com/r/debian/comments/gwzs5n/just_switched_to_testing_from_stable_am_having/>
  - possibly related: <https://superuser.com/questions/1555536/cannot-solve-the-the-following-packages-have-unmet-dependencies-issue>
