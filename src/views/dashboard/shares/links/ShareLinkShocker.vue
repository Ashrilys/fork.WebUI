<template>
    <b-container class="shocker">
        <b-row class="head">
            <b-col v-if="isOwn" cols="auto" class="pause-col" @click="togglePause">
                <span v-if="pausedOnShareLinkLevel" class="paused">
                    <i class="fa-solid fa-play"></i>
                </span>
                <span v-else>
                    <i class="fa-solid fa-pause"></i>
                </span>
            </b-col>
            <b-col cols="10" class="shocker-name-col" :class="isOwn ? '' : 'reduced-padding-left'">
                <p class="shocker-name">{{ shocker.name }}</p>
            </b-col>
            <b-col v-if="isOwn" cols="auto" class="elli" @click="ellipsis">
                <i class="fa-solid fa-ellipsis-vertical"></i>
            </b-col>
        </b-row>
        <div class="content-container">
            <div v-if="pauseReasonText !== null" class="paused-text width100">
                <b-container class="width100" style="text-align: center;">
                    <h2>Paused</h2>
                    <b-row @click="togglePause" style="cursor: pointer;">
                        <span v-if="pausedOnShareLinkLevel">
                            <i style="font-size: 38px;" class="fa-solid fa-play"></i>
                        </span>
                        <span v-else>
                            <i style="font-size: 38px;" class="fa-solid fa-pause"></i>
                        </span>
                    </b-row>
                    <h2>{{ pauseReasonText }}</h2>
                </b-container>
            </div>
            <div class="content-child" :class="pauseReasonText !== null ? 'paused' : ''">
                <b-row>
                    <b-container align-items="center" style="margin-top: 15px">
                        <b-row align-h="center">
                            <b-col md="auto" style="width: unset">
                                <round-slider v-model="shocker.state.intensity"
                                    :max="shocker.limits.intensity === null ? 100 : shocker.limits.intensity"
                                    pathColor="#1b1d1e" rangeColor="#e14a6d" start-angle="315" end-angle="+270" width="30"
                                    line-cap="round" radius="75" />

                                <p style="text-align: center;">Intensity</p>
                            </b-col>
                            <b-col md="auto" style="width: unset">
                                <round-slider v-model="shocker.state.duration"
                                    :max="shocker.limits.duration === null ? 30 : shocker.limits.duration / 1000"
                                    pathColor="#1b1d1e" rangeColor="#e14a6d" start-angle="315" end-angle="+270"
                                    line-cap="round" radius="75" width="30" min="0.3" step="0.1" />

                                <p style="text-align: center;">Duration</p>
                            </b-col>
                        </b-row>
                    </b-container>
                </b-row>
                <b-row align-h="center">
                    <b-col v-if="shocker.permissions.sound" cols="auto" md="auto">
                        <control-button style="width: 46px" text="" icon="fa-solid fa-volume-high"
                            loadingIcon="fa-solid fa-spinner fa-spin" :loading="inProgress" @click="control(3)" />
                    </b-col>
                    <b-col v-if="shocker.permissions.vibrate" cols="auto" md="auto">
                        <control-button style="width: 46px" text="" icon="fa-solid fa-water"
                            loadingIcon="fa-solid fa-spinner fa-spin" :loading="inProgress" @click="control(2)" />
                    </b-col>
                    <b-col v-if="shocker.permissions.shock" cols="auto" md="auto">
                        <control-button style="left: 0; width: 46px" text="" icon="fa-solid fa-bolt"
                            loadingIcon="fa-solid fa-spinner fa-spin" :loading="inProgress" @click="control(1)" />
                    </b-col>
                </b-row>
            </div>
        </div>
    </b-container>
</template>

<script>
import PR from '@/js/PauseReason.js';
import Loading from '../../../utils/Loading.vue';
import RoundSlider from 'vue-three-round-slider';
import ControlButton from '../../../utils/ControlButton.vue';

export default {
    components: { Loading, RoundSlider, ControlButton },

    props: ["shocker", "editMode", "isOwn"],
    data() {
        return {
            inProgress: false,
            sliderValue: 0
        }
    },
    beforeMount() {

    },
    methods: {
        ellipsis(e) {
            this.$contextmenu({
                theme: utils.isDarkMode() ? 'default dark' : 'default',
                x: e.x,
                y: e.y,
                items: [
                    {
                        label: "Remove from Share Link",
                        icon: 'fa-solid fa-trash',
                        onClick: () => {
                            this.delete();
                        }
                    }
                ]
            });
        },
        delete() {
            this.$swal({
                title: 'Remove from ShareLink?',
                html: `You are about to remove the Shocker <b>${this.shocker.name}</b> [${this.shocker.id}] from this Share Link.<br>`,
                icon: 'question',
                showCancelButton: true,
                confirmButtonColor: '#d33',
                cancelButtonColor: 'var(--secondary-seperator-color)',
                showLoaderOnConfirm: true,
                confirmButtonText: 'Remove',
                allowOutsideClick: () => !this.$swal.isLoading(),
                preConfirm: async () => {
                    try {
                        const res = await apiCall.makeCall('DELETE', `1/shares/links/${this.$route.params.id}/${this.shocker.id}`);
                        if (res.status !== 200) {
                            throw new Error(res.statusText);
                        }

                    } catch (err) {
                        this.$swal.showValidationMessage(`Request failed: ${err}`)
                    }
                },
            }).then(async (result) => {
                if (result.isConfirmed) {
                    this.$swal('Success!', 'Successfully removed Shocker from Share Link', 'success');
                    this.emitter.emit('refreshShareLink');
                }
            });
        },
        async control(type) {
            this.$emit('control', {
                id: this.shocker.id,
                intensity: parseInt(this.shocker.state.intensity),
                duration: parseFloat(this.shocker.state.duration) * 1000,
                type: this.inProgress ? 0 : type
            });

            if (this.inProgress) {
                this.inProgress = false;
                return;
            }
            this.inProgress = true;

            setTimeout(() => this.inProgress = false, this.shocker.state.duration * 1000);
        },
        async togglePause() {
            const toSet = !this.pausedOnShareLinkLevel;
            const res = await apiCall.makeCall("POST", `1/shares/links/${this.$route.params.id}/${this.shocker.id}/pause`, {
                pause: toSet
            });

            if (res === undefined || res.status !== 200) {
                toastr.error("Error while updating pause state");
                return;
            }

            this.shocker.paused = res.data.data;
        }
    },
    computed: {
        pausedOnShareLinkLevel() {
            return this.shocker.paused & PR.SHARE_LINK;
        },
        pauseReasonText() {
            const p = this.shocker.paused;
            if (p & PR.SHOCKER && p & PR.SHARE_LINK) return "Share link and shocker level"
            if (p & PR.SHARE_LINK) return "Share link level";
            if (p & PR.SHOCKER) return "Shocker level";

            if (p === 0) return null;

            return "Level: " + p;
        }
    }
}
</script>

<style scoped lang="scss">
.shocker {
    background-color: var(--secondary-background-color);
    border: solid var(--main-seperator-color) 1px;
    border-radius: 10px;
    margin: 10px 0;
    padding: 10px;
    box-shadow: rgba(0, 0, 0, 0.24) 0px 3px 8px;

    .content-container {
        position: relative;

        transition: 1s filter linear, 1s -webkit-filter linear;
        min-height: 255px;

        .paused-text {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            z-index: 100;

            svg {
                padding: 0;
            }
        }

        .content-child {
            transition: filter 0.3s ease-in-out;
            filter: blur(0px);
        }

        .paused {
            pointer-events: none;
            filter: blur(5px);
        }
    }

    .head {
        border-bottom: solid 2px var(--main-background-color);
        padding-bottom: 10px !important;
        margin-left: -10px;
        margin-right: -10px;

        .pause-col {
            cursor: pointer;
            padding-left: 4px;
            padding-right: 4px;
            width: 20px;

            .paused {
                color: rgb(255, 86, 86);
            }
        }

        .shocker-name-col {
            margin-right: auto;
            width: calc(100% - 50px);
            padding-left: 8px;

            &.reduced-padding-left {
                padding-left: 5px;
            }

            .shocker-name {
                white-space: nowrap;
                text-overflow: ellipsis;
                overflow: hidden;
                margin-bottom: 0;
            }
        }
    }

    .row {
        padding: 0 12px;

        .form-range {
            padding: 0 12px;
        }
    }
}</style>