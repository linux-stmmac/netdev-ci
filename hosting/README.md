# Hosting

`results/` of this repo is published at https://netdevci.bootlin.com/stmmac/.
On the web VM, `netdevci-sync` (every 5 minutes) fetches the repo and mirrors
`results/` into `/var/www/netdevci/stmmac/`:

- only regular files of `results/`: symlinks, devices and fifos are skipped,
  and nothing outside `results/` is ever extracted;
- no dotfiles or dot-directories at any depth (`.htaccess`, `.git*`, ...);
- a file removed from `results/` is removed from the web root;
- the VM holds no credentials: the repo is fetched anonymously over https.

## Install (as root on the VM)

```sh
apt install git rsync            # or dnf; the script refuses to run without them
install -d -o netdevci -m 0755 /var/www/netdevci/stmmac
install -m 0755 netdevci-sync /usr/local/bin/netdevci-sync
install -m 0644 netdevci-sync.service netdevci-sync.timer /etc/systemd/system/
systemctl daemon-reload
systemctl enable --now netdevci-sync.timer
systemctl start netdevci-sync.service && journalctl -u netdevci-sync -n 20
```

The service runs as the VM's `netdevci` user, but can write only
`/var/www/netdevci/stmmac` and its state directory `/var/lib/netdevci-sync`,
and sees no `/home`.

## Install without root (systemd --user)

As `netdevci`, with git and rsync installed:

```sh
install -D -m 0755 netdevci-sync ~/.local/bin/netdevci-sync
install -D -m 0644 user/netdevci-sync.service user/netdevci-sync.timer -t ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now netdevci-sync.timer
systemctl --user start netdevci-sync.service && journalctl --user -u netdevci-sync -n 20
loginctl show-user "$USER" -p Linger
```

The timer only runs while the user manager does: `Linger=yes` keeps it
running without a login and across reboots (`loginctl enable-linger`, or
ask the admin). State lives in `~/.local/state/netdevci-sync`.

## Web server

Test links point at directories (`outputs/<run>/test-outputs/<n>-<test>/`),
so `/stmmac/` needs directory listings; the logs have no extension, so they
should be served as text. Dotfiles are refused as well, in case one ever
lands there (rsync stages updates in `.~tmp~` before renaming them in).

In the `server` block serving netdevci.bootlin.com (nginx):

```nginx
location /stmmac/ {
    autoindex on;
    default_type text/plain;
    charset utf-8;
    disable_symlinks on;
    location ~ /\. { deny all; }
}
```

Then `nginx -t && systemctl reload nginx`.
