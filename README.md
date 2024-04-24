# Strongbox

 Decrypt ZFS root volumes with `systemd`, with TPM backed encryption

## Installation

Download the PKGBUILD and run `makepkg -si`

## Setup

Firstly, the root volume must have a few conditions:

1. The root is stored as a dataset within the root pool (IE `zroot/root`)
2. The root dataset has a `legacy` mountpoint. If this isn't true, it can be modified via `zfs set mountpoint=legacy POOL`
3. If encryption is used, it must be applied to the pool globally, rather than just the root dataset (IE `zroot` must have encryption, not just `zroot/root`)

Next, the following kernel arguments are needed:

1. A `zfs=` argument that points to the pool + dataset of the root (IE `zroot/root`)
2. An `address=` argument that defines the TPM address that the encryption key is sealed at. Can be set to `notpm` to simply prompt the user for a password at boot-time, or omitted entirely if encryption isn't used.

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

Where `ADDRESS` is the TPM address in the `address=` kernel argument.

To simply prompt for a password, set the value to `address=notpm`.

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