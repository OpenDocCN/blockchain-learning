# 三、.1 sdk-node 之链码安装及实例化

## 从零到壹实现 Marbles 资产管理系统 （Fabric-SDK-Node）之－链码安装及实例化

### 创建目录

在 utils 目录下创建 fc_wrangler/parts 子目录

```js
$ cd $HOME/kevin-marbles/utils
$ mkdir -p fc_wrangler/parts && cd fc_wrangler/parts 
```

在 utils/fc_wrangler/parts 目录下创建 common.js、deploy_cc.js、enrollment.js 文件并编辑

*   **common.js：** Fabric 客户端助手公共库，对信息进行处理。
*   **deploy_cc.js：**链码部署库
*   **enrollment.js：** 使用 Hyperledger Fabric Client 对用户的 KVS 进行管理。

### 创建 common.js

```js
$ vim common.js 
```

common.js 文件内容如下

```js
//-------------------------------------------------------------------
// Common/Helper Library for Fabric Client Wrangler
//-------------------------------------------------------------------
var fs = require('fs');
var path = require('path');

module.exports = function (logger) {
    var common = {};

    //------------------------------------------------------------------
    // Format Chaincode String Error Message
    //------------------------------------------------------------------
    common.format_error_msg = function (error_message) {
        var temp = {
            parsed: 'could not format error',
            raw: error_message
        };
        try {
            if (typeof error_message === 'object') {
                temp.parsed = error_message[0].toString();
            } else {
                temp.parsed = error_message.toString();
            }
            let pos = temp.parsed.lastIndexOf('::');
            if (pos === -1) pos = temp.parsed.lastIndexOf(':');
            if (pos >= 0) temp.parsed = temp.parsed.substring(pos + 2);
        }
        catch (e) {
            logger.error('[fcw] could not format error');
        }
        temp.parsed = 'Blockchain network error - ' + temp.parsed;
        return temp;
    };

    //-----------------------------------------------------------------
    // Create array - takes in array of strings poops out array of objects
    //-----------------------------------------------------------------
    common.fmt_peers = function (urls) {
        var ret = [];

        for (var i in urls) {
            ret.push({ peer_url: urls[i] });    //dumb needs to be an obj
        }

        if (ret.length === 0) {    //while we are here, lets see if its decent
            var err_msg = 'could not create peer array object from peer urls';
            logger.error('[fcw] Error', err_msg, urls);
            throw err_msg;
        }
        return ret;
    };

    //-----------------------------------------------------------------
    // Check Proposal Response
    //-----------------------------------------------------------------
    common.check_proposal_res = function (results, endorsed_hook) {
        var proposalResponses = results[0];
        var proposal = results[1];
        var header = results[2];

        //check response
        if (!proposalResponses || !proposalResponses[0] || !proposalResponses[0].response || proposalResponses[0].response.status !== 200) {
            if (endorsed_hook) endorsed_hook('failed');
            logger.error('[fcw] Failed to obtain endorsement for transaction.', proposalResponses);
            throw proposalResponses;
        }
        else {
            logger.debug('[fcw] Successfully obtained transaction endorsement');

            //call optional endorsement hook
            if (endorsed_hook) endorsed_hook(null, proposalResponses[0].response);

            //move on to ordering
            var request = {
                proposalResponses: proposalResponses,
                proposal: proposal,
                header: header
            };
            return request;
        }
    };

    //------------------------------------------------------------
    // decode base64 into string
    //------------------------------------------------------------
    common.decodeb64 = function (b64string) {
        if (!b64string) throw Error('cannot decode something that isn\'t there');
        return (Buffer.from(b64string, 'base64')).toString();
    };

    //------------------------------------------------------------
    // Delete a folder - synchronous
    //------------------------------------------------------------
    common.rmdir = function (dir_path) {
        if (fs.existsSync(dir_path)) {
            fs.readdirSync(dir_path).forEach(function (entry) {
                var entry_path = path.join(dir_path, entry);
                if (fs.lstatSync(entry_path).isDirectory()) {
                    common.rmdir(entry_path);
                }
                else {
                    fs.unlinkSync(entry_path);
                }
            });
            fs.rmdirSync(dir_path);
        }
    };

    return common;
}; 
```

### 创建 enrollment.js

我们通过在 enrollment.js 文件中创建 SDK 实例，并且根据指定的键值存储路径获取通过 CA 颁发的注册证书。

创建 enrollment.js 文件并编辑

```js
$ vim enrollment.js 
```

enrollment.js 文件内容如下

```js
//-------------------------------------------------------------------
// Enrollment HFC Library
//-------------------------------------------------------------------

module.exports = function (logger) {
    var FabricClient = require('fabric-client');
    var path = require('path');
    var common = require(path.join(__dirname, './common.js'))(logger);
    var enrollment = {};
    var User = require('fabric-client/lib/User.js');
    var CaService = require('fabric-ca-client/lib/FabricCAClientImpl.js');
    var Orderer = require('fabric-client/lib/Orderer.js');
    var Peer = require('fabric-client/lib/Peer.js');
    FabricClient.setConfigSetting('request-timeout', 90000);

    enrollment.enroll = function (options, cb) {
        var client = new FabricClient();
        var channel = client.newChannel(options.channel_id);

        var debug = {                                                        
            peer_urls: options.peer_urls,
            channel_id: options.channel_id,
            uuid: options.uuid,
            ca_url: options.ca_url,
            orderer_url: options.orderer_url,
            enroll_id: options.enroll_id,
            enroll_secret: options.enroll_secret,
            msp_id: options.msp_id,
            kvs_path: options.kvs_path
        };
        logger.info('[fcw] Going to enroll', debug);

        // Make eCert kvs (Key Value Store)
        FabricClient.newDefaultKeyValueStore({
            path: options.kvs_path         //store crypto in the kvs directory
        }).then(function (store) {
            client.setStateStore(store);
            return getSubmitter(client, options);    //do most of the work here
        }).then(function (submitter) {

            channel.addOrderer(new Orderer(options.orderer_url, options.orderer_tls_opts));

            channel.addPeer(new Peer(options.peer_urls[0], options.peer_tls_opts));
            logger.debug('added peer', options.peer_urls[0]);

            // --- Success --- //
            logger.debug('[fcw] Successfully got enrollment ' + options.uuid);
            if (cb) cb(null, { client: client, channel: channel, submitter: submitter });
            return;

        }).catch(function (err) {

            // --- Failure --- //
            logger.error('[fcw] Failed to get enrollment ' + options.uuid, err.stack ? err.stack : err);
            var formatted = common.format_error_msg(err);

            if (cb) cb(formatted);
            return;
        });
    };

    // Get Submitter - ripped this function off from fabric-client
    function getSubmitter(client, options) {
        var member;
        return client.getUserContext(options.enroll_id, true).then((user) => {
            if (user && user.isEnrolled()) {
                if (user._mspId !== options.msp_id) {                                        //if they don't match, this isn't our user! can't use it
                    logger.warn('[fcw] The msp id in KVS does not match the msp id passed to enroll. Need to clear the KVS.', user._mspId, options.msp_id);
                    common.rmdir(options.kvs_path);                                            //delete it
                    logger.error('[fcw] MSP in KVS mismatch. KVS has been deleted. Restart the app to try again.');
                    process.exit();                                                            //this is terrible, but can't seem to reset client._userContext
                } else {
                    logger.info('[fcw] Successfully loaded enrollment from persistence');    //load from KVS if we can
                    return user;
                }
            } else {

                // Need to enroll it with the CA
                var tlsOptions = {
                    trustedRoots: [options.ca_tls_opts.pem],                                //pem cert required
                    verify: false
                };
                var ca_client = new CaService(options.ca_url, tlsOptions, options.ca_name);    //ca_name is important for the IBM Cloud service
                member = new User(options.enroll_id);

                logger.debug('enroll id: "' + options.enroll_id + '", secret: "' + options.enroll_secret + '"');
                logger.debug('msp_id: ', options.msp_id, 'ca_name:', options.ca_name);

                // --- Lets Do It --- //
                return ca_client.enroll({
                    enrollmentID: options.enroll_id,
                    enrollmentSecret: options.enroll_secret

                }).then((enrollment) => {

                    // Store Certs
                    logger.info('[fcw] Successfully enrolled user \'' + options.enroll_id + '\'');
                    return member.setEnrollment(enrollment.key, enrollment.certificate, options.msp_id);
                }).then(() => {

                    // Save Submitter Enrollment
                    return client.setUserContext(member);
                }).then(() => {

                    // Return Submitter Enrollment
                    return member;
                }).catch((err) => {

                    // Send Errors
                    logger.error('[fcw] Failed to enroll and persist user. Error: ' + err.stack ? err.stack : err);
                    throw new Error('Failed to obtain an enrolled user');
                });
            }
        });
    }

    enrollment.enrollWithAdminCert = function (options, cb) {
        var client = new FabricClient();
        var channel = client.newChannel(options.channel_id);

        var debug = {                                                        // this is just for console printing, no PEM here
            peer_urls: options.peer_urls,
            channel_id: options.channel_id,
            uuid: options.uuid,
            orderer_url: options.orderer_url,
            msp_id: options.msp_id,
        };
        logger.info('[fcw] Going to enroll with admin cert! ', debug);

        // Make eCert kvs (Key Value Store)
        FabricClient.newDefaultKeyValueStore({
            path: options.kvs_path                                                     //get eCert in the kvs directory
        }).then(function (store) {
            client.setStateStore(store);
            return getSubmitterWithAdminCert(client, options);                        //admin cert is different
        }).then(function (submitter) {

            channel.addOrderer(new Orderer(options.orderer_url, options.orderer_tls_opts));

            channel.addPeer(new Peer(options.peer_urls[0], options.peer_tls_opts));    //add the first peer
            logger.debug('added peer', options.peer_urls[0]);

            // --- Success --- //
            logger.debug('[fcw] Successfully got enrollment ' + options.uuid);
            if (cb) cb(null, { client: client, channel: channel, submitter: submitter });
            return;

        }).catch(function (err) {

            // --- Failure --- //
            logger.error('[fcw] Failed to get enrollment ' + options.uuid, err.stack ? err.stack : err);
            var formatted = common.format_error_msg(err);

            if (cb) cb(formatted);
            return;
        });
    };

    // Get Submitter - ripped this function off from helper.js in fabric-client
    function getSubmitterWithAdminCert(client, options) {
        return Promise.resolve(client.createUser({
            username: options.msp_id,
            mspid: options.msp_id,
            cryptoContent: {
                privateKeyPEM: options.privateKeyPEM,
                signedCertPEM: options.signedCertPEM
            }
        }));
    }

    return enrollment;
}; 
```

### 创建 deploy_cc.js

创建 deploy_cc.js 文件并编辑，主要用于安装、实例化、升级链码

```js
$ vim deploy_cc.js 
```

deploy_cc.js 文件内容如下

```js
//-------------------------------------------------------------------
// Install + Instantiate + Upgrade Chaincode
//-------------------------------------------------------------------
var path = require('path');

module.exports = function (logger) {
    var common = require(path.join(__dirname, './common.js'))(logger);
    //var Peer = require('fabric-client/lib/Peer.js');
    var deploy_cc = {};

    //-------------------------------------------------------------------
    // Install Chaincode - Must use Admin Cert enrollment
    //-------------------------------------------------------------------
    deploy_cc.install_chaincode = function (obj, options, cb) {
        logger.debug('[fcw] Installing Chaincode');
        var client = obj.client;

        // fix GOPATH - does not need to be real!
        process.env.GOPATH = path.join(__dirname, '../../../chaincode');

        // send proposal to endorser
        var request = {
            targets: [client.newPeer(options.peer_urls[0], options.peer_tls_opts)],
            chaincodePath: options.path_2_chaincode,
            chaincodeId: options.chaincode_id,
            chaincodeVersion: options.chaincode_version,
        };
        logger.debug('[fcw] Sending install req', request);

        client.installChaincode(request).then(function (results) {

            // --- Check Install Response --- //
            common.check_proposal_res(results, options.endorsed_hook);
            if (cb) return cb(null, results);

        }).catch(function (err) {
            // --- Errors --- //
            logger.error('[fcw] Error in install catch block', typeof err, err);
            var formatted = common.format_error_msg(err);
            if (cb) return cb(formatted, null);
            else return;
        });
    };

    //-------------------------------------------------------------------
    // Instantiate Chaincode - Must use Admin Cert enrollment
    //-------------------------------------------------------------------
    deploy_cc.instantiate_chaincode = function (obj, options, cb) {
        logger.debug('[fcw] Instantiating Chaincode', options);
        var channel = obj.channel;
        var client = obj.client;

        // send proposal to endorser
        var request = {
            targets: [client.newPeer(options.peer_urls[0], options.peer_tls_opts)],
            chaincodeId: options.chaincode_id,
            chaincodeVersion: options.chaincode_version,
            fcn: 'init',
            args: options.cc_args,
            txId: client.newTransactionID(),
        };
        logger.debug('[fcw] Sending instantiate req', request);

        channel.initialize().then(() => {
            channel.sendInstantiateProposal(request
                //nothing
            ).then(
                function (results) {

                    //check response
                    var request = common.check_proposal_res(results, options.endorsed_hook);
                    return channel.sendTransaction(request);
                }
                ).then(
                function (response) {

                    // All good
                    if (response.status === 'SUCCESS') {
                        logger.debug('[fcw] Successfully ordered instantiate endorsement.');

                        // Call optional order hook
                        if (options.ordered_hook) options.ordered_hook(null, request.txId.toString());

                        setTimeout(function () {
                            if (cb) return cb(null);
                            else return;
                        }, 5000);
                    }

                    // No good
                    else {
                        logger.error('[fcw] Failed to order the instantiate endorsement.');
                        throw response;
                    }
                }
                ).catch(
                function (err) {
                    logger.error('[fcw] Error in instantiate catch block', typeof err, err);
                    var formatted = common.format_error_msg(err);

                    if (cb) return cb(formatted, null);
                    else return;
                }
                );
        });
    };

    //-------------------------------------------------------------------
    // Upgrade Chaincode - Must use Admin Cert enrollment
    //-------------------------------------------------------------------
    deploy_cc.upgrade_chaincode = function (obj, options, cb) {
        logger.debug('[fcw] Upgrading Chaincode', options);
        var channel = obj.channel;
        var client = obj.client;

        // send proposal to endorser
        var request = {
            targets: [client.newPeer(options.peer_urls[0], options.peer_tls_opts)],
            chaincodeId: options.chaincode_id,
            chaincodeVersion: options.chaincode_version,
            fcn: 'init',
            args: options.cc_args,
            txId: client.newTransactionID(),
        };
        logger.debug('[fcw] Sending upgrade cc req', request);

        channel.initialize().then(() => {
            channel.sendUpgradeProposal(request
                //nothing
            ).then(
                function (results) {

                    //check response
                    var request = common.check_proposal_res(results, options.endorsed_hook);
                    return channel.sendTransaction(request);
                }
                ).then(
                function (response) {

                    // All good
                    if (response.status === 'SUCCESS') {
                        logger.debug('[fcw] Successfully ordered upgrade cc endorsement.');

                        // Call optional order hook
                        if (options.ordered_hook) options.ordered_hook(null, request.txId.toString());

                        setTimeout(function () {
                            if (cb) return cb(null);
                            else return;
                        }, 5000);
                    }

                    // No good
                    else {
                        logger.error('[fcw] Failed to order the upgrade cc endorsement.');
                        throw response;
                    }
                }
                ).catch(
                function (err) {
                    logger.error('[fcw] Error in upgrade cc catch block', typeof err, err);
                    var formatted = common.format_error_msg(err);

                    if (cb) return cb(formatted, null);
                    else return;
                }
                );
        });
    };

    return deploy_cc;
}; 
```

### 创建 index.js

在 utils/fc_wrangler 目录下创建 index.js 文件并编辑，通过此文件完成对链码的调用，实现对链码的安装及实例化，后面也使用此文件实现对分类账本的数据状态进行操作的入口。

```js
$ cd $HOME/kevin-marbles/utils/fc_wrangler/
$ vim index.js 
```

index.js 文件完整内容如下

```js
//-------------------------------------------------------------------
// Fabric Client Wrangler - Wrapper library for the Hyperledger Fabric Client SDK
//-------------------------------------------------------------------

module.exports = function (g_options, logger) {
    var deploy_cc = require('./parts/deploy_cc.js')(logger);
    var enrollment = require('./parts/enrollment.js')(logger);
    var fcw = {};

    // ------------------------------------------------------------------------
    // Chaincode Functions
    // ------------------------------------------------------------------------

    // Install Chaincode
    fcw.install_chaincode = function (obj, options, cb_done) {
        deploy_cc.install_chaincode(obj, options, cb_done);
    };

    // Instantiate Chaincode
    fcw.instantiate_chaincode = function (obj, options, cb_done) {
        deploy_cc.instantiate_chaincode(obj, options, cb_done);
    };

    // Upgrade Chaincode
    fcw.upgrade_chaincode = function (obj, options, cb_done) {
        deploy_cc.upgrade_chaincode(obj, options, cb_done);
    };

    // ------------------------------------------------------------------------
    // Enrollment Functions
    // ------------------------------------------------------------------------

    // enroll an enrollId with the ca
    fcw.enroll = function (options, cb_done) {
        let opts = ha.get_ca(options);
        enrollment.enroll(opts, function (err, resp) {
            if (err != null) {
                opts = ha.get_next_ca(options);        //try another CA
                if (opts) {
                    logger.info('Retrying enrollment on different ca');
                    fcw.enroll(options, cb_done);
                } else {
                    if (cb_done) cb_done(err, resp);    //out of CAs, give up
                }
            } else {
                ha.success_ca_position = ha.using_ca_position;            //remember the last good one
                if (cb_done) cb_done(err, resp);
            }
        });
    };

    // enroll with admin cert
    fcw.enrollWithAdminCert = function (options, cb_done) {
        enrollment.enrollWithAdminCert(options, cb_done);
    };

    return fcw;
}; 
```