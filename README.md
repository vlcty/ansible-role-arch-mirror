# arch-mirror

This anisble playbook creates an unoffical arch mirror for your own packages.

**This whole thing is still work in progress**

## Requirements

You need an arch server somewhere to host your packages and of course your own packages. You also need a PGP key for your mirror. Packages and the database are signed with that key.

You can find the necessary steps for the key generation under "Creating a GPG key".

## Variables

The following variables exist:

| Variable | Default value | Explanation |
|----------|---------------|-------------|
| mirror_basepath | /archmirror | Where the mirror files are kept |
| mirror_search_path | - | Where the playbook searches locally for packages to upload |
| mirror_name | - | The name of the mirror |
| mirror_pgp_pubkey | - | The PGP public key |
| mirror_pgp_privkey | - | The PGP private key |

Every variable is mandatory and has to be set.

## How it works

The playbook creates the directory defined under `mirror_basepath` on the server. Afterwards every package which is found locally under `mirror_search_path` is uploaded to the server under `mirror_basepath`.  Afterwards every package is digitally signed with the PGP key and the repo metafile is created. A darkhttpd server is started to be reachable for every client over http. There is no need for https here, because every file's PGP signature will be verified.

On the client side you have to add the repository. Open the file `/etc/pacman.conf` and append the following snippet:

```
[myrepo]
Server = http://myrepo.example.com
```

Replace `myrepo` with the value of `mirror_name` and replace the server address. This step should of course also be ansibled but is not part of this role.

## Creating a GPG key

All packages should be signed with an PGP key. This key should be always the same and is therefore not automatically created (and I was too lazy to do that). You have to manually create one (or use an existing key if you want). This key will then be distributed with this role.

Step by step creating the key:

### 1. Create a new keypair

Command: `gpg --full-gen-key --expert`

You will be asked what key should be generated. I chose to use an ECC key, because it's 2018 and we should slowly move away from RSA.  
Select the number which says `ECC (sign only)`. Afterwards you will be asked which curve you want to use. Select the number which says `Curve 25519`.
Enter the expiry date you want to use. **Please use an expiry date!**  
I've set mine to five years. Afterwards enter your desired name and comment.

When asked for a password enter an empty one and confirm that you know what you are doing. This playbook can only import unencrypted private keys.

Full transscript of a key creation:
```
[root@archlinux ~]# gpg2 --expert --full-gen-key
gpg (GnuPG) 2.2.3; Copyright (C) 2017 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Please select what kind of key you want:
   (1) RSA and RSA (default)
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
   (9) ECC and ECC
  (10) ECC (sign only)
  (11) ECC (set your own capabilities)
  (13) Existing key
Your selection? 10
Please select which elliptic curve you want:
   (1) Curve 25519
   (3) NIST P-256
   (4) NIST P-384
   (5) NIST P-521
   (6) Brainpool P-256
   (7) Brainpool P-384
   (8) Brainpool P-512
   (9) secp256k1
Your selection? 1
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 5y
Key expires at Fri 13 Jan 2023 11:20:07 AM UTC
Is this correct? (y/N) y

GnuPG needs to construct a user ID to identify your key.

Real name: Icinga2 Arch Package Mirror Signing Key
Email address: info@veloc1ty.de
Comment:
You selected this USER-ID:
    "Mirror Signing Key <mirror@example.com>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? O
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
```

### 2. Export the keys

The command ```gpg --list-keys``` should display all stored keys. Example output:

```
pub   ed25519 2018-01-14 [SC] [verfällt: 2023-01-13]
      50E1EF4F007CAEC146AC098754DAD922CD39505C
uid        [ ultimativ ] Mirror Signing Key <mirror@example.com>

```

The key id in this example is `50E1EF4F007CAEC146AC098754DAD922CD39505C`. You now need to export the private and public key.

Export the public key:
```
[root@archlinux ~]# gpg --armour --export 50E1EF4F007CAEC146AC098754DAD922CD39505C
-----BEGIN PGP PUBLIC KEY BLOCK-----

hereisyourkeyblablubbfucktrump
-----END PGP PUBLIC KEY BLOCK-----
```

Export the private key:
```
[root@archlinux ~]# gpg --armour --export-secret-keys 50E1EF4F007CAEC146AC098754DAD922CD39505C
-----BEGIN PGP PRIVATE KEY BLOCK-----

hereisyourkeyblablubbfucktrumpagain
-----END PGP PRIVATE KEY BLOCK-----
```

Save the two keys into a host_vars file for your mirror. Example:

```
---
mirror_pgp_pubkey: |
    -----BEGIN PGP PUBLIC KEY BLOCK-----
    hereisyourkeyblablubbfucktrump
    -----END PGP PUBLIC KEY BLOCK-----

mirror_pgp_privkey: |
    -----BEGIN PGP PRIVATE KEY BLOCK-----
    hereisyourkeyblablubbfucktrumpagain
    -----END PGP PRIVATE KEY BLOCK-----

```

Don't forget to encrypt the host_vars file with ansible-vault!

### 3. Delete the keys from your keyring

The keys should only be on the server. Delete the keys from your local keyring with `gpg --delete-secret-keys 50E1EF4F007CAEC146AC098754DAD922CD39505C` and `gpg --delete-keys 50E1EF4F007CAEC146AC098754DAD922CD39505C`.

## Protip: Distribute the mirror key to other clients

Pacman will tell you that your packages can't be verified, because the key is not trusted. I use the following tasks to copy the repo key on every server and trust it automatically:

Hint: The public key is stored in the variable `mirror_pgp_key`.

```
- name: Uploading arch mirror pgp key
  copy:
      content: "{{ mirror_pgp_key }}"
      dest: /root/arch-mirror-pgp-key.pub
  register: pgpkey

- name: Getting fingerprint from file
  shell: gpg --with-fingerprint --with-colons /root/arch-mirror-pgp-key.pub | grep fpr | cut -d ':' -f 10
  register: pgpfingerprint
  when: pgpkey.changed == true

- name: Import arch mirror pgp key
  shell: "pacman-key --add /root/arch-mirror-pgp-key.pub"
  when: pgpkey.changed == true

- name: Creating ownertrust
  # An 'ownertrust' file is created. Basically every line contains a fingerprint
  # and the desired trustlevel. 6 is ultimate
  copy:
      content: "{{ pgpfingerprint.stdout }}:6:\n"
      dest: /tmp/ownertrust
  when: pgpkey.changed == true

- name: Import ownertrust
  shell: "gpg --homedir /etc/pacman.d/gnupg --import-ownertrust /tmp/ownertrust"
  when: pgpkey.changed == true
```

## References

How to set up a personal mirror (Arch Wiki): https://wiki.archlinux.org/index.php/Pacman/Tips_and_tricks#Custom_local_repository
