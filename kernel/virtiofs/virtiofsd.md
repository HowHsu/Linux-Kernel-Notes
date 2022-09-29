# virtiofsd

This article is about the backend of virtiofs: virtiofsd. And here I'll
pick up the rust version as an instance, while I'm also new to this languagethough. I'm going to talk about the big picture of virtiofsd as well as
diving deeply into the code to make you have a full sense of it.

### framework

Let's first look at a graph which shows the relationship of the main classes
 of virtiofsd.

![](./images/virtiofsd.drawio.svg)



### cache policy

- guest kernel page cache: G
- host kern el page cache: H

O_DIRECT: vfs flag, users can set it when opening a file
FOPEN_DIRECT_IO: fuse flag, users can set it when starting virtiofsd by `--cache`

- `--allow-direct-io true`
This means we respect the original IO flag(vfs flag) of how we leverage the host
page cache
```
vfs\fuse    FOPEN_DIRECT_IO(--cache=never)	      !FOPEN_DIRECT_IO
O_DIRECT        !G   !H                                    !G  !H
!O_DIRECT	!G    H                                     G   H
```

- `--allow-direct-io false (default)`
This means we see all the IO requests from guest as buffered IO requests on host
```
vfs\fuse    FOPEN_DIRECT_IO(--cache=never)	      !FOPEN_DIRECT_IO
O_DIRECT        !G    H                                     G   H
!O_DIRECT	!G    H                                     G   H
```
