# Configuring Windows for Yubikey SSH Authentication


I don't use Windows, and have never attempted to leverage a Yubikey on a Windows platform, so I'm relying on unverified third-party information in this scenario. Information from third-party sources indicates the following steps should work.



## Software Installation



1\. Download and install the latest versions of:

  * GPG4Win [download link](https://www.gpg4win.org/download.html)
  * PuTTY [download link](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)



#### Cygwin Users
  * Do ***not*** install the `gnupg` packages in Cygwin. Ensure the binaries installed by Gpg4Win are being used instead.
  * Install the `ssh-pageant` package


#### Native Windows SSH Client Users
  * Tunneling and connectivity issues with the relatively new Windows native SSH client have been reported
  * If you are experiencing connectivity or authentication issues, try switching to PuTTY or KiTTY



2\. Add the installation directory (e.g. `C:\Program Files (x86)\GNU\GnuPG`) to the `PATH` environment to allow for calling the `gpg` executable without having to specify the full path of the executable.




## Software Configuration


3\. Determine the GnuPG configuration directory by executing `gpg --version` from the command prompt (commonly `%HOMEPATH%\gnupg\` in Windows 10 or `C:\%APPDATA%\gnupg\` on older Windows versions).



4\. Create the following `gpg.conf` and `gpg-agent.conf` configuration files in the configuration directory determined in step 3.



### `gpg.conf`

```
# gpg.conf
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

```


### `gpg-agent.conf`

```
# gpg-agent.conf

enable-putty-support
enable-ssh-support

default-cache-ttl 60
max-cache-ttl 120

default-cache-ttl-ssh 900
max-cache-ttl-ssh 7200
```



5\.  Restart the `gpg-connect-agent` instance to load the configuration changes.

```
C:\> gpg-connect-agent killagent /bye
C:\> gpg-connect-agent /bye
```



6\. Test communication with the Yubikey via gpg status check:

    C:\> gpg --card-status
    




That's it (hopefully)?



## Next Steps

Next steps are to factory resetting and configuring the Yubikey, which will be discussed in the presentation, but also documented in [Yubikey Configuration](yubikey_configuration.md) for quick reference.

