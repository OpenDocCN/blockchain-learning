# 第九章 Hyperledger Fabric（V1.2）源码深度解析－使用 configtxgen 工具生成通道交易配置文件

### 生应用通道交易配置文件

返回至 hyperledger/fabric/common/tools/configtxgen/main 函数中，使用 if 进行判断，如果 outputChannelCreateTx 参数不为空，则说明在命令行中输入的命令是生成应用通道交易配置文件，则将获取到的概要配置信息的结构体对象，通道名称，文件保存路径及文件名称作为参数通过调用 doOutputChannelCreateTx 函数，生成指定的 ChannelTx 文件， 实现源码如下：

```go
if outputChannelCreateTx != "" {
        if err := doOutputChannelCreateTx(profileConfig, channelID, outputChannelCreateTx); err != nil {
            logger.Fatalf("Error on outputChannelCreateTx: %s", err)
        }
    } 
```

如果命令参数指定生成通道交易配置文件，则调用 doOutputChannelCreateTx 函数来实现，具体实现过程如下：

```go
// 解析者：Hanxiaodong
// QQ 群（专业 Fabric 交流群）：862733552
// 创建应用通道交易配置文件
func doOutputChannelCreateTx(conf *genesisconfig.Profile, channelID string, outputChannelCreateTx string) error {
    logger.Info("Generating new channel configtx")

    configtx, err := encoder.MakeChannelCreationTransaction(channelID, nil, nil, conf)
    if err != nil {
        return err
    }

    //将指定的内容序列化为一个 protobuf 消息之后将其写入到由 outputChannelCreateTx 变量代表（用户指定的文件）的文件中。即生成最终的目标文件。
    logger.Info("Writing new channel tx")
    err = ioutil.WriteFile(outputChannelCreateTx, utils.MarshalOrPanic(configtx), 0644)
    if err != nil {
        return fmt.Errorf("Error writing channel create tx: %s", err)
    }
    return nil
} 
```

在 doOutputChannelCreateTx 函数中，首先调用 MakeChannelCreationTransaction 函数创建通道的事务，然后对其进行序列化并只在在指定的文件中。创建通道的事务实现源码如下：

```go
// 用于创建通道的事务
func MakeChannelCreationTransaction(channelID string, signer crypto.LocalSigner, orderingSystemChannelConfigGroup *cb.ConfigGroup, conf *genesisconfig.Profile) (*cb.Envelope, error) {
    // 获取一个 common.ConfigUpdate 对象, orderingSystemChannelConfigGroup 为 nil
    newChannelConfigUpdate, err := NewChannelCreateConfigUpdate(channelID, orderingSystemChannelConfigGroup, conf)
    if err != nil {
        return nil, errors.Wrap(err, "config update generation failure")
    }
    // 进行序列化
    newConfigUpdateEnv := &cb.ConfigUpdateEnvelope{
        ConfigUpdate: utils.MarshalOrPanic(newChannelConfigUpdate),
    }

    if signer != nil {    // 创建应用通道配置时传递的 signer 值为 nil
        sigHeader, err := signer.NewSignatureHeader()
        if err != nil {
            return nil, errors.Wrap(err, "creating signature header failed")
        }

        newConfigUpdateEnv.Signatures = []*cb.ConfigSignature{{
            SignatureHeader: utils.MarshalOrPanic(sigHeader),
        }}

        newConfigUpdateEnv.Signatures[0].Signature, err = signer.Sign(util.ConcatenateBytes(newConfigUpdateEnv.Signatures[0].SignatureHeader, newConfigUpdateEnv.ConfigUpdate))
        if err != nil {
            return nil, errors.Wrap(err, "signature failure over config update")
        }

    }

    return utils.CreateSignedEnvelope(cb.HeaderType_CONFIG_UPDATE, channelID, signer, newConfigUpdateEnv, msgVersion, epoch)
} 
```

在 MakeChannelCreationTransaction 函数中，第一行便是调用 NewChannelCreateConfigUpdate 函数获取一个 common.ConfigUpdate 对象，参数 orderingSystemChannelGroup 为 nil

```go
// 解析者：Hanxiaodong
// QQ 群（专业 Fabric 交流群）：862733552
// 生成一个 ConfigUpdate，可以将它发送给 Orderer 以创建一个新通道。可选地，这个通道组可以传入 Orderer 系统通道，生成的 ConfigUpdate 将从该文件中提取适当的版本。
func NewChannelCreateConfigUpdate(channelID string, orderingSystemChannelGroup *cb.ConfigGroup, conf *genesisconfig.Profile) (*cb.ConfigUpdate, error) {
    if conf.Application == nil {
        return nil, errors.New("cannot define a new channel with no Application section")
    }

    if conf.Consortium == "" {
        return nil, errors.New("cannot define a new channel with no Consortium value")
    }

    // 只解析所配置的应用程序部分，并将其封装在通道组中
    ag, err := NewApplicationGroup(conf.Application)
    if err != nil {
        return nil, errors.Wrapf(err, "could not turn channel application profile into application group")
    }

    var template, newChannelGroup *cb.ConfigGroup

    if orderingSystemChannelGroup != nil {
        ......
    } else {    // 因为传递的 orderingSystemChannelGroup 为 nil，所以执行 else 部分
        newChannelGroup = &cb.ConfigGroup{
            Groups: map[string]*cb.ConfigGroup{
                channelconfig.ApplicationGroupKey: ag,    // 指定为应用程序部分的通道组
            },
        }

        // 假设 orgs 没有被修改
        template = proto.Clone(newChannelGroup).(*cb.ConfigGroup)
        template.Groups[channelconfig.ApplicationGroupKey].Values = nil
        template.Groups[channelconfig.ApplicationGroupKey].Policies = nil
    }
    // 进行计算
    updt, err := update.Compute(&cb.Config{ChannelGroup: template}, &cb.Config{ChannelGroup: newChannelGroup})
    if err != nil {
        return nil, errors.Wrapf(err, "could not compute update")
    }

    // 根据需要添加联盟名称，以便将通道创建到写入集中
    updt.ChannelId = channelID
    updt.ReadSet.Values[channelconfig.ConsortiumKey] = &cb.ConfigValue{Version: 0}
    updt.WriteSet.Values[channelconfig.ConsortiumKey] = &cb.ConfigValue{
        Version: 0,
        Value: utils.MarshalOrPanic(&cb.Consortium{    // 联盟信息
            Name: conf.Consortium,
        }),
    }

    return updt, nil
} 
```

NewChannelCreateConfigUpdate 函数中首先检查配置信息中的 Application 及 Consortium 是否指定，如果未指定相关信息，直接返回错误。否则调用 hyperledger/fabric/protos/common/configtx.go 源码文件中的 NewConfigGroup 函数，该函数定义了在应用程序逻辑中所涉及的组织，如链码，以及这些成员如何与 Orderer 交互。并将所有元素的 mod_policy 设置为“Admins”。实现源码如下：

```go
// 返回通道配置的应用程序组件。它定义了在应用程序逻辑中所涉及的组织，如链码，以及这些成员如何与 Orderer 交互。并将所有元素的 mod_policy 设置为“Admins”。
func NewApplicationGroup(conf *genesisconfig.Application) (*cb.ConfigGroup, error) {
    // 创建 Application 信息包含的 Version、Groups、Values、Policies、 ModPolicy 的 ConfigGroup 对象
    applicationGroup := cb.NewConfigGroup()
    if len(conf.Policies) == 0 {
        logger.Warningf("Default policy emission is deprecated, please include policy specificiations for the application group in configtx.yaml")
        addImplicitMetaPolicyDefaults(applicationGroup)
    } else {
        if err := addPolicies(applicationGroup, conf.Policies, channelconfig.AdminsPolicyKey); err != nil {
            return nil, errors.Wrapf(err, "error adding policies to application group")
        }
    }

    if len(conf.ACLs) > 0 {
        addValue(applicationGroup, channelconfig.ACLValues(conf.ACLs), channelconfig.AdminsPolicyKey)
    }

    if len(conf.Capabilities) > 0 {
        addValue(applicationGroup, channelconfig.CapabilitiesValue(conf.Capabilities), channelconfig.AdminsPolicyKey)
    }

    for _, org := range conf.Organizations {
        var err error
        applicationGroup.Groups[org.Name], err = NewApplicationOrgGroup(org)
        if err != nil {
            return nil, errors.Wrap(err, "failed to create application org")
        }
    }

    applicationGroup.ModPolicy = channelconfig.AdminsPolicyKey
    return applicationGroup, nil
} 
```

Application Org 的相关信息通过 for 循环中调用 NewApplicationOrgGroup 函数来完成，具体实现源码如下：

```go
// 返回通道配置的 Application Org 组件。它为组织定义了加密材料 (MSP)，以及它的 Anchor Peers 使用的 gossip 网络。并将所有元素的 mod_policy 设置为“Admins”。
func NewApplicationOrgGroup(conf *genesisconfig.Organization) (*cb.ConfigGroup, error) {
    mspConfig, err := msp.GetVerifyingMspConfig(conf.MSPDir, conf.ID, conf.MSPType)
    if err != nil {
        return nil, errors.Wrapf(err, "1 - Error loading MSP configuration for org %s: %s", conf.Name)
    }
    // 创建 ApplicationOrg 所包含的 Version、Groups、Values、Policies、 ModPolicy 的 ConfigGroup 对象
    applicationOrgGroup := cb.NewConfigGroup()
    if len(conf.Policies) == 0 {
        logger.Warningf("Default policy emission is deprecated, please include policy specificiations for the application org group %s in configtx.yaml", conf.Name)
        addSignaturePolicyDefaults(applicationOrgGroup, conf.ID, conf.AdminPrincipal != genesisconfig.AdminRoleAdminPrincipal)
    } else {
        if err := addPolicies(applicationOrgGroup, conf.Policies, channelconfig.AdminsPolicyKey); err != nil {
            return nil, errors.Wrapf(err, "error adding policies to application org group %s", conf.Name)
        }
    }
    addValue(applicationOrgGroup, channelconfig.MSPValue(mspConfig), channelconfig.AdminsPolicyKey)

    var anchorProtos []*pb.AnchorPeer
    for _, anchorPeer := range conf.AnchorPeers {
        anchorProtos = append(anchorProtos, &pb.AnchorPeer{
            Host: anchorPeer.Host,
            Port: int32(anchorPeer.Port),
        })
    }
    addValue(applicationOrgGroup, channelconfig.AnchorPeersValue(anchorProtos), channelconfig.AdminsPolicyKey)

    applicationOrgGroup.ModPolicy = channelconfig.AdminsPolicyKey
    return applicationOrgGroup, nil
} 
```

Application Org 的相关信息处理完毕，返回至 NewApplicationGroup 函数，返回获取到的 applicationGroup 结构对象，返回的 hyperledger/fabric/protos/common/configtx.pb.go/ConfigGroup 结构定义如下：

```go
// 用于保存配置的分层数据结构
type ConfigGroup struct {
    Version   uint64                   `protobuf:"varint,1,opt,name=version" json:"version,omitempty"`
    Groups    map[string]*ConfigGroup  `protobuf:"bytes,2,rep,name=groups" json:"groups,omitempty" protobuf_key:"bytes,1,opt,name=key" protobuf_val:"bytes,2,opt,name=value"`
    Values    map[string]*ConfigValue  `protobuf:"bytes,3,rep,name=values" json:"values,omitempty" protobuf_key:"bytes,1,opt,name=key" protobuf_val:"bytes,2,opt,name=value"`
    Policies  map[string]*ConfigPolicy `protobuf:"bytes,4,rep,name=policies" json:"policies,omitempty" protobuf_key:"bytes,1,opt,name=key" protobuf_val:"bytes,2,opt,name=value"`
    ModPolicy string                   `protobuf:"bytes,5,opt,name=mod_policy,json=modPolicy" json:"mod_policy,omitempty"`
}

// 配置数据的单个部分
type ConfigValue struct {
    Version   uint64 `protobuf:"varint,1,opt,name=version" json:"version,omitempty"`
    Value     []byte `protobuf:"bytes,2,opt,name=value,proto3" json:"value,omitempty"`
    ModPolicy string `protobuf:"bytes,3,opt,name=mod_policy,json=modPolicy" json:"mod_policy,omitempty"`
}

type ConfigPolicy struct {
    Version   uint64  `protobuf:"varint,1,opt,name=version" json:"version,omitempty"`
    Policy    *Policy `protobuf:"bytes,2,opt,name=policy" json:"policy,omitempty"`
    ModPolicy string  `protobuf:"bytes,3,opt,name=mod_policy,json=modPolicy" json:"mod_policy,omitempty"`
} 
```

NewChannelCreateConfigUpdate 执行完毕后 MakeChannelCreationTransaction 函数中获取到一个 common.ConfigUpdate 对象之后，对其进行序列化，得到一个 ConfigUpdateEnvelope 对象，最后调用 utils.CreateSignedEnvelope 函数创建所需类型的签名信封（hyperledger/fabric/protos/common/common.pb.go/Envelope 对象），并使用封装的 dataMsg（序列化后的 ConfigUpdateEnvelope 对象）对其签名。全部完成之后将对应数据通过调用 ioutil.WriteFile 函数输出到指定的文件中。