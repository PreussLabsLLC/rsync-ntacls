# rsync-ntacls

This is work-in-progress on teaching rsync to copy Windows NT ACLS as security.NTACLS
extended attribute. I'm hoping for Samba vfs_acl_xattr compatibilty at some point.

This requires the cygwin posix-for-windows environment.

Rsync has been available on Windows platforms for ages, but blissfully ignores Windows NT's
security attributes.

## Background:

### Intro to Windows permissions & security:
- https://cygwin.com/cygwin-ug-net/ntsec.html#ntsec-common

### From the horse's mouth:
- https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-security-identifiers

Windows NT's ACLS are accessed by reading a magic BACKUP_SECURITY_DATA stream
(one of possibly many streams, but every file & folder on ntfs has exactly one of these)
utilizing the privileged (needs more magic process setup) BackupRead function.

- https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-backupread
- https://learn.microsoft.com/en-us/windows/win32/api/winbase/ns-winbase-win32_stream_id

- https://www.samba.org/samba/docs/current/man-html/vfs_acl_xattr.8.html

- https://www.backupassist.com/support/en/backupassist/manage/rsync-options.htm

## Implementation:

One may be tempted to try mapping NT ACLS into Posix ACLS (which rsync supports);
but that mapping seems never to work right as NT and the -nix world can't agree
on ordering ACLS (amongst other things).

Since Samba thought about storing the NT ACLS as a security.NTACLS xattr, I tried my
luck with that approach. At this point (Dec 2022) local copies from NT to NT work, 
remote copies from NT to NT via ssh work also ~~(after excluding rsync's (POSIX) 'chmod',
which altered the NT ACLS)~~. NT to MAC has also been tested with `xattr filename` displaying
the NTACLS as hexdump. Testing restore from MAC pending..

This is based on the ntstreams.c implementation of BackupAssist's rsync,
published as patches to a 2009 version of Rsync a while back under the GPL.

## Details:

### Running examples

- local copy:\
`./rsync -Xav --inplace /cygdrive/c/Users/Paul/Contacts /tmp/copy`

- remote copy to Windows:\
`./rsync -Xave ssh /cygdrive/c/Users/Paul/Contacts Administrator@bigbob:/tmp/copy`

- remote copy to Mac:\
`./rsync -Xave ssh /cygdrive/c/Users/Paul/Contacts root@minimac:/tmp/copy`

### Notes:

- local and remote Windows systems need this version of rsync
- the `X` (xattr) option is required to carry NT ACLS
- on Mac/*nix, an rsync version with xattr support will be needed
- on Mac/*nix, xattr `rsync.security.ntacl` will hold the NT ACLS
- debugging info is currently a bit verbose

### Implementation details

.. to follow

### Build instructions

- Setup a suitable cygwin environment with C development tools (I used cygwin64)
- You may need to install certain libraries (zlib, xxhash)\
Kindly see https://github.com/WayneD/rsync/blob/master/INSTALL.md for details.
- Download the rsync 3.2.6 release from here:\
https://download.samba.org/pub/rsync/src/rsync-3.2.6.tar.gz \
.. and unpack into a local folder named rsync-3.2.6.

- Download the posted patch to a local location, say, /tmp.
.. and try the patch with \
`patch --dry-run -p0 < /tmp/rsync-3.2.6-ntacls.patch` \
if things look OK, apply with \
`patch -p0 < /tmp/rsync-3.2.6-ntacls.patch`

- Configure and build the patched rsync version: \
`./configure
make
./rsync -V`

(Sorry for the warnings, try to ignore them for now.)

- You may want to rename existing versions to preserve them as fall-back

- Copy the just built rsync version to your sender and receiver Windows systems as\
`/usr/local/bin/rsync` \
with a symlink to \
`/usr/bin/rsync`\
for remote ssh on Windows.



