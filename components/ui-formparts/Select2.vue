<template>
  <div>
    <!-- No errors -->
    <q-select v-if="error === null" filled square hide-bottom-space hide-dropdown-icon use-input fill-input
      :loading="loading" behavior="menu" :bg-color="`bg-${BgColor ? BgColor : 'white'}`" option-disable="inactive"
      input-debounce="300" :clearable="clearable" :dense="dense" :disable="disable" :label="Label" :readonly="readonly"
      :hide-selected="!Multiple" :options="options" :error="Error" :multiple="Multiple" :use-chips="UseChips"
      :stack-label="StackLabel" v-model="selected" @filter="filterFn" @focus="$emit('focus')"
      @popup-show="isOpen = true" @popup-hide="isOpen = false">
      <template v-slot:append>
        <q-icon v-if="!readonly" :class="isOpen ? 'rotate-180' : ''" name="arrow_drop_down" color="grey-8"
          style="transition: transform 0.28s" />
        <q-icon v-if="!!Icon" :name="Icon" color="grey-8" />
      </template>
      <template v-slot:no-option>
        <q-item>
          <q-item-section class="text-grey">Nenhum resultado</q-item-section>
        </q-item>
      </template>
    </q-select>
    <label v-else
      class="q-field row no-wrap items-start q-field--filled q-input q-field--square q-field--float q-field--labeled full-width bg-red-1"
      for="f_0f2b1c56-251c-42ac-9d8d-f01255b7ef19"><!---->
      <div class="q-field__inner relative-position col self-stretch">
        <div class="q-field__control relative-position row no-wrap" tabindex="-1">
          <div class="q-field__control-container col relative-position row no-wrap q-anchor--skip"><input
              class="q-field__native q-placeholder text-negative text-bold" tabindex="0" :aria-label="Label"
              id="f_0f2b1c56-251c-42ac-9d8d-f01255b7ef19" maxlength="100" type="text" disabled
              :value="`ERRO: ${error}`">
            <div class="q-field__label no-pointer-events absolute ellipsis">{{ Label }}</div><!---->
          </div>
          <div class="q-field__append q-field__marginal row no-wrap items-center"><i
              class="q-icon text-negative fas fa-circle-exclamation" aria-hidden="true" role="presentation"> </i></div>
        </div><!---->
      </div><!---->
    </label>
  </div>

</template>

<script>
export default {
  name: 'ui-formparts-select2',

  props: {
    modelValue: {
      type: [String, Number, Array, Object],
      default: null
    },
    Options: {
      type: Array,
      default: () => []
    },
    SortOptions: {
      type: Boolean,
      default: true
    },
    OptionsURL: String,
    OptionMap: Object,

    BgColor: String,
    Label: String,
    Icon: String,
    Error: Boolean,

    readonly: Boolean,
    disable: Boolean,
    dense: Boolean,
    clearable: Boolean,

    Multiple: Boolean,
    UseChips: Boolean,
    StackLabel: Boolean,

    Default: [String, Object, Number],
  },

  data() {
    return {
      filteredOptions: [],
      selected: null,
      isOpen: false,
      internalUpdate: false, // Evita loop infinito entre: selected -> emitValue -> modelValue -> setValue -> selected
      error: null,
      errorDict: {
        400: '400 - Requisição inválida.',
        401: '401 - Não autenticado.',
        403: '403 - Permissão negada.',
        404: '404 - Não Encontrado',
        500: '500 - Erro interno do servidor. Tente novamente mais tarde.',
        502: '502 - Bad Gateway.',
        503: '503 - Serviço temporariamente indisponível.',
        504: '504 - Gateway Timeout.'
      },
      remoteOptions: [],
      loading: true,
    }
  },

  watch: {
    selected(v) {
      v = v ?? this.Default ?? null;
      this.emitValue(v)
    },

    modelValue(v) {
      if (this.internalUpdate) {
        this.internalUpdate = false
        return
      }

      this.setValue(v);
    },

    Options(v) {
      const opts = v.length ? [...v] : (this.remoteOptions ?? []);

      this.filteredOptions = this.SortOptions
        ? opts.sort((a, b) => String(a.label).localeCompare(String(b.label), 'pt-BR', { sensitivity: 'base' }))
        : opts

      this.error = null; // Limpa erros anteriores ao atualizar opções
      this.setValue(this.modelValue)
    },

    OptionsURL() {
      this.getRemoteOptions()
    }
  },

  computed: {
    options() {
      return this.filteredOptions
    },

    definedOpts() {
      const options = this.Options.length ? this.Options : (this.remoteOptions ?? [])

      return this.SortOptions
        ? options.sort((a, b) => String(a.label).localeCompare(String(b.label), 'pt-BR', { sensitivity: 'base' }))
        : options
    }
  },

  methods: {
    setValue(v) {
      if (this.loading && this.Options.length === 0) {
        setTimeout(() => this.setValue(v), 300);
        return;
      }

      const source = this.definedOpts
      if (!source?.length) return

      if (!v) {
        this.selected = this.Multiple ? [] : null
        return
      }

      this.selected = this.Multiple
        ? source.filter(opt => (Array.isArray(v) ? v : []).includes(opt.value))
        : source.find(opt =>
          String(opt.value).toLowerCase() === String(v).toLowerCase()
        ) || null
    },

    filterFn(val, update) {
      update(() => {
        const opts = this.definedOpts

        if (!(val)) {
          this.filteredOptions = opts
          return
        }

        const needle = val.toLowerCase()
        this.filteredOptions = opts
          .filter(opt =>
            String(opt.label).toLowerCase().includes(needle)
          )
      })
    },

    emitValue(v) {
      if (this.Multiple) {
        this.internalUpdate = true
        const values = Array.isArray(v)
          ? v.map(item => (item instanceof Object && 'value' in item) ? item.value : item)
          : []
        this.$emit('update:model-value', values)
        return
      }

      if (v instanceof Object && "value" in v) v = v.value;

      const payload = v ? v : null
      if (payload === this.modelValue) return

      this.internalUpdate = true
      this.$emit('update:model-value', payload)
    },

    async getRemoteOptions() {
      this.loading = true;

      if (!this.OptionsURL || !this.OptionMap) {
        this.loading = false;
        return;
      }

      try {
        const { data } = await this.$http.get(this.OptionsURL);

        this.remoteOptions = data.map(item => ({
          label: item[this.OptionMap.label],
          value: item[this.OptionMap.value],
        }));

        this.filteredOptions = this.definedOpts;
        this.error = null; // Limpa erros anteriores ao atualizar opções
      } catch (e) {
        this.error = this.errorDict[e.status];
        console.error('Erro ao buscar opções remotas:', e)
      } finally {
        this.loading = false;
      }
    }
  },

  mounted() {
    if (this.OptionsURL && !this.OptionMap) {
      console.warn('Select2: OptionsURL fornecida sem OptionMap. As opções remotas não serão carregadas.')
    }

    setTimeout(async () => {
      await this.getRemoteOptions();

      if (this.Default) {
        this.selected = this.Default;
      }
    }, 200);
  }
}
</script>