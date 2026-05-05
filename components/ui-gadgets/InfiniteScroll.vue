<template>
  <div ref="scrollContainer" class="full-width" style="height:400px; overflow: auto;">
    <!-- Empty -->
    <div v-if="state == 'empty'" class="q-pa-lg text-center text-grey-8">
      <div>
        <q-icon size="lg" name="fas fa-folder-open"></q-icon> *
      </div>
      <div class="text-h6">
        Sem dados
      </div>
    </div>

    <!-- Error State -->
    <div v-if="state == 'error'" class="q-pa-lg text-center text-red-3">
      <div>
        <q-icon size="lg" name="fas fa-bomb"></q-icon>
      </div>
      <div class="text-h6">
        ERRO!
      </div>
      <div class="text-caption"><b>{{ errData.response?.status }}</b> {{ errData.response?.statusText }}</div>
      <small>Favor entrar em contato com o administrador do sistema.</small>
    </div>

    <!-- Ready -->
    <div v-if="state == 'ready'">
      <q-infinite-scroll :key="reloadCount" :disable="!hasMoreData" :scroll-target="$refs.scrollContainer"
        @load="loadData" :offset="250">
        <div class="q-pa-sm" v-for="(row, idx) in data" :key="idx">
          <slot :data="row"></slot>
        </div>

        <template v-slot:loading>
          <div class="row justify-center q-my-md">
            <q-spinner-dots color="primary" size="40px" />
          </div>
        </template>
      </q-infinite-scroll>
    </div>
  </div>
</template>

<script>
export default {
  name: 'ui-gadgets-infinite-scroll',

  props: {
    DataURL: {
      type: String,
      required: true
    },
    Limit: {
      type: Number,
      default: () => 10
    },
    Filters: {
      type: Object,
      default: () => { }
    },
    BeforeLoad: Function,
    AfterLoad: Function,
    modelValue: {},
  },

  data() {
    return {
      // Control:
      page: 1,
      hasMoreData: true,
      startLoad: false,
      isLoading: false,
      reloadCount: 0,

      // Data:
      data: [],
      errData: null
    }
  },

  computed: {
    params() {
      return {
        ...this.Filters,
        $page: this.page,
        $limit: this.Limit,
        $limit_multiplier: 1
      }
    },

    state() {
      if (!!this.errData) return 'error';

      const loadIsNotStarted = !this.startLoad;
      const hasNoData = this.data.length < 1;
      const noDataOrNoURL = !this.DataURL || hasNoData;

      if (loadIsNotStarted && noDataOrNoURL) return 'empty';

      return 'ready';
    },

    output() {
      return {
        data: this.data,
        params: this.params,
        state: this.state
      }
    }
  },

  watch: {
    Filters() {
      if (!this.DataURL || this.isLoading) return;

      this.page = 1;
      this.hasMoreData = true;
      this.data = [];
      this.startLoad = true;
      this.reloadCount++;
    },

    state() {
      this.emit()
    },

    data:{
      handler(){
        this.emit()
      }, 
      deep: true
    }
  },

  methods: {
    emit() {
      this.$emit('update:model-value', this.output)
    },

    async checkForMoreData() {
      const params = this.params;
      params.$page++;
      const response = await this.$http.get(this.DataURL, params);
      return !!response.data?.length;
    },

    async loadData(idx, done) {
      if (!this.DataURL || this.isLoading) return;

      this.hasMoreData = false;
      this.isLoading = true;

      try {
        let params = this.params
        if (!!this.BeforeLoad) params = this.BeforeLoad(this.params);


        const response = await this.$http.get(this.DataURL, params);
        var responseData = JSON.parse(JSON.stringify(response.data ?? []));
        const cloneData = responseData;

        if (!!this.AfterLoad) responseData = this.AfterLoad(cloneData, response) ?? responseData;

        if (responseData.length)
          this.data = [...this.data, ...responseData];

        this.hasMoreData = await this.checkForMoreData();

        this.page++
        if (!!done) done();
      } catch (error) {
        this.page = 1;
        this.errData = error
        console.error("There was a problem on the attempt to retrieve infinite scroll data.", error);
      } finally {
        this.isLoading = false;
        this.startLoad = false;
      }
    }
  },
}
</script>