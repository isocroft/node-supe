<div style="font-size: 3em; font-weight: 900; text-align: center;">Supe</div>
<div style="font-size: 1.75em; color: #333; text-align: center;">Flexible Framework for Fault-Tolerant Node.js Apps</div>
<div style="text-align: center; margin-top: 1em;">
  <img src="https://badge.fury.io/js/supe.svg" alt="npm">
  <img src="https://travis-ci.org/Akamaozu/node-supe.svg?branch=master" alt="Travis CI">
  <img src="https://coveralls.io/github/Akamaozu/node-supe?branch=master" alt="Code Coverage">
</div>

# Intro

## Why You Should Care About Fault Tolerance

Your coworkers, mentors and heroes write faulty code. **You** write faulty code.

JavaScript isn’t particularly forgiving when code fails, so your app rolls over and stays dead. While we have many tools for bringing a dead Node.js app back to life, we typically don’t structure them in ways that limit the problem's impact on the app.

**Fault Tolerance reduces the impact problems have on your app**. At worst you’ll have a bottleneck in a component, but it won’t bring everything else to a grinding halt, or worse yet, send it to the big farm in the cloud.

## Why You Should Try A Framework for Fault Tolerance

You definitely don’t need one but a framework makes it **really easy to start** making things more fault-tolerant while providing **a common platform for building, sharing ideas and tools** the same way Express.js does for writing web servers.

## How Supe Can Help

While making your app more fault-tolerance is very rewarding, it is a VERY HARD THING™.

- You need ways to communicate between formerly-cozy components.
- Each component now outputs data to a different stdout.
- Even though each component is still single-threaded, your overall application is now multi-threaded.

This is just the tip of the iceberg of concerns introduced by trying to be fault-tolerance.

Supe helps by providing good default solutions for these problems and allows you to modify (or overwrite) any particular one so you can implement the solution that works best for your specific challenge.

# Install
```js
npm install --save supe
```

# Getting Started

## Supervisors and Citizens

The first step to fault-tolerance is breaking your app into separate parts.

For example, instead of one file with multiple responsibilities like:

```js
// inside app.js
var http = express(),
    db = db_driver(),
    ws = websocket();
```

We're moving towards separate programs with individual responsibilities and the app becomes their manager / coordinator.

```js
// inside app.js
var http = supervisor.start( 'http', 'http.js' ),
    db = supervisor.start( 'db', 'db-driver.js'),
    ws = supervisor.start( 'ws', 'websocket.js');
```

Supe refers to the manager / coordinator as a supervisor; the components are called citizens.

## Setup Supervisor

### Create
```js
var supervisor = require('supe')();
```

### Supervise Citizens
```js
// supervise server.js
supervisor.register( 'server', 'server.js' );
supervisor.start( 'server ');

// or
supervisor.start( 'server', 'server.js' );

// supervisor will restart server.js whenever it crashes
```

## Basic Config

### Set Citizen Overcrash

NOTE: Supe will stop reviving a citizen after it crashes excessively.

```js
var supervisor = require('supe')(),
    server = supervisor.start( 'server', 'server.js', { retries: 3, duration: 3 });

// if server.js crashes more than three times in three minutes, supe considers it overcrashed
```

### Set Default Overcrash

```js
var supervisor = require('supe')({ retries: 3, duration: 3 }),
    server = supervisor.start( 'server', 'server.js' ),
    worker = supervisor.start( 'worker', 'worker.js' );

// all supervised scripts will use the default overcrash thresholds
// individual overcrash thresholds override defaults

var worker2 = supervisor.start( 'worker2', 'worker.js', { retries: 20 }),
    worker3 = supervisor.start( 'worker3', 'worker.js', { duration: 1 });
```

## Advanced Use

### Noticeboard

Supe uses [cjs-noticeboard](https://www.npmjs.com/package/cjs-noticeboard "cjs-noticeboard on npm") internally to coordinate its moving parts. 

Watch for these internal notices using the exposed noticeboard instance.

```js
// log individual citizen's output
  supervisor.noticeboard.watch( 'worker2-output', 'pipe-to-console', function( msg ){

    var details = msg.notice;
    console.log( '[worker2] ' + details.output );
  });

// log all citizen errors
  supervisor.noticeboard.watch( 'citizen-error', 'pipe-to-console', function( msg ){

    var details = msg.notice,
        error = details.error,
        citizen = details.name;

    console.log( '['+ citizen +'][error] ' + details.error );
  });

// log shutdowns
  supervisor.noticeboard.watch( 'citizen-shutdown', 'log-shutdowns', function( msg ){
    
    var details = msg.notice;
    console.log( details.name + ' shut down' );
  });

// log overcrashes
  supervisor.noticeboard.watch( 'citizen-excessive-crash', 'log-excessive-crashes', function( msg ){
    
    var details = msg.notice;
    console.log( details.name + ' crashed more than permitted threshold' );
  });

// crash supervisor when worker two overcrashes
  supervisor.noticeboard.watch( 'citizen-excessive-crash', 'log-excessive-crashes', function( msg ){
    
    var details = msg.notice;
    if( details.name === 'worker2' ) throw new Error( 'worker 2 crashed excessively' );
  });
```

### Send Messages

Supervised scripts can send messages to supe.

```js  
// inside worker.js

var supe = require('supe');
if( supe.supervised ) supe.mail.send( 'hello supervisor' );
```

Supervised scripts can route messages through supe to other supervised scripts.

```js
// inside server.js

var supe = require('supe');
if( supe.supervised ) supe.mail.send({ to: 'worker3' }, 'hello worker three' );

// inside worker.js

if( supe.supervised ) supe.mail.receive( function( envelope, ack ){

  var sender = envelope.from,
      content = envelope.msg;

  console.log( 'message from ' + sender + ': ' + content );

  ack();
});
```

Messages don't have to be strings.

```js
// inside server.js

supe.mail.send({ key: 'concurrent-users', value: 9001 });
```

Supervised scripts are given one message at a time. 

No more messages will be sent til the current one is acknowledged ( ack() ).

Once you acknowledge a message, the next message is automatically sent.

```js
// worker one
  
for( var x = 1; x <= 100; x += 1 ){  
  supe.mail.send({ to: 'worker3' }, x );
}

// worker two
  
for( var x = 100; x >= 1; x -= 1 ){  
  supe.mail.send({ to: 'worker3' }, x );
}

// worker three
  
supe.mail.receive( function( envelope, ack ){

  var sender = envelope.from,
      content = envelope.msg;

  console.log( sender + ' says: ' + content );
  ack();
});

// worker1 says 1
// worker2 says 100
// worker1 says 2
// worker2 says 99
// worker1 says 3
// worker2 says 98
// worker1 says 4
// worker2 says 97
```

You can temporarily stop receiving messages.

None will be lost because supervisor stores them until supervised script is ready to receive again.

```js
// inside worker.js
  
supe.mail.receive( function( envelope, ack ){

  var sender = envelope.from,
      content = envelope.msg;

  supe.mail.pause();
  ack();

  // acknowledging this mail does not resume stream of inbound messages
  // resume mail stream in thirty minutes

  setTimeout( supe.mail.receive, 30 * 60 * 1000 );
});

```

Change the mail receive callback whenever you want.

```js

function callback1( envelope, ack ){

  var sender = envelope.from,
      content = envelope.msg;

  console.log( 'callback one used!' );

  supe.mail.receive( callback2 );
  ack();
}

function callback2( envelope, ack ){

  var sender = envelope.from,
      content = envelope.msg;

  console.log( 'callback two used!' );

  supe.mail.receive( callback1 );
  ack();
}
  
supe.mail.receive( callback1 );

// callback one used!
// callback two used!
// callback one used!
// callback two used!
// callback one used!
// callback two used!
```

Unacknowledged mails will be returned to the top of the mailbox when supervised script crashes or shuts down.

```js
// worker one
  
for( var x = 1; x <= 100; x += 1 ){  
  supe.mail.send({ to: 'worker2' }, x );
}

// worker two

console.log( 'worker2 started' );
  
supe.mail.receive( function( envelope, ack ){

  var sender = envelope.from,
      content = envelope.msg;

  console.log( sender + ' says: ' + content );
  throw new Error( 'chaos monkey strikes' );
});

// worker2 started
// worker1 says 1
// worker2 started
// worker1 says 1
// worker2 started
// worker1 says 1
```

### Supervised Supervisors

Supervised scripts can supervise other scripts.

```js
// inside supervisor.js

var supervisor = require('supe')();

supervisor.start( 'worker', 'worker.js' );

// inside worker.js

var supe = require('supe'),
    supervisor = supe();

supervisor.start( 'subworker', 'sub-worker.js' );

if( supe.supervised ) supe.mail.send( 'started a subworker' );
```