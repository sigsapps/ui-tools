<template>
  <div class="q-pa-xs q-pa-md-none">
    <!-- Options and Controls -->
    <div class="row q-pb-sm items-center">
      <div class="col-12 col-md-8">
        <!-- Options -->
        <q-btn v-if="availableFilters.length > 0" flat round color="primary" size="sm" icon="fas fa-filter"
          @click="showFilterPanel = !showFilterPanel">
          <q-tooltip>Filtros da tabela</q-tooltip>
        </q-btn>
        <q-btn flat round color="primary" size="sm" icon="fas fa-sync" @click="reload()">
          <q-tooltip>Atualizar lista</q-tooltip>
        </q-btn>
        <q-btn flat round color="primary" size="sm" icon="fas fa-columns">
          <q-tooltip>Colunas visíveis</q-tooltip>
          <q-menu class="q-pa-sm">
            <q-option-group v-model="visibleColumns" type="checkbox" :options="columnOptions"></q-option-group>
          </q-menu>
        </q-btn>
        <q-btn :disable="this.fullData.length === 0" v-if="!!Export" flat round color="primary" size="sm"
          icon="fas fa-file-download">
          <q-tooltip>Opções de Exportação</q-tooltip>
          <q-menu class="q-pa-sm">
            <q-list class="text-primary">
              <q-item v-close-popup dense v-for="(opt, idx) in exportOptions" :key="idx" clickable
                @click="exportFile(opt.filetype, opt.filename)">
                <q-item-section avatar>
                  <q-icon :name="opt.icon" size="xs"></q-icon>
                </q-item-section>
                <q-item-section>
                  {{ opt.label }}
                </q-item-section>
              </q-item>
            </q-list>
          </q-menu>
        </q-btn>
        <q-btn :disable="this.fullData.length === 0" v-if="!!Printable" flat round color="primary" size="sm"
          icon="fas fa-print" @click="printData()">
          <q-tooltip>Imprimir</q-tooltip>
        </q-btn>

        <!-- Custom Resources -->
        <q-btn v-for="(r, i) in CustomResources" :key="i" 
          :disable="this.fullData.length === 0" flat round color="primary" size="sm" :icon="r.icon ?? 'fas fa-gear'" 
          @click="handleItemClick(r)">
          <q-tooltip>{{ r.label ?? 'Recurso personalizado' }}</q-tooltip>
          <q-menu v-if="itemHasChildren(r)" class="q-pa-sm">
            <q-list class="text-primary">
              <q-item v-for="(opt, idx) in r.children" :key="idx" v-close-popup dense clickable
                @click="opt.fn({fullData, rawData, filters: filtersValues, ...opt.params})">
                <q-item-section v-if="opt.icon" avatar><q-icon :name="opt.icon" size="xs" /></q-item-section>
                <q-item-section>{{ opt.label }}</q-item-section>
              </q-item>
            </q-list>
          </q-menu>
        </q-btn>
      </div>

      <!-- Search Field -->
      <div class="col-12 col-md-4">
        <q-input v-if="searchableColumns.length > 0" dense square filled clearable label="Pesquisar na lista"
          v-model="searchTerm">
          <template v-slot:append>
            <q-icon size="xs" name="fas fa-search" color="grey-8" />
          </template></q-input>
      </div>
    </div>

    <q-separator></q-separator>

    <!-- Filters Panel -->
    <div v-if="availableFilters.length > 0" class="row">
      <div class="col-12">
        <q-expansion-item hide-expand-icon v-model="showFilterPanel" header-style="display:none;">
          <q-toolbar class="bg-grey-3">
            <q-toolbar-title>
              Filtros da Tabela
            </q-toolbar-title>
            <q-btn size="sm" icon="fas fa-filter-circle-xmark" color="primary" flat round dense
              @click="filterParams = {}">
              <q-tooltip>Limpar filtros</q-tooltip>
            </q-btn>
          </q-toolbar>
          <div class="row q-py-sm">
            <div v-for="(f, i) in availableFilters" :key="i" class="col-12 col-md-4">
              <InputField clearable dense :type="f.type" :withSeconds="f.filterOptions?.withSeconds"
                :Label="`Filtrar por ${f.label}`" :Options="f.options ?? []" v-model="filterParams[f.field]" :Default="f.default !== null && typeof f.default != 'undefined' ? f.default : null">
              </InputField>
            </div>
          </div>
        </q-expansion-item>
      </div>
    </div>

    <q-separator></q-separator>

    <!-- Table -->
    <div class="datatable-container">
      <table>
        <thead>
          <tr>
            <th v-show="visibleColumns.includes(column.field) || column.name == 'actions'"
              :class="`${dense ? 'q-pa-xs' : 'q-pa-sm'} ${column.sortable !== false ? 'cursor-pointer' : ''}`"
              v-for="column in columns" :key="column.field" @click="sort(column)"
              :style="column.width ? `width: ${column.width};` : ''">
              <span>{{ column.label }}</span>
              <q-icon v-if="column.sortable !== false" size="0.9em" :name="getSortIcon(column)"
                :color="getColumnNumber(column) == this.pagination.sortBy ? 'primary' : null">
              </q-icon>
              <q-tooltip v-if="column.sortable !== false">Clique para ordenar p/ {{ column.label }}</q-tooltip>
            </th>
          </tr>
        </thead>

        <!-- Ready State -->
        <tbody v-if="state == 'ready'">
          <template v-for="(row, idx) in rows" :key="idx">
            <tr v-if="row != 'interval'">
              <td v-show="visibleColumns.includes(column.field) || column.name == 'actions'"
                :class="`${dense ? 'q-pa-xs' : 'q-pa-sm'} ${(!!column.align) ? `text-${column.align}` : ''}`"
                v-for="column in columns" :key="column.field" :style="column.width ? `width: ${column.width};` : ''">

                <div v-if="column.name != 'actions'">
                  <!-- In case no template is set for the td-->
                  <div v-if="!(`cell-${column.name}` in $slots)">
                    {{ column.format ? column.format(row[column.field]) : row[column.field] }}
                  </div>

                  <!-- In case a template is set for the td-->
                  <div v-if="`cell-${column.name}` in $slots">
                    <slot :name="`cell-${column.name}`" :data="row"></slot>
                  </div>
                </div>

                <!-- Especial td of actions -->
                <div class="text-center" v-if="column.name == 'actions' && showActionsColumn">
                  <q-btn v-if="showActionsBtnInRow(row)" flat dense color="primary" icon="fas fa-ellipsis-v">
                    <q-tooltip>Ações do registro</q-tooltip>
                    <q-menu>
                      <q-list>
                        <q-item class="text-primary" v-show="typeof a.hide == 'function' ? !a.hide(row) : !a.hide"
                          v-for="(a, idx) in RowActions" :key="idx" clickable v-close-popup @click="a.fn(row)">
                          <q-item-section v-if="a.icon" side>
                            <q-icon color="primary" size="sm" :name="a.icon"></q-icon>
                          </q-item-section>
                          <q-item-section>{{ a.label }}</q-item-section>
                          <q-tooltip v-if="a.tooltip">{{ a.tooltip }}</q-tooltip>
                        </q-item>
                        <q-item class="text-primary" v-show="typeof a.hide == 'function' ? !a.hide(row) : !a.hide"
                          v-for="(a, idx) in injectedRowActions(row)" :key="idx" clickable v-close-popup
                          @click="a.fn(row)">
                          <q-item-section v-if="a.icon" side>
                            <q-icon color="primary" size="sm" :name="a.icon"></q-icon>
                          </q-item-section>
                          <q-item-section>{{ a.label }}</q-item-section>
                          <q-tooltip v-if="a.tooltip">{{ a.tooltip }}</q-tooltip>
                        </q-item>
                      </q-list>
                    </q-menu>
                  </q-btn>
                </div>
              </td>
            </tr>
            <tr v-else>
              <td :colspan="columns.length" :class="`${dense ? 'q-pa-xs' : 'q-pa-sm'}`">
                <slot name="interval-row"
                  :data="{ previous: dataInPage[idx - 1], current: dataInPage, next: dataInPage[idx + 1] }">
                </slot>
              </td>
            </tr>
          </template>
        </tbody>

        <!-- Error State -->
        <tbody v-if="state == 'error'">
          <tr>
            <td class="q-pa-lg text-center text-red-3" :colspan="columns.length">
              <div>
                <div><q-icon size="lg" name="fas fa-bomb"></q-icon></div>
                <div class="text-h6">ERRO!</div>
                <div class="text-caption"><b>{{ error.response.status }}</b> {{ error.response.statusText }}</div>
                <small>Favor entrar em contato com o administrador do sistema.</small>
              </div>
            </td>
          </tr>
        </tbody>

        <!-- Empty State -->
        <tbody v-if="state == 'empty'">
          <tr>
            <td class="q-pa-lg text-center text-grey-8" :colspan="columns.length">
              <div>
                <div><q-icon size="lg" name="far fa-folder-open"></q-icon> *</div>
                <div class="text-h6">Lista Vazia.</div>
              </div>
            </td>
          </tr>
        </tbody>

        <!-- Loading State -->
        <tbody v-if="state == 'loading'">
          <tr>
            <td class="q-pa-lg text-center text-grey-8" :colspan="columns.length">
              <div><q-spinner-gears size="lg" /></div>
              <div class="text-caption">Carregando...</div>
            </td>
          </tr>
        </tbody>

        <tfoot v-if="hasAnyColumnFooter || 'footer' in $slots">
          <tr class="text-bold">
            <td v-show="visibleColumns.includes(column.field) || column.name == 'actions'"
              :class="`${dense ? 'q-pa-xs' : 'q-pa-sm'} ${(!!column.align) ? `text-${column.align}` : ''}`"
              v-for="column in columns" :key="column.field" :style="column.width ? `width: ${column.width};` : ''">
              <div v-if="!!column.footer">
                <div v-if="`foot-${column.name}` in $slots">
                  <slot :name="`foot-${column.name}`"></slot>
                </div>
                <div v-else>
                  {{ typeof column.footer == 'function' ? column.footer(fullData) : column.footer }}
                </div>
              </div>
            </td>
          </tr>
          <tr v-if="'footer' in $slots">
            <slot :name="`footer`"></slot>
          </tr>
        </tfoot>

      </table>

    </div>

    <!-- Pagination -->
    <div v-if="!IgnorePagination" class="row q-mt-lg" v-show="state == 'ready'">
      <div class="col-12 q-py-sm">
        <q-separator></q-separator>
      </div>
      <div :class="`col-12 col-md-6 ${$q.screen.lt.md ? 'text-center' : ''}`">
        <div>
          Mostrar até
          <select class="q-pa-xs" v-model="pagination.limit">
            <option value="5">5</option>
            <option value="10">10</option>
            <option value="25">25</option>
            <option value="50">50</option>
          </select>
          registros de um total de
          <b v-if="fullData.length">{{ fullData.length }}</b>
          <q-spinner-gears v-else size="xs" />.
        </div>
      </div>
      <div :class="`col-12 col-md-6 ${$q.screen.gt.sm ? 'text-right' : 'text-center q-mt-md'}`">
        <q-btn :disable="fullData.length === 0" color="primary" @click="goToPage(1)">
          <q-tooltip>Primeira Página</q-tooltip>
          <q-icon size="xs" name="fas fa-angles-left"></q-icon>
        </q-btn>
        <q-btn color="primary" @click="goToPage('prev')">
          <q-tooltip>Página Anterior</q-tooltip>
          <q-icon size="xs" name="fas fa-chevron-left"></q-icon>
        </q-btn>
        <q-btn class="q-px-sm" color="primary" v-for="page in pagination.pages" :key="page" @click="goToPage(page)"
          :flat="page == pagination.currentPage">
          <q-tooltip>Página {{ page }}</q-tooltip>
          {{ page }}
        </q-btn>
        <q-btn color="primary" @click="goToPage('next')">
          <q-tooltip>Próxima Página</q-tooltip>
          <q-icon size="xs" name="fas fa-chevron-right"></q-icon>
        </q-btn>
        <q-btn :disable="fullData.length === 0" color="primary" @click="goToPage(this.pagination.finalPage)">
          <q-tooltip>Última Página</q-tooltip>
          <q-icon size="xs" name="fas fa-angles-right"></q-icon>
        </q-btn>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'ui-gadgets-datatable',

  props: {
    Name: {
      type: String,
      required: true
    },
    modelValue: {
      type: Object,
      required: true
    },
    DataURL: {
      type: String,
      required: true
    },
    Columns: {
      type: Object,
      required: true
    },
    DefaultSorting: {
      type: Object,
      validator: (v) => ('by' in v) && ('direction' in v)
    },
    RowActions: Object,
    Filters:{
      type: Array,
      default: () => []
    },
    ExtraFilters: Object,
    Export: Object,
    Printable: Boolean,
    BeforeLoad: Function,
    OnLoaded: Function,
    IntervalRule: Function,
    IgnorePagination: Boolean,
    dense: Boolean,
    injectedRowActions: {
      type: Function,
      default: (row) => []
    },
    CustomResources: Array,
  },

  data() {
    return {
      // Pagination related vars:
      pagination: {
        pages: [],
        currentPage: 1,
        finalPage: null,
        totalPages: null,
        totalItems: null,
        pageFirstItem: null,
        pageLastItem: null,
        pageLastIndex: null,
        limit: 10,
        sortBy: '1',
        sortDir: 'ASC',
      },

      // Filters related vars:
      searchTerm: null,
      showFilterPanel: false,
      filterParams: {},

      // Columns settings:
      visibleColumns: [],
      columns: [],

      // Data:
      fullData: [],
      rawData: [],
      dataInPage: [],
      loadTimeout: null,
      error: null,

      // State:
      loading: false,
      showLoader: true,
      errorState: false
    }
  },

  watch: {
    searchTerm() {
      this.showLoader = true;
      this.pagination.currentPage = 1;
      clearTimeout(this.loadTimeout);

      this.loadTimeout = setTimeout(async () => {
        if (!!this.searchTerm)
          localStorage.setItem(`Datatable.${this.sluggedName}.searchTerm`, this.searchTerm)
        else localStorage.removeItem(`Datatable.${this.sluggedName}.searchTerm`)

        const response = await this.loadData(this.IgnorePagination);
        if (response) this.rawData = response.data;
      }, 200);
    },

    'pagination.currentPage'() {
      var persistedPagination = localStorage.getItem(`Datatable.${this.sluggedName}.pagination`);
      persistedPagination = !!persistedPagination ? JSON.parse(persistedPagination) : {};
      persistedPagination.currentPage = this.pagination.currentPage;
      localStorage.removeItem(`Datatable.${this.sluggedName}.pagination`)
      localStorage.setItem(`Datatable.${this.sluggedName}.pagination`, JSON.stringify(persistedPagination))
      clearTimeout(this.loadTimeout);

      this.loadTimeout = setTimeout(async () => {
        const response = await this.loadData(this.IgnorePagination);
        if (response) this.rawData = response.data;
      }, 200);
    },

    'pagination.limit'() {
      this.pagination.currentPage = 1;
      var persistedPagination = localStorage.getItem(`Datatable.${this.sluggedName}.pagination`);
      persistedPagination = !!persistedPagination ? JSON.parse(persistedPagination) : {};
      persistedPagination.currentPage = this.pagination.currentPage;
      persistedPagination.limit = this.pagination.limit;
      localStorage.removeItem(`Datatable.${this.sluggedName}.pagination`)
      localStorage.setItem(`Datatable.${this.sluggedName}.pagination`, JSON.stringify(persistedPagination))
      clearTimeout(this.loadTimeout);

      this.loadTimeout = setTimeout(async () => {
        const response = await this.loadData(this.IgnorePagination);
        if (response) this.rawData = response.data;
      }, 200);
    },

    'pagination.sortBy'() {
      var persistedPagination = localStorage.getItem(`Datatable.${this.sluggedName}.pagination`);
      persistedPagination = !!persistedPagination ? JSON.parse(persistedPagination) : {};
      persistedPagination.sortBy = this.pagination.sortBy;
      localStorage.removeItem(`Datatable.${this.sluggedName}.pagination`)
      localStorage.setItem(`Datatable.${this.sluggedName}.pagination`, JSON.stringify(persistedPagination))
      clearTimeout(this.loadTimeout);

      this.loadTimeout = setTimeout(async () => {
        const response = await this.loadData(this.IgnorePagination);
        if (response) this.rawData = response.data;
      }, 200);
    },

    'pagination.sortDir'() {
      var persistedPagination = localStorage.getItem(`Datatable.${this.sluggedName}.pagination`);
      persistedPagination = !!persistedPagination ? JSON.parse(persistedPagination) : {};
      persistedPagination.sortDir = this.pagination.sortDir;
      localStorage.removeItem(`Datatable.${this.sluggedName}.pagination`)
      localStorage.setItem(`Datatable.${this.sluggedName}.pagination`, JSON.stringify(persistedPagination))
      clearTimeout(this.loadTimeout);

      this.loadTimeout = setTimeout(async () => {
        const response = await this.loadData(this.IgnorePagination);
        if (response) this.rawData = response.data;
      }, 200);
    },

    filterParams: {
      handler(v) {
        this.filterHandler(v, 'filters')
      },
      deep: true
    },

    ExtraFilters: {
      handler(v) {
        this.filterHandler(v, 'extrafilters');
      },
      deep: true
    },

    visibleColumns(newVal) {
      if (newVal.length < 1) {
        this.visibleColumns = [this.Columns[0].field];
      }

      localStorage.setItem(`Datatable.${this.sluggedName}.visibleColumns`, JSON.stringify(newVal));
    },

    loading(isLoading) {
      this.showLoader = isLoading;
    },

    rawData: {
      handler(data) {
        if (this.IgnorePagination) this.dataInPage = data;
        else this.paginate(data);
        // Expose factory:
        this.exposeFactory();
        // turn off loading indicator
        this.loading = false
      },
      deep: true
    },

    fullData: {
      handler(data) {
        if (data.length > 0) {
          this.pagination.finalPage = Math.ceil(data.length / this.pagination.limit);
        } else {
          this.pagination.finalPage = this.pagination.currentPage + (Math.ceil(this.rawData.length / this.pagination.limit) - 1);
        }
      }
    }
  },

  computed: {
    showActionsColumn() {
      // Show the actions column if at least one row has a visible action
      return this.dataInPage.some(row => this.showActionsBtnInRow(row));
    },

    columnOptions() {
      return this.columns.map(clm => {
        return clm.name != 'actions' ? {
          label: clm.label,
          value: clm.field,
        } : null;
      }).filter(item => item != null);
    },

    columnFilters() {
      return this.Columns.map(clm => {
        if (!!clm.filter == false) return null;

        if (typeof clm.filter == 'string') {
          return {
            label: clm.label,
            field: clm.field,
            type: clm.filter
          };
        }

        if (!('field' in clm.filter))
          clm.filter.field = clm.field;

        return { label: clm.label, ...clm.filter };
      }).filter(item => item != null);
    },

    exportOptions() {
      var typeIcons = {
        xls: 'fas fa-file-excel',
        csv: 'fas fa-file-csv',
      };
      var typeLabels = {
        xls: 'Exportar XLS',
        csv: 'Exportar CSV',
      };

      if (!(this.Export instanceof Array))
        return [{
          ...this.Export,
          icon: typeIcons[this.Export.filetype],
          label: typeLabels[this.Export.filetype]
        }];

      return this.Export.map((opt) => ({
        ...opt,
        icon: typeIcons[opt.filetype],
        label: typeLabels[opt.filetype]
      }));
    },

    state() {
      if (this.showLoader) return 'loading';
      if (this.errorState) return 'error';
      if (this.dataInPage.length > 0) return 'ready';
      if (this.dataInPage.length == 0) return 'empty';
      return null;
    },

    searchableColumns() {
      return this.columns.filter(col => {
        const hasValidField = col.field && col.field !== '';
        const isSearchable = col.searchable !== false;
        const isNotFiltered = !(col.field in this.filterParams);

        return hasValidField && isSearchable && isNotFiltered;
      })
    },

    rows() {
      if (typeof this.IntervalRule !== 'function') return [...this.dataInPage];

      const result = [];

      this.dataInPage.forEach((current, i) => {
        const prev = this.dataInPage[i - 1];
        const next = this.dataInPage[i + 1];

        result.push(current);
        if (this.IntervalRule(prev, current, next) === true) result.push('interval');
      });

      return result;
    },

    hasAnyColumnFooter() {
      return this.Columns.some(column => !!column.footer);
    },

    filtersValues() {
      return {
        searchTerm: this.searchTerm,
        ...this.filterParams,
        ...this.ExtraFilters
      }
    },

    sluggedName() {
      // Remove spaces, lowercases and replace accents and special chars:
      return this.Name.toSlug();
    },

    availableFilters() {
      return [...this.columnFilters, ...this.Filters]
    }
  },

  methods: {
    /////////////
    // Factory:
    /////////////
    exposeFactory() {
      this.$emit('update:model-value', {
        state: this.state,
        params: this.setParams(),
        filterValues: this.filtersValues,
        dataInPage: this.dataInPage,
        visibleColumns: this.visibleColumns,
        reload: this.reload
      });
    },

    /////////////
    // Getters:
    /////////////
    async loadData(ignorePagination) {
      if (!this.loading) {
        this.loadFullData();

        // turn on loading indicator
        this.loading = true;
        this.error = null;
        this.errorState = false;

        var params = this.setParams();
        if (!!ignorePagination) {
          delete params.$page;
          delete params.$limit;
        }

        try {
          // Before Load callback:
          if (this.BeforeLoad) await this.BeforeLoad(params);

          // fetch data from server
          const response = await this.$http.get(this.DataURL, params);

          // On Loaded callback:
          if (this.OnLoaded) await this.OnLoaded(response);

          return response;
        } catch (err) {
          this.loading = false;
          this.errorState = true;
          this.error = err;
          this.$emit('error-thrown', err);
          throw err;
        }
      }
    },

    async loadFullData() {
      this.fullData = [];
      var params = this.setParams();
      delete params.$page;
      delete params.$limit;

      if (this.BeforeLoad) await this.BeforeLoad(params);

      const response = await this.$http.get(this.DataURL, params);
      this.fullData = response.data;
    },

    /////////////////////
    // Sort and Filter:
    /////////////////////
    filterHandler(filtersObject, name) {
       console.log(
        name,
        localStorage.getItem(`Datatable.${this.sluggedName}.${name}`),
      );
      // Save filters state:
      localStorage.removeItem(`Datatable.${this.sluggedName}.${name}`);

      if (Object.keys(filtersObject).length > 0)
        localStorage.setItem(`Datatable.${this.sluggedName}.${name}`, JSON.stringify(filtersObject));

      this.showLoader = true;
      this.pagination.currentPage = 1;
      clearTimeout(this.loadTimeout);

      for (let k in filtersObject) {
        if (filtersObject[k] == null)
          delete filtersObject[k]
      }

      this.loadTimeout = setTimeout(async () => {
        const response = await this.loadData(this.IgnorePagination);
        if (response) this.rawData = response.data
      }, 200);
    },

    setParams() {
      // Pagination Params:
      const pagination = {
        '$sort_by': this.pagination.sortBy ?? '1',
        '$sort_direction': this.pagination.sortDir ?? 'ASC',
        '$page': this.pagination.currentPage,
        '$limit': Number(this.pagination.limit)
      };

      // Search Filter:
      const search = {};
      if (this.searchTerm) {
        for (let i = 0; i < this.searchableColumns.length; i++) {
          let column = this.searchableColumns[i];

          const f = column.field;

          // First field:
          if (i == 0) {
            if (i == this.searchableColumns.length - 1) {
              search[f] = '$startFilterGroup$lkof$endFilterGroup|' + this.searchTerm;
            } else {
              search[f] = '$startFilterGroup$lkof|' + this.searchTerm;
            }
          }
          // All fields in the middle:
          else if (i < (this.searchableColumns.length - 1)) {
            search[f] = '$or$lkof|' + this.searchTerm;
          }
          // Last field:
          else {
            search[f] = '$endFilterGroup$or$lkof|' + this.searchTerm;
          }
        }
      }

      // Filters:
      const filters = {};
      for (let k in this.filterParams) {
        let filterConfig = this.availableFilters.find(x => x.field == k);
        let value = (!!filterConfig?.modifierFn && typeof filterConfig?.modifierFn === 'function') ? filterConfig?.modifierFn(this.filterParams[k]) : this.filterParams[k];
        if(value == null) continue;

        if (filterConfig?.type == 'text')
          filters[k] = `$lkof|${value}`;
        else if (filterConfig?.type == 'daterange' || filterConfig?.type == 'datetimerange')
          filters[k] = `$btwn|${value.from}|${value.to}`;
        else filters[k] = value;
      }

      // Extra filters:
      const extraFilters = {};
      if (!!this.ExtraFilters) {
        for (let k in this.ExtraFilters)
          if (!!this.ExtraFilters[k])
            extraFilters[k] = this.ExtraFilters[k];
      }

      return {
        ...pagination,
        ...search,
        ...filters,
        ...extraFilters
      };
    },

    getColumnNumber(column) {
      var sortNumber = null;
      if (!!column.sortBy === false) {
        // Find sort number:
        let columns = Object.keys(this.rawData[0] ?? {});
        let idx = columns.indexOf(column.field);

        if (idx == -1) return

        sortNumber = idx + 1
      } else if (typeof column.sortBy == 'string') {
        // Find sort number:
        let columns = Object.keys(this.rawData[0] ?? {});
        let idx = columns.indexOf(column.sortBy);

        if (idx == -1) return

        sortNumber = idx + 1
      } else {
        sortNumber = column.sortBy
      }

      return sortNumber;
    },

    getColumnLabel(field) {
      const columns = this.Columns.map(c => ({label: c.label, field: c.field}));
      const filters = this.Filters.map(f => ({label: f.label, field: f.field}));
      const available = [...columns, ...filters];

      let target = available.find(item => item.field == field);

      return target?.label ?? field;
    },

    getSortIcon(column) {
      if (this.getColumnNumber(column) == this.pagination.sortBy) {
        if (this.pagination.sortDir == 'ASC') return 'fas fa-sort-up';
        else if (this.pagination.sortDir == 'DESC') return 'fas fa-sort-down';
        else return 'fas fa-ban';
      } else return 'fas fa-sort';
    },

    sort(column) {
      if (column.sortable === false || (!!column.sortBy === false && this.rawData.length == 0)) return;

      var sortNumber = this.getColumnNumber(column)

      if (this.pagination.sortBy == sortNumber) {
        if (this.pagination.sortDir == 'ASC') this.pagination.sortDir = 'DESC';
        else if (this.pagination.sortDir == 'DESC') this.pagination.sortDir = 'ASC';

      } else {
        this.pagination.sortBy = sortNumber;
        this.pagination.sortDir = 'ASC';
      }

    },

    showActionsBtnInRow(row) {
      return this.RowActions.some(action => {
        const hide = action.hide;

        // Execution
        if (hide === undefined) return true;
        if (hide === Boolean) return !hide;
        if (typeof hide === 'function') return !hide(row, action);

        return true;
      });
    },

    ///////////////////////////
    // Operational Functions:
    ///////////////////////////
    reload() {
      clearTimeout(this.loadTimeout);

      this.loadTimeout = setTimeout(async () => {
        const response = await this.loadData(this.IgnorePagination);
        if (response) this.rawData = response.data;
      }, 200);
    },

    paginate(data) {
      this.pagination.finalPage = this.pagination.currentPage + (Math.ceil(data.length / this.pagination.limit) - 1);

      var initial = null;

      if (this.pagination.finalPage - this.pagination.currentPage < 1) {
        initial = this.pagination.currentPage - 4;
      } else if (this.pagination.finalPage - this.pagination.currentPage < 2) {
        initial = this.pagination.currentPage - 3;
      } else {
        initial = this.pagination.currentPage - 2;
      }
      initial = initial < 1 ? 1 : initial;

      this.pagination.pages = [];
      for (let i = initial; i <= this.pagination.finalPage; i++) {
        this.pagination.pages.push(i);
        if (this.pagination.pages.length > 4) break;
      }

      this.pagination.pageFirstItem = data.length > 0 ? (((this.pagination.currentPage * this.pagination.limit) - this.pagination.limit) + 1) : 0;

      if (data.length > 0) {
        this.pagination.pageLastItem = this.pagination.pageFirstItem;
        for (let i = 1; i < data.length; i++) {
          this.pagination.pageLastItem++;
        }
      } else this.pagination.pageLastItem = 0;

      this.pagination.pageLastIndex = data.length < this.pagination.limit ? data.length - 1 : this.pagination.limit - 1;

      this.dataInPage = [];
      for (let i = 0; i <= this.pagination.pageLastIndex; i++) {
        this.dataInPage.push(this.rawData[i]);
      }

      if (this.dataInPage.length == 0 && this.pagination.currentPage > 1) {
        this.goToPage('prev');
      }
    },

    async goToPage(page) {
      switch (page) {
        case 'next':
          if ((this.pagination.currentPage + 1) > this.pagination.finalPage) return;
          this.pagination.currentPage++;
          break;
        case 'prev':
          if ((this.pagination.currentPage - 1) < 1) return;
          this.pagination.currentPage--;
          break;
        default:
          if (page != this.pagination.currentPage)
            this.pagination.currentPage = page;
      }
    },

    async exportFile(filetype, filename) {
      filename = filename.indexOf(`.${filetype}`) ? filename : `${filename}.${filetype}`;

      var data = this.fullData;
      var blobType;
      var content;

      switch (filetype) {
        case 'xls':
          blobType = "application/vnd.ms-excel;charset=utf-8;";
          content = this.buildContentTable(data);
          break;
        case 'csv':
          blobType = "text/csv;charset=utf-8;";
          content = this.buildCsvContent(data);
          break;
      }

      // Encode the content to UTF-8
      const encoder = new TextEncoder();
      const utf8Content = encoder.encode(content);

      // Create a blob with the UTF-8 encoded content
      const blob = new Blob([utf8Content], { type: blobType });

      const link = document.createElement("a");
      const url = URL.createObjectURL(blob);
      link.setAttribute("href", url);
      link.setAttribute("download", filename);

      link.style.visibility = "hidden";

      // Adiciona o link ao DOM e dispara o clique
      document.body.appendChild(link);
      link.click();
      document.body.removeChild(link);

      this.loading = false;
    },

    escapeXml(value) {
      if (value === null || value === undefined) return "";
      if (value === false) return "Não";
      if (value === true) return "Sim";

      return String(value)
        .replace(/&/g, "&amp;")
        .replace(/</g, "&lt;")
        .replace(/>/g, "&gt;")
        .replace(/"/g, "&quot;")
        .replace(/'/g, "&#39;");
    },

    buildContentTable(data) {
      // Início do XML
      let content = `
      <?xml version="1.0" encoding="UTF-8"?>
  <html xmlns:o="urn:schemas-microsoft-com:office:office"
        xmlns:x="urn:schemas-microsoft-com:office:excel"
        xmlns="this.$http://www.w3.org/TR/REC-html40">
    <head>
      <style>
        table {
          border-collapse: collapse;
          width: 100%;
        }
        th, td {
          border: 1px solid #000;
          padding: 5px;
          text-align: left;
        }
        th {
          background-color: #f4f4f4;
          text-align: center;
        }
      </style>
      <meta this.$http-equiv="Content-Type" content="text/html; charset=UTF-8">
    </head>
    <body>
  <table>
    <thead>
      <tr>
        ${this.Columns
          .filter(column => this.visibleColumns.includes(column.field))
          .map(column => `<th>${this.escapeXml(column.label)}</th>`).join("")}
      </tr>
    </thead>
    <tbody>
  `;

      // Adiciona as linhas
      for (let j = 0; j < data.length; j++) {
        content += '<tr>';
        let row = data[j];

        for (let i = 0; i < this.Columns.length; i++) {
          let clm = this.Columns[i];
          if (!(this.visibleColumns.includes(clm.field))) continue;
          content += `<td>${!!clm.format ? clm.format(row) : (!!row[clm.field] ? row[clm.field] : '')}</td>`;
        }
        content += '</tr>';
      }
      // Fecha o XML
      content += `
    </tbody>
  </table>
  `;

      return content;
    },

    buildContentTableForPrint(data) {
      const visible = this.Columns.filter(c => this.visibleColumns.includes(c.field));
      var filters = Object.entries(this.filtersValues)
        .filter(([key, value]) => value !== null && value !== undefined && value !== '')
        .map(([key, value]) => `<li>&#8250; ${this.escapeXml(this.getColumnLabel(key))}: <span style="font-weight: normal;">"${this.escapeXml(value)}"</span></li>`).join("");
      if (filters.length == 0) filters = "<li>Nenhum filtro aplicado</li>";

      let content = `<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>${this.Name}</title>
  <style>
    @page { margin: 10mm; }
    body { font: 12px system-ui, -apple-system, Segoe UI, Roboto, Arial; }
    table { border-collapse: collapse; width: 100%; }
    th, td { border: 1px solid #000; padding: 4px; text-align: left; }
    th { background: #f4f4f4; text-align: center; }

    thead { display: table-header-group; } /* repete header */
    tfoot { display: table-footer-group; }
    tr { break-inside: avoid; page-break-inside: avoid; }
  </style>
</head>
<body>
  <table>
    <thead>
      <tr><th colspan="${visible.length}" style="text-align: center; font-size: 16px;">${this.Name}</th></tr>
      <tr><th colspan="${visible.length}" style="text-align: left; font-size: 12px;">
        <span style="text-decoration:underline;">Filtros:</span>
        <br>
        <br>
        <ul style="list-style-type: none; padding: 0; margin: 0;">${filters}</ul></th>
      </tr>
      <tr>${visible.map(c => `<th>${this.escapeXml(c.label)}</th>`).join("")}</tr>
  </thead>
  <tbody>`;

      for (let j = 0; j < data.length; j++) {
        const row = data[j];
        content += "<tr>";
        for (const clm of visible) {
          const val = clm.format ? clm.format(row) : (row?.[clm.field] ?? "");
          content += `<td>${this.escapeHtml(String(val))}</td>`;
        }
        content += "</tr>";
      }

      content += `</tbody>
</table>
</body>
<tfoot>
  <tr><td colspan="${visible.length}">
    <em>Total de registros: ${data.length}</em>
  </td></tr>
</tfoot>
</html>`;

      return content;
    },

    escapeHtml(s) {
      return s
        .replaceAll("&", "&amp;")
        .replaceAll("<", "&lt;")
        .replaceAll(">", "&gt;")
        .replaceAll('"', "&quot;")
        .replaceAll("'", "&#39;");
    },

    buildCsvContent(rawdata) {
      // Ensure the data is an array of objects
      if (!Array.isArray(rawdata) || !rawdata.length || typeof rawdata[0] !== 'object') {
        console.error('The provided data could not be converted to CSV.');
        return;
      }

      // Extract headers (keys of the first object in the array)
      const headers = this.Columns
        .filter(column => this.visibleColumns.includes(column.field))
        .map(column => column.label)

      const data = [];
      for (let j = 0; j < rawdata.length; j++) {
        let row = rawdata[j];
        let rowValues = [];

        for (let i = 0; i < this.Columns.length; i++) {
          let clm = this.Columns[i];
          if (!(this.visibleColumns.includes(clm.field))) continue;
          const f = clm.export ?? clm.field
          rowValues.push(row[f]);
        }
        data.push(rowValues);
      }

      // Generate CSV content
      const csvContent = [
        headers.join(';'), // Join headers with commas
        ...data.map(row => row.join(';')) // Map each row to CSV string
      ].join('\r\n'); // Separate rows with a newline

      return csvContent;
    },

    async printData() {
      const data = this.fullData;
      this.loading = false;

      const html = this.buildContentTableForPrint(data);

      const iframe = document.createElement("iframe");
      iframe.style.position = "fixed";
      iframe.style.right = "0";
      iframe.style.bottom = "0";
      iframe.style.width = "0";
      iframe.style.height = "0";
      iframe.style.border = "0";
      document.body.appendChild(iframe);

      const doc = iframe.contentWindow.document;
      doc.open();
      doc.write(html);
      doc.close();

      // dá um tempo pro layout e reflow terminarem
      await new Promise(r => setTimeout(r, 500));

      iframe.contentWindow.focus();
      iframe.contentWindow.print();

      setTimeout(() => iframe.remove(), 1500);
    },

    //////////////////
    // Item Functions:
    //////////////////
    handleItemClick(item) {
      if (this.itemHasChildren(item)) return;

      item.fn({
        fullData: this.fullData,
        rawData: this.rawData,
        filters: this.filtersValues,
        ...item.params
      });
    },

    itemHasChildren(item) {
      return !!item?.children;
    },
  },

  async mounted() {
    // Set columns:
    this.columns = [...this.Columns];
    this.visibleColumns = JSON.parse(localStorage.getItem(`Datatable.${this.sluggedName}.visibleColumns`)) ?? this.Columns.map(clm => clm.field)

    // Set Actions:
    if (this.RowActions && this.RowActions?.length > 0)
      this.columns.push({
        name: 'actions',
        label: 'Ações',
        align: 'center',
        sortable: false,
        filterable: false
      });

    var loadFirstData = true;
    // Set persisted filters:
    var persistedFilters = localStorage.getItem(`Datatable.${this.sluggedName}.filters`);
    if (!!persistedFilters) {
      this.showFilterPanel = true;
      setTimeout(() => this.filterParams = JSON.parse(persistedFilters), 100)
      loadFirstData = false
    }

    // Set persisted search term:
    this.searchTerm = localStorage.getItem(`Datatable.${this.sluggedName}.searchTerm`) ?? null;
    if (!!this.searchTerm) loadFirstData = false;

    // Set sorting:
    if (!!this.DefaultSorting) {
      this.pagination.sortBy = this.DefaultSorting.by;
      this.pagination.sortDir = this.DefaultSorting.direction;
      loadFirstData = false
    }

    // Set persisted pagination:
    var persistedPagination = localStorage.getItem(`Datatable.${this.sluggedName}.pagination`);
    if (!!persistedPagination) {
      persistedPagination = JSON.parse(persistedPagination);
      for (let k in persistedPagination)
        if (k in this.pagination && k != 'currentPage')
          this.pagination[k] = persistedPagination[k]

      if (!!persistedPagination.currentPage && persistedPagination.currentPage != 1) {
        setTimeout(() => this.pagination.currentPage = persistedPagination.currentPage, 200);
        loadFirstData = false
      }
    }

    // If no change occurred in any parameters, start the first load:
    if (loadFirstData) {
      const response = await this.loadData();
      if (response) this.rawData = response.data;
    }
  },
}
</script>

<style scoped>
.datatable-container {
  position: relative;
  overflow-x: scroll;
  width: 100%;
  max-height: 500px;
}

table {
  width: 100%;
  border-collapse: collapse;
  overflow-y: scroll;
}

thead {
  top: 0px;
  position: sticky;
  z-index: 2;
  background-color: white;
}

th {
  white-space: nowrap;
  /* Prevents text from wrapping */
  width: fit-content;
  /* Adjusts width to fit the content */
}

tbody>tr:nth-child(even) {
  background-color: #e2e2e2;
}

td {
  min-width: 125px;
}

tfoot {
  position: sticky;
}

select {
  cursor: pointer;
}

.cursor-pointer {
  cursor: pointer;
}
</style>
