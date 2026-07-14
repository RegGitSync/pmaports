# Filesystem Hierarchy Standard

The [Filesystem Hierarchy Standard](https://specifications.freedesktop.org/fhs/latest/)
(FHS) is the standard that defines how a filesystem tree should be laid out.
For packagers, this means defining the paths where files should be located and
which directories are good for use. The purpose of this document is to clarify
these paths and explain where to put what during packaging and usage. Anything
not listed here can be assumed to follow the FHS normally.

These rules do not apply to packages forked from Alpine or that are replacing
a files that Alpine provides due to limitations in apk-tools.

## Recommended Paths

These are paths that are allowed in packaging and are the recommended paths to
use over any others. If a package installs files in these directories, then
the package is following policy.

### /usr/

`/usr/` is the base path of the system in packaging. If possible, all files in
a package should be located under this directory. Any standard directory under
`/usr/`, unless explicitly forbidden, is allowed in packaging.

### /usr/bin/ and /usr/sbin

If possible, packages should be configured to solely use `/usr/bin/`.

postmarketOS is in an interesting state where it is usr-merged (meaning the
flat paths on `/` are symlinks to their equivalents under `/usr/`), but not
bin-merged (meaning `/usr/bin/` and `/usr/sbin/` are separate directories.)

Since all major distributions do both the usr-merge and bin-merge, copying
packaging logic from said distros can result in non-functional packages on
postmarketOS. Special care has to be put into making sure that binaries are
placed in the directory expected by upstreams. Generally, this is handled by
the build-system used by a package, but, if installing manually, make sure to
check upstream sources.

An example of this is `installkernel`. The Linux kernel expects
`installkernel` to be located at `/sbin/`. On usr-merged systems, this is
`/usr/sbin/`. But, since most distros are *also* bin-merged, its not
uncommon to see the install path on other distributions' packages instead be
`/usr/bin/`, which wouldn't work on postmarketOS.

## Paths to Avoid

These are paths that should not be in packages. Packages will not be blocked
on their inclusion, but it is highly recommended to remove and handle these
paths if possible.

### /etc/

This is generally the path where a lot of software installs their
configurations.

This path should be avoided, as only `/usr/` is allowed in packaging. It may
be that a program actually supports configuration under a named directory
under `/usr/lib/` or `/usr/share/`. Make sure to check upstream if this is
the case. If it is not, please read the section on tmpfiles.d.

### /var/

This is similar to `/etc/`, but is generally more often used to store data
that the user isn't expected to modify. Unlike `/etc/`, there usually is not
an alternative packaging path for files under this directory. As such, please
read the section on tmpfiles.d.

## Forbidden Paths

These paths are never allowed in packaging. Package inclusion will be blocked
if these directories exist in the package. This is not an exhaustive list, but,
generally these are the paths that show up the most in packages. As long as
the build-system is configured properly, no violations should occur.

### /bin/, /sbin/, /include/, and /lib/

These paths are the old raw root paths. They have been replaced by their
`/usr/` counterparts.

### /home/

This path is solely for user home directories. No file should be installed
here. If a file is needed to be installed in the user home directory, use
a skel directory instead.

### /opt/

This path is an old packaging directory commonly used by Fedora Linux.

### /tmp/

This path is where temporary files are created on the system and is usually
mounted as a tmpfs, so the files are not persistent.

### /usr/etc/

This path is an OpenSUSE-specific configuration path that is a mapping of
`/etc/` onto `/usr/`. Generally, `/usr/lib/` or `/usr/share/` fulfills the
same role as this path and build systems usually have a config to choose
which to use.

### /usr/lib64/

This path is a Debian-specific library path that should not be used on
postmarketOS. `/usr/lib/` should always be used instead.

### /usr/local/

This path is an old style user override directory meant to be a user version
of the base `/usr/` path. This is depreciated in postmarketOS and has never
been a valid path to install files.

## tmpfiles.d

For handling files or directories that install to forbidden paths, packagers
are expected to check if upstreams support alternative packaging paths, such
as `/usr/lib/` or `/usr/share/`. Sometimes, packages do not support
alternative paths. This is a bug upstream, so please file an issue with the
upstream project if there is not already one about lack of a configuration
directory under `/usr/`.

As a workaround, postmarketOS supports
[tmpfiles.d](https://www.freedesktop.org/software/systemd/man/tmpfiles.d.html)
on systemd. tmpfiles.d is a powerful tool that allows creating files and
symlinks, copying files, and much more at runtime. When working with
tmpfiles.d for configs, the best practice is to install the config that would
go in `/etc/` or `/var/` to `/usr/share/<pkgname>/` and then create a
tmpfiles.d config to symlink that file into the expected path. Sometimes
programs do not accept symlinked files. In that case, changing the tmpfiles.d
config to copy the file to the expected path instead is an alternative.

For installing files paths other than `/etc/` or `/var/`, use your best
judgement for where you think the file would go if installed correctly.
Reviewers may recommend a different path at review time if the chosen path may
not be the best path for the file..

Please make sure to install the tmpfiles.d config into the main package and
not in any subpackages. This will allow both systemd and OpenRC to use it once
OpenRC installs get support for tmpfiles.d.

Because OpenRC installs do not currently have logic to handle tmpfiles.d, the
config will need to be copied into an openrc subpackage at its original
expected install location to only be installed on OpenRC.
