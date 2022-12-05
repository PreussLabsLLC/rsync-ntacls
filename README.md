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
luck with that approach. And am having limited success. At this point (Nov 2022)
local copies from NT to NT work, remote copies from NT to NT via ssh work after excluding
rsync's (POSIX) 'chmod', which altered the NT ACLS.

This is based on the ntstreams.c implementation of BackupAssist's rsync,
published as patches to a 2009 version of Rsync a while back under the GPL.

## Details:

### Building this version

- Setup a suitable cygwin environment with C development tools (I used cygwin64)
- you may need to install certain libraries (zlib, xxhash)
- just run execute `make` in the rsync-3.2.6 folder (no ./configure)

### Running examples

- local copy:\
`./rsync -Xav --inplace /cygdrive/c/Users/Paul/Contacts /tmp/copy`

- remote copy to Windows:\
`./rsync -Xave ssh /cygdrive/c/Users/Paul/Contacts Administrator@192.168.37.226:/tmp/copy`

- remote copy to Mac:\
`./rsync -Xave ssh /cygdrive/c/Users/Paul/Contacts root@minimac:/tmp/copy`

### Notes:

- local and remote Windows systems need this version of rsync
- the `X` (xattr) option is required to carry NT ACLS
- on Mac/*nix, an rsync version with xattr support will be needed
- on Mac/*nix, xattr `rsync.security.ntacl` will hold the NT ACLS

### Implementation details

.. to follow



