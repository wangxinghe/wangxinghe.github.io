---
layout: post
comments: true
title: "ServiceManager守护进程启动过程"
description: "ServiceManager守护进程启动过程"
category: AOSP
tags: [AOSP]
---


[TOC]

>基于Android 10.0的源码剖析, 站在Luoshengyang/Innost/Gityuan肩膀上.

## 0. 文件结构

    system/core/rootdir/init.rc
	
	frameworks/native/cmds/servicemanager/servicemanager.rc
	frameworks/native/cmds/servicemanager/binder.h
	frameworks/native/cmds/servicemanager/service_manager.c
	frameworks/native/cmds/servicemanager/binder.c
 
	kernel/msm-3.18/drivers/staging/android/binder.c
	kernel/msm-3.18/drivers/staging/android/binder_alloc.c
	kernel/msm-3.18/drivers/staging/android/uapi/binder.h
	kernel/msm-3.18/include/linux/fs.h

<!--more-->

## 1. 启动过程

### 1.1 servicemanager服务定义

[ -> frameworks/native/cmds/servicemanager/servicemanager.rc]
		
	// 定义一个service, 名称为servicemanager, 路径为/system/bin/servicemanager
	service servicemanager /system/bin/servicemanager
    class core animation
    user system
    group system readproc
    critical
    onrestart restart healthd
    onrestart restart zygote
    onrestart restart audioserver
    onrestart restart media
    onrestart restart surfaceflinger
    onrestart restart inputflinger
    onrestart restart drm
    onrestart restart cameraserver
    onrestart restart keystore
    onrestart restart gatekeeperd
    onrestart restart thermalservice
    writepid /dev/cpuset/system-background/tasks
    shutdown critical

手机Root后, 通过Android Studio的Device File Explorer可以看到/system/bin/servicemanager这个目录.

### 1.2 servicemanager服务启动入口

[ -> system/core/rootdir/init.rc]

	// 启动servicemanager服务
	start servicemanager

[ -> frameworks/native/cmds/servicemanager/service_manager.c]

	// 入口函数
	int main(int argc, char** argv)
	{
	    struct binder_state *bs;
	    char *driver;
	 
	    if (argc > 1) {
	        driver = argv[1];
	    } else {
	        driver = "/dev/binder";
	    }
	 
	    // 打开binder驱动, 申请128k内存
	    bs = binder_open(driver, 128*1024);
	    // 成为上下文管理者
	    binder_become_context_manager(bs)
	    // 进入无线循环, 处理client端发来的请求
	    binder_loop(bs, svcmgr_handler);
	 
	    return 0;
	}

### 1.3 binder_open
[ -> frameworks/native/cmds/servicemanager/binder.c]

	struct binder_state *binder_open(const char* driver, size_t mapsize)
	{
	    struct binder_state *bs;
	    struct binder_version vers;
	 
	    bs = malloc(sizeof(*bs));
	    // 通过系统调用, 打开binder设备驱动, 创建proc用来保存打开设备文件/dev/binder的进程的相关信息
	    bs->fd = open(driver, O_RDWR | O_CLOEXEC);
	    // 通过系统调用, 设置binder版本信息, 使用户空间和内核空间版本信息一致
	    ioctl(bs->fd, BINDER_VERSION, &vers)
	     
	    // 设置映射内存大小
	    bs->mapsize = mapsize;
	    // 通过系统调用, mmap内存映射, mmap内存必须是page的整数倍
	    bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);

	    return bs;
	}

根据【第2节】的static const struct file_operations binder_fops可知:        
- (1) open -> binder_open
- (2) ioctl -> binder_ioctl
- (3) mmap -> binder_mmap

#### 	1.3.1 open
[ -> kernel/msm-3.18/drivers/staging/android/binder.c]

	static int binder_open(struct inode *nodp, struct file *filp)
	{
	    // 创建struct binder_proc *proc对象
	    // 用来保存打开设备文件/dev/binder的进程的信息(如tsk, context, pid)
	    struct binder_proc *proc;
	    struct binder_device *binder_dev;
	 
	    proc = kzalloc(sizeof(*proc), GFP_KERNEL);
	    spin_lock_init(&proc->inner_lock);
	    spin_lock_init(&proc->outer_lock);
	    get_task_struct(current->group_leader);
	    proc->tsk = current->group_leader;
	    mutex_init(&proc->files_lock);
	    INIT_LIST_HEAD(&proc->todo);
	    if (binder_supported_policy(current->policy)) {
	        proc->default_priority.sched_policy = current->policy;
	        proc->default_priority.prio = current->normal_prio;
	    } else {
	        proc->default_priority.sched_policy = SCHED_NORMAL;
	        proc->default_priority.prio = NICE_TO_PRIO(0);
	    }
	 
		// 获取binder_device对象
	    binder_dev = container_of(filp->private_data, struct binder_device, miscdev);
	    proc->context = &binder_dev->context;
	    binder_alloc_init(&proc->alloc);
	 
	    binder_stats_created(BINDER_STAT_PROC);
	    proc->pid = current->group_leader->pid;
	    INIT_LIST_HEAD(&proc->delivered_death);
	    INIT_LIST_HEAD(&proc->waiting_threads);
	    // 将proc保存到/dev/binder文件对应的结构体的struct file->private_data
	    filp->private_data = proc;
	 
	    // 将proc->proc_node进程节点添加到链表binder_procs
	    mutex_lock(&binder_procs_lock);
	    hlist_add_head(&proc->proc_node, &binder_procs);
	    mutex_unlock(&binder_procs_lock);
	 
	    return 0;
	}
#### 	1.3.2 ioctl
[ -> kernel/msm-3.18/drivers/staging/android/binder.c]

	static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
	{
	    int ret;
	    struct binder_proc *proc = filp->private_data;
	    struct binder_thread *thread;
	    unsigned int size = _IOC_SIZE(cmd);
	    void __user *ubuf = (void __user *)arg;
	 
	    binder_selftest_alloc(&proc->alloc);
	    // 进入休眠状态, 直到中断唤醒
	    ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
	    if (ret)
	        goto err_unlocked;
	    // 从proc中获取binder_thread, 若没有则创建新线程
	    thread = binder_get_thread(proc);
	 
	    switch (cmd) {
	    case BINDER_WRITE_READ: // binder读写操作
	        ret = binder_ioctl_write_read(filp, cmd, arg, thread);
	        break;
	    case BINDER_SET_MAX_THREADS: { // 设置binder最大线程数
	        int max_threads;
	        copy_from_user(&max_threads, ubuf, sizeof(max_threads));
	         
	        binder_inner_proc_lock(proc);
	        proc->max_threads = max_threads;
	        binder_inner_proc_unlock(proc);
	        break;
	    }
	    case BINDER_SET_CONTEXT_MGR: // 设置binder上下文管理者, 即servicemanager为守护进程
	        ret = binder_ioctl_set_ctx_mgr(filp);
	        break;
	    case BINDER_THREAD_EXIT: // binder线程退出
	        binder_thread_release(proc, thread);
	        thread = NULL;
	        break;
	    case BINDER_VERSION: { // 设置binder版本号
	        struct binder_version __user *ver = ubuf;
	 
	        if (size != sizeof(struct binder_version)) {
	            ret = -EINVAL;
	            goto err;
	        }
	        put_user(BINDER_CURRENT_PROTOCOL_VERSION, &ver->protocol_version);
	        break;
	    }
	    case BINDER_GET_NODE_INFO_FOR_REF: {
	        struct binder_node_info_for_ref info;
	        copy_from_user(&info, ubuf, sizeof(info));
	        ret = binder_ioctl_get_node_info_for_ref(proc, &info);
	        copy_to_user(ubuf, &info, sizeof(info));       
	        break;
	    }
	    case BINDER_GET_NODE_DEBUG_INFO: {
	        struct binder_node_debug_info info;
	        copy_from_user(&info, ubuf, sizeof(info));
	        ret = binder_ioctl_get_node_debug_info(proc, &info);
	        copy_to_user(ubuf, &info, sizeof(info));
	        break;
	    }
	    default:
	        ret = -EINVAL;
	        goto err;
	    }
	    ret = 0;
	err:
	err_unlocked:
	    ...
	    return ret;
	}

binder_ioctl涉及到的重要方法有:        
- (1) binder_get_thread
- (2) binder_ioctl_write_read
- (3) binder_thread_write
- (4) binder_thread_read
- (5) binder_transaction
- (6) copy_from_user
- (7) copy_to_user

关于binder_ioctl这块Android Binder源码调用的地方很多, 将会在总结talkWithDriver时详细分析.

#### 	1.3.3 mmap
[ -> kernel/msm-3.18/drivers/staging/android/binder.c]

	static int binder_mmap(struct file *filp, struct vm_area_struct *vma)
	{
		// struct vm_area_struct *vma 进程虚拟地址空间
	    int ret;
	    struct binder_proc *proc = filp->private_data;
	    const char *failure_string;
	 
	    if (proc->tsk != current->group_leader)
	        return -EINVAL;
	 
	    // 保证映射内存大小不超过4M
	    if ((vma->vm_end - vma->vm_start) > SZ_4M)
	        vma->vm_end = vma->vm_start + SZ_4M;
	 
	    vma->vm_flags = (vma->vm_flags | VM_DONTCOPY) & ~VM_MAYWRITE;
	    vma->vm_ops = &binder_vm_ops;
	    vma->vm_private_data = proc;
	    // 将进程虚拟地址空间和内核虚拟地址空间, 映射到同一个物理地址空间
	    ret = binder_alloc_mmap_handler(&proc->alloc, vma);

	    mutex_lock(&proc->files_lock);
	    proc->files = get_files_struct(current);
	    mutex_unlock(&proc->files_lock);
	    return 0;
	}
#### (1) binder_alloc_mmap_handler
[ -> kernel/msm-3.18/drivers/staging/android/binder_alloc.c]

	int binder_alloc_mmap_handler(struct binder_alloc *alloc, struct vm_area_struct *vma)
	{
		// struct vm_area_struct *vma 进程虚拟地址空间
	    // 内核虚拟地址空间
	    struct vm_struct *area;
	    struct binder_buffer *buffer;
	 
	    mutex_lock(&binder_alloc_mmap_lock);
	    // 采用VM_IOREMAP方式,分配一个内核虚拟地址空间, 和进程虚拟地址空间大小一致
	    area = get_vm_area(vma->vm_end - vma->vm_start, VM_IOREMAP);
	    // 设置buffer, 指向内核虚拟地址空间的地址
	    alloc->buffer = area->addr;
	    // 设置用户虚拟地址空间和内核虚拟地址空间偏移量, 地址偏移量=用户虚拟地址空间-内核虚拟地址空间
	    alloc->user_buffer_offset = vma->vm_start - (uintptr_t)alloc->buffer;
	    mutex_unlock(&binder_alloc_mmap_lock);
	 
	    // 分配物理页的指针数组, 数组大小为vma的等效page个数
	    alloc->pages = kzalloc(sizeof(alloc->pages[0]) *
		    ((vma->vm_end - vma->vm_start) / PAGE_SIZE), GFP_KERNEL);
	    // 设置buffer_size
	    alloc->buffer_size = vma->vm_end - vma->vm_start;
	    // 创建buffer, 指向alloc->buffer
	    buffer = kzalloc(sizeof(*buffer), GFP_KERNEL);
	    buffer->data = alloc->buffer;
	    // 将buffer地址加入所属进程的buffers队列
	    list_add(&buffer->entry, &alloc->buffers);
	    buffer->free = 1;
	    // 将空闲buffer放入alloc->free_buffers中
	    binder_insert_free_buffer(alloc, buffer);
	    // 异步可用空间大小为buffer总大小的一半
	    alloc->free_async_space = alloc->buffer_size / 2;
	    barrier();
	    // 设置进程虚拟地址空间
	    alloc->vma = vma;
	    alloc->vma_vm_mm = vma->vm_mm;
	    /* Same as mmgrab() in later kernel versions */
	    atomic_inc(&alloc->vma_vm_mm->mm_count);
	 
	    return 0;
	}

这个方法的作用是: 将进程虚拟地址空间和内核虚拟地址空间,映射到同一块物理内存空间.        
- alloc->buffer 指向内核虚拟地址空间
- alloc->pages 指向物理内存空间
- alloc->vma 指向进程虚拟地址空间
- alloc->user_buffer_offset 进程虚拟地址空间和内核虚拟地址空间的偏移量
- alloc->buffer_size 映射的空间大小

### 1.4 binder_become_context_manager
[ -> frameworks/native/cmds/servicemanager/binder.c]

	int binder_become_context_manager(struct binder_state *bs)
	{
	    // 通过ioctl, 传递BINDER_SET_CONTEXT_MGR指令
	    return ioctl(bs->fd, BINDER_SET_CONTEXT_MGR, 0);
	}

通过【第1.3.2节】可知, 会走到BINDER_SET_CONTEXT_MGR分支, 调用binder_ioctl_set_ctx_mgr方法.        
#### 1.4.1 binder_ioctl_set_ctx_mgr
[ -> kernel/msm-3.18/drivers/staging/android/binder.c]

	// 创建进程的binder_node节点,并将其设置为进程的上下文管理器节点
	static int binder_ioctl_set_ctx_mgr(struct file *filp)
	{
	    int ret = 0;
	    struct binder_proc *proc = filp->private_data;
	    struct binder_context *context = proc->context;
	    struct binder_node *new_node;
	    kuid_t curr_euid = current_euid();
	 
	    ret = security_binder_set_context_mgr(proc->tsk);
	 
	    context->binder_context_mgr_uid = curr_euid;
	 
	    new_node = binder_new_node(proc, NULL);
	    new_node->local_weak_refs++;
	    new_node->local_strong_refs++;
	    new_node->has_strong_ref = 1;
	    new_node->has_weak_ref = 1;
	    context->binder_context_mgr_node = new_node;
	 
	    binder_put_node(new_node);
	 
	    return ret;
	}

### 1.5 binder_loop
[ -> frameworks/native/cmds/servicemanager/binder.c]

	void binder_loop(struct binder_state *bs, binder_handler func)
	{
	    int res;
	    struct binder_write_read bwr;
	    uint32_t readbuf[32];
	 
	    bwr.write_size = 0;
	    bwr.write_consumed = 0;
	    bwr.write_buffer = 0;
	 
	    readbuf[0] = BC_ENTER_LOOPER;
	    // 向binder驱动发送BC_ENTER_LOOPER指令, service manager进入循环
	    binder_write(bs, readbuf, sizeof(uint32_t));
	 
	    for (;;) {
	        bwr.read_size = sizeof(readbuf);
	        bwr.read_consumed = 0;
	        bwr.read_buffer = (uintptr_t) readbuf;
	        // 进入循环, 不断从binder驱动读数据
	        res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
	        // 解析binder读缓存的信息,并交由svcmgr_handler
	        res = binder_parse(bs, 0, (uintptr_t) readbuf, bwr.read_consumed, func);
	    }
	}

#### 1.5.1 binder_write
[ -> frameworks/native/cmds/servicemanager/binder.c]
	
	int binder_write(struct binder_state *bs, void *data, size_t len)
	{
	    struct binder_write_read bwr;
	    int res;
	 
	    bwr.write_size = len;
	    bwr.write_consumed = 0;
	    bwr.write_buffer = (uintptr_t) data;
	    bwr.read_size = 0;
	    bwr.read_consumed = 0;
	    bwr.read_buffer = 0;
	    res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
	 
	    return res;
	}

binder_write的调用链是: ioctl -> binder_ioctl -> binder_ioctl_write_read -> binder_thread_write

#### 1.5.2 binder_thread_write
[ -> kernel/msm-3.18/drivers/staging/android/binder.c]

	static int binder_thread_write(struct binder_proc *proc, struct binder_thread *thread,
            binder_uintptr_t binder_buffer, size_t size, binder_size_t *consumed)
	{
	    uint32_t cmd;
	    // __user表示用户空间数据
	    void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
	    void __user *ptr = buffer + *consumed;
	    void __user *end = buffer + size;
	  
	    while (ptr < end && thread->return_error.cmd == BR_OK) {
	        int ret;
	          
	        // 获取cmd指令(BC码)
	        if (get_user(cmd, (uint32_t __user *)ptr))
	            return -EFAULT;
	        ptr += sizeof(uint32_t);
	        switch (cmd) {
	        case BC_INCREFS: ...
	        case BC_ACQUIRE: ...
	        case BC_RELEASE: ...
	        case BC_DECREFS: ...
	        case BC_INCREFS_DONE: ...
	        case BC_ACQUIRE_DONE: ...
	        case BC_ATTEMPT_ACQUIRE: ...
	        case BC_ACQUIRE_RESULT: ...
	        case BC_FREE_BUFFER: ...
	        case BC_TRANSACTION_SG: ...
	        case BC_REPLY_SG: ...
	        case BC_TRANSACTION:
	        case BC_REPLY:{
	            // 将用户空间数据拷贝到内核空间tr
	            struct binder_transaction_data tr;
	            if (copy_from_user(&tr, ptr, sizeof(tr)))
	                return -EFAULT;
	            ptr += sizeof(tr);
	             
	            binder_transaction(proc, thread, &tr, cmd == BC_REPLY, 0);
	            break;
	        }
	        case BC_REGISTER_LOOPER: ...
	        case BC_ENTER_LOOPER:
	            thread->looper |= BINDER_LOOPER_STATE_ENTERED;
	            break;
	        case BC_EXIT_LOOPER: ...
	        case BC_REQUEST_DEATH_NOTIFICATION: ...
	        case BC_CLEAR_DEATH_NOTIFICATION: ...
	        case BC_DEAD_BINDER_DONE: ...
	        }
	          
	        *consumed = ptr - buffer;
	    }
	    return 0;
	}
发送BC_ENTER_LOOPER指令, 相当往binder驱动写数据, 设置线程循环标识.

	thread->looper |= BINDER_LOOPER_STATE_ENTERED;

然后调用ioctl不断读取binder驱动数据. binder_parse解析读取到的数据.

#### 1.6 binder_parse
[ -> frameworks/native/cmds/servicemanager/binder.c]

	int binder_parse(struct binder_state *bs, struct binder_io *bio,
                 uintptr_t ptr, size_t size, binder_handler func)
	{
	    int r = 1;
	    uintptr_t end = ptr + (uintptr_t) size;
	 
	    while (ptr < end) {
	        uint32_t cmd = *(uint32_t *) ptr;
	        ptr += sizeof(uint32_t);
	 
	        switch(cmd) {
	        case BR_NOOP:
	            break;
	        case BR_TRANSACTION_COMPLETE:
	            break;
	        case BR_INCREFS:
	        case BR_ACQUIRE:
	        case BR_RELEASE:
	        case BR_DECREFS:
	            ptr += sizeof(struct binder_ptr_cookie);
	            break;
	        case BR_TRANSACTION_SEC_CTX:
	        case BR_TRANSACTION: {
	            struct binder_transaction_data *txn.transaction_data =
			            (struct binder_transaction_data*) ptr;
	 
	            if (func) {
	                unsigned rdata[256/4];
	                struct binder_io msg;
	                struct binder_io reply;
	                int res;
	                // 初始化reply
	                bio_init(&reply, rdata, sizeof(rdata), 4);
	                // 初始化msg
	                bio_init_from_txn(&msg, &txn.transaction_data);
	                // 执行svcmgr_handler方法
	                res = func(bs, &txn, &msg, &reply);
	                // 向binder发送reply结果
	                binder_send_reply(bs, &reply, txn.transaction_data.data.ptr.buffer, res);
	            }
	            break;
	        }
	        case BR_REPLY: {
	            struct binder_transaction_data *txn = (struct binder_transaction_data *) ptr;
	            binder_dump_txn(txn);
	            if (bio) {
	                bio_init_from_txn(bio, txn);
	                bio = 0;
	            } else {
	                /* todo FREE BUFFER */
	            }
	            ptr += sizeof(*txn);
	            r = 0;
	            break;
	        }
	        case BR_DEAD_BINDER: {
	            struct binder_death *death = (struct binder_death *)(uintptr_t) *
			            (binder_uintptr_t *)ptr;
	            ptr += sizeof(binder_uintptr_t);
	            death->func(bs, death->ptr);
	            break;
	        }
	        case BR_FAILED_REPLY:
	            r = -1;
	            break;
	        case BR_DEAD_REPLY:
	            r = -1;
	            break;
	        default:
	            ALOGE("parse: OOPS %d\n", cmd);
	            return -1;
	        }
	    }
	 
	    return r;
	}

经历过ioctl读操作后, 调用binder_parse解析读取的数据, 然后走BR_TRANSACTION分支, 将数据交由svcmgr_handler方法处理.

### 1.7 svcmgr_handler
[ -> frameworks/native/cmds/servicemanager/service_manager.c]

	int svcmgr_handler(struct binder_state *bs,
                   struct binder_transaction_data_secctx *txn_secctx,
                   struct binder_io *msg,
                   struct binder_io *reply)
	{
	    uint16_t *s;
	    size_t len;
	    uint32_t handle;
	  
	    struct binder_transaction_data *txn = &txn_secctx->transaction_data;
	  
	    if (txn->target.ptr != BINDER_SERVICE_MANAGER)
	        return -1;
	  
	    if (txn->code == PING_TRANSACTION)
	        return 0;
	  
	    switch(txn->code) {
	    case SVC_MGR_GET_SERVICE:
	    case SVC_MGR_CHECK_SERVICE:
	        s = bio_get_string16(msg, &len);
	        // 根据名称name查找相应服务
	        handle = do_find_service(s, len, txn->sender_euid, txn->sender_pid,
	                                 (const char*) txn_secctx->secctx);
	        bio_put_ref(reply, handle);
	        return 0;
	    case SVC_MGR_ADD_SERVICE:
	        s = bio_get_string16(msg, &len);
	        handle = bio_get_ref(msg);
	        allow_isolated = bio_get_uint32(msg) ? 1 : 0;
	        dumpsys_priority = bio_get_uint32(msg);
	        if (do_add_service(bs, s, len, handle, txn->sender_euid, allow_isolated,
			    dumpsys_priority, txn->sender_pid, (const char*) txn_secctx->secctx))
	            return -1;
	        break;
	    case SVC_MGR_LIST_SERVICES: {
	        uint32_t n = bio_get_uint32(msg);
	        uint32_t req_dumpsys_priority = bio_get_uint32(msg);
	 
	        // 检查权限
	        if (!svc_can_list(txn->sender_pid, (const char*) txn_secctx->secctx,
		        txn->sender_euid)) {
	            return -1;
	        }
	        si = svclist;
	        // walk through the list of services n times skipping services that
	        // do not support the requested priority
	        while (si) {
	            if (si->dumpsys_priority & req_dumpsys_priority) {
	                if (n == 0) break;
	                n--;
	            }
	            si = si->next;
	        }
	        if (si) {
	            bio_put_string16(reply, si->name);
	            return 0;
	        }
	        return -1;
	    }
	    }
	  
	    bio_put_uint32(reply, 0);
	    return 0;
	}

servicemanager作为服务管理者, 主要作用是`添加服务`和`获取服务`.

#### 1.7.1 do_find_service
[ -> frameworks/native/cmds/servicemanager/service_manager.c]

	uint32_t do_find_service(const uint16_t *s, size_t len, uid_t uid,
			pid_t spid, const char* sid)
	{
	    struct svcinfo *si = find_svc(s, len);
	    return si->handle;
	} 

	struct svcinfo *find_svc(const uint16_t *s16, size_t len)
	{
	    struct svcinfo *si;
	  
		// 遍历svclist链表, 并根据name查找对应的服务.
	    for (si = svclist; si; si = si->next) {
	        if ((len == si->len) &&
	            !memcmp(s16, si->name, len * sizeof(uint16_t))) {
	            return si;
	        }
	    }
	    return NULL;
	}

#### 1.7.2 do_add_service
[ -> frameworks/native/cmds/servicemanager/service_manager.c]

	int do_add_service(struct binder_state *bs, const uint16_t *s, size_t len,
			uint32_t handle, uid_t uid, int allow_isolated,
			uint32_t dumpsys_priority, pid_t spid, const char* sid)
	{
	    struct svcinfo *si;
	 
	    // handle和长度检查
	    if (!handle || (len == 0) || (len > 127))
	        return -1;
	     
	    // 权限检查
	    if (!svc_can_register(s, len, spid, sid, uid)) {
	        return -1;
	    }

	    // 查找服务
	    si = find_svc(s, len);
	    if (si) {
	        // 若服务已注册,释放相应服务
	        if (si->handle) {
	            svcinfo_death(bs, si);
	        }
	        si->handle = handle;
	    } else {
	        // 创建服务, 并添加到svclist列表头部
	        si = malloc(sizeof(*si) + (len + 1) * sizeof(uint16_t));
	 
	        si->handle = handle;
	        si->len = len;
	        memcpy(si->name, s, (len + 1) * sizeof(uint16_t));
	        si->name[len] = '\0';
	        si->death.func = (void*) svcinfo_death;
	        si->death.ptr = si;
	        si->allow_isolated = allow_isolated;
	        si->dumpsys_priority = dumpsys_priority;
	        si->next = svclist;
	        svclist = si;
	    }
	 
	    //BC_ACQUIRE，handle为目标的信息，通过ioctl发送给binder驱动
	    binder_acquire(bs, handle);
	    //BC_REQUEST_DEATH_NOTIFICATION，通过ioctl发送给binder驱动主要用于清理内存等收尾工作
	    binder_link_to_death(bs, handle, &si->death);
	    return 0;
	}

## 2. /dev/binder设备文件的创建过程

设备文件/dev/binder是在Binder驱动程序初始化的时候创建的.

[ -> kernel/msm-3.18/drivers/staging/android/binder.c]

	// binder初始化入口
	device_initcall(binder_init);

### 2.1 binder_init 
[ -> kernel/msm-3.18/drivers/staging/android/binder.c]

	// binder设备参数定义
	static char *binder_devices_param = CONFIG_ANDROID_BINDER_DEVICES;
	// kernel/configs/o/android-3.18/android-base.config
	CONFIG_ANDROID_BINDER_DEVICES="binder,hwbinder,vndbinder"
	
	// binder初始化函数
	static int __init binder_init(void)
	{
	    int ret;
	    char *device_name, *device_names;
	    struct binder_device *device;
	    struct hlist_node *tmp;
 
	    binder_alloc_shrinker_init();
	    atomic_set(&binder_transaction_log.cur, ~0U);
	    atomic_set(&binder_transaction_log_failed.cur, ~0U);
	    binder_deferred_workqueue = create_singlethread_workqueue("binder");
	 
	    // 创建binder相关文件目录
	    binder_debugfs_dir_entry_root = debugfs_create_dir("binder", NULL);
	    if (binder_debugfs_dir_entry_root)
	        binder_debugfs_dir_entry_proc = debugfs_create_dir("proc",
				    binder_debugfs_dir_entry_root);
	 
	    if (binder_debugfs_dir_entry_root) {
	        debugfs_create_file("state", S_IRUGO, binder_debugfs_dir_entry_root,
			        NULL, &binder_state_fops);
	        debugfs_create_file("stats", S_IRUGO, binder_debugfs_dir_entry_root,
			        NULL, &binder_stats_fops);
	        debugfs_create_file("transactions", S_IRUGO, binder_debugfs_dir_entry_root,
			        NULL, &binder_transactions_fops);
	        debugfs_create_file("transaction_log", S_IRUGO, binder_debugfs_dir_entry_root, 
			        &binder_transaction_log, &binder_transaction_log_fops);
	        debugfs_create_file("failed_transaction_log",S_IRUGO,binder_debugfs_dir_entry_root,
			        &binder_transaction_log_failed, &binder_transaction_log_fops);
	    }
	 
	    // 读取binder设备参数, 其值为"binder,hwbinder,vndbinder"
	    device_names = kzalloc(strlen(binder_devices_param) + 1, GFP_KERNEL);
	    strcpy(device_names, binder_devices_param);
	 
	    while ((device_name = strsep(&device_names, ","))) {
	        // 创建和初始化binder_device, 并注册设备驱动文件
	        ret = init_binder_device(device_name);
	    }
	 
	    return ret;
	}

### 2.2 init_binder_device
[ -> kernel/msm-3.18/drivers/staging/android/binder.c]

	static int __init init_binder_device(const char *name)
	{
	    int ret;
	    struct binder_device *binder_device;
	    // 创建binder_device对象
	    binder_device = kzalloc(sizeof(*binder_device), GFP_KERNEL);
		
		// 初始化binder_device对象, 设置miscdev和context属性
	    binder_device->miscdev.fops = &binder_fops;
	    binder_device->miscdev.minor = MISC_DYNAMIC_MINOR;
	    binder_device->miscdev.name = name;
	 
	    binder_device->context.binder_context_mgr_uid = INVALID_UID;
	    binder_device->context.name = name;
	    mutex_init(&binder_device->context.context_mgr_node_lock);
	 
	    // 注册设备驱动文件
	    ret = misc_register(&binder_device->miscdev);
	    hlist_add_head(&binder_device->hlist, &binder_devices);
	 
	    return ret;
	}

	// binder文件操作对象
	static const struct file_operations binder_fops = {
	    .owner = THIS_MODULE,
	    .poll = binder_poll,
	    .unlocked_ioctl = binder_ioctl,
	    .compat_ioctl = binder_ioctl,
	    .mmap = binder_mmap,
	    .open = binder_open,
	    .flush = binder_flush,
	    .release = binder_release,
	};

## 3.时序图
![seq-start-service-manager](/image/2019-12-31-start-service-manager/seq-start-service-manager.png)

## 4.总结
servicemanager和/dev/binder驱动交互, 使用的servicemanager/binder.c文件, 本质还是调用android/binder.c, 不过是在android/binder.c的基础上做了一层封装.

servicemanager守护进程启动过程分为3步:        
**(1) binder_open**        
binder_open又分为3步: open / ioctl / mmap        
- `open`:  打开binder驱动, 创建proc对象保存打开设备文件/dev/binder的进程相关信息, 并将proc保存到filp->private_data中.
- `ioctl`: 设置binder版本号, 使用户空间和内核空间版本信息一致.        
- `mmap`: 分配128K的虚拟内存映射空间, 并将进程虚拟地址空间和内核虚拟地址空间,映射到同一块物理内存空间. 

**(2) binder_become_context_manager**        
创建进程的binder_node节点, 并将其设置为进程的上下文管理器节点. 打开设备文件/dev/binder的进程proc成为守护进程. servicemanager成为上下文管理器. 

**(3) binder_loop**        
进入for循环, 等待Client端的请求. 读取到的结果经binder_parse方法解析, 交由svcmgr_handler方法处理.        
svcmgr_handler支持do_add_service和do_find_service. 每个服务对应一个svcinfo, 注册服务时会将svcinfo添加到svclist链表; 查找服务时, 遍历svclist, 根据name进行匹配.

