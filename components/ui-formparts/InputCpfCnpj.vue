<template>
  <q-input type="text" square filled hide-bottom-space :label="Label" :clearable="false" :dense="dense" :hint="hint"
    :disable="disable" :readonly="readonly" maxlength="18" :mask="mask"
    :class="`full-width bg-${BgColor ? BgColor : 'white'}`" v-model="value" :error="Error" :error-message="ErrorMsg"
    @focus="() => $emit('focus')">
    <template v-slot:append>
      <slot name="buttons"></slot>
      <q-icon :name="Icon ?? 'fas fa-id-card'" color="grey-8" />
    </template>
  </q-input>
</template>

<script>
export default {
  name: 'ui-formparts-input-cpf-cnpj',

  props: {
    Icon: String,
    modelValue: String,
    readonly: Boolean,
    disable: Boolean,
    dense: Boolean,
    clearable: Boolean,
    Default: String,
    Label: String,
    Error: Boolean,
    ErrorMsg: String,
    CpfOnly: Boolean,
    CnpjOnly: Boolean,
    BgColor: String,
    hint: String,
  },

  data() {
    return {
      value: null,
    }
  },

  computed: {
    mask() {
      let mask = '###.###.###-###';

      if (!!this.CpfOnly) mask = '###.###.###-##';
      if (!!this.CnpjOnly) mask = '##.###.###/####-##';

      if (!!this.value) {
        const onlyDigits = this.value.replace(/\D+/g, '');
        if (onlyDigits.length > 11) mask = '##.###.###/####-##';
      }

      return mask;
    }
  },

  watch: {
    modelValue(val) {
      this.value = val ?? null;
    },

    value(val) {
      if (!val) return this.clear();
      else if (val.length < 14) return;

      this.emit();
    }
  },

  methods: {
    clear() {
      this.value = this.Default || null;
      this.emit();
    },

    emit() {
      this.$emit('update:modelValue', this.value);
    }
  },

  mounted() {
    if (this.Default) {
      this.value = this.Default;
    }
  }
}
</script>