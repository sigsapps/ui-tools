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
        <b>{{ error.response && error.response.status }}</b>
        {{ error.response && error.response.statusText }}
      </div>
    </div>

    <!-- Empty -->
    <div class="text-center q-pa-xl" v-show="!showLoader && !error && data && data.length === 0">
      <div><q-icon size="lg" name="far fa-folder-open" /></div>
      <div class="text-h6">Sem Dados</div>
      <div class="text-caption">Nenhuma informação disponível para alimentar o gráfico</div>
    </div>

    <!-- Content -->
    <div v-show="!showLoader && !error && data && data.length > 0" class="chart-scroll">
      <div :style="{ minHeight: '100%', minWidth: '100%', width: dynamicWidth, height: dynamicHeight }">
        <canvas ref="canvas"></canvas>
      </div>
    </div>
  </div>
</template>

<script>
// Libs:
import Chart from 'chart.js' // v2.9.4

export default {
  name: 'ui-charts-linear',

  props: {
    BeforeLoad: Function,
    OnLoaded: Function,
    Height: { type: String, default: () => '100%' },
    Width: { type: String, default: () => '100%' },
    ChartType: { type: String, default: () => 'line' }, // 'line' | 'bar' | etc
    DataURL: { type: String, required: true },
    Filters: { type: Object, default: () => ({}) },

    /**
     * Datasets spec:
     * [
     *   { field: 'y1', label: 'Serie A', formatTooltip?: (label, value) => string }
     * ]
     */
    Datasets: {
      type: Array,
      required: true,
      validator: (v) => v.every(d => 'field' in d)
    },

    /**
     * Configs: { xAxisKey: 'name', hideLegend?: boolean }
     */
    Configs: {
      type: Object,
      required: true,
      validator: (v) => !!v.xAxisKey
    },

    /**
     * Control animation on updates (default true)
     */
    UseAnimation: { type: Boolean, default: () => true }
  },

  data() {
    return {
      vTypes: ['column'],
      hTypes: ['line', 'bar', 'area'],
      chartElement: null,
      data: [],
      loading: false,
      showLoader: false,
      error: null,
      state: 'ready', // ready | loading | error
      debounce: null,

      // Internal mapping to control datasets by "field"
      dsIndexByField: Object.create(null)
    }
  },

  computed: {
    orientation() {
      if (this.vTypes.includes(this.ChartType)) return 'vertical'
      if (this.hTypes.includes(this.ChartType)) return 'horizontal'
      throw new Error('Invalid chart type')
    },

    dynamicWidth() {
      if (this.orientation === 'vertical') return this.Width
      const w = Math.max(250, this.data.length * 25)
      return `${Math.ceil(w)}px`
    },

    dynamicHeight() {
      if (this.orientation === 'horizontal') return this.Height
      const w = Math.max(250, this.data.length * 25)
      return `${Math.ceil(w)}px`
    }
  },

  watch: {
    loading(v) { this.showLoader = v },

    Filters: {
      deep: true,
      handler() { this.debouncedLoad() }
    },

    // Se os dados mudarem (via loadData), atualizamos o gráfico
    data: {
      deep: true,
      handler() { this.updateChart() }
    },

    // Se Datasets (estrutura) mudar, reconciliamos com o gráfico já criado
    Datasets: {
      deep: true,
      handler() { this.reconcileDatasetsStructureAndUpdate() }
    },

    // Se tipo de gráfico mudar, é mais seguro recriar a instância
    // MAS o pedido foi "update ao invés de recreate".
    // Aqui, faremos um fallback que tenta migrar o type no runtime (Chart.js 2.x suporta).
    ChartType() {
      if (this.chartElement) {
        try {
          this.chartElement.config.type = this.ChartType
          this.chartElement.update(0)
        } catch (e) {
          // Se der ruim, recriar seria a saída,
          // mas mantendo a exigência do update, deixamos o tipo intacto.
          // (Você pode optar por emitir um warning)
          // console.warn('Could not switch chart type at runtime.', e)
        }
      }
    }
  },

  methods: {
    debouncedLoad() {
      this.loading = true
      if (this.debounce) clearTimeout(this.debounce)
      this.debounce = setTimeout(this.loadData, 500)
    },

    async loadData() {
      this.error = null
      this.state = 'loading'
      try {
        if (this.BeforeLoad) await this.BeforeLoad(this.Filters)

        const response = await this.$http.get(this.DataURL, this.Filters)
        this.data = Array.isArray(response.data) ? response.data : []
        if (this.OnLoaded) await this.OnLoaded(response)
        this.state = 'ready'
      } catch (error) {
        this.$utils?.notifyError?.(error)
        console.error('An error has occurred on the attempt to retrieve chart data.', error)
        this.error = error
        this.$emit('error-thrown', error)
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

      const baseOptions = this.buildBaseOptions()

      this.chartElement = new Chart(ctx, {
        type: this.ChartType,
        data: { labels: [], datasets: [] },
        options: baseOptions
      })

      // Inicializa índice por field
      this.dsIndexByField = Object.create(null)
      this.reconcileDatasetsStructureAndUpdate(true)
    },

    buildBaseOptions() {
      // Tooltips com callback por dataset (se houver)
      const tooltipCallbacks = {
        label: (tooltipItem, data) => {
          const index = tooltipItem.datasetIndex
          const ds = this.Datasets[index]
          if (ds && typeof ds.formatTooltip === 'function') {
            return ds.formatTooltip(
              data.datasets[index].label,
              tooltipItem.yLabel
            )
          }
          return data.datasets[index].label + ': ' + tooltipItem.yLabel
        }
      }

      return {
        responsive: true,
        maintainAspectRatio: false,
        legend: { display: !this.Configs.hideLegend },
        tooltips: { mode: 'label', callbacks: tooltipCallbacks },
        animation: this.UseAnimation ? { duration: 400 } : { duration: 0 },
        responsiveAnimationDuration: this.UseAnimation ? 200 : 0,
        scales: {
          xAxes: [{
            ticks: {
              autoSkip: false,
              callback: function (tick) {
                const characterLimit = 12
                if ((tick?.length ?? 0) >= characterLimit) {
                  tick = tick.slice(0, tick.length).substring(0, characterLimit - 1).trim() + '...'
                }
                return tick?.toLowerCase()
              }
            }
          }],
          yAxes: [{ ticks: { beginAtZero: true } }]
        }
      }
    },

    makePlainDatasetFromSpec(spec) {
      // Gere uma cor determinística ou aleatória
      const color = this.$utils?.randomHexColor
        ? this.$utils.randomHexColor()
        : '#3489db'

      // Retorne um objeto PLANO (não reativo)
      return {
        label: spec.label,
        data: [],              // preenchido depois
        fill: false,
        backgroundColor: color,
        borderColor: color,
        borderWidth: 3,
        // não inclua o campo 'field' aqui para não poluir o objeto interno do Chart.js
      }
    },

    reconcileDatasetsStructureAndUpdate(firstInit = false) {
      // Garante que chart exista
      this.ensureChartInitialized()
      if (!this.chartElement) return

      const chart = this.chartElement
      const current = chart.data.datasets
      const wanted = this.Datasets.map(ds => ds.field)

      // 1) Remover datasets que não existem mais
      for (let i = current.length - 1; i >= 0; i--) {
        const fieldAtI = Object.keys(this.dsIndexByField).find(f => this.dsIndexByField[f] === i)
        if (!fieldAtI || !wanted.includes(fieldAtI)) {
          current.splice(i, 1)
          this.rebuildDsIndexMapAfterRemoval(i)
        }
      }

      // 2) Adicionar datasets que faltam (na ordem de this.Datasets)
      this.dsIndexByField = Object.create(null)
      for (let i = 0; i < this.Datasets.length; i++) {
        const spec = this.Datasets[i]
        if (current[i]) {
          // Reusa a posição i e apenas garante label, cores (opcional)
          current[i].label = spec.label
        } else {
          current.push(this.makePlainDatasetFromSpec(spec))
        }
        this.dsIndexByField[spec.field] = i
      }

      // 3) Se foi chamado por um “primeiro init”, apenas faça um update leve
      if (firstInit) {
        chart.update(0)
      } else {
        // Atualize com a base de dados atual
        this.updateChart()
      }
    },

    rebuildDsIndexMapAfterRemoval(removedIndex) {
      // Ajusta o mapa após remoção de dataset em 'removedIndex'
      const newMap = Object.create(null)
      for (const field in this.dsIndexByField) {
        const idx = this.dsIndexByField[field]
        if (idx < removedIndex) newMap[field] = idx
        else if (idx > removedIndex) newMap[field] = idx - 1
        // se idx === removedIndex, some (dataset removido)
      }
      this.dsIndexByField = newMap
    },

    updateChart() {
      // Se ainda não há gráfico, inicializa
      this.ensureChartInitialized()
      const chart = this.chartElement
      if (!chart) return

      // Caso não haja dados, limpe o gráfico
      if (!this.data || this.data.length === 0) {
        chart.data.labels = []
        chart.data.datasets.forEach(d => { d.data = [] })
        chart.update(0)
        return
      }

      // Monta labels e valores por field
      const labels = []
      const seriesByField = Object.create(null)
      for (const { field } of this.Datasets) {
        seriesByField[field] = []
      }

      const xKey = this.Configs.xAxisKey

      for (let i = 0; i < this.data.length; i++) {
        const row = this.data[i]
        labels.push(row[xKey])
        for (const field in seriesByField) {
          seriesByField[field].push(row[field])
        }
      }

      // Aplica em chart.data (substituição completa, sem objetos reativos)
      chart.data.labels = labels.slice()

      // Garante estrutura (ordem) sincronizada
      for (let i = 0; i < this.Datasets.length; i++) {
        const { field, label } = this.Datasets[i]
        const idx = this.dsIndexByField[field]
        if (idx == null) continue // em teoria não deve ocorrer, mas seja defensivo
        const ds = chart.data.datasets[idx]
        ds.label = label
        ds.data = (seriesByField[field] || []).slice()
      }

      // Atualiza com/sem animação
      chart.update(this.UseAnimation ? undefined : 0)
    }
  },

  mounted() {
    // Inicializa e carrega
    this.ensureChartInitialized()
    this.loadData()
  },

  beforeUnmount() {
    if (this.debounce) clearTimeout(this.debounce)
    // Destrói com segurança
    if (this.chartElement) {
      try {
        if (typeof this.chartElement.stop === 'function') this.chartElement.stop()
        if (Chart.animationService && typeof Chart.animationService.removeChart === 'function') {
          Chart.animationService.removeChart(this.chartElement)
        }
        // Remover listeners de resize (defensivo em 2.x)
        if (this.chartElement.$resize && typeof this.chartElement.$resize.unbind === 'function') {
          this.chartElement.$resize.unbind()
        }
        this.chartElement.destroy()
      } catch (e) {
        // noop
      } finally {
        this.chartElement = null
      }
    }
  }
}
</script>

<style scoped>
.chart-scroll {
  width: 100%;
  overflow-x: auto;
  overflow-y: auto;
}

canvas {
  display: block;
  width: 100% !important;
  height: 100% !important;
}
</style>
