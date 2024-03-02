# 三、.4 sdk-node 之实现事务与查询

## 从零到壹实现 Marbles 资产管理系统 （Fabric-SDK-Node）之－实现交易与查询

在 utils/fc_wrangler/parts 目录下创建 high_availability.js、invoke_cc.js、query_cc.js、query_peer.js 文件并编辑

*   **high_availability.js：** 网络组件信息脚本
*   **invoke_cc.js：** 事务调用脚本
*   **query_cc.js：** 查询调用脚本
*   **query_peer.js：** 与账本相关信息的脚本

### 添加链码查询及事务执行的脚本

进入 utils/fc_wrangler/parts 目录

```go
$ cd $HOME/kevin-marbles/utils/fc_wrangler/parts 
```

#### 创建 high_availability.js

```go
$ vim high_availability.js 
```

high_availability.js 文件内容如下

```go
// ------------------------------------------------------------------------
// HA Functions (HA = High Availability) aka try another peer/CAs
// ------------------------------------------------------------------------
// - Remember the last peer we used that worked. Keep using this peer as long as it works.
// - If the peer has crashed, switch to the next peer.
// - Keep trying peers until we have looped through all the peers.

module.exports = function (logger) {
    var Peer = require('fabric-client/lib/Peer.js');
    var ha = {};
    ha.success_peer_position = 0;//the last peer position that was successful
    ha.using_peer_position = 0;    //the peer array position to use for the next request
    ha.success_ca_position = 0;    //the last ca position that was successful
    ha.using_ca_position = 0;    //the ca array position to use for the next enrollment

    // ------------------------------------------------------------------------
    // Change what peer the SDK is using
    // ------------------------------------------------------------------------
    ha.use_peer = function (obj, options) {
        try {
            logger.debug('Adding peer to sdk client', options.peer_url);
            obj.channel.addPeer(new Peer(options.peer_url, options.peer_tls_opts));
        }
        catch (e) {
            //might error if peer already exists, but we don't care
        }
    };

    // ------------------------------------------------------------------------
    // Switch to Another Peer - returns null if there IS another peer to switch to
    // ------------------------------------------------------------------------
    ha.switch_peer = function (obj, options) {
        if (!options || !options.peer_urls || !options.peer_tls_opts) {
            logger.error('Missing options for switch_peer()');
            return { error: 'Missing options for switch_peer()' };
        }
        let next_peer_position = ha.using_peer_position + 1;
        if (next_peer_position >= options.peer_urls.length) {                        //wrap around
            next_peer_position = 0;
        }

        // --- Tried All Peers --- //
        if (next_peer_position === ha.success_peer_position) {                        //we've tried all peers, error out
            logger.error('Exhausted all peers. There are no more peers to try.');
            return { error: 'Exhausted all peers.' };

        } else {

            try {                                                                    //remove current peer
                logger.warn('Switching peers!', ha.using_peer_position, next_peer_position);
                logger.debug('Removing peer from sdk client', options.peer_urls[ha.using_peer_position]);
                obj.channel.removePeer(new Peer(options.peer_urls[ha.using_peer_position], options.peer_tls_opts));
            } catch (e) {
                logger.error('Could not remove peer from sdk client', e);
            }

            // --- Use Next Peer --- //
            ha.using_peer_position = next_peer_position;
            const temp = {
                peer_url: options.peer_urls[ha.using_peer_position],
                peer_tls_opts: options.peer_tls_opts
            };
            ha.use_peer(obj, temp);
            return null;
        }
    };

    // ------------------------------------------------------------------------
    // Get the Event URl to use - returns null if there are NO urls
    // ------------------------------------------------------------------------
    ha.get_event_url = function (options) {
        let ret = null;
        if (options && options.event_urls && options.event_urls[ha.using_peer_position]) {
            ret = options.event_urls[ha.using_peer_position];
        }
        logger.debug('[fcw] setting target event url', ret);
        return ret;
    };

    // ------------------------------------------------------------------------
    // Get the Next Certificate Authority - returns options to use for enrollment if there IS another CA to switch to
    // ------------------------------------------------------------------------
    ha.get_next_ca = function (options) {
        if (!options || !options.ca_urls || !options.ca_tls_opts) {
            logger.error('Missing options for get_next_ca()');
            return null;
        }

        ha.using_ca_position++;
        if (ha.using_ca_position >= options.ca_urls.length) {                //wrap around
            ha.using_ca_position = 0;
        }

        if (ha.using_ca_position === ha.success_ca_position) {                //we've tried all ca, error out
            logger.error('Exhausted all CAs. There are no more CAs to try.');
            return null;
        } else {
            return ha.get_ca(options);
        }
    };

    // ------------------------------------------------------------------------
    // Get the Current Certificate Authority - returns options for enrollment
    // ------------------------------------------------------------------------
    ha.get_ca = function (options) {
        if (!options || !options.ca_urls || !options.ca_tls_opts) {
            logger.error('Missing options for get_ca()');
            return null;
        }

        options.ca_url = options.ca_urls[ha.using_ca_position];    //use this CA
        //options.ca_tls_opts = options.ca_tls_opts; //dsh todo get the array, return the right one
        //options.ca_name = options.ca_name; //dsh todo get the array, return the right one
        return options;
    };

    return ha;
}; 
```

#### 创建 invoke_cc.js

```go
$ vim invoke_cc.js 
```

invoke_cc.js 文件内容如下

```go
//-------------------------------------------------------------------
// Invoke Chaincode
//-------------------------------------------------------------------
var path = require('path');

module.exports = function (g_options, logger) {
    var common = require(path.join(__dirname, './common.js'))(logger);
    var invoke_cc = {};

    if (!g_options) g_options = {};
    if (!g_options.block_delay) g_options.block_delay = 10000;

    //-------------------------------------------------------------------
    // Invoke Chaincode - aka write to the ledger
    //-------------------------------------------------------------------
    invoke_cc.invoke_chaincode = function (obj, options, cb) {
        logger.debug('[fcw] Invoking Chaincode: ' + options.cc_function + '()');
        var eventHub;
        var channel = obj.channel;
        var client = obj.client;
        var cbCalled = false;
        var startTime = Date.now();
        var watchdog = null;

        // send proposal to endorser
        var request = {
            chaincodeId: options.chaincode_id,
            fcn: options.cc_function,
            args: options.cc_args,
            txId: client.newTransactionID(),
        };
        logger.debug('[fcw] Sending invoke req', request);

        // ---------------- Setup EventHub ---------------- //
        setup_event_hub(options);

        // ---------------- Send Proposal ---------------- //
        channel.sendTransactionProposal(request).then(function (results) {
            var request = common.check_proposal_res(results, options.endorsed_hook);//proposal was endorsed
            return channel.sendTransaction(request);
        }).then(function (response) {
            if (response.status === 'SUCCESS') {                                    //tx was ordered
                logger.debug('[fcw] Successfully ordered endorsement transaction.');

                // Call optional order hook
                if (options.ordered_hook) options.ordered_hook(null, request.txId.toString());

                // ------- [A] Use Event for Tx Confirmation ------- // option A
                if (options.target_event_url) {

                    // Watchdog for no block event
                    watchdog = setTimeout(() => {
                        logger.error('[fcw] Failed to receive block event within the timeout period');
                        eventHub.disconnect();

                        if (cb && !cbCalled) {
                            cbCalled = true;
                            return cb(null);    //timeout pass it back
                        } else return;
                    }, g_options.block_delay + 5000);

                    // ------- [B] Wait xxxx ms for Block  ------- // option B
                } else {
                    setTimeout(function () {
                        if (cb) return cb(null);
                        else return;
                    }, g_options.block_delay + 5000);
                }
            }

            // ordering failed, No good
            else {
                if (options.ordered_hook) options.ordered_hook('failed');
                logger.error('[fcw] Failed to order the transaction. Error code: ', response);
                throw response;
            }
        }).catch(function (err) {
            logger.error('[fcw] Error in invoke catch block', typeof err, err);
            if (options.target_event_url) {    //if using eventHub, disconnect
                eventHub.disconnect();
            }

            var formatted = common.format_error_msg(err);
            if (options.ordered_hook) options.ordered_hook('failed', formatted);

            if (cb) return cb(formatted, null);
            else return;
        });

        //-------------------------------------------------------------------
        // Use an event to be notified when the transaction is committed
        //-------------------------------------------------------------------
        function setup_event_hub(options) {
            if (options.event_urls !== null) {                                //iff this is null we are not going to use eventHub
                if (!options.target_event_url && options.event_urls.length >= 1) {
                    options.target_event_url = options.event_urls[0];        //if target event url not set but array is, pick the first one
                }
            } else {
                options.target_event_url = null;                             //don't use eventHub
            }
            if (options.target_event_url) {
                logger.debug('[fcw] listening to transaction event. url:', options.target_event_url);
                eventHub = client.newEventHub();
                eventHub.setPeerAddr(options.target_event_url, options.peer_tls_opts);
                eventHub.connect();

                // Wait for tx committed event - this will happen async
                eventHub.registerTxEvent(request.txId.getTransactionID(), (tx, code) => {
                    var elapsed = Date.now() - startTime + 'ms';
                    logger.info('[fcw] The chaincode transaction event has happened! success?:', code, elapsed);
                    if (watchdog) clearTimeout(watchdog);                    //stop watchdog, event happened
                    eventHub.disconnect();

                    if (code !== 'VALID') {
                        if (cb && !cbCalled) {
                            cbCalled = true;
                            return cb(common.format_error_msg('Commit code: ' + code));    //pass error back
                        } else return;
                    } else {
                        if (cb && !cbCalled) {
                            cbCalled = true;
                            return cb(null);                                            //all good, pass it back
                        } else return;
                    }
                }, function (disconnectMsg) {                                            //callback whenever eventHub is disconnected, normal
                    logger.debug('[fcw] transaction event is disconnected');
                });
            } else {
                logger.debug('[fcw] will not use tx event');
            }
        }
    };

    return invoke_cc;
}; 
```

#### 创建 query_cc.js

```go
$ vim query_cc.js 
```

query_cc.js 文件内容如下

```go
//-------------------------------------------------------------------
// Query Chaincode
//-------------------------------------------------------------------

module.exports = function (logger) {
    var query_cc = {};

    //-------------------------------------------------------------------
    // Query Chaincode - aka read the blockchain ledger
    //-------------------------------------------------------------------
    query_cc.query_chaincode = function (obj, options, cb) {
        logger.debug('[fcw] Querying Chaincode: ' + options.cc_function + '()');
        var channel = obj.channel;

        // send proposal to peer
        var request = {
            chaincodeId: options.chaincode_id,
            fcn: options.cc_function,
            args: options.cc_args,
            txId: null,                                                //apparently this is null for queries now
        };
        logger.debug('[fcw] Sending query req:', request);

        channel.queryByChaincode(request).then(function (response_payloads) {
            var formatted = format_query_resp(response_payloads);

            // --- response looks bad -- //
            if (formatted.parsed == null) {
                logger.debug('[fcw] Query parsed response is empty:', formatted.parsed);
            }
            if (formatted.error) {
                logger.debug('[fcw] Query response is an error:', formatted.error);
            }

            // --- response looks good --- //
            else {
                logger.debug('[fcw] Successful query transaction.');
            }
            if (cb) return cb(formatted.error, formatted);
        }).catch(function (err) {
            logger.error('[fcw] Error in query catch block', typeof err, err);

            if (cb) return cb(err, null);
            else return;
        });
    };

    //-----------------------------------------------------------------
    // Format Query Responses
    //------------------------------------------------------------------
    function format_query_resp(peer_responses) {
        var ret = {
            parsed: null,
            peers_agree: true,
            peer_payloads: [],
            error: null
        };
        var last = null;

        // -- iter on each peer's response -- //
        for (var i in peer_responses) {
            var as_string = peer_responses[i].toString('utf8');
            var as_obj = {};
            ret.peer_payloads.push(as_string);

            // -- compare peer responses -- //
            if (last != null) {    //check if all peers agree
                if (last !== as_string) {
                    logger.warn('[fcw] warning - some peers do not agree on query', last, as_string);
                    ret.peers_agree = false;
                }
                last = as_string;
            }

            try {
                if (as_string === '') {        //if its empty, thats okay... well its not great
                    as_obj = '';
                } else {
                    as_obj = JSON.parse(as_string);    //if we can parse it, its great
                }
                logger.debug('[fcw] Peer Query Response - len:', as_string.length, 'type:', typeof as_obj);
                if (ret.parsed === null) ret.parsed = as_obj;    //store the first one here
            } catch (e) {
                if (known_sdk_errors(as_string)) {
                    logger.error('[fcw] query resp looks like an error:', typeof as_string, as_string);
                    ret.parsed = null;
                    ret.error = as_string;
                } else if (as_string.indexOf('premature execution') >= 0) {
                    logger.warn('[fcw] query not successful, waiting on chaincode to start:', as_string);
                    ret.parsed = null;
                    ret.error = as_string;
                } else {
                    logger.warn('[fcw] warning - query resp is not json, might be okay:', typeof as_string, as_string);
                    ret.parsed = as_string;
                }
            }
        }
        return ret;
    }

    //test if this is a sdk thrown error (we want to handle chaincode thrown errors differently)
    function known_sdk_errors(str) {
        const known_errors = ['Error: failed to obtain', 'Error: Connect Failed'];        //list of known sdk errors from a query
        for (let i in known_errors) {
            if (str && str.indexOf(known_errors[i]) >= 0) {
                return true;
            }
        }
        return false;
    }

    return query_cc;
}; 
```

#### 创建 query_peer.js

```go
$ vim query_peer.js 
```

query_peer.js 文件内容如下

```go
//-------------------------------------------------------------------
// Query Peer - read the ledger / channel
//-------------------------------------------------------------------
var path = require('path');
const _ = require('lodash');

module.exports = function (logger) {
    var common = require(path.join(__dirname, './common.js'))(logger);
    var Peer = require('fabric-client/lib/Peer.js');
    var query_peer = {};

    //-------------------------------------------------------------------
    // Get Block
    //-------------------------------------------------------------------
    query_peer.query_block = function (obj, options, cb) {
        logger.debug('[fcw] Querying Block: ' + options.block_id);
        var channel = obj.channel;

        // send proposal to peer
        channel.queryBlock(Number(options.block_id)).then(
            function (block_resp) {
                if (cb) return cb(null, format_block(block_resp));
            }
        ).catch(
            function (err) {
                logger.error('[fcw] Error in query block', typeof err, err);
                var formatted = common.format_error_msg(err);

                if (cb) return cb(formatted, null);
                else return;
            }
            );
    };

    //-------------------------------------------------------------------
    // Get Channel Stats
    //-------------------------------------------------------------------
    query_peer.query_channel = function (obj, options, cb) {
        logger.debug('[fcw] Querying Channel Stats');
        var channel = obj.channel;

        // send proposal to peer
        channel.queryInfo().then(
            function (chain_resp) {
                chain_resp.currentBlockHash = buffer2hexStr(chain_resp.currentBlockHash.buffer);
                chain_resp.previousBlockHash = buffer2hexStr(chain_resp.previousBlockHash.buffer);
                if (cb) return cb(null, chain_resp);
            }
        ).catch(
            function (err) {
                logger.error('[fcw] Error in query block', typeof err, err);
                var formatted = common.format_error_msg(err);

                if (cb) return cb(formatted, null);
                else return;
            }
            );
    };

    //-------------------------------------------------------------------
    // Get Channel Members
    //-------------------------------------------------------------------
    query_peer.query_channel_members = function (obj, options, cb) {
        console.log('');
        logger.debug('[fcw] Querying Channel Members');
        var channel = obj.channel;

        channel.initialize().then(() => {
            let orgs = channel.getOrganizationUnits();
            if (cb) return cb(null, orgs);
        }).catch(function (err) {
            var formatted = common.format_error_msg(err);
            logger.error('failed to get members', formatted);
            if (cb) return cb(formatted, null);
            else return;
        });
    };

    //-------------------------------------------------------------------
    // Get List of Channels on Peer
    //-------------------------------------------------------------------
    query_peer.query_list_channels = function (obj, options, cb) {
        console.log('');
        logger.debug('List Channels:', options);
        var client = obj.client;

        // send proposal to peer
        client.queryChannels(
            new Peer(options.peer_urls[0], options.peer_tls_opts)
        ).then(function (resp) {
            resp.channels = _.sortBy(resp.channels, [channel => channel.channel_id]);
            if (cb) return cb(null, resp);
        }).catch(function (err) {
            logger.error('[fcw] Error in query block', typeof err, err);

            if (cb) return cb(err, null);
            else return;
        });
    };

    //format from byte array to hex string
    function buffer2hexStr(byteArray) {
        return byteArray.map(function (byte) {
            return ('0' + byte.toString(16)).slice(-2);
        }).join('');
    }

    // Format the Block
    // I don't have the slightest idea if this will hold up, seems ok for marbles =/ 
    function format_block(data, blockNumber) {
        var ret = {
            parsed: {
                block_id: data.header.number.low,
                data_count: data.data.data.length,
                metadata_count: data.metadata.metadata.length,
                txs: []
            },
            orig_data: data
        };
        data.blockNumber = blockNumber;        //copy it

        try {
            var tx = '';

            // -- move though the block data! -- //
            for (var i in ret.orig_data.data.data) {    //iter through transactions
                try {
                    tx = {
                        tx_id: ret.orig_data.data.data[i].payload.header.channel_header.tx_id,
                        instantiate: parse_if_instantiate(ret.orig_data.data.data[i].payload.data),
                        channel_id: ret.orig_data.data.data[i].payload.header.channel_header.channel_id,
                        chaincode_id: parse_4_chaincode_id(ret.orig_data.data.data[i].payload.data),
                        timestamp: Date.parse(ret.orig_data.data.data[i].payload.header.channel_header.timestamp),
                        creator_msp_id: parse_4_msp_id(ret.orig_data.data.data[i].payload.data),
                        endorsements: parse_4_endorsements(ret.orig_data.data.data[i].payload.data),
                        write_set: parse_4_write_set(ret.orig_data.data.data[i].payload.data),
                    };
                }
                catch (e) {
                    logger.warn('error in removing buffers - this does not matter', e);
                }

                // -- parse for parameters -- //
                var temp = stupid_parse(ret.orig_data.data.data[i].payload.data, tx.chaincode_id);
                tx.params = temp.parameters;
                tx.params_debug = temp.debug;
                ret.parsed.txs.push(tx);
            }
        }
        catch (e) {
            logger.warn('error in parsing data - this may matter', e);
        }

        // -- DONE -- //
        delete ret.orig_data;
        return ret;
    }

    // get msp id from header.creator
    function parse_4_msp_id(data) {
        try {
            return data.actions[0].header.creator.Mspid;
        } catch (e) {
            if (data.blockNumber >= 0) logger.warn('could not find msp id in tx payload', e);        //don't worry about genesis block
            return '-';
        }
    }

    // get chaincode id from rwset
    function parse_if_instantiate(data) {
        try {
            for (var i in data.actions[0].payload.action.proposal_response_payload.extension.results.ns_rwset) {
                if (data.actions[0].payload.action.proposal_response_payload.extension.results.ns_rwset[i].namespace === 'lscc') {    //skip system chaincode
                    if (data.actions[0].payload.action.proposal_response_payload.extension.results.ns_rwset[i].rwset.reads[0].version) {
                        return false;
                    } else {
                        return true;
                    }
                }
            }
        } catch (e) {
            return false;
        }
    }

    // get chaincode id from rwset
    function parse_4_chaincode_id(data) {
        try {
            for (var i in data.actions[0].payload.action.proposal_response_payload.extension.results.ns_rwset) {
                if (data.actions[0].payload.action.proposal_response_payload.extension.results.ns_rwset[i].namespace !== 'lscc') {    //skip system chaincode
                    return data.actions[0].payload.action.proposal_response_payload.extension.results.ns_rwset[i].namespace;
                }
            }
        } catch (e) {
            if (data.blockNumber >= 0) logger.warn('could not find chaincode id in tx payload', e);
            return '-';
        }
    }

    // get array of endorsements for transaction
    function parse_4_endorsements(data) {
        var msp_ids = [];
        try {
            for (var i in data.actions[0].payload.action.endorsements) {
                msp_ids.push(data.actions[0].payload.action.endorsements[i].endorser.Mspid);
            }
        } catch (e) {
            if (data.blockNumber >= 0) logger.warn('could not find endorsements for tx', e);
        }
        return msp_ids;
    }

    // get the chaincode's write set to the ledger
    function parse_4_write_set(data) {
        try {
            for (var i in data.actions[0].payload.action.proposal_response_payload.extension.results.ns_rwset) {
                if (data.actions[0].payload.action.proposal_response_payload.extension.results.ns_rwset[i].namespace !== 'lscc') {    //skip system chaincode
                    return data.actions[0].payload.action.proposal_response_payload.extension.results.ns_rwset[i].rwset.writes;
                }
            }
        } catch (e) {
            if (data.blockNumber >= 0) logger.warn('could not find chaincode id in tx payload', e);
            return [];
        }
    }

    // parse the block object to format for humans - get input parameters for tx
    function stupid_parse(data, chaincodeId) {
        var ret = { debug: {}, parameters: [] };
        var str = null;

        try {
            str = data.actions[0].payload.chaincode_proposal_payload.input.toString();
            ret.debug.original = str;
        } catch (e) {
            logger.warn('no tx data to parse for this block... might be okay');
            return ret;
        }

        // break up string
        try {
            ret.debug.startPos = str.indexOf(chaincodeId);                        //dumb detection
            ret.debug.str = str.substr(ret.debug.startPos + chaincodeId.length);
            ret.debug.stopPos = ret.debug.str.indexOf('\u0012');                //this is likely to break
            if (ret.debug.stopPos > 0) ret.debug.finalStr = ret.debug.str.substr(0, ret.debug.stopPos);
            else ret.debug.finalStr = ret.debug.str;
        } catch (e) {
            logger.warn('error parsing string in stupid parse...', e);
        }

        var word = '';
        if (!ret.debug.finalStr) {
            logger.error('parsing block data finalStr is undefined...');
            ret.parameters.push('undefined');
        }
        else if (ret.debug.finalStr.length > 5000) {                        //if its suspiciously long, don't process, self preservation
            logger.warn('parsing block data finalStr is too large, skipping', ret.debug.finalStr.length);
            ret.parameters.push('too long to show');
        } else {
            for (var i in ret.debug.finalStr) {                                //filter out gibberish
                if (ret.debug.finalStr.charCodeAt(i) >= 32 && ret.debug.finalStr.charCodeAt(i) <= 126) {
                    word += ret.debug.finalStr[i];
                }
                else {
                    if (word.length > 0) {                                    //end of word, push it
                        ret.parameters.push(word);
                    }
                    word = '';
                }
            }
            if (word.length > 0) {                                            //end of word (also its the last word), push it
                ret.parameters.push(word);
            }
        }

        return ret;
    }

    //-------------------------------------------------------------------
    // Get Installed Chaincodes
    //-------------------------------------------------------------------
    query_peer.query_installed_cc = function (obj, options, cb) {
        logger.debug('[fcw] Querying Installed Chaincodes\n');
        var channel = obj.channel;

        // send proposal to peer
        channel.queryInstalledChaincodes(
            new Peer(options.peer_urls[0], options.peer_tls_opts)
        ).then(function (resp) {
            if (cb) return cb(null, resp);
        }).catch(function (err) {
            logger.error('[fcw] Error in query installed chaincode', typeof err, err);
            var formatted = common.format_error_msg(err);

            if (cb) return cb(formatted, null);
            else return;
        });
    };

    //-------------------------------------------------------------------
    // Get Instantiated Chaincodes
    //-------------------------------------------------------------------
    query_peer.query_instantiated_cc = function (obj, options, cb) {
        logger.debug('[fcw] Querying Instantiated Chaincodes\n');
        var channel = obj.channel;

        // send proposal to peer
        channel.queryInstantiatedChaincodes().then(function (resp) {
            if (cb) return cb(null, resp);
        }).catch(function (err) {
            logger.error('[fcw] Error in query instantiated chaincodes', typeof err, err);
            var formatted = common.format_error_msg(err);

            if (cb) return cb(formatted, null);
            else return;
        });
    };

    return query_peer;
}; 
```

#### 编辑 utils/fc_wrangler/index.js 文件

打开链码调用的入口： utils/fc_wrangler/index.js 文件并编辑

```go
$ vim $HOME/kevin-marbles/utils/fc_wrangler/index.js 
```

文件中添加如下内容

```go
//-------------------------------------------------------------------
// Fabric Client Wrangler - Wrapper library for the Hyperledger Fabric Client SDK
//-------------------------------------------------------------------

module.exports = function (g_options, logger) {
    [......]
    var invoke_cc = require('./parts/invoke_cc.js')(g_options, logger);
    var query_cc = require('./parts/query_cc.js')(logger);
    var query_peer = require('./parts/query_peer.js')(logger);
    var ha = require('./parts/high_availability.js')(logger);

    [......]

    // Invoke Chaincode
    fcw.invoke_chaincode = function (obj, options, cb_done) {
        options.target_event_url = ha.get_event_url(options);    //get the desired event url to use
        invoke_cc.invoke_chaincode(obj, options, function (err, resp) {
            if (err != null) {                                            //looks like an error with the request
                if (ha.switch_peer(obj, options) == null) {    //try another peer
                    logger.info('Retrying invoke on different peer');
                    fcw.invoke_chaincode(obj, options, cb_done);
                } else {
                    if (cb_done) cb_done(err, resp);    //out of peers, give up
                }
            } else {    //all good, pass resp back to callback
                ha.success_peer_position = ha.using_peer_position;        //remember the last good one
                if (cb_done) cb_done(err, resp);
            }
        });
    };

    // Query Chaincode
    fcw.query_chaincode = function (obj, options, cb_done) {
        query_cc.query_chaincode(obj, options, function (err, resp) {
            if (err != null) {                                            //looks like an error with the request
                if (ha.switch_peer(obj, options) == null) {    //try another peer
                    logger.info('Retrying query on different peer');
                    fcw.query_chaincode(obj, options, cb_done);
                } else {
                    if (cb_done) cb_done(err, resp);//out of peers, give up
                }
            } else {    //all good, pass resp back to callback
                ha.success_peer_position = ha.using_peer_position;        //remember the last good one
                if (cb_done) cb_done(err, resp);
            }
        });
    };

    // ------------------------------------------------------------------------
    // Ledger Functions
    // ------------------------------------------------------------------------
    // Get Block Data
    fcw.query_block = function (obj, options, cb_done) {
        query_peer.query_block(obj, options, cb_done);
    };

    // ------------------------------------------------------------------------
    // Channel Functions
    // ------------------------------------------------------------------------
    // Get Members on Channel
    fcw.query_channel_members = function (obj, options, cb_done) {
        query_peer.query_channel_members(obj, options, cb_done);
    };

    // Get Block Height of Channel
    fcw.query_channel = function (obj, options, cb_done) {
        query_peer.query_channel(obj, options, function (err, resp) {
            if (err != null) {                                            //looks like an error with the request
                if (ha.switch_peer(obj, options) == null) {    //try another peer
                    logger.info('Retrying query on different peer');
                    fcw.query_channel(obj, options, cb_done);
                } else {
                    if (cb_done) cb_done(err, resp);//out of peers, give up
                }
            } else {    //all good, pass resp back to callback
                ha.success_peer_position = ha.using_peer_position;        //remember the last good one
                if (cb_done) cb_done(err, resp);
            }
        });
    };

    // Get list of installed cc's
    fcw.query_installed_cc = function (obj, options, cb_done) {
        query_peer.query_installed_cc(obj, options, cb_done);
    };

    // get list of instantiated cc's
    fcw.query_instantiated_cc = function (obj, options, cb_done) {
        query_peer.query_instantiated_cc(obj, options, cb_done);
    };

    // get list of channels
    fcw.query_list_channels = function (obj, options, cb_done) {
        query_peer.query_list_channels(obj, options, cb_done);
    };

    return fcw;
}; 
```