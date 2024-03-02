# 第四章 应用 sdk-node－链码安装及实例化

## 从零到壹构建基于 fabric-sdk-node 的项目开发实战之四

### 使用 fabric-sdk-node

在项目根目录下创建一个 `app` 的文件夹，做为所有的 JS 代码文件的存放目录，具体文件如下:

*   helper.js
*   create-channel.js
*   join-channel.js
*   install-chaincode.js
*   instantiate-chaincode.js
*   invoke-transaction.js
*   query.js

创建 `app` 目录并进入该目录：

```js
$ cd $HOME/kevin-fabric-sdk-node/
$ mkdir app && cd app 
```

#### helper.js

`helper.js` 是应用最重要的主文件，提供了核心对象的实现。

创建 `helper.js` 文件并编辑：

```js
$ vim helper.js 
```

`helper.js` 文件完整内容如下：

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
var logger = log4js.getLogger('Helper');
logger.setLevel('DEBUG');

var path = require('path');
var util = require('util');
var copService = require('fabric-ca-client');

var hfc = require('fabric-client');
hfc.setLogger(logger);
var ORGS = hfc.getConfigSetting('network-config');

var clients = {};
var channels = {};
var caClients = {};

var sleep = async function (sleep_time_ms) {
    return new Promise(resolve => setTimeout(resolve, sleep_time_ms));
}

async function getClientForOrg (userorg, username) {
    logger.debug('getClientForOrg - ****** START %s %s', userorg, username)
    // get a fabric client loaded with a connection profile for this org
    let config = '-connection-profile-path';

    // build a client context and load it with a connection profile
    // lets only load the network settings and save the client for later
    let client = hfc.loadFromConfig(hfc.getConfigSetting('network'+config));

    // This will load a connection profile over the top of the current one one
    // since the first one did not have a client section and the following one does
    // nothing will actually be replaced.
    // This will also set an admin identity because the organization defined in the
    // client section has one defined
    client.loadFromConfig(hfc.getConfigSetting(userorg+config));

    // this will create both the state store and the crypto store based
    // on the settings in the client section of the connection profile
    await client.initCredentialStores();

    // The getUserContext call tries to get the user from persistence.
    // If the user has been saved to persistence then that means the user has
    // been registered and enrolled. If the user is found in persistence
    // the call will then assign the user to the client object.
    if(username) {
        let user = await client.getUserContext(username, true);
        if(!user) {
            throw new Error(util.format('User was not found :', username));
        } else {
            logger.debug('User %s was found to be registered and enrolled', username);
        }
    }
    logger.debug('getClientForOrg - ****** END %s %s \n\n', userorg, username)

    return client;
}

var getRegisteredUser = async function(username, userOrg, isJson) {
    try {
        var client = await getClientForOrg(userOrg);
        logger.debug('Successfully initialized the credential stores');
            // client can now act as an agent for organization Org1
            // first check to see if the user is already enrolled
        var user = await client.getUserContext(username, true);
        if (user && user.isEnrolled()) {
            logger.info('Successfully loaded member from persistence');
        } else {
            // user was not enrolled, so we will need an admin user object to register
            logger.info('User %s was not enrolled, so we will need an admin user object to register',username);
            var admins = hfc.getConfigSetting('admins');
            let adminUserObj = await client.setUserContext({username: admins[0].username, password: admins[0].secret});
            let caClient = client.getCertificateAuthority();
            let secret = await caClient.register({
                enrollmentID: username,
                affiliation: userOrg.toLowerCase() + '.department1'
            }, adminUserObj);
            logger.debug('Successfully got the secret for user %s',username);
            user = await client.setUserContext({username:username, password:secret});
            logger.debug('Successfully enrolled username %s  and setUserContext on the client object', username);
        }
        if(user && user.isEnrolled) {
            if (isJson && isJson === true) {
                var response = {
                    success: true,
                    secret: user._enrollmentSecret,
                    message: username + ' enrolled Successfully',
                };
                return response;
            }
        } else {
            throw new Error('User was not enrolled ');
        }
    } catch(error) {
        logger.error('Failed to get registered user: %s with error: %s', username, error.toString());
        return 'failed '+error.toString();
    }

};

var setupChaincodeDeploy = function() {
    process.env.GOPATH = path.join(__dirname, hfc.getConfigSetting('CC_SRC_PATH'));
};

var getLogger = function(moduleName) {
    var logger = log4js.getLogger(moduleName);
    logger.setLevel('DEBUG');
    return logger;
};

exports.getClientForOrg = getClientForOrg;
exports.getLogger = getLogger;
exports.setupChaincodeDeploy = setupChaincodeDeploy;
exports.getRegisteredUser = getRegisteredUser; 
```

#### 创建通道 create-channel.js

`create-channel.js` 文件主要实现根据指定的通道交易配置文件创建指定的应用通道。

创建 `create-channel.js` 文件并编辑：

```js
$ vim create-channel.js 
```

`create-channel.js` 文件完整内容如下：

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
var util = require('util');
var fs = require('fs');
var path = require('path');

var helper = require('./helper.js');
var logger = helper.getLogger('Create-Channel');
//Attempt to send a request to the orderer with the sendTransaction method
var createChannel = async function(channelName, channelConfigPath, username, orgName) {
    logger.debug('\n====== Creating Channel \'' + channelName + '\' ======\n');
    try {
        // first setup the client for this org
        var client = await helper.getClientForOrg(orgName);
        logger.debug('Successfully got the fabric client for the organization "%s"', orgName);

        // read in the envelope for the channel config raw bytes
        var envelope = fs.readFileSync(path.join(__dirname, channelConfigPath));
        // extract the channel config bytes from the envelope to be signed
        var channelConfig = client.extractChannelConfig(envelope);

        //Acting as a client in the given organization provided with "orgName" param
        // sign the channel config bytes as "endorsement", this is required by
        // the orderer's channel creation policy
        // this will use the admin identity assigned to the client when the connection profile was loaded
        let signature = client.signChannelConfig(channelConfig);

        let request = {
            config: channelConfig,
            signatures: [signature],
            name: channelName,
            txId: client.newTransactionID(true) // get an admin based transactionID
        };

        // send to orderer
        var response = await client.createChannel(request)
        logger.debug(' response ::%j', response);
        if (response && response.status === 'SUCCESS') {
            logger.debug('Successfully created the channel.');
            let response = {
                success: true,
                message: 'Channel \'' + channelName + '\' created Successfully'
            };
            return response;
        } else {
            logger.error('\n!!!!!!!!! Failed to create the channel \'' + channelName +
                '\' !!!!!!!!!\n\n');
            throw new Error('Failed to create the channel \'' + channelName + '\'');
        }
    } catch (err) {
        logger.error('Failed to initialize the channel: ' + err.stack ? err.stack :    err);
        throw new Error('Failed to initialize the channel: ' + err.toString());
    }
};

exports.createChannel = createChannel; 
```

#### 加入通道 join-channel.js

`join-channel.js` 文件主要实现将指定的 peers 加入至指定的通道中。

创建 `join-channel.js` 文件并编辑：

```js
$ vim join-channel.js 
```

`join-channel.js` 文件完整内容如下：

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
var util = require('util');
var path = require('path');
var fs = require('fs');

var helper = require('./helper.js');
var logger = helper.getLogger('Join-Channel');

/*
 * Have an organization join a channel
 */
var joinChannel = async function(channel_name, peers, username, org_name) {
    logger.debug('\n\n============ Join Channel start ============\n')
    var error_message = null;
    var all_eventhubs = [];
    try {
        logger.info('Calling peers in organization "%s" to join the channel', org_name);

        // first setup the client for this org
        var client = await helper.getClientForOrg(org_name, username);
        logger.debug('Successfully got the fabric client for the organization "%s"', org_name);
        var channel = client.getChannel(channel_name);
        if(!channel) {
            let message = util.format('Channel %s was not defined in the connection profile', channel_name);
            logger.error(message);
            throw new Error(message);
        }

        // next step is to get the genesis_block from the orderer,
        // the starting point for the channel that we want to join
        let request = {
            txId :     client.newTransactionID(true) //get an admin based transactionID
        };
        let genesis_block = await channel.getGenesisBlock(request);

        // tell each peer to join and wait 10 seconds
        // for the channel to be created on each peer
        var promises = [];
        promises.push(new Promise(resolve => setTimeout(resolve, 10000)));

        let join_request = {
            targets: peers, //using the peer names which only is allowed when a connection profile is loaded
            txId: client.newTransactionID(true), //get an admin based transactionID
            block: genesis_block
        };
        let join_promise = channel.joinChannel(join_request);
        promises.push(join_promise);
        let results = await Promise.all(promises);
        logger.debug(util.format('Join Channel R E S P O N S E : %j', results));

        // lets check the results of sending to the peers which is
        // last in the results array
        let peers_results = results.pop();
        // then each peer results
        for(let i in peers_results) {
            let peer_result = peers_results[i];
            if(peer_result.response && peer_result.response.status == 200) {
                logger.info('Successfully joined peer to the channel %s',channel_name);
            } else {
                let message = util.format('Failed to joined peer to the channel %s',channel_name);
                error_message = message;
                logger.error(message);
            }
        }
    } catch(error) {
        logger.error('Failed to join channel due to error: ' + error.stack ? error.stack : error);
        error_message = error.toString();
    }

    // need to shutdown open event streams
    all_eventhubs.forEach((eh) => {
        eh.disconnect();
    });

    if (!error_message) {
        let message = util.format(
            'Successfully joined peers in organization %s to the channel:%s',
            org_name, channel_name);
        logger.info(message);
        // build a response to send back to the REST caller
        let response = {
            success: true,
            message: message
        };
        return response;
    } else {
        let message = util.format('Failed to join all peers to channel. cause:%s',error_message);
        logger.error(message);
        throw new Error(message);
    }
};
exports.joinChannel = joinChannel; 
```

#### 链码安装 install-chaincode.js

`install-chaincode.js` 文件主要实现对链码的安装。

创建 `install-chaincode.js` 文件并编辑：

```js
$ vim install-chaincode.js 
```

`install-chaincode.js` 文件完整内容如下：

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
var config = require('../config.json');
var helper = require('./helper.js');
var logger = helper.getLogger('install-chaincode');
var tx_id = null;

var installChaincode = async function(peers, chaincodeName, chaincodePath,
    chaincodeVersion, chaincodeType, username, org_name) {
    logger.debug('\n\n============ Install chaincode on organizations ============\n');
    helper.setupChaincodeDeploy();
    let error_message = null;
    try {
        logger.info('Calling peers in organization "%s" to join the channel', org_name);

        // first setup the client for this org
        var client = await helper.getClientForOrg(org_name, username);
        logger.debug('Successfully got the fabric client for the organization "%s"', org_name);

        tx_id = client.newTransactionID(true); //get an admin transactionID
        var request = {
            targets: peers,
            chaincodePath: chaincodePath,
            chaincodeId: chaincodeName,
            chaincodeVersion: chaincodeVersion,
            chaincodeType: chaincodeType
        };
        let results = await client.installChaincode(request);
        // the returned object has both the endorsement results
        // and the actual proposal, the proposal will be needed
        // later when we send a transaction to the orederer
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
                logger.info('install proposal was good');
            } else {
                logger.error('install proposal was bad %j',proposalResponses.toJSON());
            }
            all_good = all_good & one_good;
        }
        if (all_good) {
            logger.info('Successfully sent install Proposal and received ProposalResponse');
        } else {
            error_message = 'Failed to send install Proposal or receive valid response. Response null or status is not 200'
            logger.error(error_message);
        }
    } catch(error) {
        logger.error('Failed to install due to error: ' + error.stack ? error.stack : error);
        error_message = error.toString();
    }

    if (!error_message) {
        let message = util.format('Successfully install chaincode');
        logger.info(message);
        // build a response to send back to the REST caller
        let response = {
            success: true,
            message: message
        };
        return response;
    } else {
        let message = util.format('Failed to install due to:%s',error_message);
        logger.error(message);
        throw new Error(message);
    }
};
exports.installChaincode = installChaincode; 
```

#### 链码实例化 instantiate-chaincode.js

`instantiate-chaincode.js` 文件主要完成对链码的实例化。

创建 `instantiate-chaincode.js` 文件并编辑：

```js
$ vim instantiate-chaincode.js 
```

`instantiate-chaincode.js` 文件完整内容如下：

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
var logger = helper.getLogger('instantiate-chaincode');

var instantiateChaincode = async function(peers, channelName, chaincodeName, chaincodeVersion, functionName, chaincodeType, args, username, org_name) {
    logger.debug('\n\n============ Instantiate chaincode on channel ' + channelName +
        ' ============\n');
    var error_message = null;

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
        var tx_id = client.newTransactionID(true); // Get an admin based transactionID
                                               // An admin based transactionID will
                                               // indicate that admin identity should
                                               // be used to sign the proposal request.
        // will need the transaction ID string for the event registration later
        var deployId = tx_id.getTransactionID();

        // send proposal to endorser
        var request = {
            targets : peers,
            chaincodeId: chaincodeName,
            chaincodeType: chaincodeType,
            chaincodeVersion: chaincodeVersion,
            args: args,
            txId: tx_id
        };

        if (functionName)
            request.fcn = functionName;

        let results = await channel.sendInstantiateProposal(request, 60000); //instantiate takes much longer

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
                logger.info('instantiate proposal was good');
            } else {
                logger.error('instantiate proposal was bad');
            }
            all_good = all_good & one_good;
        }

        if (all_good) {
            logger.info(util.format(
                'Successfully sent Proposal and received ProposalResponse: Status - %s, message - "%s", metadata - "%s", endorsement signature: %s',
                proposalResponses[0].response.status, proposalResponses[0].response.message,
                proposalResponses[0].response.payload, proposalResponses[0].endorsement.signature));

            // wait for the channel-based event hub to tell us that the
            // instantiate transaction was committed on the peer
            var promises = [];
            let event_hubs = channel.getChannelEventHubsForOrg();
            logger.debug('found %s eventhubs for this organization %s',event_hubs.length, org_name);
            event_hubs.forEach((eh) => {
                let instantiateEventPromise = new Promise((resolve, reject) => {
                    logger.debug('instantiateEventPromise - setting up event');
                    let event_timeout = setTimeout(() => {
                        let message = 'REQUEST_TIMEOUT:' + eh.getPeerAddr();
                        logger.error(message);
                        eh.disconnect();
                    }, 60000);
                    eh.registerTxEvent(deployId, (tx, code, block_num) => {
                        logger.info('The chaincode instantiate transaction has been committed on peer %s',eh.getPeerAddr());
                        logger.info('Transaction %s has status of %s in blocl %s', tx, code, block_num);
                        clearTimeout(event_timeout);

                        if (code !== 'VALID') {
                            let message = until.format('The chaincode instantiate transaction was invalid, code:%s',code);
                            logger.error(message);
                            reject(new Error(message));
                        } else {
                            let message = 'The chaincode instantiate transaction was valid.';
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
                promises.push(instantiateEventPromise);
            });

            var orderer_request = {
                txId: tx_id, // must include the transaction id so that the outbound
                             // transaction to the orderer will be signed by the admin
                             // id as was the proposal above, notice that transactionID
                             // generated above was based on the admin id not the current
                             // user assigned to the 'client' instance.
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
        logger.error('Failed to send instantiate due to error: ' + error.stack ? error.stack : error);
        error_message = error.toString();
    }

    if (!error_message) {
        let message = util.format(
            'Successfully instantiate chaingcode in organization %s to the channel \'%s\'',
            org_name, channelName);
        logger.info(message);
        // build a response to send back to the REST caller
        let response = {
            success: true,
            message: message
        };
        return response;
    } else {
        let message = util.format('Failed to instantiate. cause:%s',error_message);
        logger.error(message);
        throw new Error(message);
    }
};
exports.instantiateChaincode = instantiateChaincode; 
```

### 参考资料

*   [Hyperledger Fabric-SDK-Node](https://github.com/hyperledger/fabric-sdk-node)
*   [Node SDK documentation](https://fabric-sdk-node.github.io/)
*   [fabric-samples balance-transfer](https://github.com/hyperledger/fabric-samples/tree/release/balance-transfer)