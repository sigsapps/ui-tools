<template>
  <div id="wrapper" :class="`q-pa-sm ${setAlign}`">
    <q-card v-if="cover" class="q-pa-sm">
      <q-img class="bg-grey-3" 
        :src="setSrc" 
        :ratio="setRatio" 
        :fit="setFit" 
        :height="coverHeight" 
        :width="coverWidth" />

      <FileUpload 
        class="hidden" 
        v-model="input" 
        :ReadAsURL="true"
        :accept="accept ? accept : 'image/*'"
        @activateFn="(fn) => activateFileInput = fn" 
        @update:model-value="updModelValue">
      </FileUpload>

      <q-btn 
        flat 
        round 
        class="bg-white" 
        id="btn-edit" 
        color="primary" 
        :disable="disable" 
        icon="fas fa-edit"
        @click="activateFileInput()" 
        :size="setIconSize">
        <q-tooltip v-if="!disable">Alterar imagem</q-tooltip>
      </q-btn>
    </q-card>

    <q-avatar v-else :square="square" :size="setSize" class="bg-grey-7">
      <q-img 
        :src="setSrc" 
        :ratio="setRatio" 
        :fit="setFit" />

      <FileUpload 
        class="hidden" 
        v-model="input" 
        :ReadAsURL="true"
        :accept="accept ? accept : 'image/*'"
        @activateFn="(fn) => activateFileInput = fn" 
        @update:model-value="updModelValue">
      </FileUpload>

      <q-btn 
        flat 
        round 
        id="btn-edit" 
        class="bg-white" 
        color="primary" 
        icon="fas fa-edit"
        :disable="disable" 
        :size="setIconSize"
        @click="activateFileInput()">
        <q-tooltip v-if="!disable">Alterar imagem</q-tooltip>
      </q-btn>
    </q-avatar>
  </div>
</template>

<script>
export default {
  name: 'ui-formparts-photopicker',

  props: {
    DefaultImgPath: String,
    modelValue: Object,
    disable: Boolean,
    square: Boolean,
    size: String,
    cover: Boolean,
    coverHeight: String,
    coverWidth: String,
    align: String,
    fit: String,
    accept: {
      type: String,
      validator: val => val === 'image/*' || val === 'image/png' || val === 'image/jpeg' || val === 'image/jpg'
    }
  },

  data() {
    return {
      activateFileInput: null,
      input: {
        file: null,
        name: null,
        src: null,
        size: null,
      }
    }
  },

  watch: {
    modelValue: {
      handler(v) {
        for (let k in this.input) {
          this.input[k] = v[k];
        }
      },
      deep: true
    }
  },

  computed: {
    setSrc() {
      return this.input.src ? this.input.src : this.DefaultImgPath;
    },

    setSize() {
      if (!!this.size) { return this.size + 'px'; }
      return '150px';
    },

    setIconSize() {
      if (!!this.size) {
        if (this.size <= 70) return this.size / 7 + 'px';
        if (this.size <= 100) return 'sm';
        if (this.size <= 300) return 'md';
        return 'lg';
      }
      return 'md';
    },

    setAlign() {
      if (!!this.align) {
        switch (this.align) {
          case 'left':
            return 'text-left';
          case 'right':
            return 'text-right';
          default:
            return 'text-center';
        }
      }
      return 'text-center';
    },

    setRatio() {
      return this.cover? 16/9 : 1;
    },

    setFit() {
      return this.fit? this.fit : 'cover';
    }
  },

  methods: {
    updModelValue(v) {
      this.$emit('update:model-value', v);
    },
  }

}
</script>

<style scoped>
#wrapper {
  position: relative;
  width: 100%;
}

.photo-container {
  width: 100%;
  border-radius: 50%;
  overflow: hidden;
}

.photo-container>img {
  width: 100%;
}

#btn-edit {
  position: absolute;
  right: 0px;
  bottom: 0px;
}

.square {
  border-radius: 0%;
}
</style>