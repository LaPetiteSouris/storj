// Copyright (C) 2021 Storj Labs, Inc.
// See LICENSE for copying information.

<template>
    <div class="buckets-view">
        <div class="buckets-view__title-area">
            <h1 class="buckets-view__title-area__title" aria-roledescription="title">Buckets</h1>
            <div class="buckets-view__title-area__button" :class="{ disabled: isLoading }" @click="onNewBucketButtonClick">
                <BucketIcon />
                <p class="buckets-view__title-area__button__label">New Bucket</p>
            </div>
        </div>
        <VLoader
            v-if="isLoading"
            width="100px"
            height="100px"
            class="buckets-view__loader"
        />
        <p v-if="!(isLoading || bucketsList.length)" class="buckets-view__no-buckets">No Buckets</p>
        <div v-if="!isLoading && bucketsList.length" class="buckets-view__list">
            <div class="buckets-view__list__sorting-header">
                <p class="buckets-view__list__sorting-header__name">Name</p>
                <p class="buckets-view__list__sorting-header__date">Date Added</p>
                <p class="buckets-view__list__sorting-header__empty" />
            </div>
            <div v-for="(bucket, key) in bucketsList" :key="key" class="buckets-view__list__item" @click.stop="openBucket(bucket.Name)">
                <BucketItem
                    :item-data="bucket"
                    :show-delete-bucket-popup="showDeleteBucketPopup"
                    :dropdown-key="key"
                    :open-dropdown="openDropdown"
                    :is-dropdown-open="activeDropdown === key"
                />
            </div>
        </div>
        <ObjectsPopup
            v-if="isCreatePopupVisible"
            :on-click="onCreateBucketClick"
            title="Create Bucket"
            sub-title="Buckets are simply containers that store objects and their metadata within a project."
            button-label="Create Bucket"
            :error-message="errorMessage"
            :is-loading="isRequestProcessing"
            @setName="setCreateBucketName"
            @close="hideCreateBucketPopup"
        />
        <ObjectsPopup
            v-if="isDeletePopupVisible"
            :on-click="onDeleteBucketClick"
            title="Are you sure?"
            sub-title="Deleting this bucket will delete all metadata related to this bucket."
            button-label="Confirm Delete Bucket"
            :error-message="errorMessage"
            :is-loading="isRequestProcessing"
            @setName="setDeleteBucketName"
            @close="hideDeleteBucketPopup"
        />
    </div>
</template>

<script lang="ts">
import { Bucket } from 'aws-sdk/clients/s3';
import { Component, Vue, Watch } from 'vue-property-decorator';

import { RouteConfig } from '@/router';
import { ACCESS_GRANTS_ACTIONS } from '@/store/modules/accessGrants';
import { OBJECTS_ACTIONS } from '@/store/modules/objects';
import { AccessGrant, EdgeCredentials } from '@/types/accessGrants';
import { MetaUtils } from '@/utils/meta';
import { Validator } from '@/utils/validation';
import { LocalData } from "@/utils/localData";

import VLoader from '@/components/common/VLoader.vue';
import BucketItem from '@/components/objects/BucketItem.vue';
import ObjectsPopup from '@/components/objects/ObjectsPopup.vue';

import BucketIcon from '@/../static/images/objects/bucket.svg';

// @vue/component
@Component({
    components: {
        BucketIcon,
        ObjectsPopup,
        BucketItem,
        VLoader,
    },
})
export default class BucketsView extends Vue {
    private readonly FILE_BROWSER_AG_NAME: string = 'Web file browser API key';
    private worker: Worker;
    private grantWithPermissions = '';
    private createBucketName = '';
    private deleteBucketName = '';

    public isLoading = true;
    public isCreatePopupVisible = false;
    public isDeletePopupVisible = false;
    public isRequestProcessing = false;
    public errorMessage = '';
    public activeDropdown = -1;

    /**
     * Lifecycle hook after initial render.
     * Setup gateway credentials.
     */
    public async mounted(): Promise<void> {
        await this.setBucketsView();
    }

    @Watch('selectedProjectID')
    public async handleProjectChange(): Promise<void> {
        this.isLoading = true;

        await this.$store.dispatch(OBJECTS_ACTIONS.CLEAR);
        await this.setBucketsView();
    }

    /**
     * Sets buckets view when needed.
     */
    public async setBucketsView(): Promise<void> {
        try {
            await this.setWorker();
            await this.removeTemporaryAccessGrant();
            await this.setAccess();
            await this.fetchBuckets();

            const wasDemoBucketCreated = LocalData.getDemoBucketCreatedStatus();

            if (this.bucketsList.length && !wasDemoBucketCreated) {
                LocalData.setDemoBucketCreatedStatus();

                return;
            }

            if (!this.bucketsList.length && wasDemoBucketCreated) {
                await this.removeTemporaryAccessGrant();

                return;
            }

            if (!this.bucketsList.length && !wasDemoBucketCreated) {
                if (this.isNewObjectsFlow) {
                    await this.$router.push(RouteConfig.Buckets.with(RouteConfig.BucketCreation).path);
                    return;
                }

                await this.createDemoBucket();
            }
        } catch (error) {
            await this.$notify.error(`Failed to setup Buckets view. ${error.message}`);
        } finally {
            this.isLoading = false;
        }
    }

    /**
     * Sets access to S3 client.
     */
    public async setAccess(): Promise<void> {
        const cleanAPIKey: AccessGrant = await this.$store.dispatch(ACCESS_GRANTS_ACTIONS.CREATE, this.FILE_BROWSER_AG_NAME);
        await this.$store.dispatch(OBJECTS_ACTIONS.SET_API_KEY, cleanAPIKey.secret);

        const now = new Date();
        const inThreeDays = new Date(now.setDate(now.getDate() + 3));

        await this.worker.postMessage({
            'type': 'SetPermission',
            'isDownload': true,
            'isUpload': true,
            'isList': true,
            'isDelete': true,
            'notAfter': inThreeDays.toISOString(),
            'buckets': [],
            'apiKey': cleanAPIKey.secret,
        });

        const grantEvent: MessageEvent = await new Promise(resolve => this.worker.onmessage = resolve);
        this.grantWithPermissions = grantEvent.data.value;
        if (grantEvent.data.error) {
            throw new Error(grantEvent.data.error);
        }

        const satelliteNodeURL: string = MetaUtils.getMetaContent('satellite-nodeurl');
        this.worker.postMessage({
            'type': 'GenerateAccess',
            'apiKey': this.grantWithPermissions,
            'passphrase': '',
            'projectID': this.$store.getters.selectedProject.id,
            'satelliteNodeURL': satelliteNodeURL,
        });

        const accessGrantEvent: MessageEvent = await new Promise(resolve => this.worker.onmessage = resolve);
        if (accessGrantEvent.data.error) {
            throw new Error(accessGrantEvent.data.error);
        }

        const accessGrant = accessGrantEvent.data.value;

        const gatewayCredentials: EdgeCredentials = await this.$store.dispatch(ACCESS_GRANTS_ACTIONS.GET_GATEWAY_CREDENTIALS, {accessGrant, isPublic: false});
        await this.$store.dispatch(OBJECTS_ACTIONS.SET_GATEWAY_CREDENTIALS, gatewayCredentials);
        await this.$store.dispatch(OBJECTS_ACTIONS.SET_S3_CLIENT);
    }

    /**
     * Fetches bucket using S3 client.
     */
    public async fetchBuckets(): Promise<void> {
        await this.$store.dispatch(OBJECTS_ACTIONS.FETCH_BUCKETS);
    }

    /**
     * Sets local worker with worker instantiated in store.
     */
    public setWorker(): void {
        this.worker = this.$store.state.accessGrantsModule.accessGrantsWebWorker;
        this.worker.onerror = (error: ErrorEvent) => {
            this.$notify.error(error.message);
        };
    }

    public onNewBucketButtonClick(): void {
        this.isNewObjectsFlow
            ? this.$router.push(RouteConfig.Buckets.with(RouteConfig.BucketCreation).path)
            : this.showCreateBucketPopup();
    }

    /**
     * Holds create bucket click logic.
     */
    public async onCreateBucketClick(): Promise<void> {
        if (this.isRequestProcessing) return;

        if (!this.isBucketNameValid(this.createBucketName)) return;

        this.isRequestProcessing = true;

        try {
            if (!this.edgeCredentials.accessKeyId) {
                await this.setAccess();
            }
            await this.$store.dispatch(OBJECTS_ACTIONS.CREATE_BUCKET, this.createBucketName);
            await this.$store.dispatch(OBJECTS_ACTIONS.FETCH_BUCKETS);
            this.createBucketName = '';
            this.hideCreateBucketPopup();
        } catch (error) {
            const BUCKET_ALREADY_EXISTS_ERROR = 'BucketAlreadyExists';

            if (error.name === BUCKET_ALREADY_EXISTS_ERROR) {
                await this.$notify.error('Bucket with provided name already exists.');
            } else {
                await this.$notify.error(error.message);
            }
        } finally {
            this.isRequestProcessing = false;
        }
    }

    /**
     * Creates first ever demo bucket for user.
     */
    public async createDemoBucket(): Promise<void> {
        if (this.isRequestProcessing) return;

        this.isRequestProcessing = true;

        try {
            await this.$store.dispatch(OBJECTS_ACTIONS.CREATE_DEMO_BUCKET);
            await this.$store.dispatch(OBJECTS_ACTIONS.FETCH_BUCKETS);

            LocalData.setDemoBucketCreatedStatus();
        } catch (error) {
            await this.$notify.error(error.message);
        } finally {
            this.isRequestProcessing = false;
        }
    }

    /**
     * Holds delete bucket click logic.
     */
    public async onDeleteBucketClick(): Promise<void> {
        if (this.isRequestProcessing) return;

        if (!this.isBucketNameValid(this.deleteBucketName)) return;

        this.isRequestProcessing = true;

        try {
            if (!this.edgeCredentials.accessKeyId) {
                await this.setAccess();
            }
            await this.$store.dispatch(OBJECTS_ACTIONS.DELETE_BUCKET, this.deleteBucketName);
            await this.$store.dispatch(OBJECTS_ACTIONS.FETCH_BUCKETS);
        } catch (error) {
            await this.$notify.error(error.message);
            return;
        } finally {
            this.isRequestProcessing = false;
        }

        this.deleteBucketName = '';
        this.hideDeleteBucketPopup();
    }

    /**
     * Removes temporary created access grant.
     */
    public async removeTemporaryAccessGrant(): Promise<void> {
        try {
            await this.$store.dispatch(ACCESS_GRANTS_ACTIONS.DELETE_BY_NAME_AND_PROJECT_ID, this.FILE_BROWSER_AG_NAME);
            await this.$store.dispatch(OBJECTS_ACTIONS.CLEAR);
        } catch (error) {
            await this.$notify.error(error.message);
        }
    }

    /**
     * Opens utils dropdown.
     */
    public openDropdown(key: number): void {
        if (this.activeDropdown === key) {
            this.activeDropdown = -1;

            return;
        }

        this.activeDropdown = key;
    }

    /**
     * Makes delete bucket popup visible.
     */
    public showDeleteBucketPopup(): void {
        this.deleteBucketName = '';
        this.isDeletePopupVisible = true;
    }

    /**
     * Hides delete bucket popup.
     */
    public hideDeleteBucketPopup(): void {
        this.errorMessage = '';
        this.isDeletePopupVisible = false;
    }

    /**
     * Set delete bucket name form input.
     */
    public setDeleteBucketName(name: string): void {
        this.errorMessage = '';
        this.deleteBucketName = name;
    }

    /**
     * Makes create bucket popup visible.
     */
    public showCreateBucketPopup(): void {
        this.createBucketName = '';
        this.isCreatePopupVisible = true;
    }

    /**
     * Hides create bucket popup.
     */
    public hideCreateBucketPopup(): void {
        this.errorMessage = '';
        this.isCreatePopupVisible = false;
    }

    /**
     * Set create bucket name form input.
     */
    public setCreateBucketName(name: string): void {
        this.errorMessage = '';
        this.createBucketName = name;
    }

    /**
     * Holds on bucket click. Proceeds to file browser.
     */
    public openBucket(bucketName: string): void {
        this.$store.dispatch(OBJECTS_ACTIONS.SET_FILE_COMPONENT_BUCKET_NAME, bucketName);
        this.$router.push(RouteConfig.Buckets.with(RouteConfig.EncryptData).path);
    }

    /**
     * Returns fetched buckets from store.
     */
    public get bucketsList(): Bucket[] {
        return this.$store.state.objectsModule.bucketsList;
    }

    /**
     * Returns objects flow status from store.
     */
    private get isNewObjectsFlow(): string {
        return this.$store.state.appStateModule.isNewObjectsFlow;
    }

    /**
     * Returns selected project id from store.
     */
    private get selectedProjectID(): string {
        return this.$store.getters.selectedProject.id;
    }

    /**
     * Returns edge credentials from store.
     */
    private get edgeCredentials(): EdgeCredentials {
        return this.$store.state.objectsModule.gatewayCredentials;
    }

    /**
     * Returns validation status of a bucket name.
     */
    private isBucketNameValid(name: string): boolean {
        switch (true) {
        case name.length < 3 || name.length > 63:
            this.errorMessage = 'Name must be not less than 3 and not more than 63 characters length';

            return false;
        case !Validator.bucketName(name):
            this.errorMessage = 'Name must contain only lowercase latin characters, numbers, a hyphen or a period';

            return false;

        default:
            return true;
        }
    }
}
</script>

<style scoped lang="scss">
    .buckets-view {
        display: flex;
        flex-direction: column;
        align-items: center;
        font-family: 'font_regular', sans-serif;
        font-style: normal;
        background-color: #f5f6fa;

        &__title-area {
            width: 100%;
            display: flex;
            justify-content: space-between;
            align-items: center;

            &__title {
                font-family: 'font_medium', sans-serif;
                font-weight: bold;
                font-size: 18px;
                line-height: 26px;
                color: #232b34;
                margin: 0;
                width: 100%;
                text-align: left;
            }

            &__button {
                width: 154px;
                height: 46px;
                display: flex;
                align-items: center;
                justify-content: center;
                background-color: #0068dc;
                border-radius: 4px;
                cursor: pointer;

                &__label {
                    font-weight: normal;
                    font-size: 12px;
                    line-height: 17px;
                    color: #fff;
                    margin: 0 0 0 5px;
                }

                &:hover {
                    background-color: #0000c2;
                }
            }
        }

        &__loader {
            margin-top: 100px;
        }

        &__no-buckets {
            width: 100%;
            text-align: center;
            font-size: 30px;
            line-height: 42px;
            margin: 100px 0 0;
        }

        &__list {
            margin-top: 40px;
            width: 100%;
            display: flex;
            flex-direction: column;
            overflow: hidden;
            padding-bottom: 100px;

            &__sorting-header {
                display: flex;
                align-items: center;
                padding: 0 20px 5px;
                width: calc(100% - 40px);
                font-weight: bold;
                font-size: 14px;
                line-height: 20px;
                color: #768394;
                border-bottom: 1px solid rgb(169 181 193 / 40%);

                &__name {
                    width: calc(70% - 16px);
                    margin: 0;
                }

                &__date {
                    width: 30%;
                    margin: 0;
                }

                &__empty {
                    margin: 0;
                }
            }
        }
    }

    .disabled {
        pointer-events: none;
        background-color: #dadde5;
        border-color: #dadde5;
    }
</style>
