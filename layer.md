# Image Layer Filesystem Changeset

This document describes how to serialize a filesystem and filesystem changes like removed files into a blob called a layer.
One or more layers are ordered on top of each other to create a complete filesystem.
This document will use a concrete example to illustrate how to create and consume these filesystem layers.

## Distributable Format

Layer Changesets for the [mediatype](./media-types.md) `application/vnd.oci.image.layer.tar+gzip` MUST be packaged in [tar archive][tar-archive].
Layer Changesets for the [mediatype](./media-types.md) `application/vnd.oci.image.layer.tar+gzip` MUST NOT include duplicate entries for file paths in the resulting [tar archive][tar-archive].

## Change Types

Types of changes that can occur in a changeset are:

* Additions
* Modifications
* Removals

Additions and Modifications are represented the same in the changeset tar archive.

Removals are represented using "[whiteout](#whiteouts)" file entries (See [Representing Changes](#representing-changes)).

### File Types

Throughout this document section, the use of word "files" or "entries" includes:

* regular files
* directories
* sockets
* symbolic links
* block devices
* character devices
* FIFOs

### File Attributes

Where supported, MUST include file attributes for Additions and Modifications include:

* Modification Time (`mtime`)
* User ID (`uid`)
    * User Name (`uname`) *secondary to `uid`*
* Group ID (`gid `)
    * Group Name (`gname`) *secondary to `gid`*
* Mode (`mode`)
* Extended Attributes (`xattrs`)
* Symlink reference (`linkname`)

[Sparse files](https://en.wikipedia.org/wiki/Sparse_file) SHOULD NOT be used because they lack consistent support across tar implementations.

## Creating

### Initial Root Filesystem

The initial root filesystem is the base or parent layer.

For this example, an image root filesystem has an initial state as an empty directory.
The name of the directory is not relevant to the layer itself, only for the purpose of producing comparisons.

Here is an initial empty directory structure for a changeset, with a unique directory name `rootfs-c9d-v1`.

```
rootfs-c9d-v1/
```

### Populate Initial Filesystem

Files and directories are then created:

```
rootfs-c9d-v1/
    etc/
        my-app-config
    bin/
        my-app-binary
        my-app-tools
```

The `rootfs-c9d-v1` directory is then created as a plain [tar archive][tar-archive] with relative path to `rootfs-c9d-v1`.
Entries for the following files:

```
./
./etc/
./etc/my-app-config
./bin/
./bin/my-app-binary
./bin/my-app-tools
```

### Populate a Comparison Filesystem

Create a new directory and initialize it with an copy or snapshot of the prior root filesystem.
Example commands that can preserve [file attributes](#file-attributes) to make this copy are:
* [cp(1)](http://linux.die.net/man/1/cp): `cp -a rootfs-c9d-v1/ rootfs-c9d-v1.s1/`
* [rsync(1)](http://linux.die.net/man/1/rsync):  `rsync -aHAX rootfs-c9d-v1/ rootfs-c9d-v1.s1/`
* [tar(1)](http://linux.die.net/man/1/tar): `mkdir rootfs-c9d-v1.s1 && tar --acls --xattrs -C rootfs-c9d-v1/ -c . | tar -C rootfs-c9d-v1.s1/ --acls --xattrs -x` (including `--selinux` where supported)

Any [changes](#change-types) to the snapshot MUST NOT change or affect the directory it was copied from.

For example `rootfs-c9d-v1.s1` is an identical snapshot of `rootfs-c9d-v1`.
In this way `rootfs-c9d-v1.s1` is prepared for updates and alterations.

**Implementor's Note**: *a copy-on-write or union filesystem can efficiently make directory snapshots*

Initial layout of the snapshot:

```
rootfs-c9d-v1.s1/
    etc/
        my-app-config
    bin/
        my-app-binary
        my-app-tools
```

See [Change Types](#change-types) for more details on changes.

For example, add a directory at `/etc/my-app.d` containing a default config file, removing the existing config file.
Also a change (in attribute or file content) to `./bin/my-app-tools` binary to handle the config layout change.

Following these changes, the representation of the `rootfs-c9d-v1.s1` directory:

```
rootfs-c9d-v1.s1/
    etc/
        my-app.d/
            default.cfg
    bin/
        my-app-binary
        my-app-tools
```

### Determining Changes

When two directories are compared, the relative root is the top-level directory.
The directories are compared, looking for files that have been [added, modified, or removed](#change-types).

For this example, `rootfs-c9d-v1/` and `rootfs-c9d-v1.s1/` are recursively compared, each as relative root path.

The following changeset is found:

```
Added:      /etc/my-app.d/
Added:      /etc/my-app.d/default.cfg
Modified:   /bin/my-app-tools
Deleted:    /etc/my-app-config
```

This reflects the removal of `/etc/my-app-config` and creation of a file and directory at `/etc/my-app.d/default.cfg`.
`/bin/my-app-tools` has also been replaced with an updated version.

### Representing Changes

A [tar archive][tar-archive] is then created which contains *only* this changeset:

- Added and modified files and directories in their entirety
- Deleted files or directories marked with a [whiteout file](#whiteouts)

The resulting tar archive for `rootfs-c9d-v1.s1` has the following entries:

```
./etc/my-app.d/
./etc/my-app.d/default.cfg
./bin/my-app-tools
./etc/.wh.my-app-config
```

Where the basename name of `./etc/my-app-config` is now prefixed with `.wh.`, and will therefore be removed when the changeset is applied.

## Applying

Layer Changesets of [mediatype](./media-types.md) `application/vnd.oci.image.layer.tar+gzip` are applied rather than strictly extracted in normal fashion for tar archives.

Applying a layer changeset requires consideration for the [whiteout](#whiteouts) files.
In the absence of any [whiteout](#whiteouts) files in a layer changeset, the archive is extracted like a regular tar archive.


### Changeset over existing files

This section covers applying an entry in a layer changeset, if the file path already exists.

If the file path is a directory, then the existing path just has it's attribute set from the layer changeset for that filepath.
If the file path is any other file type (regular file, FIFO, etc), then the:
* file path is unlinked (See [`unlink(2)`](http://linux.die.net/man/2/unlink))
* create the file
    * If a regular file then content written.
* set attributes on the filepath

## Whiteouts

A whiteout file is an empty file with a special filename that signifies a path should be deleted.
A whiteout filename consists of the prefix .wh. plus the basename of the path to be deleted.
As files prefixed with `.wh.` are special whiteout markers, it is not possible to create a filesystem which has a file or directory with a name beginning with `.wh.`.

Once a whiteout is applied, the whiteout itself MUST also be hidden.
Whiteout files MUST only apply to resources in lower/parent layers.
Files that are present in the same layer as a whiteout file can only be hidden by whiteout files in subsequent layers.
The following is a base layer with several resources:

```
a/
a/b/
a/b/c/
a/b/c/bar
```

When the next layer is created, the original `a/b` directory is deleted and recreated with `a/b/c/foo`:

```
a/
a/.wh..wh..opq
a/b/
a/b/c/
a/b/c/foo
```

When processing the second layer, `a/.wh..wh..opq` is applied first, before creating the new version of `a/b`, regardless of the ordering in which the whiteout file was encountered.
For example, the following layer is equivalent to the layer above:

```
a/
a/b/
a/b/c/
a/b/c/foo
a/.wh..wh..opq
```

Implementations SHOULD generate layers such that the whiteout files appear before sibling directory entries.

### Opaque Whiteout

In addition to expressing that a single entry should be removed from a lower layer, layers may remove all of the children using an opaque whiteout entry.
An opaque whiteout entry is a file with the name `.wh..wh..opq` indicating that all siblings are hidden in the lower layer.
Let's take the following base layer as an example:

```
etc/
	my-app-config
bin/
	my-app-binary
	my-app-tools
	tools/
		my-app-tool-one
```

If all children of `bin/` are removed, the next layer would have the following:

```
bin/
	.wh..wh..opq
```

This is called _opaque whiteout_ format.
An _opaque whiteout_ file hides _all_ children of the `bin/` including sub-directories and all descendants.
Using _explicit whiteout_ files, this would be equivalent to the following:

```
bin/
	.wh.my-app-binary
	.wh.my-app-tools
	.wh.tools
```

In this case, a unique whiteout file is generated for each entry.
If there were more children of `bin/` in the base layer, there would be an entry for each.
Note that this opaque file will apply to _all_ children, including sub-directories, other resources and all descendants.

Implementations SHOULD generate layers using _explicit whiteout_ files, but MUST accept both.

Any given image is likely to be composed of several of these Image Filesystem Changeset tar archives.

[tar-archive]: https://en.wikipedia.org/wiki/Tar_(computing)
