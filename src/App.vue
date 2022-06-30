<template>
  <div>
    <p>
      Your peer ID: {{ myPeerId }}
    </p>
    <p>
      Other peer ID:
      <input type="text" style="width: 420px" v-model="otherPeerId" />
      <button @click="findOtherPeer">
        Find
      </button>
    </p>
    <div v-if="otherPeerMultiaddrs.length > 0 && otherPeerProtocols.length > 0">
      <p>
        Other peer protocol: {{ otherPeerProtocol }}
        <button @click="dialProtocol">
          Dial
        </button>
      </p>
    </div>
    <p v-if="remotePeerId">
      Remote peer connected: {{ remotePeerId.toString() }}
    </p>
    <p v-for="(msg, idx) in messages" :key="'msg_' + idx">
      {{ msg }}
    </p>
    <div v-if="chatQueue">
      <input type="text" style="width: 600px" v-model="chatMessage" />
      <button @click="sendMessage">
        Send message
      </button>
    </div>
  </div>
</template>

<script lang="ts">
import KadDHT from "libp2p-kad-dht";
import Libp2p from "libp2p";
import Mplex from "libp2p-mplex";
import PeerId from "peer-id";
import pushable from "it-pushable";
import pipe from "it-pipe";
import WebRTCStar from "libp2p-webrtc-star";
import Websockets from "libp2p-websockets";
import { NOISE } from "libp2p-noise";
import { array2str, str2array } from "./utils";
import { Component, Vue } from "vue-facing-decorator";

const chatProtocol = "/chat/0.1.0";

@Component
export default class App extends Vue {
  libp2p: Libp2p;
  myPeerId: string = "";
  otherPeerId: string = "";
  otherPeerMultiaddrs: any[] = [];
  otherPeerProtocols: string[] = [];
  otherPeerMultiaddr: string = "";
  otherPeerProtocol: string = "";
  remotePeerId: any = "";
  chatMessage: string = "";
  messages: string[] = [];
  chatQueue: any = false;

  async init() {
    // Use Libp2p.create instead of createLibp2p to avoid import errors 
    this.libp2p = await Libp2p.create({
      addresses: {
        // Listen through relay servers
        // Cross-subnet P2P direct connection not supported due to NAT/firewalls 
        listen: [
          "/dns4/wrtc-star1.par.dwebops.pub/tcp/443/wss/p2p-webrtc-star",
          "/dns4/wrtc-star2.sjc.dwebops.pub/tcp/443/wss/p2p-webrtc-star",
        ],
      },
      modules: {
        transport: [Websockets, WebRTCStar],
        connEncryption: [NOISE],
        streamMuxer: [Mplex],
        dht: KadDHT,
      },
      config: {
        dht: {
          enabled: true,
        },
      },
    });

    // window.libp2p = this.libp2p;
    await this.libp2p.start();

    this.myPeerId = this.libp2p.peerId.toB58String();
    this.libp2p.handle(chatProtocol, ({ connection, stream, protocol }) => {
      this.remotePeerId = connection.remoteAddr.getPeerId();
      // Init an async data stream on the receiver side
      pipe(
        stream,
        (source) => {
          return (async function* () {
            for await (const buf of source) yield array2str(buf.slice());
          })();
        },
        async (source) => {
          for await (const msg of source) {
            this.messages.push("> " + msg);
          }
        }
      );
    });
  }

  mounted() {
    this.init();
  }

  // Wrapper function to store some infos.
  async findOtherPeer() {
    let peerId = PeerId.parse(this.otherPeerId);
    let result = await this.libp2p.peerRouting.findPeer(peerId);
    this.otherPeerMultiaddrs = result.multiaddrs;
    this.otherPeerProtocols = this.libp2p.peerStore.protoBook.get(peerId);
    this.otherPeerMultiaddr = this.otherPeerMultiaddrs[0];
    this.otherPeerProtocol = chatProtocol; // should be the last protocol (here for conveninence)
  }

  // Wrapper function to init a async data stream on the sender side.
  async dialProtocol() {
    let peerId = PeerId.parse(this.otherPeerId);
    const { stream, protocol } = await this.libp2p.dialProtocol(
      peerId,
      chatProtocol
    );
    this.chatQueue = pushable();
    pipe(
      this.chatQueue,
      (source) => {
        return (async function* () {
          for await (const msg of source) yield str2array(msg);
        })();
      },
      stream
    );
  }

  sendMessage() {
    this.messages.push("< " + this.chatMessage);
    this.chatMessage = "";
    this.chatQueue.push(this.chatMessage);
  }
}
</script>

