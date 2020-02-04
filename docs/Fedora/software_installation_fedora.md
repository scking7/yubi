# Configuring CentOS 7 for Yubikey SSH Authentication


## Software Installation


1\.  Install required packages.


    $ sudo dnf install autoconf curl gnupg2 gnupg2-smime pcsc-lite pcsc-tools pinentry pinentry-gtk pinentry-gnome3 ykpers yubikey-manager yubikey-personalization-gui yubico-piv-tool



2\. Enable on-demand availability of the PC/SC daemon via systemd.


    $ sudo systemctl enable pcscd.socket



3\. Download the Yubikey Manager GUI tool in AppImage form and set file permissions to executable. (Authenticate with any required HTTP proxies first as needed.)


```bash
$ mkdir ~/bin
$ cd !$
$ wget https://developers.yubico.com/yubikey-manager-qt/Releases/yubikey-manager-qt-1.1.2-linux.AppImage  
$ chmod 0750 *.AppImage
```



## Environment Configuration


4\. In all Gnome-Based desktop environments (Gnome 2, Gnome 3, MATE, Cinnamon, XFCE, etc.), disable Gnome-Shell's `ssh-agent` by copying & pasting the following into a terminal window (not including the `$` command-prompts). This is a kludgey process due multiple undocumented changes by Gnome developers -- we're implementing 3 different methods of attempting to get Gnome to obey our attempt to prevent it from hijacking the ssh agent.



**TODO: CHANGE ALL THIS MESS TO A SCRIPT TO BE DOWNLOADED AND RUN AS ROOT**

```bash
$ sudo -s
# cat <<EOF > /etc/X11/xinit/Xclients.d/Xclients.gnome-session.sh
if [[ $(gconftool-2 --get /apps/gnome-keyring/daemon-components/ssh) != "false" ]]; then
  gconftool-2 --type bool --set /apps/gnome-keyring/daemon-components/ssh false
fi
EOF

$ [ -f /etc/xdg/autostart/gnome-keyring-gpg.desktop ] && cp /etc/xdg/autostart/gnome-keyring-gpg.desktop ~/.config/autostart
$ cp /etc/xdg/autostart/gnome-keyring-ssh.desktop ~/.config/autostart
$ echo -e "X-GNOME-Autostart-enabled=false\nX-MATE-Autostart-enabled=false" >> ~/.config/autostart/gnome-keyring-gpg.desktop
$ echo -e "X-GNOME-Autostart-enabled=false\nX-MATE-Autostart-enabled=false" >> ~/.config/autostart/gnome-keyring-ssh.desktop

$ [ -f /etc/xdg/autostart/gnome-keyring-gpg.desktop ] && sudo mv /etc/xdg/autostart/gnome-keyring-gpg.desktop /etc/xdg/autostart/gnome-keyring-gpg.desktop.inactive
$ [ -f /etc/xdg/autostart/gnome-keyring-ssh.desktop ] && sudo mv /etc/xdg/autostart/gnome-keyring-ssh.desktop /etc/xdg/autostart/gnome-keyring-ssh.desktop.inactive


$ sudo -s 
# mv /usr/bin/gnome-keyring-daemon /usr/bin/gnome-keyring-daemon-wrapped
# cat <<EOF >> /usr/bin/gnome-keyring-daemon
#!/bin/sh
exec /usr/bin/gnome-keyring-daemon-wrapped --components=pkcs11,secrets "$@"

EOF
# chmod 0755 /usr/bin/gnome-keyring-daemon
```



5\.  Launch the Startup Applications Preferences in Gnome, MATE, and a few other Desktop Environments in order to disable the XOrg or Wayland agent and keyring daemons which will try to steal control from GPG.


    $ gnome-session-properties &


Disable any programs which reference similar to the following, as demonstrated in the image below:

  * Key Agent, SSH Key Agent
  * Certificates
  * Keys, Secrets, Key Storage, Secret Storage
  * Keyring
  * Smart Card Manager
  * Secret


![Gnome and MATE Session Preferences Window](gnome_mate_session_prefs.png)



6\. Configure `~/.bashrc` to be sourced on both login and non-login sessions by copying & pasting this command into a terminal window.


    $ echo -e "\n\n[ -f ~/.bashrc ] && . ~/.bashrc\n\n" >> ~/.bash_profile



7\. Configure `~/.bashrc` by copying & pasting the following into a terminal window (not including the `$` command-prompt).


```bash
$ cat <<EOF >> ~/.bashrc


####### GPG SSH AUTH
# Use gpg-agent for SSH instead of ssh-agent
export GPG_TTY=$(tty)
unset SSH_AGENT_PID

# Repoint SSH to gpg-agent's ssh socket
[ "${gnupg_SSH_AUTH_SOCK_by:-0}" -ne $$ ] && export SSH_AUTH_SOCK="$(/usr/bin/gpgconf --list-dirs agent-ssh-socket)"

# Start gpg-agent if not running
[ $(pidof gpg-agent 2>/dev/null) ] || gpg-agent --homedir $HOME/.gnupg --daemon --sh --enable-ssh-support > $HOME/.gnupg/env

# Source gpg-agent environment file
[ -f "$HOME/.gnupg/env" ] && . $HOME/.gnupg/env
####### 


EOF
```



8\. Replace or create `gpg.conf` by copying & pasting the following into a terminal window (not including the `$` command-prompt).


```bash
$ cat <<EOF > ~/.gnupg/gpg.conf

# Additional information
# https://www.gnupg.org/documentation/manuals/gnupg-devel/Option-Index.html

require-secmem
no-greeting
no-comments
force-mdc
escape-from-lines
interactive
no-emit-version
keyid-format 0xlong
require-cross-certification
verify-options show-uid-validity
list-options show-uid-validity show-unusable-subkeys
with-fingerprint
use-agent
ask-cert-level
auto-key-retrieve
#default-key 0x123456789ABCDEF0
default-recipient-self

personal-cipher-preferences AES256 AES192 CAMELLIA256 TWOFISH AES128
personal-digest-preferences SHA512 SHA384 SHA256 SHA224
personal-compress-preferences Uncompressed BZIP2 ZLIB ZIP

s2k-digest-algo SHA512
s2k-cipher-algo AES256

default-preference-list SHA512 SHA384 BZIP2

keyserver-options no-honor-keyserver-url no-include-attributes no-include-revoked

EOF
```



9\. Replace `gpg-agent.conf` by copying & pasting the following into a terminal window (not including the `$` command-prompt).


```bash
$ cat <<EOF > ~/.gnupg/gpg-agent.conf

enable-ssh-support
no-allow-external-cache
keep-tty
ignore-cache-for-signing

default-cache-ttl 60
max-cache-ttl 120

default-cache-ttl-ssh 900
max-cache-ttl-ssh 7200

pinentry-program /usr/local/bin/pinentry-mac


EOF
```



10\. Force restart gpg-agent after our changes.

    $ gpgconf --kill gpg-agent



11\. Source changes made to `~/.bashrc` by executing this command in a terminal window.

    $ . ~/.bashrc



That's it!



## Next Steps

Next steps are factory resetting and configuring the Yubikey, which will be discussed in the presentation, but also documented in [Yubikey Configuration](yubikey_configuration.md) for quick reference.

