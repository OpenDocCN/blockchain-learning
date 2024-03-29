# 4.3 启动 Web 应用

[TOC]

## 从零到壹实现 Marbles 资产管理系统 （Fabric-SDK-Node）之－启动应用

### 创建 app.js

```js
$ cd $HOME/kevin-marbles
$ vim app.js 
```

文件完整内容如下

```js
'use strict';
/* global process */
/* global __dirname */
var express = require('express');
var session = require('express-session');
var compression = require('compression');
var serve_static = require('serve-static');
var path = require('path');
var cookieParser = require('cookie-parser');
var http = require('http');
var app = express();
var cors = require('cors');
var ws = require('ws');        // websocket module
var winston = require('winston');    // logger module

// ------------- Init our libraries ------------- //
var wss = {};
var marbles_lib = null;
var logger = new (winston.Logger)({
    level: 'debug',
    transports: [
        new (winston.transports.Console)({ colorize: true, stderrLevels: ['error'] }),
    ]
});
var misc = require('./utils/misc.js')(logger);                                                // mis.js has generic (non-blockchain) related functions
misc.check_creds_for_valid_json();
var cp = require(__dirname + '/utils/connection_profile_lib/index.js')(process.env.creds_filename, logger);    // parses our cp file/data
var host = 'localhost';
var port = cp.getMarblesPort();
process.env.marble_company = cp.getClientsOrgName();

// fabric client wrangler wraps the SDK
var fcw = require('./utils/fc_wrangler/index.js')({ block_delay: cp.getBlockDelay() }, logger);

// websocket logic
var ws_server = require('./utils/websocket_server_side.js')(cp, fcw, logger);

// setup/startup logic
var startup_lib = require('./utils/startup_lib.js')(logger, cp, fcw, marbles_lib, ws_server);

// --- Setup Express --- //
app.set('views', path.join(__dirname, 'views'));
app.set('view engine', 'pug');
app.use(compression());
app.use(cookieParser());
app.use(serve_static(path.join(__dirname, 'public')));
app.use(session({ secret: 'lostmymarbles', resave: true, saveUninitialized: true }));
app.options('*', cors());
app.use(cors());

// ============================================================================================================================
//     HTTP Webserver Routing
// ============================================================================================================================
app.use(function (req, res, next) {
    logger.debug('------------------------------------------ incoming request ------------------------------------------');
    logger.debug('New ' + req.method + ' request for', req.url);
    req.bag = {};                                                                    // create object for client exposed session data
    req.bag.session = req.session;
    next();
});
app.use('/', require('./routes/site_router.js')(logger, cp));                        // most routes are in here

// ------ Error Handling --------
app.use(function (req, res, next) {
    var err = new Error('Not Found');
    err.status = 404;
    next(err);
});
app.use(function (err, req, res, next) {
    logger.debug('Errors -', req.url);
    var errorCode = err.status || 500;
    res.status(errorCode);
    req.bag.error = { msg: err.stack, status: errorCode };
    if (req.bag.error.status == 404) req.bag.error.msg = 'Sorry, I cannot locate that file';
    res.render('template/error', { bag: req.bag });
});

// ============================================================================================================================
//     Launch HTTP Webserver
// ============================================================================================================================
var server = http.createServer(app).listen(port, function () { });
process.env.NODE_TLS_REJECT_UNAUTHORIZED = '0';
server.timeout = 240000;
console.log('\n');
console.log('----------------------------------- Server Up - ' + host + ':' + port + ' -----------------------------------');
process.on('uncaughtException', function (err) {
    logger.error('Caught exception: ', err.stack);        // marbles never gives up! (this is bad practice, but its just a demo)
    if (err.stack.indexOf('EADDRINUSE') >= 0) {            // well, except for this error
        logger.warn('---------------------------------------------------------------');
        logger.warn('----------------------------- Ah! -----------------------------');
        logger.warn('---------------------------------------------------------------');
        logger.error('You already have something running on port ' + port + '!');
        logger.error('Kill whatever is running on that port OR change the port setting in your marbles config file: ' + cp.config_path);
        process.exit();
    }
    if (wss && wss.broadcast) {    // if possible send the error out to the client
        wss.broadcast({
            msg: 'error',
            info: 'this is a backend error!',
            e: err.stack,
        });
    }
});

// ------------------------------------------------------------------------------------------------------------------------------
// The real show starts here!
// - everything above was static setup, its not interesting
// - the steps below will run when the application starts, they are mildly interesting
// - these steps will test the ability to communicate with marbles chaincode on your blockchain network
// ------------------------------------------------------------------------------------------------------------------------------
let config_error = cp.checkConfig();
setupWebSocket();    // http server is already up, make the ws one now

if (config_error) {
    ws_server.record_state('checklist', 'failed');    // checklist step is done
    ws_server.broadcast_state();
} else {
    ws_server.record_state('checklist', 'success');    // checklist step is done
    console.log('\n');

    // --- [1] Test enrolling with our CA --- //
    startup_lib.enroll_admin(1, function (e) {
        if (e != null) {
            logger.warn('Error enrolling admin');
            ws_server.record_state('enrolling', 'failed');
            ws_server.broadcast_state();
            startup_lib.startup_unsuccessful(host, port);
        } else {
            logger.info('Success enrolling admin');
            ws_server.record_state('enrolling', 'success');
            ws_server.broadcast_state();

            // --- [2] Setup Marbles Library --- //
            startup_lib.setup_marbles_lib(host, port, function () {

                // --- [3] Check If We have Started Marbles Before --- //
                startup_lib.detect_prev_startup({ startup: true }, function (err) {
                    if (err) {
                        startup_lib.startup_unsuccessful(host, port);
                    } else {
                        console.log('\n\n- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -');
                        logger.debug('Detected that we have launched successfully before');
                        logger.debug('Welcome back - Marbles is ready');
                        logger.debug('Open your browser to http://' + host + ':' + port + ' and login as "admin"');
                        console.log('- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -\n\n');
                    }
                });

                // --- [4] Wait for the user to go to the browser (step 5 is in websocket code below) --- //
                // ZZzzzzzZZZzzzzzzzzZZzZZZZZzzzZZZzzz
            });
        }
    });
}

// ============================================================================================================================
//     Launch WebSocket Server
// ============================================================================================================================
function setupWebSocket() {
    console.log('------------------------------------------ Websocket Up ------------------------------------------');
    wss = new ws.Server({ server: server });    // start the websocket now
    wss.on('connection', function connection(ws) {

        // -- Process all websocket messages -- //
        ws.on('message', function incoming(message) {
            console.log(' ');
            console.log('-------------------------------- Incoming WS Msg --------------------------------');
            logger.debug('[ws] received ws msg:', message);
            var data = null;
            try {
                data = JSON.parse(message);    // it better be json
            } catch (e) {
                logger.debug('[ws] message error', message, e.stack);
            }

            // --- [5] Process the ws message  --- //
            if (data && data.type == 'setup') {    // its a setup request, enter the setup code
                logger.debug('[ws] setup message', data);
                startup_lib.setup_ws_steps(data);    // <-- open startup_lib.js to view the rest of the start up code

            } else if (data) {    // its a normal marble request, pass it to the lib for processing
                ws_server.process_msg(ws, data);    // <-- the interesting "blockchainy" code is this way (websocket_server_side.js)
            }
        });

        // log web socket errors
        ws.on('error', function (e) { logger.debug('[ws] error', e); });

        // log web socket connection disconnects (typically client closed browser)
        ws.on('close', function () { logger.debug('[ws] closed'); });

        // whenever someone connects, tell them our app's state
        ws.send(JSON.stringify(ws_server.build_state_msg()));                // tell client our app state
    });

    // --- Send a message to all connected clients --- //
    wss.broadcast = function broadcast(data) {
        var i = 0;
        wss.clients.forEach(function each(client) {                            // iter on each connection
            try {
                logger.debug('[ws] broadcasting to clients. ', (++i), data.msg);
                client.send(JSON.stringify(data));                            // BAM, send the data
            } catch (e) {
                logger.debug('[ws] error broadcast ws', e);
            }
        });
    };

    ws_server.setup(wss, null);
} 
```

### 创建 gulpfile .js

```js
$ vim gulpfile.js 
```

文件内容如下

```js
var path = require('path');
var gulp = require('gulp');
var sass = require('gulp-sass');
var concat = require('gulp-concat');
var cleanCSS = require('gulp-clean-css');
var rename = require('gulp-rename');
var spawn = require('child_process').spawn;
var node, env = process.env;

// --------------------- Build CSS --------------------- //
gulp.task('build-sass', function () {
    gulp.src(path.join(__dirname, '/scss/*.scss'))
        .pipe(sass().on('error', sass.logError))
        .pipe(gulp.dest(path.join(__dirname, '/scss/temp')))            //build them here first
        .pipe(concat('main.css'))                                        //concat them all
        //.pipe(gulp.dest(path.join(__dirname, '/public/css')))
        .pipe(cleanCSS())                                                //minify
        .pipe(rename('main.min.css'))
        .pipe(gulp.dest(path.join(__dirname, '/public/css')));            //dump it here
});

// -------------------- Run Marbles -------------------- //
gulp.task('server', function (a, b) {
    if (node) node.kill();
    node = spawn('node', ['app.js'], { env: env, stdio: 'inherit' });    //command, file, options
});

// -------------- Watch for File Changes --------------- //
gulp.task('watch-sass', ['build-sass'], function () {
    gulp.watch(path.join(__dirname, '/scss/*.scss'), ['build-sass']);
});
gulp.task('watch-server', function () {
    gulp.watch(path.join(__dirname, '/routes/**/*.js'), ['server']);
    gulp.watch([path.join(__dirname, '/utils/fc_wrangler/*.js')], ['server']);
    gulp.watch([path.join(__dirname, '/utils/*.js')], ['server']);
    gulp.watch(path.join(__dirname, '/app.js'), ['server']);
});

// ---------------- Runnable Gulp Tasks ---------------- //
gulp.task('marbles_local', ['env_local', 'watch-sass', 'watch-server', 'server']);    //run with command `gulp marbles_local` for a local network

// Local Fabric via Docker Compose
gulp.task('env_local', function () {
    env['creds_filename'] = 'marbles_local.json';
}); 
```

### 启动应用

进入 scripts 目录，启动网络

```js
$ cd ~/kevin-marbles/scripts
$ ./start.sh 
```

安装、实例化链码(utils/fc_wrangler/index.js 有问题，须参考源代码)

```js
$ node install_chaincode.js
$ node instantiate_chaincode.js 
```

使用以下命令启动应用：

```js
$ cd $HOME/kevin-marbles
$ gulp marbles_local 
```