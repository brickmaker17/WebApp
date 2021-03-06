<template>
  <no-ssr>
    <div class="hc-editor-wrapper" v-if="showEditor">
      <div :id="`toolbar-editor-${identifier}`" v-if="toolbar">
        <div class="ql-formats">
          <b-tooltip :label="$t('component.editor.italic')" type="is-black">
            <button class="ql-italic"></button>
          </b-tooltip>
          <b-tooltip :label="$t('component.editor.bold')" type="is-black">
            <button class="ql-bold"></button>
          </b-tooltip>
          <b-tooltip :label="$t('component.editor.strike')" type="is-black">
            <button class="ql-strike"></button>
          </b-tooltip>
        </div>
        <div class="ql-formats">
          <b-tooltip :label="$t('component.editor.blockquote')" type="is-black">
            <button class="ql-blockquote"></button>
          </b-tooltip>
        </div>
        <div class="ql-formats">
          <b-tooltip :label="$t('component.editor.listUnordered')" type="is-black">
            <button class="ql-list" value="bullet" ></button>
          </b-tooltip>
          <b-tooltip :label="$t('component.editor.listOrdered')" type="is-black">
            <button class="ql-list" value="ordered" ></button>
          </b-tooltip>
        </div>
        <div class="ql-formats">
          <b-tooltip :label="$t('component.editor.link')" type="is-black">
            <button @click.prevent="$refs.editorLinks.toggle()" class="ql-editor-button">
              <hc-icon icon="link" />
            </button>
          </b-tooltip>
          <b-tooltip :label="$t('component.editor.embed')" type="is-black">
            <button @click.prevent="$refs.editorEmbeds.toggle()" class="ql-editor-button">
              <hc-icon icon="code" />
            </button>
          </b-tooltip>
          <b-tooltip :label="$t('component.editor.emoji')" type="is-black">
            <button @click.prevent="$refs.editorEmojis.toggle()" class="ql-editor-button">
              <hc-icon icon="smile-o" />
            </button>
          </b-tooltip>
        </div>
      </div>
      <div class="hc-editor-container content hc-editor-content">
        <div class="quill-editor" :class="editorClass"
          v-model="editorText"
          :disabled="loading"
          @blur="editorBlur()"
          @focus="editorFocus()"
          @ready="editorReady($event)"
          @keydown.shift.enter.capture="handleShiftEnter"
          ref="content"
          v-quill:myQuillEditor="computedEditorOptions"></div>
        <div class="plugins" v-if="ready && myQuillBus">
          <editor-links :quill="myQuillBus" ref="editorLinks" />
          <editor-embeds :quill="myQuillBus" ref="editorEmbeds" />
          <editor-emojis :quill="myQuillBus" ref="editorEmojis" />
          <editor-mentions :quill="myQuillBus" ref="editorMentions" />
        </div>
      </div>
    </div>
  </no-ssr>
</template>

<script>
  import EditorLinks from '~/components/Editor/Links/EditorLinks'
  import EditorEmbeds from '~/components/Editor/Embeds/EditorEmbeds'
  import EditorEmojis from '~/components/Editor/Emojis/EditorEmojis'
  import EditorMentions from '~/components/Mentions/EditorMentions'
  import Emitter from 'emitter-js'
  import Delta from 'quill-delta'

  class QuillBus {
    constructor (quillEditor) {
      this.emitter = new Emitter()
      let bus = this
      this.getBounds = (index) => quillEditor.getBounds(index)
      this.updateContents = (ops) => quillEditor.updateContents(ops)
      this.setSelection = (index) => quillEditor.setSelection(index)
      this.getSelection = (focus) => quillEditor.getSelection(focus)
      this.getLine = (index) => quillEditor.getLine(index)
      this.getText = (index, length) => quillEditor.getText(index, length)
      this.format = (name, value, source) => quillEditor.format(name, value, source)
      this.insertText = (...args) => quillEditor.insertText(...args)
      quillEditor.on('text-change', (...args) => bus.emit('text-change', ...args))
      quillEditor.on('selection-change', (...args) => bus.emit('selection-change', ...args))
    }

    on (eventName, fn) {
      this.emitter.on(eventName, fn)
    }

    off (eventName, fn) {
      this.emitter.off(eventName, fn)
    }

    emit (eventName, ...args) {
      this.emitter.emit(eventName, ...args)
    }
  }

  export default {
    name: 'hc-editor',
    components: {
      EditorMentions,
      EditorLinks,
      EditorEmbeds,
      EditorEmojis
    },
    props: {
      value: {
        type: [String, Boolean],
        default: ''
      },
      loading: {
        type: Boolean,
        default: false
      },
      editorOptions: {
        type: Object,
        default: () => {}
      },
      editorClass: {
        type: String,
        default: 'story'
      },
      identifier: {
        type: [String],
        default: 'editor'
      }
    },
    data () {
      return {
        showEditor: true,
        ready: false,
        editorText: '',
        focus: true,
        myQuillBus: false,
        defaultEditorOptions: {
          theme: 'snow',
          modules: {
            toolbar: {
              container: `#toolbar-editor-${this.identifier}`,
              handlers: {
              }
            },
            urlEmbeds: {
              metaCallback: this.handleMeta
            }
          },
          placeholder: '...'
        }
      }
    },
    computed: {
      computedEditorOptions () {
        let newOptions = { ...this.defaultEditorOptions, ...this.editorOptions }
        return newOptions
      },
      changes () {
        return this.editorText !== this.value
      },
      toolbar () {
        if (!this.computedEditorOptions.modules) {
          return false
        }
        return this.computedEditorOptions.modules.toolbar
      }
    },
    methods: {
      reset () {
        this.editorText = this.value
        this.$emit('reset')
      },
      editorBlur () {
        this.focus = false
      },
      editorFocus () {
        this.focus = true
      },
      editorReady () {
        this.myQuillBus = new QuillBus(this.myQuillEditor)
        this.ready = true
      },
      handleMeta (data) {
        this.$emit('fetchedMeta', data)
      },
      // Handle shift + enter > line break outside of quill
      // Because it is not yet supported by it
      handleShiftEnter (event) {
        const quill = this.myQuillBus
        const sel = quill.getSelection()
        if (!sel) {
          return
        }
        const ops = new Delta()
          .retain(sel.index)
          .insert({linebreak: {}})
        quill.updateContents(ops)
        quill.setSelection(sel.index + 1)
        event.stopPropagation()
        event.preventDefault()
      }
    },
    watch: {
      editorText (newText, oldText) {
        if (newText !== oldText) {
          this.$emit('input', newText)
        }
      },
      value (newValue, oldValue) {
        if (newValue !== oldValue) {
          this.editorText = newValue
        }
      },
      computedEditorOptions: {
        handler: function () {
          this.showEditor = false
          this.$nextTick(() => {
            this.showEditor = true
          })
        },
        deep: true
      }
    },
    created () {
      this.editorText = this.value
    },
    mounted () {
      this.editorText = this.value
    }
  }
</script>

<style lang="scss">
  @import "assets/styles/utilities";

  .hc-editor-container {
    position: relative;
  }

  .has-error,
  .is-danger {
    .quill-editor {
      border-color: $red !important;
    }
  }
</style>
