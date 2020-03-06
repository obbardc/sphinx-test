# Debos Actions

## Apt Action

Install packages and their dependencies to the target rootfs with 'apt'.

Yaml syntax:

    - action: apt
      recommends: bool
      unauthenticated: bool
      packages:
        - package1
        - package2

Mandatory properties:

- packages -- list of packages to install

Optional properties:

- recommends -- boolean indicating if suggested packages will be installed

- unauthenticated -- boolean indicating if unauthenticated packages can be
installed


## Debootstrap Action

Construct the target rootfs with debootstrap tool.

Please keep in mind -- file `/etc/resolv.conf` will be removed after
execution. Most of the OS scripts used by `debootstrap` copy `resolv.conf`
from the host, and this may lead to incorrect configuration when becoming
part of the created rootfs.

Yaml syntax:

    - action: debootstrap
      mirror: URL
      suite: "name"
      components: <list of components>
      variant: "name"
      keyring-package:
      keyring-file:

Mandatory properties:

- suite -- release code name or symbolic name (e.g. "stable")

Optional properties:

- check-gpg -- verify GPG signatures on Release files, true by default

- mirror -- URL with Debian-compatible repository

    If no mirror is specified debos will use http://deb.debian.org/debian as default.

- variant -- name of the bootstrap script variant to use

- components -- list of components to use for packages selection.

    If no components are specified debos will use main as default.

Example:

    components: [ main, contrib ]

- keyring-package -- keyring for package validation.

- keyring-file -- keyring file for repository validation.

- merged-usr -- use merged '/usr' filesystem, true by default.


## Download Action

Download a single file from Internet and unpack it in place if needed.

Yaml syntax:

    - action: download
      url: http://example.domain/path/filename.ext
      name: firmware
      filename: output_name
      unpack: bool
      compression: gz

Mandatory properties:

- url -- URL to an object for download

- name -- string which allow to use downloaded object in other actions via
'origin' property. If 'unpack' property is set to 'true' name will refer to
temporary directory with extracted content.

Optional properties:

- filename -- use this property as the name for saved file. Useful if URL
does not contain file name in path, for example it is possible to download
files from URLs without path part.

- unpack -- hint for action to extract all files from downloaded archive.
See the 'Unpack' action for more information.

- compression -- optional hint for unpack allowing to use proper compression
method. See the 'Unpack' action for more information.


## FilesystemDeploy Action

Deploy prepared root filesystem to output image. This action requires
'image-partition' action to be executed before it.

Yaml syntax:

    - action: filesystem-deploy
      setup-fstab: bool
      setup-kernel-cmdline: bool
      append-kernel-cmdline: arguments

Optional properties:

- setup-fstab -- generate '/etc/fstab' file according to information
provided by 'image-partition' action. By default is 'true'.

- setup-kernel-cmdline -- add location of root partition to
'/etc/kernel/cmdline' file on target image. By default is 'true'.

- append-kernel-cmdline -- additional kernel command line arguments passed
to kernel.


## ImagePartition Action

This action creates an image file, partitions it and formats the
filesystems.

Yaml syntax:

    - action: image-partition
      imagename: image_name
      imagesize: size
      partitiontype: gpt
      gpt_gap: offset
      partitions:
        <list of partitions>
      mountpoints:
        <list of mount points>

Mandatory properties:

- imagename -- the name of the image file.

- imagesize -- generated image size in human-readable form, examples: 100MB,
1GB, etc.

- partitiontype -- partition table type. Currently only 'gpt' and 'msdos'
partition tables are supported.

- gpt_gap -- shifting GPT allow to use this gap for bootloaders, for example
if U-Boot intersects with original GPT placement. Only works if parted
supports an extra argument to mklabel to specify the gpt offset.

- partitions -- list of partitions, at least one partition is needed.
Partition properties are described below.

- mountpoints -- list of mount points for partitions. Properties for mount
points are described below.

Yaml syntax for partitions:

       partitions:
         - name: label
    	   name: partition name
    	   fs: filesystem
    	   start: offset
    	   end: offset
    	   features: list of filesystem features
    	   flags: list of flags
    	   fsck: bool

Mandatory properties:

- name -- is used for referencing named partition for mount points
configuration (below) and label the filesystem located on this partition.

- fs -- filesystem type used for formatting.

'none' fs type should be used for partition without filesystem.

- start -- offset from beginning of the disk there the partition starts.

- end -- offset from beginning of the disk there the partition ends.

For 'start' and 'end' properties offset can be written in human readable
form -- '32MB', '1GB' or as disk percentage -- '100%'.

Optional properties:

- features -- list of additional filesystem features which need to be
enabled for partition.

- flags -- list of additional flags for partition compatible with parted(8)
'set' command.

- fsck -- if set to `false` -- then set fs_passno (man fstab) to 0 meaning
no filesystem checks in boot time. By default is set to `true` allowing
checks on boot.

Yaml syntax for mount points:

       mountpoints:
         - mountpoint: path
    	   partition: partition label
    	   options: list of options
    	   buildtime: bool

Mandatory properties:

- partition -- partition name for mounting.

- mountpoint -- path in the target root filesystem where the named partition
should be mounted.

Optional properties:

- options -- list of options to be added to appropriate entry in fstab file.

- buildtime -- if set to true then the mountpoint only used during the debos
run. No entry in `/etc/fstab' will be created. The mountpoints directory
will be removed from the image, so it is recommended to define a
`mountpoint` path which is temporary and unique for the image, for example:
`/mnt/temporary_mount`. Defaults to false.

Layout example for Raspberry PI 3:

    - action: image-partition
      imagename: "debian-rpi3.img"
      imagesize: 1GB
      partitiontype: msdos
      mountpoints:
        - mountpoint: /
          partition: root
        - mountpoint: /boot/firmware
          partition: firmware
          options: [ x-systemd.automount ]
      partitions:
        - name: firmware
          fs: vfat
          start: 0%
          end: 64MB
        - name: root
          fs: ext4
          start: 64MB
          end: 100%
          flags: [ boot ]


## OstreeCommit Action

Create OSTree commit from rootfs.

Yaml syntax:

    - action: ostree-commit
      repository: repository name
      branch: branch name
      subject: commit message
      collection-id: org.apertis.example
      ref-binding:
        - branch1
        - branch2
      metadata:
        key: value
        vendor.key: somevalue

Mandatory properties:

- repository -- path to repository with OSTree structure; the same path is
used by 'ostree' tool with '--repo' argument. This path is relative to
'artifact' directory. Please keep in mind -- you will need a root privileges
for 'bare' repository type
(https://ostree.readthedocs.io/en/latest/manual/repo/#repository-types-and-locations).

- branch -- OSTree branch name that should be used for the commit.

Optional properties:

- subject -- one line message with commit description.

- collection-id -- Collection ID ref binding (requires libostree 2018.6).

- ref-binding -- enforce that the commit was retrieved from one of the
branch names in this array.

    If 'collection-id' is set and 'ref-binding' is empty, will default to the branch name.

- metadata -- key-value pairs of meta information to be added into commit.


## OstreeDeploy Action

Deploy the OSTree branch to the image. If any preparation has been done for
rootfs, it can be overwritten during this step.

Action 'image-partition' must be called prior to OSTree deploy.

Yaml syntax:

    - action: ostree-deploy
      repository: repository name
      remote_repository: URL
      branch: branch name
      os: os name
      tls-client-cert-path: path to client certificate
      tls-client-key-path: path to client certificate key
      setup-fstab: bool
      setup-kernel-cmdline: bool
      appendkernelcmdline: arguments
      collection-id: org.apertis.example

Mandatory properties:

- remote_repository -- URL to remote OSTree repository for pulling stateroot
branch. Currently not implemented, please prepare local repository instead.

- repository -- path to repository with OSTree structure. This path is
relative to 'artifact' directory.

- os -- os deployment name, as explained in:
https://ostree.readthedocs.io/en/latest/manual/deployment/

- branch -- branch of the repository to use for populating the image.

Optional properties:

- setup-fstab -- create '/etc/fstab' file for image

- setup-kernel-cmdline -- add the information from the 'image-partition'
action to the configured commandline.

- append-kernel-cmdline -- additional kernel command line arguments passed
to kernel.

- tls-client-cert-path -- path to client certificate to use for the remote
repository

- tls-client-key-path -- path to client certificate key to use for the
remote repository

- collection-id -- Collection ID ref binding (require libostree 2018.6).


## Overlay Action

Recursive copy of directory or file to target filesystem.

Yaml syntax:

    - action: overlay
      origin: name
      source: directory
      destination: directory

Mandatory properties:

- source -- relative path to the directory or file located in path
referenced by `origin`. In case if this property is absent then pure path
referenced by 'origin' will be used.

Optional properties:

- origin -- reference to named file or directory.

- destination -- absolute path in the target rootfs where 'source' will be
copied. All existing files will be overwritten. If destination isn't set '/'
of the rootfs will be used.


## Pack Action

Create tarball with filesystem.

Yaml syntax:

    - action: pack
      file: filename.ext
      compression: gz

Mandatory properties:

- file -- name of the output tarball, relative to the artifact directory.

- compression -- compression type to use. Only 'gz' is supported at the
moment.


## Raw Action

Directly write a file to the output image at a given offset. This is
typically useful for bootloaders.

Yaml syntax:

    - action: raw
      origin: name
      source: filename
      offset: bytes

Mandatory properties:

- origin -- reference to named file or directory.

- source -- the name of file located in 'origin' to be written into the
output image.

Optional properties:

- offset -- offset in bytes for output image file. It is possible to use
internal templating mechanism of debos to calculate offset with sectors (512
bytes) instead of bytes, for instance: '{{ sector 256 }}'. The default value
is zero.

- partition -- named partition to write to


## Other

Comments are allowed and should be prefixed with '#' symbol.

    # Declare variable 'Var'
    {{- $Var := "Value" -}}

    # Header
    architecture: arm64

    # Actions are executed in listed order
    actions:
      - action: ActionName1
        property1: true

      - action: ActionName2
        # Use value of variable 'Var' defined above
        property2: {{$Var}}

Mandatory properties for receipt:

- architecture -- target architecture

- actions -- at least one action should be listed


Supported actions

- apt -- https://godoc.org/github.com/go-debos/debos/actions#hdr-Apt_Action

- debootstrap --
https://godoc.org/github.com/go-debos/debos/actions#hdr-Debootstrap_Action

- download --
https://godoc.org/github.com/go-debos/debos/actions#hdr-Download_Action

- filesystem-deploy --
https://godoc.org/github.com/go-debos/debos/actions#hdr-FilesystemDeploy_Action

- image-partition --
https://godoc.org/github.com/go-debos/debos/actions#hdr-ImagePartition_Action

- ostree-commit --
https://godoc.org/github.com/go-debos/debos/actions#hdr-OstreeCommit_Action

- ostree-deploy --
https://godoc.org/github.com/go-debos/debos/actions#hdr-OstreeDeploy_Action

- overlay --
https://godoc.org/github.com/go-debos/debos/actions#hdr-Overlay_Action

- pack --
https://godoc.org/github.com/go-debos/debos/actions#hdr-Pack_Action

- raw -- https://godoc.org/github.com/go-debos/debos/actions#hdr-Raw_Action

- recipe --
https://godoc.org/github.com/go-debos/debos/actions#hdr-Recipe_Action

- run -- https://godoc.org/github.com/go-debos/debos/actions#hdr-Run_Action

- unpack --
https://godoc.org/github.com/go-debos/debos/actions#hdr-Unpack_Action


## Recipe Action

This action includes the recipe at the given path, and can optionally
override or set template variables.

To ensure compatibility, both the parent recipe and all included recipes
have to be for the same architecture. For convenience the parent
architecture is passed in the "architecture" template variable.

Limitations of combined recipes are equivalent to limitations within a
single recipe (e.g. there can only be one image partition action).

Yaml syntax:

    - action: recipe
      recipe: path to recipe
      variables:
        key: value

Mandatory properties:

- recipe -- includes the recipe actions at the given path.

Optional properties:

- variables -- overrides or adds new template variables.


## Run Action

Allows to run any available command or script in the filesystem or in host
environment.

Yaml syntax:

    - action: run
      chroot: bool
      postprocess: bool
      script: script name
      command: command line
      label: string

Properties 'command' and 'script' are mutually exclusive.

- command -- command with arguments; the command expected to be accessible
in host's or chrooted environment -- depending on 'chroot' property.

- script -- script with arguments; script must be located in recipe
directory.

Optional properties:

- chroot -- run script or command in target filesystem if set to true.
Otherwise the command or script is executed within the build process, with
access to the filesystem ($ROOTDIR), the image if any ($IMAGE), the recipe
directory ($RECIPEDIR) and the artifact directory ($ARTIFACTDIR). In both
cases it is run with root privileges.

- label -- if non-empty, this string is used to label output. If empty, a
label is derived from the command or script.

- postprocess -- if set script or command is executed after all other
commands and has access to the image file.

Properties 'chroot' and 'postprocess' are mutually exclusive.


## Unpack Action

Unpack files from archive to the filesystem. Useful for creating target
rootfs from saved tarball with prepared file structure.

Only (compressed) tar archives are supported currently.

Yaml syntax:

    - action: unpack
      origin: name
      file: file.ext
      compression: gz

Mandatory properties:

- file -- archive's file name. It is possible to skip this property if
'origin' referenced to downloaded file.

One of the mandatory properties may be omitted with limitations mentioned
above. It is expected to find archive with name pointed in `file` property
inside of `origin` in case if both properties are used.

Optional properties:

- origin -- reference to a named file or directory. The default value is
'artifacts' directory in case if this property is omitted.

- compression -- optional hint for unpack allowing to use proper compression
method.

Currently only 'gz', bzip2' and 'xz' compression types are supported. If not
provided an attempt to autodetect the compression type will be done.
