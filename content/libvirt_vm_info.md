Title: 使用Libvirt API获取虚拟机信息 
Date: 2014-02-11 16:37
Modified: 2015-02-11 16:37
Category: Cloud
Tags: libvirt, python
Slug: libvirt-api-for-vm-info 
Authors: Alex Yang
Summary: 使用Python libvirt sdk获取虚拟机信息 

#Libvirt API for VM information

-- <cite>杨雨, Alex Yang</cite>

##建立与libvirt的连接

    #!/usr/bin/env python
    # -*- coding: utf-8 -*-
    
    import libvirt
    import sys
    import pprint

    if __name__ == '__main__':
        conn = libvirt.openReadOnly('qemu:///system')
        if conn == None:
            print 'Failed to get libvirt connection.'
            sys.exit(1)
    
        for i in conn.listDomainsID():
            domain = conn.lookupByID(i)

##Domain Info

API
http://libvirt.org/html/libvirt-libvirt.html#virDomainGetInfo

Example

    info = domain.info()

Returns

    [1, 524288L, 524288L, 1, 12663460000000L]

 1. 运行状态，http://libvirt.org/html/libvirt-libvirt.html#virDomainState
 2. 最大允许使用的内存(KB)
 3. 使用的内存(KB)
 4. VCPU数量
 5. CPU时间（纳秒）

##CPU Stats

API
http://libvirt.org/html/libvirt-libvirt.html#virDomainGetVcpus

Example

    print domain.vcpus()

Returns

    ([(0, 1, 12000000000L, 3), (1, 1, 7320000000L, 1)], [(True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True), (True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True, True)])

(vcpu编号， vcpu状态， vcpu使用时间-纳秒， vcpu所绑定的物理cpu编号)
关于vcpu状态 http://libvirt.org/html/libvirt-libvirt.html#virVcpuState
后面的两个元组分别记录了vcpu1和vcpu2与物理cpu的affinity关系

##Memory

API
http://libvirt.org/html/libvirt-libvirt.html#virDomainMemoryStats

Example

    print domain.memoryStats()

Returns

    {'actual': 8388608L, 'rss': 8668016L}

ACTUAL - 当前的Balloon Memory值(KB)
RSS - Resident Set Size 实际使用物理内存（包含共享库占用的内存）(kB)

关于Balloon Memory
http://smilejay.com/2012/11/kvm-ballooning-overview/
https://www.usenix.org/legacy/event/osdi02/tech/full_papers/waldspurger/waldspurger_html/node6.html

关于返回值的描述
http://libvirt.org/html/libvirt-libvirt.html#virDomainMemoryStatTags

##Disk

###Disk List

Exmaple

    def listDiskDevices(domain):
        from lxml import etree
        tree = etree.fromstring(domain.XMLDesc(0))
    
        for device in filter(
            bool,
            [target.get("dev") for target in tree.findall('devices/disk/target')]):
            yield device
    
    for dev in listDiskDevices(domain):
        print dev

Output

    vda
    vdb

###Disk Info

API
http://libvirt.org/html/libvirt-libvirt.html#virDomainGetBlockInfo

Example

    for dev in listDiskDevices(domain):                                    
        print '%s: %s' % (dev, domain.blockInfo(dev, 0))

Returns

    vda: [21474836480L, 3056758784L, 3056758784L]
    vdb: [536870912000L, 536870912000L, 536870912000L]

[Capacity, Allocation, Physical]

 - Capacity, 磁盘容量(bytes)
 - Allocation, 逻辑上使用的容量大小(bytes)
 - Physical, 物理上的实际大小(bytes)

###Disk Stats

API
http://libvirt.org/html/libvirt-libvirt.html#virDomainBlockStats

Example

    for dev in listDiskDevices(domain):
        print '%s: %s' % (dev, domain.blockStats(dev))

Returns

    vda: (837102L, 8320963584L, 954472L, 11133748224L, -1L)
    vdb: (1652955L, 94059463680L, 53919378L, 476562894848L, -1L)

(读请求次数，读字节数，写请求次数，写字节数，错误次数)

##Network

###Interface List

Example

    def listNetInterfaces(domain):
        from lxml import etree
        tree = etree.fromstring(domain.XMLDesc(0))
        for iface in tree.findall('devices/interface'):
            target = iface.find('target')
            if target is not None:
                name = target.get('dev')
            else:
                continue
            mac = iface.find('mac')
            if mac is not None:
                mac_address = mac.get('address')
            else:
                continue
            fref = iface.find('filterref')
            if fref is not None:
                fref = fref.get('filter')
    
            params = dict((p.get('name').lower(), p.get('value')) 
                           for p in iface.findall('filterref/parameter'))
            yield name
        
    for iface in listNetInterfaces(domain):
        print iface

Output

    vnet6

###Interface Stats

API
http://libvirt.org/html/libvirt-libvirt.html#virDomainInterfaceStats

Example

Return

    vnet6: (262131567L, 1825647L, 0L, 0L, 171840338L, 730995L, 0L, 0L)

(rx_bytes, rx_packets, rx_errs, rx_drop, tx_bytes, tx_packets, tx_errs, tx_drop)

##参考
---

**vmitools**
https://code.google.com/p/vmitools/

**libvirt-api**
http://libvirt.org/html/libvirt-libvirt.html

**python-libvirt**
http://libvirt.org/python.html

**Libvirt Python API: CPU State Example**
http://shijicheng.info/blog/2012/10/28/libvirt-python-api/

**Libvirt Dev Guide**
http://libvirt.org/devguide.html
