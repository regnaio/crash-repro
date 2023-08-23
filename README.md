# Steps to reproduce a [crash in `node-datachannel`](https://github.com/murat-dogan/node-datachannel/issues/176)

## 1. Open a new Chrome browser window and open its DevTools Console. Please note that opening a new Chrome window seems to increase the probability of reproducing the crash, though this may not be true.

## 2. Run the following snippet in a Chrome browser DevTools Console:
```js
const iceServers = [
	{ urls: 'stun:stun1.l.google.com:19302' },
	{ urls: 'stun:stun2.l.google.com:19302' },
	{ urls: 'stun:stun3.l.google.com:19302' },
	{ urls: 'stun:stun4.l.google.com:19302' },
];

const peerConnection = new RTCPeerConnection({
	iceServers,
	iceTransportPolicy: 'all',
});

let dataChannel;

peerConnection.onconnectionstatechange = ev => {
	const { connectionState } = peerConnection;
	console.log(`peerConnection.onconnectionstatechange(): connectionState: ${connectionState}`);
};
peerConnection.ondatachannel = ev => {
	console.log('peerConnection.ondatachannel()');

	dataChannel = ev.channel;
	dataChannel.onopen = ev => {
		console.log('dataChannel.onopen()');

		peerConnection.close();
	};
	dataChannel.onmessage = ev => {
		console.log(`dataChannel.onmessage(): ev.data: ${ev.data}`);
	};
	dataChannel.onclose = ev => {
		console.log(`dataChannel.onclose()`);
	};
	dataChannel.onerror = ev => {
		console.log(`dataChannel.onerror(): ev.data: ${ev.data}`);
	};
};
peerConnection.onicecandidate = ev => {
	const { candidate } = ev;
	console.log('peerConnection.onicecandidate(): candidate:', candidate);

	console.log(`COPY LAST ONE PRINTED AS THE FINAL ANSWER: ${JSON.stringify(peerConnection.localDescription)}\n`);
};
peerConnection.oniceconnectionstatechange = ev => {
	const { iceConnectionState } = peerConnection;
	console.log(`peerConnection.oniceconnectionstatechange(): iceConnectionState: ${iceConnectionState}`);
};
peerConnection.onicegatheringstatechange = ev => {
	const { iceGatheringState } = peerConnection;
	console.log(`peerConnection.onicegatheringstatechange(): iceGatheringState: ${iceGatheringState}`);
};
peerConnection.onsignalingstatechange = ev => {
	const { signalingState } = peerConnection;
	console.log(`peerConnection.onsignalingstatechange(): signalingState: ${signalingState}`);
};
```

## 3. Run the following snippet in a Node REPL:

```js
const { default: nodeDataChannel } = await import('node-datachannel');

const { performance } = await import('node:perf_hooks');

const iceServers = [
	{
		hostname: 'stun:stun1.l.google.com',
		port: 19302,
	},
	{
		hostname: 'stun:stun2.l.google.com',
		port: 19302,
	},
	{
		hostname: 'stun:stun3.l.google.com',
		port: 19302,
	},
	{
		hostname: 'stun:stun4.l.google.com',
		port: 19302,
	},
];

const peerConnection = new nodeDataChannel.PeerConnection('', {
	iceServers,
	iceTransportPolicy: 'all',
});
peerConnection.onStateChange(state => {
	console.log(`peerConnection.onStateChange(): state: ${state}`);
});
peerConnection.onSignalingStateChange(state => {
	console.log(`peerConnection.onSignalingStateChange(): state: ${state}`);
});
peerConnection.onGatheringStateChange(state => {
	console.log(`peerConnection.onGatheringStateChange(): state: ${state}`);
});
peerConnection.onLocalDescription((sdp, type) => {
	console.log(`peerConnection.onLocalDescription(): sdp: ${sdp}, type: ${type}`);
});
peerConnection.onLocalCandidate((candidate, mid) => {
	console.log(`peerConnection.onLocalCandidate(): candidate: ${candidate}, mid: ${mid}`);

	console.log(`COPY LAST ONE PRINTED AS THE FINAL OFFER: ${JSON.stringify(peerConnection.localDescription())}\n`);
});

const dataChannel = peerConnection.createDataChannel('', {
	ordered: false,
	maxRetransmits: 0,
});
dataChannel.onOpen(() => {
	console.log('dataChannel.onOpen()');
});
dataChannel.onMessage(msg => {
	console.log('dataChannel.onMessage(): msg:', msg);
});
dataChannel.onClosed(() => {
	console.log('dataChannel.onClosed()');

	peerConnection.onStateChange(state => {});
	peerConnection.onSignalingStateChange(state => {});
	peerConnection.onGatheringStateChange(state => {});
	peerConnection.onLocalDescription((sdp, type) => {});
	peerConnection.onLocalCandidate((candidate, mid) => {});

	peerConnection.close();
});
dataChannel.onError(err => {
	console.log(`dataChannel.onError(): err: ${err}`);
});
```

## 4. In the Node REPL logs, copy the JSON stringified object after `COPY LAST ONE PRINTED AS THE FINAL OFFER: `. This is the final offer.

## 5. Paste the final offer into the `offer` variable below and run this snippet in the same Chrome DevTools Console:
```js
const offer = ; // <--- Paste final offer here
await peerConnection.setRemoteDescription(offer);

const answer = await peerConnection.createAnswer();
await peerConnection.setLocalDescription(answer);
```

## 6. In the Chrome DevTools Console logs, copy the JSON stringified object after `COPY LAST ONE PRINTED AS THE FINAL ANSWER: `. This is the final answer.

## 7. Paste the final answer into the `answer` variable below and run this snippet in the same Node REPL:
```js
const answer = ; // <--- Paste final answer here
const { sdp, type } = answer;
peerConnection.setRemoteDescription(sdp, type);
```

## 8. In the Node REPL logs, there's a probability of the following crash occuring:
```shell
FATAL ERROR: Error::New napi_get_last_error_info
 1: 0xb7a940 node::Abort() [node]
 2: 0xa8e72f node::FatalError(char const*, char const*) [node]
 3: 0xa8e738 node::OOMErrorHandler(char const*, bool) [node]
 4: 0xb40ac3 napi_fatal_error [node]
 5: 0x7fbd2ce055f0 Napi::CallbackScope::~CallbackScope() [<REPO_PATH>/node_modules/node-datachannel/build/Release/node_datachannel.node]
 6: 0x7fbd2ce05918 Napi::Error::New(napi_env__*) [<REPO_PATH>/node_modules/node-datachannel/build/Release/node_datachannel.node]
 7: 0x7fbd2cdeb5d3  [<REPO_PATH>/node_modules/node-datachannel/build/Release/node_datachannel.node]
 8: 0x7fbd2ce41255 ThreadSafeCallback::callbackFunc(Napi::Env, Napi::Function, Napi::Reference<Napi::Value>*, ThreadSafeCallback::CallbackData*) [<REPO_PATH>/node_modules/node-datachannel/build/Release/node_datachannel.node]
 9: 0xb4083c  [node]
10: 0x165f601 uv_run [node]
11: 0xabda6d node::SpinEventLoop(node::Environment*) [node]
12: 0xbc1164 node::NodeMainInstance::Run() [node]
13: 0xb35bc8 node::LoadSnapshotDataAndRun(node::SnapshotData const**, node::InitializationResult const*) [node]
14: 0xb3976f node::Start(int, char**) [node]
15: 0x7fbd34d00d0a __libc_start_main [/lib/x86_64-linux-gnu/libc.so.6]
16: 0xabbdee _start [node]
Aborted (core dumped)
```
