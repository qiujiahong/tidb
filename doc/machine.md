# 机器准备


* 操作系统 CentOS	 7.3以上


## 服务器建议配置

开发环境  

| 组件 | CPU  | 内存   | 本地存储     | 网络     | 实例数量(最低要求)  |
|------|------|--------|--------------|--------|---------------------|
| TiDB | 8核+ | 16 GB+ | 无特殊要求   | 千兆网卡 | 1（可与 PD 同机器）   |
| PD   | 4核+ | 8 GB+  | SAS, 200 GB+ | 千兆网卡 | 1（可与 TiDB 同机器） |
| TiKV | 8核+ | 32 GB+ | SSD, 200 GB+ | 千兆网卡 | 3                   |

>  8c 32GB 200GB SSD三台


## 生产环境


| 组件 | CPU   | 内存   | 硬盘类型 | 网络              | 实例数量(最低要求) |
|------|-------|--------|----------|-----------------|--------------------|
| TiDB | 16核+ | 32 GB+ | SAS      | 万兆网卡（2块最佳） | 2                  |
| PD   | 4核+  | 8 GB+  | SSD      | 万兆网卡（2块最佳） | 3                  |
| TiKV | 16核+ | 32 GB+ | SSD      | 万兆网卡（2块最佳） | 3                  |
| 监控 | 8核+  | 16 GB+ | SAS      | 千兆网卡          | 1                  |

> 9台服务器


