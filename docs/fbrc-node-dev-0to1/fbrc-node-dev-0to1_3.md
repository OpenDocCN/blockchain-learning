# 第三章 配置基于 fabric-sdk-node 应用的网络信息

## 从零到壹构建基于 fabric-sdk-node 的项目开发实战之三

### 创建配置 fabric-sdk-node

#### network-config.yaml

进入项目的 `artifacts` 目录中，创建 `network-config.yaml` 文件并编辑

```js
$ cd $HOME/kevin-fabric-sdk-node/artifacts/
$ vim network-config.yaml
```

`network-config.yaml` 文件完整内容如下：

```js
---
#
# The network connection profile provides client applications the information about the target
# blockchain network that are necessary for the applications to interact with it. These are all
# knowledge that must be acquired from out-of-band sources. This file provides such a source.
#
name: "kevin-fabric-sdk-node"

#
# Any properties with an "x-" prefix will be treated as application-specific, exactly like how naming
# in HTTP headers or swagger properties work. The SDK will simply ignore these fields and leave
# them for the applications to process. This is a mechanism for different components of an application
# to exchange information that are not part of the standard schema described below. In particular,
# the "x-type" property with the "hlfv1" value example below is used by Hyperledger Composer to
# determine the type of Fabric networks (v0.6 vs. v1.0) it needs to work with.
#
x-type: "hlfv1"

#
# Describe what the target network is/does.
#
description: "Balance Transfer Network"

#
# Schema version of the content. Used by the SDK to apply the corresponding parsing rules.
#
version: "1.0"

#
# The client section will be added on a per org basis see org1.yaml and org2.yaml
#
#client:

#
# [Optional]. But most apps would have this section so that channel objects can be constructed
# based on the content below. If an app is creating channels, then it likely will not need this
# section.
#
channels:
  # name of the channel
  kevinkongyixueyuan:
    # Required. list of orderers designated by the application to use for transactions on this
    # channel. This list can be a result of access control ("org1" can only access "ordererA"), or
    # operational decisions to share loads from applications among the orderers.  The values must
    # be "names" of orgs defined under "organizations/peers"
    orderers:
      - orderer.kevin.kongyixueyuan.com

    # Required. list of peers from participating orgs
    peers:
      peer0.org1.kevin.kongyixueyuan.com:
        # [Optional]. will this peer be sent transaction proposals for endorsement? The peer must
        # have the chaincode installed. The app can also use this property to decide which peers
        # to send the chaincode install request. Default: true
        endorsingPeer: true

        # [Optional]. will this peer be sent query proposals? The peer must have the chaincode
        # installed. The app can also use this property to decide which peers to send the
        # chaincode install request. Default: true
        chaincodeQuery: true

        # [Optional]. will this peer be sent query proposals that do not require chaincodes, like
        # queryBlock(), queryTransaction(), etc. Default: true
        ledgerQuery: true

        # [Optional]. will this peer be the target of the SDK's listener registration? All peers can
        # produce events but the app typically only needs to connect to one to listen to events.
        # Default: true
        eventSource: true

      peer1.org1.kevin.kongyixueyuan.com:
        endorsingPeer: false
        chaincodeQuery: true
        ledgerQuery: true
        eventSource: false

      peer0.org2.kevin.kongyixueyuan.com:
        endorsingPeer: true
        chaincodeQuery: true
        ledgerQuery: true
        eventSource: true

      peer1.org2.kevin.kongyixueyuan.com:
        endorsingPeer: false
        chaincodeQuery: true
        ledgerQuery: true
        eventSource: false

    # [Optional]. what chaincodes are expected to exist on this channel? The application can use
    # this information to validate that the target peers are in the expected state by comparing
    # this list with the query results of getInstalledChaincodes() and getInstantiatedChaincodes()
    chaincodes:
      # the format follows the "cannonical name" of chaincodes by fabric code
      - mycc:v0

#
# list of participating organizations in this network
#
organizations:
  Org1:
    mspid: org1.kevin.kongyixueyuan.com

    peers:
      - peer0.org1.kevin.kongyixueyuan.com
      - peer1.org1.kevin.kongyixueyuan.com

    # [Optional]. Certificate Authorities issue certificates for identification purposes in a Fabric based
    # network. Typically certificates provisioning is done in a separate process outside of the
    # runtime network. Fabric-CA is a special certificate authority that provides a REST APIs for
    # dynamic certificate management (enroll, revoke, re-enroll). The following section is only for
    # Fabric-CA servers.
    certificateAuthorities:
      - ca-org1

    # [Optional]. If the application is going to make requests that are reserved to organization
    # administrators, including creating/updating channels, installing/instantiating chaincodes, it
    # must have access to the admin identity represented by the private key and signing certificate.
    # Both properties can be the PEM string or local path to the PEM file. Note that this is mainly for
    # convenience in development mode, production systems should not expose sensitive information
    # this way. The SDK should allow applications to set the org admin identity via APIs, and only use
    # this route as an alternative when it exists.
    adminPrivateKey:
      path: artifacts/channel/crypto-config/peerOrganizations/org1.kevin.kongyixueyuan.com/users/Admin@org1.kevin.kongyixueyuan.com/msp/keystore/ca72ee77552f81537f82c98f4d2abc6d6df1c94c8022f1a176c45e35a812496d_sk
    signedCert:
      path: artifacts/channel/crypto-config/peerOrganizations/org1.kevin.kongyixueyuan.com/users/Admin@org1.kevin.kongyixueyuan.com/msp/signcerts/Admin@org1.kevin.kongyixueyuan.com-cert.pem

  # the profile will contain public information about organizations other than the one it belongs to.
  # These are necessary information to make transaction lifecycles work, including MSP IDs and
  # peers with a public URL to send transaction proposals. The file will not contain private
  # information reserved for members of the organization, such as admin key and certificate,
  # fabric-ca registrar enroll ID and secret, etc.
  Org2:
    mspid: org2.kevin.kongyixueyuan.com
    peers:
      - peer0.org2.kevin.kongyixueyuan.com
      - peer1.org2.kevin.kongyixueyuan.com
    certificateAuthorities:
      - ca-org2
    adminPrivateKey:
      path: artifacts/channel/crypto-config/peerOrganizations/org2.kevin.kongyixueyuan.com/users/Admin@org2.kevin.kongyixueyuan.com/msp/keystore/77c2e4e04f4098e29b6143f18a378a20fce8af5151b7e93aafba327fc2256c30_sk
    signedCert:
      path: artifacts/channel/crypto-config/peerOrganizations/org2.kevin.kongyixueyuan.com/users/Admin@org2.kevin.kongyixueyuan.com/msp/signcerts/Admin@org2.kevin.kongyixueyuan.com-cert.pem

#
# List of orderers to send transaction and channel create/update requests to. For the time
# being only one orderer is needed. If more than one is defined, which one get used by the
# SDK is implementation specific. Consult each SDK's documentation for its handling of orderers.
#
orderers:
  orderer.kevin.kongyixueyuan.com:
    url: grpcs://localhost:7050

    # these are standard properties defined by the gRPC library
    # they will be passed in as-is to gRPC client constructor
    grpcOptions:
      ssl-target-name-override: orderer.kevin.kongyixueyuan.com

    tlsCACerts:
      path: artifacts/channel/crypto-config/ordererOrganizations/kevin.kongyixueyuan.com/orderers/orderer.kevin.kongyixueyuan.com/tls/ca.crt

#
# List of peers to send various requests to, including endorsement, query
# and event listener registration.
#
peers:
  peer0.org1.kevin.kongyixueyuan.com:
    # this URL is used to send endorsement and query requests
    url: grpcs://localhost:7051

    grpcOptions:
      ssl-target-name-override: peer0.org1.kevin.kongyixueyuan.com
    tlsCACerts:
      path: artifacts/channel/crypto-config/peerOrganizations/org1.kevin.kongyixueyuan.com/peers/peer0.org1.kevin.kongyixueyuan.com/tls/ca.crt

  peer1.org1.kevin.kongyixueyuan.com:
    url: grpcs://localhost:7056
    grpcOptions:
      ssl-target-name-override: peer1.org1.kevin.kongyixueyuan.com
    tlsCACerts:
      path: artifacts/channel/crypto-config/peerOrganizations/org1.kevin.kongyixueyuan.com/peers/peer1.org1.kevin.kongyixueyuan.com/tls/ca.crt

  peer0.org2.kevin.kongyixueyuan.com:
    url: grpcs://localhost:8051
    grpcOptions:
      ssl-target-name-override: peer0.org2.kevin.kongyixueyuan.com
    tlsCACerts:
      path: artifacts/channel/crypto-config/peerOrganizations/org2.kevin.kongyixueyuan.com/peers/peer0.org2.kevin.kongyixueyuan.com/tls/ca.crt

  peer1.org2.kevin.kongyixueyuan.com:
    url: grpcs://localhost:8056
    eventUrl: grpcs://localhost:8058
    grpcOptions:
      ssl-target-name-override: peer1.org2.kevin.kongyixueyuan.com
    tlsCACerts:
      path: artifacts/channel/crypto-config/peerOrganizations/org2.kevin.kongyixueyuan.com/peers/peer1.org2.kevin.kongyixueyuan.com/tls/ca.crt

#
# Fabric-CA is a special kind of Certificate Authority provided by Hyperledger Fabric which allows
# certificate management to be done via REST APIs. Application may choose to use a standard
# Certificate Authority instead of Fabric-CA, in which case this section would not be specified.
#
certificateAuthorities:
  ca-org1:
    url: https://localhost:7054
    # the properties specified under this object are passed to the 'http' client verbatim when
    # making the request to the Fabric-CA server
    httpOptions:
      verify: false
    tlsCACerts:
      path: artifacts/channel/crypto-config/peerOrganizations/org1.kevin.kongyixueyuan.com/ca/ca.org1.kevin.kongyixueyuan.com-cert.pem

    # Fabric-CA supports dynamic user enrollment via REST APIs. A "root" user, a.k.a registrar, is
    # needed to enroll and invoke new users.
    registrar:
      - enrollId: admin
        enrollSecret: adminpw
    # [Optional] The optional name of the CA.
    caName: ca-org1

  ca-org2:
    url: https://localhost:8054
    httpOptions:
      verify: false
    tlsCACerts:
      path: artifacts/channel/crypto-config/peerOrganizations/org2.kevin.kongyixueyuan.com/ca/ca.org2.kevin.kongyixueyuan.com-cert.pem
    registrar:
      - enrollId: admin
        enrollSecret: adminpw
    # [Optional] The optional name of the CA.
    caName: ca-org2
```

#### 网络配置注意事项

您可以通过直接编辑 network-config.yaml 文件或为备用目标网络提供其他文件来更改配置参数。该应用程序使用可选的环境变量“TARGET_NETWORK”来控制要使用的配置文件。例如，如果您在 Amazon Web Services EC2 上部署了目标网络，则可以添加文件“network-config-aws.yaml”，并将“TARGET_NETWORK”环境设置为“aws”。该应用程序将获取“network-config-aws.yaml”文件中的设置。

#### IP 地址和 PORT 信息

如果您选择通过为对等方和订购者硬编码 IP 地址和 PORT 信息来自定义 docker-compose yaml 文件，那么您还必须将相同的值添加到 network-config.yaml 文件中。需要调整 url 和 eventUrl 设置以匹配 docker-compose yaml 文件。

```js
peer1.org1.kevin.kongyixueyuan.com:
  url: grpcs://x.x.x.x:7056
  eventUrl: grpcs://x.x.x.x:7058
```

##### network-config-aws.yaml

```js
$ vim network-config-aws.yaml
```

`network-config-aws.yaml` 文件完整内容如下：

```js
---
#
# The network connection profile provides client applications the information about the target
# blockchain network that are necessary for the applications to interact with it. These are all
# knowledge that must be acquired from out-of-band sources. This file provides such a source.
#
name: "kevin-fabric-sdk-node"

#
# Any properties with an "x-" prefix will be treated as application-specific, exactly like how naming
# in HTTP headers or swagger properties work. The SDK will simply ignore these fields and leave
# them for the applications to process. This is a mechanism for different components of an application
# to exchange information that are not part of the standard schema described below. In particular,
# the "x-type" property with the "hlfv1" value example below is used by Hyperledger Composer to
# determine the type of Fabric networks (v0.6 vs. v1.0) it needs to work with.
#
x-type: "hlfv1"

#
# Describe what the target network is/does.
#
description: "Balance Transfer Network"

#
# Schema version of the content. Used by the SDK to apply the corresponding parsing rules.
#
version: "1.0"

#
# The client section will be added on a per org basis see org1.yaml and org2.yaml
#
#client:

#
# [Optional]. But most apps would have this section so that channel objects can be constructed
# based on the content below. If an app is creating channels, then it likely will not need this
# section.
#
channels:
  # name of the channel
  kevinkongyixueyuan:
    # Required. list of orderers designated by the application to use for transactions on this
    # channel. This list can be a result of access control ("org1" can only access "ordererA"), or
    # operational decisions to share loads from applications among the orderers.  The values must
    # be "names" of orgs defined under "organizations/peers"
    orderers:
      - orderer.kevin.kongyixueyuan.com

    # Required. list of peers from participating orgs
    peers:
      peer0.org1.kevin.kongyixueyuan.com:
        # [Optional]. will this peer be sent transaction proposals for endorsement? The peer must
        # have the chaincode installed. The app can also use this property to decide which peers
        # to send the chaincode install request. Default: true
        endorsingPeer: true

        # [Optional]. will this peer be sent query proposals? The peer must have the chaincode
        # installed. The app can also use this property to decide which peers to send the
        # chaincode install request. Default: true
        chaincodeQuery: true

        # [Optional]. will this peer be sent query proposals that do not require chaincodes, like
        # queryBlock(), queryTransaction(), etc. Default: true
        ledgerQuery: true

        # [Optional]. will this peer be the target of the SDK's listener registration? All peers can
        # produce events but the app typically only needs to connect to one to listen to events.
        # Default: true
        eventSource: true

      peer1.org1.kevin.kongyixueyuan.com:
        endorsingPeer: false
        chaincodeQuery: true
        ledgerQuery: true
        eventSource: false

      peer0.org2.kevin.kongyixueyuan.com:
        endorsingPeer: true
        chaincodeQuery: true
        ledgerQuery: true
        eventSource: true

      peer1.org2.kevin.kongyixueyuan.com:
        endorsingPeer: false
        chaincodeQuery: true
        ledgerQuery: true
        eventSource: false

    # [Optional]. what chaincodes are expected to exist on this channel? The application can use
    # this information to validate that the target peers are in the expected state by comparing
    # this list with the query results of getInstalledChaincodes() and getInstantiatedChaincodes()
    chaincodes:
      # the format follows the "cannonical name" of chaincodes by fabric code
      - mycc:v0

#
# list of participating organizations in this network
#
organizations:
  Org1:
    mspid: org1.kevin.kongyixueyuan.com

    peers:
      - peer0.org1.kevin.kongyixueyuan.com
      - peer1.org1.kevin.kongyixueyuan.com

    # [Optional]. Certificate Authorities issue certificates for identification purposes in a Fabric based
    # network. Typically certificates provisioning is done in a separate process outside of the
    # runtime network. Fabric-CA is a special certificate authority that provides a REST APIs for
    # dynamic certificate management (enroll, revoke, re-enroll). The following section is only for
    # Fabric-CA servers.
    certificateAuthorities:
      - ca-org1

    # [Optional]. If the application is going to make requests that are reserved to organization
    # administrators, including creating/updating channels, installing/instantiating chaincodes, it
    # must have access to the admin identity represented by the private key and signing certificate.
    # Both properties can be the PEM string or local path to the PEM file. Note that this is mainly for
    # convenience in development mode, production systems should not expose sensitive information
    # this way. The SDK should allow applications to set the org admin identity via APIs, and only use
    # this route as an alternative when it exists.
    adminPrivateKey:
      path: artifacts/channel/crypto-config/peerOrganizations/org1.kevin.kongyixueyuan.com/users/Admin@org1.kevin.kongyixueyuan.com/msp/keystore/ca72ee77552f81537f82c98f4d2abc6d6df1c94c8022f1a176c45e35a812496d_sk
    signedCert:
      path: artifacts/channel/crypto-config/peerOrganizations/org1.kevin.kongyixueyuan.com/users/Admin@org1.kevin.kongyixueyuan.com/msp/signcerts/Admin@org1.kevin.kongyixueyuan.com-cert.pem

  # the profile will contain public information about organizations other than the one it belongs to.
  # These are necessary information to make transaction lifecycles work, including MSP IDs and
  # peers with a public URL to send transaction proposals. The file will not contain private
  # information reserved for members of the organization, such as admin key and certificate,
  # fabric-ca registrar enroll ID and secret, etc.
  Org2:
    mspid: org2.kevin.kongyixueyuan.com
    peers:
      - peer0.org2.kevin.kongyixueyuan.com
      - peer1.org2.kevin.kongyixueyuan.com
    certificateAuthorities:
      - ca-org2
    adminPrivateKey:
      path: artifacts/channel/crypto-config/peerOrganizations/org2.kevin.kongyixueyuan.com/users/Admin@org2.kevin.kongyixueyuan.com/msp/keystore/77c2e4e04f4098e29b6143f18a378a20fce8af5151b7e93aafba327fc2256c30_sk
    signedCert:
      path: artifacts/channel/crypto-config/peerOrganizations/org2.kevin.kongyixueyuan.com/users/Admin@org2.kevin.kongyixueyuan.com/msp/signcerts/Admin@org2.kevin.kongyixueyuan.com-cert.pem

#
# List of orderers to send transaction and channel create/update requests to. For the time
# being only one orderer is needed. If more than one is defined, which one get used by the
# SDK is implementation specific. Consult each SDK's documentation for its handling of orderers.
#
orderers:
  orderer.kevin.kongyixueyuan.com:
    url: grpcs://ec2-13-59-99-140.us-east-2.compute.amazonaws.com:7050

    # these are standard properties defined by the gRPC library
    # they will be passed in as-is to gRPC client constructor
    grpcOptions:
      ssl-target-name-override: orderer.kevin.kongyixueyuan.com

    tlsCACerts:
      path: artifacts/channel/crypto-config/ordererOrganizations/kevin.kongyixueyuan.com/orderers/orderer.kevin.kongyixueyuan.com/tls/ca.crt

#
# List of peers to send various requests to, including endorsement, query
# and event listener registration.
#
peers:
  peer0.org1.kevin.kongyixueyuan.com:
    # this URL is used to send endorsement and query requests
    url: grpcs://ec2-13-59-99-140.us-east-2.compute.amazonaws.com:7051

    grpcOptions:
      ssl-target-name-override: peer0.org1.kevin.kongyixueyuan.com
    tlsCACerts:
      path: artifacts/channel/crypto-config/peerOrganizations/org1.kevin.kongyixueyuan.com/peers/peer0.org1.kevin.kongyixueyuan.com/tls/ca.crt

  peer1.org1.kevin.kongyixueyuan.com:
    url: grpcs://ec2-13-59-99-140.us-east-2.compute.amazonaws.com:7056
    eventUrl: grpcs://ec2-13-59-99-140.us-east-2.compute.amazonaws.com:7058
    grpcOptions:
      ssl-target-name-override: peer1.org1.kevin.kongyixueyuan.com
    tlsCACerts:
      path: artifacts/channel/crypto-config/peerOrganizations/org1.kevin.kongyixueyuan.com/peers/peer1.org1.kevin.kongyixueyuan.com/tls/ca.crt

  peer0.org2.kevin.kongyixueyuan.com:
    url: grpcs://ec2-13-59-99-140.us-east-2.compute.amazonaws.com:8051
    grpcOptions:
      ssl-target-name-override: peer0.org2.kevin.kongyixueyuan.com
    tlsCACerts:
      path: artifacts/channel/crypto-config/peerOrganizations/org2.kevin.kongyixueyuan.com/peers/peer0.org2.kevin.kongyixueyuan.com/tls/ca.crt

  peer1.org2.kevin.kongyixueyuan.com:
    url: grpcs://ec2-13-59-99-140.us-east-2.compute.amazonaws.com:8056
    grpcOptions:
      ssl-target-name-override: peer1.org2.kevin.kongyixueyuan.com
    tlsCACerts:
      path: artifacts/channel/crypto-config/peerOrganizations/org2.kevin.kongyixueyuan.com/peers/peer1.org2.kevin.kongyixueyuan.com/tls/ca.crt

#
# Fabric-CA is a special kind of Certificate Authority provided by Hyperledger Fabric which allows
# certificate management to be done via REST APIs. Application may choose to use a standard
# Certificate Authority instead of Fabric-CA, in which case this section would not be specified.
#
certificateAuthorities:
  ca-org1:
    url: https://ec2-13-59-99-140.us-east-2.compute.amazonaws.com:7054
    # the properties specified under this object are passed to the 'http' client verbatim when
    # making the request to the Fabric-CA server
    httpOptions:
      verify: false
    tlsCACerts:
      path: artifacts/channel/crypto-config/peerOrganizations/org1.kevin.kongyixueyuan.com/ca/ca.org1.kevin.kongyixueyuan.com-cert.pem

    # Fabric-CA supports dynamic user enrollment via REST APIs. A "root" user, a.k.a registrar, is
    # needed to enroll and invoke new users.
    registrar:
      - enrollId: admin
        enrollSecret: adminpw
    # [Optional] The optional name of the CA.
    caName: ca-org1

  ca-org2:
    url: https://ec2-13-59-99-140.us-east-2.compute.amazonaws.com:8054
    httpOptions:
      verify: false
    tlsCACerts:
      path: artifacts/channel/crypto-config/peerOrganizations/org2.kevin.kongyixueyuan.com/ca/ca.org2.kevin.kongyixueyuan.com-cert.pem
    registrar:
      - enrollId: admin
        enrollSecret: adminpw
    # [Optional] The optional name of the CA.
    caName: ca-org2
```

#### org1.yaml

新建 org1.yaml 配置文件并编辑

```js
$ vim org1.yaml
```

`org1.yaml` 文件完整内容如下：

```js
---
#
# The network connection profile provides client applications the information about the target
# blockchain network that are necessary for the applications to interact with it. These are all
# knowledge that must be acquired from out-of-band sources. This file provides such a source.
#
name: "kevin-fabric-sdk-node-org1"

#
# Any properties with an "x-" prefix will be treated as application-specific, exactly like how naming
# in HTTP headers or swagger properties work. The SDK will simply ignore these fields and leave
# them for the applications to process. This is a mechanism for different components of an application
# to exchange information that are not part of the standard schema described below. In particular,
# the "x-type" property with the "hlfv1" value example below is used by Hyperledger Composer to
# determine the type of Fabric networks (v0.6 vs. v1.0) it needs to work with.
#
x-type: "hlfv1"

#
# Describe what the target network is/does.
#
description: "Balance Transfer Network - client definition for Org1"

#
# Schema version of the content. Used by the SDK to apply the corresponding parsing rules.
#
version: "1.0"

#
# The client section is SDK-specific. The sample below is for the node.js SDK
#
client:
  # Which organization does this application instance belong to? The value must be the name of an org
  # defined under "organizations"
  organization: Org1

  # Some SDKs support pluggable KV stores, the properties under "credentialStore"
  # are implementation specific
  credentialStore:
    # [Optional]. Specific to FileKeyValueStore.js or similar implementations in other SDKs. Can be others
    # if using an alternative impl. For instance, CouchDBKeyValueStore.js would require an object
    # here for properties like url, db name, etc.
    path: "./fabric-client-kv-org1"

    # [Optional]. Specific to the CryptoSuite implementation. Software-based implementations like
    # CryptoSuite_ECDSA_AES.js in node SDK requires a key store. PKCS#11 based implementations does
    # not.
    cryptoStore:
      # Specific to the underlying KeyValueStore that backs the crypto key store.
      path: "/tmp/fabric-client-kv-org1"

    # [Optional]. Specific to Composer environment
    wallet: wallet-name
```

#### org2.yaml

新建 org2.yaml 配置文件并编辑

```js
$ vim org2.yaml
```

`org2.yaml` 文件完整内容如下：

```js
---
#
# The network connection profile provides client applications the information about the target
# blockchain network that are necessary for the applications to interact with it. These are all
# knowledge that must be acquired from out-of-band sources. This file provides such a source.
#
name: "kevin-fabric-sdk-node-org2"

#
# Any properties with an "x-" prefix will be treated as application-specific, exactly like how naming
# in HTTP headers or swagger properties work. The SDK will simply ignore these fields and leave
# them for the applications to process. This is a mechanism for different components of an application
# to exchange information that are not part of the standard schema described below. In particular,
# the "x-type" property with the "hlfv1" value example below is used by Hyperledger Composer to
# determine the type of Fabric networks (v0.6 vs. v1.0) it needs to work with.
#
x-type: "hlfv1"

#
# Describe what the target network is/does.
#
description: "Balance Transfer Network - client definition for Org2"

#
# Schema version of the content. Used by the SDK to apply the corresponding parsing rules.
#
version: "1.0"

#
# The client section is SDK-specific. The sample below is for the node.js SDK
#
client:
  # Which organization does this application instance belong to? The value must be the name of an org
  # defined under "organizations"
  organization: Org2

  # Some SDKs support pluggable KV stores, the properties under "credentialStore"
  # are implementation specific
  credentialStore:
    # [Optional]. Specific to FileKeyValueStore.js or similar implementations in other SDKs. Can be others
    # if using an alternative impl. For instance, CouchDBKeyValueStore.js would require an object
    # here for properties like url, db name, etc.
    path: "./fabric-client-kv-org2"

    # [Optional]. Specific to the CryptoSuite implementation. Software-based implementations like
    # CryptoSuite_ECDSA_AES.js in node SDK requires a key store. PKCS#11 based implementations does
    # not.
    cryptoStore:
      # Specific to the underlying KeyValueStore that backs the crypto key store.
      path: "/tmp/fabric-client-kv-org2"

    # [Optional]. Specific to Composer environment
    wallet: wallet-name
```