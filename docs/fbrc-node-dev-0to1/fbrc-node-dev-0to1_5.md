# 第五章 应用 sdk-node－链码调用

## 从零到壹构建基于 fabric-sdk-node 的项目开发实战之四

### 使用 fabric-sdk-node－链码调用

#### 事务 invoke-transaction.js

`invoke-transaction.js` 主要完成事务操作，通过调用指定链码实现对账本状态的操作。

创建 `invoke-transaction.js` 文件并编辑：

```js
$ vim invoke-transaction.js 
```

`invoke-transaction.js` 文件完整内容如下：

```js
/**
 * Copyright 2017 IBM All Rights Reserved.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *    http://www.apache.org/licenses/LICENSE-2.0
 *
 *  Unless required by applicable law or agreed to in writing, software
 *  distributed under the License is distributed on an "AS IS" BASIS,
 *  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 *  See the License for the specific language governing permissions and
 *  limitations under the License.
 */
'use strict';
var path = require('path');
var fs = require('fs');
var util = require('util');
var hfc = require('fabric-client');
var helper = require('./helper.js');
var logger = helper.getLogger('invoke-chaincode');

var invokeChaincode = async function(peerNames, channelName, chaincodeName, fcn, args, username, org_name) {
    logger.debug(util.format('\n============ invoke transaction on channel %s ============\n', channelName));
    var error_message = null;
    var tx_id_string = null;
    try {
        // first setup the client for this org
        var client = await helper.getClientForOrg(org_name, username);
        logger.debug('Successfully got the fabric client for the organization "%s"', org_name);
        var channel = client.getChannel(channelName);
        if(!channel) {
            let message = util.format('Channel %s was not defined in the connection profile', channelName);
            logger.error(message);
            throw new Error(message);
        }
        var tx_id = client.newTransactionID();
        // will need the transaction ID string for the event registration later
        tx_id_string = tx_id.getTransactionID();

        // send proposal to endorser
        var request = {
            targets: peerNames,
            chaincodeId: chaincodeName,
            fcn: fcn,
            args: args,
            chainId: channelName,
            txId: tx_id
        };

        let results = await channel.sendTransactionProposal(request);

        // the returned object has both the endorsement results
        // and the actual proposal, the proposal will be needed
        // later when we send a transaction to the orderer
        var proposalResponses = results[0];
        var proposal = results[1];

        // lets have a look at the responses to see if they are
        // all good, if good they will also include signatures
        // required to be committed
        var all_good = true;
        for (var i in proposalResponses) {
            let one_good = false;
            if (proposalResponses && proposalResponses[i].response &&
                proposalResponses[i].response.status === 200) {
                one_good = true;
                logger.info('invoke chaincode proposal was good');
            } else {
                logger.error('invoke chaincode proposal was bad');
            }
            all_good = all_good & one_good;
        }

        if (all_good) {
            logger.info(util.format(
                'Successfully sent Proposal and received ProposalResponse: Status - %s, message - "%s", metadata - "%s", endorsement signature: %s',
                proposalResponses[0].response.status, proposalResponses[0].response.message,
                proposalResponses[0].response.payload, proposalResponses[0].endorsement.signature));

            // wait for the channel-based event hub to tell us
            // that the commit was good or bad on each peer in our organization
            var promises = [];
            let event_hubs = channel.getChannelEventHubsForOrg();
            event_hubs.forEach((eh) => {
                logger.debug('invokeEventPromise - setting up event');
                let invokeEventPromise = new Promise((resolve, reject) => {
                    let event_timeout = setTimeout(() => {
                        let message = 'REQUEST_TIMEOUT:' + eh.getPeerAddr();
                        logger.error(message);
                        eh.disconnect();
                    }, 3000);
                    eh.registerTxEvent(tx_id_string, (tx, code, block_num) => {
                        logger.info('The chaincode invoke chaincode transaction has been committed on peer %s',eh.getPeerAddr());
                        logger.info('Transaction %s has status of %s in blocl %s', tx, code, block_num);
                        clearTimeout(event_timeout);

                        if (code !== 'VALID') {
                            let message = util.format('The invoke chaincode transaction was invalid, code:%s',code);
                            logger.error(message);
                            reject(new Error(message));
                        } else {
                            let message = 'The invoke chaincode transaction was valid.';
                            logger.info(message);
                            resolve(message);
                        }
                    }, (err) => {
                        clearTimeout(event_timeout);
                        logger.error(err);
                        reject(err);
                    },
                        // the default for 'unregister' is true for transaction listeners
                        // so no real need to set here, however for 'disconnect'
                        // the default is false as most event hubs are long running
                        // in this use case we are using it only once
                        {unregister: true, disconnect: true}
                    );
                    eh.connect();
                });
                promises.push(invokeEventPromise);
            });

            var orderer_request = {
                txId: tx_id,
                proposalResponses: proposalResponses,
                proposal: proposal
            };
            var sendPromise = channel.sendTransaction(orderer_request);
            // put the send to the orderer last so that the events get registered and
            // are ready for the orderering and committing
            promises.push(sendPromise);
            let results = await Promise.all(promises);
            logger.debug(util.format('------->>> R E S P O N S E : %j', results));
            let response = results.pop(); //  orderer results are last in the results
            if (response.status === 'SUCCESS') {
                logger.info('Successfully sent transaction to the orderer.');
            } else {
                error_message = util.format('Failed to order the transaction. Error code: %s',response.status);
                logger.debug(error_message);
            }

            // now see what each of the event hubs reported
            for(let i in results) {
                let event_hub_result = results[i];
                let event_hub = event_hubs[i];
                logger.debug('Event results for event hub :%s',event_hub.getPeerAddr());
                if(typeof event_hub_result === 'string') {
                    logger.debug(event_hub_result);
                } else {
                    if(!error_message) error_message = event_hub_result.toString();
                    logger.debug(event_hub_result.toString());
                }
            }
        } else {
            error_message = util.format('Failed to send Proposal and receive all good ProposalResponse');
            logger.debug(error_message);
        }
    } catch (error) {
        logger.error('Failed to invoke due to error: ' + error.stack ? error.stack : error);
        error_message = error.toString();
    }

    if (!error_message) {
        let message = util.format(
            'Successfully invoked the chaincode %s to the channel \'%s\' for transaction ID: %s',
            org_name, channelName, tx_id_string);
        logger.info(message);

        return tx_id_string;
    } else {
        let message = util.format('Failed to invoke chaincode. cause:%s',error_message);
        logger.error(message);
        throw new Error(message);
    }
};

exports.invokeChaincode = invokeChaincode; 
```

#### 查询 query.js

`query.js` 文件中的代码主要完成相关的查询实现，具体功能如下：

*   查询链码信息
*   根据块号查询块信息
*   根据事务 ID 查询事务信息
*   根据 hash 获取区块信息
*   查询已安装、实例化的链码信息
*   查询通道信息等功能

创建 `query.js` 文件并编辑：

```js
$ vim query.js 
```

`query.js` 文件完整内容如下：

```js
/**
 * Copyright 2017 IBM All Rights Reserved.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *    http://www.apache.org/licenses/LICENSE-2.0
 *
 *  Unless required by applicable law or agreed to in writing, software
 *  distributed under the License is distributed on an "AS IS" BASIS,
 *  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 *  See the License for the specific language governing permissions and
 *  limitations under the License.
 */
var path = require('path');
var fs = require('fs');
var util = require('util');
var hfc = require('fabric-client');
var helper = require('./helper.js');
var logger = helper.getLogger('Query');

var queryChaincode = async function(peer, channelName, chaincodeName, args, fcn, username, org_name) {
    try {
        // first setup the client for this org
        var client = await helper.getClientForOrg(org_name, username);
        logger.debug('Successfully got the fabric client for the organization "%s"', org_name);
        var channel = client.getChannel(channelName);
        if(!channel) {
            let message = util.format('Channel %s was not defined in the connection profile', channelName);
            logger.error(message);
            throw new Error(message);
        }

        // send query
        var request = {
            targets : [peer], //queryByChaincode allows for multiple targets
            chaincodeId: chaincodeName,
            fcn: fcn,
            args: args
        };
        let response_payloads = await channel.queryByChaincode(request);
        if (response_payloads) {
            for (let i = 0; i < response_payloads.length; i++) {
                logger.info(args[0]+' now has ' + response_payloads[i].toString('utf8') +
                    ' after the move');
            }
            return args[0]+' now has ' + response_payloads[0].toString('utf8') +
                ' after the move';
        } else {
            logger.error('response_payloads is null');
            return 'response_payloads is null';
        }
    } catch(error) {
        logger.error('Failed to query due to error: ' + error.stack ? error.stack : error);
        return error.toString();
    }
};
var getBlockByNumber = async function(peer, channelName, blockNumber, username, org_name) {
    try {
        // first setup the client for this org
        var client = await helper.getClientForOrg(org_name, username);
        logger.debug('Successfully got the fabric client for the organization "%s"', org_name);
        var channel = client.getChannel(channelName);
        if(!channel) {
            let message = util.format('Channel %s was not defined in the connection profile', channelName);
            logger.error(message);
            throw new Error(message);
        }

        let response_payload = await channel.queryBlock(parseInt(blockNumber, peer));
        if (response_payload) {
            logger.debug(response_payload);
            return response_payload;
        } else {
            logger.error('response_payload is null');
            return 'response_payload is null';
        }
    } catch(error) {
        logger.error('Failed to query due to error: ' + error.stack ? error.stack : error);
        return error.toString();
    }
};
var getTransactionByID = async function(peer, channelName, trxnID, username, org_name) {
    try {
        // first setup the client for this org
        var client = await helper.getClientForOrg(org_name, username);
        logger.debug('Successfully got the fabric client for the organization "%s"', org_name);
        var channel = client.getChannel(channelName);
        if(!channel) {
            let message = util.format('Channel %s was not defined in the connection profile', channelName);
            logger.error(message);
            throw new Error(message);
        }

        let response_payload = await channel.queryTransaction(trxnID, peer);
        if (response_payload) {
            logger.debug(response_payload);
            return response_payload;
        } else {
            logger.error('response_payload is null');
            return 'response_payload is null';
        }
    } catch(error) {
        logger.error('Failed to query due to error: ' + error.stack ? error.stack : error);
        return error.toString();
    }
};
var getBlockByHash = async function(peer, channelName, hash, username, org_name) {
    try {
        // first setup the client for this org
        var client = await helper.getClientForOrg(org_name, username);
        logger.debug('Successfully got the fabric client for the organization "%s"', org_name);
        var channel = client.getChannel(channelName);
        if(!channel) {
            let message = util.format('Channel %s was not defined in the connection profile', channelName);
            logger.error(message);
            throw new Error(message);
        }

        let response_payload = await channel.queryBlockByHash(Buffer.from(hash), peer);
        if (response_payload) {
            logger.debug(response_payload);
            return response_payload;
        } else {
            logger.error('response_payload is null');
            return 'response_payload is null';
        }
    } catch(error) {
        logger.error('Failed to query due to error: ' + error.stack ? error.stack : error);
        return error.toString();
    }
};
var getChainInfo = async function(peer, channelName, username, org_name) {
    try {
        // first setup the client for this org
        var client = await helper.getClientForOrg(org_name, username);
        logger.debug('Successfully got the fabric client for the organization "%s"', org_name);
        var channel = client.getChannel(channelName);
        if(!channel) {
            let message = util.format('Channel %s was not defined in the connection profile', channelName);
            logger.error(message);
            throw new Error(message);
        }

        let response_payload = await channel.queryInfo(peer);
        if (response_payload) {
            logger.debug(response_payload);
            return response_payload;
        } else {
            logger.error('response_payload is null');
            return 'response_payload is null';
        }
    } catch(error) {
        logger.error('Failed to query due to error: ' + error.stack ? error.stack : error);
        return error.toString();
    }
};
//getInstalledChaincodes
var getInstalledChaincodes = async function(peer, channelName, type, username, org_name) {
    try {
        // first setup the client for this org
        var client = await helper.getClientForOrg(org_name, username);
        logger.debug('Successfully got the fabric client for the organization "%s"', org_name);

        let response = null
        if (type === 'installed') {
            response = await client.queryInstalledChaincodes(peer, true); //use the admin identity
        } else {
            var channel = client.getChannel(channelName);
            if(!channel) {
                let message = util.format('Channel %s was not defined in the connection profile', channelName);
                logger.error(message);
                throw new Error(message);
            }
            response = await channel.queryInstantiatedChaincodes(peer, true); //use the admin identity
        }
        if (response) {
            if (type === 'installed') {
                logger.debug('<<< Installed Chaincodes >>>');
            } else {
                logger.debug('<<< Instantiated Chaincodes >>>');
            }
            var details = [];
            for (let i = 0; i < response.chaincodes.length; i++) {
                logger.debug('name: ' + response.chaincodes[i].name + ', version: ' +
                    response.chaincodes[i].version + ', path: ' + response.chaincodes[i].path
                );
                details.push('name: ' + response.chaincodes[i].name + ', version: ' +
                    response.chaincodes[i].version + ', path: ' + response.chaincodes[i].path
                );
            }
            return details;
        } else {
            logger.error('response is null');
            return 'response is null';
        }
    } catch(error) {
        logger.error('Failed to query due to error: ' + error.stack ? error.stack : error);
        return error.toString();
    }
};
var getChannels = async function(peer, username, org_name) {
    try {
        // first setup the client for this org
        var client = await helper.getClientForOrg(org_name, username);
        logger.debug('Successfully got the fabric client for the organization "%s"', org_name);

        let response = await client.queryChannels(peer);
        if (response) {
            logger.debug('<<< channels >>>');
            var channelNames = [];
            for (let i = 0; i < response.channels.length; i++) {
                channelNames.push('channel id: ' + response.channels[i].channel_id);
            }
            logger.debug(channelNames);
            return response;
        } else {
            logger.error('response_payloads is null');
            return 'response_payloads is null';
        }
    } catch(error) {
        logger.error('Failed to query due to error: ' + error.stack ? error.stack : error);
        return error.toString();
    }
};

exports.queryChaincode = queryChaincode;
exports.getBlockByNumber = getBlockByNumber;
exports.getTransactionByID = getTransactionByID;
exports.getBlockByHash = getBlockByHash;
exports.getChainInfo = getChainInfo;
exports.getInstalledChaincodes = getInstalledChaincodes;
exports.getChannels = getChannels; 
```

#### app.js

应用启动主文件，加载相关配置并进行应用初始化，通过编写的相应函数发出调用请求。

返回至项目根目录下

```js
$ cd $HOME/kevin-fabric-sdk-node 
```

创建并编辑应用主文件：

```js
$ vim app.js 
```

`app.js` 文件内容如下：

```js
/**
 * Copyright 2017 IBM All Rights Reserved.
 *
 * Licensed under the Apache License, Version 2.0 (the 'License');
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *    http://www.apache.org/licenses/LICENSE-2.0
 *
 *  Unless required by applicable law or agreed to in writing, software
 *  distributed under the License is distributed on an 'AS IS' BASIS,
 *  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 *  See the License for the specific language governing permissions and
 *  limitations under the License.
 */
'use strict';
var log4js = require('log4js');
var logger = log4js.getLogger('SampleWebApp');
var express = require('express');
var session = require('express-session');
var cookieParser = require('cookie-parser');
var bodyParser = require('body-parser');
var http = require('http');
var util = require('util');
var app = express();
var expressJWT = require('express-jwt');
var jwt = require('jsonwebtoken');
var bearerToken = require('express-bearer-token');
var cors = require('cors');

require('./config.js');
var hfc = require('fabric-client');

var helper = require('./app/helper.js');
var createChannel = require('./app/create-channel.js');
var join = require('./app/join-channel.js');
var install = require('./app/install-chaincode.js');
var instantiate = require('./app/instantiate-chaincode.js');
var invoke = require('./app/invoke-transaction.js');
var query = require('./app/query.js');
var host = process.env.HOST || hfc.getConfigSetting('host');
var port = process.env.PORT || hfc.getConfigSetting('port');
///////////////////////////////////////////////////////////////////////////////
//////////////////////////////// SET CONFIGURATONS ////////////////////////////
///////////////////////////////////////////////////////////////////////////////
app.options('*', cors());
app.use(cors());
//support parsing of application/json type post data
app.use(bodyParser.json());
//support parsing of application/x-www-form-urlencoded post data
app.use(bodyParser.urlencoded({
    extended: false
}));
// set secret variable
app.set('secret', 'thisismysecret');
app.use(expressJWT({
    secret: 'thisismysecret'
}).unless({
    path: ['/users']
}));
app.use(bearerToken());
app.use(function(req, res, next) {
    logger.debug(' ------>>>>>> new request for %s',req.originalUrl);
    if (req.originalUrl.indexOf('/users') >= 0) {
        return next();
    }

    var token = req.token;
    jwt.verify(token, app.get('secret'), function(err, decoded) {
        if (err) {
            res.send({
                success: false,
                message: 'Failed to authenticate token. Make sure to include the ' +
                    'token returned from /users call in the authorization header ' +
                    ' as a Bearer token'
            });
            return;
        } else {
            // add the decoded user name and org name to the request object
            // for the downstream code to use
            req.username = decoded.username;
            req.orgname = decoded.orgName;
            logger.debug(util.format('Decoded from JWT token: username - %s, orgname - %s', decoded.username, decoded.orgName));
            return next();
        }
    });
});

///////////////////////////////////////////////////////////////////////////////
//////////////////////////////// START SERVER /////////////////////////////////
///////////////////////////////////////////////////////////////////////////////
var server = http.createServer(app).listen(port, function() {});
logger.info('****************** SERVER STARTED ************************');
logger.info('***************  http://%s:%s  ******************',host,port);
server.timeout = 240000;

function getErrorMessage(field) {
    var response = {
        success: false,
        message: field + ' field is missing or Invalid in the request'
    };
    return response;
}

///////////////////////////////////////////////////////////////////////////////
///////////////////////// REST ENDPOINTS START HERE ///////////////////////////
///////////////////////////////////////////////////////////////////////////////
// Register and enroll user
app.post('/users', async function(req, res) {
    var username = req.body.username;
    var orgName = req.body.orgName;
    logger.debug('End point : /users');
    logger.debug('User name : ' + username);
    logger.debug('Org name  : ' + orgName);
    if (!username) {
        res.json(getErrorMessage('\'username\''));
        return;
    }
    if (!orgName) {
        res.json(getErrorMessage('\'orgName\''));
        return;
    }
    var token = jwt.sign({
        exp: Math.floor(Date.now() / 1000) + parseInt(hfc.getConfigSetting('jwt_expiretime')),
        username: username,
        orgName: orgName
    }, app.get('secret'));
    let response = await helper.getRegisteredUser(username, orgName, true);
    logger.debug('-- returned from registering the username %s for organization %s',username,orgName);
    if (response && typeof response !== 'string') {
        logger.debug('Successfully registered the username %s for organization %s',username,orgName);
        response.token = token;
        res.json(response);
    } else {
        logger.debug('Failed to register the username %s for organization %s with::%s',username,orgName,response);
        res.json({success: false, message: response});
    }

});
// Create Channel
app.post('/channels', async function(req, res) {
    logger.info('<<<<<<<<<<<<<<<<< C R E A T E  C H A N N E L >>>>>>>>>>>>>>>>>');
    logger.debug('End point : /channels');
    var channelName = req.body.channelName;
    var channelConfigPath = req.body.channelConfigPath;
    logger.debug('Channel name : ' + channelName);
    logger.debug('channelConfigPath : ' + channelConfigPath); //../artifacts/channel/mychannel.tx
    if (!channelName) {
        res.json(getErrorMessage('\'channelName\''));
        return;
    }
    if (!channelConfigPath) {
        res.json(getErrorMessage('\'channelConfigPath\''));
        return;
    }

    let message = await createChannel.createChannel(channelName, channelConfigPath, req.username, req.orgname);
    res.send(message);
});
// Join Channel
app.post('/channels/:channelName/peers', async function(req, res) {
    logger.info('<<<<<<<<<<<<<<<<< J O I N  C H A N N E L >>>>>>>>>>>>>>>>>');
    var channelName = req.params.channelName;
    var peers = req.body.peers;
    logger.debug('channelName : ' + channelName);
    logger.debug('peers : ' + peers);
    logger.debug('username :' + req.username);
    logger.debug('orgname:' + req.orgname);

    if (!channelName) {
        res.json(getErrorMessage('\'channelName\''));
        return;
    }
    if (!peers || peers.length == 0) {
        res.json(getErrorMessage('\'peers\''));
        return;
    }

    let message =  await join.joinChannel(channelName, peers, req.username, req.orgname);
    res.send(message);
});
// Install chaincode on target peers
app.post('/chaincodes', async function(req, res) {
    logger.debug('==================== INSTALL CHAINCODE ==================');
    var peers = req.body.peers;
    var chaincodeName = req.body.chaincodeName;
    var chaincodePath = req.body.chaincodePath;
    var chaincodeVersion = req.body.chaincodeVersion;
    var chaincodeType = req.body.chaincodeType;
    logger.debug('peers : ' + peers); // target peers list
    logger.debug('chaincodeName : ' + chaincodeName);
    logger.debug('chaincodePath  : ' + chaincodePath);
    logger.debug('chaincodeVersion  : ' + chaincodeVersion);
    logger.debug('chaincodeType  : ' + chaincodeType);
    if (!peers || peers.length == 0) {
        res.json(getErrorMessage('\'peers\''));
        return;
    }
    if (!chaincodeName) {
        res.json(getErrorMessage('\'chaincodeName\''));
        return;
    }
    if (!chaincodePath) {
        res.json(getErrorMessage('\'chaincodePath\''));
        return;
    }
    if (!chaincodeVersion) {
        res.json(getErrorMessage('\'chaincodeVersion\''));
        return;
    }
    if (!chaincodeType) {
        res.json(getErrorMessage('\'chaincodeType\''));
        return;
    }
    let message = await install.installChaincode(peers, chaincodeName, chaincodePath, chaincodeVersion, chaincodeType, req.username, req.orgname)
    res.send(message);});
// Instantiate chaincode on target peers
app.post('/channels/:channelName/chaincodes', async function(req, res) {
    logger.debug('==================== INSTANTIATE CHAINCODE ==================');
    var peers = req.body.peers;
    var chaincodeName = req.body.chaincodeName;
    var chaincodeVersion = req.body.chaincodeVersion;
    var channelName = req.params.channelName;
    var chaincodeType = req.body.chaincodeType;
    var fcn = req.body.fcn;
    var args = req.body.args;
    logger.debug('peers  : ' + peers);
    logger.debug('channelName  : ' + channelName);
    logger.debug('chaincodeName : ' + chaincodeName);
    logger.debug('chaincodeVersion  : ' + chaincodeVersion);
    logger.debug('chaincodeType  : ' + chaincodeType);
    logger.debug('fcn  : ' + fcn);
    logger.debug('args  : ' + args);
    if (!chaincodeName) {
        res.json(getErrorMessage('\'chaincodeName\''));
        return;
    }
    if (!chaincodeVersion) {
        res.json(getErrorMessage('\'chaincodeVersion\''));
        return;
    }
    if (!channelName) {
        res.json(getErrorMessage('\'channelName\''));
        return;
    }
    if (!chaincodeType) {
        res.json(getErrorMessage('\'chaincodeType\''));
        return;
    }
    if (!args) {
        res.json(getErrorMessage('\'args\''));
        return;
    }

    let message = await instantiate.instantiateChaincode(peers, channelName, chaincodeName, chaincodeVersion, chaincodeType, fcn, args, req.username, req.orgname);
    res.send(message);
});
// Invoke transaction on chaincode on target peers
app.post('/channels/:channelName/chaincodes/:chaincodeName', async function(req, res) {
    logger.debug('==================== INVOKE ON CHAINCODE ==================');
    var peers = req.body.peers;
    var chaincodeName = req.params.chaincodeName;
    var channelName = req.params.channelName;
    var fcn = req.body.fcn;
    var args = req.body.args;
    logger.debug('channelName  : ' + channelName);
    logger.debug('chaincodeName : ' + chaincodeName);
    logger.debug('fcn  : ' + fcn);
    logger.debug('args  : ' + args);
    if (!chaincodeName) {
        res.json(getErrorMessage('\'chaincodeName\''));
        return;
    }
    if (!channelName) {
        res.json(getErrorMessage('\'channelName\''));
        return;
    }
    if (!fcn) {
        res.json(getErrorMessage('\'fcn\''));
        return;
    }
    if (!args) {
        res.json(getErrorMessage('\'args\''));
        return;
    }

    let message = await invoke.invokeChaincode(peers, channelName, chaincodeName, fcn, args, req.username, req.orgname);
    res.send(message);
});
// Query on chaincode on target peers
app.get('/channels/:channelName/chaincodes/:chaincodeName', async function(req, res) {
    logger.debug('==================== QUERY BY CHAINCODE ==================');
    var channelName = req.params.channelName;
    var chaincodeName = req.params.chaincodeName;
    let args = req.query.args;
    let fcn = req.query.fcn;
    let peer = req.query.peer;

    logger.debug('channelName : ' + channelName);
    logger.debug('chaincodeName : ' + chaincodeName);
    logger.debug('fcn : ' + fcn);
    logger.debug('args : ' + args);

    if (!chaincodeName) {
        res.json(getErrorMessage('\'chaincodeName\''));
        return;
    }
    if (!channelName) {
        res.json(getErrorMessage('\'channelName\''));
        return;
    }
    if (!fcn) {
        res.json(getErrorMessage('\'fcn\''));
        return;
    }
    if (!args) {
        res.json(getErrorMessage('\'args\''));
        return;
    }
    args = args.replace(/'/g, '"');
    args = JSON.parse(args);
    logger.debug(args);

    let message = await query.queryChaincode(peer, channelName, chaincodeName, args, fcn, req.username, req.orgname);
    res.send(message);
});
//  Query Get Block by BlockNumber
app.get('/channels/:channelName/blocks/:blockId', async function(req, res) {
    logger.debug('==================== GET BLOCK BY NUMBER ==================');
    let blockId = req.params.blockId;
    let peer = req.query.peer;
    logger.debug('channelName : ' + req.params.channelName);
    logger.debug('BlockID : ' + blockId);
    logger.debug('Peer : ' + peer);
    if (!blockId) {
        res.json(getErrorMessage('\'blockId\''));
        return;
    }

    let message = await query.getBlockByNumber(peer, req.params.channelName, blockId, req.username, req.orgname);
    res.send(message);
});
// Query Get Transaction by Transaction ID
app.get('/channels/:channelName/transactions/:trxnId', async function(req, res) {
    logger.debug('================ GET TRANSACTION BY TRANSACTION_ID ======================');
    logger.debug('channelName : ' + req.params.channelName);
    let trxnId = req.params.trxnId;
    let peer = req.query.peer;
    if (!trxnId) {
        res.json(getErrorMessage('\'trxnId\''));
        return;
    }

    let message = await query.getTransactionByID(peer, req.params.channelName, trxnId, req.username, req.orgname);
    res.send(message);
});
// Query Get Block by Hash
app.get('/channels/:channelName/blocks', async function(req, res) {
    logger.debug('================ GET BLOCK BY HASH ======================');
    logger.debug('channelName : ' + req.params.channelName);
    let hash = req.query.hash;
    let peer = req.query.peer;
    if (!hash) {
        res.json(getErrorMessage('\'hash\''));
        return;
    }

    let message = await query.getBlockByHash(peer, req.params.channelName, hash, req.username, req.orgname);
    res.send(message);
});
//Query for Channel Information
app.get('/channels/:channelName', async function(req, res) {
    logger.debug('================ GET CHANNEL INFORMATION ======================');
    logger.debug('channelName : ' + req.params.channelName);
    let peer = req.query.peer;

    let message = await query.getChainInfo(peer, req.params.channelName, req.username, req.orgname);
    res.send(message);
});
//Query for Channel instantiated chaincodes
app.get('/channels/:channelName/chaincodes', async function(req, res) {
    logger.debug('================ GET INSTANTIATED CHAINCODES ======================');
    logger.debug('channelName : ' + req.params.channelName);
    let peer = req.query.peer;

    let message = await query.getInstalledChaincodes(peer, req.params.channelName, 'instantiated', req.username, req.orgname);
    res.send(message);
});
// Query to fetch all Installed/instantiated chaincodes
app.get('/chaincodes', async function(req, res) {
    var peer = req.query.peer;
    var installType = req.query.type;
    logger.debug('================ GET INSTALLED CHAINCODES ======================');

    let message = await query.getInstalledChaincodes(peer, null, 'installed', req.username, req.orgname)
    res.send(message);
});
// Query to fetch channels
app.get('/channels', async function(req, res) {
    logger.debug('================ GET CHANNELS ======================');
    logger.debug('peer: ' + req.query.peer);
    var peer = req.query.peer;
    if (!peer) {
        res.json(getErrorMessage('\'peer\''));
        return;
    }

    let message = await query.getChannels(peer, req.username, req.orgname);
    res.send(message);
}); 
```

### 参考资料

*   [Hyperledger Fabric-SDK-Node](https://github.com/hyperledger/fabric-sdk-node)
*   [Node SDK documentation](https://fabric-sdk-node.github.io/)
*   [fabric-samples balance-transfer](https://github.com/hyperledger/fabric-samples/tree/release/balance-transfer)