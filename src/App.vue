<script lang="ts">
import { Component, Vue } from 'vue-property-decorator';
import RibbonMenu from '@/RibbonMenu.vue';
import FormulaBar from '@/FormulaBar.vue';
import OpenSheet from '@/OpenSheet.vue';
import SheetSelector from '@/SheetSelector.vue';
import StatusBar from '@/StatusBar.vue';
import { OpenDocument } from '@/lib/core/Document';
import { UI } from '@/lib/UI';
import { Appearance } from '@/lib/core/Appearance';

@Component({
  components: {
    StatusBar,
    SheetSelector,
    OpenSheet,
    FormulaBar,
    RibbonMenu,
  },
})
export default class App extends Vue {
  private uiManager!: UI;
  private appearance: Appearance = new Appearance();
  private fileMode: 'local' | 'file' = 'file';
  private state = {
    isMerged: false,
    label: 'A1',
    value: null,
    originalValue: null,
  };

  public onReady(ui: UI) {
    this.uiManager = ui;
  }

  public onChange(doc: OpenDocument) {
    this.appearance = doc.ActiveSheet.SelectedAppearance;
    this.state.isMerged = this.uiManager.isMerged;
    this.state.label = doc.ActiveSheet.SelectionLabel;
    this.state.value = doc.ActiveSheet.SelectedValue;
    this.state.originalValue = doc.ActiveSheet.SelectedValue;

    if (this.fileMode === 'local') {
      localStorage.setItem('data', JSON.stringify(doc.save()));
    }
  }

  public onCommit() {
    if (this.state.value !== this.state.originalValue) {
      this.uiManager.execCmd('change-value', null, null, this.state.value);
    }
  }

  public onAbort() {
    this.state.value = this.state.originalValue;
  }

  public onAction({actionName, args}: { actionName: string, args: any }) {
    if (actionName === 'save-ows') {
      this.onSave();
      return;
    } else if (actionName === 'load-ows') {
      this.onLoad();
      return;
    }
    this.uiManager.execCmd(actionName, args);
  }

  public onModeChanged(newMode: 'file' | 'local') {
    if (this.fileMode === 'file' && localStorage.getItem('data') !== null) {
      (this.$refs.sheet as any).load(localStorage.getItem('data'));
    }
    this.fileMode = newMode;
  }

  public onLoad() {
    const win = window as any;
    const ui = this.uiManager;

    if (win.File && win.FileReader && win.FileList && win.Blob) {
      const f = document.createElement('input');
      f.type = 'file';
      f.accept = '.ows';
      f.addEventListener('change', function(evt: any) {
        const file = evt.target.files[0];
        const reader = new FileReader();
        reader.addEventListener('load', function(loadEvt: any) {
          const rawData = loadEvt.target.result;
          ui.load(rawData);
        });
        reader.readAsText(file, 'utf8');
      });
      f.click();
    } else {
      alert('The File APIs are not fully supported in this browser.');
    }
  }

  public onSave() {
    const content = this.uiManager.save();
    const uriContent = 'data:application/octet-stream,' + encodeURIComponent(content);
    const a = document.createElement('a');
    a.download = 'document.ows';
    a.href = uriContent;
    a.target = '_blank';
    a.click();
  }

}
</script>

<template>
    <div id="app">
        <ribbon-menu :appearance="appearance"
                     :state="state"
                     @mode-changed="onModeChanged"
                     @action="onAction"/>
        <formula-bar :label="state.label"
                     v-model="state.value"
                     @commit="onCommit"
                     @abort="onAbort"/>
        <open-sheet ref="sheet" @ready="onReady" @change="onChange"/>
        <status-bar/>
    </div>
</template>

<style lang="scss">

</style>
