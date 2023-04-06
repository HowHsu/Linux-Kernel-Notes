# User && Group && Permission

## 1. UID

UID stands for user id, which is used by Linux system to distinguish different users
in multi-user environment. There are three UIDs in Linux: ruid, euid and suid.
The first impartant thing about it is: UID is a concept for process.

### ruid: real user id

ruid is set when a user logins in to a shell, then this shell process and all its
descendant processes all have ruid = logined in user.
**And the ruid doesn't change through all the time.**

### euid: effective user id

euid is the current uid of a process, you may be confused about this because "shouldn't
euid always equals ruid?".
In most cases, the answer is true. But euid can change if suid is there. The euid is the UID used for resources access privilege check.

### suid: saved user id

Here, you must distinguish it with SUID(Setup-UID bit) for a binary file. SUID is a bit which can be set up for a binary file.
When a binary file with SUID bit run by a process A, it runs with `egid = the file's uid` not `egid = A's euid`.

Ok, the above statement is almost true but actually not accurate.
Let's go through an example to get a clear sense of `suid` and `SUID` bit.

```
An example of such program is "passwd". If you list it in full, you will see that it has Set-UID bit and the owner is "root". When a normal user, say "ananta", runs "passwd", passwd starts with:

Real-UID = ananta
Effective-UID = ananta
Saved-UID = root

The the program calls a system call "seteuid( 0 )" and since SUID=0, the call will succeed and the UIDs will be:

Real-UID = ananta
Effective-UID = root
Saved-UID = root

After that, "passwd" process will be able to access /etc/passwd and change password for user "ananta". Note that user "ananta" cannot write to /etc/passwd on it's own.
```

**Note one thing, setting a Set-UID bit on a executable file is not enough to make it run as privileged process. The program itself must make a system call to change the euid**

Ok, let's do a conclusion:
- the euid exists because there is suid which leads to uid changing during a process runs. 
- the suid exists because we want some unprivileged users to access some high privilege resources in a secure way.(by executing some safe binary file)

More detail of SUID's design and example, please see: [SUID](./virtiofs/virtiofsd.md#review-some-useful-knowledge)

### setresuid/setreuid/seteuid
TODO

## 2. GID

### rgid, egid, sgid
Similar like the UID section.

### primary group and secondary/supplemental group
In this section let's focus on groups. There is two sorts of groups:

- Primary group – Specifies a group that the operating system assigns to files that are created by the user. Each user must belong to a primary group.

- Secondary groups – Specifies one or more groups to which a user also belongs. Users can belong to up to 15 secondary groups.

**A user must belong to a primary group**
**A user can have supplemental/secondary groups to gain rights to access resources owned by users from other groups.**
For example, user1's primary group is group1, and user1 has supplemental groups group2, user2's primary group is group2.
file2's user/owner is user2, group is group2. **Then user1 has
the group2 rights when accessing file2.**

## 3. SUID bit && SGID bit && Sticky bit

Here `set-user-ID` and `set-group-ID` are also called `setuid` and `setgid`. This article explains
these two bits quite clearly: [setuid/setgid](https://www.cbtnuggets.com/blog/technology/system-admin/linux-file-permissions-understanding-setuid-setgid-and-the-sticky-bit)

In short:
- setuid: a bit that makes an executable run with the privileges of the owner of the file

- setgid:
    - if it's on a binary file, that makes it run with the privileges of the group of the file
    - if it's on a directory, that makes new files under this directory has same group info as that dir.

- sticky bit: a bit set on directory that allows users can only delete/move/rename their own files/dirs in this directory.

### a brief table of `SUID`, `SGID` and `Sticky` bits

```
		                binary file					                directory
SUID		the process' user = the file's user		                no meaning
SGID		the process' group = the file's group		all new files created by any user in this dir has the same group with the dir's.
Sticky		no meaning					            a user can only del/move/rename its own files/dirs in this dir(users' created dirs in this dir don't inherit the sticky bit)
```

### How to know if `SUID`/`SGID`/`Sticky` bit is set?

Just `ls -l` a file, you can see something like `-rwxrwxrwx` for the file's permission.
If `SUID` bit is set: `rwsrwxrwx`
If `SGID` bit is set: `rwxrwsrwx`
If `Sticky` bit is set: `rwxrwxrwt`

## 4. Permission bits mean for a file

There are three groups: `owner, group, other`. And for each object, there are three
permissions: `read, write, execute`, aka, `r, w, x`.

	- For a file: 
		- `r`: you can read the file
		- `w`: you can write the file(not include deleting the file)
		- `x`: you can execute the file

	- For a directory:
		- `r`: you can read the directory structure
		- `w`: you can modify the directory structure(add/remove files)
		- `x`: you can enter this directory

**one thing to notice here is the permission to remove/delete a file is on its parent directory
not itself**
