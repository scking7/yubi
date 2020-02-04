# Primary Key Configuration Environment

Unless you know exactly what you are doing and with whom you will be securely 
communicating and what PGP clients those recipients are using, do **not** use 
these configuration files for day-to-day GnuPG operations.

Doing so will almost certainly result in incompatibilities for other
recipients when they attempt to decrypt content and/or verify PGP signatures
you generate in your GnuPG environment.

These GnuPG configuration files should only be leveraged in an isolated 
environment used when working with a Primary / Master PGP key.

