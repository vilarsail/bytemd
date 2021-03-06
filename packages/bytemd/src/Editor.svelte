<script>
  import { onMount, createEventDispatcher } from 'svelte';
  import codemirror from 'codemirror';
  import 'codemirror/mode/markdown/markdown.js';
  import Toolbar from './Toolbar.svelte';
  import Viewer from './Viewer.svelte';
  import { dataUrlFileHandler } from './utils.js';

  const dispatch = createEventDispatcher();

  export let value = '';
  export let containerStyle;
  export let fileHandler = dataUrlFileHandler;
  export let plugins = [];
  export let mode = 'split';
  export let editorConfig;

  let textarea;
  let viewer;
  let cm;

  let activeTab = 0;
  function setActiveTab(e) {
    activeTab = e.detail.value;
  }

  $: if (cm && value !== cm.getValue()) {
    cm.setValue(value);
  }

  onMount(() => {
    cm = codemirror.fromTextArea(textarea, {
      mode: 'markdown',
      lineNumbers: true,
      lineWrapping: true,
      ...editorConfig,
    });
    cm.setValue(value);
    cm.on('change', (doc, change) => {
      if (change.origin !== 'setValue') {
        dispatch('change', { value: cm.getValue() });
      }
    });
    cm.on('scroll', cm => {
      requestAnimationFrame(() => {
        const editorInfo = cm.getScrollInfo();
        const ratio =
          editorInfo.top / (editorInfo.height - editorInfo.clientHeight);
        viewer.scrollTo(0, ratio * (viewer.scrollHeight - viewer.clientHeight));
      });
    });
    cm.on('paste', async (_, e) => {
      const { items } = e.clipboardData;
      for (let i = 0; i < items.length; i++) {
        // console.log(items[i]);
        if (!items[i].type.startsWith('image/')) continue;

        e.preventDefault();
        const url = await fileHandler(items[i].getAsFile());
        const text = cm.getSelection();
        cm.replaceSelection(`![${text}](${url})`);
        cm.focus();
        return;
      }
    });
    cm.on('drop', async (_, e) => {
      const { items } = e.dataTransfer;
      for (let i = 0; i < items.length; i++) {
        if (!items[i].type.startsWith('image/')) continue;

        e.preventDefault();
        const url = await fileHandler(items[i].getAsFile());
        const text = cm.getSelection();
        cm.replaceSelection(`![${text}](${url})`);
        cm.focus();
        return;
      }
    });
  });
</script>

<style>
  .bytemd {
    border: 1px solid #eee;
    display: flex;
    flex-direction: column;
  }
  .bytemd-body {
    flex-grow: 1;
    display: flex;
    overflow: auto;
  }
  .bytemd-editor {
    flex: 1;
    overflow: hidden;
    height: 100%;
  }
  .bytemd-editor :global(.CodeMirror) {
    height: 100%;
  }
  .bytemd-viewer {
    padding: 20px;
    flex: 1;
    overflow: auto;
    border-left: 1px solid #eee;
  }
  .hidden {
    display: none;
  }
</style>

<div class="bytemd" style={containerStyle}>
  <Toolbar
    {cm}
    {fileHandler}
    {plugins}
    {mode}
    {activeTab}
    on:tab={setActiveTab} />
  <div class="bytemd-body">
    <div class="bytemd-editor" class:hidden={mode === 'tab' && activeTab === 1}>
      <textarea bind:this={textarea} />
    </div>
    <div
      class="bytemd-viewer"
      bind:this={viewer}
      class:hidden={mode === 'tab' && activeTab === 0}>
      <Viewer {value} {plugins} />
    </div>
  </div>
</div>
