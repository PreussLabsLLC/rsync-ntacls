# rsync-ntacls

This is work-in-progress on teaching rsync to copy Windows NT ACLS as security.NTACLS
extended attribute.\
I'm hoping for Samba vfs_acl_xattr compatibilty at some point.

Rsync has been available on Windows platforms for ages, but blissfully ignores Windows NT's
security attributes.

Microsoft has provided its RoboCopy tool to transfer files locally or remotely 
while preserving NTFS security attributes (given the right options). But it 
relies on the SMB protocol for remote access (chatty and sometimes hampered by propagation
delays on remote links) and does not support delta transfers (whole files are copied).

This patch enables rsync on windows to transfer NTFS security attributes while offering
rsync's delta transfers.\
This requires the cygwin posix-for-windows environment.

## Background:

### Everything You Always Wanted to Know About SIDs * But Were Afraid to Ask: [^1]
- https://cygwin.com/cygwin-ug-net/ntsec.html#ntsec-common

### From the Horse's Mouth:
- https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-security-identifiers
- https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/robocopy

Windows NT's ACLS are accessed by reading a magic BACKUP_SECURITY_DATA stream
(one of possibly many streams, but every file & folder on ntfs has exactly one of these)
utilizing the privileged (needs more magic process setup) BackupRead function.

- https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-backupread
- https://learn.microsoft.com/en-us/windows/win32/api/winbase/ns-winbase-win32_stream_id
- https://www.samba.org/samba/docs/current/man-html/vfs_acl_xattr.8.html
- https://www.backupassist.com/support/en/backupassist/manage/rsync-options.htm
- https://www.backupassist.com/rsync/rsync_3.0.7-ba_6.1.1.patch
- https://noah.org/ssh/cygwin-sshd.html

## Implementation:

One may be tempted to try mapping NT ACLS into Posix ACLS (which rsync supports);
but that mapping seems never to work right as NT and the -nix world can't agree
on ordering ACLS (amongst other things).

Since Samba thought about storing the NT ACLS as a security.NTACLS xattr, I tried my
luck with that approach. At this point (Dec 2022) local copies from NT to NT work, 
remote copies from NT to NT via ssh work also ~~(after excluding rsync's (POSIX) 'chmod',
which altered the NT ACLS)~~. NT to MAC has also been tested with `xattr -l filename` displaying
the NTACLS as hexdump.

The WIN32 magic is based on the ntstreams.c implementation of BackupAssist's rsync,
published as patches to a 2009 version of Rsync a while back under the GPL.

## Details:

### Running Examples:

- local copy:\
`rsync -Xav /cygdrive/c/Users/Paul/Contacts /tmp/copy`

- remote copy to Windows:\
`rsync -Xave ssh /cygdrive/c/Users/Paul/Contacts Administrator@bigbob:/tmp/copy`

- remote copy to Mac:\
`rsync -Xave ssh /cygdrive/c/Users/Paul/Contacts root@minimac:/tmp/copy`

### Notes:

- local and remote Windows systems need this version of rsync
- the `X` (xattr) option is required to carry NT ACLS
- other possibly existing named NTFS streams are skipped
- the rsync processes need to be started from an elevated (Administrators) session\
in order to utilize the WIN32 BackupRead/Write functions
- for rsync-over-ssh the cygwin sshd needs to be configured, ideally with key-based authentication
- on Mac/*nix, an rsync version with xattr support will be needed
- on Mac/*nix, xattr `rsync.security.ntacl` will hold the NT ACLS
- restore from Mac/*nix will restore the NT ACLS (as NT ACLS, not xattr)
- debugging info is currently a bit verbose

### Implementation Details:

- added one module, ntacsl.c, to rsync's Makefile.in
- modified three ~~amigos~~ functions to list, get, and set xattrs in lib/sysxattrs.c\
instead of acting on xattrs, they read and write NTFS ACLS contained in the BACKUP_SECURITY_DATA stream.\
Rsync is managing/comparing/transferring the ACLS as xattrs automatically.
- chown on restore altered NTFS security attributes, thus disabled in rsync.c
- Rysnc on windows needs to think it's running as 'root', thus a hardcoded UID 0 in rsync.h
- minor adjustments in options.c, version.h, and usage.c

### Build Instructions:

- Setup a suitable cygwin environment with C development tools (I used cygwin64)
- You may need to install certain libraries (zlib, xxhash)\
Kindly see https://github.com/WayneD/rsync/blob/master/INSTALL.md for details.
- Download the rsync 3.2.6 release from here:\
https://download.samba.org/pub/rsync/src/rsync-3.2.6.tar.gz \
.. and unpack into a local folder named rsync-3.2.6.

- Download the posted patch to a local location, say, /tmp.
.. and try the patch from the folder above rsync-3.2.6 with \
`patch --dry-run -p0 < /tmp/rsync-3.2.6-ntacls.patch` \
if things look OK, apply with \
`patch -p0 < /tmp/rsync-3.2.6-ntacls.patch`

- Configure and build the patched rsync version: \
`./configure`\
`make`\
`./rsync -V`\
You should be greeted by something like this:
```
rsync  version 3.2.6pl protocol version 31
Copyright (C) 1996-2022 by Andrew Tridgell, Wayne Davison, and others.
Web site: https://rsync.samba.org/
Capabilities:
    64-bit files, 64-bit inums, 64-bit timestamps, 64-bit long ints,
    socketpairs, symlinks, symtimes, hardlinks, no hardlink-specials,
    hardlink-symlinks, IPv6, atimes, batchfiles, inplace, append, ACLs,
    xattrs, optional secluded-args, no iconv, prealloc, stop-at, crtimes
Optimizations:
    no SIMD-roll, no asm-roll, openssl-crypto, no asm-MD5
Checksum list:
    xxh128 xxh3 xxh64 (xxhash) md5 md4 none
Compress list:
    zstd lz4 zlibx zlib none

@(#) ntacls.c pl 1.0 Dec  8 2022 10:40:34 https://github.com/PreussLabsLLC/rsync-ntacls

rsync comes with ABSOLUTELY NO WARRANTY.  This is free software, and you
are welcome to redistribute it under certain conditions.  See the GNU
General Public Licence for details.

```
- You may want to rename existing versions to preserve them as fall-back
- Copy the just built rsync version to your sender and receiver Windows systems as\
`/usr/local/bin/rsync` \
with a symlink to \
`/usr/bin/rsync`\
for remote ssh on Windows.

[^1]: https://en.wikipedia.org/wiki/Everything_You_Always_Wanted_to_Know_About_Sex*_(*But_Were_Afraid_to_Ask)_(film)
