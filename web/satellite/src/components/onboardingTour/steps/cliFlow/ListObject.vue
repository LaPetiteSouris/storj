// Copyright (C) 2021 Storj Labs, Inc.
// See LICENSE for copying information.

<template>
    <CLIFlowContainer
        :on-back-click="onBackClick"
        :on-next-click="onNextClick"
        title="Listing a bucket"
    >
        <template #icon>
            <Icon />
        </template>
        <template #content class="list-object">
            <p class="list-object__msg">
                To view the cheesecake photo in our bucket, let's use the list command:
            </p>
            <OSContainer>
                <template #windows>
                    <TabWithCopy value="./uplink.exe ls sj://cakes" aria-role-description="windows-list" />
                </template>
                <template #linux>
                    <TabWithCopy value="uplink ls sj://cakes" aria-role-description="linux-list" />
                </template>
                <template #macos>
                    <TabWithCopy value="uplink ls sj://cakes" aria-role-description="macos-list" />
                </template>
            </OSContainer>
        </template>
    </CLIFlowContainer>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator';

import { RouteConfig } from "@/router";

import CLIFlowContainer from "@/components/onboardingTour/steps/common/CLIFlowContainer.vue";
import OSContainer from "@/components/onboardingTour/steps/common/OSContainer.vue";
import TabWithCopy from "@/components/onboardingTour/steps/common/TabWithCopy.vue";

import Icon from "@/../static/images/onboardingTour/listObjectStep.svg";

// @vue/component
@Component({
    components: {
        CLIFlowContainer,
        Icon,
        OSContainer,
        TabWithCopy,
    }
})
export default class ListObject extends Vue {
    /**
     * Holds on back button click logic.
     */
    public async onBackClick(): Promise<void> {
        await this.$router.push(RouteConfig.OnboardingTour.with(RouteConfig.OnbCLIStep.with(RouteConfig.UploadObject)).path);
    }

    /**
     * Holds on next button click logic.
     */
    public async onNextClick(): Promise<void> {
        await this.$router.push(RouteConfig.OnboardingTour.with(RouteConfig.OnbCLIStep.with(RouteConfig.DownloadObject)).path);
    }
}
</script>

<style scoped lang="scss">
    .list-object {
        font-family: 'font_regular', sans-serif;

        &__msg {
            font-size: 16px;
            line-height: 24px;
            color: #1b2533;
        }
    }
</style>
