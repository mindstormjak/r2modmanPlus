<template>
    <div>
        <div class="titlebar">
            <div class="titlebar_start">
                <p class="titlebar_title">{{ appName }}</p>
                <p class="titlebar_version">{{ appVersion }}</p>
            </div>
            <div class="titlebar_buttons">
                <div class="btn" @click="() => ipcRenderer.send('window-minimize')">
                    <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 8.47 8.47">
                        <path
                            d="M 0.71464503,4.235 H 7.7550979" stroke="currentColor" fill="currentColor"  stroke-linecap="round" stroke-width="1.59"/>
                    </svg>
                </div>
                <div class="btn" @click="() => ipcRenderer.send('window-maximize')">
                    <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 8.47 8.47">
                        <path
                            d="M 0.70215499,0.70215499 H 7.767847 V 7.7678511 H 0.70215499 Z" stroke="currentColor" fill="none" stroke-linecap="round" stroke-width="1.59" />
                    </svg>
                </div>
                <div class="closebutton btn" @click="() => ipcRenderer.send('window-close')">
                    <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 8.47 8.47">
                        <path d="m7.67.794-6.88 6.88m0-6.88 6.88 6.88" stroke="currentColor" fill="currentColor" stroke-linecap="round" stroke-width="1.59" />
                    </svg>
                </div>
            </div>
        </div>
        <router-view v-if="visible"/>
        <ErrorModal />
    </div>
</template>

<script lang="ts">
import Component, { mixins } from 'vue-class-component';
import 'bulma-steps/dist/js/bulma-steps.min.js';
import ManagerSettings from './r2mm/manager/ManagerSettings';
import ProfileProvider from './providers/ror2/model_implementation/ProfileProvider';
import ProfileImpl from './r2mm/model_implementation/ProfileImpl';
import LogOutput from './r2mm/data/LogOutput';
import LogOutputProvider from './providers/ror2/data/LogOutputProvider';
import ThunderstoreDownloaderProvider from './providers/ror2/downloading/ThunderstoreDownloaderProvider';
import BetterThunderstoreDownloader from './r2mm/downloading/BetterThunderstoreDownloader';
import { ipcRenderer } from 'electron';
import PathResolver from './r2mm/manager/PathResolver';
import path from 'path';
import ThemeManager from './r2mm/manager/ThemeManager';
import 'bulma-switch/dist/css/bulma-switch.min.css';
import LoggerProvider, { LogSeverity } from './providers/ror2/logging/LoggerProvider';
import ManagerInformation from './_managerinf/ManagerInformation';
import LocalModInstallerProvider from './providers/ror2/installing/LocalModInstallerProvider';
import LocalModInstaller from './r2mm/installing/LocalModInstaller';
import { Logger } from './r2mm/logging/Logger';
import FileUtils from './utils/FileUtils';
import LinkProvider from './providers/components/LinkProvider';
import LinkImpl from './r2mm/component_override/LinkImpl';
import FsProvider from './providers/generic/file/FsProvider';
import NodeFs from './providers/generic/file/NodeFs';
import { DataFolderProvider } from './providers/ror2/system/DataFolderProvider';
import { DataFolderProviderImpl } from './r2mm/system/DataFolderProviderImpl';
import InteractionProvider from './providers/ror2/system/InteractionProvider';
import InteractionProviderImpl from './r2mm/system/InteractionProviderImpl';
import ZipProvider from './providers/generic/zip/ZipProvider';
import AdmZipProvider from './providers/generic/zip/AdmZipProvider';
import ManagerSettingsMigration from './r2mm/manager/ManagerSettingsMigration';
import BindLoaderImpl from './providers/components/loaders/bind_impls/BindLoaderImpl';
import PlatformInterceptorProvider from './providers/generic/game/platform_interceptor/PlatformInterceptorProvider';
import PlatformInterceptorImpl from './providers/generic/game/platform_interceptor/PlatformInterceptorImpl';
import ProfileInstallerProvider from './providers/ror2/installing/ProfileInstallerProvider';
import InstallationRules from './r2mm/installing/InstallationRules';
import InstallationRuleApplicator from './r2mm/installing/default_installation_rules/InstallationRuleApplicator';
import GenericProfileInstaller from './r2mm/installing/profile_installers/GenericProfileInstaller';
import ConnectionProviderImpl from './r2mm/connection/ConnectionProviderImpl';
import ConnectionProvider from './providers/generic/connection/ConnectionProvider';
import UtilityMixin from './components/mixins/UtilityMixin.vue';
import ErrorModal from './components/modals/ErrorModal.vue';

@Component({
    components: {
        ErrorModal,
    }
})
export default class App extends mixins(UtilityMixin) {
    public visible: boolean = false;
    readonly ipcRenderer = ipcRenderer;

    async created() {
        // Load settings using the default game before the actual game is selected.
        const settings: ManagerSettings = await this.$store.dispatch('resetActiveGame');

        this.hookThunderstoreModListRefresh();
        await this.checkCdnConnection();

        InstallationRuleApplicator.apply();
        InstallationRules.validate();

        ipcRenderer.once('receive-appData-directory', async (_sender: any, appData: string) => {
            PathResolver.APPDATA_DIR = path.join(appData, 'r2modmanPlus-local');
            // Legacy path. Needed for migration.
            PathResolver.CONFIG_DIR = path.join(PathResolver.APPDATA_DIR, "config");

            if (ManagerSettings.NEEDS_MIGRATION) {
                await ManagerSettingsMigration.migrate();
            }

            PathResolver.ROOT = settings.getContext().global.dataDirectory || PathResolver.APPDATA_DIR;

            // If ROOT directory was set previously but no longer exists (EG: Drive disconnected) then fallback to original.
            try {
                await FileUtils.ensureDirectory(PathResolver.ROOT);
            } catch (e) {
                PathResolver.ROOT = PathResolver.APPDATA_DIR;
            }

            await FileUtils.ensureDirectory(PathResolver.APPDATA_DIR);

            await ThemeManager.apply();
            ipcRenderer.once('receive-is-portable', async (_sender: any, isPortable: boolean) => {
                ManagerInformation.IS_PORTABLE = isPortable;
                LoggerProvider.instance.Log(LogSeverity.INFO, `Starting manager on version ${ManagerInformation.VERSION.toString()}`);
                this.visible = true;
            });
            ipcRenderer.send('get-is-portable');
        });
        ipcRenderer.send('get-appData-directory');

        this.$watch('$q.dark.isActive', () => {
            document.documentElement.classList.toggle('html--dark', this.$q.dark.isActive);
        });
    }

    beforeCreate() {

        ConnectionProvider.provide(() => new ConnectionProviderImpl());
        FsProvider.provide(() => new NodeFs());

        ProfileProvider.provide(() => new ProfileImpl());
        LogOutputProvider.provide(() => LogOutput.getSingleton());

        const betterThunderstoreDownloader = new BetterThunderstoreDownloader();
        ThunderstoreDownloaderProvider.provide(() => betterThunderstoreDownloader);

        ZipProvider.provide(() => new AdmZipProvider());
        LocalModInstallerProvider.provide(() => new LocalModInstaller());
        ProfileInstallerProvider.provide(() => new GenericProfileInstaller());
        LoggerProvider.provide(() => new Logger());
        LinkProvider.provide(() => new LinkImpl());
        InteractionProvider.provide(() => new InteractionProviderImpl());
        DataFolderProvider.provide(() => new DataFolderProviderImpl());

        PlatformInterceptorProvider.provide(() => new PlatformInterceptorImpl());

        BindLoaderImpl.bind();
    }

    get appName(): string {
        return ManagerInformation.APP_NAME;
    }

    get appVersion(): string {
        return ManagerInformation.VERSION.toString();
    }

}
</script>
