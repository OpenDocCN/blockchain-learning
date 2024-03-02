# 第六章 Hyperledger Fabric（V1.2）源码深度解析－cryptogen

无论是开发 Hyperledger Fabric 应用还是测试，都需要有完整的网络环境，而 Hyperledger Fabric 网络环境包含了较多的内容，包括网络节点，身份证书，私钥等等。这些内容（Network Artifacts）全部都可以使用 Hyperledger Fabric 中的 cryptogen 工具来负责生成，现在我们从源码中来分析一下相关的内容是如何生成的。

### 组织结构的定义

cryptogen 工具的源码文件所在路径：hyperledger/fabric/common/tools/cryptogen/main.go，下面我们来分析其中的具体内容。

首先在源码中定义了一些常量，指定了用户名，管理员名称，默认的主机/节点名规则及默认的节点完整域名命名规则。如下所示：

```go
const (
    userBaseName            = "User"
    adminBaseName           = "Admin"
    defaultHostnameTemplate = "{{.Prefix}}{{.Index}}"
    defaultCNTemplate       = "{{.Hostname}}.{{.Domain}}"
) 
```

定义相关的结构体，指定组成内容，如下：

```go
type HostnameData struct {
    Prefix string
    Index  int
    Domain string
}

type SpecData struct {
    Hostname   string
    Domain     string
    CommonName string
}

type NodeTemplate struct {
    Count    int      `yaml:"Count"`
    Start    int      `yaml:"Start"`
    Hostname string   `yaml:"Hostname"`
    SANS     []string `yaml:"SANS"`
}

type NodeSpec struct {
    Hostname           string   `yaml:"Hostname"`
    CommonName         string   `yaml:"CommonName"`
    Country            string   `yaml:"Country"`
    Province           string   `yaml:"Province"`
    Locality           string   `yaml:"Locality"`
    OrganizationalUnit string   `yaml:"OrganizationalUnit"`
    StreetAddress      string   `yaml:"StreetAddress"`
    PostalCode         string   `yaml:"PostalCode"`
    SANS               []string `yaml:"SANS"`
}

type UsersSpec struct {
    Count int `yaml:"Count"`
}

type OrgSpec struct {
    Name          string       `yaml:"Name"`
    Domain        string       `yaml:"Domain"`
    EnableNodeOUs bool         `yaml:"EnableNodeOUs"`
    CA            NodeSpec     `yaml:"CA"`
    Template      NodeTemplate `yaml:"Template"`
    Specs         []NodeSpec   `yaml:"Specs"`
    Users         UsersSpec    `yaml:"Users"`
}

type Config struct {
    OrdererOrgs []OrgSpec `yaml:"OrdererOrgs"`
    PeerOrgs    []OrgSpec `yaml:"PeerOrgs"`
} 
```

### 默认配置模板

如果用户在生成 Network Artifacts 时没有指定对应所使用的配置文件模板，那么 Hyperledger Fabric 会使用一个默认的配置文件作为生成 Network Artifacts 的配置信息模板。定义一个 defaultConfig 变量，其值代表默认配置模板信息：

```go
// 解析者：Hanxiaodong
// QQ 群（专业 Fabric 交流群）：862733552
var defaultConfig = `
# ---------------------------------------------------------------------------
# "OrdererOrgs" - Definition of organizations managing orderer nodes
# ---------------------------------------------------------------------------
OrdererOrgs:
  # ---------------------------------------------------------------------------
  # Orderer
  # ---------------------------------------------------------------------------
  - Name: Orderer
    Domain: example.com

    # ---------------------------------------------------------------------------
    # "Specs" - See PeerOrgs below for complete description
    # ---------------------------------------------------------------------------
    Specs:
      - Hostname: orderer

# ---------------------------------------------------------------------------
# "PeerOrgs" - Definition of organizations managing peer nodes
# ---------------------------------------------------------------------------
PeerOrgs:
  # ---------------------------------------------------------------------------
  # Org1
  # ---------------------------------------------------------------------------
  - Name: Org1
    Domain: org1.example.com
    EnableNodeOUs: false

    # ---------------------------------------------------------------------------
    # "CA"
    # ---------------------------------------------------------------------------
    # Uncomment this section to enable the explicit definition of the CA for this
    # organization.  This entry is a Spec.  See "Specs" section below for details.
    # ---------------------------------------------------------------------------
    # CA:
    #    Hostname: ca # implicitly ca.org1.example.com
    #    Country: US
    #    Province: California
    #    Locality: San Francisco
    #    OrganizationalUnit: Hyperledger Fabric
    #    StreetAddress: address for org # default nil
    #    PostalCode: postalCode for org # default nil

    # ---------------------------------------------------------------------------
    # "Specs"
    # ---------------------------------------------------------------------------
    # Uncomment this section to enable the explicit definition of hosts in your
    # configuration.  Most users will want to use Template, below
    #
    # Specs is an array of Spec entries.  Each Spec entry consists of two fields:
    #   - Hostname:   (Required) The desired hostname, sans the domain.
    #   - CommonName: (Optional) Specifies the template or explicit override for
    #                 the CN.  By default, this is the template:
    #
    #                              "{{.Hostname}}.{{.Domain}}"
    #
    #                 which obtains its values from the Spec.Hostname and
    #                 Org.Domain, respectively.
    #   - SANS:       (Optional) Specifies one or more Subject Alternative Names
    #                 to be set in the resulting x509\. Accepts template
    #                 variables {{.Hostname}}, {{.Domain}}, {{.CommonName}}. IP
    #                 addresses provided here will be properly recognized. Other
    #                 values will be taken as DNS names.
    #                 NOTE: Two implicit entries are created for you:
    #                     - {{ .CommonName }}
    #                     - {{ .Hostname }}
    # ---------------------------------------------------------------------------
    # Specs:
    #   - Hostname: foo # implicitly "foo.org1.example.com"
    #     CommonName: foo27.org5.example.com # overrides Hostname-based FQDN set above
    #     SANS:
    #       - "bar.{{.Domain}}"
    #       - "altfoo.{{.Domain}}"
    #       - "{{.Hostname}}.org6.net"
    #       - 172.16.10.31
    #   - Hostname: bar
    #   - Hostname: baz

    # ---------------------------------------------------------------------------
    # "Template"
    # ---------------------------------------------------------------------------
    # Allows for the definition of 1 or more hosts that are created sequentially
    # from a template. By default, this looks like "peer%d" from 0 to Count-1.
    # You may override the number of nodes (Count), the starting index (Start)
    # or the template used to construct the name (Hostname).
    #
    # Note: Template and Specs are not mutually exclusive.  You may define both
    # sections and the aggregate nodes will be created for you.  Take care with
    # name collisions
    # ---------------------------------------------------------------------------
    Template:
      Count: 1
      # Start: 5
      # Hostname: {{.Prefix}}{{.Index}} # default
      # SANS:
      #   - "{{.Hostname}}.alt.{{.Domain}}"

    # ---------------------------------------------------------------------------
    # "Users"
    # ---------------------------------------------------------------------------
    # Count: The number of user accounts _in addition_ to Admin
    # ---------------------------------------------------------------------------
    Users:
      Count: 1

  # ---------------------------------------------------------------------------
  # Org2: See "Org1" for full specification
  # ---------------------------------------------------------------------------
  - Name: Org2
    Domain: org2.example.com
    EnableNodeOUs: false
    Template:
      Count: 1
    Users:
      Count: 1
` 
```

> 该默认模板文件的内容与 fabric-samples/first-network/crypto-config.yaml 的配置文件内容大致相同，不同之处在 EnableNodeOUs 与 Template 的值。

定义变量 app，指定相关命令及该命令可以附带的参数信息：

```go
//命令行参数
var (
    app = kingpin.New("cryptogen", "Utility for generating Hyperledger Fabric key material")

    gen           = app.Command("generate", "Generate key material")
    outputDir     = gen.Flag("output", "The output directory in which to place artifacts").Default("crypto-config").String()
    genConfigFile = gen.Flag("config", "The configuration template to use").File()

    showtemplate = app.Command("showtemplate", "Show the default configuration template")

    version       = app.Command("version", "Show version information")
    ext           = app.Command("extend", "Extend existing network")
    inputDir      = ext.Flag("input", "The input directory in which existing network place").Default("crypto-config").String()
    extConfigFile = ext.Flag("config", "The configuration template to use").File()
) 
```

kingpin.New()是创建了一个 Kingpin application 实例，然后调用其 Command 函数创建相应的子命令 generate、showtemplate、version、extend，并为 generate 与 extend 两个子命令使用 Flag 函数分别指定可附带的参数，其中 generate 子命令可附带的参数为：ouput、config，extend 子命令可附带的参数为：input、config。

### 生成 crypto

定义主函数，根据命令行中指定的命令执行相关的功能：

```go
func main() {
    kingpin.Version("0.0.1")
    switch kingpin.MustParse(app.Parse(os.Args[1:])) {

    // 如果是 "generate" 命令
    case gen.FullCommand():
        generate()

    case ext.FullCommand():
        extend()

        // 如果是 "showtemplate" 命令
    case showtemplate.FullCommand():
        fmt.Print(defaultConfig)
        os.Exit(0)

        // 如果是 "version" 命令
    case version.FullCommand():
        printVersion()
    }

} 
```

如果命令行中给定的是“generate”命令，则调用 generate()函数，如果给定的是“extend”命令，则调用 extend()函数，如果是“showtemplate”命令，则直接输出已定义的默认配置模板信息，如果是“version”命令，则调用 printVersion()函数直接输出版本信息。

我们在此对 generate 命令进行解释，所调用的 generate()函数源码如下：

```go
func generate() {

    config, err := getConfig()
    if err != nil {
        fmt.Printf("Error reading config: %s", err)
        os.Exit(-1)
    }

    for _, orgSpec := range config.PeerOrgs {
        err = renderOrgSpec(&orgSpec, "peer")
        if err != nil {
            fmt.Printf("Error processing peer configuration: %s", err)
            os.Exit(-1)
        }
        generatePeerOrg(*outputDir, orgSpec)
    }

    for _, orgSpec := range config.OrdererOrgs {
        err = renderOrgSpec(&orgSpec, "orderer")
        if err != nil {
            fmt.Printf("Error processing orderer configuration: %s", err)
            os.Exit(-1)
        }
        generateOrdererOrg(*outputDir, orgSpec)
    }
} 
```

首先通过调用 getConfig()函数获取配置信息，如果在命令行中指定了 config 参数，则根据指定的文件所在路径及文件名称读取配置内容并返回，反之使用定义的默认配置模板信息。

然后利用 for 循环从配置信息中获取关于 PeerOrgs 的内容，调用 renderOrgSpec(&orgSpec, "peer")生成指定的域，

```go
// 解析者：Hanxiaodong
// QQ 群（专业 Fabric 交流群）：862733552
func renderOrgSpec(orgSpec *OrgSpec, prefix string) error {
    // 首先处理模板所有节点
    for i := 0; i < orgSpec.Template.Count; i++ {
        data := HostnameData{
            Prefix: prefix,
            Index:  i + orgSpec.Template.Start,
            Domain: orgSpec.Domain,
        }

        hostname, err := parseTemplateWithDefault(orgSpec.Template.Hostname, defaultHostnameTemplate, data)
        if err != nil {
            return err
        }

        spec := NodeSpec{
            Hostname: hostname,
            SANS:     orgSpec.Template.SANS,
        }
        orgSpec.Specs = append(orgSpec.Specs, spec)
    }

    // 修改所有通用节点规范以添加域
    for idx, spec := range orgSpec.Specs {
        err := renderNodeSpec(orgSpec.Domain, &spec)
        if err != nil {
            return err
        }

        orgSpec.Specs[idx] = spec
    }

    // 以相同的方式处理 CA 节点规范
    if len(orgSpec.CA.Hostname) == 0 {
        orgSpec.CA.Hostname = "ca"
    }
    err := renderNodeSpec(orgSpec.Domain, &orgSpec.CA)
    if err != nil {
        return err
    }

    return nil
} 
```

之后再调用 generatePeerOrg(*outputDir, orgSpec)函数，将生成的 Orgs 结构信息保存在指定的输出目录中（如果没有指定输出目录，则默认为当前的"crypto-config"目录）。生成 Orgs 结构信息过程较为复杂，涉及到 CA 相关的内容。大体步骤如下：

1.  指定组织中相应的各目录路径，如下：

    ```go
     orgDir := filepath.Join(baseDir, "peerOrganizations", orgName)
        caDir := filepath.Join(orgDir, "ca")
        tlsCADir := filepath.Join(orgDir, "tlsca")
        mspDir := filepath.Join(orgDir, "msp")
        peersDir := filepath.Join(orgDir, "peers")
        usersDir := filepath.Join(orgDir, "users")
        adminCertsDir := filepath.Join(mspDir, "admincerts") 
    ```

2.  调用 signCA, err := ca.NewCA(caDir, orgName, orgSpec.CA.CommonName, orgSpec.CA.Country, orgSpec.CA.Province, orgSpec.CA.Locality, orgSpec.CA.OrganizationalUnit, orgSpec.CA.StreetAddress, orgSpec.CA.PostalCode)根据的 ca 所属路径，创建 CA 的实例并保存签名密钥对；

3.  调用 tlsCA, err := ca.NewCA(tlsCADir, orgName, "tls"+orgSpec.CA.CommonName, orgSpec.CA.Country, orgSpec.CA.Province, orgSpec.CA.Locality, orgSpec.CA.OrganizationalUnit, orgSpec.CA.StreetAddress, orgSpec.CA.PostalCode)生成 TLS CA 的实例并保存签名密钥对;

4.  调用 err = msp.GenerateVerifyingMSP(mspDir, signCA, tlsCA, orgSpec.EnableNodeOUs)生成 MSP 相关的内容;

5.  调用 generateNodes(peersDir, orgSpec.Specs, signCA, tlsCA, msp.PEER, orgSpec.EnableNodeOUs)生成 Peer 节点相关的内容;

6.  调用 generateNodes(usersDir, users, signCA, tlsCA, msp.CLIENT, orgSpec.EnableNodeOUs)生成用户相关的内容；

7.  通过调用 err = copyAdminCert(usersDir, adminCertsDir, adminUser.CommonName)函数，将管理证书复制到 org 的 MSP admincerts 目录下；

8.  最后通过一个 for 循环，将管理证书复制到组织的每个 Peer 节点的 MSP admincert 目录下。

至此为止，关于 Org 组织相关的所有内容已经全部创建完成，接下来利用 for 循环从配置信息中获取关于 OrdererOrgs 的内容，调用 renderOrgSpec(&orgSpec, "orderer")生成生成指定的域，之后再调用 generateOrdererOrg(*outputDir, orgSpec)函数，将生成的 Orderer 结构信息保存在指定的输出目录中（如果没有指定输出目录，则默认为当前的"crypto-config"目录），生成 Orderer 组织的结构过程与生成 Orgs 组织的结构过程类似，不再赘述。