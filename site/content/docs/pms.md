---
title: "Package Manager System"
type: docs
---

# Package Manager System

The Package Manager System of the Macaroni OS is
[luet](https://github.com/geaaru/luet/). It's written
in Golang and it has zero dependencies and permits to
prepare environments "from scratch".

It was initially based on branch 0.16 of the Mocaccino OS's
PMS but later is totally rewritten.

It's composed of two different binary: the `luet` binary is
the PMS client used by users to install packages and the
`luet-build` binary is used to build packages and bump a
new repository revision.

It will be rebranding soon in `anise`.

# Luet

## 1.Repositories or Wagons

In computer science in general, the term `repository` describes the place
where is available a list of packages.

I saw often in my life that mapping computer science's terms with real entities
helps users to create relationships in their minds.
So, playing with the term, a synonym of `repository` is also `coal/salt's mine
or gold mine`. Following this concept in `Luet` a repository is a `wagon` where
the stones are the packages that a user could require that are got from the gold mine.

The `wagon identity` is the card that describes the specific wagon: the name,
the revision id, the date when the wagon is been prepared, and the URLs are
the mine tracks used to get the stones. In addition, for every wagon identity
there are different `wagon document` that contains:

1. the metafile `repository.meta.yaml.tar[.gz|.zstd]`: this file is deprecated and will be
   removed soon. It contains metadata of all stones available in the wagon.
   Will be soon removed because having a big file with all metadata together
   consumes a lot of memory resources in the sync phase. It's better to have
   a tarball with multiple files with the right directories tree.

2. the tree tarball `tree.tar[.gz|.zstd]`: this file contains the packages
   specs (*definition.yaml*) of the packages present on the source repository
   when the repository is been bumped.

3. the compiler tree tarball `compilertree.tar[.gz|.zstd]`: this file contains
   the build specs (*build.yaml*) of the packages present of the source repository
   when the repository is been bumped.

The wagon documents are validated with a checksum through the first repository
file `repository.yaml` that is been created with a limited number of pieces of
information to speed up the checks of the updates.

Hereinafter, an example of the files downloaded in the repo sync phase:

```bash
$> luet repo update --debug mottainai-stable
DEBUG (root.go:#51:github.com/geaaru/luet/cmd.LoadConfig) Using config file: /etc/luet/luet.yaml
DEBUG (loader.go:#38:github.com/geaaru/luet/pkg/repository.LoadRepositories) Parsing Repository Directory /etc/luet/repos.conf.d ...
DEBUG (wagon.go:#155:github.com/geaaru/luet/pkg/v2/repository.(*WagonRepository).Sync) Sync of the repository mottainai-stable in progress...
DEBUG (http.go:#243:github.com/geaaru/luet/pkg/v2/repository/client.(*HttpClient).DownloadFile) Downloading https://dl.macaronios.org/repos/mottainai/repository.yaml
DEBUG (http.go:#255:github.com/geaaru/luet/pkg/v2/repository/client.(*HttpClient).DownloadFile) Downloaded repository.yaml of 0.00 MB ( 0.00 MiB/s )
DEBUG (http.go:#243:github.com/geaaru/luet/pkg/v2/repository/client.(*HttpClient).DownloadFile) Downloading https://dl.macaronios.org/repos/mottainai/tree.tar.zst
DEBUG (http.go:#255:github.com/geaaru/luet/pkg/v2/repository/client.(*HttpClient).DownloadFile) Downloaded tree.tar.zst of 0.00 MB ( 0.00 MiB/s )
DEBUG (wagon.go:#207:github.com/geaaru/luet/pkg/v2/repository.(*WagonRepository).Sync) Tree tarball for the repository mottainai-stable downloaded correctly.
DEBUG (http.go:#243:github.com/geaaru/luet/pkg/v2/repository/client.(*HttpClient).DownloadFile) Downloading https://dl.macaronios.org/repos/mottainai/repository.meta.yaml.tar.zst
DEBUG (http.go:#255:github.com/geaaru/luet/pkg/v2/repository/client.(*HttpClient).DownloadFile) Downloaded repository.meta.yaml.tar.zst of 0.30 MB ( 0.17 MiB/s )
DEBUG (wagon.go:#215:github.com/geaaru/luet/pkg/v2/repository.(*WagonRepository).Sync) Metadata tarball for the repository mottainai-stable downloaded correctly.
DEBUG (wagon.go:#228:github.com/geaaru/luet/pkg/v2/repository.(*WagonRepository).Sync) Decompress tree of the repository mottainai-stable...
🏠 Repository:               mottainai-stable Revision:  88 - 2023-04-21 16:15:48 +0000 UTC
DEBUG (wagon.go:#289:github.com/geaaru/luet/pkg/v2/repository.(*WagonRepository).ExplodeMetadata) 
🏠 Repository:               mottainai-stable unpacking metadata. 
DEBUG (wagon_stones.go:#1100:github.com/geaaru/luet/pkg/v2/repository.(*WagonStones).LoadCatalog) [mottainai-stable] Found metafile /var/cache/luet/repos/mottainai-stable/metafs/repository.meta.yaml
DEBUG (wagon_stones.go:#1122:github.com/geaaru/luet/pkg/v2/repository.(*WagonStones).LoadCatalog) [mottainai-stable] metadata loaded in 199480 µs.
```

In particolar, the sync process download first the `repository.yaml` file:

```bash
$> curl https://dl.macaronios.org/repos/mottainai/repository.yaml
name: mottainai-stable
description: MottainaiCI Official Repository
urls:
- http://localhost:8000
type: http
enable: true
revision: 88
last_update: "1682093748"
index: []
repo_files:
  compilertree:
    filename: compilertree.tar.gz
    compressiontype: gzip
    checksums:
      sha256: 3e9827097fd8e046c0d74f44c25e8c474204e3bda2a7ca91f20c81965c2001bb
  meta:
    filename: repository.meta.yaml.tar.zst
    compressiontype: zstd
    checksums:
      sha256: c0305b0e757827f8538da003cbe6cf03d88fd8636313739892a2dc26cfaa46e3
  tree:
    filename: tree.tar.zst
    compressiontype: zstd
    checksums:
      sha256: 1fe37e7bf858bd8dd3f9c3d1c968081218d6dee8a12c0e3c26cd2c9109159b8e
```

and compare the `revision` and `last_update` fields to understand
if it's needed go ahead with the download of the wagon document's files.

The `urls` attribute of the `repository.yaml` is not used at the moment and
it sets with value used locally. It possible that will be removed in the near future
or used in validation phase.

The stones could be uncompressed, so shared as simple tar files, or compressed
in different formats: gzip or zstd.

The wagons used are based on YAML files that are installed under the directory
`/etc/luet/repos.conf.d` by default. Additional directories where read the
repositories YAML files could be added through the `luet.yaml` configuration
file. In particular, the directories are defined through the attribute
`repos_confdir`:

```yaml
# Define the list of directories where luet
# try for files with .yml extension that define
# luet repositories.
repos_confdir:
- /etc/luet/repos.conf.d
- /my-repos/
```

The location where the repositories metadata are unpacked depends on the
configuration parameter `system.database_path` that has the default value
equal to */var/cache/luet*.

In particular, under the directory */var/cache/luet* is present the `luet.db` file
that contains the local BoltDB database where are registered the installed packages,
the files of the installed packages, and the `finalize` of the packages when present.

At the moment the only supported database is BoltDB, but will be added the support
of new databases soon.

Under the directory */var/cache/luet* there are two directories:

* `packages`: under this directory have stored the tarballs downloaded before
  executing the installation.

* `repos`: under this directory are created the directories for every repository
  synced locally.

So, for every repository under the local repository are unpacked the metadata
tarballs, in particular:

* `metafs`: under the directory *metafs* is present the file *repository.meta.yaml*
  that contains the metadata of all packages. This directory will be removed in
  the next release of luet.

* `repository.yaml`: the *repository.yaml* contains the local data with the
  wagon identity data.

* `treefs`: under the directory *treefs* is unpacked the tree of the repository
  with the following pattern: `<package-category>/<package-name>/<package-version>/`.
  Under the package directory are present:

   - `definition.yaml`: the file from the repository sources with the main
     package metadata

   - `metadata.yaml`: this file is generated from client-side when
     a repository is synced. Will be generated by `luet-build` soon or
     just replaced only by the *metadata.json* file.

   - `metadata.json`: this file is generated from the client-side when a
     repository is synced. Will be generated by `luet-build` soon.
     It's used the JSON file instead of YAML because to elaborate
     the JSON file is faster than YAML.

* `provides.yaml`: to speed up the solver logic under the directory `treefs` is
  generated the file *provides.yaml* the contains a map of the packages with
  *provides*. Will be generated by `luet-build` soon.

```bash
$> ls  /var/cache/luet/repos/macaroni-eagle/treefs/
app-accessibility-2  dev-db           dev-libs       media-libs      provides.yaml    sys-libs-4.8
app-admin            dev-db-10.5      dev-libs-1     media-libs-1.0  sci-libs         sys-power
app-arch             dev-db-11        dev-libs-2     media-libs-2    seed             sys-process
app-crypt            dev-db-13        dev-libs-3     net-analyzer    sys-apps         system
app-crypt-1          dev-db-3         dev-lisp       net-dialup      sys-auth         toolchain
app-doc              dev-db-8.0       dev-lisp-2     net-dns         sys-block        virtual
app-editors          dev-go           dev-perl       net-firewall    sys-cluster      virtual-1
app-emulation        dev-java-11      dev-php        net-ftp         sys-devel        virtual-11
app-eselect          dev-java-17      dev-python     net-libs        sys-devel-1.16   virtual-17
app-metrics          dev-java-18      dev-python-3   net-libs-1.1    sys-devel-11     virtual-18
app-misc             dev-java-2       dev-scheme-12  net-mail        sys-devel-2      virtual-entities
app-portage          dev-java-3.6     dev-tex        net-misc        sys-devel-2.36   www-apps
app-shells           dev-java-8       dev-texlive    net-nds         sys-devel-2.69   www-client
app-text             dev-lang         dev-util       net-p2p         sys-devel-9.2.0  www-servers
app-text-3.0         dev-lang-2       dev-util-3     net-print       sys-fs           www-servers-2
app-text-4.1.2       dev-lang-2.7     dev-vcs        net-proxy       sys-fs-3         x11-apps
app-text-4.2         dev-lang-3.7     gnome-extra    net-vpn         sys-kernel       x11-base
app-text-4.3         dev-lang-7.4     mail-mta       net-wireless    sys-libs         x11-libs
app-text-4.5         dev-lang-8.1     media-fonts    perl-core       sys-libs-18.1    x11-misc
dev-cpp              dev-lang-stable  media-gfx      pkglist         sys-libs-2.2     x11-proto
```

Hereinafter, an example of a package directory:
```
$> ls  /var/cache/luet/repos/macaroni-eagle/treefs/sys-devel-9.2.0/gcc/9.2.0+1/
definition.yaml  metadata.json  metadata.yaml
```


### 1.1 Show repositories

This command permits to see all installed repositories. In particular,
the enabled repositories are colored on green and instead the
disabled repositories are in red.

```bash
$> luet repo list --help
List of the configured repositories.

Usage:
  luet repo list [OPTIONS] [flags]

Flags:
      --disabled      Show only disabled repositories.
      --enabled       Show only enabled repositories.
  -h, --help          help for list
  -q, --quiet         Show only name of the repositories.
  -t, --type string   Filter repositories of a specific type
  -u, --urls          Show URLs of the repository. (only in normal mode).
```

The availables options are:

* `--urls`: Show the repositories URLs

* `--enabled`: Show only the enabled repositories

* `--disabled`: Show only the disabled repositories

* `--type <type>`: Filter for repositories of type specified.
  The types are: 'http', 'docker', 'disk'.

* `--quiet`: Show only name of the repositories.

### 1.2 Enable one or more repositories

This command permits to enable repositories.

```bash
$> luet repo enable --help
Enable one or more repositories.

Usage:
  luet repo enable <repo1> ... <repoN> [flags]

Flags:
  -h, --help   help for enable
```

### 1.3 Disable one or more repositories

This command permits to disable repositories.

```bash
$> luet repo disable --help
Disable one or more repositories.

Usage:
  luet repo disable <repo1> ... <repoN> [flags]

Flags:
  -h, --help   help for disable
```

### 1.4 Update / Sync one or more repositories

This command permits to sync repositories metadata locally. When the repository
is not defined it tries to sync all enabled repositories.

```bash
# luet repo update --help
Update a specific cached repository or all cached repositories.

Usage:
  luet repo update [repo1] [repo2] [OPTIONS] [flags]

Aliases:
  update, up

Examples:

# Update all cached repositories:
$> luet repo update

# Update only repo1 and repo2
$> luet repo update repo1 repo2


Flags:
  -f, --force           Force resync.
  -h, --help            help for update
  -i, --ignore-errors   Ignore errors on sync repositories.
```

* `--force|-f` option permits to force updates of the local trees also when
  the revision is the same. This is needed for example when a new luet release
  introduces new local logics.

* `--ignore-errors|-i` option permits to ignore errors on sync. In this
  case, luet exiting always with zero.
