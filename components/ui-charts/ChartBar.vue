<template>
  <ChartLinear ChartType="bar" :BeforeLoad="BeforeLoad" :OnLoaded="OnLoaded" :Height="Height" :DataURL="DataURL" :Filters="Filters"
    :Datasets="Datasets" :Configs="Configs">
  </ChartLinear>
</template>

<script>
export default {
  name: 'ui-charts-bar',

  props: {
    BeforeLoad: Function,
    OnLoaded: Function,
    Height: String,
    DataURL: String,
    Filters: Object,
    Datasets: Array,
    Configs: Object,
  },
}
</script>
