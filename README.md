# Freebsd vs Linux: VFS

This document analyzes the Virtual File System of FreeBSD and Linux. It examines the differences between FreeBSD vnode locking and Linux inode locking, traces the execution path of the open() system call in the FreeBSD kernel, identifies the VFS layer functions involved, and explains where vnode allocation occurs.

# FreeBSD vnode locking vs Linux inode locking

## FreeBSD vnode locking
### vnode structure
A vnode in FreeBSD is an internal data structure used to represent an active file, directory, device node or sockets. It is represented by the ***struct vnode*** and is uniquely allocated for each active filesystem entitiy. The vnode structure is defined in the `sys/sys/vnode.h` header file and contains a lot of entries so the following strcuture will only contain entries relevant to the discussion.

```text
struct vnode {
    const struct vop_vector *v_op;
    void *v_data;

    struct lock v_lock;
    struct lock *v_vnlock;
    struct mtx v_interlock;

    u_int v_usecount;
    u_int v_holdcnt;
};
```
Important entries here are 
***v_data*** which points to the object associated with the vnode and ***v_op*** which points to the vnode operation vector which is used to dispatch filesystem operations.

### locking mechanism

vnode locking in FreeBSD is used to serialize access to filesystem objects represented by ***struct vnode***. This prevents conditions where multiple kernel threads attempt to perform operations on the same file or directory concurrently.

The vnode lock is referenced through the **v_vnlock** field of ***struct vnode***.

Vnode locks are typically acquired using int `vn_lock`(***struct vnode *vp, int flags***) where ***vp*** is the vnode being locked and ***flags*** specifies the lock type. This function is defined in `sys/kern/vfs_vnops.c`.
Before invoking filesystem operations such as lookup, create, read, write, etc the VFS layer typically acquires the vnode lock through vn_lock(). The lock is released after the operation completes.

## Linux inode locking
### inode structure
The Linux VFS represents filesystem metadata using ***struct inode***, defined in `include/linux/fs.h`. The full structure contains many fields and configuration-dependent members. The following structure only contains fields relevant to VFS operations and locking:
```text
struct inode {
    umode_t i_mode;
    kuid_t i_uid;
    kgid_t i_gid;
    loff_t i_size;
    const struct inode_operations *i_op;
    const struct file_operations *i_fop;
    struct rw_semaphore i_rwsem;
    atomic_t i_count;
};
```
Here **i_mode** is the file types and permissions, **i_uid** and **i_gid** is the file ownership, **i_size** is the size of the file and so on.
unlike FreeBSD's vnode the Linux inode directly stores file metadata such as ownership, permissions, timestamps, and file size. In FreeBSD, these metadata fields are typically stored in structures referenced through the vnode's v_data pointer.

### inode locking mechanism

Linux synchronizes access to filesystem objects using the inode's i_rwsem field, which is a read-write semaphore defined in struct inode. It protects inode metadata and serializes filesystem operations that modify a file or directory.

The VFS acquires and releases the inode lock using helper functions such as:

inode_lock(struct inode *inode);
inode_unlock(struct inode *inode);

inode_lock_shared(struct inode *inode);
inode_unlock_shared(struct inode *inode);

inode_lock() acquires the inode's i_rwsem in exclusive mode, while inode_lock_shared() acquires it in shared mode. These functions are commonly used before performing operations such as file creation, deletion, rename, or metadata updates to ensure that only one thread modifies the inode at a time.

Unlike FreeBSD, which performs synchronization through the vnode (v_vnlock), Linux centers its VFS synchronization around the inode because the inode directly stores the file's metadata. This design allows concurrent read access while ensuring exclusive access for operations that modify the inode or directory contents.

## Comparison: FreeBSD vnode vs Linux inode

| Feature | FreeBSD Vnode | Linux Inode |
|---------|---------------|-------------|
| Locking mechanism | Uses `v_vnlock`, typically acquired through `vn_lock()`. | Uses `i_rwsem`, typically acquired through `inode_lock()`. |
| Filesystem abstraction | Provides a filesystem-independent interface for all supported filesystems. | Uses `struct inode` together with `struct dentry` to represent filesystem objects. |
| Operation dispatch | Uses the vnode operation vector (`v_op`) and the VOP interface. | Uses `inode_operations` (`i_op`) and `file_operations` (`i_fop`). |
| Metadata fields | Does not directly store file metadata such as file size, ownership, or timestamps. | Directly stores metadata such as `i_size`, `i_uid`, `i_gid`, and timestamps. |
| Filesystem-specific data | Stores a pointer (`v_data`) to filesystem-specific objects such as a UFS inode, ZFS znode, or NFS node. | The inode itself is the filesystem-specific metadata object. |

# Tracing open("/tmp/test.txt", O_RDWR | O_CREAT, 0644) in FreeBSD

## `open()`

***open()*** is a libc fucntion used for opening or creating files. In this example, the pathname "/tmp/test.txt" identifies the file to be opened. The O_RDWR flag requests that the file be opened for both reading and writing, while the O_CREAT flag instructs the kernel to create the file if it does not already exist. The mode 0644 specifies the permissions assigned to the file if it is created.

When the call enters the FreeBSD kernel, control passes to the system call entry point sys_openat() in ***sys/kern/vfs_syscalls.c***, which begins the VFS pathname lookup and file opening process. If the operation completes successfully, open() returns a file descriptor that the process uses for subsequent operations such as read(), write(), and close().

## `sys_openat()`

Control enters the FreeBSD kernel through the ***sys_openat()*** system call entry point, which is implemented in ***sys/kern/vfs_syscalls.c***. Although the user invokes ***open()***, FreeBSD implements file opening through ***sys_openat()***, with ***open()*** using the current working directory as the starting point for pathname resolution.

The ***sys_openat()*** function validates the system call arguments and forwards the request to `kern_openat()`, which performs the kernel-level processing required to open or create the file. This includes pathname resolution, permission checking, vnode lookup, and file creation if the `O_CREAT` flag is specified.

## `kern_openat()`

The `kern_openat()` function is implemented in `sys/kern/vfs_syscalls.c` and acts as a kernel wrapper for the file-opening operation. It receives the validated arguments from `sys_openat()` and forwards them to `openatfp()`, which performs the actual file opening process.

## `openatfp()`

The openatfp() function performs the core processing required to open or create a file. It is called by `kern_openat()` and is implemented in `sys/kern/vfs_syscalls.c`.

The function performs the following operations:
### Validates the open flags
Calls `openflags()` to validate and normalize the user-supplied open flags.
### Allocates a file structure
Calls `falloc_noinstall()` to allocate a struct file that will eventually be associated with a file descriptor.
### Initializes pathname lookup
Uses `NDINIT_ATRIGHTS()` to initialize a struct nameidata, which stores the information required for pathname resolution.
### Invokes the VFS layer
Calls `vn_open_cred()`, passing the initialized nameidata structure, open flags, file creation mode, user credentials, and the allocated file structure. This function performs pathname resolution, vnode lookup, permission checking, and file creation if O_CREAT is specified.
### Initializes the opened file
On success, `finit_open()` associates the opened vnode with the allocated file structure.
### Releases the vnode lock
Calls `VOP_UNLOCK()` after the vnode has been successfully opened.
### Installs the file descriptor
Calls `finstall_refed()` to install the file into the calling process's file descriptor table and returns the newly allocated file descriptor to user space.

## `vn_open_cred()`

The `vn_open_cred()` function, implemented in `sys/kern/vfs_vnops.c`, performs the core VFS operations required to open or create a file. It receives the pathname information prepared by openatfp() and determines whether the file already exists or must be created.

### Initializes pathname lookup
Because the ***O_CREAT flag*** is specified, `vn_open_cred()` configures the pathname lookup for a create operation.
### Resolves the pathname
Calls `namei()` to resolve the pathname `/tmp/test.txt`. This function traverses the directory hierarchy and determines whether the target file already exists.
### Creates the file if necessary
If `namei()` does not find the file, `VOP_CREATE()` is called to create a new vnode representing the file.
### Opens the vnode
Calls `vn_open_vnode()` to perform permission checks and complete the open operation on the vnode.
### Returns the opened vnode
On success, the vnode is returned to `openatfp()`, where it is associated with a struct file and eventually installed into the process's file descriptor table.

## `namei()`
The `namei()` function, implemented in `sys/kern/vfs_lookup.c` and receives the initialized nameidata structure from `vn_open_cred()`. It prepares the pathname for lookup, checks whether the requested path is present in the VFS name cache, and then repeatedly invokes `vfs_lookup()` to traverse each component of the pathname.

If the lookup is successful, the vnode corresponding to test.txt is returned in ***ndp->ni_vp***. If the file does not exist and the operation is a create request ***O_CREAT***, namei() returns the parent directory vnode in ***ndp->ni_dvp*** while leaving ***ndp->ni_vp*** as NULL. This allows vn_open_cred() to invoke VOP_CREATE() to create the new file.
## `vfs_lookup()`
The `vfs_lookup() function` implemented in `sys/kern/vfs_lookup.c` performs the actual traversal of the pathname. It processes each component of the path one at a time until the target file is located. For each pathname component `vfs_lookup()` invokes the filesystem's `VOP_LOOKUP()` operation. When the final component (test.txt) is reached, `vfs_lookup()` returns the vnode if the file already exists. If the file does not exist and the operation was initiated with O_CREAT, control returns to `vn_open_cred()`, which invokes `VOP_CREATE()` to create a new vnode for the file.
## `VOP_CREATE()`
If `namei()` determines that the target file does not exist and the ***O_CREAT flag*** is specified, `vn_open_cred()` invokes the `VOP_CREATE()` vnode operation to create a new regular file.

`VOP_CREATE()` is part of the FreeBSD vnode operations interface. Rather than implementing file creation itself, it dispatches the request to the underlying filesystem's create operation. The filesystem allocates the required on-disk metadata, initializes a new vnode representing the file, and returns it to the VFS.

On successful completion, the newly created vnode is stored in ndp->ni_vp and returned to vn_open_cred(), which continues the file opening process by invoking vn_open_vnode().
### `vn_open_vnode()`
After the pathname has been resolved and the vnode has been obtained, `vn_open_cred()` invokes `vn_open_vnode()` to complete the open operation.

The `vn_open_vnode()` function performs the necessary permission checks, verifies that the requested access mode is permitted, and invokes the filesystem's `VOP_OPEN()` operation. `VOP_OPEN()` allows the underlying filesystem to perform any filesystem-specific processing required before the file can be accessed.

If the operation succeeds, control returns to `vn_open_cred()`, which then returns to `openatfp()`. The opened vnode is subsequently associated with a struct file, installed into the process's file descriptor table, and the corresponding file descriptor is returned to the calling process.
## Vnode Allocation
Vnode allocation is performed by the underlying filesystem. When `VOP_CREATE()` creates a new file, the filesystem allocates and initializes a new vnode representing the file. This allocation is typically performed using `getnewvnode()`, which is implemented in `sys/kern/vfs_subr.c`.

After the vnode has been initialized, it is returned to the VFS through `VOP_CREATE()` and stored in `ndp->ni_vp`. The VFS then continues the open operation by invoking `vn_open_vnode()`.
