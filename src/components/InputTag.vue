
<template>
  <div class="input-tag-container" :class="{
      'input-tag-container--active': isInputActive,
    }">
    <i class="iconfont iconwuliao-shanchu color-alert" v-if="clearBtnVisible" @click="innerTags = []"></i>
    <div
      @click.stop="focusNewTag"
      :class="{
        'read-only': readOnly,
        'vue-input-tag-wrapper--active': isInputActive,
      }"
      :style="{height: inputHeight, maxHeight}"
      class="vue-input-tag-wrapper"
    >
      <span @click.stop="changeActive(index)"
        v-for="(tag, index) in innerTags"
        :key="index"
        :class="{'input-tag-active': activeIndex === index || isSelectAll, 'input-tag-error': validateTag(tag)}"
        class="input-tag">
        <span @dblclick.stop="changeEdit(index)" v-if="index !== editIndex">{{ tag }}</span>
        <input
          type="text"
          class="edit-tag"
          ref="input"
          v-if="index === editIndex"
          :value="tag"
          @blur="modifyTag"
          @keydown.stop
          @keyup.enter.stop="modifyTag">
        <a v-if="!readOnly" @click.prevent.stop="remove(index)" class="remove">
          <slot name="remove-icon" />
        </a>
      </span>
      <input
        v-if="!readOnly && !isLimit"
        ref="inputtag"
        :placeholder="computedPlaceholder"
        type="text"
        v-model.trim="newTag"
        @paste="pasteFn"
        @keydown.delete.stop="removeLastTag"
        @keydown.stop="addNew"
        @blur="handleInputBlur"
        @focus="handleInputFocus"
        class="new-tag"
      />
    </div>
  </div>
</template>

<script>
const removeRepeat = (arr) => {
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
  codeName15: /^([a-zA-Z0-9_]{1,15})$/
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
      default () {
        return this.$lang('最多可查询500条，以逗号、空格或回车键隔开')
      }
    },
    readOnly: {
      type: Boolean,
      default: false
    },
    validate: {
      type: [String, Function, Object],
      default: ''
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
      default: false
    }
  },
  data() {
    return {
      newTag: '',
      innerTags: [...this.value],
      isInputActive: false,
      activeIndex: -1,
      editIndex: -1,
      isFocus: false,
      isSelectAll: false
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
      if (this.isInputActive && wrapperDom.clientHeight < 96 && !this.innerTags.length) {
        return '96px'
      }
      if (!this.isInputActive) {
        return '26px'
      }
      return 'auto'
    }
  },
  watch: {
    value() {
      this.innerTags = [...this.value]
    }
  },
  methods: {
    changeEdit(index) {
      this.editIndex = index
      setTimeout(() => {
        const editTagDom = document.querySelector('.edit-tag')
        editTagDom.focus()
        editTagDom.select()
      }, 300)
    },
    modifyTag() {
      this.innerTags[this.editIndex] = document.querySelector('.edit-tag').value
      this.tagChange()
      this.editIndex = -1
      this.focusNewTag()
    },
    changeActive(index) {
      this.activeIndex = index
      this.isSelectAll = false
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
        this.innerTags = removeRepeat([...this.innerTags, ...tags.filter(item => item)])
        this.tagChange()
        e.target.value = ''
        this.newTag = ''
      }, 100)
    },
    focusNewTag() {
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
      this.isInputActive = true
    },
    handleInputBlur(e) {
      this.addNew(e)
    },
    async addNew(e) {
      if (e.keyCode === 65 && e.ctrlKey) {
        this.isSelectAll = true
        this.$nextTick(() => {
          e && e.preventDefault()
          this.$el.querySelector('.new-tag').blur()
        })
      }
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
      return !validators.codeName.test(tag)
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
      if (this.isFocus) {
        switch (e.keyCode) {
          case 13: // enter
            this.changeEdit(this.activeIndex)
            break
          case 8: // delete
          case 46: // delete
            if (this.isSelectAll) {
              this.innerTags = []
            }
            this.innerTags.splice(this.activeIndex, 1)
            if (!this.innerTags.length || this.activeIndex === this.innerTags.length) {
              this.focusNewTag()
            }
            if (this.activeIndex === this.innerTags.length) {
              this.activeIndex = -1
            }
            this.isSelectAll = false
            break
          case 37: // left
            this.activeIndex > 0 && this.activeIndex--
            break
          case 39: // right
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
  },
  destroyed() {
    document.removeEventListener('keydown', this.changeTag)
    document.removeEventListener('click', this.blurTag)
  }
}
</script>

<style lang="scss">
@import "~@scss/variable.scss";

::-webkit-input-placeholder { /* WebKit browsers */
  color: $c-fc-3;
}
.vue-input-tag-wrapper {
  background-color: #fff;
  border: 1px solid #DDD;
  border-radius: 5px;
  overflow-y: auto;
  padding-left: 4px;
  padding-top: 4px;
  cursor: text;
  text-align: left;
  -webkit-appearance: textfield;
  display: flex;
  flex-wrap: wrap;
  transition: all .5s;
  max-height: 102px;
  .input-tag {
    border-radius: 2px;
    display: inline-block;
    font-size: 13px;
    color: $c-fc-3;
    margin-bottom: 3px;
    margin-right: 6px;
    padding: 4px 8px;
    background:rgba(221,221,221,0.1);
    border-radius:2px;
    border:1px solid rgba(221,221,221,0.5);
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
    border-color: #FF6A69;
    background: rgba(255,106,105,0.1);
    color: #FF6A69;
    .remove {
      color: #FF6A69;
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
  .edit-tag {
    position: relative;
    top: -2px;
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
    height: 14px;
    vertical-align: top;
  }
  .input-tag-active {
    background: rgba(221,221,221,0.5);;
  }
}
.vue-input-tag-wrapper.read-only {
  cursor: default;
}
.input-tag-container {
  position: relative;
  height: 32px;
  .iconwuliao-shanchu {
    position: absolute;
    cursor: pointer;
    top: -28px;
    right: 7px;
    color: #F56C6C;
  }
}
.input-tag-container--active {
  height: 32px;
  position: relative;
  z-index: 100;
}
.vue-input-tag-wrapper--active {
  position: absolute;
  border-color: #69A1FF;
  box-sizing: border-box;
  width: 100%;
  left: 0;
  top: 0;
}
</style>
