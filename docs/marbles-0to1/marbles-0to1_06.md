# 2.2 设置环境-连接概要库

### 创建目录 utils/connection_profile_lib/parts

在项目根目录下创建一个连接概要库目录并进入该目录：

```js
$ cd $HOME/kevin-marbles
$ mkdir -p utils/connection_profile_lib/parts
$ cd utils/connection_profile_lib/parts 
```

在该目录下创建相关的 helper.js、ca.js、chaincode.js、detect_env.js、orderer.js、org.js、other.js、peer.js、verification.js 文件并编辑

#### 创建 helper.js

创建 helper.js 文件并编辑，helper.js 文件主要用于获取应用的 port、event 等配置信息

```js
$ vim helper.js 
```

helper.js 文件完整内容如下

```js
// ============================================================================================================================
// Get config file fields
// ============================================================================================================================

module.exports = function (cp, logger) {
    var helper = {};

    // get the marble owner names
    helper.getMarbleUsernamesConfig = function () {
        return cp.getMarblesField('usernames');
    };

    // get the marbles trading company name from config file
    helper.getCompanyNameFromFile = function () {
        return cp.getMarblesField('company');
    };

    // get the marble's server port number
    helper.getMarblesPort = function () {
        return cp.getMarblesField('port');
    };

    // get the status of marbles previous startup
    helper.getEventsSetting = function () {
        if (cp.config['use_events']) {
            return cp.config['use_events'];
        }
        return false;
    };

    // get the re-enrollment period in seconds
    helper.getKeepAliveMs = function () {
        var sec = cp.getMarblesField('keep_alive_secs');
        if (!sec) sec = 30;        // default to 30 seconds
        return (sec * 1000);
    };

    return helper;
}; 
```

#### 创建 detect_env.js

定义一个网络信息检测文件，用于检测网络环境信息。

创建 detect_env.js 文件并编辑

```js
$ vim detect_env.js 
```

detect_env.js 文件完整内容如下

```js
// ============================================================================================================================
//     Detect if CP is in the env or in a file.  env overrides a file.
// ============================================================================================================================
module.exports = function (logger) {
    var detect_env = {};

    // See if marbles has the connection profile as an environmental variable or not - return cp if so, null if not
    detect_env.getConnectionProfileFromEnv = function () {

        // --- Get Service Credentials  --- //
        if (process.env.VCAP_SERVICES) {                                                // if we are in IBM Cloud this will be set
            logger.info('Detected that we are in IBM Cloud');
            let VCAP = null;
            try {
                VCAP = JSON.parse(process.env.VCAP_SERVICES);
                console.log('vcap:', JSON.stringify(VCAP, null, 2));
            } catch (e) {
                logger.warn('VCAP from IBM Cloud is not JSON... this is not great');
            }
            for (let plan_name in VCAP) {
                if (plan_name.indexOf('blockchain') >= 0) {
                    logger.info('pretty sure this is the IBM Blockchain Platform service:', plan_name);
                    if (VCAP[plan_name][0].credentials && VCAP[plan_name][0].credentials) {                // pull the first org
                        const firstOrg = Object.keys(VCAP[plan_name][0].credentials)[0];
                        if (firstOrg) {
                            process.env.SERVICE_CREDENTIALS = VCAP[plan_name][0].credentials[firstOrg];    // found our service credentials
                        }
                    }
                }
            }
        }

        // --- Get Connection Profile  --- //
        if (process.env.CONNECTION_PROFILE) {
            try {
                const cp = JSON.parse(process.env.CONNECTION_PROFILE);
                //console.log('cp:', typeof cp, Object.keys(cp));
                return cp;
            } catch (e) {
                logger.error('CONNECTION_PROFILE from IBM Cloud is not JSON... this is bad');
            }
            logger.error('uh oh, we are using IBM Cloud but I could not find CONNECTION_PROFILE');
        }

        return null;
    };

    return detect_env;
}; 
```

#### 创建 ca.js

创建 ca.js 文件并编辑，主要用于获取 CA 相关的信息

```js
$ vim ca.js 
```

ca.js 文件完整内容如下

```js
// ============================================================================================================================
//     Get certificate authorities fields from connection profile data
// ============================================================================================================================
module.exports = function (cp, logger) {
    var helper = {};

    // find the first ca in the certificateAuthorities field for this org
    helper.getFirstCaName = function (orgName) {
        const org = cp.creds.organizations[orgName];
        if (org && org.certificateAuthorities) {
            if (org.certificateAuthorities && org.certificateAuthorities[0]) {
                return org.certificateAuthorities[0];
            }
        }
        logger.error('CA not found');
        return null;
    };

    // get a ca obj
    helper.getCA = function (key) {
        if (key === undefined || key == null) {
            logger.error('CA key not passed');
            return null;
        } else {
            if (cp.creds.certificateAuthorities) {
                return cp.creds.certificateAuthorities[key];
            } else {
                return null;
            }
        }
    };

    // get a ca's http url
    helper.getCasUrl = function (key) {
        if (key === undefined || key == null) {
            logger.error('CA key not passed');
            return null;
        } else {
            let ca = helper.getCA(key);
            if (ca) {
                return ca.url;
            } else {
                logger.error('CA not found');
                return null;
            }
        }
    };

    // get all the ca http urls
    helper.getAllCaUrls = function () {
        let ret = [];
        for (let id in cp.creds.certificateAuthorities) {
            ret.push(cp.creds.certificateAuthorities[id].url);
        }
        return ret;
    };

    // get a ca's name, could be null
    helper.getCaName = function (key) {
        if (key === undefined || key == null) {
            logger.error('CA key not passed');
            return null;
        } else {
            let ca = helper.getCA(key);
            if (ca) {
                return ca.caName;
            } else {
                logger.error('CA not found');
                return null;
            }
        }
    };

    // get a ca's tls options
    helper.getCaTlsCertOpts = function (key) {
        if (key === undefined || key == null) {
            logger.error('CA key not passed');
            return null;
        } else {
            let ca = helper.getCA(key);
            return cp.buildTlsOpts(ca);
        }
    };

    // get an enrollment user
    helper.getEnrollObj = function (caKey, user_index) {
        if (caKey === undefined || caKey == null) {
            logger.error('CA key not passed');
            return null;
        } else {
            var ca = helper.getCA(caKey);
            if (ca && ca.registrar && ca.registrar[user_index]) {
                return ca.registrar[user_index];
            } else {
                logger.error('Cannot find enroll id at index.', caKey, user_index);
                return null;
            }
        }
    };

    return helper;
}; 
```

#### 创建 chaincode.js

创建 chaincode.js 文件并编辑，主要用于获取链码 ID 及版本信息

```js
$ vim chaincode.js 
```

chaincode.js 文件完整内容如下

```js
// ============================================================================================================================
//     Get chaincode fields from connection profile data
// ============================================================================================================================
module.exports = function (cp, logger) {
    var helper = {};

    // ----------------------------------------------------------
    // get the first chaincode id on the network
    // ----------------------------------------------------------
    helper.getChaincodeId = function () {
        if (process.env.CHAINCODE_ID) {                                                    // detected a preferred chaincode id instead of first
            //console.log('debug: found preferred chaincode id', process.env.CHAINCODE_ID);
            return process.env.CHAINCODE_ID;
        } else {                                                                        // else get the first chaincode we see
            var channel = cp.getChannelId();
            if (channel && cp.creds.channels[channel] && cp.creds.channels[channel].chaincodes) {
                if (Array.isArray(cp.creds.channels[channel].chaincodes)) {                // config version 1.0.2 way
                    let chaincode = cp.creds.channels[channel].chaincodes[0];            // first one
                    if (chaincode) {
                        return chaincode.split(':')[0];
                    }
                } else {
                    let chaincode = Object.keys(cp.creds.channels[channel].chaincodes);    // config version 1.0.0 and 1.0.1 way
                    return chaincode[0];                                                // first one
                }
            }
            logger.warn('No chaincode ID found in connection profile... might be okay if we haven\'t instantiated marbles yet');
            return null;
        }
    };

    // ----------------------------------------------------------
    // get the first chaincode version on the network
    // ----------------------------------------------------------
    helper.getChaincodeVersion = function () {
        if (process.env.CHAINCODE_VERSION) {                                            // detected a preferred chaincode version instead of first
            //console.log('debug: found preferred chaincode version', process.env.CHAINCODE_VERSION);
            return process.env.CHAINCODE_VERSION;
        } else {                                                                        // else get the first chaincode we see
            var channel = cp.getChannelId();
            var chaincodeId = helper.getChaincodeId();
            if (channel && chaincodeId) {
                if (Array.isArray(cp.creds.channels[channel].chaincodes)) {                // config version 1.0.2 way
                    let chaincode = cp.creds.channels[channel].chaincodes[0];            // first one
                    if (chaincode) {
                        return chaincode.split(':')[1];
                    }
                } else {
                    return cp.creds.channels[channel].chaincodes[chaincodeId];    // config version 1.0.0 and 1.0.1 way
                }
            }
            logger.warn('No chaincode version found in connection profile... might be okay if we haven\'t instantiated marbles yet');
            return null;
        }
    };

    return helper;
}; 
```

#### 创建 orderer.js

创建 orderer.js 文件并编辑， 主要用于获取 Orderer 节点的相关信息

```js
$ vim orderer.js 
```

orderer.js 文件完整内容如下

```js
// ============================================================================================================================
//     Get orderer fields from connection profile data
// ============================================================================================================================

module.exports = function (cp, logger) {
    var helper = {};

    // get the first orderer in the channels field
    helper.getFirstOrdererName = function (ch) {
        const channel = cp.creds.channels[ch];
        if (channel && channel.orderers && channel.orderers[0]) {
            return channel.orderers[0];
        }
        throw new Error('Orderer not found for this channel', ch);
    };

    // get an orderer object
    helper.getOrderer = function (key) {
        if (key === undefined || key == null) {
            throw new Error('Orderers key not passed');
        } else {
            if (cp.creds.orderers) {
                return cp.creds.orderers[key];
            } else {
                return null;
            }
        }
    };

    // get an orderer's grpc url
    helper.getOrderersUrl = function (key) {
        if (key === undefined || key == null) {
            throw new Error('Orderers key not passed');
        } else {
            let orderer = helper.getOrderer(key);
            if (orderer) {
                return orderer.url;
            }
            else {
                throw new Error('Orderer not found.');
            }
        }
    };

    // get a orderer's tls options
    helper.getOrdererTlsCertOpts = function (key) {
        if (key === undefined || key == null) {
            throw new Error('Orderer\'s key not passed');
        } else {
            let orderer = helper.getOrderer(key);
            return cp.buildTlsOpts(orderer);
        }
    };

    return helper;
}; 
```

#### 创建 org.js

编辑 org.js 文件并编辑， 主要用于获取 org 的相关信息

```js
$ vim org.js 
```

org.js 文件完整内容如下

```js
// ============================================================================================================================
//     Get org fields from connection profile data
// ============================================================================================================================
var fs = require('fs');
var os = require('os');
var path = require('path');

module.exports = function (cp, logger) {
    var helper = {};
    var misc = require('../../misc.js')(logger);                                                // mis.js has generic (non-blockchain) related functions

    // find the first org name in the organization field
    helper.getFirstOrg = function () {
        if (cp.creds.organizations) {
            const orgs = Object.keys(cp.creds.organizations);
            if (orgs && orgs[0]) {
                return orgs[0];
            }
        }
        logger.error('Org not found.');
        return null;
    };

    // get the org name from the cp or config file
    helper.getClientsOrgName = function () {
        if (cp.creds && cp.creds.client && cp.creds.client['x-organizationName']) {
            return misc.saferCompanyNames(cp.creds.client['x-organizationName']);
        } else {
            return misc.saferCompanyNames(cp.getCompanyNameFromFile());                //fallback
        }
    };

    // find the org name in the client field
    helper.getClientOrg = function () {
        if (cp.creds.client && cp.creds.client.organization) {
            return cp.creds.client.organization;
        }
        logger.error('Org not found.');
        return null;
    };

    // get the marble usernames from the cp or config file
    helper.getMarbleUsernames = function () {
        if (cp.using_env) {
            let org = cp.getClientOrg();
            if (org) org = org.toLowerCase();
            else org = '-';

            if (org.indexOf('org1') >= 0) {
                return ['Hanxiaodong', 'Lixu', 'Hanru'];
            } else if (org.indexOf('org2') >= 0) {
                return ['andre', 'andrew', 'aaron'];
            } 
        }

        return cp.getMarbleUsernamesConfig();    // fallback use config file
    };

    // get this org's msp id
    helper.getOrgsMSPid = function (key) {
        if (key === undefined || key == null) {
            throw new Error('Org key not passed');
        }
        else {
            if (cp.creds.organizations && cp.creds.organizations[key]) {
                return cp.creds.organizations[key].mspid;
            }
            else {
                throw new Error('Org key not found.', key);
            }
        }
    };

    // get an admin private key PEM certificate
    helper.getAdminPrivateKeyPEM = function (orgName) {
        if (orgName && cp.creds.organizations && cp.creds.organizations[orgName]) {
            if (!cp.creds.organizations[orgName].adminPrivateKey) {
                if (!cp.creds.organizations[orgName]['x-adminKeyStore'] || !cp.creds.organizations[orgName]['x-adminKeyStore'].path) {
                    throw new Error('Admin private key is not found in the creds json file: ' + orgName);
                } else {
                    const path2key = getCryptoFromCP(cp.creds.organizations[orgName]['x-adminKeyStore'].path);
                    if (path2key) {
                        return cp.loadPem({ path: path2key });
                    }
                }
            } else {
                return cp.loadPem(cp.creds.organizations[orgName].adminPrivateKey);
            }
        }
        throw new Error('Cannot find org.', orgName);
    };

    // get an admin's signed cert PEM
    helper.getAdminSignedCertPEM = function (orgName) {
        if (orgName && cp.creds.organizations && cp.creds.organizations[orgName]) {
            if (!cp.creds.organizations[orgName].signedCert) {
                if (!cp.creds.organizations[orgName]['x-adminCert'] || !cp.creds.organizations[orgName]['x-adminCert'].path) {
                    throw new Error('Admin certificate is not found in the creds json file: ' + orgName);
                } else {
                    return cp.loadPem({ path: cp.creds.organizations[orgName]['x-adminCert'].path });
                }
            } else {
                return cp.loadPem(cp.creds.organizations[orgName].signedCert);
            }
        }
        throw new Error('Cannot find org.', orgName);
    };

    // return path to private key from kvs
    function getCryptoFromCP(kvsPath, cb) {
        let kvs_path = kvsPath;
        if (kvsPath.indexOf('$HOME') >= 0) {
            kvs_path = kvsPath.replace('$HOME', os.homedir()).substr(1);
        }
        if (fs.existsSync(kvs_path)) {    // check if folder exists
            const entries = fs.readdirSync(kvs_path);
            for (let i in entries) {
                const entry_path = path.join(kvs_path, entries[i]);
                if (fs.lstatSync(entry_path).isFile()) {        // found a file, hope its the key/cert we need
                    return entry_path;
                }
            }
        }
        return null;
    }

    return helper;
}; 
```

#### 创建 peer.js

创建 peer.js 文件并编辑， 主要用于获取 peer 相关的信息

```js
$ vim peer.js 
```

peer.js 文件完整内容如下

```js
// ============================================================================================================================
// Get peer fields from connection profile data
// ============================================================================================================================
module.exports = function (cp, logger) {
    const helper = {};

    // find the first peer in the peers field for this org
    helper.getFirstPeerName = function (ch) {
        const channel = cp.creds.channels[ch];
        if (channel && channel.peers) {
            const peers = Object.keys(channel.peers);
            if (peers && peers[0]) {
                return peers[0];
            }
        }
        throw new Error('Peer not found on this channel', ch);
    };

    // get a peer object
    helper.getPeer = function (key) {
        if (key === undefined || key == null) {
            throw new Error('Peer key not passed');
        }
        else {
            if (cp.creds.peers) {
                return cp.creds.peers[key];
            }
            else {
                return null;
            }
        }
    };

    // get a peer's grpc url
    helper.getPeersUrl = function (key) {
        if (key === undefined || key == null) {
            throw new Error('Peer key not passed');
        }
        else {
            let peer = helper.getPeer(key);
            if (peer) {
                return peer.url;
            }
            else {
                throw new Error('Peer key not found.');
            }
        }
    };

    // get all peers grpc urls and event urls, on this channel
    helper.getAllPeerUrls = function (channelId) {
        let ret = {
            urls: [],
            eventUrls: []
        };
        if (cp.creds.channels && cp.creds.channels[channelId]) {
            for (let peerId in cp.creds.channels[channelId].peers) {    //iter on the peers on this channel
                ret.urls.push(cp.creds.peers[peerId].url);    //get the grpc url for this peer
                ret.eventUrls.push(cp.creds.peers[peerId].eventUrl); //get the grpc EVENT url for this peer
            }
        }
        return ret;
    };

    // get a peer's grpc event url
    helper.getPeerEventUrl = function (key) {
        if (key === undefined || key == null) {
            throw new Error('Peer key not passed');
        } else {
            let peer = helper.getPeer(key);
            if (peer) {
                return peer.eventUrl;
            }
            else {
                throw new Error('Peer key not found.');
            }
        }
    };

    // get a peer's tls options
    helper.getPeerTlsCertOpts = function (key) {
        if (key === undefined || key == null) {
            throw new Error('Peer\'s key not passed');
        } else {
            let peer = helper.getPeer(key);
            return cp.buildTlsOpts(peer);
        }
    };

    return helper;
}; 
```

#### 创建 verification.js

创建 verification.js 文件并编辑，主要用于验证 CP 数据

```js
$ vim verification.js 
```

verification.js 文件完整内容如下

```js
// ============================================================================================================================
// Look over the CP data to see if its a-ok
// ============================================================================================================================
var path = require('path');

module.exports = function (cp, logger) {
    var helper = {};

    // --------------------------------------------------------------------------------------------
    // check if marbles UI and marbles chaincode work together
    // --------------------------------------------------------------------------------------------
    helper.errorWithVersions = function (v) {
        var package_json = require(path.join(__dirname, '../../../package.json'));        // get release version of marbles from package.json
        var version = package_json.version;
        if (!v || !v.parsed) v = { parsed: '0.x.x' };    //default
        if (v.parsed[0] !== version[0]) {    //only check the major version
            console.log('\n');
            logger.warn('---------------------------------------------------------------');
            logger.warn('----------------------------- Ah! -----------------------------');
            logger.warn('---------------------------------------------------------------');
            logger.error('Looks like you are using an old version of marbles chaincode...');
            logger.warn('The INTERNAL version of the chaincode found is: v' + v.parsed);
            logger.warn('But this UI is expecting INTERNAL chaincode version: v' + version[0] + '.x.x');
            logger.warn('This mismatch won\'t work =(');
            logger.warn('Install and instantiate the chaincode found in the ./chaincode folder on your channel ' + cp.getChannelId());
            logger.warn('----------------------------------------------------------------------');
            console.log('\n\n');
            return true;
        }
        return false;
    };

    // --------------------------------------------------------------------------------------------
    // check if config has missing entries
    // --------------------------------------------------------------------------------------------
    helper.check_for_missing = function () {
        let errors = [];
        const channel = cp.getChannelId();

        if (!channel) {
            errors.push('There is no channel data in the "channels" field');
        } else {
            let org_2_use, first_ca, first_orderer, first_peer;
            try {
                console.log('Welcome aboard:\t', process.env.marble_company);
                console.log('Channel:\t', channel);
                org_2_use = cp.getClientOrg();
                console.log('Org:\t\t', org_2_use);
                first_ca = cp.getFirstCaName(org_2_use);
                console.log('CA:\t\t', first_ca);
                first_orderer = cp.getFirstOrdererName(channel);
                console.log('Orderer:\t', first_orderer);
                first_peer = cp.getFirstPeerName(channel);
                console.log('Peer:\t\t', first_peer);
            } catch (e) {
                // errors are logged below
            }

            console.log('Chaincode ID:\t', cp.getChaincodeId());
            console.log('Chaincode Version: ', cp.getChaincodeVersion());

            if (!org_2_use) {
                errors.push('There is no org data in the "client" field for provided connection profile');
            }
            if (!cp.getCA(first_ca)) {
                errors.push('There is no CA data in the "certificateAuthorities" field for provided connection profile');
            }
            if (!first_orderer) {
                errors.push('There is no orderer data in the "orderer" field for provided connection profile');
            } else {
                if (!cp.getOrderer(first_orderer)) {
                    errors.push('There is no Orderer data in the "orderers" field for provided connection profile');
                }
            }
            if (!first_peer) {
                errors.push('There is no org peer in the "peer" field for provided connection profile');
            } else {
                if (!cp.getPeer(first_peer)) {
                    errors.push('There is no Peer data in the "peers" field for provided connection profile');
                }
            }
        }

        if (errors.length > 0) {
            console.log('\n');
            logger.warn('----------------------------------------------------------------------');
            logger.warn('------------------------------- Whoops -------------------------------');
            logger.warn('------- You are missing some data in your connection profile ---------');
            logger.warn('----------------------------------------------------------------------');
            for (var i in errors) {
                logger.error(errors[i]);
            }
            logger.warn('----------------------------------------------------------------------');
            if (!cp.using_env) logger.error('Fix this file: ./config/' + cp.getNetworkCredFileName());
            else logger.error('Fix your env variable "CONNECTION_PROFILE"');
            logger.warn('----------------------------------------------------------------------');
            logger.warn('See this file for help:');
            logger.warn('https://github.com/IBM-Blockchain/marbles/blob/v4.0/docs/config_file.md');
            logger.warn('----------------------------------------------------------------------');
            console.log('\n\n');
            return errors;
        }
        return helper.check_protocols();    //run the next check
    };

    // --------------------------------------------------------------------------------------------
    // check if config has protocol errors - returns error array when there is a problem
    // --------------------------------------------------------------------------------------------
    helper.check_protocols = function () {
        let errors = [];
        const channel = cp.getChannelId();
        const org_2_use = cp.getClientOrg();
        const first_ca = cp.getFirstCaName(org_2_use);
        const first_orderer = cp.getFirstOrdererName(channel);
        const first_peer = cp.getFirstPeerName(channel);

        if (cp.getCasUrl(first_ca).indexOf('grpc') >= 0) {
            errors.push('You accidentally typed "grpc" in your CA url. It should be "http://" or "https://"');
        }
        if (cp.getOrderersUrl(first_orderer).indexOf('http') >= 0) {
            errors.push('You accidentally typed "http" in your Orderer url. It should be "grpc://" or "grpcs://"');
        }
        if (cp.getPeersUrl(first_peer).indexOf('http') >= 0) {
            errors.push('You accidentally typed "http" in your Peer discovery url. It should be "grpc://" or "grpcs://"');
        }
        if (cp.getPeerEventUrl(first_peer).indexOf('http') >= 0) {
            errors.push('You accidentally typed "http" in your Peer events url. It should be "grpc://" or "grpcs://"');
        }

        if (errors.length > 0) {
            console.log('\n');
            logger.warn('----------------------------------------------------------------------');
            logger.warn('------------------------ Close but no cigar --------------------------');
            logger.warn('---------------- You have at least one protocol typo -----------------');
            logger.warn('----------------------------------------------------------------------');
            for (var i in errors) {
                logger.error(errors[i]);
            }
            logger.warn('----------------------------------------------------------------------');
            logger.error('Fix this file: ./config/' + cp.getNetworkCredFileName());
            logger.warn('----------------------------------------------------------------------');
            logger.warn('See this file for help:');
            logger.warn('https://github.com/IBM-Blockchain/marbles/blob/v4.0/docs/config_file.md');
            logger.warn('----------------------------------------------------------------------');
            console.log('\n\n');
            return errors;
        }
        return null;
    };

    // --------------------------------------------------------------------------------------------
    // check if user has changed the settings from the default ones - returns error array when there is a problem
    // --------------------------------------------------------------------------------------------
    helper.checkConfig = function () {
        let errors = [];
        if (cp.getNetworkName() === 'Place Holder Network Name') {
            console.log('\n');
            logger.warn('----------------------------------------------------------------------');
            logger.warn('----------------------------- Hey Buddy! -----------------------------');
            logger.warn('------------------------- It looks like you --------------------------');
            logger.error('----------------------------- skipped -------------------------------');
            logger.warn('------------------------- some instructions --------------------------');
            logger.warn('----------------------------------------------------------------------');
            logger.warn('Your network config JSON has a network name of "Place Holder Network Name"...');
            logger.warn('I\'m afraid you cannot use the default settings as is.');
            logger.warn('These settings must be edited to point to YOUR network.');
            logger.warn('----------------------------------------------------------------------');
            logger.error('Fix this file: ./config/' + cp.getNetworkCredFileName());
            logger.warn('It must have credentials/hostnames/ports/channels/etc for YOUR network');
            logger.warn('How/where would I get that info? Are you using the IBM Cloud Blockchain service? Then look at these instructions(near the end): ');
            logger.warn('https://github.com/IBM-Blockchain/marbles/blob/v4.0/docs/install_chaincode.md');
            logger.warn('----------------------------------------------------------------------');
            console.log('\n\n');
            errors.push('Using default values');
            return errors;
        }
        return helper.check_for_missing();    //run the next check
    };

    return helper;
}; 
```

#### 创建 other.js

创建 other.js 文件并编辑，主要用于从连接概要文件中获取其它的信息，如延时信息，kv 存储路径信息。

```js
$ vim other.js 
```

other.js 文件完整内容如下

```js
// ============================================================================================================================
//     Get *** fields from connection profile data
// ============================================================================================================================
var path = require('path');
var os = require('os');

module.exports = function (cp, logger) {
    var helper = {};

    // ----------------------------------------------------------
    // get the chaincode id on network
    // ----------------------------------------------------------
    helper.getBlockDelay = function () {
        let ret = 1000;
        var channel = cp.getChannelId();
        if (cp.creds.channels && cp.creds.channels[channel] && cp.creds.channels[channel]['x-blockDelay']) {
            if (!isNaN(cp.creds.channels[channel]['x-blockDelay'])) {
                ret = cp.creds.channels[channel]['x-blockDelay'];
            }
        }
        return ret;
    };

    // ----------------------------------------------------------
    // get key value store location
    // ----------------------------------------------------------
    helper.getKvsPath = function (opts) {
        const id = cp.makeUniqueId();
        const default_path = path.join(os.homedir(), '.hfc-key-store/', id);

        if (opts && opts.going2delete) {    //if this is for a delete, return default so we don't wipe a kvs someone setup
            return default_path;    //do the default one
        }

        // -- Using Custom KVS -- //
        if (cp.creds.client && cp.creds.client.credentialStore) {
            const kvs_path = cp.creds.client.credentialStore.path;
            let ret = path.join(__dirname, '../../../config/' + kvs_path + '/');
            if (kvs_path.indexOf('/') === 0) {
                ret = kvs_path;        //its an absolute path
            }
            if (ret.indexOf('$HOME') >= 0) {
                ret = ret.replace('$HOME', os.homedir()).substr(1);
            }
            return ret;            //use the kvs provided in the json
        } else {
            return default_path;    //make a new kvs folder in the home dir
        }
    };

    return helper;
}; 
```