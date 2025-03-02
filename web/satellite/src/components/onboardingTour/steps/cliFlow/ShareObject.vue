// Copyright (C) 2021 Storj Labs, Inc.
// See LICENSE for copying information.

<template>
    <CLIFlowContainer
        :on-back-click="onBackClick"
        :on-next-click="onNextClick"
        title="Share a link with the world"
    >
        <template #icon>
            <img class="image" src="@/../static/images/onboardingTour/shareStep.png" alt="share">
        </template>
        <template #content class="share-object">
            <p class="share-object__msg">
                You can generate a shareable URL and view the geographic distribution of your object via the Link
                Sharing Service. Run the uplink share --url command.
            </p>
            <OSContainer>
                <template #windows>
                    <TabWithCopy value="./uplink.exe share --url sj://cakes/cheesecake.jpg" aria-role-description="windows-share" />
                </template>
                <template #linux>
                    <TabWithCopy value="uplink share --url sj://cakes/cheesecake.jpg" aria-role-description="linux-share" />
                </template>
                <template #macos>
                    <TabWithCopy value="uplink share --url sj://cakes/cheesecake.jpg" aria-role-description="macos-share" />
                </template>
            </OSContainer>
            <p class="share-object__msg">
                Copy the URL that is returned by the
                <b class="share-object__msg__bold">uplink share --url</b>
                command and paste into your browser window.
            </p>
            <p class="share-object__msg">
                You will see your file and a map with real distribution of your files' pieces uploaded to the network.
                You can share it with anyone you'd like.
            </p>
        </template>
    </CLIFlowContainer>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator';

import { RouteConfig } from "@/router";

import CLIFlowContainer from "@/components/onboardingTour/steps/common/CLIFlowContainer.vue";
import OSContainer from "@/components/onboardingTour/steps/common/OSContainer.vue";
import TabWithCopy from "@/components/onboardingTour/steps/common/TabWithCopy.vue";

// @vue/component
@Component({
    components: {
        CLIFlowContainer,
        OSContainer,
        TabWithCopy,
    }
})
export default class ShareObject extends Vue {
    /**
     * Holds on back button click logic.
     */
    public async onBackClick(): Promise<void> {
        await this.$router.push(RouteConfig.OnboardingTour.with(RouteConfig.OnbCLIStep.with(RouteConfig.DownloadObject)).path);
    }

    /**
     * Holds on next button click logic.
     */
    public async onNextClick(): Promise<void> {
        await this.$router.push(RouteConfig.OnboardingTour.with(RouteConfig.OnbCLIStep.with(RouteConfig.SuccessScreen)).path);
    }
}
</script>

<style scoped lang="scss">
    .share-object {
        font-family: 'font_regular', sans-serif;

        &__msg {
            font-size: 16px;
            line-height: 24px;
            color: #1b2533;

            &:nth-of-type(2),
            &:last-of-type {
                margin-top: 20px;
            }
        }
    }

    .image {
        max-width: 267px;
        max-height: 90px;
    }
</style>
