---
title: DiagnosticOverIP 介绍
date: 2018-02-26 10:05:20
tags: [UDS, DoIP, AUTOSAR, ARCCORE]
categories: UDS
---
# 介绍
## 使用场景
1. 车辆的检查和维修。  
2. 车辆/ECU的软件更新。  
3. 车辆/ECU下线检测和维修。  

## 术语
### DoIP edge node
DoIP边缘节点，是车内网络连接到诊断口的第一个节点，连接Ethernet Activation Line。

### DoIP entity
DoIP实体，包括DoIP节点和DoIP网关。

### DoIP gateway
DoIP网关，支持DoIP消息的路由。

### DoIP Node
DoIP节点，支持DoIP协议但没有路由功能。

### Network Node
普通网络节点，不支持DoIP协议。

## 协议集
|OSI layers|Vehicle manufacturer enhanced diagnostics|
|--|--|
|Application|ISO 14229-1/ISO 14229‑5|
|Presentation|Vehicle manufacturer specific|
|Session|ISO 14229-2|
|Transport|ISO 13400-2|
|Network|ISO 13400-2|
|Data link|ISO 13400-3|
|Physical|ISO 13400-3|

## 网络架构
对ECU的诊断有以下几种连接方式：  

1. 测试设备与DoIP网关相连，网关将UDS消息转发到传统网络节点（如CAN）；
2. 测试设备直接与DoIP节点相连。测试设备通过网关认证后，网关开启直连的通路;
3. 测试设备与DoIP网关相连，通过网关认证后网关将DoIP消息完整的路由到其他DoIP节点。  
    
通过DoIP直接访问ECU可以获得最好的性能，因为没有网关的制约。缺点是存在一定的安全隐患，一旦连接开启，其他测试设备也能访问到车内节点。通过网关路由可以在网关上增加安全策略，但网关的性能将影响到诊断的表现。


# DoIP协议                    
## Intenet Protocol(IP)
IP协议包括IPv6和IPv4，协议推荐使用IPv6，IPv4用于兼容旧的网络。

## ARP和NDP
如果使用IPv4，那么需要支持ARP(AddressResolutionProtocol);  
如果使用IPv6, 那么需要支持NDP(NeighbourDiscoveryProtocol)。  

## ICMP
ICMP(InternalControlMessageProtocol)属于IP协议簇的一部分，它用来发送一些错误消息，比如host无法连接或者请求的服务不可用。

## TCP
TCP端口号: **13400**。 
测试设备发送Request的源端口动态分配，目标端口13400；  
DoIP节点发送Response的源端口为13400，目标端口为Request的源端口（动态分配）。  

## UDP
测试设备请求车辆信息或者发送控制命令时目标端口为13400, 源端口动态分配。  
DoIP节点发送Response消息时，目标端口动态分配(即对应Request的源端口），源端口可以设置为13400也可以动态分配。  
DoIP节点发送非Response消息时（如Vehicle Announcement message), 目标端口使用13400，源端口可以设置为13400也可以动态分配。  

## DHCP
所有DoIP节点都不应作为DHCP Server，而是作为DHCP Client（DHCPv6 Client)。
Host名称应按照格式”DoIP-XXXXX"定义，XXXXX部分根据厂商要求设置。  
DoIP节点应该在ActivationLine激活时启动IP地址分配程序。  

## 数据传输顺序
DoIP消息的数据使用大端模式传输，即高字节在前，低字节在后。  

## 消息格式
### DoIP Generic Header
所有的DoIP消息包含一个8个字节长度的消息头DoIP Header:  

|Item|Position(Byte)|Length(Byte)|
|--|--|--|
|Protocol version|0|1|
|Inverse protocol version |1|1|
|Payload type|2|2|
|Payload length|4|4|
|Payload type specific message content|8|...|
  
Payload type的定义(接收）：  

|PayloadType value|PayloadType name|Connection Kind|
|--|--|--|
|0x0001|Vehicle Identification request message|UDP|
|0x0002|Vehicle identification request with EID|UDP|
|0x0003|Vehicle identification request message with VIN|UDP|
|0x0005|Routing activation request|TCP|
|0x0008|Alive Check response|TCP|
|0x4001|DoIP entity status request|UDP|
|0x4003|Diagnostic power mode information request|UDP|
|0x8001|Diagnostic message|TCP|
  
Payload type的定义(发送）：  

|PayloadType value|PayloadType name|Connection Kind|
|--|--|--|
|0x0000|Generic DoIP header negative acknowledge|UDP/TCP|
|0x0004|Vehicle announcement message/vehicle identification response|UDP|
|0x0006|Routing activation response|TCP|
|0x0007|Alive Check request|TCP|
|0x4002|DoIP entity status response|UDP|
|0x4004|Diagnostic power mode information response|UDP|
|0x8002|Diagnostic message positive acknowledgement|TCP|
|0x8003|Diagnostic message negative acknowledgement|TCP|
  
## 消息说明
### Vehicle Information Request
测试设备通过发送VehicleInformationRequest来获取网络上DoIP节点的信息，有三种方式：
1. 不带参数(PayloadType=0x0001)。
2. 带EID(比如使用MAC地址）。
3. 带VIN。

### Vehicle announcement/vehicle identification response message
DoIP节点分配到IP地址后需要以一定间隔（A\_DoIP\_Announce\_Interval）发送VehicleAnnouncement一定次数（A\_ DoIP\_Announce\_Num）。  
DoIP节点收到VehicleInformationReqeust后延迟一定时间（A\_DoIP\_Announce\_Wait）后发送VehicleIdentificationResponseMessage。  
这两个消息的Payload数据定义是一样的，区别在于前者目标端口使用13400，广播发送；后者为点对点发送，目标端口为request的源端口。
DoIP发送VehicleAnnoucement时IP地址应设置为广播地址, FF02::1(IPv6)。  

### Routing activation request and response
Routing activation request用于建立诊断设备与DoIP节点的连接关系。诊断设备将自己的逻辑地址SA发送给DoIP节点，DoIP节点把SA与socket连接进行关联，后面才能进行诊断服务。

### Diagnostic message and diagnostic message acknowledgement
DiagnosticMessage是用于承载诊断数据的帧，数据包括测试设备源地址，目标节点地址和诊断服务数据（如22 F1 90)。测试设备发送诊断请求和DoIP节点发送诊断响应都使用DiagnosticMessage。  
DoIP节点接收到DiagnosticMessage并且检验通过后要发送ACK应答DiagnosticMessageAcknowledgement，应答数据包括DoIP节点源地址，测试设备目标地址和ACK/NACK，以及可选的诊断服务数据（与服务请求一致）。注意，这个应答不是诊断服务请求的响应数据，仅仅是作为接收到请求的应答。    

### Alive check request and alive check response
用于检查某个TCP socket是否存在。

### Diagnostic power mode information request and response
测试设备用来获取车辆是否处于Diagnostic power mode。  

### DoIP entity status information request and response
用于获取DoIP节点的一些信息，比如：
- 连接的节点是DoIP网关还是普通的DoIP节点。
- 允许的最大TCP连接数。
- 当前TCP连接数。
- 可以处理的最大请求数据长度。
  
## Socket处理
测试设备和DoIP节点之间的连接由逻辑地址和socket句柄唯一确定。Socket句柄包含源/目标的IP地址、端口号和协议类型（TCP/UDP)。

### 连接状态
- Socket建立后处于“initialized”状态。
- 接收到RoutineActivationMessage后，进入“Registered[Pending for Authentication]"状态，同时测试设备的SA与连接进行关联。  
- 完成认证机制或者不需要认证，进入“Registered [Pending for Confirmation]”状态。  
- 完成确认机制或者不需要确认，进入“Registered [Routing Active]”状态。在进入“Registered [Routing Active]”状态前不响应其他DoIP消息。  
- 如果前面步骤超时或者认证、确认失败，进入“Finalize”状态。关闭TCP连接，进入监听状态。  

## 逻辑地址
一个逻辑地址对应一个DoIP节点。测试设备通过**VehicleDiscoveryProcess**可以把逻辑地址与IP地址关联在一起。对于功能寻址，没有机制将IP地址与多个IoIP节点映射起来，所以发送功能请求时需要测试设备自己向各个IP发送数据包。

# DoIP诊断
DoIP诊断的基本步骤如下（以直连为例，不考虑网关认证）：  
1. 测试设备与DoIP节点物理连接，(激活ActivationLine)；
2. DHCP服务分配IP地址获取IP-AutoConfiguration功能生成地址；
3. 车内ECU发出VehicleIdentification或者测试设备主动请求VehicleIdentification;
4. 测试设备与指定节点建立TCP连接；
5. 测试设备通过TCP连接发送RoutingActivation消息；
6. 测试设备通过TCP连接发送DiagnosticMessage(Request)；
7. DoIP节点收到请求后回复DiagnosticMessageAck;
8. DoIP节点发送响应数据DiagnosticMessage(Response);  
  
## 建立连接和车辆发现
### 直接连接
测试设备和节点直接相连。  
假设没有DHCP Server存在，DHCP程序不能获取IP地址，这种情况下Auto-IP机制可以使得各自分配一个有效的本地IP地址。  
DoIP节点配置完IP地址后，开始广播VehicleAnnoucementMessage，发送自身的VIN，EID, GID和逻辑地址。通过UDP发送，目标端口号13400，发送3次。  
测试设备的IP地址分配有可能比DoIP节点晚，导致测试设备可能没有收到DoIP节点初始发出的广播消息，这时测试设备可以通过发送VehicleIdentificationRequestMessage来获取信息。
  
### 网关连接
这种情况下测试设备的物理连接通常是后接入，很可能没有接收到DoIP节点的VehicleAnnoucement，所以需要发送VehicleIdentificationRequest来查询。

## DoIP会话
建立通信的第一步是创建一个socket，目标端口是13400。同时，DoIP节点需要准备好接收请求的资源。  
创建socket后，开始启动相关的超时定时器，等待接收RoutingActivationMessage，其他消息不处理。此时连接处于“initialize"状态。  
要激活initialize状态下的连接，测试设备需要发送RoutingActivationMessage。假设测试设备合格且不需要其他认证，连接进入“registered [Routing Active]”状态，此状态下就可以处理DiagnosticMessage了。  
DoIP节点接收到任何数据后，先检查DoIP消息头，如果Payload是DiagnosticMessage，则执行诊断服务程序。  
  

# 软件架构
AUTOSAR架构中DoIP的上下层级关系为：  
DCM -> PduR -> **DoIP** -> SocketAdaptor -> TCP/IP -> EthernetInterface -> EthernetDriver  

- SocketAdaptor(SoAd): SocketAdaptor作为TCP/IP协议栈的抽象层，提供Socket通信功能。
- PduR：负责DoIP模块和DCM模块之间数据的路由。  

## DoIP模块 
### 建立通讯
1. 控制DoIP Activation Line的状态，`DoIP_ActivationLineSwitch`。没有Activate时需要忽略PduR或者SoAd的数据。 
2. 状态从Inactive切换到Active时，调用`SoAd_GetSoConId`获取SocketId，然后调用`SoAd_RequestIpAddrAssignment`两次来请求分配IP地址，SoConId使用获得的SocketId，LocalIpAddrPtr设置为NULL_PTR, type第一次设为TCPIP_IPADDR_ASSIGNMENT_LINKLOCAL_DOIP，第二次设为TCPIP_IPADDR_ASSIGNMENT_DHCP。  
3. 状态从Active切换到Inactive时，调用`SoAd_GetSoConId`和`SoAd_CloseSoCon`关闭所有Socket通信。  
4. 所有UDP socket连接关闭后，调用`SoAd_ReleaseIpAddrAssignment`释放IP地址。  

