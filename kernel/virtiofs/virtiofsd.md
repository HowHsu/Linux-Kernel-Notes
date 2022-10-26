# virtiofsd

This article is about the backend of virtiofs: virtiofsd. And here I'll
pick up the rust version as an instance, while I'm also new to this languagethough. I'm going to talk about the big picture of virtiofsd as well as
diving deeply into the code to make you have a full sense of it.

### framework

Let's first look at a graph which shows the relationship of the main classes
 of virtiofsd.

![](./images/virtiofsd.drawio.svg)



### cache policy

- guest kernel page cache: GCache
- host kern el page cache: HCache

O_DIRECT: vfs flag, users can set it when opening a file
FOPEN_DIRECT_IO: fuse flag, users can set it when starting virtiofsd by `--cache`

- `--allow-direct-io true`
This means we respect the original IO flag(vfs flag) of how we leverage the host
page cache
```
vfs\fuse    FOPEN_DIRECT_IO(--cache=never)	      !FOPEN_DIRECT_IO(--cache=auto/always)
O_DIRECT        !GCache   !HCache                     !GCache  !HCache
!O_DIRECT	!GCache    HCache                      GCache   HCache
```

- `--allow-direct-io false (default)`
This means we see all the IO requests from guest as buffered IO requests on host
```
vfs\fuse    FOPEN_DIRECT_IO(--cache=never)	      !FOPEN_DIRECT_IO(--cache=auto/always)
O_DIRECT        !GCache    HCache                     !GCache   HCache
!O_DIRECT	!GCache    HCache                      GCache   HCache
```
#### a more clear inclusion

for guest page cache:

`--cache=never`: we don't use guest page cache anyway!
`--cache=auto/always`: we use guest page cache for buffered IO

for host page cache:
`!O_DIRECT`: we use host page cache anyway!
`O_DIRECT`: we use host page cache if `--allow-direct-io=false`
