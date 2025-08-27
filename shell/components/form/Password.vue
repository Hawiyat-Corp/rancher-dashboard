<script>
import { mapGetters } from 'vuex';
import { LabeledInput } from '@components/Form/LabeledInput';
import { CHARSET, randomStr } from '@shell/utils/string';
import { copyTextToClipboard } from '@shell/utils/clipboard';
import { _CREATE } from '@shell/config/query-params';

export default {
  emits: ['update:value', 'blur'],

  components: { LabeledInput },
  props:      {
    value: {
      default: '',
      type:    String,
    },
    isRandom: {
      default: false,
      type:    Boolean,
    },
    label: {
      default: '',
      type:    String,
    },
    name: {
      default: '',
      type:    String
    },
    autocomplete: {
      type:    String,
      default: ''
    },
    required: {
      default: false,
      type:    Boolean,
    },
    ignorePasswordManagers: {
      default: false,
      type:    Boolean,
    },
    mode: {
      type:    String,
      default: _CREATE,
    }
  },
  data() {
    return { reveal: false };
  },
  computed: {
    ...mapGetters({ t: 'i18n/t' }),
    password: {
      get() {
        return this.value;
      },
      set(val) {
        this.$emit('update:value', val);
      }
    },
    attributes() {
      const attributes = { };

      if (this.name) {
        attributes.id = this.name;
        attributes.name = this.name;
      }
      if (this.autocomplete) {
        attributes.autocomplete = this.autocomplete;
      }

      return attributes;
    },
    hideShowLabel() {
      return this.reveal ? this.t('action.hide') : this.t('action.show');
    }
  },
  watch: {
    isRandom() {
      if (this.isRandom) {
        this.generatePassword();
      }
    }
  },
  created() {
    if (this.isRandom) {
      this.generatePassword();
    }
  },
  methods: {
    copyTextToClipboard,
    generatePassword() {
      this.password = randomStr(16, CHARSET.ALPHA_NUM);
    },
    show(reveal) {
      this.reveal = reveal;
    },
    focus() {
      this.$refs.input.$refs.value.focus();
    },
    hideShowFn() {
      this.reveal ? this.reveal = false : this.reveal = true;
    }
  }
};
</script>

<template>
  <div class="password">
    <LabeledInput
      ref="input"
      v-model:value="password"
      v-bind="attributes"
      :type="isRandom || reveal ? 'text' : 'password'"
      :readonly="isRandom"
      :label="label"
      :required="required"
      :disabled="isRandom"
      :ignore-password-managers="ignorePasswordManagers"
      :mode="mode"
      @blur="$emit('blur', $event)"
    >
      <template #suffix>
        <div
          v-if="isRandom"
          class="addon"
        >
          <a
            href="#"
            @click.prevent.stop="copyTextToClipboard(password)"
          >{{ t('action.copy') }}</a>
        </div>
        <div
          v-else
          class="addon"
        >
          <a
            href="#"
            tabindex="0"
            class="hide-show"
            role="button"
            :aria-label="reveal ? t('action.ariaLabel.hidePass', { area: label }) : t('action.ariaLabel.showPass', { area: label })"
            @click.prevent.stop="hideShowFn"
            @keyup.space.prevent.stop="hideShowFn"
          >
            <i :class="['eye-icon', reveal ? 'revealed' : 'hidden']" />
          </a>
        </div>
      </template>
    </LabeledInput>
    <div
      v-if="isRandom"
      class="mt-10 genPassword"
    >
      <a
        href="#"
        @click.prevent.stop="generatePassword"
      ><i class="icon icon-refresh" /> {{ t('changePassword.newGeneratedPassword') }}</a>
    </div>
  </div>
</template>

<style lang="scss" scoped>
  .password {
    display: flex;
    flex-direction: column;

    .labeled-input {
      .addon {
        display: flex;
        align-items: center;
        justify-content: center;
        padding-left: 12px;
        min-width: 44px;

        .hide-show:focus-visible {
          @include focus-outline;
          outline-offset: 4px;
        }

        .eye-icon {
          width: 20px;
          height: 20px;
          display: inline-block;
          background-repeat: no-repeat;
          background-position: center;
        }
        .eye-icon.hidden {
          /* outline eye */
          background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='20' height='20' viewBox='0 0 24 24' fill='none' stroke='%23E5ECF6' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'%3E%3Cpath d='M1 12s4-7 11-7 11 7 11 7-4 7-11 7S1 12 1 12z'/%3E%3Ccircle cx='12' cy='12' r='3'/%3E%3C/svg%3E");
        }
        .eye-icon.revealed {
          /* eye-off */
          background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='20' height='20' viewBox='0 0 24 24' fill='none' stroke='%23E5ECF6' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'%3E%3Cpath d='M17.94 17.94A10.94 10.94 0 0 1 12 20c-7 0-11-8-11-8a21.77 21.77 0 0 1 5.06-6.94'/%3E%3Cpath d='M1 1l22 22'/%3E%3Cpath d='M9.9 4.24A10.94 10.94 0 0 1 12 4c7 0 11 8 11 8a21.77 21.77 0 0 1-3.34 4.58'/%3E%3Cpath d='M14.12 14.12A3 3 0 0 1 9.88 9.88'/%3E%3C/svg%3E");
        }
      }
    }
    .genPassword {
      display: flex;
      justify-content: flex-end;
    }
  }

</style>
