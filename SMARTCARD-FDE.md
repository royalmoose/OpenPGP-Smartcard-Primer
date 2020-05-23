# FDE using OpenPGP Smartcard

How does this work?  
Instead of using a passphrase on boot, we can use a LUKS key file encrypted with GPG to unlock the disk partition on your computer.

**General setup process:**  
1. Create a dedicated GPG keyring environment for boot.  
2. Generate a LUKS key file and assign it to a slot in your encrypted LUKS partition.  
3. Encrypt the LUKS key file with your GPG private key.  
4. Create and edit existing `initramdisk` hooks.  
5. *Optional-* Remove the option to unlock with a passphrase.


## 1. Prepare the GPG Keyring Environment for boot
***Variables***  
These will depend on your environment. These need to be changed when following the guide.

|Variable|Description|Example|
|--|--|--|
|`<PubKeyLocation>`|This is the location of your Public Key.|`public.key`|
|`<KeyUUID>`|This is the ID for your key pair. This can be first or last name, or email assigned to your GPG key pair.|`john.smith@gmail.com`|
|`<LUKSPartition>`|This is the location of the LUKS partition you want to unlock when booting. You can use `lsblk --fs` to check to see which partition it is.|`/dev/sda3`|

1. Jump into root:
  >This will differ depending whether you have a dedicated root account set up or not.

	`su -l`  
	or  
	`sudo -i`

2. Create the directory the keyring will be stored:  
	`mkdir /etc/luks_gpg`  
	`chown root:root /etc/luks_gpg`  
	`chmod 700 /etc/luks_gpg`

3. Create the Private Key stubs in the keyring:  
	`gpg --homedir /etc/luks_gpg --card-edit`

4. Import your Public Key into the keyring.  
	>If a public key is already available, this can be imported normally. If not, this can be fetched using the OpenPGP smart card URL.

	*If Public Key already available:*  
	`gpg --homedir /etc/luks_gpg --import <PubKeyLocation>`  

	*If not available, fetch from smart card URL:*  
	`gpg --homedir /etc/luks_gpg --card-edit`  
	`gpg/card> fetch`  
  `gpg/card> quit`

6. **Trust the imported Public Key:**  
	`gpg --homedir /etc/luks_gpg --edit-key <KeyUUID>`


## 2. Assign new LUKS key file  
1. Generate 256-bit random key:  
	`dd if=/dev/random bs=1 count=256 > /etc/luks_gpg/disk.key`

2. Add the decryption key to a slot in your encrypted partition:  
	`cryptsetup luksAddKey <LUKSPartition> /etc/luks_gpg/disk.key`

4.  Verify with:  
	`cryptsetup luksDump <LUKSPartition>`

5.  Encrypt the decryption key using GnuPG private key:  
	`gpg --homedir /etc/luks_gpg/ --encrypt --recipient <KeyUUID> /etc/luks_gpg/disk.key`

6.  Shred the plaintext disk encryption key:  
	`shred -u /etc/luks_gpg/disk.key`


## 3. Create the Decryption script  
1. Create the following script:  
	`/usr/local/sbin/luks_gpg_decrypt.sh`  

	Containing:

        #!/bin/sh

        #Set the keyring environment
        export GNUPGHOME=/etc/luks_gpg/

        #Request PIN from user
        read -p "Enter PIN: " -s pincode

        #Decrypt the keyfile
        echo "$pincode" | gpg --batch --pinentry-mode loopback --passphrase-fd 0 \
        	--no-tty --decrypt /etc/luks_gpg/disk.key.gpg

3.  Fix script permissions:  
	`chown root:root /usr/local/sbin/luks_gpg_decrypt.sh`  
	`chmod 750 /usr/local/sbin/luks_gpg_decrypt.sh`  

4. You can test the script by running the following:  
	`/usr/local/sbin/luks_gpg_decrypt.sh | cryptsetup --key-file - --test-passphrase open <LUKSPartition>`

5. Update the `/etc/crypttab` hook by appending the following to the first line:  
	   `,keyscript=/usr/local/sbin/luks_gnupg_decrypt.sh`


## 4. Create the `initramdisk` hook

1. Backup the existing initramdisk file:  
	 `p -a /boot/initrd.img*-amd64 /boot/initrd.bak`  
	 This will allow us to boot back into the system using the passphrase if something fails.

2. Create the following file:  
	`/etc/initramfs-tools/hooks/luks_gpg`  

	Containing:

		#!/bin/sh

		PREREQ="cryptroot"

		prereqs()
		{
			echo "$PREREQ"
		}

		case $1 in
		prereqs)
			prereqs
			exit 0
			;;
		esac

		. /usr/share/initramfs-tools/hook-functions

		#Deploy the keyring
		cp -a /etc/luks_gpg/ ${DESTDIR}/etc/

		#Deploy the GPG binaries
		copy_exec /usr/bin/gpg
		copy_exec /usr/bin/gpg-agent
		copy_exec /usr/bin/pinentry-curses
		copy_exec /usr/lib/gnupg/scdaemon

3.  Fix script permissions:  
	`chown root:root /etc/initramfs-tools/hooks/luks_gpg`  
	`chmod 750 /etc/initramfs-tools/hooks/luks_gpg`  

4. Update `initramdisk`:  
	`update-initramfs -u -k all`  
	>This should return no errors. If syntax errors persist and cannot be fixed, copy a hook in `/etc/initramfs-tools/hooks/` and edit that to match the hook above.

5. Reboot.
> ***Boot errors***  
> If there is an error in booting, you can fall back to the backup
> initramdisk file and use the passphrase only:  
> 1.  When you reach the bootloader, move the cursor to  `Debian GNU/Linux`  entry.
> 2.  Press  `e`.
> 3.  Navigate to the  _initrd_  line, and rename filename to  `initrd.bak`.
> 4.  Press  `Ctrl-x`  to boot using the backup initramdisk.
