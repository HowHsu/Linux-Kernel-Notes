# User and Group

## UID

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

## GID

In this section let's focus on groups. There is two sorts of groups:

- Primary group – Specifies a group that the operating system assigns to files that are created by the user. Each user must belong to a primary group.

- Secondary groups – Specifies one or more groups to which a user also belongs. Users can belong to up to 15 secondary groups.

**A user must belong to a primary group**
**
