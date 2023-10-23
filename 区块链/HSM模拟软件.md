# HSM

## 1. 首次构建镜像

1. 构建Ubuntu镜像，安装java

```
#Dockerfile
FROM docker.io/ubuntu:20.04
RUN export DEBIAN_FRONTEND=noninteractive && apt-get update -y && apt-get install -y supervisor cmake wget curl git vim build-essential openjdk-11-jdk
```

2. 启动Ubuntu

```
#startup.sh
docker run -d -v /data/hsm:/root/hsm/  --name hsm  -it hsm bash
```

3. 进入容器

```
docker exec -it 019f2c8d8f74 /bin/bash
```

4. 操作HSM

```
cd /root/hsm/SW_PTK_7.2.1_Client_RevA/SDKs/Linux64
#启动安装程序
sh ./safeNet-install.sh

# 进入安装程序
3 install a package from this CD

# 安装以下部分
1 7.2.1 SafeNet ProtectToolkit C Runtime
2 7.2.1 SafeNet ProtectToolkit C SDK
3 7.2.1 SafeNet ProtectToolkit Java Runtime
4 7.2.1 SafeNet ProtectToolkit Java SDK
5 7.2.1 SafeNet ProtectToolkit FM SDK
6 5.6-2 Embedded Linux Development Kit (ELDK).
7 7.2.1 SafeNet HSM Net Server
8 7.2.1 SafeNet Network HSM Access Provider

#启动脚本
cd /opt/safenet/protecttoolkit7/ptk
source setvars.sh

#配置CLASSPATH
cd /root/hsm/SafeNet_tech/jcprov/samples
export CLASSPATH=/opt/safenet/protecttoolkit7/ptk/lib/jcprov.jar:`pwd`

#编译
javac BIP32KeyDerivation.java

#运行
java BIP32KeyDerivation -keyName test -create

#返回结果
Generating Keys "test" in slot 0 and 2
Done
Generating derived key :
03423A2E4B313D664AF184D880588A23AC3189283482A472B4992B0F2BDE3E7BC6


```



## 2. 测试用

```
#进入容器
docker exec -it 019f2c8d8f74 /bin/bash

#启动脚本
cd /opt/safenet/protecttoolkit7/ptk
source setvars.sh

#设置CLASSPATH
cd /root/hsm/SafeNet_tech/jcprov/samples
export CLASSPATH=/opt/safenet/protecttoolkit7/ptk/lib/jcprov.jar:`pwd`

#运行
java BIP32KeyDerivation -keyName test -create
```



## 3. 原理及源码

### 3.1 操作模式

- PCI模式

![PCI Operating Mode](https://www.thalesdocs.com/gphsm/ptk/protectserver3/docs/images/ps_ptk_images/operation_modes/pcie_mode.svg)

<img src="https://www.trentonsystems.com/hs-fs/hubfs/ssp8268 pcie slots diagram.jpg?width=750&name=ssp8268 pcie slots diagram.jpg" alt="PCIe Gen 4 vs. Gen 3 Slots, Speeds" style="zoom:50%;" />

- Network模式

![PSE Network Mode](https://www.thalesdocs.com/gphsm/ptk/protectserver3/docs/images/ps_ptk_images/operation_modes/network_mode_pse.svg)

- 软件模式

软件模拟器模式，在本地计算机上，无需访问硬件安全模块。

软件仿真器版本通常用作最终将使用 ProtectToolkit-C 硬件变体的应用程序的开发和测试环境。



### 3.2 BIP32

```
java BIP32KeyDerivation -keyName test -create
```

1. 首先会解析命令行参数，根据传入的参数来执行不同的操作。如果命令行中指定了 -create 参数，则会生成一个新的主密钥（Secret Key），否则将查找已生成的主密钥。

```
// PKCS #11, Cryptoki: 
CryptokiEx.C_CreateObject
```



2. 调用generateMasterKeyPair生成MasterKeyPair

```
// PKCS #11, Cryptoki: 
CryptokiEx.C_DeriveKey
```

- CKA.LABEL：标签，用于标识密钥对象。
- CKA.TOKEN：标识这是一个存储在令牌上的对象。
- CKA.SENSITIVE：标识这个密钥对象是否敏感。
- CKA.DERIVE：标识这个密钥对象是否可用于派生其他密钥。
- CKA.KEY_TYPE：指定密钥的类型为 BIP32 密钥。



3. 调用generateChildKeyPair生成childKeyPair

```
// PKCS #11, Cryptoki: 
CryptokiEx.C_DeriveKey

C_GetAttributeValue
```



### 3.3 EccDemo

使用了 Safenet.jcprov 库来操作 PKCS#11 接口（Cryptoki）并与硬件安全模块（HSM）进行交互。

1. 私钥生成

```
java EccDemo -g -n test
```

- var0：CK_SESSION_HANDLE，一个 PKCS#11 会话的句柄，用于与硬件安全模块（HSM）进行交互。
- var1：String，指定密钥对的名称，将作为标签（LABEL）保存在 HSM 中，用于后续查找密钥对象。
- var2：CK_OBJECT_HANDLE，传入方法的引用参数，用于接收生成的 ECC 公钥对象的句柄。
- var3：CK_OBJECT_HANDLE，传入方法的引用参数，用于接收生成的 ECC 私钥对象的句柄。
- var4：CK_MECHANISM，用于指定密钥对生成的机制，这里使用 CKM.EC_KEY_PAIR_GEN，表示 ECC 密钥对生成机制。
- var5：byte[]，表示选定的椭圆曲线参数的 DER 编码。这里使用 getDerEncodedNamedCurve 方法获取名为 "prime192v1" 的椭圆曲线的 DER 编码参数。
- var6：CK_ATTRIBUTE[]，用于定义 ECC 公钥对象的属性，包括类别（CLASS）、是否存储在令牌中（TOKEN）、是否敏感（SENSITIVE）、密钥类型（KEY_TYPE）、标签（LABEL）、是否支持验证（VERIFY）和 ECC 参数（EC_PARAMS）。这里的属性用于创建 ECC 公钥对象。
- var7：CK_ATTRIBUTE[]，用于定义 ECC 私钥对象的属性，包括类别（CLASS）、是否存储在令牌中（TOKEN）、是否敏感（SENSITIVE）、密钥类型（KEY_TYPE）、标签（LABEL）和是否支持签名（SIGN）。这里的属性用于创建 ECC 私钥对象。



```
// PKCS #11, Cryptoki: 
CryptokiEx.C_GenerateKeyPair
```

CryptokiEx.C_GenerateKeyPair：通过调用 Cryptoki 接口的 C_GenerateKeyPair 方法生成 ECC 密钥对。该方法接受 ECC 公钥和私钥的属性定义，以及机制（在本例中为 ECC 密钥对生成机制），并生成相应的 ECC 密钥对对象。



2. 签名

```
java EccDemo -n test
```

调用eccSign 方法用于对给定的数据进行 ECC 数字签名。

- var0：CK_SESSION_HANDLE，一个 PKCS#11 会话的句柄，用于与硬件安全模块（HSM）进行交互。

- var1：CK_OBJECT_HANDLE，表示 ECC 私钥对象的句柄。该 ECC 私钥将用于对数据进行签名。

- var2：byte[]，表示要签名的数据。

- var3：long，表示要签名的数据的长度。

- var5：CK_MECHANISM，用于指定签名的机制，这里使用 CKM.ECDSA，表示 ECC 数字签名机制。

- var7：LongRef，一个引用参数，用于接收签名结果的长度。

- CryptokiEx.C_SignInit：通过调用 Cryptoki 接口的 C_SignInit 方法初始化签名操作。该方法指定了 ECC 签名机制和使用的私钥。

- CryptokiEx.C_Sign：通过调用 Cryptoki 接口的 C_Sign 方法对指定的数据进行签名。在第一次调用时，var7 参数传入 null，用于获取签名结果的长度；第二次调用时，var8 参数传入一个合适大小的字节数组，用于接收签名结果。

- var8：byte[]，用于接收签名结果。在调用 C_Sign 方法后，var8 中将存储生成的 ECC 数字签名。



eccVerify 方法用于验证给定的签名和消息。

```
// PKCS #11, Cryptoki: 
CryptokiEx.C_Verify
```