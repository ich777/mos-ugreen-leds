<template>
  <v-menu :close-on-content-click="false" location="bottom start">
    <template #activator="{ props: menuProps }">
      <v-text-field
        v-bind="menuProps"
        :model-value="modelValue"
        :label="label"
        :disabled="disabled"
        readonly
        density="comfortable"
        hide-details
      >
        <template #prepend-inner>
          <div
            :style="{
              width: '20px',
              height: '20px',
              borderRadius: '4px',
              background: modelValue,
              border: '1px solid rgba(128, 128, 128, 0.5)',
            }"
          />
        </template>
      </v-text-field>
    </template>
    <v-color-picker
      :model-value="modelValue"
      :modes="['rgb', 'hex']"
      mode="hex"
      show-swatches
      :swatches="swatches"
      @update:model-value="onUpdate"
    />
  </v-menu>
</template>

<script setup>
defineProps({
  modelValue: { type: String, default: '#ffffff' },
  label: { type: String, default: '' },
  disabled: { type: Boolean, default: false },
});

const emit = defineEmits(['update:modelValue']);

const swatches = [
  ['#ffffff', '#ff0000'],
  ['#ffa500', '#ffff00'],
  ['#00ff00', '#00ffff'],
  ['#0000ff', '#ff00ff'],
];

const onUpdate = (value) => {
  if (typeof value !== 'string') return;
  emit('update:modelValue', value.slice(0, 7).toLowerCase());
};
</script>
