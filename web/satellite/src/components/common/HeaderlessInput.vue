// Copyright (C) 2019 Storj Labs, Inc.
// See LICENSE for copying information.

<template>
    <div class="input-wrap" :aria-roledescription="roleDescription">
        <div class="label-container">
            <ErrorIcon v-if="error" />
            <p v-if="isLabelShown" class="label-container__label" :style="style.labelStyle">{{ label }}</p>
            <p v-if="error" class="label-container__error" aria-roledescription="error-text" :style="style.errorStyle">{{ error }}</p>
        </div>
        <input
            v-model="value"
            class="headerless-input"
            :class="{'inputError' : error, 'password': isPassword}"
            :placeholder="placeholder"
            :type="type"
            :style="style.inputStyle"
            :optionsShown="optionsShown"
            @input="onInput"
            @change="onInput"
            @focus="showPasswordStrength"
            @blur="hidePasswordStrength"
            @click="showOptions"
            @optionsList="optionsList"
        >

        <!-- Shown if there are input choice options  -->
        <InputCaret v-if="optionsList.length > 0" class="headerless-input__caret" />
        <ul v-if="optionsShown" v-click-outside="hideOptions" class="headerless-input__options-wrapper">
            <li
                v-for="(option, index) in optionsList"
                :key="index"
                class="headerless-input__option"
                @click="chooseOption(option)"
            >
                {{ option }}
            </li>
        </ul>
        <!-- end of option render logic-->

        <!--2 conditions of eye image (crossed or not) -->
        <PasswordHiddenIcon
            v-if="isPasswordHiddenState"
            class="input-wrap__image"
            @click="changeVision"
        />
        <PasswordShownIcon
            v-if="isPasswordShownState"
            class="input-wrap__image"
            @click="changeVision"
        />
        <!-- end of image-->
    </div>
</template>

<script lang="ts">
import { Component, Prop, Vue } from 'vue-property-decorator';

import InputCaret from '@/../static/images/common/caret.svg';
import PasswordHiddenIcon from '@/../static/images/common/passwordHidden.svg';
import PasswordShownIcon from '@/../static/images/common/passwordShown.svg';
import ErrorIcon from '@/../static/images/register/ErrorInfo.svg';

// Custom input component for login page
// @vue/component
@Component({
    components: {
        InputCaret,
        ErrorIcon,
        PasswordHiddenIcon,
        PasswordShownIcon,
    },
})
export default class HeaderlessInput extends Vue {
    private readonly textType: string = 'text';
    private readonly passwordType: string = 'password';

    private type: string = this.textType;
    private isPasswordShown = false;

    public value: string;

    @Prop({default: ''})
    protected readonly initValue: string;

    @Prop({default: ''})
    protected readonly label: string;
    @Prop({default: 'default'})
    protected readonly placeholder: string;
    @Prop({default: false})
    protected readonly isPassword: boolean;
    @Prop({default: '48px'})
    protected readonly height: string;
    @Prop({default: '100%'})
    protected readonly width: string;
    @Prop({default: ''})
    protected readonly error: string;
    @Prop({default: Number.MAX_SAFE_INTEGER})
    protected readonly maxSymbols: number;
    @Prop({default: () => []})
    protected readonly optionsList: string[];
    @Prop({default: false})
    protected optionsShown: boolean;
    @Prop({default: false})
    protected inputClicked: boolean;

    @Prop({default: false})
    private readonly isWhite: boolean;
    @Prop({default: false})
    private readonly withIcon: boolean;
    @Prop({default: 'input-container'})
    private readonly roleDescription: boolean;

    public created() {
        this.type = this.isPassword ? this.passwordType : this.textType;
        this.value = this.initValue;
    }

    /**
     * Used to set default value from parent component.
     * @param value
     */
    public setValue(value: string): void {
        this.value = value;
    }

    public showPasswordStrength(): void {
        this.$emit('showPasswordStrength');
    }

    public hidePasswordStrength(): void {
        this.$emit('hidePasswordStrength');
    }

    /**
     * triggers on input.
     */
    public onInput(event: Event): void {
        const target = event.target as HTMLInputElement;
        if (target.value.length > this.maxSymbols) {
            this.value = target.value.slice(0, this.maxSymbols);
        } else {
            this.value = target.value;
        }

        this.$emit('setData', this.value);
    }

    /**
     * Triggers input type between text and password to show/hide symbols.
     */
    public changeVision(): void {
        this.isPasswordShown = !this.isPasswordShown;
        this.type = this.isPasswordShown ? this.textType : this.passwordType;
    }

    /**
     * Chose a dropdown option as the input value.
     */
    public chooseOption(option: string): void {
        this.value = option;
        this.$emit('setData', this.value);
        this.optionsShown = false;
    }

    /**
     * Show dropdown options when the input is clicked, if they exist.
     */
    public showOptions(): void {
        if (this.optionsList.length > 0) {
            this.optionsShown = true;
            this.inputClicked = true;
        }
    }

    /**
     * Hide the dropdown options from view when there is a click outside of the dropdown.
     */
    public hideOptions(): void {
        if (this.optionsList.length > 0 && !this.inputClicked && this.optionsShown) {
            this.optionsShown = false;
            this.inputClicked = false;
        }
        this.inputClicked = false;
    }

    public get isLabelShown(): boolean {
        return !!(!this.error && this.label);
    }

    public get isPasswordHiddenState(): boolean {
        return this.isPassword && !this.isPasswordShown;
    }

    public get isPasswordShownState(): boolean {
        return this.isPassword && this.isPasswordShown;
    }

    /**
     * Returns style objects depends on props.
     */
    protected get style(): Record<string, unknown> {
        return {
            inputStyle: {
                width: this.width,
                height: this.height,
                padding: this.withIcon ? '0 30px 0 50px' : '',
            },
            labelStyle: {
                color: this.isWhite ? 'white' : '#354049',
            },
            errorStyle: {
                color: this.isWhite ? 'white' : '#FF5560',
            },
        };
    }
}
</script>

<style scoped lang="scss">
    .input-wrap {
        position: relative;
        width: 100%;
        font-family: 'font_regular', sans-serif;

        &__image {
            position: absolute;
            right: 25px;
            bottom: 5px;
            transform: translateY(-50%);
            z-index: 20;
            cursor: pointer;

            &:hover .input-wrap__image__path {
                fill: #2683ff !important;
            }
        }

        .headerless-input {
            font-size: 16px;
            line-height: 21px;
            resize: none;
            height: 46px;
            padding: 0 30px 0 0;
            text-indent: 20px;
            border: 1px solid rgb(56 75 101 / 40%);
            border-radius: 6px;
            box-sizing: border-box;

            &__caret {
                position: absolute;
                right: 28px;
                bottom: 18px;
            }

            &__options-wrapper {
                border: 1px solid rgb(56 75 101 / 40%);
                position: absolute;
                width: calc(100% - 5px);
                top: 70px;
                padding: 0;
                background: #fff;
                z-index: 21;
                border-radius: 0 0 6px 6px;
                list-style: none;
                border-top: none;
                height: 176px;
                margin-top: 0;
            }

            &__option {
                cursor: pointer;
                padding: 20px 22px;

                &:hover {
                    background: #2582ff;
                    color: #fff;
                }
            }
        }

        .headerless-input::placeholder {
            color: #384b65;
            opacity: 0.4;
        }

        &:focus-within {

            .headerless-input {
                position: relative;
                z-index: 22;

                &__options-wrapper {
                    border-top: 3px solid #145ecc;
                }

                &__caret {
                    z-index: 23;
                }
            }
        }
    }

    .label-container {
        display: flex;
        justify-content: flex-start;
        align-items: flex-end;
        padding-bottom: 8px;
        flex-direction: row;

        &__label {
            font-size: 16px;
            line-height: 21px;
            color: #354049;
            margin-bottom: 0;
        }

        &__add-label {
            margin-left: 5px;
            font-size: 16px;
            line-height: 21px;
            color: rgb(56 75 101 / 40%);
        }

        &__error {
            font-size: 16px;
            margin: 18px 0 0 10px;
        }
    }

    .inputError::placeholder {
        color: #eb5757;
        opacity: 0.4;
    }

    .error {
        color: #ff5560;
        margin-left: 10px;
    }

    .password {
        padding-right: 75px;
    }
</style>
