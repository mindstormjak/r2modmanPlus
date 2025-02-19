<template>
    <div class="page-with-topbar">
        <Hero title="Downloads" subtitle="Monitor progress of downloads" hero-type="is-info"/>
        <template v-if="activeDownloads.length === 0">
            <div class='text-center top'>
                <div class="margin-right">
                    <br/>
                    <h3 class='title is-4'>You don't have anything downloading.</h3>
                    <h4 class='subtitle is-5'>
                        Click <router-link :to="{name: 'manager.online'}">here</router-link> to download something.
                    </h4>
                </div>
            </div>
        </template>
        <template v-else>
            <div v-for="([assignId, downloadObject], index) of activeDownloads" :key="`download-progress-${index}`">
                <div>
                    <div class="container margin-right">
                        <div class="border-at-bottom pad pad--sides">
                            <div class="card is-shadowless">
                                <p><strong>{{ downloadObject.initialMods.join(", ") }}</strong></p>
                                <p v-if="downloadObject.progress < 100">Downloading: {{ downloadObject.modName }}</p>
                                <p>{{Math.min(Math.floor(downloadObject.progress), 100)}}% complete</p>
                                <Progress v-if="!downloadObject.failed"
                                    :max='100'
                                    :value='downloadObject.progress'
                                    :className="['is-info']"
                                />
                                <Progress v-else-if="downloadObject.failed"
                                    :max='100'
                                    :value='100'
                                    :className="['is-danger']"
                                />
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </template>
    </div>
</template>

<script lang="ts">

import { Component, Vue } from 'vue-property-decorator';
import { Hero } from '../components/all';
import DownloadModModal from '../components/views/DownloadModModal.vue';
import Progress from '../components/Progress.vue';
import Timeout = NodeJS.Timeout;

@Component({
    components: {
        Progress,
        Hero,
    }
})
export default class DownloadMonitor extends Vue {
    public refreshInterval!: Timeout;
    public activeDownloads: [number, any][] = [];

    created() {
        this.activeDownloads = [...DownloadModModal.allVersions].reverse();
        this.refreshInterval = setInterval(() => {
            this.activeDownloads = [...DownloadModModal.allVersions].reverse();
        }, 100);
    }

    destroyed() {
        clearInterval(this.refreshInterval);
    }

}

</script>
