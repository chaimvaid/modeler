<template>
  <div class="inspector-container">
    <vue-form-renderer
      v-if="highlightedNode"
      :data="data"
      @update="updateDefinition"
      :config="config"
    />
  </div>
</template>

<script>
import Vue from 'vue';
import { VueFormRenderer, renderer } from '@processmaker/vue-form-builder';
import {
  FormInput,
  FormSelect,
  FormTextArea,
  FormCheckbox,
  FormRadioButtonGroup,
  FormCodeEditor,
} from '@processmaker/vue-form-elements';
import store, { saveDebounce } from '@/store';
import { id as sequenceFlowId } from '@/components/nodes/sequenceFlow';
import noop from 'lodash/noop';
import debounce from 'lodash/debounce';
import omit from 'lodash/omit';
import processInspectorConfig from './process';
import sequenceExpressionInspectorConfig from './sequenceExpression';

Vue.component('FormText', renderer.FormText);
Vue.component('FormInput', FormInput);
Vue.component('FormSelect', FormSelect);
Vue.component('FormTextArea', FormTextArea);
Vue.component('FormCheckbox', FormCheckbox);
Vue.component('FormRadioButtonGroup', FormRadioButtonGroup);
Vue.component('FormCodeEditor', FormCodeEditor);
Vue.component('VueFormRenderer', VueFormRenderer);

export default {
  props: ['nodeRegistry', 'moddle', 'processNode'],
  data() {
    return {
      inspectorHandler: null,
    };
  },
  computed: {
    highlightedNode() {
      return store.getters.highlightedNode;
    },
    config() {
      if (!this.highlightedNode) {
        return {
          name: 'Empty',
          items: [],
        };
      }

      const { type, definition } = this.highlightedNode;

      if (this.highlightedNode === this.processNode) {
        return processInspectorConfig;
      }

      if (
        type === sequenceFlowId &&
        ['bpmn:ExclusiveGateway', 'bpmn:InclusiveGateway'].includes(definition.sourceRef.$type)
      ) {
        return sequenceExpressionInspectorConfig;
      }

      return this.nodeRegistry[type].inspectorConfig;
    },
    updateDefinition() {
      if (!this.highlightedNode) {
        return noop;
      }

      if (this.highlightedNode === this.processNode) {
        return value => this.defaultInspectorHandler(omit(value, ['artifacts', 'flowElements', 'laneSets']), this.processNode, this.setNodeProp);
      }

      return this.nodeRegistry[this.highlightedNode.type].inspectorHandler
        ? value => this.nodeRegistry[this.highlightedNode.type].inspectorHandler(value, this.highlightedNode, this.setNodeProp, this.moddle)
        : value => this.defaultInspectorHandler(value, this.highlightedNode, this.setNodeProp);
    },
    data() {
      if (!this.highlightedNode) {
        return {};
      }

      const type = this.highlightedNode && this.highlightedNode.type;

      return type && this.nodeRegistry[type].inspectorData
        ? this.nodeRegistry[type].inspectorData(this.highlightedNode)
        : Object.entries(this.highlightedNode.definition).reduce((data, [key, value]) => {
          data[key] = value;
          return data;
        }, {});
    },
  },
  methods: {
    setNodeProp: debounce(function(node, key, value) {
      store.commit('updateNodeProp', { node, key, value });
      this.$emit('save-state');
    }, saveDebounce),
    defaultInspectorHandler(value, node, setNodeProp) {
      /* Go through each property and rebind it to our data */
      for (const key in value) {
        if (node.definition.get(key) !== value[key]) {
          setNodeProp(node, key, value[key]);
        }
      }
    },
  },
};
</script>

<style>
.inspector-container {
  overflow-y: auto;
  font-size: 0.75em;
  text-align: left;
  padding: 8px;
  width: 320px;
  background-color: #eee;
  border-left: 1px solid #aaa;
}
</style>
