# How to Load DRBD on Talos Linux

This guide shows you how to set the DRBD® Module Loader when using [Talos Linux].

To complete this guide, you should be familiar with:

* editing `LinstorSatelliteConfiguration` resources.
* using the `talosctl` command line to to access Talos Linux nodes.
* using the `kubectl` command line tool to access the Kubernetes cluster.

## Configure Talos system extension for DRBD

By default, the DRBD Module Loader will try to find the necessary header files to build DRBD from source on the host system. In [Talos Linux] these header files are not included in the host system. Instead, the Kernel modules is packed into a system extension.

Ensure Talos has the correct `drbd` [system extension](https://github.com/siderolabs/extensions) loaded for the running Kernel.

This can be achieved by updating the machine config:

```yaml
machine:
  install:
    extensions:
      - image: ghcr.io/siderolabs/drbd:9.2.0-v1.3.5
```
**NOTE**: Replace `v1.3.5` with the Talos version running.

Validate `drbd` module is loaded:
```shell
$ talosctl -n <NODE_IP> read /proc/modules
drbd 643072 - - Live 0xffffffffc0010000 (O)
```

## Configure the DRBD Module Loader

To change the DRBD Module Loader, so that it uses the modules provided by system extension, apply the following `LinstorSatelliteConfiguration`:

```yaml
apiVersion: piraeus.io/v1
kind: LinstorSatelliteConfiguration
metadata:
  name: talos-loader-override
spec:
  patches:
    - target:
        kind: Pod
        name: satellite
      patch: |
        apiVersion: v1
        kind: Pod
        metadata:
          name: satellite
        spec:
          initContainers:
            - name: drbd-shutdown-guard
              $patch: delete
            - name: drbd-module-loader
              $patch: delete
          volumes:
            - name: run-systemd-system
              $patch: delete
            - name: run-drbd-shutdown-guard
              $patch: delete
            - name: systemd-bus-socket
              $patch: delete
            - name: lib-modules
              $patch: delete
            - name: usr-src
              $patch: delete
            - name: etc-lvm-backup
              hostPath:
                path: /var/etc/lvm/backup
                type: DirectoryOrCreate
            - name: etc-lvm-archive
              hostPath:
                path: /var/etc/lvm/archive
                type: DirectoryOrCreate
```

Explanation:

- `/etc/lvm/*` is read-only in Talos and therefore can't be used. Let's use `/var/etc/lvm/*` instead.
- Talos does not ship with Systemd, so everything Systemd related needs to be removed
- `/usr/lib/modules` and `/usr/src` are not needed as the Kernel module is already compiled and needs just to be used.

[Talos Linux]: https://talos.dev