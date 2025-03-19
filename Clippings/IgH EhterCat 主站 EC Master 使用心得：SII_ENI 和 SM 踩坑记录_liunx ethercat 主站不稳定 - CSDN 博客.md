---
url: https://blog.csdn.net/wy278738285/article/details/134110564
title: IgH EhterCat 主站 EC Master 使用心得：SII,ENI 和 SM 踩坑记录_liunx ethercat 主站不稳定 - CSDN 博客
date: 2025-03-19 19:14:07
tag: 
summary: 
---
声明一下：本文假设你对 EtherCat 有理解和对 IgH 有使用经验，遇到问题的时候来看最合适。假如你是初学者，类似帖子有很多的，也可以找 ChatGPT 对话学习效果更好。

2013 年，我首次接触 EtherCat 通讯。初碰实时 Linux+Xenomai2（Ubuntu10.04）的时候，有很多东西要做，根本没经历研究 IgH 的主站，直接选择 acontis 收费主站。

2017 年，我与北京 acontis 总代的技术支持聊了一下才知道，原来以第一个带时钟的从站作为 DC 时钟时，acontis 会新开一个实时线程来作通讯。这对于更低的通讯周期的要求的我来说，是很麻烦的事情。于是我转过头来奔向 IgH。

此时我已经使用了 Ubuntu14.04.5+Linux3.18.20，这个内核版本是 Xenomai 官方 patch 支持 Linux3.x 的最后一个版本。别问我为什么不选 Linux3.14，问就是 Uubntu14.04.5 在 Linux3.14 下不稳定。而 IgH 的网卡驱动最接近这个内核版本的的就是 Linux3.14。直接暴力手动 patch 移植 intel 网卡驱动。运气很好没怎么踩坑，经过无数次强制重启直到我开始怀疑我的硬盘还行不行的时候，我成功了 。这个时候我飘了，决定把我的 Y430P 笔记本电脑网卡的也移植一下。结果失败了，我猜想原因是出现在硬件联络层。这个时候我才想起来当初 acontis 技术说过的：事实上，Realtek 网卡效果没有 Intel 网卡效果好的。后来我看了这两家驱动代码才有点明白，大概就是网卡厂商开放的功能是不同的，Intel 的内核驱动代码量比 Realtek 多很多。这或许对实时数据收发是有益处的。

用过 IgH 的人都知道，IgH 只需要配置从站的 Process Data（SM2 和 SM3）就够了。其他配置直接从 EEPROM 读取的。做 EtherCat 产品嘛，TwinCat 肯定是标杆，我就像模像样的写了一个解析 acontis 的 ENI 中的 SM2 和 SM3 的 PDO 部分配置的工具（其他的不用我管，不改源代码我也管不了），用来配置 IgH。除了显得我们很专业，也为了防止用户自己换其他品牌驱动后出现生产或事故问题。

本以为事情就这样愉快的结束了。我可以一直陪甘雨老婆逛提瓦特大陆了。然而一年后，同事说：我们采购了一家 I/O 板卡, IgH 切不到 OP。What？Master 代码我一共没看几行，直接凌乱了，甘雨老婆都不香了。我开始阅读和调试 ioctl.c，master.c 和 slave.c。是的，你没有看错，我没读 fsm*.c, 我就是这么的无知。问题已经被我找到了，IgH 读 EEPROM 的 SII 来配置 SM0~SM3，但是没读到，或读到了错误的 SII（这点我一直不理解，为什么 SII 和 ESI 会不一样），当时我定位到了 slave 部分（最外围嘛），SII 不对，所以 MailBox 失败，COE 失败, SM1 到 SM3 就都没有配置。之前说了，我只配置了 SM2 和 SM3。没有 SM0 和 SM1 的时候，IgH 就把 SM2 和 SM3 配置到了 SM0 和 SM1。而 IgH 是写死的，SM0 和 SM1 走 MailBox ，SM3 和 SM4 走 Process Date，通讯肯定有问题的。

既然如此，我从 ENI 读一下 SM0 和 SM1，在 ioctl.c 加一个函数传递配置，slave.c 加一个函数配置 sync 数组，搞定。甘雨老婆我来了！

```
    //伪代码，照这个生成SII数据
    uint8_t data[8 * EC_MAX_SYNC_MANAGERS] = {0}; 
    size_t size = sii_mb_sync_count * 8 + sii_pd_sync_count * 8;
 
    for(j = 0; j < sii_mb_sync_count; j++){  
        EC_WRITE_U16(data + j*8 + 0, sii_mb_sync[j].physical_start_address);
        EC_WRITE_U16(data + j*8 + 2, sii_mb_sync[j].default_length);
        EC_WRITE_U8(data  + j*8 + 4, sii_mb_sync[j].control_bytes);
        EC_WRITE_U8(data  + j*8 + 6, sii_mb_sync[j].enable);
    }
    for(k = 0; k < sii_pd_sync_count; k++){  
        EC_WRITE_U16(data + (j+k)*8 + 0, sii_pd_sync[k].physical_start_address);
        EC_WRITE_U16(data + (j+k)*8 + 2, sii_pd_sync[k].default_length);
        EC_WRITE_U8(data  + (j+k)*8 + 4, sii_pd_sync[k].control_bytes);
        EC_WRITE_U8(data  + (j+k)*8 + 6, sii_pd_sync[k].enable);
    }
            
    ecrt_master_set_sii_sync(master, Index, mailbox_protocols, data, size);
  
```

```
// lib/master.C下添加
int ecrt_master_set_sii_sync(ec_master_t *master, uint16_t slave_position, uint16_t mailbox_protocols,
        uint8_t *data, size_t size)
{
    ec_ioctl_slave_sii_sync_t sii;
    int ret;
    sii.mailbox_protocols = mailbox_protocols;
    sii.slave_position = slave_position;
    sii.size = size;
    sii.data = data;
 
    ret = ioctl(master->fd, EC_IOCTL_SLAVE_INIT_SII_SYNC, &sii);
    if (EC_IOCTL_IS_ERROR(ret)) {
        fprintf(stderr, "Failed to execute SDO download: %s\n",
                strerror(EC_IOCTL_ERRNO(ret)));
        return -EC_IOCTL_ERRNO(ret);
    }
    return 0;
}
```

```
//ioctl.h
typedef struct {
    // inputs
    uint16_t slave_position;
    uint16_t mailbox_protocols; /**< Supported mailbox protocols. */
    uint8_t *data; /**< Category data. */
    size_t size; /**< Number of bytes. */
} ec_ioctl_slave_sii_sync_t;
#define EC_IOCTL_SLAVE_INIT_SII_SYNC   EC_IOW(0x5a, ec_ioctl_slave_sii_sync_t)
```

```
//ioctl.c中添加，记得在EC_IOCTL中i调用这个函数
/*****************************************************************************/
static ATTRIBUTES int ec_ioctl_slave_init_sii_sync(
        ec_master_t *master, /**< EtherCAT master. */
        void *arg, /**< Userspace address to store the results. */
        ec_ioctl_context_t *ctx /**< Private data structure of file handle. */
        )
{
    //ec_slave_config_t *sc;
    ec_slave_t *slave;
    ec_ioctl_slave_sii_sync_t data;
    uint8_t *sii_data = NULL;
    int ret;
 
    if (unlikely(!ctx->requested))
        return -EPERM;
 
    if (copy_from_user(&data, (void __user *) arg, sizeof(data)))
        return -EFAULT;
 
    if (!data.size)
        return -EINVAL;
 
    if (!(sii_data = kmalloc(data.size, GFP_KERNEL))) {
        return -ENOMEM;
    }
 
    if (copy_from_user(sii_data, (void __user *) data.data, data.size)) {
        kfree(sii_data);
        return -EFAULT;
    }
 
    if (down_interruptible(&master->master_sem)) {
        kfree(sii_data);
        return -EINTR;
    }
 
    if (!(slave = ec_master_find_slave(
                    master, 0, data.slave_position))) {
        up(&master->master_sem);
        EC_MASTER_ERR(master, "Slave %u does not exist!\n",
                data.slave_position);
        kfree(sii_data);
        return -EINVAL;
    }
    /*
    if (!(sc = ec_master_get_config(master, data.slave_position))) {
        up(&master->master_sem);
        EC_MASTER_ERR(master, "Slave config %u does not exist!\n",
                data.slave_position);
        kfree(sii_data);       
        return -ENOENT;
    }
    
    sc->use_eni_sii = true;
    sc->mailbox_protocols = data.mailbox_protocols;
    sc->sii_syncs_data = kmalloc(data.size, GFP_KERNEL);
    memcpy(sc->sii_syncs_data, sii_data, data.size);
    sc->sii_syncs_size = data.size;
    */
    //ec_slave_clear_sync_managers(slave);
    slave->sii.mailbox_protocols = data.mailbox_protocols;
    if(0 != (ret = ec_slave_init_sii_syncs(slave, sii_data, data.size))){
        EC_MASTER_ERR(master, "Slave %u fetch_sii_syncs Failed!\n",
                data.slave_position);
    }
    up(&master->master_sem); /** \todo sc could be invalidated */
    kfree(sii_data);
    return 0;
}
```

```
//slave.c中添加
int ec_slave_init_sii_syncs(
        ec_slave_t *slave, /**< EtherCAT slave. */
        const uint8_t *data, /**< Category data. */
        size_t data_size /**< Number of bytes. */
        )
{
    unsigned int i, count, total_count;
    ec_sync_t *sync;
    size_t memsize;
    ec_sync_t *syncs;
    uint8_t index;
 
    // one sync manager struct is 4 words long
    if (data_size % 8) {
        EC_SLAVE_ERR(slave, "Invalid SII sync manager category size %zu.\n",
                data_size);
        return -EINVAL;
    }
 
    count = data_size / 8;
    if (count) {
        slave->sii.sync_count = 0;
 
        total_count = count + slave->sii.sync_count;
        if (total_count > EC_MAX_SYNC_MANAGERS) {
            EC_SLAVE_ERR(slave, "Exceeded maximum number of"
                    " sync managers!\n");
            return -EOVERFLOW;
        }
        memsize = sizeof(ec_sync_t) * total_count;
        if (!(syncs = kmalloc(memsize, GFP_KERNEL))) {
            EC_SLAVE_ERR(slave, "Failed to allocate %zu bytes"
                    " for sync managers.\n", memsize);
            return -ENOMEM;
        }
 
        for (i = 0; i < slave->sii.sync_count; i++)
            ec_sync_init_copy(syncs + i, slave->sii.syncs + i);
 
        // initialize new sync managers
        for (i = 0; i < count; i++, data += 8) {
            index = i + slave->sii.sync_count;
            sync = &syncs[index];
 
            ec_sync_init(sync, slave);
            sync->physical_start_address = EC_READ_U16(data);
            sync->default_length = EC_READ_U16(data + 2);
            sync->control_register = EC_READ_U8(data + 4);
            sync->enable = EC_READ_U8(data + 6);
            EC_SLAVE_DBG(slave, 1, "index = %d, start_addr = %d, def_len = %d, control_reg = %d, enable = %d\n",
                index, sync->physical_start_address, sync->default_length, sync->control_register, sync->enable);
        }
 
        if (slave->sii.syncs)
            kfree(slave->sii.syncs);
        slave->sii.syncs = syncs;
        slave->sii.sync_count = total_count;
    }
 
    return 0;
}
```

又过差不多一年，同事说，客户要求，加工过程中，从站要经常断电，从新上电以后切不到 OP！要知道，不管是机器人还是机床，基本都是要么一起通电要么一起断电的。所以这个真没考虑过。这回直接查到 fsm*.c 了, 当网络拓扑结构发生变化，slaves 被从新扫描，我之前配置的 slave 部分丢了。断电以后没有人去配置 slave 的 sync 部分了，因此切不到 OP。一番 debug 后，最新的配置方式是把 SM 信息写到 master 结构的 slave 中, 在 fsm 阶段替换 slave 的 SII 中的 sync 部分，这样 SM 部分彻底告别 SII 了。SM 的部分我想基本完事了吧。

```
    //伪代码，照这个生成SII数据
    uint8_t data[8 * EC_MAX_SYNC_MANAGERS] = {0}; 
    size_t size = sii_mb_sync_count * 8 + sii_pd_sync_count * 8;
 
    for(j = 0; j < sii_mb_sync_count; j++){  
        EC_WRITE_U16(data + j*8 + 0, sii_mb_sync[j].physical_start_address);
        EC_WRITE_U16(data + j*8 + 2, sii_mb_sync[j].default_length);
        EC_WRITE_U8(data  + j*8 + 4, sii_mb_sync[j].control_bytes);
        EC_WRITE_U8(data  + j*8 + 6, sii_mb_sync[j].enable);
    }
    for(k = 0; k < sii_pd_sync_count; k++){  
        EC_WRITE_U16(data + (j+k)*8 + 0, sii_pd_sync[k].physical_start_address);
        EC_WRITE_U16(data + (j+k)*8 + 2, sii_pd_sync[k].default_length);
        EC_WRITE_U8(data  + (j+k)*8 + 4, sii_pd_sync[k].control_bytes);
        EC_WRITE_U8(data  + (j+k)*8 + 6, sii_pd_sync[k].enable);
    }
              
    ecrt_slave_config_sii_sync(slave_config, mailbox_protocols, data, size);
```

```
// lib/slave_config.c
/*****************************************************************************/
int ecrt_slave_config_sii_sync(ec_slave_config_t *sc, uint16_t mailbox_protocols,
        uint8_t *data, size_t size)
{
    ec_ioctl_config_sii_sync_t sii;
    int ret;
    sii.config_index = sc->index;
    sii.mailbox_protocols = mailbox_protocols;
    sii.size = size;
    sii.data = data;
 
    ret = ioctl(sc->master->fd, EC_IOCTL_SC_INIT_SII_SYNC, &sii);
    if (EC_IOCTL_IS_ERROR(ret)) {
        fprintf(stderr, "Failed to execute SDO download: %s\n",
                strerror(EC_IOCTL_ERRNO(ret)));
        return -EC_IOCTL_ERRNO(ret);
    }
    return 0;
}
```

```
//ioctl.h
typedef struct {
    // inputs
    uint32_t config_index;
    uint16_t mailbox_protocols; /**< Supported mailbox protocols. */
    uint8_t *data; /**< Category data. */
    size_t size; /**< Number of bytes. */
} ec_ioctl_config_sii_sync_t;
#define EC_IOCTL_SC_INIT_SII_SYNC      EC_IOW(0x5b, ec_ioctl_config_sii_sync_t)
 
```

```
//ioctl.c中添加，记得在EC_IOCTL中i调用这个函数
/*****************************************************************************/
 
static ATTRIBUTES int ec_ioctl_config_init_sii_sync(
        ec_master_t *master, /**< EtherCAT master. */
        void *arg, /**< ioctl() argument. */
        ec_ioctl_context_t *ctx /**< Private data structure of file handle. */
        )
{
    ec_ioctl_config_sii_sync_t data;
    ec_slave_config_t *sc;
    uint8_t *sii_data = NULL;
 
    if (unlikely(!ctx->requested))
        return -EPERM;
 
    if (copy_from_user(&data, (void __user *) arg, sizeof(data))) {
        return -EFAULT;
    }
    if (!data.size)
        return -EINVAL;
 
    if (!(sii_data = kmalloc(data.size, GFP_KERNEL))) {
        return -ENOMEM;
    }
 
    if (copy_from_user(sii_data, (void __user *) data.data, data.size)) {
        kfree(sii_data);
        return -EFAULT;
    }
 
    if (down_interruptible(&master->master_sem)) {
        kfree(sii_data);
        return -EINTR;
    }
 
    if (!(sc = ec_master_get_config(
                    master, data.config_index))) {
        up(&master->master_sem);
        EC_MASTER_ERR(master, "Slave config %u does not exist!\n",
                data.config_index);
        kfree(sii_data);
        return -EINVAL;
    }
 
    sc->use_eni_sii = true;
    sc->mailbox_protocols = data.mailbox_protocols;
    sc->sii_syncs_data = kmalloc(data.size, GFP_KERNEL);
    memcpy(sc->sii_syncs_data, sii_data, data.size);
    sc->sii_syncs_size = data.size;
 
    up(&master->master_sem); /** \todo sc could be invalidated */
    kfree(sii_data);
 
    return 0;
}
```

```
//fsm_slave_config.c 中的函数，自己对比修改一下 #if 1 部分
void ec_fsm_slave_config_enter_mbox_sync(
        ec_fsm_slave_config_t *fsm /**< slave state machine */
        )
{
    ec_slave_t *slave = fsm->slave;
    ec_datagram_t *datagram = fsm->datagram;
#if 1
    ec_slave_config_t *sc = slave->config;
    ec_master_t *master = slave->master;
#endif
    unsigned int i;
 
    // slave is now in INIT
    if (slave->current_state == slave->requested_state) {
        fsm->state = ec_fsm_slave_config_state_end; // successful
        EC_SLAVE_DBG(slave, 1, "Finished configuration.\n");
        return;
    }
#if 1
    if(master->active && sc && sc->use_eni_sii){
        if(0 != (ec_slave_init_sii_syncs(slave, sc->sii_syncs_data, sc->sii_syncs_size)) ){
            EC_SLAVE_ERR(slave, "init_sii_syncs Failed!\n");
        }
        slave->sii.mailbox_protocols = sc->mailbox_protocols;
    }
#endif
    if (!slave->sii.mailbox_protocols) {
        // no mailbox protocols supported
        EC_SLAVE_DBG(slave, 1, "Slave does not support"
                " mailbox communication.\n");
#ifdef EC_SII_ASSIGN
        ec_fsm_slave_config_enter_assign_pdi(fsm);
#else
        ec_fsm_slave_config_enter_boot_preop(fsm);
#endif
        return;
    }
 
    EC_SLAVE_DBG(slave, 1, "Configuring mailbox sync managers...\n");
 
    if (slave->requested_state == EC_SLAVE_STATE_BOOT) {
        ec_sync_t sync;
 
        ec_datagram_fpwr(datagram, slave->station_address, 0x0800,
                EC_SYNC_PAGE_SIZE * 2);
        ec_datagram_zero(datagram);
 
        ec_sync_init(&sync, slave);
        sync.physical_start_address = slave->sii.boot_rx_mailbox_offset;
        sync.control_register = 0x26;
        sync.enable = 1;
        ec_sync_page(&sync, 0, slave->sii.boot_rx_mailbox_size,
                EC_DIR_INVALID, // use default direction
                0, // no PDO xfer
                datagram->data);
        slave->configured_rx_mailbox_offset =
            slave->sii.boot_rx_mailbox_offset;
        slave->configured_rx_mailbox_size =
            slave->sii.boot_rx_mailbox_size;
 
        ec_sync_init(&sync, slave);
        sync.physical_start_address = slave->sii.boot_tx_mailbox_offset;
        sync.control_register = 0x22;
        sync.enable = 1;
        ec_sync_page(&sync, 1, slave->sii.boot_tx_mailbox_size,
                EC_DIR_INVALID, // use default direction
                0, // no PDO xfer
                datagram->data + EC_SYNC_PAGE_SIZE);
        slave->configured_tx_mailbox_offset =
            slave->sii.boot_tx_mailbox_offset;
        slave->configured_tx_mailbox_size =
            slave->sii.boot_tx_mailbox_size;
 
    } else if (slave->sii.sync_count >= 2) { // mailbox configuration provided
        ec_datagram_fpwr(datagram, slave->station_address, 0x0800,
                EC_SYNC_PAGE_SIZE * slave->sii.sync_count);
        ec_datagram_zero(datagram);
 
        for (i = 0; i < 2; i++) {
            ec_sync_page(&slave->sii.syncs[i], i,
                    slave->sii.syncs[i].default_length,
                    NULL, // use default sync manager configuration
                    0, // no PDO xfer
                    datagram->data + EC_SYNC_PAGE_SIZE * i);
        }
 
        slave->configured_rx_mailbox_offset =
            slave->sii.syncs[0].physical_start_address;
        slave->configured_rx_mailbox_size =
            slave->sii.syncs[0].default_length;
        slave->configured_tx_mailbox_offset =
            slave->sii.syncs[1].physical_start_address;
        slave->configured_tx_mailbox_size =
            slave->sii.syncs[1].default_length;
    } else { // no mailbox sync manager configurations provided
        ec_sync_t sync;
 
        EC_SLAVE_DBG(slave, 1, "Slave does not provide"
                " mailbox sync manager configurations.\n");
 
        ec_datagram_fpwr(datagram, slave->station_address, 0x0800,
                EC_SYNC_PAGE_SIZE * 2);
        ec_datagram_zero(datagram);
 
        ec_sync_init(&sync, slave);
        sync.physical_start_address = slave->sii.std_rx_mailbox_offset;
        sync.control_register = 0x26;
        sync.enable = 1;
        ec_sync_page(&sync, 0, slave->sii.std_rx_mailbox_size,
                NULL, // use default sync manager configuration
                0, // no PDO xfer
                datagram->data);
        slave->configured_rx_mailbox_offset =
            slave->sii.std_rx_mailbox_offset;
        slave->configured_rx_mailbox_size =
            slave->sii.std_rx_mailbox_size;
 
        ec_sync_init(&sync, slave);
        sync.physical_start_address = slave->sii.std_tx_mailbox_offset;
        sync.control_register = 0x22;
        sync.enable = 1;
        ec_sync_page(&sync, 1, slave->sii.std_tx_mailbox_size,
                NULL, // use default sync manager configuration
                0, // no PDO xfer
                datagram->data + EC_SYNC_PAGE_SIZE);
        slave->configured_tx_mailbox_offset =
            slave->sii.std_tx_mailbox_offset;
        slave->configured_tx_mailbox_size =
            slave->sii.std_tx_mailbox_size;
    }
 
    fsm->take_time = 1;
 
    fsm->retries = EC_FSM_RETRIES;
    fsm->state = ec_fsm_slave_config_state_mbox_sync;
}
```

总结一句：ENI 很重要，ENI 很重要，ENI 很重要。

唠叨一句：大多数 Slave 厂商用 TwinCat 配 ESI 跑测试，他才不管从站的 SII 对不对呢。