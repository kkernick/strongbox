# Strongbox

 Decrypt ZFS root volumes with `systemd`, with TPM backed encryption

 > [!warning]
 > Version 2, introduced with commit b151cb7, uses a completely new method for encryption, which will break existing installations.

## Installation

Download the PKGBUILD and run `makepkg -si`

## Setup

`strongbox` relies on the `strongbox` ZFS user properties to determine the decryption method that should be applied for a dataset. There are two properties that should be defined for each dataset (Although inheritance can provide defaults). The first is `strongbox:address`, which specifies the TPM NVRAM address that the secret is sealed, in hexadecimal format such as `0x81010010`. This value need only be specified for *encryption roots*, so if your pool is encrypted at the pool level, datasets need not have an empty value, since `strongbox` only considers the encryption root for each dataset and uses *its* value for decryption.

The second property is `strongbox:strategy`, which defines the strategy for dataset decryption in the initramfs. There are four options:

1. `both`: Try using the TPM to decrypt the encryption root at `strongbox:address`; if it fails, fallback to manual password entry. This should be the default value, as it permits a seamless decryption in an attested state, but allows fallback in case of issues.
2. `tpm`: Use the TPM to decrypt the encryption root at `strongbox:address`. If it fails, leave the dataset encrypted. Use this for datasets that should only be available if the state can be attested; if you have more than one encryption root, each will be prompted for if `both` is the strategy, which can be tedious. Use `tpm` for non-essential datasets.
3. `pin`: Only prompt for manual password entry. Use this if you do not have a TPM, or want to proactively defend against Evil Maid attacks by requiring a password to boot.
4. `none`: Do not try and decrypt the dataset at boot time. Use this if you have encrypted datasets that you don't need decrypted at boot time, such as a home dataset tied to the user's password.
5. `enhanced`: Use the `-e` flag within `handle.tpm` for enhanced sealing. With this setup, secrets are sealed with PCR attestation, but are additionally encrypted with a PIN. This is useful for the root dataset, as it prevents cold-boot attacks while requiring a simpler PIN on boot rather than the encryption key.

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

## Pool Imports

By default, `strongbox` only imports the root pool during bootup in the initramfs. If you wish to use the TPM to decrypt other pools using the TPM and attested state in the initramfs, you can trivially add overrides to the `strongbox-import` service. Simply create the `/usr/lib/systemd/system/strongbox-import.service.d` folder, and create `.conf` files that contain the drop-ins. For example:

```
[Service]
ExecStart=zpool import -c /etc/zfs/zdata.cache zdata -N
```

Note that the entire `/etc/zfs` directory is copied into the initramfs, which means that so long as your zpool's cache file exists in that folder, it will be included and available within the boot environment
