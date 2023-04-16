<script lang="ts">
	import { createLibp2p, type Libp2p } from "libp2p";
	import { webSockets } from "@libp2p/websockets";
	import { webRTCStar } from "@libp2p/webrtc-star";
	import { gossipsub } from "@chainsafe/libp2p-gossipsub";
	import { kadDHT } from "@libp2p/kad-dht";
	import { mplex } from "@libp2p/mplex";
	import { noise } from "@chainsafe/libp2p-noise";
	import { pubsubPeerDiscovery, TOPIC } from "@libp2p/pubsub-peer-discovery";
	import { multiaddr } from "@multiformats/multiaddr";
	import { onDestroy, onMount } from "svelte";

	const PUBSUB_TOPIC = "pubsub_chat_topic_here";

	let libp2p: Libp2p;

	let messages: string[] = [];
	let myMultiaddr = "";

	let multiaddrInput = "";
	let messageInput = "";

	onMount(async () => {
		let wrtcStar = webRTCStar();
		libp2p = await createLibp2p({
			addresses: {
				listen: [
					"/dns4/wrtc-star1.par.dwebops.pub/tcp/443/wss/p2p-webrtc-star",
					"/dns4/wrtc-star2.sjc.dwebops.pub/tcp/443/wss/p2p-webrtc-star",
				],
			},
			transports: [webSockets(), wrtcStar.transport],
			connectionEncryption: [noise()],
			streamMuxers: [mplex()],
			peerDiscovery: [
				pubsubPeerDiscovery(),
				// uncomment this for auto peer discovery
				// wrtcStar.discovery,
			],
			dht: kadDHT(),
			pubsub: gossipsub({
				allowPublishToZeroPeers: true,
			}),
		});

		await libp2p.start();

		myMultiaddr = libp2p.getMultiaddrs()[0].toString();

		libp2p.pubsub.subscribe(PUBSUB_TOPIC);
		libp2p.pubsub.addEventListener("message", ({ detail }) => {
			if (detail.topic == TOPIC) return;

			let text = new TextDecoder().decode(detail.data);

			messages = [...messages, text];
		});

		libp2p.addEventListener("peer:connect", ({ detail }) => {
			let peerId = detail.remotePeer.toString();

			messages = [...messages, `connected to ${peerId}`];
		});
	});

	onDestroy(async () => {
		await libp2p.stop();
	});

	function connectToMultiaddr() {
		libp2p.dial(multiaddr(multiaddrInput));
	}

	async function sendMessage() {
		let { recipients } = await libp2p.pubsub.publish(
			PUBSUB_TOPIC,
			new TextEncoder().encode(messageInput)
		);

		if (recipients.length != 0) messages = [...messages, messageInput];
	}
</script>

<button
	disabled={myMultiaddr == ""}
	on:click={() => navigator.clipboard.writeText(myMultiaddr)}
	>click to copy multiaddr</button
>

<label>
	multiaddr
	<input bind:value={multiaddrInput} type="text" />
	<button on:click={connectToMultiaddr}>connect</button>
</label>

<br />

<label>
	message
	<input bind:value={messageInput} type="text" />
	<button on:click={sendMessage}>send</button>
</label>

<ul>
	{#each messages as message}
		<li>{message}</li>
	{/each}
</ul>
