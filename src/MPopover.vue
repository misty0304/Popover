<template>
  <component :is="tagName">
    <transition
      :name="transition"
      :enter-active-class="enterActiveClass"
      :leave-active-class="leaveActiveClass"
      @after-leave="doDestroy"
    >
      <span ref="popper" v-show="!disabled && showPopper">
        <slot>{{ content }}</slot>
      </span>
    </transition>
    <slot name="reference"></slot>
  </component>
</template>

<script>
import Popper from 'popper.js'
function on(element, event, handler) {
  if (element && event && handler) {
    document.addEventListener
      ? element.addEventListener(event, handler, false)
      : element.attachEvent('on' + event, handler)
  }
}
function off(element, event, handler) {
  if (element && event) {
    document.removeEventListener
      ? element.removeEventListener(event, handler, false)
      : element.detachEvent('on' + event, handler)
  }
}
export default {
  name: 'MPopover',
  props: {
    // 是否显示popover
    showPopover: {
      type: Boolean,
      default() {
        return false
      }
    },
    tagName: {
      type: String,
      default: 'span'
    },
    trigger: {
      type: String,
      default: 'hover',
      validator: value =>
        [
          'clickToOpen',
          'click',
          'clickToToggle',
          'hover',
          'focus'
        ].indexOf(value) > -1
    },
    delayOnMouseOver: {
      type: Number,
      default: 10
    },
    delayOnMouseOut: {
      type: Number,
      default: 10
    },
    disabled: {
      type: Boolean,
      default: false
    },
    content: String,
    enterActiveClass: String,
    leaveActiveClass: String,
    boundariesSelector: String,
    reference: {},
    forceShow: {
      type: Boolean,
      default: false
    },
    dataValue: {
      default: null
    },
    appendToBody: {
      type: Boolean,
      default: false
    },
    visibleArrow: {
      type: Boolean,
      default: true
    },
    transition: {
      type: String,
      default: ''
    },
    stopPropagation: {
      type: Boolean,
      default: false
    },
    preventDefault: {
      type: Boolean,
      default: false
    },
    options: {
      type: Object,
      default() {
        return {}
      }
    }
  },
  data() {
    return {

      referenceElm: null,
      popperJS: null,
      showPopper: false,
      currentPlacement: '',
      popperOptions: {
        placement: 'bottom',
        computeStyle: {
          gpuAcceleration: false
        }
      }
    }
  },
  watch: {
    showPopover(value) {
      this.showPopper = value
    },
    showPopper(value) {
      if (value) {
        this.$emit('show', this)
        if (this.popperJS) {
          this.popperJS.enableEventListeners()
        }
        this.updatePopper()
      } else {
        if (this.popperJS) {
          this.popperJS.disableEventListeners()
        }
        this.$emit('hide', this)
      }
    },
    forceShow: {
      handler(value) {
        this[value ? 'doShow' : 'doClose']()
      },
      immediate: true
    },
    disabled(value) {
      if (value) {
        this.showPopper = false
      }
    }
  },
  created() {
    this.appendedArrow = false
    this.appendedToBody = false
    this.popperOptions = Object.assign(this.popperOptions, this.options)
  },
  mounted() {
    this.referenceElm = this.reference || this.$slots.reference[0].elm
    this.popper = this.$slots.default[0].elm
    switch (this.trigger) {
      case 'clickToOpen':
        on(this.referenceElm, 'click', this.doShow)
        on(document, 'click', this.handleDocumentClick)
        break
      case 'click':
      case 'clickToToggle':
        on(this.referenceElm, 'click', this.doToggle)
        on(document, 'click', this.handleDocumentClick)
        break
      case 'hover':
        on(this.referenceElm, 'mouseover', this.onMouseOver)
        on(this.popper, 'mouseover', this.onMouseOver)
        on(this.referenceElm, 'mouseout', this.onMouseOut)
        on(this.popper, 'mouseout', this.onMouseOut)
        break
      case 'focus':
        on(this.referenceElm, 'focus', this.onMouseOver)
        on(this.popper, 'focus', this.onMouseOver)
        on(this.referenceElm, 'blur', this.onMouseOut)
        on(this.popper, 'blur', this.onMouseOut)
        break
    }
  },
  methods: {
    // 是否显示
    doToggle(event) {
      if (this.stopPropagation) {
        event.stopPropagation()
      }
      if (this.preventDefault) {
        event.preventDefault()
      }
      if (!this.forceShow) {
        this.showPopper = !this.showPopper
      }
    },
    // 显示
    doShow() {
      this.showPopper = true
    },
    // 关闭
    doClose() {
      this.showPopper = false
    },
    // 开始销毁
    doDestroy() {
      if (this.showPopper) {
        return
      }
      if (this.popperJS) {
        this.popperJS.destroy()
        this.popperJS = null
      }
      if (this.appendedToBody) {
        this.appendedToBody = false
        document.body.removeChild(this.popper.parentElement)
      }
    },
    // 新建
    createPopper() {
      this.$nextTick(() => {
        if (this.visibleArrow) {
          this.appendArrow(this.popper)
        }
        if (this.appendToBody && !this.appendedToBody) {
          this.appendedToBody = true
          document.body.appendChild(this.popper.parentElement)
        }
        if (this.popperJS && this.popperJS.destroy) {
          this.popperJS.destroy()
        }
        if (this.boundariesSelector) {
          const boundariesElement = document.querySelector(this.boundariesSelector)
          if (boundariesElement) {
            this.popperOptions.modifiers = Object.assign({}, this.popperOptions.modifiers)
            this.popperOptions.modifiers.preventOverflow = Object.assign(
              {},
              this.popperOptions.modifiers.preventOverflow
            )
            this.popperOptions.modifiers.preventOverflow.boundariesElement = boundariesElement
          }
        }
        this.popperOptions.onCreate = () => {
          this.$emit('created', this)
          this.$nextTick(this.updatePopper)
        }
        this.popperJS = new Popper(this.referenceElm, this.popper, this.popperOptions)
      })
    },
    // 销毁
    destroyPopper() {
      off(this.referenceElm, 'click', this.doToggle)
      off(this.referenceElm, 'mouseup', this.doClose)
      off(this.referenceElm, 'mousedown', this.doShow)
      off(this.referenceElm, 'focus', this.doShow)
      off(this.referenceElm, 'blur', this.doClose)
      off(this.referenceElm, 'mouseout', this.onMouseOut)
      off(this.referenceElm, 'mouseover', this.onMouseOver)
      off(document, 'click', this.handleDocumentClick)
      this.showPopper = false
      this.doDestroy()
    },
    // 放入指向箭头
    appendArrow(element) {
      if (this.appendedArrow) {
        return
      }
      this.appendedArrow = true
      const arrow = document.createElement('div')
      arrow.setAttribute('x-arrow', '')
      arrow.className = 'popper__arrow'
      element.appendChild(arrow)
    },
    // 刷新
    updatePopper() {
      this.popperJS ? this.popperJS.scheduleUpdate() : this.createPopper()
    },
    // 鼠标移入
    onMouseOver() {
      clearTimeout(this._timer)
      this._timer = setTimeout(() => {
        this.showPopper = true
      }, this.delayOnMouseOver)
    },
    // 鼠标移出
    onMouseOut() {
      clearTimeout(this._timer)
      this._timer = setTimeout(() => {
        this.showPopper = false
      }, this.delayOnMouseOut)
    },
    // 点击外部元素
    handleDocumentClick(e) {
      if (
        !this.$el ||
        !this.referenceElm ||
        this.elementContains(this.$el, e.target) ||
        this.elementContains(this.referenceElm, e.target) ||
        !this.popper ||
        this.elementContains(this.popper, e.target)
      ) {
        return
      }
      this.$emit('documentClick', this)
      if (this.forceShow) {
        return
      }
      this.showPopper = false
    },
    elementContains(elm, otherElm) {
      if (typeof elm.contains === 'function') {
        return elm.contains(otherElm) // 判断是否点击弹层内元素
      }
      return false
    }
  },
  destroyed() {
    this.destroyPopper()
  }
}
</script>

<style lang="scss">
.m-popper-content {
  width: auto;
  background-color: #ffffff;
  color: #212121;
  text-align: center;
  padding: 2px;
  display: inline-block;
  border-radius: 3px;
  position: absolute;
  font-size: 14px;
  font-weight: normal;
  z-index: 200000;
  -moz-box-shadow: #e6e6e6 0 0 6px 0;
  -webkit-box-shadow: #e6e6e6 0 0 6px 0;
  box-shadow: #e6e6e6 0 0 6px 0;
  .popper__arrow {
    width: 0;
    height: 0;
    border-style: solid;
    position: absolute;
    margin: 5px;
  }
}
.m-popper-content[x-placement^='top'] {
  margin-bottom: 5px;
  .popper__arrow {
    border-width: 5px 5px 0 5px;
    border-color: #ffffff transparent transparent transparent;
    bottom: -5px;
    left: calc(50% - 5px);
    margin-top: 0;
    margin-bottom: 0;
  }
}
.m-popper-content[x-placement^='bottom'] {
  margin-top: 5px;
  .popper__arrow {
    border-width: 0 5px 5px 5px;
    border-color: transparent transparent #ffffff transparent;
    top: -5px;
    left: calc(50% - 5px);
    margin-top: 0;
    margin-bottom: 0;
  }
}
.m-popper-content[x-placement^='right'] {
  margin-left: 5px;
  .popper__arrow {
    border-width: 5px 5px 5px 0;
    border-color: transparent #ffffff transparent transparent;
    left: -5px;
    top: calc(50% - 5px);
    margin-left: 0;
    margin-right: 0;
  }
}
.m-popper-content[x-placement^='left'] {
  margin-right: 5px;
  .popper__arrow {
    border-width: 5px 0 5px 5px;
    border-color: transparent transparent transparent #ffffff;
    right: -5px;
    top: calc(50% - 5px);
    margin-left: 0;
    margin-right: 0;
  }
}
</style>
