# GrootFS (BOSH release) [![slack.cloudfoundry.org](https://slack.cloudfoundry.org/badge.svg)](https://slack.cloudfoundry.org)

A [BOSH](http://docs.cloudfoundry.org/bosh/) release for deploying
[GrootFS](https://github.com/cloudfoundry/grootfs).

This release makes a lot of [assumptions](#assumptions) based on expected use
with [garden-runc-release](https://github.com/cloudfoundry/garden-runc-release)
in the context of Cloud Foundry / Diego. It may be useful in combination with
Garden-runC outside of Cloud Foundry, but it's unlikely to be useful in any
other context.

## Deploying with Diego / Cloud Foundry

There are two possible approaches to add GrootFS to an existing CF/Diego deployment. Choose
the one which suits your needs best:

### Using the new BOSH Operations File feature

Our supported way to add GrootFS to a CloudFoundry / Diego deployment is by applying a GrootFS
[BOSH Operations file](https://github.com/cppforlife/go-patch/blob/master/docs/examples.md) to
your existing Diego deployment manifest. In
order to use operations files, you will need to be using the new [Golang-based BOSH CLI](https://github.com/cloudfoundry/bosh-cli).

The GrootFS operations file is located here: `grootfs-release/manifests/operations/grootfs.yml`.
It contains a description of the manifest changes BOSH needs to make to your existing deployment
in order to add GrootFS. You may need to edit the file to ensure that the referenced job names
match those in your existing CF / Diego deployment. Once you've checked this, run:

```
bosh deploy \
  --ops-file grootfs-release/manifests/operations/grootfs.yml \
  your-existing-cf-and-diego-manifest.yml
```

### Modifying manifests manually

If you don't want to use the operations file approach, you can alternatively make the equivalent
changes to you deployment manifests manually.

Given an existing Diego / CF deployment, take the following steps to add
grootfs-release to the cells in your Diego deployment:

* Select a version of grootfs-release - you can find the compatible version of
   grootfs-release for your version of garden-runc-release in the [Garden-runC
   release notes](https://github.com/cloudfoundry/garden-runc-release/releases)
* Add grootfs-release to the `releases` section of your deployment manifest
   (and upload grootfs-release to your BOSH director)
* Add the `grootfs` job to the `templates` section of your cell job
* Add the following properties to your cell job
```
garden:
  image_plugin: "/var/vcap/packages/grootfs/bin/grootfs"
  image_plugin_extra_args: ["--config=/var/vcap/jobs/grootfs/config/grootfs_config.yml"]

  # if you have capi.nsync.diego_privileged_containers and capi.stager.diego_privileged_containers set to true
  privileged_image_plugin: "/var/vcap/packages/grootfs/bin/grootfs"
  privileged_image_plugin_extra_args: ["--config=/var/vcap/jobs/grootfs/config/privileged_grootfs_config.yml"]
```
* Set the `diego.rep.preloaded_rootfses` property on your Diego cells to
  `[cflinuxfs2:/var/vcap/packages/cflinuxfs2/rootfs.tar]`. Make sure you're
  using version 1.45.0 or later of
  [cflinuxfs2-rootfs-release](https://github.com/cloudfoundry/cflinuxfs2-rootfs-release),
  otherwise the rootfs tarball will not be present on disk on the cells. If you
  are using a custom rootfs, you'll need to start providing that as a tarball,
  and set this property accordingly.

If you have set any of the following `garden` properties, you should set them on
`grootfs` to get the same behaviour:
- `graph_cleanup_threshold_in_mb`
- `insecure_docker_registry_list`
- `persistent_image_list` _(change directory paths to tarballs - e.g._
  `/var/vcap/packages/cflinuxfs2/rootfs.tar`_)_

There is no equivalent to the `garden.docker_registry_endpoint` property. If you
need this, GrootFS may not be suitable for you yet. Please open an issue and let
us know your use case though!

You may also want to set the following optional properties on `grootfs` for
various reasons. See the [grootfs job spec](jobs/grootfs/spec) for more info on
what these properties do:
- `dropsonde_port`
- `log_level`
- `store_size_bytes`
- `driver`
- `skip_mount`

There should be no need to recreate cells when transitioning to GrootFS, though
you may wish to do so anyway in order to clear out any cruft left behind by
Garden-runC's previous image management implementation.

### Drivers

It's possible to chose a filesystem driver to use by providing the `grootfs.driver`
property in the manifest. The default driver is `btrfs`, and the other possible
value for this property is `overlay-xfs`.

At this moment it's not supported to change the `driver` of a running instance. The
vm must be recreated.

## Assumptions

### newuidmap / newgidmap

We ship custom implementations of [`newuidmap` and
`newgidmap`](https://github.com/cloudfoundry/idmapper) that ignore `/etc/subuid`
and `/etc/subgid`. These custom binaries have a specific uid/gid mapping hard
coded. Currently, Garden-runC only ever uses one this one specific mapping.
Hard coding the mapping reduces the possible attack surface for exploits, which
we care about since these binaries execute as `root`. The usual approach of
describing allowed mappings in `/etc/sub{u,g}id` doesn't play very nicely with
BOSH.

### volume creation

Depending on the `driver` property, the release will create a btrfs or xfs volumes.
The release will never recreate this volume on an update if the file
already exists, even if you change it's size in the manifest. Current flow:

* If there's no volume file: create volume file -> format with btrfs/xfs -> mount
* If there's a volume file: check if it's formatted with btrfs/xfs
  * if yes -> mount
  * if no -> format with btrfs/xfs -> mount

When using the btrfs driver, only one btrfs volume will be created with `store_size` bytes,
and internally it will be split into two subfolders (privileged and unprivileged)
for the privileged and unprivileged stores.

When using the xfs driver two volumes will be created, one for privileged and one
for unprivileged containers. Each one of them with `store_size` bytes.

Even though garden will be calling grootfs as root, the unprivileged store
will be owned by the user ~4294967294 (or the max uid possible), in order to
allow the running user of the container to have access to it's filesystem.

## Contributing

In order to help us extend GrootFS, we recommend opening a Github issue to
describe the proposed features or changes. We also welcome pull requests.

## Shipped packages

* btrfs-progs 4.4.1
* e2fsprogs 1.43.3
* autoconf 2.69
* automake 1.15
* gettext 0.19.8.1
* libtool 2.4.6
* pkg-config 0.29
* lzo 2.09
* util-linux 2.28
* zlib 1.2.8
* go 1.7.5
* xfsprogs 4.3.0

## License

Apache License 2.0
