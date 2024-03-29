# 2.3 设置环境-连接概要库入口

### 进入 utils 目录

返回 utils 目录下

```js
$ cd $HOME/kevin-marbles/utils 
```

#### 创建 misc.js 并编辑

检查区块可靠性文件是否正确，创建唯一 ID 串

```js
$ vim misc.js 
```

misc.js 文件完整内容如下：

```js
var fs = require('fs');
var path = require('path');

module.exports = function (logger) {
    var misc = {};
    let cp_path = null;
    var detect_env = require(path.join(__dirname, './connection_profile_lib/parts/detect_env.js'))(logger);

    // Check if blockchain creds files are okay
    misc.check_creds_for_valid_json = function (cb) {
        if (!detect_env.getConnectionProfileFromEnv()) {
            if (!process.env.creds_filename) {
                process.env.creds_filename = 'marbles_local.json';                //default to a file
            }

            var config_path = path.join(__dirname, '../config/' + process.env.creds_filename);
            try {
                let configFile = require(config_path);
                cp_path = path.join(__dirname, '../config/' + configFile.cred_filename);
                let creds = require(cp_path);
                if (creds.name) {
                    logger.info('Checking connection profile is done');
                    return null;
                } else {
                    throw 'missing network id';
                }
            } catch (e) {
                logger.error('---------------------------------------------------------------');
                logger.error('----------------------------- Bah -----------------------------');
                logger.error('---------- The connection profile file is malformed -----------');
                logger.error('---------------------------------------------------------------');
                logger.error('Fix this file: ' + cp_path);
                logger.warn('It must be valid JSON. You may have dropped a comma or added one too many?');
                logger.warn('----------------------------------------------------------------------');
                logger.error(e);
                process.exit();        //all stop
            }
        }
    };

    // Create Random integer between and including min-max
    misc.getRandomInt = function (min, max) {
        min = Math.ceil(min);
        max = Math.floor(max);
        return Math.floor(Math.random() * (max - min)) + min;
    };

    // Create a Random string of x length
    misc.randStr = function (length) {
        var text = '';
        var possible = 'abcdefghijkmnpqrstuvwxyz0123456789';
        for (var i = 0; i < length; i++) text += possible.charAt(Math.floor(Math.random() * possible.length));
        return text;
    };

    // Real simple hash
    misc.simple_hash = function (a_string) {
        var hash = 0;
        for (var i in a_string) hash ^= a_string.charCodeAt(i);
        return hash;
    };

    // Sanitize marble owner names
    misc.saferNames = function (usernames) {
        var ret = [];
        for (var i in usernames) {
            var name = usernames[i].replace(/\W+/g, '');    // names should not contain many things...
            if (name !== '') ret.push(name.toLowerCase());
        }
        return ret;
    };

    // Sanitize string for filesystem
    misc.saferString = function (str) {
        let ret = '';
        if (str && typeof str === 'string') {
            ret = str.replace(/\W+/g, '');
        }
        return ret;
    };

    // Sanitize company names
    misc.saferCompanyNames = function (companyName) {
        var name = companyName.replace(/[^A-Za-z0-9_-~!@#$%^&*+=;<>{}\s]/g, '');            // names should not contain many things...
        return name;
    };

    // Delete a folder
    misc.rmdir = function (dir_path) {
        if (fs.existsSync(dir_path)) {
            fs.readdirSync(dir_path).forEach(function (entry) {
                var entry_path = path.join(dir_path, entry);
                if (fs.lstatSync(entry_path).isDirectory()) {
                    misc.rmdir(entry_path);
                }
                else {
                    fs.unlinkSync(entry_path);
                }
            });
            fs.rmdirSync(dir_path);
        }
    };

    return misc;
}; 
```

### 创建 index.js

连接概要库中的各个文件对相关功能进行了封装，我们需要一个入口点来对各个文件进行解析，如：获取网络名称、获取通道、加载证书等功能。因此，在 utils/connection_profile_lib 目录下创建一个 index.js 文件并编辑

```js
$ cd $HOME/kevin-marbles/utils/connection_profile_lib 
$ vim index.js 
```

index.js 文件完整内容如下

```js
// ============================================================================================================================
//     Connection Profile Parsing Library
//
// This file is to help safely parse the Connection Profile JSON data
// The connection profile is either in a file ("../../config/" folder) OR it was provided in the environmental variable "CONNECTION_PROFILE"
//
// - every file in this library's folder is getting rolled into the `cp` object. ie any function defined in those files is accessible to this one.
// - if you need something from the connection profile, chance are its defined somewhere in this folder
// ============================================================================================================================
var fs = require('fs');
var path = require('path');
var os = require('os');

module.exports = function (config_filename, logger) {
    var cp = {};
    var detect_env = require('./parts/detect_env.js')(logger);
    var misc = require('../misc.js')(logger);

    if (!config_filename) {
        config_filename = 'marbles_local.json';                                        // default config file name
    }
    cp.config_path = path.join(__dirname, '../../config/' + config_filename);
    cp.config = require(cp.config_path);                                            // load the config file
    logger.info('Loaded config file', cp.config_path);                                // path to config file

    // --------------------------------------------------------------------------------
    // Detect if connection profile data is in an environmental variable instead of a file
    // --------------------------------------------------------------------------------
    let use_env = detect_env.getConnectionProfileFromEnv();
    if (use_env) {                                                                    // to use env or not to use env
        cp.using_env = true;
        cp.cp_path = 'there-is-no-file-using-env';
        cp.creds = use_env;
        logger.info('Loaded connection profile from an environmental variable');
    } else {                                                                        // not in env, look for the files
        cp.using_env = false;
        cp.cp_path = path.join(__dirname, '../../config/' + cp.config.cred_filename);
        cp.creds = require(cp.cp_path);                                                // load the credential file
        logger.info('Loaded connection profile file', cp.cp_path);                    // path to the blockchain connection profile
    }
    cp.config_filename = config_filename;

    // --------------------------------------------------------------------------------
    // Parse connection profile data to get various things:
    // --------------------------------------------------------------------------------
    // get network id
    cp.getNetworkName = function () {
        return cp.creds.name;
    };

    // get cred file name
    cp.getNetworkCredFileName = function () {
        return cp.config.cred_filename;
    };

    // build the tls options for the sdk
    cp.buildTlsOpts = function (node_obj) {
        let ret = {
            'ssl-target-name-override': null,
            pem: null,
            'grpc.http2.keepalive_time': 300,    //grpc 1.2.4
            'grpc.keepalive_time_ms': 300000,    //grpc 1.3.7
            'grpc.http2.keepalive_timeout': 35,    //grpc 1.2.4
            'grpc.keepalive_timeout_ms': 3500,    //grpc 1.3.7
        };
        if (node_obj) {
            if (node_obj.tlsCACerts) {
                ret.pem = cp.loadPem(node_obj.tlsCACerts);
            }
            if (node_obj.grpcOptions) {
                for (var field in ret) {
                    if (node_obj.grpcOptions[field]) {
                        ret[field] = node_obj.grpcOptions[field];
                    }
                }
            }
        }
        return ret;
    };

    // get the very first channel name from creds
    cp.getFirstChannelId = function () {
        if (cp.creds && cp.creds.channels) {
            var channels = Object.keys(cp.creds.channels);
            if (channels[0]) {
                return channels[0];
            }
        }
        throw Error('No channels found in connection profile... this is problematic. A channel needs to be created before marbles can execute.');
    };

    // get the very first channel name from creds
    cp.getChannelId = function () {
        if (process.env.CHANNEL_ID) {    // if env version is set, use it
            return process.env.CHANNEL_ID;
        } else {
            return cp.getFirstChannelId();    // else just grab the first one we got
        }
    };

    // load cert from file path OR just pass cert back
    cp.loadPem = function (obj) {
        if (obj && obj.path) {    // looks like field is a path to a file
            var path2cert = path.join(__dirname, '../../config/' + obj.path);
            if (obj.path.indexOf('/') === 0) {
                path2cert = obj.path;    //its an absolute path
            }
            if (path2cert.indexOf('$HOME') >= 0) {
                path2cert = path2cert.replace('$HOME', os.homedir()).substr(1);
            }
            logger.debug('loading pem from a path: ' + path2cert);
            return fs.readFileSync(path2cert, 'utf8') + '\r\n';         //read from file, LOOKING IN config FOLDER
        } else if (obj.pem) {    // looks like field is the pem we need
            logger.debug('loading pem from JSON.');
            return obj.pem;
        }
        return null;    //can be null if network is not using TLS
    };

    // safely retrieve marbles config file fields
    cp.getMarblesField = function (marbles_field) {
        try {
            if (cp.config[marbles_field]) {
                return cp.config[marbles_field];
            } else {
                logger.warn('"' + marbles_field + '" not found in config json: ' + cp.config_path);
                return null;
            }
        } catch (e) {
            logger.warn('"' + marbles_field + '" not found in config json: ' + cp.config_path);
            return null;
        }
    };

    // --------------------------------------------------------------------------------
    // Absorb all the functions into the cp object
    // --------------------------------------------------------------------------------
    const chaincode = require('./parts/chaincode.js')(cp, logger);
    const ca = require('./parts/ca.js')(cp, logger);
    const peer = require('./parts/peer.js')(cp, logger);
    const helper = require('./parts/helper.js')(cp, logger);
    const orderer = require('./parts/orderer.js')(cp, logger);
    const other = require('./parts/other.js')(cp, logger);
    for (const func in chaincode) cp[func] = chaincode[func];
    for (const func in ca) cp[func] = ca[func];
    for (const func in peer) cp[func] = peer[func];
    for (const func in helper) cp[func] = helper[func];
    for (const func in orderer) cp[func] = orderer[func];
    for (const func in other) cp[func] = other[func];
    const org = require('./parts/org.js')(cp, logger);
    for (const func in org) cp[func] = org[func];

    console.log('\n\n\nConnection Profile Lib Functions:()');
    for (let i in cp) {
        if (typeof cp[i] === 'function') console.log('  ' + i + '()');
    }
    const verification = require('./parts/verification.js')(cp, logger);            // do this one last, it needs the others roll in first
    for (const func in verification) cp[func] = verification[func];
    // --------------------------------------------------------------------------------
    // --------------------------------------------------------------------------------
    // Build CP Options
    // --------------------------------------------------------------------------------
    // make a unique id for the sample & channel & org & peer
    cp.makeUniqueId = function () {
        const net_name = cp.getNetworkName();
        const channel = cp.getChannelId();
        const org = cp.getClientOrg();
        const first_peer = cp.getFirstPeerName(channel);
        return misc.saferString('marbles-' + net_name + channel + org + first_peer);
    };

    // build the marbles lib module options
    cp.makeMarblesLibOptions = function () {
        const channel = cp.getChannelId();
        const org_2_use = cp.getClientOrg();
        const first_ca = cp.getFirstCaName(org_2_use);
        const first_peer = cp.getFirstPeerName(channel);
        const first_orderer = cp.getFirstOrdererName(channel);
        return {
            block_delay: cp.getBlockDelay(),
            channel_id: cp.getChannelId(),
            chaincode_id: cp.getChaincodeId(),
            event_urls: (cp.getEventsSetting()) ? cp.getAllPeerUrls(channel).eventUrls : null,    //null is important
            chaincode_version: cp.getChaincodeVersion(),
            ca_tls_opts: cp.getCaTlsCertOpts(first_ca),
            orderer_tls_opts: cp.getOrdererTlsCertOpts(first_orderer),
            peer_tls_opts: cp.getPeerTlsCertOpts(first_peer),
            peer_urls: cp.getAllPeerUrls(channel).urls,
        };
    };

    // build the enrollment options for using an enroll ID
    cp.makeEnrollmentOptions = function (userIndex) {
        if (userIndex === undefined || userIndex == null) {
            throw new Error('User index not passed');
        } else {
            const channel = cp.getChannelId();
            const org_2_use = cp.getClientOrg();
            const first_ca = cp.getFirstCaName(org_2_use);
            const first_peer = cp.getFirstPeerName(channel);
            const first_orderer = cp.getFirstOrdererName(channel);
            const org_name = cp.getClientOrg();
            const user_obj = cp.getEnrollObj(first_ca, userIndex);        //there may be multiple users
            return {
                channel_id: channel,
                uuid: cp.makeUniqueId(),
                ca_urls: cp.getAllCaUrls(),
                ca_name: cp.getCaName(first_ca),
                orderer_url: cp.getOrderersUrl(first_orderer),
                peer_urls: [cp.getPeersUrl(first_peer)],
                enroll_id: user_obj.enrollId,
                enroll_secret: user_obj.enrollSecret,
                msp_id: org_name,
                ca_tls_opts: cp.getCaTlsCertOpts(first_ca),
                orderer_tls_opts: cp.getOrdererTlsCertOpts(first_orderer),
                peer_tls_opts: cp.getPeerTlsCertOpts(first_peer),
                kvs_path: cp.getKvsPath()
            };
        }
    };

    // build the enrollment options using an admin cert
    cp.makeEnrollmentOptionsUsingCert = function () {
        const channel = cp.getChannelId();
        const first_peer = cp.getFirstPeerName(channel);
        const first_orderer = cp.getFirstOrdererName(channel);
        const org_name = cp.getClientOrg();
        return {
            channel_id: channel,
            uuid: cp.makeUniqueId(),
            orderer_url: cp.getOrderersUrl(first_orderer),
            peer_urls: [cp.getPeersUrl(first_peer)],
            msp_id: org_name,
            privateKeyPEM: cp.getAdminPrivateKeyPEM(org_name),
            signedCertPEM: cp.getAdminSignedCertPEM(org_name),
            orderer_tls_opts: cp.getOrdererTlsCertOpts(first_orderer),
            peer_tls_opts: cp.getPeerTlsCertOpts(first_peer),
            kvs_path: cp.getKvsPath()
        };
    };

    // write new settings to config files
    cp.write = function (obj) {
        console.log('saving the creds file has been disabled temporarily');
        var creds_file = cp.creds;
        const channel = cp.getChannelId();
        const org_2_use = cp.getClientOrg();
        const first_peer = cp.getFirstPeerName(channel);
        const first_ca = cp.getFirstCaName(org_2_use);
        const first_orderer = cp.getFirstOrdererName(channel);

        try {
            creds_file = JSON.parse(fs.readFileSync(cp.cp_path, 'utf8'));
        } catch (e) {
            logger.error('file not found', cp.cp_path, e);
        }

        if (obj.ordererUrl) {
            creds_file.orderers[first_orderer].url = obj.ordererUrl;
        }
        if (obj.peerUrl) {
            creds_file.peers[first_peer].url = obj.peerUrl;
        }
        if (obj.caUrl) {
            creds_file.certificateAuthorities[first_ca].url = obj.caUrl;
        }
        if (obj.chaincodeId) {
            let version = cp.getChaincodeVersion();
            if (obj.chaincodeVersion) {     // changing both id and version
                version = obj.chaincodeVersion;
            }
            creds_file.channels[channel].chaincodes = [obj.chaincodeId + ':' + version];
        }
        if (obj.chaincodeVersion) {
            let chaincodeId = cp.getChaincodeId();
            if (obj.chaincodeId) {    // changing both id and version
                chaincodeId = obj.chaincodeId;
            }
            creds_file.channels[channel].chaincodes = [chaincodeId + ':' + obj.chaincodeVersion];
        }
        if (obj.channelId) {
            const old_channel_obj = JSON.parse(JSON.stringify(creds_file.channels[channel]));
            creds_file.channels = {};
            creds_file.channels[obj.channelId] = old_channel_obj;
        }
        if (obj.enrollId && obj.enrollSecret) {
            creds_file.certificateAuthorities[first_ca].registrar[0] = {
                enrollId: obj.enrollId,
                enrollSecret: obj.enrollSecret
            };
        }

        try {
            fs.writeFileSync(cp.cp_path, JSON.stringify(creds_file, null, 4), 'utf8');    //save to file
        } catch (e) {
            logger.error('could not write file', cp.cp_path, e);
        }
        cp.creds = creds_file;                                                            //replace old copy
    };

    return cp;
}; 
```