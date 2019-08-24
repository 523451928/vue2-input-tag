<template>
  <div class="input-tag-container" :class="{'input-tag-container--active': isInputActive}">
    <!-- <i class="iconfont iconwuliao-shanchu color-alert" v-if="clearBtnVisible" @click="clearFn"></i> -->
    <div @click.stop="focusNewTag" :class="{
        'read-only': readOnly,
        'vue-input-tag-wrapper--active': isInputActive,
      }" :style="{height: inputHeight, maxHeight}" class="vue-input-tag-wrapper" @scroll="scrollFn">
      <span @click="changeActive(index)" v-for="(tag, index) in innerTags" :key="index" :data-index="index"
        :class="{'input-tag-active': selectIndexArr.includes(index) || activeIndex === index, 'input-tag-error': validateTag(tag)}"
        class="input-tag">
        <span @dblclick.stop="changeEdit(index)" :data-index="index" v-if="index !== editIndex">{{ tag }}</span>
        <input type="text" class="edit-tag" ref="input" v-if="index === editIndex" :value="tag" @blur="modifyTag" @keydown.stop
          @keyup.esc.stop="notModify(tag)" @keyup.enter.stop="modifyTag">
        <a v-if="!readOnly" @click.prevent.stop="remove(index)" class="remove">
          <slot name="remove-icon" />
        </a>
      </span>
      <span v-show="innerTags.length && isInputActive" class="del-btn" @click="clearFn"
        :style="{top: scrollTop + 'px', bottom: -scrollTop + 'px'}">
        <span class="icon-container">
          <i v-show="clearBtnVisible" class="iconfont iconwuliao-shanchu"></i>
          <!-- <br />
          <i class="iconfont iconjiesuanpingtai-fuzhi" @click="copyText"></i> -->
        </span>
      </span>
      <input v-if="!readOnly && !isLimit" ref="inputtag" :placeholder="computedPlaceholder" type="text" v-model.trim="newTag"
        @paste="pasteFn" @keydown.delete.stop="removeLastTag" @keydown.stop="addNew" @blur="handleInputBlur" @focus="handleInputFocus"
        class="new-tag" />
    </div>
  </div>
</template>

<script>
const removeRepeat = arr => {
  if (typeof arr[0] === 'object') {
    for (let i = 0; i < arr.length; i++) {
      arr[i] = JSON.stringify(arr[i])
    }
    arr = [...new Set(arr)]
    for (let i = 0; i < arr.length; i++) {
      arr[i] = JSON.parse(arr[i])
    }
  } else {
    arr = [...new Set(arr)]
  }
  return arr
}
const copyText = txt => {
  if (typeof txt !== 'string' || !txt.length) {
    return
  }
  let txtEl = document.createElement('input')
  txtEl.setAttribute('type', 'text')
  txtEl.setAttribute('value', txt)
  txtEl.style.position = 'fixed'
  txtEl.style.left = '-999999px'
  txtEl.style.top = '-9999px'
  document.body.appendChild(txtEl)
  txtEl.select()
  document.execCommand('copy')
  document.body.removeChild(txtEl)
  txtEl = null
}

const validators = {
  email: new RegExp(
    /^(([^<>()[\]\\.,:\s@\"]+(\.[^<>()[\]\\.,:\s@\"]+)*)|(\".+\"))@((\[[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\])|(([a-zA-Z\-0-9]+\.)+[a-zA-Z]{2,}))$/
  ),
  url: new RegExp(
    /^(https?|ftp|rmtp|mms):\/\/(([A-Z0-9][A-Z0-9_-]*)(\.[A-Z0-9][A-Z0-9_-]*)+)(:(\d+))?\/?/i
  ),
  text: new RegExp(/^[a-zA-Z]+$/),
  digits: new RegExp(/^[\d() \.\:\-\+#]+$/),
  isodate: new RegExp(
    /^\d{4}[\/\-](0?[1-9]|1[012])[\/\-](0?[1-9]|[12][0-9]|3[01])$/
  ),
  codeName: /^([a-zA-Z0-9_]{1,20})$/,
  isNumber: /^([0-9_]{1,20})$/
}

export default {
  name: 'BaseInputTag',
  props: {
    value: {
      type: Array,
      default: () => []
    },
    placeholder: {
      type: String,
      default() {
        return this.$lang('最多可查询500条，以逗号、空格或回车键隔开')
      }
    },
    readOnly: {
      type: Boolean,
      default: false
    },
    validate: {
      type: [String, Function, Object],
      default: 'codeName'
    },
    addTagOnKeys: {
      type: Array,
      default: function() {
        return [
          13, // Return
          188, // Comma ','
          9, // Tab
          32 // black space
        ]
      }
    },
    addTagOnBlur: {
      type: Boolean,
      default: false
    },
    limit: {
      type: Number,
      default: -1
    },
    allowDuplicates: {
      type: Boolean,
      default: false
    },
    beforeAdding: {
      type: Function
    },
    maxHeight: {
      type: String,
      default: '180px'
    },
    clearBtnVisible: {
      type: Boolean,
      default: true
    }
  },
  data() {
    return {
      newTag: '',
      innerTags: [...this.value],
      selectIndexArr: [],
      isInputActive: false,
      activeIndex: -1,
      editIndex: -1,
      isFocus: false,
      isCtrl: false,
      scrollTop: 0
    }
  },
  computed: {
    isLimit() {
      return this.limit > 0 && Number(this.limit) === this.innerTags.length
    },
    computedPlaceholder() {
      if (!this.innerTags.length) {
        return this.placeholder
      }
      return ''
    },
    inputHeight() {
      const wrapperDom = document.querySelector('.vue-input-tag-wrapper')
      if (
        this.isInputActive &&
        wrapperDom.clientHeight < 100 &&
        !this.innerTags.length
      ) {
        return '100px'
      }
      if (!this.isInputActive) {
        return '27px'
      }
      return 'auto'
    }
  },
  watch: {
    value() {
      this.innerTags = [...this.value]
    },
    selectIndexArr(newVal) {
      if (newVal.length > 1) {
        this.activeIndex = -1
      }
    },
    activeIndex(newVal) {
      if (newVal !== -1) {
        this.selectIndexArr = []
      }
    }
  },
  methods: {
    scrollFn(e) {
      this.scrollTop = e.target.scrollTop
    },
    clearFn() {
      this.innerTags = []
      this.tagChange()
    },
    changeEdit(index) {
      this.editIndex = index
      setTimeout(() => {
        const editTagDom = document.querySelector('.edit-tag')
        if (editTagDom) {
          editTagDom.focus()
          editTagDom.select()
        }
      }, 300)
    },
    async modifyTag() {
      const value = document.querySelector('.edit-tag').value
      const isValid = await this.validateIfNeeded(value)
      if (isValid) {
        this.innerTags[this.editIndex] = value
        this.tagChange()
        this.editIndex = -1
        this.focusNewTag()
      }
    },
    notModify() {
      this.editIndex = -1
    },
    selectTag(index) {
      if (this.selectIndexArr.includes(index)) {
        const arr = this.selectIndexArr.filter(item => index !== item)
        this.selectIndexArr = arr
      } else {
        this.selectIndexArr.push(index)
      }
      if (this.activeIndex !== -1) {
        this.selectIndexArr.push(this.activeIndex)
      }
      this.selectIndexArr = this.selectIndexArr.sort((a, b) => a - b)
      this.activeIndex = -1
      this.isFocus = true
      if (!this.isInputActive) {
        this.isInputActive = true
      }
    },
    changeActive(index) {
      if (this.isCtrl) return
      this.selectIndexArr = []
      this.activeIndex = index
      this.isFocus = true
      if (!this.isInputActive) {
        this.isInputActive = true
      }
    },
    pasteFn(e) {
      setTimeout(() => {
        const value = e.target.value
        if (!value) {
          return
        }
        let tags = value.split(' ')
        tags = tags.join(',').split(',')
        tags = tags.join('，').split('，')
        this.innerTags = removeRepeat([
          ...this.innerTags,
          ...tags.filter(item => item)
        ])
        this.tagChange()
        e.target.value = ''
        this.newTag = ''
      }, 100)
    },
    focusNewTag(e) {
      if (e && e.target) {
        const dataIndex = e.target.getAttribute('data-index')
        if (this.isCtrl && dataIndex) {
          this.selectTag(Number(dataIndex))
        }
        if (dataIndex) {
          return
        }
      }
      this.isFocus = true
      if (this.readOnly || !this.$el.querySelector('.new-tag')) {
        return
      }
      this.$el.querySelector('.new-tag').focus()
    },
    blurTag() {
      this.isFocus = false
      this.isInputActive = false
    },
    handleInputFocus() {
      this.activeIndex = -1
      this.selectIndexArr = []
      this.isInputActive = true
    },
    handleInputBlur(e) {
      this.addNew(e)
    },
    copyText() {
      if (this.activeIndex !== -1) {
        return copyText(this.innerTags[this.activeIndex])
      }
      const arr = this.innerTags.filter((item, index) =>
        this.selectIndexArr.includes(index)
      )
      copyText(arr.join(' '))
    },
    async addNew(e) {
      // ctrl + c 复制
      if (e.keyCode === 67 && e.ctrlKey) {
        this.copyText()
      }
      // ctrl + a 权限
      if (e.keyCode === 65 && e.ctrlKey) {
        this.selectIndexArr = this.innerTags.map((item, index) => index)
        this.$nextTick(() => {
          e && e.preventDefault()
          this.$el.querySelector('.new-tag').blur()
        })
      }
      // 左键改变选择项
      if (e.keyCode === 37 && !this.newTag) {
        this.activeIndex = this.innerTags.length - 1
        this.$refs.inputtag.blur()
      }

      const keyShouldAddTag = e
        ? this.addTagOnKeys.indexOf(e.keyCode) !== -1
        : true

      const typeIsNotBlur = e && e.type !== 'blur'

      if (
        (!keyShouldAddTag && (typeIsNotBlur || !this.addTagOnBlur)) ||
        this.isLimit
      ) {
        return
      }
      // 添加标签
      const tag = this.beforeAdding
        ? await this.beforeAdding(this.newTag)
        : this.newTag.trim()

      const isValid = await this.validateIfNeeded(tag)

      if (
        tag &&
        isValid &&
        (this.allowDuplicates || this.innerTags.indexOf(tag) === -1)
      ) {
        this.innerTags.push(tag)
        this.newTag = ''
        this.tagChange()
        this.activeIndex = -1
        document.querySelector('.vue-input-tag-wrapper').scrollTop = 1000
        e && e.preventDefault()
      }
    },
    validateTag(tag) {
      return !validators[this.validate].test(tag)
    },
    validateIfNeeded(tagValue) {
      if (this.validate === '' || this.validate === undefined) {
        return true
      }

      if (typeof this.validate === 'function') {
        return this.validate(tagValue)
      }

      if (
        typeof this.validate === 'string' &&
        Object.keys(validators).indexOf(this.validate) > -1
      ) {
        return validators[this.validate].test(tagValue)
      }

      if (
        typeof this.validate === 'object' &&
        this.validate.test !== undefined
      ) {
        return this.validate.test(tagValue)
      }
      return true
    },
    changeTag(e) {
      if (e.keyCode === 67 && e.ctrlKey) {
        this.copyText()
      }
      if (this.isFocus) {
        switch (e.keyCode) {
          case 13: // enter
            this.changeEdit(this.activeIndex)
            break
          case 8: // delete
          case 46: // delete
            if (this.activeIndex !== -1) {
              this.innerTags.splice(this.activeIndex, 1)
            } else {
              const tags = this.innerTags.filter(
                (item, index) => !this.selectIndexArr.includes(index)
              )
              this.innerTags = tags
            }
            this.selectIndexArr = []
            if (
              !this.innerTags.length ||
              this.activeIndex === this.innerTags.length
            ) {
              this.focusNewTag()
            }
            if (this.activeIndex === this.innerTags.length) {
              this.activeIndex = -1
            }
            break
          case 37: // left
            if (this.selectIndexArr.length) {
              const firstIndex = this.selectIndexArr[0]
              if (firstIndex > 0) {
                this.activeIndex = firstIndex - 1
              }
              return
            }
            this.activeIndex > 0 && this.activeIndex--
            break
          case 39: // right
            if (this.selectIndexArr.length) {
              const lastIndex = this.selectIndexArr[
                this.selectIndexArr.length - 1
              ]
              if (lastIndex === this.innerTags.length) {
                this.activeIndex = -1
                return this.focusNewTag()
              }
              this.activeIndex = lastIndex
            }
            if (this.activeIndex < this.innerTags.length - 1) {
              return this.activeIndex++
            }
            this.activeIndex = -1
            this.focusNewTag()
            break
        }
        this.tagChange()
      }
    },
    remove(index) {
      this.innerTags.splice(index, 1)
      this.activeIndex = -1
      this.tagChange()
    },
    removeLastTag() {
      if (this.newTag) {
        return
      }
      this.$refs.inputtag.blur()
      if (this.activeIndex === -1) {
        this.activeIndex = this.innerTags.length - 1
        return
      }
      // this.innerTags.pop()
      this.remove(this.activeIndex)
      this.activeIndex--
      this.tagChange()
    },
    tagChange() {
      this.$emit('update:tags', this.innerTags)
      this.$emit('input', this.innerTags)
    }
  },
  mounted() {
    document.addEventListener('keydown', this.changeTag)
    document.addEventListener('click', this.blurTag)
    const isWin =
      navigator.platform === 'Win32' || navigator.platform === 'Windows'
    const isMac =
      navigator.platform === 'Mac68K' ||
      navigator.platform === 'MacPPC' ||
      navigator.platform === 'Macintosh' ||
      navigator.platform === 'MacIntel'
    document.addEventListener('keydown', e => {
      if ((e.keyCode === 17 && isWin) || (e.keyCode === 91 && isMac)) {
        this.isCtrl = true
      }
    })
    document.addEventListener('keyup', e => {
      if ((e.keyCode === 17 && isWin) || (e.keyCode === 91 && isMac)) {
        this.isCtrl = false
      }
    })
  },
  destroyed() {
    document.removeEventListener('keydown', this.changeTag)
    document.removeEventListener('click', this.blurTag)
  }
}
</script>

<style lang="scss">
::-webkit-input-placeholder {
  /* WebKit browsers */
  color: #999999;
}
.vue-input-tag-wrapper {
  background-color: #fff;
  border: 1px solid #ddd;
  border-radius: 5px;
  overflow-y: auto;
  padding-left: 4px;
  padding-top: 4px;
  cursor: text;
  text-align: left;
  -webkit-appearance: textfield;
  display: flex;
  flex-wrap: wrap;
  transition: all 0.5s;
  max-height: 102px;
  .input-tag {
    border-radius: 2px;
    display: inline-block;
    font-size: 13px;
    color: #999999;
    margin-bottom: 3px;
    margin-right: 6px;
    padding: 4px 8px;
    background: rgba(221, 221, 221, 0.1);
    border-radius: 2px;
    border: 1px solid rgba(221, 221, 221, 0.5);
    box-sizing: border-box;
    height: 24px;
    line-height: 1;
    cursor: default;
    .remove {
      cursor: pointer;
      // color: $c-fc-3;
      padding-left: 5px;
      position: relative;
      top: -1px;
      &:hover {
        text-decoration: none;
      }
      &:empty::before {
        content: ' x';
      }
    }
  }
  .input-tag-error {
    border-color: #ff6a69;
    background: rgba(255, 106, 105, 0.1);
    color: #ff6a69;
    .remove {
      color: #ff6a69;
    }
  }
  .new-tag {
    background: transparent;
    border: 0;
    color: #777;
    font-size: 13px;
    font-weight: 400;
    margin-bottom: 3px;
    outline: none;
    padding: 2px;
    padding-left: 0;
    flex-grow: 1;
    width: 100px;
    height: 16px;
    vertical-align: top;
  }
  .input-tag-active {
    background: rgba(221, 221, 221, 0.5);
  }
}
.vue-input-tag-wrapper.read-only {
  cursor: default;
}
.input-tag-container {
  height: 32px;
}
.input-tag-container--active {
  height: 32px;
  position: relative;
  z-index: 100;
}
.vue-input-tag-wrapper--active {
  position: absolute;
  border-color: #69a1ff;
  box-sizing: border-box;
  width: 100%;
  left: 0;
  top: 0;
}
</style>
