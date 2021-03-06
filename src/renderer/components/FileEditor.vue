<template>
  <div class="fe-wrapper" ref="ref_wrapper">
    <div class="fe-bar-container" v-show="file.searchMode">
      <div class="fe-bar-search">
        <el-input
            placeholder="Search" size="mini"
            ref="ref_input_search"
            v-model="searchString"
            clearable>
        </el-input>
        <el-checkbox v-model="isMatchCase">Match Case</el-checkbox>
        <el-checkbox v-model="isReplace">Replace</el-checkbox>
        <el-tag size="medium" v-show="searchData.finder">{{ searchData.matches }}</el-tag>
      </div>
      <div class="fe-bar-replace" v-show="isReplace">
        <el-input
            placeholder="Replace" size="mini"
            v-model="replaceString"
            clearable>
        </el-input>
        <el-button type="primary" size="mini" plain @click="handleButtonReplaceAllClick">Replace all</el-button>
      </div>
    </div>

    <div class="fe-editor-container" :style="editorStyle"></div>
  </div>
</template>

<script>
  import { mapState, mapGetters } from 'vuex'
  import debounce from 'lodash/debounce'
  import isEqual from 'lodash/isEqual'
  import Quill from 'quill'
  import Delta from 'quill-delta'
  import MagicUrl from 'quill-magic-url'
  import findAndReplaceDOMText from 'findandreplacedomtext'
  Quill.register('modules/magicUrl', MagicUrl)

  export default {
    name: 'file-editor',

    props: {
      file: Object,
      active: Boolean
    },

    data () {
      return {
        editor: null,
        savedUndo: null, // latest undo operation after last file saving (to detect if file is changed)

        // DOM elements
        editorContainerElement: null,
        barContainerElement: null,
        toolbarElement: null,
        editorElement: null,

        // Search properties
        searchString: '',
        replaceString: '',
        searchData: {
          initSelection: 0,
          initContents: null,
          finder: null,
          matches: 0
        },
        isMatchCase: false,
        isReplace: false
      }
    },

    computed: {
      ...mapState({
        editorSettings: state => state.settings.editor,
        displayEditorToolbar: state => state.view.editorToolbar.visible
      }),
      ...mapGetters([
        'editorStyle'
      ])
    },

    watch: {
      active: {
        handler: function (active) {
          if (!active) return
          this.$nextTick(() => {
            if (this.file.searchMode) this.$refs.ref_input_search.focus()
            else this.editor.focus()
          })
        },
        immediate: true
      },

      displayEditorToolbar (display) {
        this.toggleToolbar(display)
      },

      'file.flags.savedCounter' () {
        const undoLength = this.editor.history.stack.undo.length
        this.savedUndo = undoLength > 0 ? this.editor.history.stack.undo[undoLength - 1] : null
      },

      // ********** Search/replace stuff **********

      'file.searchMode' (searchMode) {
        if (searchMode) { // search/replace mode is on
          this.searchData.initSelection = this.editor.getSelection(true).index
          this.searchData.initContents = this.editor.getContents()
          this.editor.disable()
          this.toolbarElement.style.opacity = '0.3'
          this.$nextTick(() => {
            this.$refs.ref_input_search.focus()
          })
        } else {
          this.resetSearch()
          this.searchData.initSelection = 0
          this.searchData.initContents = null
          this.editor.enable()
          this.toolbarElement.style.opacity = ''
          this.editor.focus()
        }
      },
      searchString () {
        this.doSearch()
      },
      isMatchCase () {
        this.doSearch()
      }
    },

    mounted () {
      // Create Quill instance
      const toolbarOptions = {
        container: [
          [{ 'header': [1, 2, 3, 4, 5] }],
          [{ 'font': [] }],
          [{ 'size': ['small', false, 'large'] }],
          ['bold', 'italic', 'underline', 'strike'],
          ['blockquote', 'code-block'],
          [{ 'list': 'ordered' }, { 'list': 'bullet' }, { 'align': [] }],
          [{ 'script': 'super' }, { 'script': 'sub' }],
          [{ 'indent': '-1' }, { 'indent': '+1' }],
          // [{ 'direction': 'rtl' }],
          [{ 'color': [] }, { 'background': [] }],
          ['link', 'image', 'video'],
          ['clean']
        ]
      }
      const options = {
        placeholder: `Let's type something interesting here...`,
        theme: 'snow',
        modules: {
          toolbar: toolbarOptions,
          clipboard: {
            matchVisual: false
          },
          history: {
            delay: 400,
            maxStack: 400
          },
          magicUrl: true
        }
      }
      this.editorContainerElement = this.$refs.ref_wrapper.querySelector('.fe-editor-container')
      this.editor = new Quill(this.editorContainerElement, options)

      // Initial file data were provided
      if (this.file.data) {
        this.loadContents()
      }

      // Cache additional DOM elements
      this.barContainerElement = this.$refs.ref_wrapper.querySelector('.fe-bar-container')
      this.toolbarElement = this.$refs.ref_wrapper.querySelector('.ql-toolbar')
      this.editorElement = this.$refs.ref_wrapper.querySelector('.ql-editor')

      this.toggleToolbar(this.displayEditorToolbar)

      this.editor.on('text-change', (delta, oldDelta, source) => {
        const undoLength = this.editor.history.stack.undo.length
        const lastUndo = undoLength > 0 ? this.editor.history.stack.undo[undoLength - 1] : null

        if ((!this.file.path && this.editor.getLength() === 1) || this.savedUndo === lastUndo) {
          if (this.file.flags.wasChanged) this.$store.commit('FILE_SET_FLAGS', { fileId: this.file.id, flags: { wasChanged: false } })
          return
        }
        if (!this.file.flags.wasChanged) {
          this.$store.commit('FILE_SET_FLAGS', { fileId: this.file.id, flags: { wasChanged: true } })
        }
      })
    },

    methods: {
      loadContents () {
        this.editor.clipboard.dangerouslyPasteHTML(this.file.data)
        this.editor.history.clear()
      },

      toggleToolbar (display) {
        if (display) this.toolbarElement.classList.remove('d-none')
        else this.toolbarElement.classList.add('d-none')
      },

      formatText (format, value) {
        const range = this.editor.getSelection()
        const currentFormat = this.editor.getFormat(range)
        if (currentFormat[format] === value) return
        this.editor.format(format, value, 'user')
      },

      removeFormat () {
        const range = this.editor.getSelection()
        this.editor.removeFormat(range)
      },

      transformText (t) {
        const range = this.editor.getSelection()
        let text = this.editor.getText(range.index, range.length)

        if (t === 'u') {
          text = text.toUpperCase()
        } else if (t === 'l') {
          text = text.toLowerCase()
        } else if (t === 's') {
          text = text.toLowerCase()
          const separatorList = ['. ', '? ', '! ', '.\n', '?\n', '!\n', '\n']
          separatorList.forEach(separator => {
            text = text.split(separator).map(sentence => capitalize(sentence)).join(separator)
          })
        } else if (t === 't') {
          text = text.toLowerCase()
          text = text.split(' ').map(word => capitalize(word)).join(' ')
          text = text.split('\n').map(word => capitalize(word)).join('\n')
        }

        // Create format groups to keep all existing formats
        let insertions = {
          text: [],
          attr: []
        }
        let len, format, prevFormat
        for (let i = range.index, j = 0; i < range.index + range.length; i += 1, j += 1) {
          len = insertions.attr.length
          format = this.editor.getFormat(i, 1) // get current character format

          if (len > 0) {
            prevFormat = insertions.attr[len - 1]
            if (isEqual(prevFormat, format)) { // format is still the same
              insertions.text[len - 1] = insertions.text[len - 1] + text[j]
            } else {
              insertions.text.push(text[j])
              insertions.attr.push(format)
            }
          } else {
            insertions.text.push(text[j])
            insertions.attr.push(format)
          }
        }

        const delta = new Delta()
        delta.retain(range.index).delete(range.length)
        for (let i = 0; i < insertions.attr.length; i += 1) {
          delta.insert(insertions.text[i], insertions.attr[i])
        }
        this.editor.updateContents(delta)

        function capitalize (s) {
          return s.charAt(0).toUpperCase() + s.slice(1)
        }
      },

      // ********** Search/replace methods **********

      resetSearch () {
        if (this.searchData.finder &&
          this.searchData.finder.reverts.length > 0 &&
          !this.searchData.finder.options.replace) { // there are contents modifications (search highlight); do not revert replace
          try {
            this.searchData.finder.revert()
          } catch (err) { // fallback if an error occurred (restore contents)
            this.editor.setContents(this.searchData.initContents)
          }
        }
        this.searchData.finder = null
        if (!this.file.searchMode) this.editor.setSelection(this.searchData.initSelection)
      },
      doSearch: debounce(function () {
        this.resetSearch() // reset previous search
        if (!this.searchString) return

        this.regexpString = RegExp(this.escapeRegExp(this.searchString), this.isMatchCase ? 'g' : 'gi')
        this.searchData.finder = findAndReplaceDOMText(this.editorElement, {
          find: this.regexpString,
          wrap: 'em',
          wrapClass: 'markedText',
          forceContext: function (el) {
            // Using https://developer.mozilla.org/en-US/docs/Web/API/Element/matches
            return el.matches('p') || el.matches('li')
          }
        })
        this.searchData.matches = this.searchData.finder.reverts.length
      }, 500),
      doReplace () {
        this.resetSearch() // reset previous search
        if (!this.searchString || !this.replaceString) return

        this.editor.enable() // keep changes in history
        this.regexpString = RegExp(this.escapeRegExp(this.searchString), this.isMatchCase ? 'g' : 'gi')
        this.searchData.finder = findAndReplaceDOMText(this.editorElement, {
          find: this.regexpString,
          replace: this.replaceString,
          forceContext: function (el) {
            // Using https://developer.mozilla.org/en-US/docs/Web/API/Element/matches
            return el.matches('p') || el.matches('li')
          }
        })
        this.searchData.matches = this.searchData.finder.reverts.length
        setTimeout(() => {
          this.editor.disable()
        }, 10)
      },
      escapeRegExp (s) {
        return String(s).replace(/([.*+?^=!:${}()|[\]/\\])/g, '\\$1')
      },

      // ********** Handlers **********

      handleButtonReplaceAllClick () {
        this.doReplace()
      }
    }
  }
</script>

<style>
  .fe-wrapper {
    height: 100%;
    display: flex;
    flex-direction: column;
  }

  .fe-bar-container {
    margin-top: 2px;

    & .el-input {
      width: 40%;
      margin-right: 4px;
    }

    & .el-checkbox, & .el-tag {
      margin-left: 12px;
    }

    & .el-checkbox {
      font-weight: normal;
    }

    & >>> .el-checkbox__label {
      font-size: 12px;
      padding-left: 8px;
    }

    & .el-tag {
      font-size: 14px;
    }
  }

  .fe-bar-search, .fe-bar-replace {
    display: flex;
    align-items: center;
    padding: 2px 4px;
  }

  .fe-editor-container {
    overflow: auto;
    flex: 1;
  }
</style>
