<template>
    <div id="wrapper">
        <div id="video-info"> Stream will start playing automatically<br />when it is live </div>

        <!-- WebRTC Player -->
        <video id="remote-video" controls playsinline></video>

        <div id="network-warning">Your connection isn't fast enough to play this stream!</div>
    </div>
</template>

<script>
import { WebRTCAdaptor } from './js/webrtc_adaptor.js'

export default {
    data() {
        return {
            iceConnected: false,
            token: undefined, // the token to play stream. It's mandatory if token security is enabled on server side.
            autoPlay: true,
            webRTCAdaptor: null,
            placeHolder: null,
            subscriberId: undefined,
            subscriberCode: undefined,
        }
    },
    props: {
        streamId: {
            type: String,
            required: true,
        },
        streamURL: {
            type: String,
            default: 'wss://stream.live.alogo.io:5443/WebRTCAppEE/websocket',
        },
    },
    methods: {
        hideWebRTCElements() {
            this.setWebRTCElementsVisibility(false)
        },
        setWebRTCElementsVisibility(show) {
            document.getElementById('remote-video').style.display = show == true ? 'block' : 'none'
        },
        setPlaceHolderVisibility(show) {
            this.placeHolder.style.display = show == true ? 'block' : 'none'
        },
        playWebRTCVideo() {
            this.setWebRTCElementsVisibility(true)

            document.getElementById('remote-video').muted = false

            if (this.autoPlay) {
                document
                    .getElementById('remote-video')
                    .play()
                    .catch(function(error) {
                        console.log('Error autoplay: ' + error)
                        console.log('User interaction needed to start playing')
                    })
            }
        },
        initializeWebRTCPlayer() {
            var pc_config = {
                iceServers: [
                    {
                        urls: 'stun:stun1.l.google.com:19302',
                    },
                ],
            }

            var sdpConstraints = {
                OfferToReceiveAudio: true,
                OfferToReceiveVideo: true,
            }
            var mediaConstraints = {
                video: false,
                audio: false,
            }

            this.iceConnected = false

            this.webRTCAdaptor = new WebRTCAdaptor({
                websocket_url: this.streamURL,
                mediaConstraints: mediaConstraints,
                peerconnection_config: pc_config,
                sdp_constraints: sdpConstraints,
                remoteVideoId: 'remote-video',
                isPlayMode: true,
                debug: true,
                callback: (info, description) => {
                    if (info == 'initialized') {
                        console.log('initialized')
                        this.iceConnected = false
                        this.webRTCAdaptor.getStreamInfo(this.streamId)
                    } else if (info == 'streamInformation') {
                        console.log('stream information')
                        this.webRTCAdaptor.play(this.streamId, this.token, this.subscriberId, this.subscriberCode)
                    } else if (info == 'play_started') {
                        //joined the stream
                        console.log('play started')
                        this.setPlaceHolderVisibility(false)
                        this.playWebRTCVideo()
                    } else if (info == 'play_finished') {
                        //leaved the stream
                        console.log('play finished')
                        this.hideWebRTCElements()
                        this.setPlaceHolderVisibility(true)
                        //if play_finished event is received, it has two meanings
                        //1. stream is really finished
                        //2. ice connection cannot be established and server reports play_finished event
                        //check that publish may start again
                        if (this.iceConnected) {
                            //webrtc connection was successful and try to play again with webrtc
                            setTimeout(function() {
                                this.webRTCAdaptor.getStreamInfo(this.streamId)
                            }, 3000)
                        } else alert('Error in play_finished step')
                    } else if (info == 'closed') {
                        //console.log("Connection closed");
                        if (typeof description != 'undefined') {
                            console.log('Connecton closed: ' + JSON.stringify(description))
                        }
                    } else if (info == 'bitrateMeasurement') {
                        console.debug(description)
                        if (description.audioBitrate + description.videoBitrate > description.targetBitrate) {
                            document.getElementById('network-warning').style.display = 'block'
                            setTimeout(function() {
                                document.getElementById('network-warning').style.display = 'none'
                            }, 3000)
                        }
                    } else if (info == 'ice_connection_state_changed') {
                        console.debug('ice connection state changed to ' + description.state)
                        if (description.state == 'connected' || description.state == 'completed') {
                            //it means the ice connection has been established
                            this.iceConnected = true
                        }
                    } else if (info == 'resolutionChangeInfo') {
                        console.log('Resolution is changed to ' + description['streamHeight'])
                        let getVideo = document.getElementById('remote-video')
                        getVideo.pause()
                        setTimeout(function() {
                            getVideo.play()
                        }, 2000)
                    }
                },
                callbackError: function(error) {
                    // Some of the possible errors: NotFoundError, SecurityError, PermissionDeniedError
                    console.log('error callback: ' + JSON.stringify(error))
                },
            })
        },
        main() {
            this.initializeWebRTCPlayer()
        },
    },
    mounted() {
        this.placeHolder = document.getElementById('video-info')

        this.main()
    },
}
</script>

<style lang="css">
#wrapper {
    position: relative;
    overflow: hidden;
    min-height: 350px;
}

#video-info {
    color: inherit;
    position: absolute;
    top: 30%;
    left: 50%;
    text-align:center;
    font-size: inherit;
    font-family: inherit;
    line-height: 1.4;
    -webkit-transform: translate(-50%, -50%);
    transform: translate(-50%, -50%);
}

#remote-video {
    position: absolute;
    display: none;
    width: 100%;
    height: 100%;
}

#network-warning {
    color: #856404;
    padding: 1em;
    margin: 1em;
    background-color: #fff3cd;
    border-color: #ffeeba;
    border-radius: .25em;
    display: none;
}
</style>
