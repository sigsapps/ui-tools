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
        :ReadAsURL="true"
        :accept="accept ? accept : 'image/*'"
        @activateFn="(fn) => activateFileInput = fn"
        @update:model-value="onFilePicked">
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
        :ReadAsURL="true"
        :accept="accept ? accept : 'image/*'"
        @activateFn="(fn) => activateFileInput = fn"
        @update:model-value="onFilePicked">
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

    <!-- Dialogo de Recorte -->
    <Modal v-model="showCropDialog" Title="Ajustar Imagem" Icon="fas fa-crop-simple" Persistent Width="520px"
      :Actions="cropModalActions" @hide="cancelCrop">
      <div class="cropper-viewport">
        <img ref="cropperImg" :src="rawSrc" style="max-width:100%; display:block;" @load="initCropper">
      </div>
      <div class="text-caption text-grey-7 text-center q-mt-sm">
        Arraste a imagem para posicionar e use os botões (ou a roda do mouse) para ajustar o zoom.
      </div>
      <div class="row justify-center q-gutter-sm q-mt-sm">
        <q-btn dense round outline color="grey-8" icon="fas fa-magnifying-glass-plus" @click="zoom(0.1)">
          <q-tooltip>Aumentar zoom</q-tooltip>
        </q-btn>
        <q-btn dense round outline color="grey-8" icon="fas fa-magnifying-glass-minus" @click="zoom(-0.1)">
          <q-tooltip>Diminuir zoom</q-tooltip>
        </q-btn>
        <q-btn dense round outline color="grey-8" icon="fas fa-rotate-left" @click="rotate(-90)">
          <q-tooltip>Girar à esquerda</q-tooltip>
        </q-btn>
        <q-btn dense round outline color="grey-8" icon="fas fa-rotate-right" @click="rotate(90)">
          <q-tooltip>Girar à direita</q-tooltip>
        </q-btn>
      </div>
    </Modal>
  </div>
</template>

<script>
import Cropper from 'cropperjs';
import 'cropperjs/dist/cropper.css';

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
      },
      // Recorte:
      showCropDialog: false,
      rawSrc: null,
      rawFile: null,
      cropper: null,
      cropModalActions: [
        { label: 'Aplicar', color: 'positive', icon: 'fas fa-check', fn: () => this.confirmCrop() }
      ],
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
    onFilePicked(v) {
      if (!v || !v.file) return;

      this.rawFile = v.file;
      this.rawSrc = v.src;
      this.showCropDialog = true;
    },

    initCropper() {
      if (this.cropper) { this.cropper.destroy(); }

      this.cropper = new Cropper(this.$refs.cropperImg, {
        aspectRatio: this.setRatio,
        viewMode: 1,
        dragMode: 'move',
        cropBoxMovable: false,
        cropBoxResizable: false,
        autoCropArea: 1,
        background: false,
        responsive: true,
        zoomOnWheel: true,
      });
    },

    zoom(ratio) {
      this.cropper?.zoom(ratio);
    },

    rotate(deg) {
      this.cropper?.rotate(deg);
    },

    cancelCrop() {
      if (this.cropper) {
        this.cropper.destroy();
        this.cropper = null;
      }
      this.rawSrc = null;
      this.rawFile = null;
      this.showCropDialog = false;
    },

    confirmCrop() {
      if (!this.cropper) return;

      const rawFile = this.rawFile;
      const mimeType = rawFile.type || 'image/png';
      const canvas = this.cropper.getCroppedCanvas({ imageSmoothingQuality: 'high' });

      canvas.toBlob((blob) => {
        const croppedFile = new File([blob], rawFile.name, { type: blob.type || mimeType });

        const result = {
          file: croppedFile,
          name: rawFile.name,
          src: canvas.toDataURL(mimeType),
          size: croppedFile.size,
        };

        for (let k in this.input) { this.input[k] = result[k]; }
        this.$emit('update:model-value', result);

        this.cropper.destroy();
        this.cropper = null;
        this.rawSrc = null;
        this.rawFile = null;
        this.showCropDialog = false;
      }, mimeType);
    },
  }

}
</script>

<style scoped>
#wrapper {
  position: relative;
  width: 100%;
}

.cropper-viewport {
  max-height: 60vh;
  overflow: hidden;
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