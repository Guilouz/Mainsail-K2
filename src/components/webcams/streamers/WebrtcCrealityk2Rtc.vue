<template>
    <div class="position-relative d-flex">
        <video
            v-show="status === 'connected'"
            ref="stream"
            class="webcamStream"
            :style="webcamStyle"
            autoplay
            muted
            playsinline />
        <webcam-nozzle-crosshair v-if="nozzleCrosshair" :webcam="camSettings" />
        <v-row v-if="status !== 'connected'">
            <v-col class="_webcam_webrtc_output text-center d-flex flex-column justify-center align-center">
                <v-progress-circular v-if="status === 'connecting'" indeterminate color="primary" class="mb-3" />
                <span class="mt-3">{{ capitalize(status) }}</span>
            </v-col>
        </v-row>
    </div>
</template>

<script lang="ts">
import { Component, Mixins, Prop, Ref, Watch } from 'vue-property-decorator'
import BaseMixin from '@/components/mixins/base'
import { GuiWebcamStateWebcam } from '@/store/gui/webcams/types'
import WebcamMixin from '@/components/mixins/webcam'
import { capitalize } from '@/plugins/helpers'
import WebcamNozzleCrosshair from '@/components/webcams/WebcamNozzleCrosshair.vue'

@Component({
    components: { WebcamNozzleCrosshair },
})
export default class WebrtcCrealityk2Rtc extends Mixins(BaseMixin, WebcamMixin) {
    capitalize = capitalize

    pc: RTCPeerConnection | null = null
    useStun = false
    aspectRatio: null | number = null
    status: string = 'connecting'
    restartTimer: number | null = null

    @Prop({ required: true }) readonly camSettings!: GuiWebcamStateWebcam
    @Prop({ default: null }) declare readonly printerUrl: string | null
    @Prop({ type: String, default: null }) readonly page!: string | null
    @Ref() declare stream: HTMLVideoElement

    get url() {
        return this.convertUrl(this.camSettings?.stream_url, this.printerUrl)
    }

    get webcamStyle() {
        const output = {
            transform: this.generateTransform(
                this.camSettings.flip_horizontal ?? false,
                this.camSettings.flip_vertical ?? false,
                this.camSettings.rotation ?? 0
            ),
            aspectRatio: 16 / 9,
        }

        if (this.aspectRatio) output.aspectRatio = this.aspectRatio

        return output
    }

    get nozzleCrosshair() {
        return this.camSettings.extra_data?.nozzleCrosshair ?? false
    }

    get expanded(): boolean {
        if (this.page !== 'dashboard') return true

        return this.$store.getters['gui/getPanelExpand']('webcam-panel', this.viewport) ?? false
    }

    // start or stop the video when the expanded state changes
    @Watch('expanded', { immediate: true })
    expandChanged(newExpanded: boolean): void {
        if (!newExpanded) {
            this.terminate()
            return
        }

        this.start()
    }

    // This WebRTC signaling pattern is designed for camera-streamer, a common webcam server the supports WebRTC.
    async start() {
        if (this.restartTimer) {
            this.log('Clearing restart timer before starting stream')
            window.clearTimeout(this.restartTimer)
        }

        if (!this.expanded) {
            this.log('Not expanded, not starting stream')
            return
        }

        this.log(`Requesting ICE servers from ${this.url}`)

        try {
          this.pc?.close()

          const iceServers = [
            { urls: 'stun:stun.l.google.com:19302' }
          ]

          try {
            //const rtcSessionDescriptionInit = await response.json() as RTCSessionDescriptionInit

            //this.remoteId = ('id' in rtcSessionDescriptionInit && typeof rtcSessionDescriptionInit.id === 'string')
            //  ? rtcSessionDescriptionInit.id
            //  : null

            const config: RTCConfigurationWithSdpSemantics = {
              sdpSemantics: 'unified-plan'
            }

            config.iceServers = iceServers

            const pc = this.pc = new RTCPeerConnection(config)

            //pc.onicecandidate = event => {
            //  if(event.candidate === null) sendOfferToCall(pc.localDescription.sdp)
            //};

            pc.onconnectionstatechange = () => this.onConnectionStateChange()
            pc.onicecandidate = async event => {
              if(event.candidate === null) {
                const response = await fetch(`${this.url}call/webrtc_local`, {
                  body: btoa(JSON.stringify({
                    type: "offer",
                    sdp: pc.localDescription?.sdp
                  })),
                  headers: {
                    'Content-Type': 'plain/text'
                  },
                  method: 'POST'
                })


                const res = await response.text()

                const jsonRes =  JSON.parse(atob(res))
                if(jsonRes.type == 'answer') {
                  pc.setRemoteDescription(new RTCSessionDescription(jsonRes));
                }
              }
            }

            pc.addTransceiver('video', {
              direction: 'recvonly'
            })

            pc.ontrack = (event: RTCTrackEvent) => {
              if (event.track.kind === 'video') {
                this.stream.srcObject = event.streams[0]
              }
            }
            await pc.createOffer().then(d => pc.setLocalDescription(d));
            console.log("Set Creality K2 Camera")
          } catch (e) {
            this.log('Failed to start stream', e)
          }
        } finally {}
    }

    onConnectionStateChange() {
        this.status = this.pc?.connectionState ?? 'connecting'

        this.log(`State: ${this.status}`)

        if (['failed', 'disconnected'].includes(this.status)) {
            this.restartStream(5000)
        }
    }

    onTrack(e: RTCTrackEvent) {
        if (e.track.kind !== 'video') return

        this.stream.srcObject = e.streams[0]
    }

    log(msg: string, obj?: any) {
        const message = `[WebRTC creality k2] ${msg}`
        if (obj) {
            window.console.log(message, obj)
            return
        }

        window.console.log(message)
    }

    beforeDestroy() {
        this.terminate()
        if (this.restartTimer) window.clearTimeout(this.restartTimer)
    }

    terminate() {
        this.log('Terminating stream')
        this.pc?.close()
    }

    restartStream(delay = 500) {
        this.terminate()

        if (this.restartTimer) return

        this.restartTimer = window.setTimeout(async () => {
            this.restartTimer = null
            await this.start()
        }, delay)
    }

    @Watch('url')
    changedUrl() {
        this.restartStream()
    }
}
</script>

<style scoped>
.webcamStream {
    width: 100%;
}

._webcam_webrtc_output {
    aspect-ratio: calc(3 / 2);
}

video {
    width: 100%;
}
</style>