# Freebsd vs Linux: VFS

This document analyzes the Virtual File System of FreeBSD and Linux. It examines the differences between FreeBSD vnode locking and Linux inode locking, traces the execution path of the open() system call in the FreeBSD kernel, identifies the VFS layer functions involved, and explains where vnode allocation occurs.

# FreeBSD vnode locking vs Linux inode locking

## FreeBSD vnode locking
### vnode structure
A vnode in FreeBSD is an internal data structure used to represent an active file, directory, device node or sockets. It is represented by the ***struct vnode*** and is uniquely allocated for each active filesystem entitiy. The vnode structure is defined in the ***sys/sys/vnode.h*** header file.

```text
This structure only contains the relevant entries
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
