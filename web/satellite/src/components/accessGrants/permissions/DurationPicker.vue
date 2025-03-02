// Copyright (C) 2020 Storj Labs, Inc.
// See LICENSE for copying information.

<template>
    <div v-click-outside="closePicker" :class="`duration-picker${containerStyle}`">
        <div class="duration-picker__list">
            <ul class="duration-picker__list__column">
                <li class="duration-picker__list__column-item" @click="onForeverClick">Forever</li>
                <li class="duration-picker__list__column-item" @click="onOneDayClick">24 Hours</li>
                <li class="duration-picker__list__column-item" @click="onOneWeekClick">1 Week</li>
            </ul>
            <ul class="duration-picker__list__column">
                <li class="duration-picker__list__column-item" @click="onOneMonthClick">1 month</li>
                <li class="duration-picker__list__column-item" @click="onSixMonthsClick">6 Months</li>
                <li class="duration-picker__list__column-item" @click="onOneYearClick">1 Year</li>
            </ul>
        </div>
        <hr class="duration-picker__break">
        <div class="duration-picker__wrapper">
            <VDateRangePicker :on-date-pick="onCustomRangePick" :is-open="true" />
        </div>
    </div>
</template>

<script lang="ts">
import { Component, Vue, Prop } from 'vue-property-decorator';

import { ACCESS_GRANTS_ACTIONS } from '@/store/modules/accessGrants';
import { APP_STATE_ACTIONS } from "@/utils/constants/actionNames";
import { DurationPermission } from '@/types/accessGrants';

import VDateRangePicker from "@/components/common/VDateRangePicker.vue";

// @vue/component
@Component({
    components: {
        VDateRangePicker,
    },
})
export default class DurationPicker extends Vue {
    @Prop({default: ''})
    private readonly containerStyle: string;
    /**
     * onCustomRangePick holds logic for choosing custom date range.
     * @param dateRange
     */
    public onCustomRangePick(dateRange: Date[]): void {
        const before = dateRange[0];
        const after = new Date(dateRange[1].getFullYear(), dateRange[1].getMonth(), dateRange[1].getDate(), 23, 59, 59);

        const permission: DurationPermission = new DurationPermission(before, after);
        const fromFormattedString = before.toLocaleDateString('en-US', { month: 'short', day: 'numeric', year: '2-digit' });
        const toFormattedString = after.toLocaleDateString('en-US', { month: 'short', day: 'numeric', year: '2-digit' });
        const rangeLabel = `${fromFormattedString} - ${toFormattedString}`;

        this.$store.dispatch(ACCESS_GRANTS_ACTIONS.SET_DURATION_PERMISSION, permission);
        this.$emit('setLabel', rangeLabel);
    }

    /**
     * Holds on "forever" choice click logic.
     */
    public onForeverClick(): void {
        const permission = new DurationPermission(null, null);

        this.$store.dispatch(ACCESS_GRANTS_ACTIONS.SET_DURATION_PERMISSION, permission);
        this.$emit('setLabel', 'Forever');
        this.closePicker();
    }

    /**
     * Holds on "1 month" choice click logic.
     */
    public onOneMonthClick(): void {
        const now = new Date();
        const inAMonth = new Date(now.setMonth(now.getMonth() + 1));
        const permission = new DurationPermission(new Date(), inAMonth);

        this.$store.dispatch(ACCESS_GRANTS_ACTIONS.SET_DURATION_PERMISSION, permission);
        this.$emit('setLabel', '1 Month');
        this.closePicker();
    }

    /**
     * Holds on "24 hours" choice click logic.
     */
    public onOneDayClick(): void {
        const now = new Date();
        const inADay = new Date(now.setDate(now.getDate() + 1));
        const permission = new DurationPermission(new Date(), inADay);

        this.$store.dispatch(ACCESS_GRANTS_ACTIONS.SET_DURATION_PERMISSION, permission);
        this.$emit('setLabel', '24 Hours');
        this.closePicker();
    }

    /**
     * Holds on "1 week" choice click logic.
     */
    public onOneWeekClick(): void {
        const now = new Date();
        const inAWeek = new Date(now.setDate(now.getDate() + 7));
        const permission = new DurationPermission(new Date(), inAWeek);

        this.$store.dispatch(ACCESS_GRANTS_ACTIONS.SET_DURATION_PERMISSION, permission);
        this.$emit('setLabel', '1 Week');
        this.closePicker();
    }

    /**
     * Holds on "6 month" choice click logic.
     */
    public onSixMonthsClick(): void {
        const now = new Date();
        const inSixMonth = new Date(now.setMonth(now.getMonth() + 6));
        const permission = new DurationPermission(new Date(), inSixMonth);

        this.$store.dispatch(ACCESS_GRANTS_ACTIONS.SET_DURATION_PERMISSION, permission);
        this.$emit('setLabel', '6 Months');
        this.closePicker();
    }

    /**
     * Holds on "1 year" choice click logic.
     */
    public onOneYearClick(): void {
        const now = new Date();
        const inOneYear = new Date(now.setFullYear(now.getFullYear() + 1));
        const permission = new DurationPermission(new Date(), inOneYear);

        this.$store.dispatch(ACCESS_GRANTS_ACTIONS.SET_DURATION_PERMISSION, permission);
        this.$emit('setLabel', '1 Year');
        this.closePicker();
    }

    /**
     * Closes duration picker.
     */
    public closePicker(): void {
        this.$store.dispatch(APP_STATE_ACTIONS.CLOSE_POPUPS);
    }
}
</script>

<style scoped lang="scss">
    @mixin date-container {
        background: #fff;
        width: 600px;
        border: 1px solid #384b65;
        border-radius: 6px;
        margin: 0 auto;
        box-shadow: 0 4px 8px 0 rgb(0 0 0 / 20%), 0 6px 20px 0 rgb(0 0 0 / 19%);
        position: absolute;
        z-index: 1;
        top: 100%;
    }

    .duration-picker {
        @include date-container;

        right: 0;

        &__access-date-container {
            @include date-container;

            right: -88%;
        }

        &__list {
            column-count: 2;
            column-gap: 48px;
            padding: 10px 24px 0;

            &__column {
                list-style-type: none;
                padding-left: 0;
                margin-top: 0;
            }

            &__column-item {
                font-size: 14px;
                font-weight: 400;
                padding: 10px 0 10px 12px;
                border-left: 7px solid #fff;
                color: #1b2533;

                &:hover {
                    font-weight: bold;
                    background: #f5f6fa;
                    border-left: 6px solid #2582ff;
                    cursor: pointer;
                }
            }
        }

        &__break {
            width: 84%;
            margin: 10px auto;
        }

        &__wrapper {
            width: 100%;
        }
    }
</style>
