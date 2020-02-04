Maschine Local Provisioning
===========================

My personal workstation playbook. Provisions an Arch Linux installation with a KDE Plasma desktop.
It's supposed to be used after the [Arch Installtion Guide](INSTALL.md) is completed, you have setup a working internet connection and have installed `ansible` & `git`.

## Running

After installing `ansible`, install and update the submodules:

    $ git submodule init && git submodule update

Then run the playbook as root.

    # ansible-playbook maschine.yml

When run, Ansible will prompt for the user password. This only needs to be
provided on the first run when the user is being created. On later runs,
providing any password -- whether the current user password or a new one --
will have no effect.

## Special Backup Directories

- KMail
  - `~/.local/share/local-mail/`
  - `~/.local/share/akonadi/`
  - `~/.config/akonadi/`
  - `~/.config/akonadi_*_resource`
  - `~/.config/emailidentities`
  - `~/.config/emaildefaults`
- KWallet
  - `~/.local/share/kwalletd`
- NetworkManager (e.g. for VPN configs)
  - `/etc/NetworkManager/system-connections/`
  - `~/.local/share/networkmanagement/certificates/`

## Credits

Parts of this playbook were inspired by [spark](https://github.com/pigmonkey/spark), written by [pigmonkey](https://github.com/pigmonkey).
I also use [ansible-aur](https://github.com/pigmonkey/ansible-aur) by the same author, to install base aur packages.
