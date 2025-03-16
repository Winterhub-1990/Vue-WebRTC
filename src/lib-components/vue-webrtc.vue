<template>
    <div class="video-list">
        <div v-for="item in videoList"
             :key="item.id"
             class="video-item">
            <video controls autoplay playsinline ref="videos" :height="cameraHeight" :muted="item.muted" :id="item.id"></video>
        </div>
    </div>
</template>

<script>
import { defineComponent, ref, onMounted, onBeforeUnmount } from 'vue';
import { io } from "socket.io-client";
//import SimpleSignalClient from 'simple-signal-client'; //Consider deprecating and implement yourself

// Helper function for generating random room IDs
function generateRoomId() {
    const crypto = window.crypto || window.msCrypto; // For cross-browser compatibility
    const buffer = new Uint8Array(20);
    crypto.getRandomValues(buffer);
    return Array.from(buffer, (byte) => byte.toString(16).padStart(2, '0')).join('');
}

export default defineComponent({
    name: 'vue-webrtc',
    props: {
        roomId: {
            type: String,
            default() {
                return generateRoomId(); // Generate a unique room ID by default
            }
        },
        socketURL: {
            type: String,
            required: true, // Make this required and enforce HTTPS in production
            validator: (url) => url.startsWith('https://') // Enforce HTTPS
        },
        cameraHeight: {
            type: [Number, String],
            default: 160
        },
        autoplay: {
            type: Boolean,
            default: true
        },
        screenshotFormat: {
            type: String,
            default: 'image/jpeg'
        },
        enableAudio: {
            type: Boolean,
            default: true
        },
        enableVideo: {
            type: Boolean,
            default: true
        },
        enableLogs: {
            type: Boolean,
            default: false
        },
        peerOptions: {
            type: Object,
            default: () => ({}) // Return a new object
        },
        //REMOVE rejectUnauthorized - this is a HUGE SECURITY RISK and should never be used in production
        ioOptions: {
            type: Object,
            default: () => {
                return {transports: ['polling', 'websocket'] }; // Removed rejectUnauthorized
            }
        },
        deviceId: {
            type: String,
            default: null
        },
        //ADD: JWT Token Prop for Authentication
        jwtToken: {
            type: String,
            default: null //Require a JWT token for secured access
        }
    },
    setup(props, { emit }) {
        const signalClient = ref(null);
        const videoList = ref([]);
        const socket = ref(null);
        const videos = ref([]);

        // Centralized logging function
        const log = (message, data) => {
            if (props.enableLogs) {
                console.log(message, data || '');
            }
        };

        //Joining Room Function
        const joinRoom = async () => {
            log('Joining room...');

            //Authentication
            if (!props.jwtToken) {
                console.error("JWT Token is Required");
                return;
            }

            socket.value = io(props.socketURL, {
                ...props.ioOptions,
                auth: {
                    token: props.jwtToken
                }
            });

            //Error Handling
            socket.value.on('connect_error', (err) => {
                console.error('Socket connection error:', err);
                emit('socket-error', err); //Emit error for parent component to handle
            });

            //Simple-signal-client
            //try {
            //    signalClient.value = new SimpleSignalClient(socket.value);
            //} catch (error) {
            //    console.error("Signal client initialization failed: ", error);
            //    return;
            //}

            const constraints = {
                video: props.enableVideo,
                audio: props.enableAudio,
            };

            if (props.deviceId && props.enableVideo) {
                constraints.video = { deviceId: { exact: props.deviceId } };
            }

            let localStream = null;
            try {
                localStream = await navigator.mediaDevices.getUserMedia(constraints);
                log('getUserMedia success', localStream);
            } catch (error) {
                console.error("getUserMedia failed:", error);
                emit('media-error', error);
                return;
            }

            joinedRoom(localStream, true);

            //New Way of Handeling Signaling
            socket.value.on("discover", (discoveryData) => {
                log('discovered', discoveryData);

                discoveryData.peers.forEach(peerID => {
                    if (peerID !== socket.value.id) {
                        connectToPeer(peerID);
                    }
                });

                emit('opened-room', props.roomId);
            });

            socket.value.on("request", async (request) => {
                log("request", request);
                try {
                    // Accept peer request and add local stream to the peer connection
                    const peer = new RTCPeerConnection(props.peerOptions);

                    // Add local stream to peer
                    localStream.getTracks().forEach(track => peer.addTrack(track, localStream));

                    // Set up event handlers for the peer connection
                    peer.ontrack = (event) => {
                        log("Received remote stream");
                        joinedRoom(event.streams[0], false);
                    };

                    peer.onicecandidate = (event) => {
                        if (event.candidate) {
                            socket.value.emit("ice-candidate", {
                                target: request.socketId,
                                candidate: event.candidate
                            });
                        }
                    };

                    peer.oniceconnectionstatechange = () => {
                        if (peer.iceConnectionState === "disconnected") {
                            console.log("Peer disconnected");
                        }
                    };

                    // Create offer and send it to the peer
                    const offer = await peer.createOffer();
                    await peer.setLocalDescription(offer);

                    socket.value.emit("offer", {
                        target: request.socketId,
                        offer: offer
                    });

                    // Handle incoming answer from the peer
                    socket.value.on("answer", async (data) => {
                        if (data.socketId === request.socketId) {
                            try {
                                await peer.setRemoteDescription(data.answer);
                            } catch (error) {
                                console.error("Error setting remote description: ", error);
                            }
                        }
                    });

                    // Handle incoming ICE candidates from the peer
                    socket.value.on("ice-candidate", async (data) => {
                        if (data.socketId === request.socketId && data.candidate) {
                            try {
                                await peer.addIceCandidate(data.candidate);
                            } catch (error) {
                                console.error("Error adding ICE candidate: ", error);
                            }
                        }
                    });

                } catch (error) {
                    console.error("Error accepting peer request:", error);
                }

            });

            const connectToPeer = async (peerID) => {
                if (peerID === socket.value.id) return;
                log(`Connecting to peer: ${peerID}`);

                try {
                    // Create a new RTCPeerConnection
                    const peerConnection = new RTCPeerConnection(props.peerOptions);

                    // Add local stream to peer connection
                    localStream.getTracks().forEach(track => peerConnection.addTrack(track, localStream));

                    peerConnection.ontrack = (event) => {
                        log("Received remote stream");
                        joinedRoom(event.streams[0], false);
                    };

                    peerConnection.onicecandidate = (event) => {
                        if (event.candidate) {
                            socket.value.emit("ice-candidate", {
                                target: peerID,
                                candidate: event.candidate
                            });
                        }
                    };

                    socket.value.on("ice-candidate", async (data) => {
                        if (data.socketId === peerID && data.candidate) {
                            try {
                                await peerConnection.addIceCandidate(data.candidate);
                            } catch (error) {
                                console.error("Error adding ICE candidate: ", error);
                            }
                        }
                    });

                    // Create offer
                    const offer = await peerConnection.createOffer();
                    await peerConnection.setLocalDescription(offer);

                    socket.value.emit("offer", {
                        target: peerID,
                        offer: offer
                    });

                    socket.value.on("answer", async (data) => {
                        if (data.socketId === peerID) {
                            try {
                                await peerConnection.setRemoteDescription(data.answer);
                            } catch (error) {
                                console.error("Error setting remote description: ", error);
                            }
                        }
                    });

                    socket.value.on("ice-candidate", async (data) => {
                        if (data.socketId === peerID && data.candidate) {
                            try {
                                await peerConnection.addIceCandidate(data.candidate);
                            } catch (error) {
                                console.error("Error adding ICE candidate: ", error);
                            }
                        }
                    });

                    peerConnection.oniceconnectionstatechange = () => {
                        if (peerConnection.iceConnectionState === "disconnected") {
                            console.log("Peer disconnected");
                        }
                    };

                    peerConnection.onerror = (error) => {
                        console.error("Peer connection error: ", error);
                    }

                } catch (error) {
                    console.error("Error connecting to peer:", error);
                }
            };

            socket.value.emit("discover", props.roomId);

        };

        const onPeer = (peer, localStream) => {
            log('onPeer');
            peer.addStream(localStream);

            peer.on('stream', (remoteStream) => {
                joinedRoom(remoteStream, false);

                peer.on('close', () => {
                    videoList.value = videoList.value.filter(item => item.id !== remoteStream.id);
                    emit('left-room', remoteStream.id);
                });

                peer.on('error', (err) => {
                    log('peer error ', err);
                    emit('peer-error', err);
                });
            });
        };

        const joinedRoom = (stream, isLocal) => {
            const found = videoList.value.find(video => video.id === stream.id);

            if (!found) {
                const video = {
                    id: stream.id,
                    muted: isLocal,
                    stream: stream,
                    isLocal: isLocal
                };

                videoList.value.push(video);
            }

            // Using nextTick to ensure videos are rendered before assigning srcObject
            nextTick(() => {
                videos.value.forEach(videoElement => {
                    if (videoElement.id === stream.id) {
                        videoElement.srcObject = stream;
                    }
                });
            });

            emit('joined-room', stream.id);
        };

        const leaveRoom = () => {
            log('Leaving room');

            videoList.value.forEach(v => v.stream.getTracks().forEach(t => t.stop()));
            videoList.value = [];

            //if (signalClient.value) {
            //    signalClient.value.peers().forEach(peer => peer.removeAllListeners());
            //    signalClient.value.destroy();
            //    signalClient.value = null;
            //}

            if (socket.value) {
                socket.value.disconnect();
                socket.value = null;
            }

            emit('left-room', props.roomId);
        };

        const capture = () => {
            return getCanvas().toDataURL(props.screenshotFormat);
        };

        const getCanvas = () => {
            const video = videos.value[0];

            if (!video) {
                console.warn("No video element found for capturing.");
                return null;
            }

            const canvas = document.createElement('canvas');
            canvas.height = video.clientHeight;
            canvas.width = video.clientWidth;
            const ctx = canvas.getContext('2d');
            ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
            return canvas;
        };

        const shareScreen = async () => {
            log("Starting screen share...");
            if (!navigator.mediaDevices) {
                console.error('HTTPS is required to load cameras for screen sharing.');
                emit('media-error', 'HTTPS is required for screen sharing');
                return;
            }

            try {
                const screenStream = await navigator.mediaDevices.getDisplayMedia({ video: true, audio: false });
                joinedRoom(screenStream, true);
                emit('share-started', screenStream.id);

                //if (signalClient.value) {
                //    signalClient.value.peers().forEach(p => onPeer(p, screenStream));
                //}
            } catch (error) {
                console.error('Error sharing screen:', error);
                emit('media-error', error);
            }
        };

        onMounted(() => {
            joinRoom();
        });

        onBeforeUnmount(() => {
            leaveRoom();
        });

        return {
            signalClient,
            videoList,
            socket,
            videos,
            joinRoom,
            leaveRoom,
            capture,
            getCanvas,
            shareScreen,
            log
        };
    }
});
</script>

<style scoped>
.video-list {
    background: whitesmoke;
    height: auto;
    display: flex;
    flex-direction: row;
    justify-content: center;
    flex-wrap: wrap;
}

.video-list div {
    padding: 0;
}

.video-item {
    background: #c5c4c4;
    display: inline-block;
}
</style>
