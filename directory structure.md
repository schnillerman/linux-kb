[Explanation of Linux' directory structure:](https://www.howtogeek.com/117435/htg-explains-the-linux-directory-structure-explained/):

### `/`
<details><summary>The Root Directory</summary>
  Everything on your Linux system is located under the / directory, known as the root directory. You can think of the / directory as being similar to the C:\ directory on Windows—but this isn't strictly true, as Linux doesn't have drive letters. While another partition would be located at D:\ on Windows, this other partition would appear in another folder under / on Linux.

![grafik](https://github.com/user-attachments/assets/f3496051-b6ed-4e43-b6ca-7a2240da6749)
</details>

### `/bin`
<details><summary>Essential User Binaries</summary>
 The /bin directory contains the essential user binaries (programs) that must be present when the system is mounted in single-user mode. Applications such as Firefox, if they aren't installed as Snaps, are stored in /usr/bin, while important system programs and utilities such as the bash shell are located in /bin. The /usr directory may be stored on another partition. Placing these files in the /bin directory ensures the system will have these important utilities even if no other file systems are mounted. The /sbin directory is similar: it contains essential system administration binaries.

  ![grafik](https://github.com/user-attachments/assets/58154dc0-1e22-4378-922c-b16c97da59d6)
</details>

### `/boot`
<details><summary>Static Boot Files</summary>
 The /boot directory contains the files needed to boot the system. For example, the GRUB boot loader's files and your Linux kernels are stored here. The boot loader's configuration files aren't located here, though; they're in /etc with the other configuration files. 
</details>

### `/cdrom`
<details><summary>Historical Mount Point for CD-ROMs</summary>
  The /cdrom directory isn't part of the FHS standard, but you'll still find it on Ubuntu and other operating systems. It's a temporary location for CD-ROMs inserted in the system. However, the standard location for temporary media is inside the /media directory.
</details>

### `/dev`
<details><summary>Device Files</summary>
  Linux exposes devices as files, and the /dev directory contains a number of special files that represent devices. These are not actual files as we know them, but they appear as files. For example, /dev/sda represents the first SATA drive in the system. If you wanted to partition it, you could start a partition editor and tell it to edit /dev/sda.
  
  This directory also contains pseudo-devices, which are virtual devices that don't actually correspond to hardware. For example, /dev/random produces random numbers. /dev/null is a special device that produces no output and automatically discards all input; when you pipe the output of a command to /dev/null, you discard it.

![grafik](https://github.com/user-attachments/assets/26b4806e-a7c2-4a7b-b5b6-51b731250d6f)

</details>

### `/etc`
<details><summary>Configuration Files</summary>
  The /etc directory contains configuration files, which can generally be edited by hand in a text editor. Note that the /etc/ directory contains system-wide configuration files. User-specific configuration files are located in each user's home directory.
</details>

### `/home`
<details><summary>Home Folders</summary>
  The /home directory contains a home folder for each user. For example, if your user name is bob, you have a home folder located at /home/bob. This home folder contains the user's data files and user-specific configuration files. Each user only has write access to their own home folder and must obtain elevated permissions (become the root user) to modify other files on the system.
</details>

### `/lib`
<details><summary>Essential Shared Libraries</summary>
  The /lib directory contains libraries needed by the essential binaries in the /bin and /sbin folder. Libraries needed by the binaries in the /usr/bin folder are located in /usr/lib. You'll also see a counterpart /lib64 folder on 64-bit systems.
</details>

### `/lost+found`
<details><summary>Recovered Files</summary>
  Each Linux file system has a lost+found directory. If the file system crashes, a file system check will be performed at next boot. Any corrupted files found will be placed in the lost+found directory, so you can attempt to recover as much data as possible.
</details>

### `/media`
<details><summary>Removable Media</summary>
  The /media directory contains subdirectories where removable media devices inserted into the computer are mounted. For example, when you insert a CD into your Linux system, a directory will automatically be created inside the /media directory. You can access the contents of the CD inside this directory.
</details>

### `/mnt`
<details><summary>Temporary Mount Points</summary>
  Historically speaking, the /mnt directory is where system administrators mounted temporary file systems while using them. For example, if you're mounting a Windows partition to perform some file recovery operations, you might mount it at /mnt/windows. However, you can mount other file systems anywhere on the system.
</details>

### `/opt`
<details><summary>Optional Packages</summary>
  The /opt directory contains subdirectories for optional software packages. It's commonly used by proprietary software that doesn't obey the standard file system hierarchy. For example, a proprietary program might dump its files in /opt/application when you install it.
</details>

### `/proc`
<details><summary>Kernel and Process Files</summary>
  The /proc directory similar to the /dev directory because it doesn't contain standard files. It contains special files that represent system and process information.

  ![grafik](https://github.com/user-attachments/assets/a7ac1797-943d-449d-b15a-f682806e6225)
</details>

### `/root`
<details><summary>Root Home Directory</summary>
  The /root directory is the home directory of the root user. Instead of being located at /home/root, it's located at /root. This is distinct from /, which is the system root directory.
</details>

### `/run`
<details><summary>Application State Files</summary>
  The /run directory gives applications a standard place to store transient files they require like sockets and process IDs. These files can't be stored in /tmp because files in /tmp may be deleted.
</details>

### `/sbin`
<details><summary>System Administration Binaries</summary>
  The /sbin directory is similar to the /bin directory. It contains essential binaries that are generally intended to be run by the root user for system administration.

  ![grafik](https://github.com/user-attachments/assets/e2c78c43-c219-421c-9bff-6c9a3396ea8c)
</details>

### `/snap`
<details><summary>Storage for Snap Packages</summary>
  Another directory that isn't part of the FHS but is common to see these days is /snap. It holds installed Snap packages and other files associated with Snap. Ubuntu now uses Snaps by default, but if you're using a different distro that doesn't, you won't see this directory.
</details>

### `/srv`
<details><summary>Service Data</summary>
  The /srv directory contains "data for services provided by the system." If you were using the Apache HTTP server to serve a website, you'd likely store your website's files in a directory inside the /srv directory.
</details>

### `/tmp`
<details><summary>Temporary Files</summary>
  Applications store temporary files in the /tmp directory. These files are generally deleted whenever your system is restarted and may be deleted at any time by utilities such as systemd-tmpfiles.
</details>

### `/usr`
<details><summary>User Binaries & Read-Only Data</summary>
  The /usr directory contains applications and files used by users, as opposed to applications and files used by the system. For example, non-essential applications are located inside the /usr/bin directory instead of the /bin directory and non-essential system administration binaries are located in the /usr/sbin directory instead of the /sbin directory. Libraries for each are located inside the /usr/lib directory. The /usr directory also contains other directories. For example, architecture-independent files like graphics are located in /usr/share.

  The /usr/local directory is where locally compiled applications install to by default. This prevents them from mucking up the rest of the system.

  ![grafik](https://github.com/user-attachments/assets/bd8f2ad2-a051-4e95-b0e1-8b9855be9b16)
</details>

### `/var`
<details><summary>Variable Data Files</summary>
  The /var directory is the writable counterpart to the /usr directory, which must be read-only in normal operation. Log files and everything else that would normally be written to /usr during normal operation are written to the /var directory. For example, you'll find log files in /var/log.


</details>
