# NewPciNetDevice()
## 函数功能说明
NewPciNetDevice 函数是一个工厂函数，用于创建并初始化一个 PCI 网络设备对象。

这个函数是 Kubernetes RDMA 共享设备插件的核心组件，负责将物理 PCI 设备信息转换为插件可用的设备对象，为后续的设备发现、筛选和资源分配提供基础数据。

该函数的主要功能包括：

1. 设备信息收集：从 PCI 设备对象中提取基本信息（PCI地址、厂商ID、设备ID）
2. 网络接口识别：通过 PCI 地址获取对应的网络接口名称
3. 驱动信息获取：查询当前绑定到 PCI 设备的驱动程序
4. 链路类型检测：通过 netlink 获取网络接口的封装类型
5. RDMA 设备规范验证：获取并验证 RDMA 设备的规格信息
6. 对象构造：将所有收集到的信息封装到 pciNetDevice 结构体中返回

## 函数调用图
```
NewPciNetDevice (pci_net_device.go:41)
├── utils.GetNetNames(pciAddr) (utils.go:68)
│   └── os.Lstat(netDir) (系统调用)
│   └── os.ReadDir(netDir) (系统调用)
├── utils.GetPCIDevDriver(pciAddr) (utils.go:85)
│   └── os.Readlink(driverLink) (系统调用)
├── nLink.LinkByName(ifName) (types.go:138)
│   └── netlink.LinkByName() (第三方库)
├── rds.Get(pciAddr) (types.go:124)
└── rds.VerifyRdmaSpec(rdmaSpec) (types.go:125)
```

## 详细分析
### 输入参数
- dev *ghw.PCIDevice：PCI 设备信息对象
- rds types.RdmaDeviceSpec：RDMA 设备规范接口
- nLink types.NetlinkManager：网络链接管理接口
### 处理流程
1. 网络接口名称获取：

- 调用 utils.GetNetNames(pciAddr) 从 /sys/bus/pci/devices/{pciAddr}/net/ 目录读取网络接口名称
- 如果找到多个接口，使用第一个并记录警告
2. 驱动程序信息获取：

- 调用 utils.GetPCIDevDriver(pciAddr) 从 /sys/bus/pci/devices/{pciAddr}/driver 的符号链接获取驱动名称
- 如果未找到驱动，记录警告并返回 nil
3. 链路类型检测：
- 如果存在网络接口名称，通过 nLink.LinkByName(ifName) 获取网络链接信息
- 提取链接的封装类型（如 Ethernet、InfiniBand 等）

4. RDMA 规范验证：

- 调用 rds.Get(pciAddr) 获取 RDMA 设备规范
- 如果未找到规范，记录警告并返回 nil
- 调用 rds.VerifyRdmaSpec(rdmaSpec) 验证规范完整性
5. 对象构造：

将所有收集到的信息封装到 pciNetDevice 结构体
返回实现的 types.PciNetDevice 接口

# rds.Get(pciAddr) rds.VerifyRdmaSpec(rdmaSpec)
```go
rdmaSpec := rds.Get(pciAddr)
if err := rds.VerifyRdmaSpec(rdmaSpec); err != nil {
    return nil, fmt.Errorf("missing RDMA device spec for device %s, %v", pciAddr, err)
}
```
1. 获取RDMA设备规范：

- 调用 rds.Get(pciAddr) 方法，根据PCI地址获取对应的RDMA设备规范
- 返回一个 []*pluginapi.DeviceSpec 切片，包含RDMA设备在容器中的挂载路径信息
2. 验证RDMA规范完整性：

- 调用 rds.VerifyRdmaSpec(rdmaSpec) 方法，验证获取到的RDMA设备规范是否完整
- 检查是否包含所有必需的RDMA设备（如 rdma_cm、umad、uverbs 等）

## 函数调用图
```
NewPciNetDevice (pci_net_device.go:72-75)
├── rds.Get(pciAddr) (rdma_device_spec.go:42)
│   └── utils.GetRdmaDevices(pciAddress) (utils.go:47)
│       ├── rdmamap.GetRdmaDevicesForPcidev(pciAddress) (第三方库)
│       └── rdmamap.GetRdmaCharDevices(resource) (第三方库)
│   └── 构造pluginapi.DeviceSpec对象
│       ├── HostPath: RDMA设备在宿主机上的路径
│       ├── ContainerPath: RDMA设备在容器中的挂载路径  
│       └── Permissions: "rwm" (读、写、管理权限)
└── rds.VerifyRdmaSpec(rdmaSpec) (rdma_device_spec.go:52)
    └── containsRdmaDev(rdmaSpec, rdmaDev) (rdma_device_spec.go:62)  # rdmaDev为rds.rdmaDevs的内容
        ├── path.Split(devSpec.HostPath) (标准库)
        └── strings.Contains(devSpecName, rdmaDev) (标准库)
```

## 关键组件说明
1. RDMA设备规范获取 (rds.Get)
输入：PCI设备地址（如 0000:01:00.0）
处理：通过 utils.GetRdmaDevices 查询该PCI设备对应的RDMA字符设备
输出：pluginapi.DeviceSpec 数组，用于Kubernetes设备插件的设备挂载配置
2. RDMA规范验证 (rds.VerifyRdmaSpec)
验证逻辑：检查获取的RDMA设备规范是否包含所有必需的RDMA设备
必需设备：基于 requiredRdmaDevices 列表（rdma_cm、umad、uverbs 等）
匹配方式：通过设备名称的部分匹配（如 /dev/infiniband/rdma_cm 包含 rdma_cm）
3. 错误处理机制
验证失败场景：当PCI设备缺少必需的RDMA设备支持时
错误信息格式："missing RDMA device spec for device {pciAddr}, {error}"
影响：设备创建失败，该PCI设备不会被注册为可用的RDMA设备资源

# rds.Get(pciAddr)
```go
func (rf *rdmaDeviceSpec) Get(pciAddress string) []*pluginapi.DeviceSpec {
	// 用于存储RDMA设备规范
	deviceSpec := make([]*pluginapi.DeviceSpec, 0)

	// 根据PCI地址获取对应的RDMA设备路径列表
	rdmaDevices := utils.GetRdmaDevices(pciAddress)
	// 构建设备规范对象：
	for _, device := range rdmaDevices {
		deviceSpec = append(deviceSpec, &pluginapi.DeviceSpec{
			HostPath:      device,
			ContainerPath: device,
			Permissions:   "rwm"})
	}

	return deviceSpec
}
```

## GetRdmaDevices(pciAddress string) []string
```go
// GetRdmaDevices return rdma devices for given device pci address
func GetRdmaDevices(pciAddress string) []string {
	// 调用第三方库 rdmamap 的 GetRdmaDevicesForPcidev 函数
	// 根据PCI地址（如 0000:01:00.0）获取该PCI设备对应的RDMA资源列表
	// 返回的是RDMA资源标识符（如 mlx5_0、mlx5_1 等）
	rdmaResources := rdmamap.GetRdmaDevicesForPcidev(pciAddress)

	// 初始化设备列表：
	// 创建一个空的字符串切片，容量预设为RDMA资源的数量
	// 使用预设容量可以提高性能，避免多次内存分配
	rdmaDevices := make([]string, 0, len(rdmaResources))
	for _, resource := range rdmaResources {
		// 对每个RDMA资源（如 mlx5_0），调用 rdmamap.GetRdmaCharDevices 函数
		// 获取该资源对应的RDMA字符设备路径列表
		rdmaResourceDevices := rdmamap.GetRdmaCharDevices(resource)
		rdmaDevices = append(rdmaDevices, rdmaResourceDevices...)
	}

	return rdmaDevices
}
```

### 系统级工作原理
这个函数实际上是通过查询Linux系统的以下位置来获取RDMA设备信息：

1. PCI设备到RDMA资源的映射：

- 查询 /sys/bus/pci/devices/{pciAddress}/infiniband/ 目录
- 获取该PCI设备对应的InfiniBand设备名称
2. RDMA字符设备发现：

- 查询 `/sys/class/infiniband/{resource}/device/infiniband_verbs/` 目录
- 获取uverbs字符设备路径
- 查询 `/dev/infiniband/` 目录获取其他RDMA字符设备

### 调用关系图
```
GetRdmaDevices(pciAddress)
├── rdmamap.GetRdmaDevicesForPcidev(pciAddress)
│   └── 查询 /sys/bus/pci/devices/{pciAddress}/infiniband/
│   └── 返回RDMA资源标识符列表
└── 遍历每个资源:
    └── rdmamap.GetRdmaCharDevices(resource)
        ├── 查询 /sys/class/infiniband/{resource}/device/infiniband_verbs/
        ├── 查询 /dev/infiniband/ 目录
        └── 返回字符设备路径列表
```

### GetRdmaDevicesForPcidev(pciAddress string) []string
```go
// Get list of RDMA devices for a pci device.
// When switchdev mode is used, there may be more than one rdma device.
// Example pcidevName: 0000:05:00.0,
// when found, returns list of devices one or more devices names such as
// mlx5_0, mlx5_10
// PciDevDir = "/sys/bus/pci/devices"
// RdmaClassName     = "infiniband"

// 查询 /sys/bus/pci/devices/{pciAddress}/infiniband/ 目录
// 获取该PCI设备对应的InfiniBand设备名称
func GetRdmaDevicesForPcidev(pcidevName string) []string {
	dirName := filepath.Join(PciDevDir, pcidevName, RdmaClassName)
	return getRdmaDevicesFromDir(dirName)
}
```
### rdmamap库函数 func GetRdmaCharDevices(rdmaDeviceName string) []string 
GetRdmaCharDevices 函数根据给定的RDMA设备名称（如 mlx5_0），查找并返回该设备对应的所有RDMA字符设备路径。函数会尝试获取5种不同类型的RDMA字符设备，并将成功找到的设备路径收集到一个列表中返回。

```go
// 关心的设备："rdma_cm", "umad", "uverbs"
func GetRdmaCharDevices(rdmaDeviceName string) []string {
	// 创建一个空的字符串切片，用于存储找到的RDMA字符设备路径
    var rdmaCharDevices []string

    // 查询相关字符设备路径
	ucm, err := getUcmDevice(rdmaDeviceName)
	if err == nil {
		rdmaCharDevices = append(rdmaCharDevices, ucm)
	}
	issm, err := getIssmDevice(rdmaDeviceName)
	if err == nil {
		rdmaCharDevices = append(rdmaCharDevices, issm)
	}
	umad, err := getUmadDevice(rdmaDeviceName)
	if err == nil {
		rdmaCharDevices = append(rdmaCharDevices, umad)
	}
	uverb, err := getUverbDevice(rdmaDeviceName)
	if err == nil {
		rdmaCharDevices = append(rdmaCharDevices, uverb)
	}
	rdmaCm, err := getRdmaUcmDevice()
	if err == nil {
		rdmaCharDevices = append(rdmaCharDevices, rdmaCm)
	}

	return rdmaCharDevices
}
```
#### 调用图
```
GetRdmaCharDevices(rdmaDeviceName)
├── getUcmDevice(rdmaDeviceName)
│   └── getCharDevice(rdmaDeviceName, ""/sys/class/infiniband_cm"", "ucm")
│       ├── 查询 /dev/infiniband/ucm 设备文件
│       └── 返回设备路径或错误
├── getIssmDevice(rdmaDeviceName)  
│   └── getCharDevice(rdmaDeviceName, "/sys/class/infiniband_mad", "issm")
│       ├── 查询 /dev/infiniband/issm* 设备文件
│       └── 返回设备路径或错误
├── getUmadDevice(rdmaDeviceName)
│   └── getCharDevice(rdmaDeviceName, "/sys/class/infiniband_mad","umad")
│       ├── 查询 /dev/infiniband/umad* 设备文件
│       └── 返回设备路径或错误
├── getUverbDevice(rdmaDeviceName)
│   └── getCharDevice(rdmaDeviceName, "/sys/class/infiniband_verbs","uverbs")
│       ├── 查询 /dev/infiniband/uverbs* 设备文件
│       └── 返回设备路径或错误
└── getRdmaUcmDevice()
    └── os.Stat("/dev/infiniband/rdma_cm")
    └── if info.Name() == "rdma_cm" {
		    return "/dev/infiniband/rdma_cm", nil
	    }   
```


#### 系统级查找过程
每个 get*Device 函数实际上会查询Linux系统的以下位置：
- /dev/infiniband/ 目录下的字符设备文件
- /sys/class/infiniband/{deviceName}/ 目录下的设备信息
- 系统设备映射表

### getCharDevice() 函数
```go
func getCharDevice(rdmaDeviceName, classDir, charDevPrefix string) (string, error) {
	// 打开classDir目录
    fd, err := os.Open(classDir)
	if err != nil {
		return "", err
	}
	defer fd.Close()

    // 读取目录中的所有文件和子目录
	fileInfos, err := fd.Readdir(-1)
	if err != nil {
		return "", nil
	}

    // 遍历目录中的每个文件/目录条目
	for i := range fileInfos {
        // 跳过当前目录(.)和上级目录(..)条目
		if fileInfos[i].Name() == "." || fileInfos[i].Name() == prevDir {
			continue
		}
        //检查文件名是否包含指定的字符设备前缀（如 "ucm", "issm", "umad" 等）
		if !strings.Contains(fileInfos[i].Name(), charDevPrefix) {
			continue
		}
        // 调用 isDirForRdmaDevice 函数检查该目录是否与指定的RDMA设备关联
        // 如果不关联，跳过该条目
		dirName := filepath.Join(classDir, fileInfos[i].Name())
		if !isDirForRdmaDevice(rdmaDeviceName, dirName) {
			continue
		}
        // 构造设备文件路径并返回：/dev/infiniband/{charDevPrefix}*
		deviceFile := filepath.Join("/dev/infiniband", fileInfos[i].Name())
		return deviceFile, nil
	}
	return "", fmt.Errorf("no ucm device found")
}
```
### 实际工作流程示例
假设调用：getCharDevice("mlx5_0", "/sys/class/infiniband_verbs", "uverbs")

1. 打开目录：打开 /sys/class/infiniband_verbs
2. 读取内容：获取目录中的所有条目
3. 遍历筛选：
- 跳过 . 和 ..
- 检查文件/目录名包含 "uverbs"
- 检查文件/目录是否与 "mlx5_0" 设备关联
4. 返回结果：找到匹配的设备，如 /dev/infiniband/uverbs0


# VerifyRdmaSpec()
```go
var requiredRdmaDevices = []string{"rdma_cm", "umad", "uverbs"}
// VerifyRdmaSpec verify rdma spec is complete
func VerifyRdmaSpec(rdmaSpec []*pluginapi.DeviceSpec) error {
	// 遍历每个RDMA设备规范
	for _, devSpec := range rdmaSpec {
		// 检查是否包含所有必需的RDMA设备
		if !containsRdmaDev(rdmaSpec, devSpec) {
			// 如果缺少任何必需的RDMA设备，返回错误
			return fmt.Errorf("missing RDMA device spec for device %s, %s", devSpec.HostPath, devSpec.ContainerPath)
		}
	}

	// 如果所有必需的RDMA设备都存在，返回 nil 表示验证通过
	return nil
}
```

## 具体实例说明

假设我们有一个PCI设备 0000:01:00.0，通过 rds.Get(pciAddr) 获取到的RDMA设备规范如下：
```go
rdmaSpec := []*pluginapi.DeviceSpec{
    {
        HostPath:      "/dev/infiniband/rdma_cm",
        ContainerPath: "/dev/infiniband/rdma_cm",
        Permissions:   "rwm"
    },
    {
        HostPath:      "/dev/infiniband/uverbs0", 
        ContainerPath: "/dev/infiniband/uverbs0",
        Permissions:   "rwm"
    },
    {
        HostPath:      "/dev/infiniband/umad0",
        ContainerPath: "/dev/infiniband/umad0", 
        Permissions:   "rwm"
    }
}

```

### 校验过程分解
1. 检查 rdma_cm 设备：
```go
// 遍历rdmaSpec中的每个设备规范
for _, devSpec := range rdmaSpec {
    // 提取设备名称：从 "/dev/infiniband/rdma_cm" 提取 "rdma_cm"
    _, devSpecName := path.Split(devSpec.HostPath)
    // 检查是否包含 "rdma_cm"
    if strings.Contains(devSpecName, "rdma_cm") {
        // 找到匹配，继续检查下一个必需设备
        break
    }
}
```
2. 检查 uverbs0 设备：类似
3. 检查 umad0 设备：类似
