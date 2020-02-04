# Yubikey Software Installation for macOS via Homebrew


Instructions for software installation via Homebrew to support configuration and usage of a Yubikey for FIDO U2F and SSH PKI authentication. Valid for macOS Mojave, High Sierra, and Sierra, leveraging Homebrew, an  opensource macOS software repository project.


*This overview assumes a functional Homebrew installation with Internet connectivity, successful proxy authentication as appropriate, etc. For assistance in setting up Homebrew, see [Installing Homebrew](installing_homebrew.md).*



1\. Install packages via Homebrew


```
$ brew install gnupg lsusb pidof pwgen ssh-copy-id wget ykman ykpers pinentry pinentry-mac	
$ brew cask install gpg-suite
```



2\. Modify `.bashrc` / `.zshrc`


```
$ cat <<EOF >> ~/.bashrc


### GPG SSH AUTH
# Use gpg-agent for SSH instead of ssh-agent
export GPG_TTY=$(tty)
unset SSH_AGENT_PID

# Repoint SSH to gpg-agent's ssh socket
[ "${gnupg_SSH_AUTH_SOCK_by:-0}" -ne $$ ] && export SSH_AUTH_SOCK=$HOME/gnupg/S.gpg-agent.ssh

# Start gpg-agent if not running
[ $(pidof gpg-agent 2>/dev/null) ] || gpg-agent --homedir $HOME/.gnupg --daemon --sh --enable-ssh-support > $HOME/.gnupg/env

# Source gpg-agent environment file
[ -f "$HOME/.gnupg/env" ] && . $HOME/.gnupg/env


EOF
```


```
$ sed -i -e 's/^PATH=.*/&:\/usr\/local\/MacGPG2\/bin/g' ~/.bashrc
```



3\. Backup original file copies


```
$ mkdir -p ~/.gnupg/orig
$ mv ~/.gnupg/gpg.conf ~/.gnupg/orig/
$ mv ~/.gnupg/gpg-agent.conf ~/.gnupg/orig/
```



4\. Replace `gpg.conf`


```
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



5\. Replace `gpg-agent.conf`:


```
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



6\. Kill any pre-existing `gpg-agent` or `ssh-agent` instances which may have been running, as they are not configured with our new configuration changes. First cleanly and then more forcefully for good measure.

    $ gpgconf --kill gpg-agent
    $ killall gpg-agent
    $ killall ssh-agent



7\. Source changes made to `/.bashrc`

    $ . ~/.bashrc



That's it!


Next steps are to configure the Yubikey by factory resetting the OpenPGP applet and configuring the Yubikey mode, which will be discussed in the presentation, but also documented in [Yubikey Configuration](yubikey_configuration.md) for quick reference.

