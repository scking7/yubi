# Configuring CentOS/RHEL 7 for Yubikey SSH Authentication


## Software Installation


1\.  Many of the required packages are part of the EPEL yum repository, so that repo will be installed first.


    $ sudo yum install epel-release



2\.  Install required packages.


    $ sudo yum install autoconf curl gnupg2 gnupg2-smime pcsc-lite pcsc-tools pinentry pinentry-gtk pinentry-gnome3 ykpers yubikey-manager yubikey-personalization-gui yubico-piv-tool



3\. Enable on-demand availability of the PC/SC daemon via systemd.


    $ sudo systemctl enable pcscd.socket



4\. Download the Yubikey Manager GUI tool in AppImage form, **verify its legitimacy**, and set file permissions to executable.

(*Need to [know more](https://itsfoss.com/use-appimage-linux/) about AppImage packages?*)

Download the latest version of the Yubikey Manager **in AppImage format**, as listed at <https://developers.yubico.com/yubikey-manager-qt/Releases/>, along with its matching `.sig`, for instance v1.1.4.


```bash
$ mkdir ~/bin
$ cd !$
$ wget https://developers.yubico.com/yubikey-manager-qt/Releases/yubikey-manager-qt-1.1.4-linux.AppImage 
$ wget https://developers.yubico.com/yubikey-manager-qt/Releases/yubikey-manager-qt-1.1.4-linux.AppImage.sig
```



5\. Determine the key used to sign the package by attempting to verify the signature. The missing key will be identified by GPG.

```bash
$ gpg --verify yubikey-manager*.sig
gpg: assuming signed data in 'yubikey-manager-qt-1.1.4-linux.AppImage'
gpg: Signature made Wed Jan 29 06:11:10 2020 CST
gpg:                using RSA key 59944611C823D88CEB7245B906FC004369E7D338
gpg: Can't check signature: No public key
```

6\. Initiate the download and import the key from OpenPGP's key server, **but do not complete the import process when prompted just yet -- we need to verify the key fingerprint first**. (*Note that the key ID is obtained from the signature verification attempt in step 5 above.*)


```bash
$ echo "keyserver hkps://keys.openpgp.org" >> ~/.gnupg/dirmngr.conf
$ gpg --recv-keys 59944611C823D88CEB7245B906FC004369E7D338

pub  rsa4096/0x514F078FF4AB24C3  created: 2016-05-23  expires: 2020-05-01
      Key fingerprint = 8D0B 4EBA 9345 254B CEC0  E843 514F 078F F4AB 24C3

     Dag Heyman <dag@yubico.com>

Do you want to import this key? (y/N)
```

Here we see the fingerprint of the signing key indicated by GPG. Compare this fingerprint with the matching key as listed on Yubico's [Software Signing Page](https://developers.yubico.com/Software_Projects/Software_Signing.html).

**If and only if the email address and key fingerprint listed on the webpage** (*see example image immediately below*) **match the output of the `--recv-keys` output from step 5 above**, confirm the key import. **Otherwise decline the import and do not use the software package as its legitimacy cannot be verified.**


![Key Fingerprint Verification](fpverify.png)



7\. Assuming the key fingerprints matched and the key was successfully imported, re-attempt the signature verification. Based upon the current configuration of GPG, some warnings regarding trust level or trust signatures may be indicated, but may be safely disregarded. **The key data point to verify authenticity is to ensure the signature is reported as "Good signature" (marked below via "`<=<=<=<=<=<=<`).**

```bash
$ gpg --verify ./yubikey-manager*.sig
gpg: assuming signed data in './yubikey-manager-qt-1.1.4-linux.AppImage'
gpg: Signature made Wed Jan 29 06:11:10 2020 CST
gpg:                using RSA key 59944611C823D88CEB7245B906FC004369E7D338
gpg: Good signature from "Dag Heyman <dag@yubico.com>" [unknown]      <=<=<=<=<=<=<
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 8D0B 4EBA 9345 254B CEC0  E843 514F 078F F4AB 24C3
     Subkey fingerprint: 5994 4611 C823 D88C EB72  45B9 06FC 0043 69E7 D338
```



8\. Lastly, ensure the AppImag file is executable.

    $ chmod 0750 *.AppImage



## Environment Configuration


9\. In all Gnome-Based desktop environments (Gnome 2, Gnome 3, MATE, Cinnamon, XFCE, etc.), disable Gnome-Shell's `ssh-agent` by copying & pasting the following into a terminal window (not including the `$` command-prompts).


```bash
$ sudo cat <<EOF >> /etc/X11/xinit/Xclients.d/Xclients.gnome-session.sh
if [[ $(gconftool-2 --get /apps/gnome-keyring/daemon-components/ssh) != "false" ]]; then
  gconftool-2 --type bool --set /apps/gnome-keyring/daemon-components/ssh false
fi
EOF

$ sudo mv /etc/xdg/autostart/gnome-keyring-gpg.desktop /etc/xdg/autostart/gnome-keyring-gpg.desktop.inactive
$ sudo mv /etc/xdg/autostart/gnome-keyring-ssh.desktop /etc/xdg/autostart/gnome-keyring-ssh.desktop.inactive
```



10\. Launch the Startup Applications Preferences in Gnome, MATE, and a few other Desktop Environments in order to disable the XOrg or Wayland agent and keyring daemons which will try to steal control from GPG.


    $ gnome-session-properties &


Once launched, disable any auto-start program entries which contain or reference these items as demonstrated in the image below (or similarly named, as the naming convention often changes):

  * Key Agent, SSH Key Agent
  * Certificates
  * Keys, Secrets, Key Storage, Secret Storage
  * Keyring
  * Smart Card Manager
  * Secret


![Gnome and MATE Session Preferences Window](gnome_mate_session_prefs.png)


**NOTE: If any such items were enabled, it is necessary to log out of the current Xorg/Wayland session and log back in to ensure these daemons and applets are properly shutdown. Alternatively, manually kill the PIDs or reboot the workstation.**



11\. Configure `~/.bashrc` to be sourced on both login and non-login sessions by copying & pasting this command into a terminal window.


    $ echo -e "\n\n[ -f ~/.bashrc ] && . ~/.bashrc\n\n" >> ~/.bash_profile



12\. Configure `~/.bashrc` by copying & pasting the following into a terminal window (not including the `$` command-prompt).


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
$
```



13\. Replace or create `gpg.conf` by copying & pasting the following into a terminal window (not including the `$` command-prompt).


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
$
```



14\. Replace `gpg-agent.conf` by copying & pasting the following into a terminal window (not including the `$` command-prompt).


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
$
```


15\. Kill any pre-existing `gpg-agent` or `ssh-agent` instances which may have been running, first cleanly and then more forcefully for good measure.

    $ gpgconf --kill gpg-agent
    $ killall gpg-agent
    $ killall ssh-agent



16\. Source changes made to `~/.bashrc` by executing this command in a terminal window.

    $ . ~/.bashrc



That's it!



## Next Steps

Next steps are factory resetting and configuring the Yubikey, which will be discussed in the presentation, but also documented in [Yubikey Configuration](yubikey_configuration.md) for quick reference.

