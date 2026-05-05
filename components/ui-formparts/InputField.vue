<template>
  <div :class="ignorePadding ? '' : (dense ? 'q-pa-xs' : 'q-pa-sm')">
    <!-- Input text: -->
    <q-input v-if="type == 'text' || type == null" type="text" square filled hide-bottom-space :ref="inputRefId"
      :label="Label" :clearable="clearable" :dense="dense" :disable="disable" :readonly="readonly" :unmasked-value="unmaskedValue"
      :maxlength="maxlength" :mask="Mask" :reverse-fill-mask="ReverseFillMask" :fill-mask="FillMask" :hint="hint"
      :class="`full-width bg-${BgColor ? BgColor : 'white'}`" v-model="value" :error="Error" :error-message="ErrorMsg"
      @focus="() => $emit('focus')" @update:model-value="updModelValue" @click="inputClicked">
      <template v-slot:append>
        <slot name="buttons"></slot>
        <q-icon v-if="!!Icon" :name="Icon" color="grey-8" />
      </template>
    </q-input>

    <!-- Input password: -->
    <q-input v-if="type == 'password'" :type="isPassword ? 'password' : 'text'" square filled hide-bottom-space :ref="inputRefId" :label="Label"
      :clearable="clearable" :dense="dense" :disable="disable" :readonly="readonly" :maxlength="maxlength" :hint="hint"
      :class="`full-width bg-${BgColor ? BgColor : 'white'}`" v-model="value" :error="Error"
      @focus="() => $emit('focus')" @update:model-value="updModelValue" @click="inputClicked" :error-message="ErrorMsg">
      <template v-slot:append>
        <slot name="buttons"></slot>
        <q-icon
          :name="isPassword ? 'visibility_off' : 'visibility'"
          class="cursor-pointer"
          @click="isPassword = !isPassword"
        />
        <q-icon v-if="!!Icon" :name="Icon" color="grey-8" />
      </template>
    </q-input>

    <!-- Input textarea: -->
    <q-input v-if="type == 'textarea'" type="textarea" square filled hide-bottom-space :ref="inputRefId" :label="Label"
      :clearable="clearable" :dense="dense" :disable="disable" :readonly="readonly" :maxlength="maxlength" :hint="hint"
      :class="`full-width bg-${BgColor ? BgColor : 'white'}`" v-model="value" :error="Error"
      @focus="() => $emit('focus')" @update:model-value="updModelValue" @click="inputClicked" :error-message="ErrorMsg">
      <template v-slot:append>
        <slot name="buttons"></slot>
        <q-icon v-if="!!Icon" :name="Icon" color="grey-8" />
      </template>
    </q-input>

    <!-- Input email: -->
    <q-input v-if="type == 'email'" type="email" square filled hide-bottom-space maxlength="255" :ref="inputRefId"
      :label="Label" :clearable="clearable" :dense="dense" :disable="disable" :readonly="readonly" :hint="hint"
      :class="`full-width bg-${BgColor ? BgColor : 'white'}`" v-model="value" :error="Error"
      @focus="() => $emit('focus')" @update:model-value="updModelValue" @click="inputClicked" :error-message="ErrorMsg">
      <template v-slot:append>
        <slot name="buttons"></slot>
        <q-icon v-if="!!Icon" :name="Icon" color="grey-8" />
      </template>
    </q-input>

    <!-- Input number: -->
    <q-input v-if="type == 'number'" :ref="inputRefId" square filled hide-bottom-space :step="step" :max="max"
      :min="min" :label="Label" :clearable="clearable" :dense="dense" :disable="disable" :readonly="readonly" :hint="hint"
      :class="`full-width bg-${BgColor ? BgColor : 'white'}`" v-model="value" :error="Error"
      @focus="() => $emit('focus')" type="number"
      @update:model-value="(v) => { value = value === '' ? null : value; updModelValue(value) }" @click="inputClicked"
      :error-message="ErrorMsg">
      <template v-slot:append>
        <slot name="buttons"></slot>
        <q-icon v-if="!!Icon" :name="Icon" color="grey-8" />
      </template>
    </q-input>

    <!-- Input Checkbox -->
    <q-checkbox v-if="type == 'checkbox'" :ref="inputRefId" :label="Label" :disable="disable" :readonly="readonly"
      v-model="value" :error="Error" @focus="() => $emit('focus')" @update:model-value="updModelValue">
    </q-checkbox>

    <!-- Input select: -->
    <Select2 v-if="type == 'select'" :BgColor="BgColor" :clearable="clearable" :dense="dense" :disable="disable"
      :readonly="readonly" v-model="value" :Options="Options" :OptionsURL="OptionsURL" :OptionMap="OptionMap" :Label="Label" :Icon="Icon" :Error="Error"
      :Multiple="Multiple" :UseChips="UseChips" :StackLabel="StackLabel" :Default="Default" :SortOptions="SortOptions"
      @focus="() => $emit('focus')" @update:model-value="updModelValue">
    </Select2>

    <!-- Input phone: -->
    <InputPhone :Icon="Icon" v-if="type == 'phone'" :dense="dense" :clearable="clearable" :disable="disable" :hint="hint"
      :readonly="readonly" v-model="value" :Default="Default" :Label="Label" :Error="Error"
      @focus="() => $emit('focus')" @update:model-value="updModelValue">
    </InputPhone>

    <!-- Input CPF/CNPJ: -->
    <InputCpfCnpj :Icon="Icon" v-if="type == 'cpf' || type == 'cnpj' || type == 'cpf+cnpj'" :CpfOnly="type == 'cpf'" :hint="hint"
      :CnpjOnly="type == 'cnpj'" :dense="dense" :clearable="clearable" :disable="disable" :readonly="readonly"
      v-model="value" :Default="Default" :Label="Label" :Error="Error" @focus="() => $emit('focus')"
      @update:model-value="updModelValue">
    </InputCpfCnpj>

    <!-- Input date: -->
    <InputDate v-if="type == 'date'" :dense="dense" :clearable="clearable" :disable="disable" :readonly="readonly" :hint="hint"
      v-model="value" :Default="Default" :Label="Label" :Error="Error" @focus="() => $emit('focus')"
      @update:model-value="updModelValue">
    </InputDate>

    <!-- Input datetime: Está com erro, não digita horário -->
    <InputDate v-if="type == 'datetime'" withTime :withSeconds="withSeconds" :dense="dense" :clearable="clearable" :hint="hint"
      :disable="disable" :readonly="readonly" v-model="value" :Default="Default" :Label="Label" :Error="Error"
      @focus="() => $emit('focus')" @update:model-value="updModelValue">
    </InputDate>

    <!-- Input date picker: -->
    <DatePicker v-if="type == 'datepicker'" :BgColor="BgColor" :dense="dense" :disable="disable" :readonly="readonly" :hint="hint"
      :todayBtn="todayBtn" :dateOptions="dateOptions" :minDatePage="minDatePage" :maxDatePage="maxDatePage"
      v-model="value" :Default="Default" :Label="Label" :Error="Error" @focus="() => $emit('focus')"
      @update:model-value="updModelValue">
    </DatePicker>

    <!-- Input daterange picker: -->
    <DatePicker v-if="type == 'daterangepicker'" :BgColor="BgColor" :dense="dense" :disable="disable"
      :readonly="readonly" :todayBtn="todayBtn" :dateOptions="dateOptions" :minDatePage="minDatePage" :hint="hint"
      :maxDatePage="maxDatePage" v-model="value" range :Default="Default" :Label="Label" :Error="Error"
      @focus="() => $emit('focus')" @update:model-value="updModelValue">
    </DatePicker>

    <!-- Input datetime picker: -->
    <DatePicker v-if="type == 'datetimepicker'" :withSeconds="withSeconds" :BgColor="BgColor" :dense="dense"
      :disable="disable" :todayBtn="todayBtn" :dateOptions="dateOptions" :minDatePage="minDatePage" :hint="hint"
      :maxDatePage="maxDatePage" :readonly="readonly" v-model="value" withTime :Default="Default" :Label="Label"
      :Error="Error" @focus="() => $emit('focus')" @update:model-value="updModelValue">
    </DatePicker>

    <!-- Input datetimerange picker: -->
    <DatePicker v-if="type == 'datetimerange'" :withSeconds="withSeconds" :BgColor="BgColor" :dense="dense"
      :todayBtn="todayBtn" :dateOptions="dateOptions" :minDatePage="minDatePage" :maxDatePage="maxDatePage" :hint="hint"
      :disable="disable" :readonly="readonly" v-model="value" range withTime :Default="Default" :Label="Label"
      :Error="Error" @focus="() => $emit('focus')" @update:model-value="updModelValue">
    </DatePicker>

    <!-- Input time: -->
    <TimePicker v-if="type == 'time'" :withSeconds="withSeconds" :BgColor="BgColor" :dense="dense" :disable="disable" :hint="hint"
      :readonly="readonly" v-model="value" :Default="Default" :Label="Label" :Error="Error" square
      @update:model-value="updModelValue" @focus="() => $emit('focus')">
    </TimePicker>

    <!-- Input color: -->
    <InputColor v-if="type == 'color'" :BgColor="BgColor" :clearable="clearable" :dense="dense" :disable="disable" :hint="hint"
      :readonly="readonly" v-model="value" :Default="Default" :Label="Label" :Error="Error"
      @focus="() => $emit('focus')" @update:model-value="updModelValue">
    </InputColor>

    <!-- Input file: -->
    <FileUpload v-if="type == 'file'" :accept="accept" :clearable="clearable" :disable="disable" :readonly=readonly :hint="hint"
      v-model="value" :ReadAsURL="ReadAsURL" :Label="Label" :Icon="Icon" :Error="Error" @focus="() => $emit('focus')"
      @update:model-value="updModelValue" @fileupload-before-choose="broadcast('fileupload-before-choose')"
      @fileupload-chosen="broadcast('fileupload-chosen')">
    </FileUpload>

    <!-- Input Editor -->
    <InputEditor v-if="type == 'editor'" :disable="disable" :readonly=readonly v-model="value" :Label="Label"
      :Error="Error" :placeholder="placeholder" @focus="() => $emit('focus')" @update:model-value="updModelValue">
    </InputEditor>
  </div>

</template>

<script>
export default {
  name: 'ui-formparts-inputfield',

  props: {
    // General
    modelValue: [String, Object, Number],
    BgColor: String,
    Label: String,
    Icon: String,
    Error: { type: Boolean, default: false },
    ErrorMsg: String,
    type: String,
    clearable: Boolean,
    dense: Boolean,
    disable: Boolean,
    maxlength: String,
    readonly: Boolean,
    // Date
    dateOptions: { type: [Array, Function] },
    todayBtn: Boolean,
    minDatePage: String,
    maxDatePage: String,
    // Time
    withSeconds: Boolean,
    // Select
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
    Multiple: Boolean,
    UseChips: Boolean,
    StackLabel: Boolean,
    // Number
    step: String,
    max: String,
    min: String,
    // Mask
    Mask: String,
    ReverseFillMask: Boolean,
    FillMask: {
      type: [String, Boolean],
      default: false,
    },
    SelectOnClick: {
      type: Boolean,
      default: false,
    },
    // Others
    Default: {
      type: [String, Object, Number, Boolean],
      default: () => null
    },
    accept: String,
    ReadAsURL: Boolean,
    placeholder: String,
    ignorePadding: {
      type: Boolean,
      default: false
    },
    hint: String,
    unmaskedValue: Boolean,
  },

  data() {
    return {
      value: null,
      inputRefId: null,
      isPassword: true,
    }
  },

  watch: {
    modelValue(v) {
      this.value = v;
    }
  },

  methods: {
    inputClicked() {
      if (this.SelectOnClick) {
        const nativeInput = this.$refs[this.inputRefId].$el.querySelector('input');
        if (!!nativeInput) nativeInput.select();
      }
    },

    updModelValue(v) {
      this.$emit('update:model-value', v);
    },

    broadcast(eventName, data) {
      this.$emit(eventName, data);
    }
  },

  created() {
    this.inputRefId = `InputFieldRef-${this.$utils.uniqid()}`
    this.value = this.modelValue;
  },

  mounted() {
    this.$emit('expose-ref', this.$refs[this.inputRefId]);
    if(this.Default !== null && typeof this.Default != 'undefined'){
      this.updModelValue(this.Default);
    }
  }
}
</script>