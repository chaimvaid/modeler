<template>
  <div class="modeler">
    <div class="modeler-container">
      <controls :controls="controls" :style="{ height: parentHeight }"/>

      <div
        class="paper-container"
        :class="cursor"
        :style="{ width: parentWidth, height: parentHeight }"
      >
        <div class="history-buttons">
          <button @click="undo" :disabled="!canUndo">Undo</button>
          <button @click="redo" :disabled="!canRedo">Redo</button>
        </div>

        <drop @drop="handleDrop" @dragover="validateDropTarget">
          <div ref="paper"/>
        </drop>
      </div>

      <InspectorPanel
        :style="{ height: parentHeight }"
        :nodeRegistry="nodeRegistry"
        :moddle="moddle"
        :processNode="processNode"
        @save-state="pushToUndoStack"
      />
    </div>

    <component
      v-for="node in nodes"
      :is="node.type"
      :key="node.definition.id"
      :graph="graph"
      :paper="paper"
      :node="node"
      :id="node.definition.id"
      :highlighted="highlightedNode === node"
      :collaboration="collaboration"
      :process-node="processNode"
      :processes="processes"
      :plane-elements="planeElements"
      :moddle="moddle"
      :nodeRegistry="nodeRegistry"
      @add-node="addNode"
      @remove-node="removeNode"
      @set-cursor="cursor = $event"
      @set-pool-target="poolTarget = $event"
      @click="highlightNode(node)"
      @unsetPools="unsetPools"
      @setPools="setPools"
      @save-state="pushToUndoStack"
    />
  </div>
</template>

<script>
import Vue from 'vue';
import joint from 'jointjs';
import BpmnModdle from 'bpmn-moddle';
import controls from './controls';
import { highlightPadding } from '@/mixins/crownConfig';
import uniqueId from 'lodash/uniqueId';
import pull from 'lodash/pull';
import debounce from 'lodash/debounce';
import { startEvent } from '@/components/nodes';
import store from '@/store';
import InspectorPanel from '@/components/inspectors/InspectorPanel';
import undoRedoStore from '@/undoRedoStore';

// Our renderer for our inspector
import { Drop } from 'vue-drag-drop';

import { id as poolId } from './nodes/pool';
import { id as laneId } from './nodes/poolLane';
import { id as sequenceFlowId } from './nodes/sequenceFlow';
import { id as associationId } from './nodes/association';

const version = '1.0';

export default {
  components: {
    Drop,
    controls,
    InspectorPanel,
  },
  data() {
    return {
      /* Custom parsers for handling certain bpmn node types */
      parsers: {},

      // What bpmn moddle extensions should we register
      extensions: [],

      // Our controls/nodes to show in our palette
      controls: {},

      // Our node types, keyed by the id
      nodeRegistry: {},

      // Our jointjs data graph model
      graph: null,

      // Our jointjs paper
      paper: null,

      definitions: null,
      context: null,
      planeElements: null,
      canvasDragPosition: null,
      processNode: null,
      collaboration: null,
      moddle: null,
      dragPoint: { x: null, y: null },
      allowDrop: true,
      poolTarget: null,
      processes: [],
      cursor: null,
      parentHeight: null,
      parentWidth: null,
    };
  },
  computed: {
    nodes: () => store.getters.nodes,
    canUndo() {
      return undoRedoStore.getters.canUndo;
    },
    canRedo() {
      return undoRedoStore.getters.canRedo;
    },
    currentXML() {
      return undoRedoStore.getters.currentState;
    },
    highlightedNode: () => store.getters.highlightedNode,
  },
  methods: {
    pushToUndoStack() {
      this.toXML((err,xml) => {
        undoRedoStore.dispatch('pushState', xml);
      });
    },
    undo() {
      undoRedoStore
        .dispatch('undo')
        .then(this.loadXML.cancel)
        .then(this.loadXML);
    },
    redo() {
      undoRedoStore
        .dispatch('redo')
        .then(this.loadXML.cancel)
        .then(this.loadXML);
    },
    setPools(poolDefinition) {
      if (!this.collaboration) {
        this.collaboration = this.moddle.create('bpmn:Collaboration');
        this.definitions.get('rootElements').push(this.collaboration);
        this.collaboration.set('id', 'collaboration_0');
        this.plane.set('bpmnElement', this.collaboration);
      }

      if (poolDefinition.get('processRef')) {
        if (!this.collaboration.get('participants').includes(poolDefinition)) {
          this.collaboration.get('participants').push(poolDefinition);
        }

        return;
      }

      let process;
      if (this.collaboration.get('participants').length === 0) {
        process = this.processNode.definition;
      } else {
        process = this.moddle.create('bpmn:Process');
        this.processes.push(process);
        process.set('id', `process_${this.processes.length}`);
        process.set('isExecutable', false);

        this.definitions.get('rootElements').push(process);
      }

      poolDefinition.set('processRef', process);
      this.collaboration.get('participants').push(poolDefinition);
    },
    unsetPools() {
      pull(this.definitions.get('rootElements'), this.collaboration);
      this.plane.set('bpmnElement', this.processNode.definition);
      this.collaboration = null;
    },
    highlightNode(node) {
      store.commit('highlightNode', node);
    },
    /**
     * Register an inspector component to configure extended attributes and elements
     * for specific bpmn extensions and execution environments. If the inspector
     * component is already registered, it is replaced by the new one.
     */
    registerInspectorExtension(node, config) {
      const registeredIndex = node.inspectorConfig[0].items.findIndex(item => {
        return config.id && config.id === item.id;
      });
      if (registeredIndex === -1) {
        node.inspectorConfig[0].items.push(config);
      } else {
        node.inspectorConfig[0].items[registeredIndex] = config;
      }
    },
    /**
     * Register a BPMN Moddle extension in order to support extensions to the bpmn xml format.
     * This is used to support new attributes and elements that would be needed for specific
     * bpmn execution environments.
     */
    registerBpmnExtension(namespace, extension) {
      this.extensions[namespace] = extension;
    },
    // This registers a node to use in the bpmn modeler
    registerNode(nodeType, customParser) {
      const defaultParser = nodeType.implementation
        ? definition => definition.get('implementation') === nodeType.implementation && nodeType.id
        : () => nodeType.id;

      this.nodeRegistry[nodeType.id] = nodeType;

      Vue.component(nodeType.id, nodeType.component);

      if (nodeType.control) {
        // Register the control for our control palette
        if (!this.controls[nodeType.category]) {
          this.$set(this.controls, nodeType.category, []);
        }

        this.controls[nodeType.category].push({
          type: nodeType.id,
          icon: nodeType.icon,
          label: nodeType.label,
        });
      }

      const types = Array.isArray(nodeType.bpmnType)
        ? nodeType.bpmnType
        : [nodeType.bpmnType];

      types.forEach(bpmnType => {
        const parser = customParser || defaultParser;

        this.parsers[bpmnType]
          ? this.parsers[bpmnType].push(parser)
          : this.parsers[bpmnType] = [parser];
      });
    },
    // Parses our definitions and graphs and stores them in our id based lookup model
    parse() {
      // Get the top level objects
      // All root elements are either bpmn:process or bpmn:collaboration types
      // There should only be one collaboration

      this.collaboration = this.definitions.rootElements.find(({ $type }) => $type === 'bpmn:Collaboration');
      this.processes = this.definitions.rootElements.filter(({ $type }) => $type === 'bpmn:Process');

      /* Get the diagram; there should only be one diagram. */
      this.plane = this.definitions.diagrams[0].plane;
      this.planeElements = this.plane.get('planeElement');

      this.processNode = {
        definition: this.processes[0],
        diagram: this.planeElements.find(diagram => diagram.bpmnElement.id === this.processes[0].id),
      };

      /* Add any pools */
      if (this.collaboration) {
        this.collaboration.get('participants').forEach(this.setNode);
      }

      /* Iterate through all elements in each process. */
      this.processes.forEach(process => {
        /* Add any lanes */
        if (process.get('laneSets')[0]) {
          process.laneSets[0].lanes.forEach(this.setNode);
        }

        /* Add all other elements */

        const flowElements = process.get('flowElements');
        const artifacts = process.get('artifacts');


        /* First load the flow elements */
        flowElements
          .filter(definition => definition.$type !== 'bpmn:SequenceFlow')
          .forEach((definition) => this.setNode(definition, flowElements, artifacts));

        /* Then the sequence flows */
        flowElements
          .filter(definition => {
            if (definition.$type !== 'bpmn:SequenceFlow') {
              return false;
            }

            return this.hasSourceAndTarget(definition);
          })
          .forEach((definition) => this.setNode(definition, flowElements, artifacts));

        /* Then the artifacts */
        artifacts
          .filter(definition => definition.$type !== 'bpmn:Association')
          .forEach((definition) => this.setNode(definition, flowElements, artifacts));

        /* Then the associations */
        artifacts
          .filter(definition => {
            if (definition.$type !== 'bpmn:Association') {
              return false;
            }

            return this.hasSourceAndTarget(definition);
          })
          .forEach((definition) => this.setNode(definition, flowElements, artifacts));
      });

      store.commit('highlightNode', this.processNode);
    },
    setNode(definition, flowElements, artifacts) {
      /* Get the diagram element for the corresponding flow element node. */
      const diagram = this.planeElements.find(diagram => diagram.bpmnElement.id === definition.id);

      if (!this.parsers[definition.$type]) {
        if (process.env.NODE_ENV !== 'production') {
          /* eslint-disable-next-line no-console */
          console.warn(`Unsupported element type in parse: ${definition.$type}`);
        }

        pull(flowElements, [
          definition,
          ...definition.get('incoming'),
          ...definition.get('outgoing'),
        ]);
        pull(artifacts, definition);
        pull(this.planeElements, diagram);

        return;
      }

      const type = this.parsers[definition.$type]
        .reduce((type, parser) => {
          return parser(definition, this.moddle) || type;
        }, null);

      const unnamedElements = ['bpmn:TextAnnotation'];
      const requireName = unnamedElements.indexOf(definition.$type) === -1;
      if (requireName && !definition.get('name')) {
        definition.set('name', '');
      }

      store.commit('addNode', {
        type,
        definition,
        diagram,
      });
    },
    hasSourceAndTarget(definition) {
      const hasSource = definition.sourceRef && this.parsers[definition.sourceRef.$type];
      const hasTarget = definition.targetRef && this.parsers[definition.targetRef.$type];

      return hasSource && hasTarget;
    },
    loadXML(xml = this.currentXML) {
      this.moddle.fromXML(xml, (err, definitions, context) => {
        if (!err) {
          // Update definitions export to our own information
          definitions.exporter = 'ProcessMaker Modeler';
          definitions.exporterVersion = version;
          this.definitions = definitions;
          this.context = context;
          this.initializeUniqueId(context);

          store.commit('clearNodes');

          this.$nextTick(() => {
            this.parse();
            this.$emit('parsed');
          });
        }
      });
    },
    toXML(cb) {
      this.moddle.toXML(this.definitions, { format: true }, cb);
    },
    handleDrop(transferData, event) {
      if (!this.allowDrop) {
        return;
      }

      const type = transferData.type;

      // Add to our processNode
      const definition = this.nodeRegistry[type].definition(this.moddle);

      // Now, let's modify planeElement
      const diagram = this.nodeRegistry[type].diagram(this.moddle);

      // Handle transform
      const paperOrigin = this.paper.localToPagePoint(0, 0);
      diagram.bounds.x = event.clientX - paperOrigin.x;
      diagram.bounds.y = event.clientY - paperOrigin.y;

      // Our BPMN models are updated, now add to our nodes
      // @todo come up with random id
      this.addNode({ type, definition, diagram });
    },
    addNode({ type, definition, diagram }) {
      /*
       * If we are adding a pool, first, create a bpmn:Collaboration, or get the current bpmn:Collaboration,
       * if one exists.
       *
       * For each process, bpmn:Collaboration will contain a bpmn:participant (a pool is a graphical represnetation of a participant).
       * If there are currently no pools, don't create a new process, use the current one instead, and add (embed) all current flow elements to it.
       *
       * For lanes, it will be bpmn:laneSet > bpmn:lanes (TODO).
       */
      if (type !== poolId) {
        /* Check if this.poolTarget is set, and if so, add to appropriate process. */
        const targetProcess = this.poolTarget
          ? this.processes.find(({ id }) => id === this.poolTarget.component.node.definition.get('processRef').id)
          : this.processNode.definition;

        if (type === laneId) {
          targetProcess
            .get('laneSets')[0]
            .get('lanes')
            .push(definition);
        } else if (definition.$type === 'bpmn:TextAnnotation' || definition.$type === 'bpmn:Association' ) {
          targetProcess.get('artifacts').push(definition);
        } else {
          targetProcess.get('flowElements').push(definition);
        }
      }

      const id = uniqueId('node_');
      definition.id = id;

      if (diagram) {
        diagram.id = `${id}_di`;
        diagram.bpmnElement = definition;
      }

      this.planeElements.push(diagram);

      store.commit('addNode', {
        type,
        definition,
        diagram,
        pool: this.poolTarget,
      });

      if (![sequenceFlowId, laneId, associationId].includes(type)) {
        setTimeout(() => this.pushToUndoStack());
      }

      this.poolTarget = null;
    },
    initializeUniqueId(context) {
      let last = uniqueId() * 1;
      context.references.forEach((ref)=>{
        const ma = ref.id.match(/^node_(\d+)$/),
          index = ma && ma[1] * 1;
        while (last < index) last = uniqueId() * 1;
      });
    },
    removeNode(node) {
      store.commit('removeNode', node);
      this.$nextTick(() => {
        this.pushToUndoStack();
      });
    },
    handleResize() {
      const { clientWidth, clientHeight } = this.$el.parentElement;
      this.parentWidth = clientWidth + 'px';
      this.parentHeight = clientHeight + 'px';

      this.paper.setDimensions(clientWidth, clientHeight);
    },
    validateDropTarget(transferData, { clientX, clientY }) {
      /* You can drop a pool anywhere (a pool will not be embedded into another pool) */
      if (transferData.type === poolId) {
        this.allowDrop = true;
        return;
      }

      /* If there are no pools on the grid, allow dragging components anywhere */
      if (
        !this.collaboration ||
        this.collaboration.get('participants').length === 0
      ) {
        this.allowDrop = true;
        return;
      }

      const { x, y } = this.dragPoint;
      if (clientX === x && clientY === y) {
        /* We don't need to re-calcaulte values if mouse position hasn't changed */
        return;
      }

      /* The mouse co-ordinates are set so we can compare them above if this function runs again */
      this.dragPoint = { x: clientX, y: clientY };

      /* Determine if we are over a pool, and only allow dropping elements over a pool */

      const localMousePosition = this.paper.clientToLocalPoint({
        x: clientX,
        y: clientY,
      });
      const pool = this.graph
        .findModelsFromPoint(localMousePosition)
        .find(({ component }) => {
          return component && component.node.type === poolId;
        });

      if (!pool) {
        this.allowDrop = false;
        this.poolTarget = null;
      } else {
        this.allowDrop = true;
        this.poolTarget = pool;
      }
    },
    addStartEvent() {
      /* Add an initial startEvent node if the graph is empty */
      if (this.nodes.length > 0) {
        return;
      }

      const definition = startEvent.definition(this.moddle);
      const diagram = startEvent.diagram(this.moddle);

      diagram.bounds.x = 150;
      diagram.bounds.y = 150;

      this.addNode({
        definition,
        diagram,
        type: startEvent.id,
      });
    },
  },
  created() {
    this.loadXML = debounce(this.loadXML.bind(this), 0, { leading: false });

    /* Initialize the BpmnModdle and its extensions */
    window.ProcessMaker.EventBus.$emit('modeler-init', {
      registerInspectorExtension: this.registerInspectorExtension,
      registerBpmnExtension: this.registerBpmnExtension,
      registerNode: this.registerNode,
    });

    this.moddle = new BpmnModdle(this.extensions);
  },
  mounted() {
    this.graph = new joint.dia.Graph();
    store.commit('setGraph', this.graph);
    this.graph.set('interactiveFunc', cellView => {
      return {
        elementMove: cellView.model.get('elementMove'),
      };
    });
    this.paper = new joint.dia.Paper({
      el: this.$refs.paper,
      model: this.graph,
      gridSize: 10,
      drawGrid: true,
      clickThreshold: 10,
      perpendicularLinks: true,
      interactive: this.graph.get('interactiveFunc'),
      highlighting: {
        default: { options: { padding: highlightPadding } },
      },
    });

    this.handleResize();
    window.addEventListener('resize', this.handleResize);

    this.paper.on('blank:pointerclick', () => {
      store.commit('highlightNode', this.processNode);
    });

    this.paper.on('blank:pointerdown', (event, x, y) => {
      this.canvasDragPosition = { x, y };
    });
    this.paper.on('cell:pointerup blank:pointerup', () => {
      this.canvasDragPosition = null;
    });

    this.$el.addEventListener('mousemove', event => {
      if (this.canvasDragPosition) {
        this.paper.translate(
          event.offsetX - this.canvasDragPosition.x,
          event.offsetY - this.canvasDragPosition.y
        );
      }
    });

    this.paper.on('cell:pointerclick', (cellView, evt, x, y) => {
      const clickHandler = cellView.model.get('onClick');
      if (clickHandler) {
        clickHandler(cellView, evt, x, y);
      }
    });

    this.paper.on('cell:pointerdown', cellView => {
      if (cellView.model.component) {
        cellView.model.toFront({ deep: true });
        this.graph
          .getConnectedLinks(cellView.model)
          .forEach(link => link.toFront());

        /* If the element belongs to a pool, bring the pool to the front as well */
        if (cellView.model.component.node.pool) {
          const poolShape = cellView.model.component.node.pool;

          poolShape.toFront({ deep: true });
          poolShape.getEmbeddedCells()
            .filter(cell => {
              return cell.component && cell.component.node.type !== laneId;
            })
            .forEach(cell => cell.toFront());
        }

        if (cellView.model.component.node.type === poolId) {
          cellView.model.getEmbeddedCells()
            .filter(cell => {
              return cell.component && cell.component.node.type !== laneId;
            })
            .forEach(cell => cell.toFront());
        }

        cellView.model.component.$emit('click');
      }
    });

    /* Register custom nodes */
    window.ProcessMaker.EventBus.$emit('modeler-start', {
      loadXML: this.loadXML,
    });
  },
};
</script>

<style lang="scss">
@import '~jointjs/dist/joint.css';

$cursors: default, not-allowed;

.modeler {
  position: relative;
  width: inherit;
  max-width: inherit;
  height: inherit;
  max-height: inherit;
  overflow: hidden;

  .modeler-container {
    max-width: 100%;
    width: 100%;
    display: flex;
    flex-direction: row;

    .paper-container {
      height: 100%;
      max-height: 100%;
      min-height: 100%;
      overflow: hidden;
      position: relative;

      .history-buttons {
        position: absolute;
        z-index: 1;
        left: 1rem;
        top: 1rem;

        > button {
          cursor: pointer;
        }
      }
    }

    @each $cursor in $cursors {
      .paper-container.#{$cursor} {
        .joint-paper,
        .joint-paper * {
          cursor: #{$cursor} !important;
        }
      }
    }
  }
}
</style>
