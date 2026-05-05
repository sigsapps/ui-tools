<template>
  <div>
    <!-- Loading -->
    <div class="text-center q-pa-xl text-grey-8" v-show="showLoader">
      <div><q-spinner-gears size="lg" /></div>
      <div class="text-caption">Carregando...</div>
    </div>

    <!-- Error -->
    <div class="text-center q-pa-xl text-red-3" v-if="!!error && !showLoader">
      <div><q-icon size="lg" name="fas fa-bomb" /></div>
      <div class="text-h6">ERRO!</div>
      <div class="text-caption">
        <b>{{ error?.response?.status }}</b> {{ error?.response?.statusText }}
      </div>
    </div>

    <!-- Empty -->
    <div class="text-center q-pa-xl" v-show="!showLoader && !error && (data?.length === 0)">
      <div><q-icon size="lg" name="far fa-folder-open" /></div>
      <div class="text-h6">Sem Dados</div>
      <div class="text-caption">Nenhuma informação disponível para alimentar o gráfico</div>
    </div>

    <!-- Content -->
    <div v-show="!showLoader && !error && data && data.length > 0">
      <div class="text-center" :style="{ width: Size, height: height }">
        <canvas ref="canvas"></canvas>
      </div>
    </div>
  </div>
</template>

<script>
// Libs:
import Chart from 'chart.js' // v2.9.4

export default {
  name: 'ui-charts-pie',

  props: {
    ShowLegend: { type: Boolean, default: false },
    ShowPercentage: { type: Boolean, default: false },
    BeforeLoad: Function,
    OnLoaded: Function,
    Size: { type: String, default: () => '100%' },
    ChartType: { type: String, default: () => 'pie' }, // 'pie' | 'doughnut'
    DataURL: { type: String, required: true },
    Filters: { type: Object, default: () => ({}) },
    /**
     * Configs: { LabelField: 'name', ValueField: 'value', hideLegend?: boolean }
     */
    Configs: {
      type: Object,
      required: true,
      validator: (v) => !!v.LabelField && !!v.ValueField
    }
  },

  data() {
    return {
      chartElement: null,
      data: [],
      loading: false,
      showLoader: false,
      error: null,
      state: 'ready', // ready | loading | error
      debounce: null
    }
  },

  computed: {
    height() {
      const wrapper = this.$refs.canvas?.parentElement
      if (!wrapper) return '250px' // fallback enquanto não renderizou

      // obtém a largura real do wrapper (em px)
      const width = wrapper.offsetWidth || 300

      // calcula a altura proporcional ao tamanho da largura
      // (ajuste o fator conforme o formato desejado)
      const aspectRatio = 1 // 1 = quadrado; 0.75 = 4:3; 0.5 = metade da largura
      const heightPx = width * aspectRatio

      return heightPx + 'px'
    }
  },

  watch: {
    loading(v) { this.showLoader = v },

    Filters: {
      deep: true,
      handler() { this.debouncedLoad() }
    },

    data: {
      deep: true,
      handler() { this.updateChart() }
    },

    // Tenta alternar entre 'pie' e 'doughnut' sem recriar
    ChartType() {
      if (this.chartElement) {
        try {
          this.chartElement.config.type = this.ChartType
          this.chartElement.update(0)
        } catch (_) {
          // se falhar, mantenha o tipo atual (evitamos recriar)
        }
      }
    }
  },

  methods: {
    debouncedLoad() {
      this.loading = true
      if (this.debounce) clearTimeout(this.debounce)
      this.debounce = setTimeout(this.loadData, 400)
    },

    async loadData() {
      this.error = null
      this.state = 'loading'
      try {
        if (this.BeforeLoad) await this.BeforeLoad(this.Filters)

        const resp = await this.$http.get(this.DataURL, this.Filters)
        this.data = Array.isArray(resp.data) ? resp.data : []

        if (this.OnLoaded) await this.OnLoaded(resp)
        this.state = 'ready'
      } catch (err) {
        this.$utils?.notifyError?.(err)
        console.error('Error fetching chart data', err)
        this.error = err
        this.$emit('error-thrown', err)
        this.state = 'error'
      } finally {
        this.debounce = null
        this.loading = false
      }
    },

    ensureChartInitialized() {
      if (this.chartElement) return
      const canvas = this.$refs.canvas
      if (!canvas) return
      const ctx = canvas.getContext('2d')

      this.chartElement = new Chart(ctx, {
        type: this.ChartType,
        data: {
          labels: [],
          datasets: [{
            label: '',
            data: [],
            backgroundColor: [],
            hoverOffset: 4
          }]
        },
        options: this.buildBaseOptions()
      })
    },

    buildBaseOptions() {
      const self = this
      return {
        responsive: true,
        maintainAspectRatio: false,
        legend: { display: this.ShowLegend },
        tooltips: {
          callbacks: {
            label: function (tooltipItem, data) {
              const ds = data.datasets[tooltipItem.datasetIndex]
              const value = Number(ds.data[tooltipItem.index]) || 0
              const total = ds.data.reduce((a, b) => a + Number(b || 0), 0)
              const pct = total > 0 ? ((value / total) * 100).toFixed(2) + '%' : '0%'
              const labelText = data.labels[tooltipItem.index] || ''
              // Se ShowPercentage estiver ativo e o rótulo já tiver %, não duplica:
              if (self.ShowPercentage) {
                return `${labelText}: ${value} (${pct})`
              }
              return `${labelText}: ${value}`
            }
          }
        },
        animation: { duration: 400 },
        responsiveAnimationDuration: 200
      }
    },

    // cor estável por rótulo (evita “piscar” a cada update)
    colorForLabel(label) {
      // hash simples -> hsl
      let h = 0
      for (let i = 0; i < String(label).length; i++) {
        h = (h * 31 + String(label).charCodeAt(i)) >>> 0
      }
      const hue = h % 360
      const sat = 65
      const light = 55
      return `hsl(${hue}, ${sat}%, ${light}%)`
    },

    updateChart() {
      this.ensureChartInitialized()
      const ch = this.chartElement
      if (!ch) return

      if (!this.data || this.data.length === 0) {
        ch.data.labels = []
        ch.data.datasets[0].data = []
        ch.data.datasets[0].backgroundColor = []
        ch.update(0)
        return
      }

      const labelKey = this.Configs.LabelField
      const valueKey = this.Configs.ValueField

      const total = this.data.reduce((acc, row) => acc + Number(row?.[valueKey] || 0), 0)

      const labels = []
      const values = []
      const colors = []

      for (let i = 0; i < this.data.length; i++) {
        const row = this.data[i]
        const label = row?.[labelKey]
        const value = Number(row?.[valueKey] || 0)

        labels.push(label)
        values.push(value)
        colors.push(this.colorForLabel(label))
      }

      // substitui arrays por cópias planas (evita objetos reativos)
      ch.data.labels = labels.slice()
      ch.data.datasets[0].data = values.slice()
      ch.data.datasets[0].backgroundColor = colors.slice()

      ch.update()
    }
  },

  async mounted() {
    this.ensureChartInitialized()
    await this.loadData()
  },

  beforeUnmount() {
    // destrói com segurança, sem recriar nada
    if (!this.chartElement) return
    try {
      if (typeof this.chartElement.stop === 'function') this.chartElement.stop()
      if (Chart.animationService && typeof Chart.animationService.removeChart === 'function') {
        Chart.animationService.removeChart(this.chartElement)
      }
      if (this.chartElement.$resize && typeof this.chartElement.$resize.unbind === 'function') {
        this.chartElement.$resize.unbind()
      }
      this.chartElement.destroy()
    } catch (_) {
      // noop
    } finally {
      this.chartElement = null
    }
  }
}
</script>