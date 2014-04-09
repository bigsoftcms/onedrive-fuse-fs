onedrive-fuse-fs
----------------------------------------

Script to mount Microsoft OneDrive (formerly known as SkyDrive) folder as a FUSE
filesystem.

This module is single-threaded and tries to do all operations as synchronously
as possible - i.e. operations do not return until they're sequentially performed
through the OneDrive API, with the exception of metadata caches and any
fuse/kernel caches.

That, lots of limitations, and the fact that API calls over https tend to be
quite slow and frequent even with a keepalive connections, make it unsuitable
for use as a general-purpose fs.

Main purpose of such abstraction is a convenient browsing (in shell or any
general-purpose file manager), transfer or synchronization of files with simple
unix tools like "ls", "cp" or "rsync".

Service was called SkyDrive prior to 2014-02-19, when it got renamed to OneDrive.
This repo was similarly renamed from skydrive-fuse-fs to onedrive-fuse-fs.


Limitations
----------------------------------------

OneDrive is not a filesystem (and in fact, basically a key-value storage), and
doesn't expose anything resembling a posix fs interface, so lots of limitations
apply.

By default (with "api_cache" option), module aggressively caches all retreived
data to avoid time-consuming https requests, so any changes performed on
OneDrive outside of the script, might be invisible until remount or cache
expiration / invalidation.

Even with api_cache disabled, lots of operations will still have a race
conditions in them.
For example, rename operation has to perform 4-6 API calls to check source and
destination paths, whether it's a file/folder metadata update or a "move"
operation, etc, and if something happens with remote paths between these checks,
operation may fail with unpredictable results.

Rename operations are not atomic if destination path exists and is a file.

Write and truncate operations are performed by downloading the whole file,
modifying it locally, uploading file back under temporary name, deleting the
original file and renaming uploaded file to original name.

Not all potential API errors are translated into errno's (and probably not all
of them can be mapped to these at all), and might manifest as EFAULT ("Bad
Address") errors.
Use "--debug" option to see which API calls might result in these - all
fuse-requested operations and resulting API requests are logged there.

uid/gid/mode can potentially be stored in object metadata, but that is not
implemented, and I don't see much point in it.


Usage
----------------------------------------

Requires [python-onedrive](http://pypi.python.org/pypi/python-onedrive/) module
installed, use any of these lines:

	pip install 'python-onedrive[conf]'   # system-wide by default, probably needs root
	pip install --user 'python-onedrive[conf]'   # unprivileged install to ~/
	easy_install python-onedrive PyYAML   # try to avoid that one

("python-onedrive[conf]" installs python-onedrive + PyYAML, latter being is a
requirement for configuration-file handling in python-onedrive; install it in a
similar way if you're using `python setup.py install` or some older packaging
tools)

Before using this module, configuraton file with authentication data must be
created, as described in python-onedrive documentation
[here](https://github.com/mk-fg/python-onedrive#command-line-usage).

Usage from checkout (no installation - apart from aforementioned module(s) -
necessary), can be run from unprivileged user:

	% ./onedrivefs /mnt/onedrive

Install to system-wide $PATH (might need system-wide fusepy module as well):

	# pip install fusepy
	# install -m755 onedrivefs /usr/local/sbin/

Use from $PATH with `mount` command:

	# mount -t fuse.onedrivefs ~/.lcrc /mnt/onedrive

Examples with fstab(5) syntax (any one of these will do):

	/var/lib/onedrive.yaml /mnt/onedrive fuse.onedrivefs defaults 0 0
	/var/lib/onedrive.yaml:Pics /mnt/onedrive fuse.onedrivefs defaults 0 0

Available mount options can be found in `onedrivefs --help` output
(onedrive-specific) and `man mount.fuse` (more general fuse options).

Latter example mounts "Pics" OneDrive folder instead of root.
"subdir" fuse module can be also used to the same effect.

Make sure configuration file (`/var/lib/onedrive.yaml` in fstab examples above)
is not accessible to anyone but root (or dedicated user), and is writable -
refreshed access tokens will be stored there.
