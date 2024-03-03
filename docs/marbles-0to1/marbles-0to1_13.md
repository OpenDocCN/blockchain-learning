# 3.5 调用链码实现事务及查询脚本

## 从零到壹实现 Marbles 资产管理系统 （Fabric-SDK-Node）之－调用链码实现交易/查询

我们应该创建一个应用中关于链码操作的公共 JS 库，该库中每个函数都使用 FCW 库与 Peers/Orderer 节点进行通信。

```js
$ cd $HOME/kevin-marbles/utils 
```

### 创建 marbles_cc_lib.js 文件并编辑

通过调用 utils/fc_wrangler/index.js 文件实现对事务的相关操作

```js
$ vim marbles_cc_lib.js 
```

marbles_cc_lib.js 文件完整内容如下

```js
//-------------------------------------------------------------------
// Marbles Chaincode Library
// - this contains the most interesting code pieces of marbles.
// - each function is using the FCW library to communicate to the peer/orderer
// - from here we can interact with our chaincode.
//   - the cc_function is the chaincode function we will call
//   - the cc_args are the arguments to pass to your chaincode function
//-------------------------------------------------------------------

module.exports = function (enrollObj, g_options, fcw, logger) {
    var marbles_chaincode = {};

    // Chaincode -------------------------------------------------------------------------------

    //check if chaincode exists
    marbles_chaincode.check_if_already_instantiated = function (options, cb) {
        console.log('');
        logger.info('Checking for chaincode...');

        var opts = {
            peer_urls: g_options.peer_urls,
            peer_tls_opts: g_options.peer_tls_opts,
            channel_id: g_options.channel_id,
            chaincode_id: g_options.chaincode_id,
            chaincode_version: g_options.chaincode_version,
            cc_function: 'read',
            cc_args: ['selftest']
        };
        fcw.query_chaincode(enrollObj, opts, function (err, resp) {  // send a request to our peer
            if (err != null) {
                if (cb) return cb(err, resp);
            }
            else {
                if (resp.parsed == null || isNaN(resp.parsed)) {     //if nothing is here, no chaincode
                    if (cb) return cb({ error: 'chaincode not found' }, resp);
                }
                else {
                    if (cb) return cb(null, resp);
                }
            }
        });
    };

    //check chaincode version
    marbles_chaincode.check_version = function (options, cb) {
        console.log('');
        logger.info('Checking chaincode and ui compatibility...');

        var opts = {
            peer_urls: g_options.peer_urls,
            peer_tls_opts: g_options.peer_tls_opts,
            channel_id: g_options.channel_id,
            chaincode_id: g_options.chaincode_id,
            chaincode_version: g_options.chaincode_version,
            cc_function: 'read',
            cc_args: ['marbles_ui']
        };
        fcw.query_chaincode(enrollObj, opts, function (err, resp) {
            if (err != null) {
                if (cb) return cb(err, resp);
            }
            else {
                if (resp.parsed == null) {//if nothing is here, no chaincode
                    if (cb) return cb({ error: 'chaincode not found' }, resp);
                }
                else {
                    if (cb) return cb(null, resp);
                }
            }
        });
    };

    // Marbles -------------------------------------------------------------------------------

    //create a marble
    marbles_chaincode.create_a_marble = function (options, cb) {
        console.log('');
        logger.info('Creating a marble...');

        var opts = {
            peer_urls: g_options.peer_urls,
            peer_tls_opts: g_options.peer_tls_opts,
            channel_id: g_options.channel_id,
            chaincode_id: g_options.chaincode_id,
            chaincode_version: g_options.chaincode_version,
            event_urls: g_options.event_urls,
            endorsed_hook: options.endorsed_hook,
            ordered_hook: options.ordered_hook,
            cc_function: 'init_marble',
            cc_args: [
                'm' + leftPad(Date.now() + randStr(5), 19),
                options.args.color,
                options.args.size,
                options.args.owner_id,
                options.args.auth_company
            ],
        };
        fcw.invoke_chaincode(enrollObj, opts, function (err, resp) {
            if (cb) {
                if (!resp) resp = {};
                resp.id = opts.cc_args[0];    //pass marble id back
                cb(err, resp);
            }
        });
    };

    //get marble
    marbles_chaincode.get_marble = function (options, cb) {
        logger.info('fetching marble ' + options.marble_id + ' list...');

        var opts = {
            peer_urls: g_options.peer_urls,
            peer_tls_opts: g_options.peer_tls_opts,
            channel_id: g_options.channel_id,
            chaincode_version: g_options.chaincode_version,
            chaincode_id: g_options.chaincode_id,
            cc_function: 'read',
            cc_args: [options.args.marble_id]
        };
        fcw.query_chaincode(enrollObj, opts, cb);
    };

    //set marble owner
    marbles_chaincode.set_marble_owner = function (options, cb) {
        console.log('');
        logger.info('Setting marble owner...');

        var opts = {
            peer_urls: g_options.peer_urls,
            peer_tls_opts: g_options.peer_tls_opts,
            channel_id: g_options.channel_id,
            chaincode_id: g_options.chaincode_id,
            chaincode_version: g_options.chaincode_version,
            event_urls: g_options.event_urls,
            endorsed_hook: options.endorsed_hook,
            ordered_hook: options.ordered_hook,
            cc_function: 'set_owner',
            cc_args: [
                options.args.marble_id,
                options.args.owner_id,
                options.args.auth_company
            ],
        };
        fcw.invoke_chaincode(enrollObj, opts, cb);
    };

    //delete marble
    marbles_chaincode.delete_marble = function (options, cb) {
        console.log('');
        logger.info('Deleting a marble...');

        var opts = {
            peer_urls: g_options.peer_urls,
            peer_tls_opts: g_options.peer_tls_opts,
            channel_id: g_options.channel_id,
            chaincode_id: g_options.chaincode_id,
            chaincode_version: g_options.chaincode_version,
            event_urls: g_options.event_urls,
            endorsed_hook: options.endorsed_hook,
            ordered_hook: options.ordered_hook,
            cc_function: 'delete_marble',
            cc_args: [options.args.marble_id, options.args.auth_company],
        };
        fcw.invoke_chaincode(enrollObj, opts, cb);
    };

    //get history for key
    marbles_chaincode.get_history = function (options, cb) {
        logger.info('Getting history for...', options.args);

        var opts = {
            peer_urls: g_options.peer_urls,
            peer_tls_opts: g_options.peer_tls_opts,
            channel_id: g_options.channel_id,
            chaincode_id: g_options.chaincode_id,
            chaincode_version: g_options.chaincode_version,
            event_urls: g_options.event_urls,
            endorsed_hook: options.endorsed_hook,
            ordered_hook: options.ordered_hook,
            cc_function: 'getHistory',
            cc_args: [options.args.id]
        };
        fcw.query_chaincode(enrollObj, opts, cb);
    };

    //get multiple marbles/owners by start and stop ids
    marbles_chaincode.get_multiple_keys = function (options, cb) {
        logger.info('Getting marbles between ids', options.args);

        var opts = {
            peer_urls: g_options.peer_urls,
            peer_tls_opts: g_options.peer_tls_opts,
            channel_id: g_options.channel_id,
            chaincode_id: g_options.chaincode_id,
            chaincode_version: g_options.chaincode_version,
            event_urls: g_options.event_urls,
            endorsed_hook: options.endorsed_hook,
            ordered_hook: options.ordered_hook,
            cc_function: 'getMarblesByRange',
            cc_args: [options.args.start_id, options.args.stop_id]
        };
        fcw.query_chaincode(enrollObj, opts, cb);
    };

    // Owners -------------------------------------------------------------------------------

    //register a owner/user
    marbles_chaincode.register_owner = function (options, cb) {
        console.log('');
        logger.info('Creating a marble owner...');

        var opts = {
            peer_urls: g_options.peer_urls,
            peer_tls_opts: g_options.peer_tls_opts,
            channel_id: g_options.channel_id,
            chaincode_id: g_options.chaincode_id,
            chaincode_version: g_options.chaincode_version,
            event_urls: g_options.event_urls,
            endorsed_hook: options.endorsed_hook,
            ordered_hook: options.ordered_hook,
            cc_function: 'init_owner',
            cc_args: [
                'o' + leftPad(Date.now() + randStr(5), 19),
                options.args.marble_owner,
                options.args.owners_company
            ],
        };
        fcw.invoke_chaincode(enrollObj, opts, function (err, resp) {
            if (cb) {
                if (!resp) resp = {};
                resp.id = opts.cc_args[0];    //pass owner id back
                cb(err, resp);
            }
        });
    };

    //get a owner/user
    marbles_chaincode.get_owner = function (options, cb) {
        var full_username = build_owner_name(options.args.marble_owner, options.args.owners_company);
        console.log('');
        logger.info('Fetching owner ' + full_username + ' list...');

        var opts = {
            peer_urls: g_options.peer_urls,
            peer_tls_opts: g_options.peer_tls_opts,
            channel_id: g_options.channel_id,
            chaincode_id: g_options.chaincode_id,
            chaincode_version: g_options.chaincode_version,
            cc_function: 'read',
            cc_args: [full_username]
        };
        fcw.query_chaincode(enrollObj, opts, cb);
    };

    //get the owner list
    marbles_chaincode.get_owner_list = function (options, cb) {
        console.log('');
        logger.info('Fetching owner index list...');

        var opts = {
            peer_urls: g_options.peer_urls,
            peer_tls_opts: g_options.peer_tls_opts,
            channel_id: g_options.channel_id,
            chaincode_id: g_options.chaincode_id,
            chaincode_version: g_options.chaincode_version,
            cc_function: 'read',
            cc_args: ['_ownerindex']
        };
        fcw.query_chaincode(enrollObj, opts, cb);
    };

    // disable a marble owner
    marbles_chaincode.disable_owner = function (options, cb) {
        console.log('');
        logger.info('Disabling a marble owner...');

        var opts = {
            peer_urls: g_options.peer_urls,
            peer_tls_opts: g_options.peer_tls_opts,
            channel_id: g_options.channel_id,
            chaincode_id: g_options.chaincode_id,
            chaincode_version: g_options.chaincode_version,
            event_urls: g_options.event_urls,
            endorsed_hook: options.endorsed_hook,
            ordered_hook: options.ordered_hook,
            cc_function: 'disable_owner',
            cc_args: [
                options.args.owner_id,
                options.args.auth_company
            ],
        };
        fcw.invoke_chaincode(enrollObj, opts, function (err, resp) {
            if (cb) {
                if (!resp) resp = {};
                resp.id = opts.cc_args[0];    //pass owner id back
                cb(err, resp);
            }
        });
    };

    //build full name
    marbles_chaincode.build_owner_name = function (username, company) {
        return build_owner_name(username, company);
    };

    // All ---------------------------------------------------------------------------------

    //build full name
    marbles_chaincode.read_everything = function (options, cb) {
        console.log('');
        logger.info('Fetching EVERYTHING...');

        var opts = {
            peer_urls: g_options.peer_urls,
            peer_tls_opts: g_options.peer_tls_opts,
            channel_id: g_options.channel_id,
            chaincode_version: g_options.chaincode_version,
            chaincode_id: g_options.chaincode_id,
            cc_function: 'read_everything',
            cc_args: ['']
        };
        fcw.query_chaincode(enrollObj, opts, cb);
    };

    // get block height of the channel
    marbles_chaincode.channel_stats = function (options, cb) {
        var opts = {
            peer_urls: g_options.peer_urls,
            peer_tls_opts: g_options.peer_tls_opts
        };
        fcw.query_channel(enrollObj, opts, cb);
    };

    // Other -------------------------------------------------------------------------------

    // Format Owner's Actual Key Name
    function build_owner_name(username, company) {
        return username.toLowerCase() + '.' + company;
    }

    // random string of x length
    function randStr(length) {
        var text = '';
        var possible = 'abcdefghijkmnpqrstuvwxyz0123456789ABCDEFGHJKMNPQRSTUVWXYZ';
        for (var i = 0; i < length; i++) text += possible.charAt(Math.floor(Math.random() * possible.length));
        return text;
    }

    // left pad string with "0"s
    function leftPad(str, length) {
        for (var i = str.length; i < length; i++) str = '0' + String(str);
        return str;
    }

    return marbles_chaincode;
}; 
```

### 创建 startup_lib.js 文件

创建一个启动库文件，其中包含了应用初始状态在启动期间所需调用的函数，具体功能通过调用 marbles_cc_lib.js 实现。

返回到 utils 目录下

```js
$ cd $HOME/kevin-marbles/utils
$ vim startup_lib.js 
```

startup_lib.js 文件完整内容如下

```js
// ============================================================================================================================
// startup_lib.js
// This file has the functions we call during start up
// ============================================================================================================================
var async = require('async');

module.exports = function (logger, cp, fcw, marbles_lib, ws_server) {
    var startup_lib = {};
    var enrollObj = {};
    var misc = require('./misc.js')(logger);    //random non-blockchain related functions
    var more_entropy = misc.randStr(32);
    var cc_detect_attempt = 0;

    // --------------------------------------------------------
    // Handle WS Setup Messages
    // --------------------------------------------------------
    startup_lib.setup_ws_steps = function (data) {

        // --- [6] Enroll the admin (repeat if needed)  --- //
        if (data.configure === 'enrollment') {
            startup_lib.removeKVS();
            cp.write(data);                                                                //write new config data to file
            startup_lib.enroll_admin(1, function (e) {
                if (e == null) {
                    startup_lib.setup_marbles_lib('localhost', cp.getMarblesPort(), function () {
                        startup_lib.detect_prev_startup({ startup: false }, function (err) {
                            if (err) {
                                startup_lib.create_assets(cp.getMarbleUsernames());     //builds marbles, then starts webapp
                            }
                        });
                    });
                }
            });
        }

        // --- [7] Find instantiated chaincode --- //
        else if (data.configure === 'find_chaincode') {
            cp.write(data);                                                                //write new config data to file
            startup_lib.enroll_admin(1, function (e) {                                    //re-enroll b/c we may be using new peer/order urls
                if (e == null) {
                    startup_lib.setup_marbles_lib('localhost', cp.getMarblesPort(), function () {
                        startup_lib.detect_prev_startup({ startup: true }, function (err) {
                            if (err) {
                                startup_lib.create_assets(cp.getMarbleUsernames());     //builds marbles, then starts webapp
                            }
                        });
                    });
                }
            });
        }

        // --- [8] Register marble owners --- /
        else if (data.configure === 'register') {
            startup_lib.create_assets(data.build_marble_owners);
        }
    };

    // Wait for the user to help correct the config file so we can startup!
    startup_lib.startup_unsuccessful = function (host, port) {
        console.log('\n\n- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -');
        logger.info('Detected that we have NOT launched successfully yet');
        logger.debug('Open your browser to http://' + host + ':' + port + ' and login as "admin" to initiate startup');
        console.log('- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -\n\n');
        // we wait here for the user to go the browser, then setup_marbles_lib() will be called from WS msg
    };

    // Find if marbles has started up successfully before
    startup_lib.detect_prev_startup = function (opts, cb) {
        logger.info('Checking ledger for marble owners listed in the config file');
        marbles_lib.read_everything(null, function (err, resp) {                    //read the ledger for marble owners
            if (err != null) {
                logger.warn('Error reading ledger');
                if (cb) cb(true);
            } else {
                if (!detectCompany(resp) || startup_lib.find_missing_owners(resp)) {    //check if each user in the settings file has been created in the ledger
                    logger.info('We need to make marble owners');                        //there are marble owners that do not exist!
                    ws_server.record_state('register_owners', 'waiting');
                    ws_server.broadcast_state();
                    if (cb) cb(true);
                } else {
                    ws_server.record_state('register_owners', 'success');        //everything is good
                    ws_server.broadcast_state();
                    logger.info('Everything is in place');
                    if (cb) cb(null);
                }
            }
        });
    };

    // Detect if we have created users for this company yet
    function detectCompany(data) {
        if (data && data.parsed) {
            for (let i in data.parsed.owners) {
                if (data.parsed.owners[i].company === process.env.marble_company) {
                    logger.debug('This company has registered marble owners');
                    return true;
                }
            }
        }

        logger.debug('This company has not registered marble owners');
        return false;
    }

    // Detect if there are marble usernames in the settings doc that are not in the ledger
    startup_lib.find_missing_owners = function (resp) {
        let ledger = (resp) ? resp.parsed : [];
        let user_base = cp.getMarbleUsernames();

        for (let x in user_base) {
            let found = false;
            logger.debug('Looking for marble owner:', user_base[x]);
            for (let i in ledger.owners) {
                if (user_base[x] === ledger.owners[i].username) {
                    found = true;
                    break;
                }
            }
            if (found === false) {
                logger.debug('Did not find marble username:', user_base[x]);
                return true;
            }
        }
        return false;
    };

    // setup marbles library and check if cc is instantiated
    startup_lib.setup_marbles_lib = function (host, port, cb) {
        var opts = cp.makeMarblesLibOptions();
        marbles_lib = require('./marbles_cc_lib.js')(enrollObj, opts, fcw, logger);
        ws_server.setup(null, marbles_lib);
        cc_detect_attempt++; // keep track of how many times we've done this

        logger.debug('Checking if chaincode is already instantiated or not', cc_detect_attempt);
        const channel = cp.getChannelId();
        const first_peer = cp.getFirstPeerName(channel);
        var options = {
            peer_urls: [cp.getPeersUrl(first_peer)],
        };

        marbles_lib.check_if_already_instantiated(options, function (not_instantiated, enrollUser) {
            if (not_instantiated) {        // if this is truthy we have not yet instantiated.... error
                console.log('debug', typeof not_instantiated, not_instantiated);
                if (cc_detect_attempt <= 40 && typeof not_instantiated === 'string' && not_instantiated.indexOf('premature execution') >= 0) {
                    console.log('');
                    logger.debug('Chaincode is still starting! this can take a minute or two.  I\'ll check again in a moment.', cc_detect_attempt);
                    ws_server.record_state('find_chaincode', 'polling');
                    ws_server.broadcast_state();
                    return setTimeout(function () {        // try again in a few seconds, this loops for awhile so ... beware
                        startup_lib.setup_marbles_lib(host, port, cb);
                    }, 15 * 1000);
                } else {
                    console.log('');
                    logger.debug('Chaincode was not detected: "' + cp.getChaincodeId() + '", all stop');
                    logger.debug('Open your browser to http://' + host + ':' + port + ' and login to tweak settings for startup');
                    ws_server.record_state('find_chaincode', 'failed');
                    ws_server.broadcast_state();
                }
            } else {    // else we already instantiated
                console.log('\n----------------------------- Chaincode found on channel "' + cp.getChannelId() + '" -----------------------------\n');
                cc_detect_attempt = 0;    // reset

                // --- Check Chaincode Compatibility  --- //
                marbles_lib.check_version(options, function (err, resp) {
                    if (cp.errorWithVersions(resp)) {                                // incompatible cc w/app
                        ws_server.record_state('find_chaincode', 'failed');
                        ws_server.broadcast_state();
                    } else {                                                        // compatible cc w/app
                        logger.info('Chaincode version is good');
                        ws_server.record_state('find_chaincode', 'success');
                        ws_server.broadcast_state();
                        if (cb) cb(null);
                    }
                });
            }
        });
    };

    // Enroll an admin with the CA for this peer/channel
    startup_lib.enroll_admin = function (attempt, cb) {
        fcw.enroll(cp.makeEnrollmentOptions(0), function (errCode, obj) {
            if (errCode != null) {
                logger.error('could not enroll...');

                // --- Try Again ---  //
                if (attempt >= 2) {
                    if (cb) cb(errCode);
                } else {
                    startup_lib.removeKVS();
                    startup_lib.enroll_admin(++attempt, cb);
                }
            } else {
                enrollObj = obj;
                if (cb) cb(null);
            }
        });
    };

    // Create marbles and marble owners, owners first
    startup_lib.create_assets = function (build_marbles_users) {
        build_marbles_users = misc.saferNames(build_marbles_users);
        logger.info('Creating marble owners and marbles');
        var owners = [];

        if (build_marbles_users && build_marbles_users.length > 0) {
            async.each(build_marbles_users, function (username, owner_cb) {
                logger.debug('- creating marble owner: ', username);

                // --- Create Each User --- //
                startup_lib.create_owners(0, username, function (errCode, resp) {
                    owners.push({ id: resp.id, username: username });
                    owner_cb();
                });

            }, function (err) {
                logger.info('finished creating owners, now for marbles');
                if (err == null) {

                    var marbles = [];
                    var marblesEach = 3;                                                //number of marbles each owner gets
                    for (var i in owners) {
                        for (var x = 0; x < marblesEach; x++) {
                            marbles.push(owners[i]);
                        }
                    }
                    logger.debug('prepared marbles obj', marbles.length, marbles);

                    // --- Create Marbles--- //
                    setTimeout(function () {
                        async.each(marbles, function (owner_obj, marble_cb) {             //iter through each one
                            startup_lib.create_marbles(owner_obj.id, owner_obj.username, marble_cb);
                        }, function (err) {                                                //marble owner creation finished
                            logger.debug('- finished creating asset');
                            if (err == null) {
                                startup_lib.all_done();                                                //delay for peer catch up
                            }
                        });
                    }, cp.getBlockDelay());
                }
            });
        }
        else {
            logger.debug('- there are no new marble owners to create');
            startup_lib.all_done();
        }
    };

    // Create the marble owner
    startup_lib.create_owners = function (attempt, username, cb) {
        const channel = cp.getChannelId();
        const first_peer = cp.getFirstPeerName(channel);
        var options = {
            peer_urls: [cp.getPeersUrl(first_peer)],
            args: {
                marble_owner: username,
                owners_company: process.env.marble_company
            }
        };
        marbles_lib.register_owner(options, function (e, resp) {
            if (e != null) {
                console.log('');
                logger.error('error creating the marble owner', e, resp);
                cb(e, resp);
            }
            else {
                cb(null, resp);
            }
        });
    };

    // Create 1 marble
    startup_lib.create_marbles = function (owner_id, username, cb) {
        var randOptions = startup_lib.build_marble_options(owner_id, username, process.env.marble_company);
        const channel = cp.getChannelId();
        const first_peer = cp.getFirstPeerName(channel);
        console.log('');
        logger.debug('[startup] going to create marble:', randOptions);
        var options = {
            chaincode_id: cp.getChaincodeId(),
            peer_urls: [cp.getPeersUrl(first_peer)],
            args: randOptions
        };
        marbles_lib.create_a_marble(options, function () {
            return cb();
        });
    };

    // Create random marble arguments (it is not important for it to be random, just more fun)
    startup_lib.build_marble_options = function (id, username, company) {
        var colors = ['white', 'green', 'blue', 'purple', 'red', 'pink', 'orange', 'black', 'yellow'];
        var sizes = ['35', '16'];
        var color_index = misc.simple_hash(more_entropy + company) % colors.length;        //build a pseudo random index to pick a color
        var size_index = misc.getRandomInt(0, sizes.length);                            //build a random size for this marble
        return {
            color: colors[color_index],
            size: sizes[size_index],
            owner_id: id,
            auth_company: process.env.marble_company
        };
    };

    // Clean Up OLD KVS
    startup_lib.removeKVS = function () {
        try {
            logger.warn('removing older kvs and trying to enroll again');
            misc.rmdir(cp.getKvsPath({ going2delete: true }));            //delete old kvs folder
            logger.warn('removed older kvs');
        } catch (e) {
            logger.error('could not delete old kvs', e);
        }
    };

    // We are done, inform the clients
    startup_lib.all_done = function () {
        console.log('\n------------------------------------------ All Done ------------------------------------------\n');
        ws_server.record_state('register_owners', 'success');
        ws_server.broadcast_state();
        ws_server.check_for_updates(null);                                    //call the periodic task to get the state of everything
    };

    return startup_lib;
}; 
```