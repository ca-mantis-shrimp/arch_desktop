# The Immutable Arch Distribution Configuration
This is the site of an experiment, what i want is to go through the process of BUILDING an arch-based format that is going to be able to incrementally update by using systemd-repart to manage partitions and we can then load new versions onto the off-partition and boot into it using systemd-boot with the ability to roll back whenever

The other way we can think of this is as a systemd-based operating system that uses arch linux as its base and we can then layer on top of it whatever we want, this means that we can have a base arch system that is immutable and then we can have layers on top of it that can be updated or changed without affecting the base system.

## Goals

The goal is to make a system that is:

- Stable: this is coming off the experience of not being able to boot after an update. Immutable systems are perfect for this as we can have the speed of arch updates with the stability of being able to roll back to a previous version
- Simple: One big reason we use systemd is to avoid complexity and utilize what i feel has been powerful innovations in the systemd ecosystem
- Structured: Using declarative tools helps to do these first two values, we can reduce our reliance on bundles of scripts, and can even have customizations be done using configuration rather than imperative checks
  Now... as i said, this is an experiment, so while another goal is that we would LIKE to have this be full immutable (read-only `/usr` and all), we need to be pragmatic and its more important that things works well rather than needing to have everything be perfectly immutable.

### Architectures

For our building, we are goiing to be using [mkosi](https://github.com/systemd/mkosi) to build our images, but this can be seen as top layer that will be utilizing several other tools to make it all work:

- [systemd-sysext](https://man.archlinux.org/man/systemd-sysext.8.en) for layering changes on to the additional pieces to make them easy
- [systemd-repart](https://www.freedesktop.org/software/systemd/man/latest/systemd-repart.html)
- [systemd-boot](https://www.freedesktop.org/software/systemd/man/latest/systemd-boot.html) for booting the backups properly
- [systemd-networkd](https://wiki.archlinux.org/title/Systemd-networkd)
  - with [systemd-resolved](https://wiki.archlinux.org/title/Systemd-resolved) for networking
- [portable services](https://systemd.io/PORTABLE_SERVICES/) for managing services where possible
- [ostree](https://ostree.readthedocs.io/en/latest/) may be a good option but thats optional....
- [pacman](https://archlinux.org/pacman/) for managing base packages
- [paru](https://github.com/Morganamilo/paru) for managing AUR packages

#### User-space tooling

after that, we will try to be very disciplined about installing stuff using sandboxed apps like:

- [flatpak](https://flatpak.org/) for desktop applications
- [podman](https://podman.io/) for stuff where we need more control but cant use portable services
- [uv](https://docs.astral.sh/uv/) for managing user-space applications in python
- [cargo](https://github.com/rust-lang/cargo) for rust applications
- [npm](https://www.npmjs.com/) for node applications

## Assumptions

This is a more low-level project since we need to be thoughtful about how we are going to handle our various structures, so this is built for MY context and tries to work in my environment,

this is why we arent running something like [particleos](https://github.com/systemd/particleos) which is trying to be allot more general purpose, but that introduces complexity. if youre trying to look for the ultimate generic system, go look at that.

For this, if this works for you, use it! otherwise, feel free to fork and modify it to fit your needs and your context

### Devices

We have a few things that we are working to get this operating system on:

- A linux desktop
  - Intel CPU
  - NVIDIA GPU
  - integrated network drive
- a linux homelab
  - AMD APU
- Cloud VMs
  - From the major cloud providers and hoping to make it work in something up there when necessary

I already carry my own [homed](https://systemd.io/HOME_DIRECTORY/) partition on a dedicated NVME drive that i carry with me. Therefore, they will all assume that we have a separate home partition with a separate home directory

also, the config of the home directory is managed using [chezmoi](https://www.chezmoi.io/) so that we can have a consistent home directory across all devices but it will primarily live on the external home partition this is just so things know

## Quick Start

First, build a `mkosi.rootpw` file with the root password you want to use for the system.

Then, run the following commands to build and run the image:

```bash
mkosi build

```

Then, to run the image in QEMU, use:

```bash
mkosi qemu
```

or to run it in a shell:

```bash
mkosi shell
```

Login credentials: `root` / `root`


