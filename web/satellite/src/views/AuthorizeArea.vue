// Copyright (C) 2022 Storj Labs, Inc.
// See LICENSE for copying information.

<template>
    <div class="authorize-area">
        <div class="authorize-area__logo-wrapper">
            <LogoIcon class="logo" @click="location.reload()" />
        </div>

        <div class="authorize-area__content-area">
            <div v-if="requestErr" class="authorize-area__content-area__container">
                <p>{{ requestErr }}</p>
            </div>
            <div v-else class="authorize-area__content-area__container">
                <p v-if="client.appLogoURL" class="authorize-area__content-area__client-app-logo">
                    <img :alt="client.appName" :src="client.appLogoURL">
                </p>

                <p class="authorize-area__content-area__client-app">
                    {{ client.appName }} would like permission to:
                </p>

                <div class="authorize-area__permissions-area">
                    <div class="authorize-area__permissions-area__container">
                        <p class="authorize-area__permissions-area__header">Verify your Storj Identity</p>
                        <p>Access and view your account info.</p>
                    </div>
                    <div class="authorize-area__permissions-area__container">
                        <p class="authorize-area__permissions-area__header">Sync data to Storj DCS</p>
                        <p>Automatically send updates to:</p>

                        <div class="authorize-area__input-wrapper">
                            <HeaderlessInput
                                label="Project"
                                role-description="project"
                                :error="projectErr"
                                :options-list="Object.keys(projects)"
                                @setData="setProject"
                            />
                        </div>

                        <div class="authorize-area__input-wrapper">
                            <HeaderlessInput
                                label="Bucket"
                                role-description="bucket"
                                :error="bucketErr"
                                :value="selectedBucketName"
                                :options-list="buckets"
                                @setData="setBucket"
                            />
                            <div v-if="!bucketExists" class="info-box">
                                <p class="info-box__message">
                                    This bucket will be created.
                                </p>
                            </div>
                        </div>

                        <div class="authorize-area__input-wrapper">
                            <HeaderlessInput
                                label="Passphrase"
                                role-description="passphrase"
                                placeholder="Passphrase"
                                :error="passphraseErr"
                                is-password="true"
                                @setData="setPassphrase"
                            />
                        </div>
                    </div>
                    <div class="authorize-area__permissions-area__container">
                        <p class="authorize-area__permissions-area__container__header">Perform the following actions</p>
                        <p>{{ actions }} objects.</p>
                    </div>
                </div>

                <form method="post">
                    <input v-model="oauthData.client_id" type="hidden" name="client_id">
                    <input v-model="oauthData.redirect_uri" type="hidden" name="redirect_uri">
                    <input v-model="oauthData.response_type" type="hidden" name="response_type">
                    <input v-model="oauthData.state" type="hidden" name="state">
                    <input v-model="scope" type="hidden" name="scope">

                    <input class="authorize-area__content-area__container__button" :class="{ 'disabled-button': !valid }" type="submit" :disabled="!valid" value="Authorize">
                    <p class="authorize-area__content-area__container__cancel" @click.prevent="onDeny">Cancel</p>
                </form>
            </div>
        </div>
    </div>
</template>

<script lang="ts">
import {Component, Vue} from 'vue-property-decorator';
import HeaderlessInput from '@/components/common/HeaderlessInput.vue';
import LogoIcon from '@/../static/images/logo.svg';
import {Validator} from '@/utils/validation';
import {RouteConfig} from '@/router';
import {BUCKET_ACTIONS} from '@/store/modules/buckets';
import {PROJECTS_ACTIONS} from '@/store/modules/projects';
import {USER_ACTIONS} from '@/store/modules/users';
import {Project} from '@/types/projects';
import {ErrorUnauthorized} from '@/api/errors/ErrorUnauthorized';
import {APP_STATE_ACTIONS} from '@/utils/constants/actionNames';
import {AppState} from '@/utils/constants/appStateEnum';
import {ACCESS_GRANTS_ACTIONS} from '@/store/modules/accessGrants';
import {OAuthClient, OAuthClientsAPI} from '@/api/oauthClients';
import {URLSearchParams} from "url";

const oauthClientsAPI = new OAuthClientsAPI();

// @vue/component
@Component({
    components: {
        HeaderlessInput,
        LogoIcon,
    },
})
export default class Authorize extends Vue {
    private requestErr = '';

    private oauthData: {
      client_id?: string;
      redirect_uri?: string;
      state?: string;
      response_type?: string;
      scope?: string;
    } = {};
    private clientKey = '';

    private client: OAuthClient = {
        id: '',
        redirectURL: '',
        appName: '',
        appLogoURL: '',
    };

    private projects: Record<string, Project> = {};
    private buckets: Array<string> = [];

    private selectedProjectID = '';
    private selectedBucketName = '';
    private providedPassphrase = '';
    private scope = '';

    private valid = false;
    private projectErr = '';
    private bucketErr = '';
    private passphraseErr = '';

    private actions = '';
    private bucketExists = false;

    private worker: Worker;

    private async ensureLogin(): Promise<void> {
        try {
            await this.$store.dispatch(USER_ACTIONS.GET);
        } catch (error) {
            if (!(error instanceof ErrorUnauthorized)) {
                await this.$store.dispatch(APP_STATE_ACTIONS.CHANGE_STATE, AppState.ERROR);
                await this.$notify.error(error.message);
            }

            const query = new URLSearchParams(this.oauthData).toString();
            const path = `${RouteConfig.Authorize.path}?${query}#${this.clientKey}`;

            await this.$router.push(`${RouteConfig.Login.path}?return_url=${encodeURIComponent(path)}`);
            return;
        }
    }

    private async ensureWorker(): Promise<void> {
        try {
            await this.$store.dispatch(ACCESS_GRANTS_ACTIONS.STOP_ACCESS_GRANTS_WEB_WORKER);
            await this.$store.dispatch(ACCESS_GRANTS_ACTIONS.SET_ACCESS_GRANTS_WEB_WORKER);
        } catch (error) {
            await this.$notify.error(`Unable to set access grants wizard. ${error.message}`);
            return;
        }

        this.worker = this.$store.state.accessGrantsModule.accessGrantsWebWorker;
        this.worker.onerror = (error: ErrorEvent) => this.$notify.error(error.message);
    }

    private async verifyClientConfiguration(): Promise<void> {
        const clientID: string = this.oauthData.client_id ?? "";
        const redirectURL: string = this.oauthData.redirect_uri ?? "";
        const state: string = this.oauthData.state ?? "";
        const responseType: string = this.oauthData.response_type ?? "";
        const scope: string = this.oauthData.scope ?? "";

        if (!clientID || !redirectURL) {
            this.requestErr = 'Both client_id and redirect_uri must be provided.';
            return;
        }

        let client: OAuthClient;
        try {
            client = await oauthClientsAPI.get(clientID);
        } catch (e) {
            this.requestErr = e.message;
            return;
        }

        let err: { [key: string]: string } | null = null;

        if (!state || !responseType || !scope) {
            err = {
                error_description: 'The request is missing a required parameter (state, response_type, or scope).',
            };
        } else if (!this.clientKey) {
            err = {
                error_description: 'An encryption key must be provided in the fragment of the request.',
            };
        } else if (!redirectURL.startsWith(client.redirectURL)) {
            err = {
                error_description: 'The provided redirect url does not match the one in our system.',
            };
        }

        if (err) {
            location.href = `${redirectURL}?${(new URLSearchParams(err)).toString()}`;
            return
        }

        this.client = client;

        // initialize the form

        this.setBucket(slugify(this.client.appName));
        this.actions = formatObjectPermissions(scope);
    }

    private async loadProjects(): Promise<void> {
        await this.$store.dispatch(PROJECTS_ACTIONS.FETCH);

        const projects = {};
        for (const project of this.$store.getters.projects) {
            projects[project.name] = project;
        }

        this.projects = projects;
    }

    /**
     * Lifecycle hook after initial render.
     * Makes activated banner visible on successful account activation.
     */
    public async mounted(): Promise<void> {
        this.oauthData = this.$route.query;
        this.clientKey = this.$route.hash ? this.$route.hash.substring(1) : "";

        await this.ensureLogin();
        await this.ensureWorker();

        await this.verifyClientConfiguration();
        if (this.requestErr) {
            return
        }

        await this.loadProjects();
    }

    public async setProject(value: string): Promise<void> {
        if (!this.projects[value]) {
            this.projectErr = 'project does not exist';
            return
        }

        await this.$store.dispatch(PROJECTS_ACTIONS.SELECT, this.projects[value].id);
        await this.$store.dispatch(BUCKET_ACTIONS.FETCH_ALL_BUCKET_NAMES);

        this.selectedProjectID = this.$store.getters.selectedProject.id;
        this.buckets = this.$store.state.bucketUsageModule.allBucketNames.sort();

        this.setBucket(this.selectedBucketName);
    }

    public setBucket(value: string): void {
        this.selectedBucketName = value;
        this.bucketExists = this.selectedProjectID.length == 0 || value.length == 0 || this.buckets.includes(value);

        this.setScope();
    }

    public setPassphrase(value: string): void {
        this.providedPassphrase = value;

        this.setScope();
    }

    public async setScope(): Promise<void> {
        if (!this.validate()) {
            return;
        }

        this.worker.postMessage({
            'type': 'DeriveAndEncryptRootKey',
            'passphrase': this.providedPassphrase,
            'projectID': this.selectedProjectID,
            'aesKey': this.clientKey,
        });

        const event: MessageEvent = await new Promise(resolve => this.worker.onmessage = resolve);

        if (event.data.error) {
            await this.$notify.error(event.data.error);
            return;
        }

        const scope = this.oauthData.scope,
            project = this.selectedProjectID,
            bucket = this.selectedBucketName,
            cubbyhole = event.data.value;

        this.scope = `${scope} project:${project} bucket:${bucket} cubbyhole:${cubbyhole}`;
    }

    public async onDeny(): Promise<void> {
        location.href = `${this.oauthData.redirect_uri}?${new URLSearchParams({
            error_description: 'The resource owner or authorization server denied the request',
        }).toString()}`;
    }

    private validate(): boolean {
        this.projectErr = '';
        this.bucketErr = '';
        this.passphraseErr = '';

        if (this.selectedProjectID == '') {
            this.projectErr = 'Missing project.';
        }

        if (!Validator.bucketName(this.selectedBucketName)) {
            this.bucketErr = 'Name must contain only lowercase latin characters, numbers, a hyphen or a period';
        }

        if (this.providedPassphrase == '') {
            this.passphraseErr = 'A passphrase must be provided.';
        }

        this.valid = this.projectErr == '' &&
            this.bucketErr == '' &&
            this.passphraseErr == '';

        return this.valid;
    }
}

const validPerms = {
    'list': true,
    'read': true,
    'write': true,
    'delete': true,
};

function slugify(name: string): string {
    name = name.toLowerCase();
    name = name.replace(/\s+/g, "-");
    return name;
}

function formatObjectPermissions(scope: string): string {
    const scopes = scope.split(" ");
    const perms: string[] = [];

    for (const scope of scopes) {
        if (scope.startsWith("object:")) {
            const perm = scope.substring("object:".length);
            if (validPerms[perm]) {
                perms.push(perm);
            }
        }
    }

    perms.sort();

    if (perms.length == 0) {
        return "";
    } else if (perms.length == 1) {
        return perms[0];
    } else if (perms.length == 2) {
        return `${perms[0]} and ${perms[1]}`;
    }

    return `${perms.slice(0, perms.length - 1).join(", ")}, and ${perms[perms.length - 1]}`;
}
</script>

<style scoped lang="scss">
    .authorize-area {
        display: flex;
        flex-direction: column;
        font-family: 'font_regular', sans-serif;
        background-color: #f5f6fa;
        position: fixed;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        min-height: 100%;
        overflow-y: scroll;

        .info-box {
            background-color: #e9f3ff;
            border-radius: 6px;
            padding: 20px;
            margin-top: 25px;
            width: 100%;
            box-sizing: border-box;

            &.error {
                background-color: #fff9f7;
                border: 1px solid #f84b00;
            }

            &__header {
                display: flex;
                align-items: center;

                &__label {
                    font-family: 'font_bold', sans-serif;
                    font-size: 16px;
                    color: #1b2533;
                    margin-left: 15px;
                }
            }

            &__message {
                font-size: 16px;
                color: #1b2533;
            }
        }

        &__permissions-area {
            background-color: #fafafb;
            border: 1px solid #d8dee3;
            padding: 10px 9px 10px 24px;
            border-radius: 8px;

            &__header {
                font-size: 18px;
                margin-bottom: 4px;
            }

            &__container {
                padding: 24px 0;
                border-bottom: 1px solid #d8dee3;

                p {
                    line-height: 24px;
                    vertical-align: middle;
                }

                &:last-of-type {
                    border-bottom: none;
                }
            }
        }

        &__logo-wrapper {
            text-align: center;
            margin: 70px 0;
        }

        &__divider {
            margin: 0 20px;
            height: 22px;
            width: 2px;
            background-color: #acbace;
        }

        &__input-wrapper {
            margin-top: 20px;
            padding-right: 24px;
            width: 100%;
            box-sizing: border-box;
        }

        &__expand {
            display: flex;
            align-items: center;
            cursor: pointer;
            position: relative;

            &__value {
                font-size: 16px;
                line-height: 21px;
                color: #acbace;
                margin-right: 10px;
                font-family: 'font_regular', sans-serif;
                font-weight: 700;
            }

            &__dropdown {
                position: absolute;
                top: 35px;
                left: 0;
                background-color: #fff;
                z-index: 1000;
                border: 1px solid #c5cbdb;
                box-shadow: 0 8px 34px rgb(161 173 185 / 41%);
                border-radius: 6px;
                min-width: 250px;

                &__item {
                    display: flex;
                    align-items: center;
                    justify-content: flex-start;
                    padding: 12px 25px;
                    font-size: 14px;
                    line-height: 20px;
                    color: #7e8b9c;
                    cursor: pointer;
                    text-decoration: none;

                    &__name {
                        font-family: 'font_bold', sans-serif;
                        margin-left: 15px;
                        font-size: 14px;
                        line-height: 20px;
                        color: #7e8b9c;
                    }

                    &:hover {
                        background-color: #f2f2f6;
                    }
                }
            }
        }

        &__content-area {
            background-color: #f5f6fa;
            padding: 0 20px;
            margin-bottom: 50px;
            display: flex;
            flex-direction: column;
            align-items: center;
            border-radius: 20px;
            box-sizing: border-box;

            &__activation-banner {
                padding: 20px;
                background-color: rgb(39 174 96 / 10%);
                border: 1px solid #27ae60;
                color: #27ae60;
                border-radius: 6px;
                width: 570px;
                margin-bottom: 30px;

                &__message {
                    font-size: 16px;
                    line-height: 21px;
                    margin: 0;
                }
            }

            &__client-app-logo {
                text-align: center;
                margin-bottom: 60px;
            }

            &__client-app {
                text-align: center;
                font-size: 22px;
                font-weight: bold;
                margin-bottom: 16px;
            }

            &__container {
                display: flex;
                flex-direction: column;
                padding: 60px 80px;
                background-color: #fff;
                width: 610px;
                border-radius: 20px;
                box-sizing: border-box;
                margin-bottom: 20px;

                &__title-area {
                    display: flex;
                    justify-content: space-between;
                    align-items: center;

                    &__title {
                        font-size: 24px;
                        line-height: 49px;
                        letter-spacing: -0.1007px;
                        color: #252525;
                        font-family: 'font_bold', sans-serif;
                        font-weight: 800;
                    }

                    &__satellite {
                        font-size: 16px;
                        line-height: 21px;
                        color: #848484;
                    }
                }

                &__button {
                    font-family: 'font_regular', sans-serif;
                    font-weight: 700;
                    margin-top: 40px;
                    display: flex;
                    justify-content: center;
                    align-items: center;
                    background-color: #376fff;
                    border-radius: 50px;
                    color: #fff;
                    cursor: pointer;
                    width: 100%;
                    height: 48px;

                    &:hover {
                        background-color: #0059d0;
                    }
                }

                &__cancel {
                    align-self: center;
                    font-size: 16px;
                    line-height: 21px;
                    color: #0068dc;
                    text-align: center;
                    margin-top: 30px;
                    cursor: pointer;
                }

                &__recovery {
                    font-size: 16px;
                    line-height: 19px;
                    color: #0068dc;
                    cursor: pointer;
                    margin-top: 20px;
                    text-align: center;
                    width: 100%;
                }
            }

            &__footer-item {
                margin-top: 30px;
                font-size: 14px;
            }
        }
    }

    .logo {
        cursor: pointer;
    }

    .disabled,
    .disabled-button {
        pointer-events: none;
        color: #acb0bc;
    }

    .disabled-button {
        background-color: #dadde5;
        border-color: #dadde5;
    }

    @media screen and (max-width: 750px) {

        .authorize-area {

            &__content-area {

                &__container {
                    width: 100%;
                    padding: 60px;
                }
            }

            &__expand {

                &__dropdown {
                    left: -200px;
                }
            }
        }
    }

    @media screen and (max-width: 414px) {

        .authorize-area {

            &__logo-wrapper {
                margin: 40px;
            }

            &__content-area {
                padding: 0;

                &__container {
                    padding: 0 20px 20px;
                    background: transparent;
                }
            }
        }
    }
</style>
