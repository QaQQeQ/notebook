
---

### 一、了解: LVM 核心概念
LVM (Logical Volume Manager) 是在硬盘分区和文件系统之间的一个逻辑层，它允许用户对磁盘空间进行灵活的管理，解决了传统分区不易调整大小的难题。

*   **三大核心组件**:
    *   **PV (Physical Volume / 物理卷)**: LVM 的基本存储单元，通常是整个硬盘或一个标准的 Linux 分区。它是构建 LVM 的“砖块”。
    *   **VG (Volume Group / 卷组)**: 一个或多个 PV 组成的“存储池”。VG 将所有 PV 的空间整合起来，统一进行管理和分配。
    *   **LV (Logical Volume / 逻辑卷)**: 从 VG 中划分出的一块逻辑空间，相当于一个“虚拟分区”。LV 可以被格式化成文件系统并挂载使用。

*   **基本工作单元**:
    *   **PE (Physical Extent)**: 物理扩展单元。VG 中存储空间的最小划分单位，默认为 4MB。LV 的大小必须是 PE 的整数倍。



---

### 二、构建与拆除: LVM 的生命周期
此部分介绍从无到有创建 LVM 逻辑卷，以及反向拆除的完整流程。顺序至关重要。

*   **创建流程 (PV → VG → LV)**
    1.  **创建 PV**: 将物理分区或磁盘初始化为物理卷。
        *   **核心示例**: `pvcreate /dev/sdb1`
    2.  **创建 VG**: 使用一个或多个 PV 创建卷组。
        *   **核心示例**: `vgcreate myvg /dev/sdb1 /dev/sdc1`
    3.  **创建 LV**: 从指定的 VG 中划分出逻辑卷。
        *   **核心示例 (按大小)**: `lvcreate -L 10G -n mylv myvg`
        *   **核心示例 (按 PE 数量)**: `lvcreate -l 100 -n mylv myvg`
        *   **核心示例 (按 百分比)**:`lvcreate -l +100%FREE -n mylv myvg`

*   **查看状态**
    *   **简要查看**: `pvs`, `vgs`, `lvs`
    *   **详细查看**: `pvdisplay`, `vgdisplay`, `lvdisplay`

*   **拆除流程 (LV → VG → PV)**
    1.  **删除 LV**: 删除指定的逻辑卷。
        *   **核心示例**: `lvremove /dev/myvg/mylv`
    2.  **删除 VG**: 删除指定的卷组。
        *   **核心示例**: `vgremove myvg`
    3.  **删除 PV**: 移除物理卷标识。
        *   **核心示例**: `pvremove /dev/sdb1 /dev/sdc1`

---

### 三、扩容与迁移: LVM 的生产管理
此部分介绍 LVM 最核心的优势——在线调整存储空间和无缝迁移数据。

*   **扩容 LV (逻辑卷)**
    *   **功能**: 为一个已存在的 LV 增加容量。这是一个**两步过程**：先扩大 LV，再扩展其上的文件系统。
    *   **前提是：VG卷组还有剩余未分配空间**
    *   **核心示例 (增加2GB)**:
        ```bash
        # 1. 扩大逻辑卷
        lvextend -L +2G /dev/myvg/mylv
        lvextend -rL +2G /dev/myvg/mylv   #-r：扩容的同时扩展文件系统
        # 2. 扩展文件系统 (ext4示例)
        resize2fs /dev/myvg/mylv
        # (xfs 文件系统使用 xfs_growfs)
        ```
    *   **核心示例 (扩展到 VG 的全部剩余空间)**:
        ```bash
        lvextend -l +100%FREE /dev/myvg/mylv
        ```

*   **扩容 VG (卷组)**
    *   **功能**: 向 VG 中添加新的 PV，以增加整个存储池的容量。
    *   **核心示例**:
        ```bash
        # 1. 将新磁盘/分区创建为 PV
        pvcreate /dev/sdd1
        # 2. 将新 PV 添加到现有 VG
        vgextend myvg /dev/sdd1
        ```
*   **扩容 PV (物理卷)**
    *   **功能**: 当底层物理分区或磁盘本身被扩大后，让 LVM 识别到 PV 的新增空间。
    *   **核心示例**: `pvresize /dev/sdb1`

*   **更换/迁移 PV (物理卷)**
    *   **功能**: 在 VG 保持在线的情况下，将一个旧 PV 上的所有数据无缝迁移到新 PV 上，然后安全移除旧 PV。
    *   **核心流程**:
        1.  **准备新 PV**: `pvcreate /dev/sde1`
        2.  **扩容 VG**: `vgextend myvg /dev/sde1` (将新 PV 加入卷组)
        3.  **迁移数据**: `pvmove /dev/sdb1 /dev/sde1` (将旧 PV 的数据移到新 PV)
        4.  **缩减 VG**: `vgreduce myvg /dev/sdb1` (从卷组中移除旧 PV)
        5.  **移除 PV**: `pvremove /dev/sdb1` (移除旧 PV 的 LVM 标记)

---

### 四、限制: LV 的最大容量
逻辑卷的最大容量受限于两个主要因素。

*   **制约因素 1: PE 的大小**
    *   LV 的最大容量 = PE 的大小 × PE 的最大数量。
    *   PE 大小在创建 VG 时通过 `-s` 选项设定，一旦设定无法更改。
    *   **示例**:
        *   若 PE=4MB (默认)，则 LV 最大为 256GB。
        *   若 PE=16MB，则 LV 最大为 1TB。

*   **制约因素 2: PE 的数量**
    *   PE 的总数受到 LVM 版本和操作系统内核架构的限制。
    *   **主流 lvm2 版本**:
        *   **32位内核**: LV 最大容量限制为 16TB。
        *   **64位内核**: LV 最大容量限制为 8EB ，远超当前实际需求。