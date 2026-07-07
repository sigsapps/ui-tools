<template>
  <q-input hide-bottom-space :class="`full-width bg-${BgColor ? BgColor : 'white'}`" :dense="dense" readonly filled :hint="hint"
    square v-model="formattedDate" :error="Error" :label="Label">
    <template v-slot:append>
      <q-icon id="clear-button" v-if="!!formattedDate && !readonly" name="cancel" clickable @click="clear()"
        class="cursor-pointer">
        <q-tooltip>Limpar seleção</q-tooltip>
      </q-icon>
      <q-icon v-if="readonly" name="fas fa-calendar-alt" color="grey-8" class="cursor-ban"></q-icon>
      <q-icon v-else clickable @click="$emit('focus');" name="fas fa-calendar-alt" color="primary"
        class="cursor-pointer">
        <q-tooltip>Selecionar {{ range ? 'Período' : 'Data' }}</q-tooltip>
        <q-popup-proxy cover transition-show="scale" transition-hide="scale" style="margin: 0;"
          @hide="finalizePendingSingleDay">
          <q-card>
            <q-date :emit-immediately="!range" :range="range" :today-btn="todayBtn" :options="dateOptions"
              :navigation-min-year-month="minDatePage" :navigation-max-year-month="maxDatePage" v-model="date"
              @update:model-value="updateModelValue()" @range-start="onRangeStart" @range-end="onRangeEnd">
            </q-date>
            <div v-if="withTime" class="q-pa-sm">
              <TimePicker :withSeconds="withSeconds" dense :Label="range ? 'De:' : 'Hora'" v-model="firstTime"
                :Default="defaultFirstTime" @update:model-value="updateModelValue()">
              </TimePicker>
              <TimePicker :withSeconds="withSeconds" v-if="range" dense Label="Até:" v-model="lastTime"
                :Default="defaultLastTime" class="q-pt-sm" @update:model-value="updateModelValue()">
              </TimePicker>
            </div>
            <div class="row items-center justify-end q-pa-sm">
              <q-btn v-close-popup class="q-mr-sm" label="Digitar" color="primary" dense @click="typingMode = true" />
              <q-btn v-close-popup label="Fechar" color="primary" flat dense />
            </div>
          </q-card>
        </q-popup-proxy>
      </q-icon>
    </template>
  </q-input>
</template>

<script>
export default {
  name: 'ui-formparts-datepicker',

  props: {
    modelValue: { type: [String, Object] },
    Default: { type: [String, Object] },
    BgColor: String,
    Label: String,
    Error: { type: Boolean, default: false },
    dateOptions: { type: [Array, Function] },
    dense: Boolean,
    range: Boolean,
    readonly: Boolean,
    withTime: Boolean,
    withSeconds: Boolean,
    todayBtn: Boolean,
    minDatePage: String,
    maxDatePage: String,
    hint: String,
  },

  data() {
    return {
      defaultFirstTime: null,
      defaultLastTime: null,
      firstTime: null,
      lastTime: null,
      date: null,
      // Guards against a reactive feedback loop: applying an external value below sets date/firstTime/
      // lastTime, which would otherwise re-trigger updateModelValue() and re-emit a (new but equivalent)
      // object, which would come back around as a "changed" modelValue and re-apply forever.
      isApplyingExternalValue: false,
      // In range mode, q-date needs a second click on a day to complete the selection - a
      // single click alone never emits anything and leaves the field blank. This tracks that
      // first click so we can finalize it as a one-day range if the popup closes before a
      // second click happens.
      pendingRangeStart: null,
    }
  },

  watch: {
    date() {
      if (this.isApplyingExternalValue) return;
      this.updateModelValue();
    },

    firstTime() {
      if (this.isApplyingExternalValue) return;
      this.updateModelValue();
    },

    lastTime() {
      if (this.isApplyingExternalValue) return;
      this.updateModelValue();
    },

    // Keeps the calendar popup in sync whenever the bound value changes - including when it
    // arrives asynchronously (e.g. an edit screen loading the entity's data after this component
    // has already mounted). Without this, the popup's internal state stays stuck on whatever it
    // was initialized with, and can even overwrite a value the user just picked.
    modelValue: {
      immediate: true,
      handler(v) {
        this.applyValue(v ?? this.Default ?? null);
      }
    },
  },

  methods: {
    setDefault() {
      this.applyValue(this.Default ?? null);
    },

    applyValue(v) {
      // An external value takes precedence over any in-progress range click.
      this.pendingRangeStart = null;

      // q-date works with a 'YYYY/MM/DD' mask internally, while values coming from the model
      // (loaded entities, or this component's own emitted values) use 'YYYY-MM-DD'.
      const toSlash = (d) => (d ? d.replaceAll('-', '/') : d);

      var date;
      var firstTime;
      var lastTime;

      if (!!v) {
        // For date range:
        if (typeof v == 'object') {
          date = {
            from: null,
            to: null
          };
          firstTime = null;
          lastTime = null;

          if (!!v.from) {
            if (v.from.includes(':')) {
              let arr = v.from.split(' ');
              date.from = toSlash(arr[0]);
              firstTime = arr[1];
            } else {
              date.from = toSlash(v.from);
            }
          }

          if (!!v.to) {
            if (v.to.includes(':')) {
              let arr = v.to.split(' ');
              date.to = toSlash(arr[0]);
              lastTime = arr[1];
            } else {
              date.to = toSlash(v.to);
            }
          }
        }

        // For single date:
        else {
          if (v.includes(':')) {
            let arr = v.split(' ');
            date = toSlash(arr[0]);
            firstTime = arr[1];
          } else {
            date = toSlash(v);
          }
        }
      } else {
        date = null;
        firstTime = null;
        lastTime = null;
      }

      this.isApplyingExternalValue = true;
      this.date = date;
      this.firstTime = firstTime;
      this.lastTime = lastTime;
      this.$nextTick(() => { this.isApplyingExternalValue = false; });

      if (!!this.Default && firstTime) {
        this.defaultFirstTime = firstTime;
      }
      if (!!this.Default && lastTime) {
        this.defaultLastTime = lastTime;
      }
    },

    updateModelValue() {
      var val;

      // For date range:
      if (this.range && this.date != null) {
        const normalize = (d) => (d ? d.replaceAll('/', '-') : null);

        let from = null; 
        let to = null;
        
        if(typeof this.date === 'object') {
          from = normalize(this.date.from);
          to   = normalize(this.date.to);
        } else {
          const normalized = normalize(this.date);
          from = normalized;
          to   = normalized;
        }

        if (from && this.firstTime) from = `${from} ${this.firstTime}`;
        if (to && this.lastTime) to = `${to} ${this.lastTime}`;

        val = { from, to };
      }

      // Legacy code:
      // if (this.date != null && (typeof this.date == 'object' || this.range)) {
      //   // 1. Handle date:
      //   val = {
      //     from: !!this.date.from ? this.date.from.replaceAll('/', '-') : null,
      //     to: !!this.date.to ? this.date.to.replaceAll('/', '-') : null,
      //   };
      //   // 2. Handle first time:
      //   if (!!val.from && this.firstTime) val.from = `${val.from} ${this.firstTime}`
      //   // 3. Handle last time:
      //   if (!!val.to && this.lastTime) val.to = `${val.to} ${this.lastTime}`
      // }

      // For single date:
      else {
        // 1. Handle date:
        val = !!this.date ? this.date.replaceAll('/', '-') : null;
        // 2. Handle first time:
        if (!!val && this.firstTime) val = `${val} ${this.firstTime}`
      }
      this.$emit('update:model-value', val);
    },

    clear() {
      this.setDefault();
      this.$emit('update:model-value', this.Default);
      if (this.withTime) {
        this.defaultFirstTime = this.firstTime ? this.firstTime : '00:00:00';
        if (this.range)
          this.defaultLastTime = this.lastTime ? this.lastTime : '23:59:59';

      }
    },

    onRangeStart(day) {
      const pad = (n) => String(n).padStart(2, '0');
      this.pendingRangeStart = `${day.year}/${pad(day.month)}/${pad(day.day)}`;
    },

    onRangeEnd() {
      // A second click completed the range normally - nothing left to finalize.
      this.pendingRangeStart = null;
    },

    finalizePendingSingleDay() {
      if (!this.range || !this.pendingRangeStart) return;

      this.date = { from: this.pendingRangeStart, to: this.pendingRangeStart };
      this.pendingRangeStart = null;
    }
  },

  computed: {
    formattedDate() {
      if (!!this.modelValue) {
        if (typeof this.modelValue == 'object') {
          if (!this.modelValue?.from || !this.modelValue?.to) return null;

          let fromData = this.modelValue.from.split('-');
          let toData = this.modelValue.to.split('-');

          var dayFrom = fromData[2];
          var dayTo = toData[2];
          if (this.modelValue.from.includes(':')) {
            let dayData = dayFrom.split(' ');
            dayFrom = dayData[0];
            fromData[0] = `${fromData[0]} ${dayData[1]}`;
          }
          if (this.modelValue.to.includes(':')) {
            let dayData = dayTo.split(' ');
            dayTo = dayData[0];
            toData[0] = `${toData[0]} ${dayData[1]}`;
          }

          return `De: ${dayFrom}/${fromData[1]}/${fromData[0]}  ->  Até: ${dayTo}/${toData[1]}/${toData[0]}`
        } else {
          var dateData = this.modelValue.split('-');

          var day = dateData[2];
          if (this.modelValue.includes(':')) {
            let dayData = day.split(' ');
            day = dayData[0];
            dateData[0] = `${dateData[0]} ${dayData[1]}`;
          }
          return `${day}/${dateData[1]}/${dateData[0]}`
        }
      }
      else return null;
    },
  },

}
</script>
<style scoped>
.cursor-ban {
  cursor: not-allowed
}

#clear-button {
  opacity: 0.7;
}

#clear-button:hover {
  opacity: 1;
}
</style>