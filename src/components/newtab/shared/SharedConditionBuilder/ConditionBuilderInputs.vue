<template>
  <div
    v-for="(item, index) in inputsData"
    :key="item.id"
    class="condition-input scroll"
  >
    <div
      v-if="item.category === 'value'"
      class="flex items-end space-x-2 flex-wrap"
    >
      <ui-select
        :model-value="item.type"
        class="flex-shrink-0"
        @change="updateValueType($event, index)"
      >
        <optgroup
          v-for="(types, label) in filterValueTypes(index)"
          :key="label"
          :label="label"
        >
          <option v-for="type in types" :key="type.id" :value="type.id">
            {{ type.name }}
          </option>
        </optgroup>
      </ui-select>
      <template v-for="(_, name) in item.data" :key="item.id + name">
        <v-remixicon
          v-if="name === 'code'"
          :title="t('workflow.conditionBuilder.topAwait')"
          name="riInformationLine"
        />
        <edit-autocomplete
          :disabled="name === 'code'"
          :class="[name === 'code' ? 'w-full' : 'flex-1']"
          :style="{ marginLeft: name === 'code' ? 0 : null }"
        >
          <shared-codemirror
            v-if="name === 'code'"
            v-model="inputsData[index].data[name]"
            class="code-condition mt-2"
            style="margin-left: 0"
          />
          <ui-input
            v-else
            v-model="inputsData[index].data[name]"
            :title="conditionBuilder.inputTypes[name].label"
            :placeholder="conditionBuilder.inputTypes[name].label"
            autocomplete="off"
            class="w-full"
          />
        </edit-autocomplete>
      </template>
    </div>
    <ui-select
      v-else-if="item.category === 'compare'"
      :model-value="inputsData[index].type"
      @change="updateCompareType($event, index)"
    >
      <optgroup
        v-for="(types, category) in conditionOperators"
        :key="category"
        :label="category"
      >
        <option v-for="type in types" :key="type.id" :value="type.id">
          {{ type.name }}
        </option>
      </optgroup>
    </ui-select>
  </div>
</template>
<script setup>
import { ref, watch, defineAsyncComponent } from 'vue';
import { nanoid } from 'nanoid';
import { useI18n } from 'vue-i18n';
import cloneDeep from 'lodash.clonedeep';
import { conditionBuilder } from '@/utils/shared';
import EditAutocomplete from '../../workflow/edit/EditAutocomplete.vue';

const SharedCodemirror = defineAsyncComponent(() =>
  import('../SharedCodemirror.vue')
);

const props = defineProps({
  data: {
    type: Array,
    default: () => [],
  },
  autocomplete: {
    type: Array,
    default: () => [],
  },
});
const emit = defineEmits(['update']);

const conditionOperators = conditionBuilder.compareTypes.reduce((acc, type) => {
  if (!acc[type.category]) acc[type.category] = [];

  acc[type.category].push(type);

  return acc;
}, {});

const { t } = useI18n();
const inputsData = ref(cloneDeep(props.data));

function getDefaultValues(items) {
  const defaultValues = {
    value: {
      id: nanoid(),
      type: 'value',
      category: 'value',
      data: { value: '' },
    },
    compare: { id: nanoid(), category: 'compare', type: 'eq' },
  };

  if (typeof items === 'string') return defaultValues[items];

  return items.map((item) => defaultValues[item]);
}
function filterValueTypes(index) {
  const exclude = ['element#visible', 'element#invisible'];

  return conditionBuilder.valueTypes.reduce((acc, item) => {
    if (index < 1 || !exclude.includes(item.id)) {
      (acc[item.category] = acc[item.category] || []).push(item);
    }

    return acc;
  }, {});
}
function updateValueType(newType, index) {
  const type = conditionBuilder.valueTypes.find(({ id }) => id === newType);

  if (index === 0 && !type.compareable) {
    inputsData.value.splice(index + 1);
  } else if (inputsData.value.length === 1) {
    inputsData.value.push(...getDefaultValues(['compare', 'value']));
  }

  inputsData.value[index].type = newType;
  inputsData.value[index].data = { ...type.data };
}
function updateCompareType(newType, index) {
  const { needValue } = conditionBuilder.compareTypes.find(
    ({ id }) => id === newType
  );

  if (!needValue) {
    inputsData.value.splice(index + 1);
  } else if (inputsData.value.length === 2) {
    inputsData.value.push(getDefaultValues('value'));
  }

  inputsData.value[index].type = newType;
}

watch(
  inputsData,
  (value) => {
    emit('update', value);
  },
  { deep: true }
);
</script>
<style>
.code-condition .cm-content {
  white-space: pre-wrap;
}
</style>
