<template>
  <q-input type="text" square filled hide-bottom-space :label="Label" :clearable="clearable" :dense="dense"
    :disable="disable" :readonly="readonly" :mask="mask" :hint="hint"
    :class="`full-width bg-${BgColor ? BgColor : 'white'}`" v-model="value" :error="hasError" :error-message="ErrorMsg"
    @focus="() => $emit('focus')">
    <template v-slot:append>
      <slot name="buttons"></slot>
      <q-icon :name="Icon ?? 'fas fa-calendar-alt'" color="grey-8" />
    </template>
  </q-input>
</template>

<script>
export default {
  name: 'ui-formparts-inputdate',

  props: {
    modelValue: String,
    readonly: Boolean,
    disable: Boolean,
    dense: Boolean,
    clearable: Boolean,
    withTime: Boolean,
    withSeconds: Boolean,
    Icon: String,
    BrazilianFormat: Boolean,
    Default: String,
    Label: String,
    Error: Boolean,
    ErrorMsg: String,
    BgColor: String,
    hint: String,
  },

  data() {
    return {
      value: null,
      state: 'pending',
      error: false,
      errormsg: '',
    }
  },

  computed: {
    hasError() {
      return this.state === 'error' || this.error || this.Error;
    },

    mask() {
      return `${this.withTime ? `##/##/#### ##:##${this.withSeconds ? ':##' : ''}` : '##/##/####'}`
    }
  },

  watch: {
    modelValue(val) {
      if (val && val.length >= 10) {
        let date = val.split(' ')[0];
        let time = val.split(' ')[1] ?? null;

        // If typing mode is enabled, we set the modelValue to the current date, but reverting from 'yyyy-mm-dd' to 'dd/mm/yyyy':
        let dateParts = date.split('-');
        this.value = `${dateParts[2]}/${dateParts[1]}/${dateParts[0]}`;

        if (this.withTime) {
          this.value += ` ${time ?? '00:00:00'}`;
        }
      } else {
        // Reset the typing mode options when closed
        this.value = null;
      }
    },

    value(val) {
      this.error = false;
      this.errormsg = '';
      this.state = 'pending';

      if (!val) return this.clear();
      else if (val.length < 10) return;

      this.emit(val);
    }
  },

  methods: {
    clear() {
      this.value = this.Default || null;
      this.error = false;
      this.errormsg = '';
      this.emit(this.value);
    },

    emit(val) {
      let emitValue = null;

      if (!!val) {
        const [datePart, timePart] = val.split(' ');
        const [day, month, year] = datePart.split('/').map(Number);

        if (!(day > 0 && day <= 31 && month > 0 && month <= 12 && year > 0)) {
          this.error = true;
          this.errormsg = 'Data inválida';
          this.state = 'error';
          return;
        }

        if (!!timePart) {
          const timeParts = timePart.split(':');
          if (timeParts.length < 2 || timeParts.length > 3) {
            this.error = true;
            this.errormsg = 'Hora inválida';
            this.state = 'error';
            return;
          }
          const [hours, minutes, seconds] = timeParts.map(Number);
          if (!(hours >= 0 && hours < 24 && minutes >= 0 && minutes < 60 && (seconds === undefined || (seconds >= 0 && seconds < 60)))) {
            this.error = true;
            this.errormsg = 'Hora inválida';
            this.state = 'error';
            return;
          }
        }

        this.state = 'success';

        emitValue = `${year}-${month.toString().padStart(2, '0')}-${day.toString().padStart(2, '0')}`;
        if (this.withTime) emitValue += ` ${timePart ?? '00:00:00'}`;
      }

      this.$emit('update:modelValue', emitValue);
      this.$emit('update:state', this.state);
    }
  },

  mounted() {
    if (this.Default) {
      this.value = this.Default;
    }
  }
}
</script>