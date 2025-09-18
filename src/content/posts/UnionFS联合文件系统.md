---
title: UnionFS联合文件系统
published: 2025-04-20
description: ''
image: ''
tags: [云原生,k8s,容器]
category: '笔记'
draft: false 
lang: ''
---
# UnionFS联合文件系统   
## 1. 什么是UnionFS?   
UnionFS即联合文件系统,它允许<u>将多个不同的目录（或文件系统）透明地**叠加（overlay）**在一起，形成一个统一的视图</u>。用户看到的是一个合并后的目录结构，而实际文件可能来自不同的底层目录或文件系统。   

## 2. UnionFS的特性   
1. 叠加挂载（Overlay Mount）
    - 将多个目录（称为分支或层）按优先级合并为一个虚拟目录。
    - 上层文件会覆盖下层同名文件，但下层文件仍然存在（除非被显式删除）。

2. 写时复制（Copy-on-Write, CoW）
    - UnionFS的只读层默认不可写，修改文件时会先复制到可写层，避免直接修改底层数据。
    
3. 分层结构（Layering）
    - 支持多层叠加（如 Docker 镜像的分层机制），每一层可以独立管理。
    
---------
**这些特性意味着UnionFS可以实现:**

1. **文件系统的分层和复用**：可以将一个基础的文件系统作为多个子文件系统的共同底层，然后在子文件系统中进行添加或修改，对外展示最终叠加层。   
2. **文件系统的跨平台和移植性**：可以把不同操作系统或设备上的文件系统联合在一起，实现数据的共享和访问。   
3. **高效存储与传输**：由于CoW特性，仅在只读文件被修改时会复制，多个上层可基于同一个下层，实现存储的复用。   
4. **数据的保护**：因为修改均在上层可读写层进行，不会影响下层原始数据。   

## 3. UnionFS中的各种概念，规则   

### 概念解析：

1. **下层（lowerdir）**：
   - **底层只读层**，通常包含基础文件（如 Docker 镜像的基础层）。
   - 所有文件初始状态下从这里读取。
   - **不可直接修改**：任何修改都会通过写时复制（CoW）转移到上层。
2. **上层（upperdir）**：
   - **可写层**，用户对文件的修改（增、删、改）会记录在这里。
   - 优先级高于下层：同名文件在上层时会覆盖下层的文件。
3. **合并视图（merged）**：
   - 用户看到的统一目录，是下层和上层的叠加结果。

### 覆盖规则：

- 读取文件：  
  - 优先从 `upperdir` 查找，若不存在则从 `lowerdir` 读取。

- 修改文件：
  - 若修改 `lowerdir` 中的文件，UnionFS 会触发**copy-up操作**将文件复制到`upperdir`，再修改副本（CoW）。

- 删除文件：
  - 在 `merged` 视图删除文件时，会在 `upperdir` 添加一个“白名单”标记（如 `char` 文件），隐藏下层文件而非真正删除。


## 4. **UnionFS 的各种实现**

- **OverlayFS**：Linux 内核原生支持的现代 UnionFS（性能更好，Docker 默认使用）。
- **AUFS（Advanced Multi-Layered Unification Filesystem）**：早期 Docker 使用的实现，现逐渐被 OverlayFS 取代。
- **传统UnionFS实现（如早期Union Mount方案）**：传统的 Unix 实现，功能较简单。
> 截至目前[^本文撰写时]，docker在绝大多数发行版中默认使用基于OverlayFS的Overlay2驱动   
> 


## 5. Docker的UnionFS结构设计   

docker截至目前[^本文撰写时]默认使用Overlay2驱动。`overlay2` 是 Docker 基于 OverlayFS 开发的**存储驱动（Storage Driver）**，专为容器镜像的分层管理优化。

### 1. Overlay2 的磁盘结构与工作流程

#### 1. **镜像与容器存储结构**

- 镜像层：每个镜像层对应```/var/lib/docker/overlay2/<layer-id>```目录，包含```diff```（实际文件）、```link```（短标识符）、```lower```（父层引用）等文件 。
- 容器层：容器运行时创建 ```/var/lib/docker/overlay2/<container-id>```目录，包含：
  - `upper`：可写层（Upperdir）
  - `merged`：联合挂载后的视图
  - ```work```：内部操作目录 。

------
这里补充图示方便理解(此处补充[来源](https://www.thebyte.com.cn/container/unionfs.html)，侵删)   

- bootfs（boot file system）：包含操作系统 bootloader 和 kernel。用户不能修改 bootfs，在内核启动后，bootfs 会被卸载。
- rootfs（root file system）：包含系统常见的目录结构，如/dev 、/lib、/proc、/bin、/etc/、/bin 等

![](https://www.thebyte.com.cn/assets/docker-filesystems-multilayer-f7adb11e.png)



#### 2. **符号链接优化**

- 短标识符目录（`l/`）：为避免文件路径过长（挂载路径常包含长哈希）导致系统限制（Linux 的 mount 命令参数总长度受限于 页面大小通常为 4KB），Overlay2 使用符号链接（如 ```l/ABC123 -> ../<layer-id>/diff```）简化层引用 。

#### 3. 读写行为的底层机制

1. **文件读取**
   - 容器进程访问文件时，OverlayFS 按以下顺序查找：
    ```txt
    容器层（upperdir） → 镜像层（lowerdir，从最上层向最底层查找）
    ```
   - **如果文件在可写层存在**：直接读取。

   - **如果文件仅在镜像层存在**：从对应的只读层读取。

2. **文件修改（CoW）**
   
   - 修改现有文件： 若文件来自镜像层（只读），Overlay2 会先将文件复制到可写层，再修改副本。原始镜像层文件保持不变。
   - **删除文件**： 在可写层创建**白名单标记文件**（如 `.wh.<filename>`），隐藏镜像层中的文件，而非实际删除。
3. **新建文件**
   
   - 直接写入容器的可写层（`upperdir`），无需与镜像层交互。

### 2. 为什么使用`overlay2`而不直接使用 OverlayFS？

1. **解决 OverlayFS 的早期限制**
   - 原始 OverlayFS 驱动（`overlay`）：早期 Docker 使用 ```overlay```驱动，但存在以下问题：
     - 最多支持 40 层镜像（硬编码限制）。
     - 对硬链接（hard link）和 inode 管理不完善，导致存储冗余。
   - `overlay2` 的改进:
     - 支持最多 128 层镜像，更符合容器镜像的分层需求。
     - 优化 inode 共享机制，减少存储占用。

2. **容器化场景的适配**
   - Docker 镜像的每一层对应一个 OverlayFS 的 `lowerdir`，而容器运行时会在最上层创建可写层（`upperdir`）。
   - `overlay2` 驱动通过智能合并多个 `lowerdir`，提升容器启动速度和文件操作性能。


[^本文撰写时间]:2025年4月20日
