<template>

  <div class="container-fluid mt-2">

    <moloch-loading v-if="initialLoading && !error">
    </moloch-loading>

    <moloch-error v-if="error"
      :message="error">
    </moloch-error>

    <div v-show="!error">

      <button type="button"
        v-b-tooltip.hover
        @click="cancelTasks"
        v-has-permission="'createEnabled'"
        title="Cancel ALL cancelable tasks"
        class="pull-right btn btn-sm btn-warning">
        <span class="fa fa-ban"></span>&nbsp;
        Cancel ALL Tasks
      </button>

      <moloch-paging v-if="stats"
        class="mt-1 ml-2"
        :info-only="true"
        :records-total="recordsTotal"
        :records-filtered="recordsFiltered">
      </moloch-paging>

      <moloch-table
        id="esTasksTable"
        :data="stats"
        :loadData="loadData"
        :columns="columns"
        :no-results="true"
        :action-column="true"
        :desc="query.desc"
        :sortField="query.sortField"
        page="esTasks"
        table-classes="table-sm text-right small mt-2"
        table-state-name="esTasksCols"
        table-widths-state-name="esTasksColWidths">
        <template slot="actions"
          slot-scope="{ item }">
          <a v-if="item.cancellable"
            class="btn btn-xs btn-danger"
            @click="cancelTask(item.taskId)"
            v-b-tooltip.hover
            title="Cancel task"
            v-has-permission="'createEnabled'">
            <span class="fa fa-trash-o">
            </span>
          </a>
        </template>
      </moloch-table>

    </div>

  </div>

</template>

<script>
import MolochTable from '../utils/Table';
import MolochError from '../utils/Error';
import MolochLoading from '../utils/Loading';
import MolochPaging from '../utils/Pagination';

let reqPromise; // promise returned from setInterval for recurring requests
let respondedAt; // the time that the last data load successfully responded

export default {
  name: 'EsTasks',
  props: [
    'user',
    'dataInterval',
    'refreshData',
    'searchTerm',
    'pageSize'
  ],
  components: {
    MolochTable,
    MolochError,
    MolochPaging,
    MolochLoading
  },
  data: function () {
    return {
      stats: null,
      error: '',
      initialLoading: true,
      totalValues: null,
      averageValues: null,
      recordsTotal: null,
      recordsFiltered: null,
      query: {
        filter: this.searchTerm || undefined,
        sortField: 'action',
        desc: false,
        cancellable: false
      },
      columns: [ // es tasks table columns
        // default columns
        { id: 'action', name: 'Action', sort: 'action', default: true, width: 200 },
        { id: 'description', name: 'Description', sort: 'description', default: true, width: 300, breakword: true },
        { id: 'start_time_in_millis', name: 'Start Time', sort: 'start_time_in_millis', width: 180, default: true, dataFunction: (item) => { return this.$options.filters.timezoneDateString(item.start_time_in_millis, this.user.settings.timezone, this.user.settings.ms); } },
        { id: 'running_time_in_nanos', name: 'Running Time', sort: 'running_time_in_nanos', width: 120, default: true, dataFunction: (item) => { return this.$options.filters.commaString(this.$options.filters.round(item.running_time_in_nanos / 1000000, 1)); } },
        { id: 'childrenCount', name: 'Children', sort: 'childrenCount', default: true, width: 100, dataFunction: (item) => { return this.$options.filters.roundCommaString(item.childrenCount); } },
        { id: 'user', name: 'User', sort: 'user', default: true, width: 100 },
        // all the rest of the available stats
        { id: 'cancellable', name: 'Cancellable', sort: 'cancellable', width: 100 },
        { id: 'id', name: 'ID', sort: 'id', width: 80 },
        { id: 'node', name: 'Node', sort: 'node', width: 180 },
        { id: 'taskId', name: 'Task ID', sort: 'taskId', width: 150 },
        { id: 'type', name: 'Type', sort: 'type', width: 100 }
      ]
    };
  },
  computed: {
    loading: {
      get: function () {
        return this.$store.state.loadingData;
      },
      set: function (newValue) {
        this.$store.commit('setLoadingData', newValue);
      }
    }
  },
  watch: {
    dataInterval: function () {
      if (reqPromise) { // cancel the interval and reset it if necessary
        clearInterval(reqPromise);
        reqPromise = null;

        if (this.dataInterval === '0') { return; }

        this.setRequestInterval();
      } else if (this.dataInterval !== '0') {
        this.loadData();
        this.setRequestInterval();
      }
    },
    'query.cancellable': function () {
      this.loadData();
    },
    refreshData: function () {
      if (this.refreshData) {
        this.loadData();
      }
    },
    pageSize: function () {
      this.loadData();
    }
  },
  created: function () {
    // set a recurring server req if necessary
    if (this.dataInterval !== '0') {
      this.setRequestInterval();
    }
  },
  methods: {
    /* exposed page functions ------------------------------------ */
    cancelTask (taskId) {
      this.$http.post(`api/estasks/${taskId}/cancel`)
        .then((response) => {
          // remove the task from the list
          for (let i = 0, len = this.stats.length; i < len; i++) {
            if (this.stats[i].taskId === taskId) {
              this.stats.splice(i, 1);
              return;
            }
          }
        }).catch((error) => {
          this.$emit('errored', error.text || error);
        });
    },
    cancelTasks () {
      this.$http.post('api/estasks/cancelall')
        .then((response) => {
          // remove cancellable tasks
          for (let i = this.stats.length - 1, len = 0; i >= len; i--) {
            if (this.stats[i].cancellable) {
              this.stats.splice(i, 1);
            }
          }
        }).catch((error) => {
          this.$emit('errored', error.text || error);
        });
    },
    /* helper functions ------------------------------------------ */
    setRequestInterval: function () {
      reqPromise = setInterval(() => {
        if (respondedAt && Date.now() - respondedAt >= parseInt(this.dataInterval)) {
          this.loadData();
        }
      }, 500);
    },
    loadData: function (sortField, desc) {
      this.loading = true;
      respondedAt = undefined;

      this.query.filter = this.searchTerm;

      this.query.size = this.pageSize || 500;

      if (desc !== undefined) { this.query.desc = desc; }
      if (sortField) { this.query.sortField = sortField; }

      this.$http.get('api/estasks', { params: this.query })
        .then((response) => {
          respondedAt = Date.now();
          this.error = '';
          this.loading = false;
          this.initialLoading = false;
          this.stats = response.data.data;
          this.recordsTotal = response.data.recordsTotal;
          this.recordsFiltered = Math.min(response.data.recordsFiltered, this.pageSize);
        }, (error) => {
          respondedAt = undefined;
          this.loading = false;
          this.initialLoading = false;
          this.error = error.text || error;
        });
    }
  },
  beforeDestroy: function () {
    if (reqPromise) {
      clearInterval(reqPromise);
      reqPromise = null;
    }
  }
};
</script>

<style>
#esTasksTable td {
  vertical-align: top;
}
</style>
