Maschine Local Provisioning
===========================

My personal workstation playbook. Provisions an Arch Linux installation with a KDE Plasma desktop. Mainly used so i don't forget how i configured my machine ;)

It's supposed to be used after the [Arch Installation Guide](INSTALL.md) is completed, you have setup a working internet connection and have installed `ansible` & `git`.

## Running

After installing `ansible`, install and update the submodules:

    $ git submodule init && git submodule update

Then run the playbook as root.

    # ansible-playbook maschine.yml

When run, Ansible will prompt for the user password. This only needs to be provided on the first run, when the user is being created.
On later runs, providing any password -- whether it is the current user password or a new one -- will have no effect.

## Shell

ZSH will be set as the default shell for all users, including root. The grml config is used.

If you wish to add to your `.zshrc` config, add a file under `~/.zsh`, because the default `.zshrc` will be overwritten on the next Ansible run.

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
