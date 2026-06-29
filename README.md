# Freebsd vs Linux: VFS

This document analyzes the Virtual File System of FreeBSD and Linux. It examines the differences between FreeBSD vnode locking and Linux inode locking, traces the execution path of the open() system call in the FreeBSD kernel, identifies the VFS layer functions involved, and explains where vnode allocation occurs.

# FreeBSD vnode locking vs Linux inode locking

## FreeBSD vnode locking
### vnode structure
A vnode in FreeBSD is an internal data structure used to represent an active file, directory, device node or sockets. It is represented by the ***struct vnode*** and is uniquely allocated for each active filesystem entitiy. The vnode structure is defined in the ***sys/sys/vnode.h*** header file and contains a lot of entries so the following strcuture will only contain entries relevant to the discussion.

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

Vnode locks are typically acquired using int **vn_lock**(***struct vnode *vp, int flags***) where ***vp*** is the vnode being locked and ***flags*** specifies the lock type. This function is defined in ***sys/kern/vfs_vnops.c***.
Before invoking filesystem operations such as lookup, create, read, write, etc the VFS layer typically acquires the vnode lock through vn_lock(). The lock is released after the operation completes.

## Linux inode locking
### inode structure
The Linux VFS represents filesystem metadata using ***struct inode***, defined in ***include/linux/fs.h***. The full structure contains many fields and configuration-dependent members. The following structure only contains fields relevant to VFS operations and locking:
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





