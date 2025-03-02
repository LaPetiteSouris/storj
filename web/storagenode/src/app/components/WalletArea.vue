// Copyright (C) 2019 Storj Labs, Inc.
// See LICENSE for copying information.

<template>
    <div class="wallet-area" :class="{'with-wallet-features': isZkSyncEnabled}">
        <WalletIcon
            class="wallet-area__image"
            alt="wallet image"
        />
        <div class="wallet-area__wallet-address-section">
            <p class="wallet-area__wallet-address-section__label">{{ label }}</p>
            <p class="wallet-area__wallet-address-section__bold-text">{{ walletAddress }}</p>
        </div>
        <a
            v-if="!isZkSyncEnabled"
            class="wallet-area__button"
            :href="`https://etherscan.io/address/${walletAddress}#tokentxns`"
            target="_blank"
            rel="noopener noreferrer"
        >
            <b class="wallet-area-button-label">View on Etherscan</b>
        </a>
        <div v-else class="wallet-area__buttons-area">
            <a
                class="wallet-area__button"
                :href="`https://zkscan.io/explorer/accounts/${walletAddress}`"
                target="_blank"
                rel="noopener noreferrer"
            >
                <b class="wallet-area-button-label">View on zkScan</b>
            </a>
            <a
                class="wallet-area__button"
                :href="`https://etherscan.io/address/${walletAddress}#tokentxns`"
                target="_blank"
                rel="noopener noreferrer"
            >
                <b class="wallet-area-button-label">View on Etherscan</b>
            </a>
            <div class="wallet-area__buttons-area__active-wallet-area">
                <CheckIcon class="wallet-area__buttons-area__active-wallet-area__icon" />
                <p class="wallet-area__buttons-area__active-wallet-area__label">zkSync is opted-in</p>
            </div>
        </div>
    </div>
</template>

<script lang="ts">
import { Component, Prop, Vue } from 'vue-property-decorator';

import CheckIcon from '@/../static/images/common/greenCheck.svg';
import WalletIcon from '@/../static/images/wallet.svg';

// @vue/component
@Component({
    components: {
        CheckIcon,
        WalletIcon,
    },
})
export default class WalletArea extends Vue {
    @Prop({default: ''})
    private readonly label: string;
    @Prop({default: ''})
    private readonly walletAddress: string;
    @Prop({default: () => []})
    private readonly walletFeatures: string[];

    public get isZkSyncEnabled(): boolean {
        return this.walletFeatures.includes('zksync');
    }
}
</script>

<style scoped lang="scss">
    .wallet-area {
        background-color: var(--block-background-color);
        padding: 40px 30px;
        display: flex;
        justify-content: space-between;
        align-items: center;
        border: 1px solid var(--block-border-color);
        border-radius: 12px;
        position: relative;

        &__image {
            margin-right: 40px;
        }

        &__wallet-address-section {
            height: auto;
            display: flex;
            flex-direction: column;
            justify-content: space-between;
            margin-right: 20px;

            &__label {
                font-size: 14px;
                color: var(--regular-text-color);
            }

            &__bold-text {
                font-family: 'font_bold', sans-serif;
                font-size: 18px;
                color: var(--regular-text-color);
                word-break: break-all;
            }
        }

        &__button {
            font-size: 14px;
            width: 168px;
            height: 44px;
            display: flex;
            align-items: center;
            justify-content: center;
            background-color: var(--button-background-color);
            border: 1px solid var(--block-border-color);
            border-radius: 12px;
            color: var(--regular-text-color);
            text-decoration: none;

            &:hover {
                background-color: #4d72b7;
                cursor: pointer;

                .wallet-area-button-label {
                    color: #fff;
                }
            }
        }

        &__buttons-area {
            display: flex;
            align-items: center;
            justify-content: flex-start;
            margin-top: 10px;

            & .wallet-area__button {
                margin-right: 12px;
            }

            &__active-wallet-area {
                display: flex;
                align-items: center;
                justify-content: flex-start;

                &__icon {
                    background: white;
                    border-radius: 50%;

                    path {
                        fill: var(--wallet-feature-opted-in);
                    }
                }

                &__label {
                    font-family: 'font_semiBold', sans-serif;
                    font-size: 14px;
                    line-height: 17px;
                    margin-left: 7.5px;
                    color: var(--wallet-feature-opted-in);
                }
            }
        }
    }

    .with-wallet-features {
        flex-direction: column;
        justify-content: flex-start;
        align-items: flex-start;

        & .wallet-area__wallet-address-section {
            margin: 15px 0;
        }
    }

    @media screen and (max-width: 1000px) {

        .wallet-area {
            flex-direction: column;
            justify-content: flex-start;
            align-items: flex-start;
            padding-bottom: 25px;

            &__wallet-address-section {
                margin-bottom: 20px;

                &__label {
                    margin: 20px 0 6px;
                }
            }
        }
    }

    @media screen and (max-width: 500px) {

        p {
            margin: 0;
        }

        .wallet-area {

            &__wallet-address-section {
                margin-bottom: 10px;

                &__label {
                    margin: 20px 0 6px;
                }
            }

            &__buttons-area {
                flex-direction: column;
                width: 100%;

                & .wallet-area__button {
                    margin: 0 0 15px;
                    width: 100%;
                }

                &__active-wallet-area {
                    margin-top: 10px;
                }
            }
        }
    }
</style>
