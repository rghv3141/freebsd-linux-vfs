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





