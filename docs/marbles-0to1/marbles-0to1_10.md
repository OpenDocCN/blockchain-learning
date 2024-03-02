# 三、.2 调用链码安装及实例化脚本

## 从零到壹实现 Marbles 资产管理系统 （Fabric-SDK-Node）之－链码安装及实例化调用脚本

在项目根目录下创建 script 目录，用于存放脚本文件

```js
$ cd $HOME/kevin-marbles
$ mkdir scripts && cd scripts 
```

### 安装链码 install_chaincode.js

创建 install_chaincode.js 文件并编辑

```js
$ vim install_chaincode.js 
```

install_chaincode.js 文件完整内容如下

```js
// ============================================================================================================================
//     Install Chaincode
// This file shows how to install chaincode onto a Hyperledger Fabric Peer via the SDK + FC Wrangler
// ============================================================================================================================
var winston = require('winston');    //logger module
var path = require('path');
var logger = new (winston.Logger)({
    level: 'debug',
    transports: [
        new (winston.transports.Console)({ colorize: true }),
    ]
});

// --- Set Details Here --- //
var config_file = 'marbles_local.json';        //set config file name
var chaincode_id = 'marbles';    //set desired chaincode id to identify this chaincode
var chaincode_ver = 'v4';        //set desired chaincode version

//  --- Use (optional) arguments if passed in --- //
var args = process.argv.slice(2);
if (args[0]) {
    config_file = args[0];
    logger.debug('Using argument for config file', config_file);
}
if (args[1]) {
    chaincode_id = args[1];
    logger.debug('Using argument for chaincode id');
}
if (args[2]) {
    chaincode_ver = args[2];
    logger.debug('Using argument for chaincode version');
}

var cp = require(path.join(__dirname, '../utils/connection_profile_lib/index.js'))(config_file, logger);            //set the config file name here
var fcw = require(path.join(__dirname, '../utils/fc_wrangler/index.js'))({ block_delay: cp.getBlockDelay() }, logger);

console.log('---------------------------------------');
logger.info('Lets install some chaincode -', chaincode_id, chaincode_ver);
console.log('---------------------------------------');

logger.info('First we enroll');
fcw.enrollWithAdminCert(cp.makeEnrollmentOptionsUsingCert(), function (enrollErr, enrollResp) {
    if (enrollErr != null) {
        logger.error('error enrolling', enrollErr, enrollResp);
    } else {
        console.log('---------------------------------------');
        logger.info('Now we install');
        console.log('---------------------------------------');

        const channel = cp.getChannelId();
        const first_peer = cp.getFirstPeerName(channel);
        var opts = {
            peer_urls: [cp.getPeersUrl(first_peer)],
            path_2_chaincode: 'marbles',
            chaincode_id: chaincode_id,
            chaincode_version: chaincode_ver,
            peer_tls_opts: cp.getPeerTlsCertOpts(first_peer)
        };
        fcw.install_chaincode(enrollResp, opts, function (err, resp) {
            console.log('---------------------------------------');
            logger.info('Install done. Errors:', (!err) ? 'nope' : err);
            console.log('---------------------------------------');
        });
    }
}); 
```

### 实例化链码 instantiate_chaincode.js

在当前的 script 目录中创建 instantiate_chaincode.js 文件并编辑

```js
$ vim instantiate_chaincode.js 
```

instantiate_chaincode.js 文件完整内容如下

```js
// ============================================================================================================================
//     Instantiate Chaincode
// This file shows how to instantiate chaincode onto a Hyperledger Fabric Channel via the SDK + FC Wrangler
// ============================================================================================================================
var winston = require('winston');    //logger module
var path = require('path');
var logger = new (winston.Logger)({
    level: 'debug',
    transports: [
        new (winston.transports.Console)({ colorize: true }),
    ]
});

// --- Set Details Here --- //
var config_file = 'marbles_local.json';    //set config file name
var chaincode_id = 'marbles';    //use same ID during the INSTALL proposal
var chaincode_ver = 'v4';    //use same version during the INSTALL proposal

//  --- Use (optional) arguments if passed in --- //
var args = process.argv.slice(2);
if (args[0]) {
    config_file = args[0];
    logger.debug('Using argument for config file', config_file);
}
if (args[1]) {
    chaincode_id = args[1];
    logger.debug('Using argument for chaincode id');
}
if (args[2]) {
    chaincode_ver = args[2];
    logger.debug('Using argument for chaincode version');
}

var cp = require(path.join(__dirname, '../utils/connection_profile_lib/index.js'))(config_file, logger);            //set the config file name here
var fcw = require(path.join(__dirname, '../utils/fc_wrangler/index.js'))({ block_delay: cp.getBlockDelay() }, logger);

console.log('---------------------------------------');
logger.info('Lets instantiate some chaincode -', chaincode_id, chaincode_ver);
console.log('---------------------------------------');
logger.warn('Note: the chaincode should have been installed before running this script');

logger.info('First we enroll');
fcw.enrollWithAdminCert(cp.makeEnrollmentOptionsUsingCert(), function (enrollErr, enrollResp) {
    if (enrollErr != null) {
        logger.error('error enrolling', enrollErr, enrollResp);
    } else {
        console.log('---------------------------------------');
        logger.info('Now we instantiate');
        console.log('---------------------------------------');

        const channel = cp.getChannelId();
        const first_peer = cp.getFirstPeerName(channel);
        var opts = {
            peer_urls: [cp.getPeersUrl(first_peer)],
            channel_id: cp.getChannelId(),
            chaincode_id: chaincode_id,
            chaincode_version: chaincode_ver,
            cc_args: ['12345'],
            peer_tls_opts: cp.getPeerTlsCertOpts(first_peer)
        };
        fcw.instantiate_chaincode(enrollResp, opts, function (err, resp) {
            console.log('---------------------------------------');
            logger.info('Instantiate done. Errors:', (!err) ? 'nope' : err);
            console.log('---------------------------------------');
        });
    }
}); 
```

### 链码升级 upgrade_chaincode.js

创建 upgrade_chaincode.js 文件并编辑

```js
$ vim upgrade_chaincode.js 
```

完整内容如下

```js
var winston = require('winston');        //logger module
var path = require('path');
var logger = new (winston.Logger)({
    level: 'debug',
    transports: [
        new (winston.transports.Console)({ colorize: true }),
    ]
});

// --- Set Details Here --- //
var config_file = 'marbles_local.json';        //set config file name
var chaincode_id = 'marbles01';    //use same ID during the PREVIOUS instantiate proposal
var chaincode_ver = 'v5';    //use same version during the INSTALL proposal

//  --- Use (optional) arguments if passed in --- //
var args = process.argv.slice(2);
if (args[0]) {
    config_file = args[0];
    logger.debug('Using argument for config file', config_file);
}
if (args[1]) {
    chaincode_id = args[1];
    logger.debug('Using argument for chaincode id');
}
if (args[2]) {
    chaincode_ver = args[2];
    logger.debug('Using argument for chaincode version');
}

var cp = require(path.join(__dirname, '../utils/connection_profile_lib/index.js'))(config_file, logger);            //set the config file name here
var fcw = require(path.join(__dirname, '../utils/fc_wrangler/index.js'))({ block_delay: cp.getBlockDelay() }, logger);

console.log('---------------------------------------');
logger.info('Lets upgrade some chaincode -', chaincode_id, chaincode_ver);
console.log('---------------------------------------');
logger.warn('Note: the chaincode "' + chaincode_id + '" should have been installed AND instantiated before running this script');
let msg = `Note: the chaincode "` + chaincode_id + `" and version "` + chaincode_ver + `
            should have been installed before running this script`;
logger.warn(msg);

logger.info('First we enroll');
fcw.enrollWithAdminCert(cp.makeEnrollmentOptionsUsingCert(), function (enrollErr, enrollResp) {
    if (enrollErr != null) {
        logger.error('error enrolling', enrollErr, enrollResp);
    } else {
        console.log('---------------------------------------');
        logger.info('Now we upgrade');
        console.log('---------------------------------------');

        const channel = cp.getChannelId();
        const first_peer = cp.getFirstPeerName(channel);
        var opts = {
            peer_urls: [cp.getPeersUrl(first_peer)],
            channel_id: cp.getChannelId(),
            chaincode_id: chaincode_id,
            chaincode_version: chaincode_ver,
            peer_tls_opts: cp.getPeerTlsCertOpts(first_peer),
            cc_args: ['666666'],
        };
        fcw.upgrade_chaincode(enrollResp, opts, function (err, resp) {
            console.log('---------------------------------------');
            logger.info('Upgrade done. Errors:', (!err) ? 'nope' : err);
            console.log('---------------------------------------');
        });
    }
}); 
```