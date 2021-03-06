<template>
  <q-input-frame
    class="q-input"

    :prefix="prefix"
    :suffix="suffix"
    :stack-label="stackLabel"
    :float-label="floatLabel"
    :error="error"
    :warning="warning"
    :disable="disable"
    :inverted="inverted"
    :dark="dark"
    :before="before"
    :after="after"
    :color="color"

    :focused="focused"
    :length="length"
    :top-addons="isTextarea"

    @click="__onClick"
    @focus="__onFocus"
  >
    <slot name="before"></slot>

    <template v-if="isTextarea">
      <div class="col row relative-position">
        <q-resize-observable @resize="__updateArea()"></q-resize-observable>
        <textarea
          class="col q-input-target q-input-shadow absolute-top"
          ref="shadow"
          :value="model"
          :rows="minRows"
        ></textarea>

        <textarea
          ref="input"
          class="col q-input-target q-input-area"

          :name="name"
          :placeholder="inputPlaceholder"
          :disabled="disable"
          :readonly="readonly"
          :maxlength="maxLength"
          :rows="minRows"
          v-bind="attributes"

          :value="model"
          @input="__set"

          @focus="__onFocus"
          @blur="__onInputBlur"
          @keydown="__onKeydown"
          @keyup="__onKeyup"
        ></textarea>
      </div>
    </template>

    <input
      v-else
      ref="input"
      class="col q-input-target"
      :class="[`text-${align}`]"

      :name="name"
      :placeholder="inputPlaceholder"
      :pattern="pattern"
      :disabled="disable"
      :readonly="readonly"
      :maxlength="maxLength"
      v-bind="attributes"

      :min="min"
      :max="max"
      :step="inputStep"

      :type="inputType"
      :value="model"
      @input="__set"

      @focus="__onFocus"
      @blur="__onInputBlur"
      @keydown="__onKeydown"
      @keyup="__onKeyup"
    />

    <q-icon
      v-if="isPassword && !noPassToggle && length"
      slot="after"
      :name="showPass ? 'visibility' : 'visibility_off'"
      class="q-if-control"
      @mousedown="__clearTimer"
      @touchstart="__clearTimer"
      @click="togglePass"
    ></q-icon>

    <q-icon
      v-if="editable && clearable && length"
      slot="after"
      name="cancel"
      class="q-if-control"
      @mousedown="__clearTimer"
      @touchstart="__clearTimer"
      @click="clear"
    ></q-icon>

    <q-spinner
      v-if="isLoading"
      slot="after"
      size="24px"
      class="q-if-control"
    ></q-spinner>

    <slot name="after"></slot>
    <slot></slot>
  </q-input-frame>
</template>

<script>
import FrameMixin from '../../mixins/input-frame'
import InputMixin from '../../mixins/input'
import inputTypes from './input-types'
import { frameDebounce } from '../../utils/debounce'
import { between } from '../../utils/format'
import { QInputFrame } from '../input-frame'
import { QResizeObservable } from '../observables'
import { QSpinner } from '../spinner'

export default {
  name: 'q-input',
  mixins: [FrameMixin, InputMixin],
  components: {
    QInputFrame,
    QSpinner,
    QResizeObservable
  },
  props: {
    value: { required: true },
    type: {
      type: String,
      default: 'text',
      validator: t => inputTypes.includes(t)
    },
    minRows: Number,
    clearable: Boolean,
    noPassToggle: Boolean,
    readonly: Boolean,
    attributes: Object,

    min: Number,
    max: Number,
    step: {
      type: Number,
      default: 1
    },
    maxDecimals: Number
  },
  data () {
    return {
      showPass: false,
      model: this.value,
      shadow: {
        val: this.model,
        set: this.__set,
        loading: false,
        hasFocus: () => {
          return document.activeElement === this.$refs.input
        },
        register: () => {
          this.watcher = this.$watch('model', val => {
            this.shadow.val = val
          })
        },
        unregister: () => {
          this.watcher()
        },
        getEl: () => {
          return this.$refs.input
        }
      }
    }
  },
  watch: {
    value (v) {
      this.model = v
    }
  },
  provide () {
    return {
      __input: this.shadow
    }
  },
  computed: {
    isNumber () {
      return this.type === 'number'
    },
    isPassword () {
      return this.type === 'password'
    },
    isTextarea () {
      return this.type === 'textarea'
    },
    isLoading () {
      return this.loading || this.shadow.loading
    },
    pattern () {
      if (this.isNumber) {
        return '[0-9]*'
      }
    },
    inputStep () {
      if (this.isNumber) {
        return this.step
      }
    },
    inputType () {
      return this.isPassword
        ? (this.showPass ? 'text' : 'password')
        : this.type
    },
    length () {
      return this.model !== null && this.model !== undefined
        ? ('' + this.model).length
        : 0
    },
    editable () {
      return !this.disable && !this.readonly
    }
  },
  methods: {
    togglePass () {
      this.showPass = !this.showPass
      clearTimeout(this.timer)
      this.focus()
    },
    clear () {
      clearTimeout(this.timer)
      this.focus()
      if (this.editable) {
        this.model = this.isNumber ? null : ''
        this.$emit('input', this.model)
      }
    },

    __clearTimer () {
      this.$nextTick(() => clearTimeout(this.timer))
    },

    __set (e) {
      let val = e.target ? e.target.value : e
      if (this.isNumber) {
        val = parseFloat(val)
        if (isNaN(val)) {
          // Number.isInteger(this.maxDecimals)
          val = null
        }
        else if (Number.isInteger(this.maxDecimals)) {
          val = parseFloat(val.toFixed(this.maxDecimals))
        }
      }
      if (val !== this.model) {
        this.$emit('input', val)
      }
      this.model = val
    },
    __updateArea () {
      const shadow = this.$refs.shadow
      if (shadow) {
        let h = shadow.scrollHeight
        const max = this.maxHeight || h
        this.$refs.input.style.minHeight = `${between(h, 19, max)}px`
      }
    }
  },
  mounted () {
    this.__updateArea = frameDebounce(this.__updateArea)
    if (this.isTextarea) {
      this.__updateArea()
      this.watcher = this.$watch('model', this.__updateArea)
    }
  },
  beforeDestroy () {
    if (this.watcher !== void 0) {
      this.watcher()
    }
  }
}
</script>
