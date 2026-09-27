<script setup lang="ts">
import { computed, ref, watch } from "vue";
import { operatorRegistry } from "lib";
import { ParsedSignature } from "lib/HelperClasses/ParsedSignature";
import LogicProgrammerVisualOutput from "../../components/LogicProgrammerVisualOutput.vue";
import StepDisplayPanels from "../../components/StepDisplayPanels.vue";
import Tile from "../../components/Tile.vue";
import TileGrid from "../../components/TileGrid.vue";
import { FULL_TILE_SPAN, assignOperatorRows } from "pages-lib/tileLayout";
import type { VisualStep } from "pages-lib/visualTransformerLogic";

const props = defineProps<{
  operatorKey: string;
}>();

type OperatorPageInstance = {
  getParsedSignature(): ParsedSignature;
};

type OperatorPageClass = {
  new (normalizeSignature?: boolean): OperatorPageInstance;
  internalName: string;
  nicknames: string[];
  interactName: string;
  displayName?: string;
  fullDisplayName?: string;
  tooltipInfo?: string;
};

const operatorClass = computed(() => {
  return operatorRegistry[
    props.operatorKey as keyof typeof operatorRegistry
  ] as unknown as OperatorPageClass;
});

const SIGNATURE_ARROW = "\u2192";

type OperatorSignatureSlot = {
  label: string;
  text: string;
};

type OperatorSignatureDisplay = {
  main: string;
  slots: OperatorSignatureSlot[];
};

const signatureLetter = (index: number): string => {
  let remaining = index;
  let label = "";
  do {
    label = String.fromCharCode(65 + (remaining % 26)) + label;
    remaining = Math.floor(remaining / 26) - 1;
  } while (remaining >= 0);
  return label;
};

const buildOperatorSignature = (
  root: TypeRawSignatureAST.RawSignatureNode
): OperatorSignatureDisplay => {
  const operatorNumbers = new Map<
    TypeRawSignatureAST.RawSignatureNode,
    number
  >();
  const anyLetters = new Map<number, string>();
  const slots: {
    label: number;
    obscured: TypeRawSignatureAST.RawSignatureFunction;
  }[] = [];

  const anyLetter = (typeID: number): string => {
    if (!anyLetters.has(typeID)) {
      anyLetters.set(typeID, signatureLetter(anyLetters.size));
    }
    return anyLetters.get(typeID)!;
  };

  const assign = (node: TypeRawSignatureAST.RawSignatureNode): void => {
    switch (node.type) {
      case "Function":
        assign(node.from);
        assign(node.to);
        return;
      case "Operator": {
        const label = operatorNumbers.size + 1;
        operatorNumbers.set(node, label);
        slots.push({ label, obscured: node.obscured });
        assign(node.obscured);
        return;
      }
      case "List":
        assign(node.listType);
        return;
      case "Any":
        anyLetter(node.typeID);
        return;
      default:
        return;
    }
  };

  const render = (
    node: TypeRawSignatureAST.RawSignatureNode,
    isReturnPosition = false
  ): string => {
    switch (node.type) {
      case "Function": {
        const body = `${render(node.from)} ${SIGNATURE_ARROW} ${render(
          node.to,
          true
        )}`;
        return isReturnPosition ? `(${body})` : body;
      }
      case "Operator":
        return `Operator<${operatorNumbers.get(node) ?? "?"}>`;
      case "List":
        return `List<${render(node.listType)}>`;
      case "Any":
        return `Any<${anyLetter(node.typeID)}>`;
      default:
        return node.type;
    }
  };

  assign(root);

  return {
    main: render(root),
    slots: slots.map(({ label, obscured }) => ({
      label: `${label}`,
      text: render(obscured),
    })),
  };
};

const operatorSignature = computed<OperatorSignatureDisplay | null>(() => {
  try {
    const operator = new operatorClass.value(false);
    return buildOperatorSignature(operator.getParsedSignature().getAst());
  } catch {
    return null;
  }
});

const variableId = ref(0);
const variableName = ref("");

watch(
  operatorClass,
  (nextOperator) => {
    variableId.value = 0;
    variableName.value = nextOperator.interactName;
  },
  { immediate: true }
);

watch(variableId, (nextValue) => {
  if (!Number.isFinite(nextValue)) {
    variableId.value = 0;
    return;
  }

  const normalized = Math.max(0, Math.trunc(nextValue));
  if (normalized !== nextValue) {
    variableId.value = normalized;
  }
});

const operatorAst = computed<TypeAST.Operator>(() => {
  const name = variableName.value.trim();
  return {
    type: "Operator",
    opName: props.operatorKey as TypeOperatorKey,
    ...(name ? { varName: name } : {}),
  };
});

const operatorTabRef = ref<any>(null);
const patternTabRef = ref<any>(null);

const operatorTabSteps = computed<VisualStep[]>(
  () => (operatorTabRef.value?.steps ?? []) as VisualStep[]
);
const patternTabSteps = computed<VisualStep[]>(
  () => (patternTabRef.value?.steps ?? []) as VisualStep[]
);

const operatorRows = (cols: number) =>
  assignOperatorRows({
    panelsShareRow: cols >= 2,
    operatorDisplayFitsRow4: cols >= 3,
  });
</script>

<template>
  <article class="doc-page operator-doc-page">
    <TileGrid v-slot="{ cols, resetLayout }">
      <Tile id="title" :span="FULL_TILE_SPAN" :default-col="0" :default-row="0">
        <div class="tile-title-block">
          <div class="tile-title-text">
            <h2>{{ operatorKey }}</h2>
          </div>
          <div class="tile-actions">
            <button type="button" class="tile-reset" @click="resetLayout">
              Reset layout
            </button>
          </div>
        </div>
      </Tile>

      <Tile id="info" :span="FULL_TILE_SPAN" :default-col="0" :default-row="1">
        <dl class="operator-meta">
          <div class="operator-meta-card">
            <dt>Internal name</dt>
            <dd>{{ operatorClass.internalName }}</dd>

            <dt>Nicknames</dt>
            <dd v-if="operatorClass.nicknames.length">
              {{ operatorClass.nicknames.join(", ") }}
            </dd>
            <dd v-else>None</dd>

            <template v-if="operatorClass.tooltipInfo">
              <dt>Description</dt>
              <dd>{{ operatorClass.tooltipInfo }}</dd>
            </template>

            <template v-if="operatorSignature">
              <dt>Signature</dt>
              <dd class="operator-signature">
                <div class="operator-signature-main">
                  {{ operatorSignature.main }}
                </div>
                <div
                  v-for="slot in operatorSignature.slots"
                  :key="slot.label"
                  class="operator-signature-slot"
                >
                  {{ slot.label }}: {{ slot.text }}
                </div>
              </dd>
            </template>
          </div>
        </dl>
      </Tile>

      <Tile id="variableId" :span="1" :default-col="0" :default-row="2">
        <label class="field">
          <span>Variable ID</span>
          <input
            v-model.number="variableId"
            class="select"
            type="number"
            min="0"
            step="1"
            aria-label="Variable ID"
          />
        </label>
      </Tile>

      <Tile id="variableName" :span="1" :default-col="1" :default-row="2">
        <label class="field">
          <span>Variable name</span>
          <input
            v-model="variableName"
            class="select"
            type="text"
            aria-label="Variable name"
          />
        </label>
      </Tile>

      <Tile
        id="operatorTab"
        :span="1"
        :default-col="0"
        :default-row="operatorRows(cols).operatorTab - 1"
      >
        <h3>Operator Tab</h3>
        <LogicProgrammerVisualOutput
          ref="operatorTabRef"
          :ast="operatorAst"
          :start-variable-id="variableId"
          :show-step-numbers="false"
          :show-step-titles="false"
          :show-display-panels="false"
          operator-preview-mode="pattern"
        />
      </Tile>

      <Tile
        id="patternTab"
        :span="1"
        :default-col="1"
        :default-row="operatorRows(cols).patternTab - 1"
      >
        <h3>Pattern Tab</h3>
        <LogicProgrammerVisualOutput
          ref="patternTabRef"
          :ast="operatorAst"
          :start-variable-id="variableId"
          :show-step-numbers="false"
          :show-step-titles="false"
          :show-display-panels="false"
        />
      </Tile>

      <Tile
        id="operatorDisplay"
        :span="1"
        :default-col="0"
        :default-row="operatorRows(cols).operatorDisplay - 1"
      >
        <StepDisplayPanels :steps="operatorTabSteps" />
      </Tile>

      <Tile
        id="patternDisplay"
        :span="1"
        :default-col="1"
        :default-row="operatorRows(cols).patternDisplay - 1"
      >
        <StepDisplayPanels :steps="patternTabSteps" />
      </Tile>
    </TileGrid>
  </article>
</template>

<style scoped>
.operator-signature {
  display: grid;
  gap: 0.2rem;
  font-family: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
  font-size: 0.9rem;
}

.operator-signature-main {
  overflow-wrap: anywhere;
}

.operator-signature-slot {
  color: #4a6974;
}
</style>
