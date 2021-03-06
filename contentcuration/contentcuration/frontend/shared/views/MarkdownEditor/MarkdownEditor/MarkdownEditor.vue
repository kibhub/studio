<template>

  <div
    style="position: relative;"
    class="wrapper"
    :class="{highlight, uploading: Boolean(uploadingChecksum)}"
    @dragenter="highlight = true"
    @dragover="highlight = true"
    @dragleave="highlight = false"
    @drop="highlight = false"
  >
    <div
      ref="editor"
      class="editor"
    ></div>

    <FormulasMenu
      v-if="formulasMenu.isOpen"
      v-model="formulasMenu.formula"
      v-click-outside="onClick"
      class="formulas-menu"
      :anchorArrowSide="formulasMenu.anchorArrowSide"
      :mathQuill="mathQuill"
      style="position:absolute"
      :style="formulasMenu.style"
      @insert="onFormulasMenuInsert"
      @cancel="onFormulasMenuCancel"
    />
    <ImagesMenu
      v-if="imagesMenu.isOpen"
      v-click-outside="onClick"
      class="images-menu"
      :anchorArrowSide="imagesMenu.anchorArrowSide"
      style="position:absolute"
      :style="imagesMenu.style"
      :src="imagesMenu.src"
      :alt="imagesMenu.alt"
      :handleFileUpload="handleFileUpload"
      :getFileUpload="getFileUpload"
      :imagePreset="imagePreset"
      @insert="insertImageToEditor"
      @cancel="onImagesMenuCancel"
    />
  </div>

</template>

<script>

  /**
   * * * * * * * * * * * * * * * * * * *
   * See docs/markdown_editor_viewer.md
   * * * * * * * * * * * * * * * * * * *
   */

  import '../mathquill/mathquill.js';
  import 'codemirror/lib/codemirror.css';
  import '@toast-ui/editor/dist/toastui-editor.css';

  import Vue from 'vue';
  import Editor from '@toast-ui/editor';
  import debounce from 'lodash/debounce';

  import imageUpload from '../plugins/image-upload';
  import formulas from '../plugins/formulas';
  import minimize from '../plugins/minimize';
  import formulaHtmlToMd from '../plugins/formulas/formula-html-to-md';
  import formulaMdToHtml from '../plugins/formulas/formula-md-to-html';
  import imagesHtmlToMd from '../plugins/image-upload/image-html-to-md';
  import imagesMdToHtml from '../plugins/image-upload/image-md-to-html';

  import { CLASS_MATH_FIELD_ACTIVE, CLASS_IMG_FIELD_NEW } from '../constants';
  import ImageField from './ImageField/ImageField';
  import { clearNodeFormat, getExtensionMenuPosition } from './utils';
  import keyHandlers from './keyHandlers';
  import FormulasMenu from './FormulasMenu/FormulasMenu';
  import ImagesMenu from './ImagesMenu/ImagesMenu';
  import ClickOutside from 'shared/directives/click-outside';
  import { registerMarkdownFormulaElement } from 'shared/views/MarkdownEditor/plugins/formulas/MarkdownFormula';

  registerMarkdownFormulaElement();

  const wrapWithSpaces = html => `&nbsp;${html}&nbsp;`;

  const ImageFieldClass = Vue.extend(ImageField);

  export default {
    name: 'MarkdownEditor',
    components: {
      FormulasMenu,
      ImagesMenu,
    },
    directives: {
      ClickOutside,
    },
    model: {
      prop: 'markdown',
      event: 'update',
    },
    props: {
      markdown: {
        type: String,
      },
      // Inject function to handle file uploads
      handleFileUpload: {
        type: Function,
      },
      // Inject function to get file upload object
      getFileUpload: {
        type: Function,
      },
      imagePreset: {
        type: String,
      },
    },
    data() {
      return {
        editor: null,
        highlight: false,
        // will be an HTMLCollection, set in mounted()
        imageEls: [],
        imageFields: [],
        formulasMenu: {
          isOpen: false,
          formula: '',
          anchorArrowSide: null,
          style: {
            top: 'initial',
            left: 'initial',
            right: 'initial',
          },
        },
        imagesMenu: {
          isOpen: false,
          anchorArrowSide: null,
          src: '',
          alt: '',
          style: {
            top: 'initial',
            left: 'initial',
            right: 'initial',
          },
        },
        activeImageComponent: null,
        uploadingChecksum: '',
        mathQuill: null,
        keyDownEventListener: null,
        clickEventListener: null,
      };
    },
    computed: {
      // Disabling next line as it's used to watch dropped in images
      // eslint-disable-next-line kolibri/vue-no-unused-properties
      file() {
        return this.getFileUpload(this.uploadingChecksum);
      },
    },
    watch: {
      markdown(newMd, previousMd) {
        if (newMd !== previousMd && newMd !== this.editor.getMarkdown()) {
          this.editor.setMarkdown(newMd);
          this.updateCustomNodeBufferSpaces();
          this.initImageFields();
        }
      },
      imageEls() {
        this.initImageFields();
        this.cleanUpImageFields();
      },
      'file.error'() {
        // eslint-disable-next-line
        console.error('The image could not be uploaded');
      },
      'file.progress'(progress) {
        if (progress === 0) {
          // eslint-disable-next-line
          console.log('The image upload has started');
        } else if (progress === 1) {
          // eslint-disable-next-line
          console.log('The image upload has finished');
          this.insertImageToEditor({ src: this.file.url, alt: '' });
          this.uploadingChecksum = '';
        }
      },
    },
    mounted() {
      this.mathQuill = MathQuill.getInterface(2);

      // This is currently the only way of inheriting and adjusting
      // default TUI's convertor methods
      // see https://github.com/nhn/tui.editor/issues/615
      const tmpEditor = new Editor({
        el: this.$refs.editor,
      });
      const Convertor = tmpEditor.convertor.constructor;
      class CustomConvertor extends Convertor {
        toMarkdown(html, toMarkOptions) {
          // Delete the img rule to prevent images from getting
          // pre-parsed (need to factor in width and height)
          delete toMarkOptions.renderer.rules.IMG;
          let content = super.toMarkdown(html, toMarkOptions);
          content = formulaHtmlToMd(content);
          content = imagesHtmlToMd(content);
          content = content.replaceAll('&nbsp;', ' ');
          return content;
        }
      }
      tmpEditor.remove();

      const createBoldButton = () => {
        {
          const button = document.createElement('button');
          button.className = 'tui-bold tui-toolbar-icons';
          return button;
        }
      };
      const createItalicButton = () => {
        {
          const button = document.createElement('button');
          button.className = 'tui-italic tui-toolbar-icons';
          return button;
        }
      };
      const options = {
        el: this.$refs.editor,
        minHeight: '240px',
        height: 'auto',
        initialValue: this.markdown,
        initialEditType: 'wysiwyg',
        usageStatistics: false,
        toolbarItems: [
          {
            type: 'button',
            options: {
              el: createBoldButton(),
              command: 'Bold',
              tooltip: this.$tr('bold'),
            },
          },
          {
            type: 'button',
            options: {
              el: createItalicButton(),
              command: 'Italic',
              tooltip: this.$tr('italic'),
            },
          },
        ],
        hideModeSwitch: true,
        plugins: [
          [
            imageUpload,
            {
              onImageDrop: this.onImageDrop,
              onImageUploadToolbarBtnClick: this.onImageUploadToolbarBtnClick,
              toolbarBtnTooltip: this.$tr('image'),
            },
          ],
          [
            formulas,
            {
              onFormulasToolbarBtnClick: this.onFormulasToolbarBtnClick,
              toolbarBtnTooltip: this.$tr('formulas'),
            },
          ],
          [
            minimize,
            {
              onMinimizeToolbarBtnClick: this.onMinimizeToolbarBtnClick,
              toolbarBtnTooltip: this.$tr('minimize'),
            },
          ],
        ],
        customConvertor: CustomConvertor,
        // https://github.com/nhn/tui.editor/blob/master/apps/editor/docs/custom-html-renderer.md
        customHTMLRenderer: {
          text(node) {
            let content = formulaMdToHtml(node.literal);
            content = imagesMdToHtml(content);
            return {
              type: 'html',
              content,
            };
          },
        },
      };

      this.editor = new Editor(options);

      this.editor.focus();

      this.editor.on('change', () => {
        this.$emit('update', this.editor.getMarkdown());
        this.$set(this.imageEls, this.$el.getElementsByTagName('img'));
      });

      this.initMathFields();

      this.$set(this.imageEls, this.$el.getElementsByTagName('img'));

      this.editor.getSquire().addEventListener('willPaste', this.onPaste);
      this.keyDownEventListener = this.$el.addEventListener('keydown', this.onKeyDown, true);
      this.clickEventListener = this.$el.addEventListener('click', this.onClick);

      // Make sure all custom nodes have buffer spaces around them.
      // Note: this is debounced because it's called every keystroke
      const editorEl = this.$refs.editor;
      this.updateCustomNodeBufferSpaces = debounce(() => {
        editorEl.querySelectorAll('span[is]').forEach(el => {
          el.editing = true;
          const hasLeftwardSpace = el => {
            return (
              el.previousSibling &&
              el.previousSibling.textContent &&
              el.previousSibling.textContent.match(/\s$/)
            );
          };
          const hasRightwardSpace = el => {
            return (
              el.nextSibling &&
              el.nextSibling.textContent &&
              el.nextSibling.textContent.match(/^\s/)
            );
          };
          if (!hasLeftwardSpace(el)) {
            el.insertAdjacentText('beforebegin', '\xa0');
          }
          if (!hasRightwardSpace(el)) {
            el.insertAdjacentText('afterend', '\xa0');
          }
        });
      }, 150);

      this.updateCustomNodeBufferSpaces();
    },
    activated() {
      this.editor.focus();
    },
    beforeDestroy() {
      this.editor.getSquire().removeEventListener('willPaste', this.onPaste);
      this.$el.removeEventListener(this.keyDownEventListener, this.onKeyDown, true);
      this.$el.removeEventListener(this.clickEventListener, this.onClick);
    },
    methods: {
      /**
       * Handle Squire keydown events - TUI WYSIWYG editor is built
       * on top of Squire (https://github.com/neilj/Squire)
       * and provides its instance including API methods
       *
       * TUI's `addKeyEventHandler` is not sufficient because they
       * internally override some of the actions that need to be
       * customized from here => needs to be set on Squire level
       * Modifying default Squire key events is not documented but there's
       * a recommended solution here https://github.com/neilj/Squire/issues/107
       */
      onKeyDown(event) {
        const squire = this.editor.getSquire();

        // Apply squire selection workarounds
        this.fixSquireSelectionOnKeyDown(event);

        if (event.key in keyHandlers) {
          keyHandlers[event.key](squire);
        }

        // ESC should close menus if any are open
        // or close the editor if none are open
        if (event.key === 'Escape') {
          event.stopImmediatePropagation();
          if (this.formulasMenu.isOpen) {
            this.resetFormulasMenu();
          } else if (this.imagesMenu.isOpen) {
            this.resetImagesMenu();
          } else {
            this.onMinimizeToolbarBtnClick();
          }
        }

        // Add keyboard shortcuts handlers for custom markdown
        // editor toolbar buttons: image upload (ctrl+p),
        // formulas (ctrl+f), minimize (ctrl+m)
        // Needs to be done here because TUI editor currently
        // doesn't support customizing shortcuts
        // https://github.com/nhn/tui.editor/issues/281
        if (event.ctrlKey === true && event.key === 'p') {
          event.stopImmediatePropagation();
          this.onImageUploadToolbarBtnClick();
        }

        if (event.ctrlKey === true && event.key === 'f') {
          this.onFormulasToolbarBtnClick();
        }

        if (event.ctrlKey === true && event.key === 'm') {
          this.onMinimizeToolbarBtnClick();
        }

        // Allow default keyboard shortcut handlers
        // for supported actions: bold (ctrl+b), italics (ctrl+i),
        // select all (ctrl+a), copy (ctrl+c),
        // cut (ctrl+x), paste (ctrl+v)
        // Disable all remaining default keyboard shortcuts.
        if (event.ctrlKey === true && ['b', 'i', 'a', 'c', 'x', 'v', 'z'].includes(event.key)) {
          return;
        }

        // Disable all remaining Ctrl key shortcuts
        if (event.ctrlKey === true) {
          event.preventDefault();
          event.stopPropagation();
        }

        this.updateCustomNodeBufferSpaces();
      },
      onPaste(event) {
        const fragment = clearNodeFormat({
          node: event.fragment,
          ignore: ['b', 'i'],
        });
        event.fragment = fragment;
      },
      onImageDrop(fileUpload) {
        this.highlight = false;
        this.handleFileUpload([fileUpload])
          .then(files => {
            const fileUpload = files[0];
            if (fileUpload && fileUpload.checksum) {
              this.uploadingChecksum = fileUpload.checksum;
            } else {
              this.uploadingChecksum = '';
            }
          })
          .catch(() => {
            this.uploadingChecksum = '';
          });
      },
      onImageUploadToolbarBtnClick() {
        if (this.imagesMenu.isOpen === true) {
          return;
        }
        this.activeImageComponent = null;

        const cursor = this.editor.getSquire().getCursorPosition();
        const position = getExtensionMenuPosition({
          editorEl: this.$el,
          targetX: cursor.x,
          targetY: cursor.y + cursor.height,
        });
        this.resetImagesMenu();
        this.openImagesMenu({ position });
      },
      onFormulasToolbarBtnClick() {
        if (this.formulasMenu.isOpen === true) {
          return;
        }

        const cursor = this.editor.getSquire().getCursorPosition();
        const formulasMenuPosition = getExtensionMenuPosition({
          editorEl: this.$el,
          targetX: cursor.x,
          targetY: cursor.y + cursor.height,
        });

        this.resetFormulasMenu();
        this.openFormulasMenu({ position: formulasMenuPosition });
      },
      onMinimizeToolbarBtnClick() {
        this.$emit('minimize');
        // Make sure tooltip gets removed from screen
        document.querySelectorAll('.tui-tooltip').forEach(tooltip => {
          tooltip.style.display = 'none';
        });
      },
      fixSquireSelectionOnKeyDown(event) {
        /**
         *  On 'backspace' events, Squire doesn't behave consistently in both Chrome and FireFox,
         *  particularly when dealing with custom elements or elements with `contenteditable=false`.
         *
         *  This function modifies the selection with the intention of correcting it before Squire
         *  handles it in the usual way.
         *
         *  This is a tricky workaround, so please edit this function with care.
         */

        const squire = this.editor.getSquire();
        const selection = squire.getSelection();

        // Prevent Squire from deleting custom editor nodes when the cursor is left of one.
        const isCustomNode = node => node && node.hasAttribute && node.hasAttribute('is');
        const getElementAtRelativeOffset = (selection, offset) =>
          selection &&
          squire.getSelectionInfoByOffset(selection.endContainer, selection.endOffset + offset)
            .element;
        const getLeftwardElement = selection => getElementAtRelativeOffset(selection, -1);
        const getRightwardElement = selection => getElementAtRelativeOffset(selection, 1);
        const moveCursor = (selection, amount) => {
          if (amount > 0) {
            selection.setStart(selection.startContainer, selection.startOffset + amount);
          } else {
            selection.setEnd(selection.endContainer, Math.max(0, selection.endOffset + amount));
          }
          return selection;
        };
        // make sure Squire doesn't delete rightward custom nodes when 'backspace' is pressed
        if (event.key !== 'ArrowRight' && event.key !== 'Delete') {
          if (isCustomNode(getRightwardElement(selection))) {
            squire.setSelection(moveCursor(selection, -1));
          }
        }
        // make sure Squire doesn't get stuck with a broken cursor position when deleting
        // elements with `contenteditable="false"` in FireFox
        let leftwardElement = getLeftwardElement(selection);
        if (event.key === 'Backspace') {
          if (selection.startContainer.tagName === 'DIV') {
            // This happens normally when deleting from the beginning of an empty line...
            if (isCustomNode(selection.startContainer.childNodes[selection.startOffset - 1])) {
              // ...but on FireFox it also happens if you press 'backspace' and the leftward
              // element has `contenteditable="false"` (which is necessary on FireFox for
              // a different reason).  As a result, Squire.js gets stuck. The trick here is
              // to fix its selection so it knows what to delete.
              let fixedStartContainer =
                selection.startContainer.childNodes[selection.startOffset - 1];
              let fixedEndContainer = selection.endContainer.childNodes[selection.endOffset - 1];
              if (fixedStartContainer && fixedEndContainer) {
                selection.setStart(fixedStartContainer, 0);
                selection.setEnd(fixedEndContainer, 1);
                squire.setSelection(selection);
              }
            }
          } else if (isCustomNode(leftwardElement)) {
            // In general, if the cursor is to the right of a custom node and 'backspace'
            // is pressed, add that node to the selection so that it will be deleted.
            selection.setStart(leftwardElement, 0);
            squire.setSelection(selection);
          } else if (isCustomNode(selection.startContainer)) {
            // if any part of a custom node is in the selection, include the whole thing
            selection.setStart(selection.startContainer, 0);
            squire.setSelection(selection);
          } else if (isCustomNode(selection.endContainer)) {
            selection.setEnd(selection.endContainer, 0);
            squire.setSelection(selection);
          }
        }
      },
      onClick(event) {
        this.highlight = false;
        const target = event.target;

        let mathFieldEl = null;
        if (target.getAttribute('is') === 'markdown-formula') {
          mathFieldEl = target;
        }
        const clickedOnMathField = mathFieldEl !== null;
        const clickedOnActiveMathField =
          clickedOnMathField && mathFieldEl.classList.contains(CLASS_MATH_FIELD_ACTIVE);
        const clickedOnFormulasMenu =
          target.classList.contains('formulas-menu') || target.closest('.formulas-menu');
        const clickedOnImagesMenu =
          target.classList.contains('images-menu') || target.closest('.images-menu');
        const clickedOnEditorToolbarBtn = target.classList.contains('tui-toolbar-icons');

        // skip markdown editor toolbar buttons clicks
        // they have their own handlers defined
        if (clickedOnEditorToolbarBtn) {
          return;
        }

        // no need to do anything when the formulas menu clicked
        if (clickedOnFormulasMenu) {
          return;
        }

        // no need to do anything when an active math field clicked
        if (clickedOnActiveMathField) {
          return;
        }

        // no need to do anything when the images menu clicked
        if (clickedOnImagesMenu) {
          return;
        }

        // if clicked outside of open formulas menu
        // and active math field then close the formulas
        // menu and clear data related to its previous state
        if (this.formulasMenu.isOpen) {
          this.insertFormulaToEditor();
          this.removeMathFieldActiveClass();
          this.resetFormulasMenu();
        }

        // if clicked outside of open images menu close images
        // menu and clear data related to its previous state
        if (this.imagesMenu.isOpen) {
          this.resetImagesMenu();
        }

        // no need to continue if regular text clicked
        if (!clickedOnMathField) {
          return;
        }

        // open formulas menu if a math field clicked
        const formulasMenuFormula = mathFieldEl.getVueInstance().latex;
        const formulasMenuPosition = getExtensionMenuPosition({
          editorEl: this.$el,
          targetX: mathFieldEl.getBoundingClientRect().left,
          targetY: mathFieldEl.getBoundingClientRect().bottom,
        });
        // just a little visual enhancement to make clear
        // that the formula menu is linked to a math field
        // element being edited
        if (formulasMenuPosition.left !== null) {
          formulasMenuPosition.left += 10;
        }
        if (formulasMenuPosition.right !== null) {
          formulasMenuPosition.right -= 10;
        }

        mathFieldEl.classList.add(CLASS_MATH_FIELD_ACTIVE);

        // `nextTick` - to be sure that the formula
        // was reset before a new formula set
        // important when a math field is being edited in open
        // formulas menu and another math field is clicked
        this.$nextTick(() => {
          this.openFormulasMenu({
            position: formulasMenuPosition,
            formula: formulasMenuFormula,
          });
        });
      },
      onFormulasMenuInsert() {
        this.insertFormulaToEditor();

        this.removeMathFieldActiveClass();
        this.resetFormulasMenu();
        this.editor.focus();
      },
      onFormulasMenuCancel() {
        this.removeMathFieldActiveClass();
        this.resetFormulasMenu();
        this.editor.focus();
      },
      // Set custom `markdown-formula` nodes as `editing=true`.
      initMathFields() {
        this.$el.querySelectorAll('span[is="markdown-formula"]').forEach(el => {
          el.editing = true;
        });
      },
      findActiveMathField() {
        return this.$el.getElementsByClassName(CLASS_MATH_FIELD_ACTIVE)[0] || null;
      },
      removeMathFieldActiveClass() {
        const activeMathField = this.findActiveMathField();

        if (activeMathField !== null) {
          activeMathField.classList.remove(CLASS_MATH_FIELD_ACTIVE);
        }
      },
      openFormulasMenu({ position, formula = null }) {
        this.resetMenus();
        const top = `${position.top}px`;

        let left, right, anchorArrowSide;

        if (position.left !== null) {
          right = 'initial';
          left = `${position.left}px`;
          anchorArrowSide = 'left';
        } else {
          left = 'initial';
          right = `${position.right}px`;
          anchorArrowSide = 'right';
        }

        this.formulasMenu = {
          isOpen: true,
          formula: formula !== null ? formula : '',
          anchorArrowSide,
          style: {
            top,
            left,
            right,
          },
        };
      },
      /**
       * Insert formula from formulas menu to markdown editor.
       * If there is an active math field, replace it by a new
       * element with the updated formula. Otherwise insert a new
       * element with the formula on a current cursor position.
       */
      insertFormulaToEditor() {
        const formula = this.formulasMenu.formula;

        if (!formula) {
          return;
        }

        const formulaEl = document.createElement('span');
        formulaEl.setAttribute('is', 'markdown-formula');
        formulaEl.setAttribute('editing', true);
        formulaEl.innerHTML = formula;
        let formulaHTML = formulaEl.outerHTML;
        const activeMathFieldEl = this.findActiveMathField();

        if (activeMathFieldEl !== null) {
          activeMathFieldEl.outerHTML = formulaHTML;
        } else {
          // insert non-breaking spaces to allow users to write text before and after
          let squire = this.editor.getSquire();
          squire.insertHTML(wrapWithSpaces(formulaHTML));
        }
      },
      resetFormulasMenu() {
        this.formulasMenu = {
          isOpen: false,
          formula: '',
          anchorArrowSide: null,
          style: {
            top: 'initial',
            left: 'initial',
            right: 'initial',
          },
        };
      },

      /* IMAGE MENU */
      /**
       * Initialize elements with image field class with ImageField component
       */
      initImageFields({ newOnly = false } = {}) {
        for (let imageEl of this.imageEls) {
          if (!newOnly || imageEl.classList.contains(CLASS_IMG_FIELD_NEW)) {
            const ImageComponent = new ImageFieldClass({
              propsData: {
                src: imageEl.getAttribute('src'),
                alt: imageEl.getAttribute('alt'),
                width: imageEl.getAttribute('width'),
                height: imageEl.getAttribute('height'),
              },
            });
            ImageComponent.$mount();
            ImageComponent.$on('edit', ({ event, component, image }) => {
              this.activeImageComponent = component;
              const position = getExtensionMenuPosition({
                editorEl: this.$el,
                targetX: event.clientX,
                targetY: event.clientY,
              });
              this.openImagesMenu({
                position,
                alt: image.alt,
                src: image.src,
              });
            });
            imageEl.replaceWith(ImageComponent.$el);
            imageEl.classList.remove(CLASS_IMG_FIELD_NEW);
            // add to tracking array.
            this.imageFields.push(ImageComponent);
          }
        }
      },
      cleanUpImageFields() {
        this.imageFields.forEach((imageField, index) => {
          // Editor only removes <img> reliably - div and other elements remain
          const imageFieldImg = imageField.$el.getElementsByTagName('img')[0];
          const imageHasBeenDeleted = !this.imageEls.includes(imageFieldImg);

          if (imageHasBeenDeleted) {
            // Unmount and remove all listeners
            imageField.$destroy();
            // Delete object from the array (will leave undefined)
            delete this.imageFields[index];
          }
        });

        // Clean out all falsey (undefined, here) objects from imageFields
        this.imageFields = this.imageFields.filter(imageField => !!imageField);
      },
      openImagesMenu({ position, src = '', alt = '' }) {
        this.resetMenus();
        const top = `${position.top}px`;

        let left, right, anchorArrowSide;

        if (position.left !== null) {
          right = 'initial';
          left = `${position.left}px`;
          anchorArrowSide = 'left';
        } else {
          left = 'initial';
          right = `${position.right}px`;
          anchorArrowSide = 'right';
        }
        this.imagesMenu = {
          isOpen: true,
          anchorArrowSide,
          src,
          alt,
          style: {
            top,
            left,
            right,
          },
        };
      },
      /**
       * Insert image from images menu to markdown editor.
       * Emit an uploaded event to let parent process files
       * properly
       */
      insertImageToEditor(imageData) {
        if (!imageData) {
          return;
        }
        if (this.activeImageComponent) {
          this.activeImageComponent.setImageData(imageData);
        } else {
          // Create a "dummy" HTML element for Vue to attach to
          const imageEl = document.createElement('img');
          imageEl.classList.add(CLASS_IMG_FIELD_NEW);
          imageEl.src = imageData.src;
          imageEl.alt = imageData.alt;

          // insert non-breaking spaces to allow users to write text before and after
          this.editor.getSquire().insertHTML(wrapWithSpaces(imageEl.outerHTML));

          this.initImageFields({ newOnly: true });
        }
        this.resetImagesMenu();
      },
      onImagesMenuCancel() {
        this.resetImagesMenu();
        this.editor.focus();
      },
      resetImagesMenu() {
        this.imagesMenu = {
          isOpen: false,
          anchorArrowSide: null,
          src: '',
          alt: '',
          style: {
            top: 'initial',
            left: 'initial',
            right: 'initial',
          },
        };
      },
      resetMenus() {
        this.resetImagesMenu();
        this.resetFormulasMenu();
      },
    },
    $trs: {
      bold: 'Bold (Ctrl+B)',
      italic: 'Italic (Ctrl+I)',
      image: 'Insert image (Ctrl+P)',
      formulas: 'Insert formula (Ctrl+F)',
      minimize: 'Minimize (Ctrl+M)',
    },
  };

</script>

<style lang="less" scoped>

  @import '../mathquill/mathquill.css';

  .uploading {
    cursor: progress;
  }

  .formulas-menu,
  .images-menu {
    z-index: 2;
  }

  .wrapper {
    border: 4px solid transparent;
    &.highlight {
      border-color: var(--v-primary-base);
    }
  }

  // TODO (when updating to new frontend files structure)
  // find better location for following styles that
  // are supposed to be common to all editable fields
  /deep/ .mq-editable-field {
    border: 0;

    .mq-to,
    .mq-from,
    .mq-sup,
    .mq-sup-inner,
    .mq-sub,
    .mq-numerator,
    .mq-denominator {
      padding-right: 1px;
      padding-left: 1px;
      border: 1px solid gray;
    }

    .mq-int .mq-sup {
      border: 0;
    }
  }

</style>
