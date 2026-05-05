<template>
  <q-select behavior="menu" hide-bottom-space :bg-color="`bg-${BgColor ? BgColor : 'white'}`" option-disable="inactive"
    :disable="disable" hide-dropdown-icon :dense="dense" filled square :clearable="clearable"
    v-model="selected" use-input hide-selected fill-input input-debounce="300" :options="options" @filter="filterFn"
    :label="Label" :error="Error" @focus="$emit('focus')" :readonly="readonly"
    @popup-show="isOpen = true" @popup-hide="isOpen = false">
    <template v-slot:append>
      <q-icon v-if="!readonly" :class="isOpen? 'rotate-180':''" name="arrow_drop_down" color="grey-8" style="transition: transform 0.28s" />
      <q-icon v-if="!!Icon" :name="Icon" color="grey-8" />
    </template>
    <template v-slot:no-option>
      <q-item>
        <q-item-section class="text-grey">
          Nenhum resultado
        </q-item-section>
      </q-item>
    </template>
  </q-select>
</template>

<script>
export default {
  name: 'ui-formparts-select2',

  props: {
    EmitWholeOption: Boolean,
    BgColor: String,
    Options: Array,
    Label: String,
    Icon: String,
    Error: Boolean,
    readonly: Boolean,
    disable: Boolean,
    dense: Boolean,
    modelValue: {
      types: ['string', 'object', 'number']
    },
    clearable: Boolean
  },

  data() {
    return {
      rawData: [],
      selected: null,
      debounceId: null,
      isOpen: false,
    }
  },

  computed: {
    options() {
      if (!!this.rawData?.length) {
        return [...this.rawData].sort((a, b) =>
          String(a.label).localeCompare(String(b.label), 'pt-BR', { sensitivity: 'base' }))
      }
      return [];
    }
  },

  watch: {
    selected(v) {
      const emitVal = !!this.EmitWholeOption ? v : v?.value;
      this.$emit('update:model-value', emitVal);
    },

    modelValue(v) {
      this.setValue(v);
    },

    Options: {
      handler(v) {
        this.rawData = v;
      },

      deep: true
    }
  },

  methods: {
    setValue(v) {
      if(!!this.debounceId){
        clearTimeout(this.debounceId);
        this.debounceId = null;
      }

      if (!(this.rawData?.length))
        this.debounceId = setTimeout(() => this.setValue(v), 100);
      else {
        if (!(!!v)) {
          this.selected = null;
        } else {
          this.selected = this.rawData.find((opt) => {
            return String(opt.value).toLocaleLowerCase() == String(v).toLocaleLowerCase();
          });
        }
      }
    },

    filterFn(val, update) {
      if (!(!!val)) {
        update(() => {
          this.rawData = this.Options;
        })
        return
      }

      update(() => {
        const needle = val.toLowerCase()
        this.rawData = this.Options.filter(v => String(v.label).toLowerCase().includes(needle));
      })
    },
  },

  mounted(){
    this.rawData = this.Options;
  }
}
</script>