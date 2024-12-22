# Strongbox

 Decrypt ZFS root volumes with `systemd`, with TPM backed encryption

 > [!warning]
 > Version 2, introduced with commit b151cb7, uses a completely new method for encryption, which will break existing installations.

## Installation

Download the PKGBUILD and run `makepkg -si`

## Setup

`strongbox` relies on the `strongbox:address` ZFS user property to determine the decryption method that should be applied for a dataset. This can be set via `zfs set strongbox:address=VALUE dataset`. There are three values:

1. `noboot`: For datasets that should not be decrypted on boot.
2. `notpm`: For datasets that do not have a secret stored on the TPM, and to which a key should be prompted for.
3 A hexadecimal address within the TPM NVRAM, such as `0x81010010`.

For inherited encryption, decryption will only be done on the encryption root.

Additionally, the root dataset must be specified via the `zfs=` kernel argument, such as `zfs=rpool/root`.

Then, enable the following `systemd` services:

1. `strongbox.service`: Handles loading the root's key and mounting at boot-time
2. `strongbox-import.service`: Handles importing the root volume
3. `strongbox-shutdown-ramfs.service`: `mkinitcpio` does not properly generate a shutdown ramfs, which prevents ZFS from properly exporting on shutdown. This service handles it.

Finally, add the `strongbox` `mkinitcpio` hook to the HOOKS array. You must use the `systemd` hook, as opposed to the `base` hook, and `strongbox` must be *after* `systemd` and `keyboard`.

No other services from `zfs-utils` needs to be enabled (`zfs.target`, `zfs-import.service`, etc) However, you will need a `cachefile` and `hostid` before generating the initcpio. This can be done with:

```bash
zpool set cachefile=/etc/zfs/zpool.cache ROOT
zgenhostid $(hostid)
```

## Encryption and TPM

If the keystatus of the root pool is unavailable at boot, `strongbox` will default to using `handle.tpm` for decryption. This allows for the encryption key to be sealed using the TPM, with PCRs 0+7, which will then be randomized after loading. This ensures that, once the password is loaded, it cannot be retrieved from the boot system. To use this, run the following:

```bash
handle.tpm -m seal -a ADDRESS
```

If you have already booted into a system using strongbox, and hence the PCR values have been randomized, you can use the `strongbox.enroll` kernel argument. Setting it to `on` will disable the PCR clearing on bootup, giving you an accurate PCR state once in the system, and allowing you to seal secrets to the same environment as what would be in the boot environment. Ensure that secure boot is enabled on boot so that its PCR value is used.

> [!warning]
> `strongbox.enroll=on` allows anyone with root access to leak TPM secrets. Only use this for enrollment, and then immediately disable!

## Home Decryption

`strongbox` provides a mechanism for automatically unlocking a home dataset on user-login, utilizing PAM.

The requirements for this are:

1. The home dataset must be located at `ROOT/home/USER` where `ROOT` is the pool defined in `zfs=`, and `USER` is the respective user.
2. The home dataset is encrypted, identical to the user's password.

To enable, add the following line to the end of the `auth` block in `/etc/pam.d/system-auth`:

```
    auth       required		       pam_exec.so 	    expose_authtok  /usr/bin/home-decrypt
```

Then, any login of the user will automatically:

1. Unlock the home dataset
2. Mount a `tmpfs` to the user's `.cache`
3. Mount the home dataset to `/home/USER`
