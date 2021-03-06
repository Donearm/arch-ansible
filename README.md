# Arch-Ansible: an Ansible playbook to install Arch Linux

![Screenshot of a system installed with arch-ansible][screenshot]

Arch-Ansible is a playbook designed to install Arch Linux on a target
machine. It was conceived to ease the preparation of virtual machines,
but it could be used to install on bare metal, with some tweaks.

## Ansible version

The playbook has been tested using Ansible 2.7 and higher.

## Installed system

Unless some steps are skipped or customized, the installed system will
run AwesomeWM. No greeter is installed by default: each user's
`.xinitrc` is configured to launch AwesomeWM when calling `startx`.

The `physlock` package is installed by default to provide a lightweight
lock screen for privacy and added security.

A bunch of default utilities like a PDF reader or gvim a preinstalled.
These are handled by the `utils` and `xutils` roles.

Users (including `root`) get their passwords from
`roles/users/defaults/main.yaml`. The file is stored in cleartext within
this repository to show its structure and allow modification. In real
scenarios, people should customize it and then encrypt it with
ansible-vault before committing to a VCS.

Currently only GPT installations are supported, using `refind` as the
bootloader. Swap space is not configured.

There is support for installing hypervisor guest additions as part of
the process.  This can be disabled by skipping the `virtguest` tag.  As
of today, only VirtualBox is supported.

The playbook relies on the [Yay](https://github.com/Jguer/yay) AUR
helper to install packages. Using `yay` instead of the stock `pacman`
module allows for uniform installation of binary and AUR packages. You
are free to uninstall `yay` after the provisioning is complete and to
install your favorite AUR helper.

## Playbook structure

The playbook is broken into two big parts, identified by tags:

* `bootstrap` is meant to run against a system running an Arch Linux
  installation media. It takes care of partitioning the disk, installing
  a bootloader and a set of base packages. Then reboot.
* `mainconfig` runs against the installed base system, adding additional
  packages, installing a DE, creating users and configuring locales. At
  the end, the system is read for use.

When used together, they build a complete system from an installation
media. `mainconfig` can also be run independently of `bootstrap`,
provided that the initial system state allows for Ansible incoming
connections.

This scenario could be used to provision an existing Vagrant box, for
which `bootstrap` would be of no use, since it is already partitioned
and base packages are installed. Some minor adjustments might be
required in this case (i.e. Vagrant boxes likely come with pre-installed
VirtualBox guest utilities without X support, which will cause
`virtualbox-guest-utils` not to install).

### bootstrap

The `bootstrap` phase can be tweaked by skipping tags:

* `partitioning` can be used to disable partitioning. It can be useful
  if a user wants to prepare a more complex layout by hand before
  launching the playbook. Once partitions have been mounted under a
  certain folder (typically `/mnt`) it makes no difference who mounted
  them;
* `bootloader` can be used to disable bootloader installation. Again,
  this may be useful if the user wants to customize the bootloader setup
  or use something different than Syslinux.

Installation of base packages cannot be skipped, but it can be
customized by editing `roles/base_packages/defaults/main.yaml`. It
already contains a very minimal set of packages and there is no
advantage is adding additional tools here.

By default, this phase is disabled. To run it, add the `bootstrap` tag
to the call.

### mainconfig

This tag marks the tasks that does the heavy lifting. It configures
locales, creates users and sets their initial passwords, and prepare the
system to work behind a proxy. These steps are compulsory.

Additional steps include installing utilities and GUI apps, a desktop
environment and applying default customizations to users. These steps
can be skipped or selected one by one using tags:

* `virtguest` install hypervisor guest additions;
* `awesome` installs the AwesomeWM for all non-root users;
* `yay` copies `yay` settings to each user home folder;
* `ttf_fonts` installs additional fonts;
* `utils` installs some non-X utilities, listed in
  `roles/utils/defaults/main.yaml`;
* `kitty` installs the default terminal emulator, kitty-terminal;
* `tmux` installs the terminal multiplexer tmux;
* `imv` installs the default image viewer, the simple, X11/Wayalnd
  supporting imv;
* `mpd` install the MPD music player daemon, with `ncmpcpp` as its
  client;
* `rtorrent` installs rTorrent, a ncurses bittorrent client;
* `xutils` installs some X utilities, listed in
  `roles/xutils/defaults/main.yaml`.

Roles which install X apps will automatically pull X.org as a
dependency.

### Reboot

After both `bootstrap` and `mainconfig` the system will be rebooted to
ensure a clean start. This can be disabled by skipping the `reboot` tag.

### Force handlers to run again

If during execution, the playbook fails while executing a handler, the
next time it runs the handler will not run again, because the notifying
task will report an `ok` status.

As a workaround, you can force all handlers to run again by setting the
variable `run_handlers` to `true`. This works by causing all tasks that
trigger a handler to report a `changed` status.

It works differently than `--force-handlers`. As per Ansible documentation:

> When handlers are forced, they will run when notified even if a task fails
> on that host.

While, in our scenario, handlers would _not_ be notified by tasks when they
return `ok`.

## Configuration

By design, global configuration items are those which are used by multiple
roles and are stored in `group_vars/all/00-defaults.yaml`. Other variables,
which are local to a specific role, are stored under
`roles/$ROLE/defaults/main.yaml`. Both groups can be overridden by placing a
new file under `group_vars`, `host_vars` or using the command line. This way,
one can keep the default configuration and just change the target system
hostname or locale.

### Global configuration

The file `group_vars/all/00-default.yaml` contains global configuration
options that affect how the playbook work.

    global_device_node: /dev/sda
    global_device_efi_number: 1 
    global_partition_number: 1 
    global_efi_mount_point: /boot 
    global_mount_point: /mnt 

These options define the disk that will be used for partitioning during
the bootstrap phase, the device number of the EFI partition to create,
the index of the root partition, the EFI partition mount point and the
place where the root partition is going to be mounted. If partitioning
is skipped, `global_mount_point` is still relevant because the user must
manually mount volumes there.

    global_admins:
      - manu

Initial users to create. Passwords for individual users (including root) and
other user settings (i.e. groups) can be set in
`roles/users/defaults/main.yaml`.

    global_passwordless_sudo_user: package_builder

During certain tasks (such as when building packages from the AUR) the
playbook will need to drop privileges and use a non-root user, which
must be able to use sudo without a password. Think of a typical `makepkg
-s` call, which won't work as `root` but will then need to become `root`
to install dependencies.

This username is used to create a disposable unprivileged user for those
tasks. All its data are automatically purged before the playbook ends,
so that there are no users with passwordless sudo capabilities on the
system, unless you create one.

    global_portable_image: False

This variable controls whether the resulting installation should be
site-independent or not. If set to false, the playbook assumes that
settings such as custom repos and proxy configuration must persist in
the installed system. If set to true, such settings will be reverted in
the final setup.  This is useful, for example, if the installation
process requires using an HTTP proxy, but the system is then going to be
moved to a different network where a proxy is not needed. A typical case
is provisioning a VM image with Packer from behind a proxy: the final
image should not carry such site-specific proxy settings if it is going
to be shared with a wider audience.

If working behind a (HTTP(S)) proxy, add appropriate definitions for

* `http_proxy`
* `https_proxy`
* `no_proxy`

This will automatically trigger proxy-related tasks and configure the
installed system to work behind a proxy (by setting appropriate
environment variables).

### Role configuration

#### roles/base_packages/defaults/main.yaml

    base_packages_list:
      - base
      - base-devel
      - ...

List of base packages to be installed on the target system during the
`bootstrap` phase.

#### roles/hostname/defaults/main.yaml

    hostname_hostname: archlinux

Hostname information.

#### roles/locale/defaults/main.yaml

    locale_timezone: Europe/Rome
    locale_locale: it_IT.UTF-8
    locale_keymap: it

Locale information.

#### roles/makepkg/defaults/main.yaml

    makepkg_aur_url: https://aur.archlinux.org/cgit/aur.git/snapshot/

URL from which AUR packages are downloaded.

#### roles/ttf_fonts/defaults/main.yaml

    ttf_fonts_packages:
      - ttf-dejavu
      - ttf-ubuntu-font-family
      - ttf-iosevka

Packages installed by the `ttf_fonts` role.

#### roles/users/defaults/main.yaml

    users_info:
      root:
        password: "..."
      manu: password: "..." is_admin: true   # Optional item, true if missing
      groups:   [video, docker, adbusers] # Optional item, empty list if missing

Settings for new users created on the target system. This should contain
passwords for all users defined in `global_admins`. Users that have
`is_admin` set to a truthy value will be added to the `wheel` group.
Additional groups can be specified as the `groups` list. To retain backward
compatibility, all fields but `password` are optional. Don't try to add sudo
capabilities by manually adding `wheel` to `groups`, use `is_admin` instead.
This allows for a degree of flexibility in how sudo access is implemented.
For `root`, only `password` is considered.

#### roles/utils/defaults/main.yaml

    utils_packages:
      - docker-compose
      - go
      - ntfs-3g
      - p7zip
      - ...

Packages installed by the `utils` role.

#### roles/virtguest/defaults/main.yaml

    virtguest_supported_hypervisors:
      - virtualbox

    virtguest_virtualbox_packages:
      - linux-headers
      - virtualbox-guest-utils

#### roles/virtguest_force/defaults/main.yaml

    virtguest_force: ""

By default, the playbook uses facts to detect if the target system is running
under an hypervisor. If so, some behaviour will be adjusted accordingly:
guest additions are installed and enabled automatically and the screensaver
is disabled by default. If the hypervisor is unsupported, the playbook bails
out. To proceed under an unsupported hypervisor without its additions, skip
the `virtguest` tag.

Set `virtguest_force` to one of the members of
`virtguest_supported_hypervisors` to force the installation of the
corresponding additional packages. This may be useful in two cases:

* Ansible's setup module fails to detect the hypervisor for whatever
  reason;
* the user wants to install guest packages that do not correspond to the
  detected hypervisor.

#### roles/xscreensaver/defaults/main.yaml

    # Set to:
    # - "active" to force the screensaver to be active after installation
    # - "inactive" to force the screensaver to be inactive after installation
    # - Empty to preserve the default behaviour (fact-based choice)
    xscreensaver_override: ""

Can be used to override the default screensaver behaviour.

#### roles/xfce/defaults/main.yaml

    xfce_packages:
      - xfce4
      - xfce4-goodies

Packages installed by the `xfce` role.

#### roles/xfce_user_customizations/defaults/main.yaml

    xfce_user_customizations_packages:
      - gtk-engine-murrine
      - numix-gtk-theme-git
      - ...

Packages installed by the `xfce_user_customizations` role.

#### roles/xorg/defaults/main.yaml

    xorg_packages:
      - xorg
      - xorg-apps
      - ...

Packages installed by the `xorg` role. These are pulled as dependencies by
`xutils` and other roles that depend on a working X11 environment.

#### roles/xutils/defaults/main.yaml

    xutils_packages:
      - calibre
      - dropbox
      - firefox
      - ...

Packages installed by the `xutils` role.

#### roles/custom_repos/defaults/main.yaml

    custom_repos_list: [
    #  {
    #    name: cache,
    #    server: "http://10.0.2.2/x86_64/",
    #    siglevel: Optional TrustAll
    #  }
    ]

    custom_repos_servers: [
    #  "http://localhost:8080/$repo/os/$arch"
    ]

It is possible to add extra repositories or mirrors during the
installation process. `custom_repos_list` lists additional repositories,
while `custom_repos_servers` lists additional mirrors for predefined
repositories. They will persists in the final system if it is not
configured as a portable installation.

Both repositories and mirrors take precedence over those already
configured. This means that:

* additional repositories will take precedence over `core`, `extra` and
  other official repositories, so they can be used to override some
  official package with a local version;
* additional mirrors will be used instead of other mirrors present in
  the mirrorlist, unless they are not reachable and pacman tries the
  next one.

## Simple customization

### Add a non-X11 package

If you want to add a new non-X11 package to every installation and it does
not require special configuration, add it to the package list in
`roles/utils/meta/main.yaml`.

### Add an X11 package

If you want to add a new X11 package to every installation and it does not
require special configuration, add it to the package list in
`roles/xutils/meta/main.yaml`.

### Add a package that requires configuration

If you want to add a new package to every installation and it
requires special configuration (i.e. configuration files to be copied),
create a new role for it that includes files and templates.

Delegate the actual installation to the `packages` role. AUR packages
are fine since yay is used.

    import_role:
      name: packages
      vars:
        packages:
          - new_package_1
          - new_package_2

This approach allows for the maximum flexibility (i.e. to install and
configure an additional DE). If the package requires X11, add the `xorg`
role to its dependencies.

## Side projects

This project comes with two side projects, designed to ease the
provisioning of VM's with Arch-Ansible. They rely on HashiCorp
[Vagrant](https://www.vagrantup.com/) and
[Packer](https://www.packer.io/) and can be found under the folders with
the corresponding names. For simplicity, I'll refer to them as
[Arch-Vagrant](vagrant/README.md) and [Arch-Packer](packer/README.md).
Click the links to read their own docs.

## Integration with pkgproxy

If you are going to do multiple installations using arch-ansible in a short
interval, it may be best to save and reuse downloaded packages across runs.
To do that, the simplest way is to use the
[pkgproxy](https://github.com/buckket/pkgproxy) tool, which acts as a cache
between an Arch Linux mirror and your system.

It is very simple to use: you run it, configure it as a mirror for your
system, and every downloaded package will get stuffed into its cache.
arch-ansible has support for additional mirrors, so it is easy to tell it to
use your local pkgproxy instance by adding an extra file as described in
[Configuration](#Configuration). For example, you may add the following
contents to `group_vars/all/50-pkgproxy.yaml`:

    custom_repos_servers:
      - http://10.0.2.2:8080/$repo/os/$arch

The exact hostname and port will vary depending on where you run pkgproxy. The example
above assumes that Arch is running inside a VM, while pkgproxy is running on
the host, on port 8080.

## Bare metal installation

_NOTE: unless noted, all commands are intented to be run on the target
machine._

arch-ansible can provision physical machines, not just VM's. But some care
must be taken in preparing the host/group variables files, plus some
network-related actions. Also, you will need a separate PC to be used as the
Ansible controller, which must be able to connect to the target machine.

The target will be rebooted multiple times during the installation and if you
are using DHCP the IP may change across reboots. You may want to configure
your local DHCP server to give the target a fixed IP by using a MAC address
reservation.

When rebooting from the install media into the installed system, it would be
useful if the system firmware were configured to automatically boot into the
new system, otherwise the installation media would boot again and the
installation could not proceed.

The target will need to be accessible over SSH. Ensure that
`/etc/ssh/sshd_config` allows root login: it should contain the following
line:

    PermitRootLogin yes

Then give the root user a password:

    # passwd

And start SSH:

    # systemctl start sshd

At this point incoming connections are allowed, but it is better to let SSH
use a private key for authentication rather than a password. Let's generate a
new keypair (which can be reused across all installations; note the `-N ''`,
which means the private key is unencrypted so no password is asked when
connecting) and copy the public part to the target:

    # On the controller
    $ ssh-keygen -N '' -f ~/.ssh/arch-install
    $ ssh-copy-id -i ~/.ssh/arch-install root@your.machine.local

Edit your `$HOME/.ssh/config` file to associate your target system with this key:

    Host your.machine.local
    IdentityFile ~/.ssh/arch-install
    IdentitiesOnly yes

Edit the Ansible inventory file to point to the target machine.

Apply any other customization via host/group variables. And finally run:

    # On the controller
    $ ansible-playbook -i hosts.yaml --tags bootstrap,mainconfig site.yaml

After installation is complete, you can delete the key pair generated above,
from both the controller and the target.

[screenshot]: docs/screenshot.png

<!-- vi: set tw=72 et sw=2 fo=tcroqan autoindent: -->
