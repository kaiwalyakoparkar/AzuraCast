<template>
    <profile-header
        v-bind="pickProps(props, headerPanelProps)"
        :station="profileInfo.station"
    />

    <div
        id="profile"
        class="row row-of-cards"
    >
        <div class="col-lg-7">
            <template v-if="hasStarted">
                <profile-now-playing
                    v-bind="pickProps(props, nowPlayingPanelProps)"
                    @api-call="makeApiCall"
                />

                <profile-schedule
                    :schedule-items="profileInfo.schedule"
                />

                <profile-streams
                    :station="profileInfo.station"
                />
            </template>
            <template v-else>
                <now-playing-not-started-panel />
            </template>

            <profile-public-pages
                v-bind="pickProps(props, {...publicPagesPanelProps,...embedModalProps})"
            />
        </div>

        <div class="col-lg-5">
            <profile-requests
                v-if="stationSupportsRequests"
                v-bind="pickProps(props, requestsPanelProps)"
            />

            <profile-streamers
                v-if="stationSupportsStreamers"
                v-bind="pickProps(props, streamersPanelProps)"
            />

            <template v-if="hasActiveFrontend">
                <profile-frontend
                    v-bind="pickProps(props, frontendPanelProps)"
                    :frontend-running="profileInfo.services.frontend_running"
                    @api-call="makeApiCall"
                />
            </template>

            <template v-if="hasActiveBackend">
                <profile-backend
                    v-bind="pickProps(props, backendPanelProps)"
                    :backend-running="profileInfo.services.backend_running"
                    @api-call="makeApiCall"
                />
            </template>
            <template v-else>
                <profile-backend-none />
            </template>
        </div>
    </div>
</template>

<script setup lang="ts">
import ProfileStreams from './Profile/StreamsPanel.vue';
import ProfileHeader from './Profile/HeaderPanel.vue';
import ProfileNowPlaying from './Profile/NowPlayingPanel.vue';
import ProfileSchedule from './Profile/SchedulePanel.vue';
import ProfileRequests from './Profile/RequestsPanel.vue';
import ProfileStreamers from './Profile/StreamersPanel.vue';
import ProfilePublicPages from './Profile/PublicPagesPanel.vue';
import ProfileFrontend from './Profile/FrontendPanel.vue';
import ProfileBackendNone from './Profile/BackendNonePanel.vue';
import ProfileBackend from './Profile/BackendPanel.vue';
import NowPlayingNotStartedPanel from "./Profile/NowPlayingNotStartedPanel.vue";
import {BackendAdapter, FrontendAdapter} from '~/entities/RadioAdapters';
import NowPlaying from '~/entities/NowPlaying';
import {computed} from "vue";
import {useAxios} from "~/vendor/axios";
import backendPanelProps from "./Profile/backendPanelProps";
import embedModalProps from "./Profile/embedModalProps";
import frontendPanelProps from "./Profile/frontendPanelProps";
import headerPanelProps from "./Profile/headerPanelProps";
import nowPlayingPanelProps from "./Profile/nowPlayingPanelProps";
import publicPagesPanelProps from "./Profile/publicPagesPanelProps";
import requestsPanelProps from "./Profile/requestsPanelProps";
import streamersPanelProps from "./Profile/streamersPanelProps";
import {pickProps} from "~/functions/pickProps";
import useRefreshableAsyncState from "~/functions/useRefreshableAsyncState";
import {useIntervalFn} from "@vueuse/core";
import {useSweetAlert} from "~/vendor/sweetalert";
import {useNotify} from "~/functions/useNotify";
import {useTranslate} from "~/vendor/gettext";

const props = defineProps({
    ...backendPanelProps,
    ...embedModalProps,
    ...frontendPanelProps,
    ...headerPanelProps,
    ...nowPlayingPanelProps,
    ...publicPagesPanelProps,
    ...requestsPanelProps,
    ...streamersPanelProps,
    profileApiUri: {
        type: String,
        required: true
    },
    stationSupportsRequests: {
        type: Boolean,
        required: true
    },
    stationSupportsStreamers: {
        type: Boolean,
        required: true
    }
});

const hasActiveFrontend = computed(() => {
    return props.frontendType !== FrontendAdapter.Remote;
});

const hasActiveBackend = computed(() => {
    return props.backendType !== BackendAdapter.None;
});

const {axios} = useAxios();

const {state: profileInfo, execute: reloadProfile} = useRefreshableAsyncState(
    () => axios.get(props.profileApiUri).then((r) => r.data),
    {
        station: {
            ...NowPlaying.station
        },
        services: {
            backend_running: false,
            frontend_running: false,
            has_started: false,
            needs_restart: false
        },
        schedule: []
    }
);

const profileReloadTimeout = computed(() => {
    return (!document.hidden) ? 15000 : 30000
});

useIntervalFn(
    reloadProfile,
    profileReloadTimeout
);

const {showAlert} = useSweetAlert();
const {notify} = useNotify();
const {$gettext} = useTranslate();

const makeApiCall = (uri) => {
    showAlert({
        title: $gettext('Are you sure?')
    }).then((result) => {
        if (!result.value) {
            return;
        }

        axios.post(uri).then((resp) => {
            notify(resp.data.formatted_message, {
                variant: (resp.data.success) ? 'success' : 'warning'
            });
        });
    });
};
</script>
