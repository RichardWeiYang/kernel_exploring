简单描述一下用户使用nvdimm设备前需要做的配置，主要分成两个方面

  * Dimm
  * Region
  * Namespace

这其实也就是对应了nvdimm上的三个概念，让我们一个个来熟悉。

# Dimm

Dimm就是真实的硬件，长得和内存差不多。这个的配置不由软件决定，而是由硬件系统插线决定的。

不过我们可以通过软件查看硬件的情况。

```
# ixpdimm-cli show -topology
MemoryType Capacity  DimmID PhysicalID DeviceLocator
...
AEP DIMM   125.6 GiB 0x0000 26         CPU1_DIMM_A1
AEP DIMM   125.6 GiB 0x0100 37         CPU1_DIMM_D1
...
```

这显示了现在有两个dimm是nvdimm，以及容量和设备号等情况。

嗯，暂时就知道这些，等学习了再回来补充。

# Region

在使用设备前，我们需要先创建Region。这玩意有点高级。

## 查看Region

```
# ndctl list -R
{
  "dev":"region0",
  "size":268435456000,
  "available_size":0,
  "type":"pmem",
  "numa_node":0,
  "iset_id":-600138270574898608,
  "persistence_domain":"unknown"
}
```

这里显示当前只有一个Region。

## 创建Region

创建有点讨厌，用命令行也行，但是好像会卡住。暂时只好在EFI中操作。不过命令行格式差不多。

在EFI的shell中输入，就可以创建一个只有PersistentMemoryType的Region了。

```
Shell> ApachePassCli.efi create –goal PersistentMemoryType=AppDirect
```

这个命令的完整格式是：

```
  [MemoryMode = (0|%)] [PersistentMemoryType = (AppDirect|AppDirectNotInterleaved)] [Reserved = (0|%)] [Config = (MM|AD|MM+AD)]
```

从这个命令中可以看出，Region可以有两种模式: Memory 和 PersistentMemory。

# Namespace

有了Region后，可以在Region中再建立Namespace。建立了Namespace之后，就可以使用nvdimm了。

## 查看Namespace

```
# ndctl list
{
  "dev":"namespace0.0",
  "mode":"fsdax",
  "size":264239054848,
  "uuid":"4c268f38-ee5d-4a3b-af4c-8285e08a4547",
  "raw_uuid":"e7d5888c-c1d2-405b-b304-6ad9d7b6d18a",
  "sector_size":512,
  "blockdev":"pmem0",
  "numa_node":0
}
```


## 创建Namespace

```
# ndctl create-namespace -f -e namespace0.0 -m devdax
{
  "dev":"namespace0.0",
  "mode":"devdax",
  "size":"246.09 GiB (264.24 GB)",
  "uuid":"7a928d1b-c74e-4028-987f-f7ef3eb1a9df",
  "raw_uuid":"aeeb2f3e-3a28-4b8f-bc4b-0106fe9f770d",
  "daxregion":{
    "id":0,
    "size":"246.09 GiB (264.24 GB)",
    "align":2097152,
    "devices":[
      {
        "chardev":"dax0.0",
        "size":"246.09 GiB (264.24 GB)"
      }
    ]
  },
  "numa_node":0
}
```
