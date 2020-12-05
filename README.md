# volumio-api-examples

NodeJS:

I’ve more or less finished configuring Volumio 2 on a Raspberry Pi 3. I’ve added a generic IR receiver to control it with a remote, by mapping some buttons to specific actions. The “buttons” part is documented in the previous link and other places. The “actions” part needed some programming to get them do exactly what I wanted. I thought I’d share the scripts I have, in case someone finds them useful.

These are Node.js scripts intended to be run from the command line, so we need the “node” command and possibly some other packages/modules. I don’t remember exactly what I needed to install, but I think it was easy to find out when I tried running this kind of scripts. I don’t claim these are the best possible scripts, but they seem to do their job for me.

Toggle play/pause: If the current mode is “play”, it pauses, otherwise it plays.

[code]var io = require(‘socket.io-client’);
var socket = io.connect(‘http://localhost:3000’);
socket.emit(‘getState’, ‘’);
socket.on(‘pushState’, function(data) {
switch (data.status) {
case ‘play’:
socket.emit(‘pause’);
break;
default:
socket.emit(‘play’);
break;
}
socket.removeAllListeners(‘pushState’);
socket.on(‘pushState’, function() { socket.disconnect(); } );
} );

setTimeout(function() { socket.disconnect(); }, 3000);[/code]

Change volume: Takes one argument (i.e., run with "node "). A number sets the volume to that number, “+” and “-” increase or decrease the volume by 5%, “mute” toggles mute on/off.

[code]var io = require(‘socket.io-client’);
var socket = io.connect(‘http://localhost:3000’);
socket.emit(‘getState’, ‘’);
socket.on(‘pushState’, function(data) {
switch (process.argv[2]) {
case ‘+’:
socket.emit(‘volume’, Math.min(data.volume+5, 100));
break;
case ‘-’:
socket.emit(‘volume’, Math.max(data.volume-5, 0));
break;
case ‘mute’:
switch (data.mute) {
case true:
socket.emit(‘unmute’, true);
break;
case false:
socket.emit(‘mute’, true);
break;
}
break;
default:
socket.emit(‘volume’, Math.max(Math.min(Number(process.argv[2]), 100), 0));
}
process.exit();
} );

setTimeout(function() { socket.disconnect(); }, 3000);[/code]

Play custom webradio: Clears the playlist and plays the radio given as the argument

[code]var io = require(‘socket.io-client’);
var socket = io.connect(‘http://localhost:3000’);
socket.emit(‘clearQueue’);
var fs = require(‘fs’);
var content = fs.readFileSync(’/data/favourites/my-web-radio’);
var webradios = JSON.parse(content);
var radio = webradios.find(o => o.name === process.argv[2]);
if (radio == null) {
socket.disconnect();
} else {
socket.emit(‘addPlay’, {‘service’:radio.service,‘title’:radio.name,‘uri’:radio.uri});
socket.on(‘pushState’, function() { socket.disconnect(); } );
}

setTimeout(function() { socket.disconnect(); }, 3000);[/code]

Queue a random album: Clears the playlist and puts a random album from the library. (I use random.org to provide random numbers, use your own apiKey. It could be easily adapted to use a local random number generator.)

[code]var RandomOrg = require(‘random-org’);
var io = require(‘socket.io-client’);
var random = new RandomOrg({‘apiKey’:’#########################’});
var socket = io.connect(‘http://localhost:3000’);
socket.emit(‘browseLibrary’, {‘uri’:‘albums://’});
socket.on(‘pushBrowseLibrary’,function(data) {
var list = data.navigation.lists[0].items;
random.generateIntegers({‘min’:0, ‘max’:list.length-1, ‘n’:1})
.then(function(result) {
select = list[result.random.data[0]];
socket.emit(‘clearQueue’);
socket.emit(‘addToQueue’, {‘uri’:select.uri});
});
}
);
socket.on(‘pushQueue’, function(data) { if (data.length > 0) { socket.disconnect(); } } );

setTimeout(function() { socket.disconnect(); }, 5000);[/code]

Once I have these scripts I can assign them to the remote buttons in .lircrc
