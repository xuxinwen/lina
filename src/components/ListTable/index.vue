<template>
  <div>
    <TableAction :table-url="tableConfig.url" :search-table="search" v-bind="headerActions" :selected-rows="selectedRows" :reload-table="reloadTable" />
    <IBox class="table-content">
      <AutoDataTable :key="tableConfig.url" ref="dataTable" :config="tableConfig" @selection-change="handleSelectionChange" />
      <Dialog :title="$tc('Export')" :visible.sync="showExportDialog" center @confirm="handleDialogConfirm('export')" @cancel="handleDialogCancel('export')">
        <el-form>
          <el-form-item :label="this.$t('action.ExportRange')" :label-width="'100px'">
            <el-radio v-model="exportOption" class="export-item" label="1">{{ this.$t('action.ExportAll') }}</el-radio>
            <el-radio v-model="exportOption" class="export-item" label="2">{{ this.$t('action.ExportOnlySelectedItems') }}</el-radio>
            <!-- <el-radio v-model="exportOption" class="export-item" label="3">仅导出搜索项</el-radio> -->
          </el-form-item>
        </el-form>
      </Dialog>
      <Dialog :title="importTitle" :visible.sync="showImportDialog" center @confirm="handleDialogConfirm('import')" @cancel="handleDialogCancel('import')">
        <el-form>
          <el-form-item :label="importTitle" :label-width="'100px'">
            <el-radio v-model="importOption" class="export-item" label="1">{{ this.$tc('Import') }}</el-radio>
            <el-radio v-model="importOption" class="export-item" label="2">{{ this.$tc('Update') }}</el-radio>
          </el-form-item>
        </el-form>
        <div v-if="importOption==='1'" style="margin-bottom:20px;margin-left: 55px;">{{ this.$t('action.DownloadTheImportedTemplateOrUseTheExportedCSVFormat') }} <a style="color: #428bca;" :href="downloadImportTempUrl">{{ this.$t('action.DownloadImportTemplate') }}</a></div>
        <div v-else style="margin-bottom:20px;margin-left: 55px;">{{ this.$t('action.DownloadTheUpdatedTemplateOrUsTheExportedCSVFormat') }} <a style="color: #428bca;" @click="downloadUpdateTempUrl">{{ this.$t('action.DownloadUpdateTemplate') }}</a></div>

        <div style="margin-left:55px;">
          <el-upload
            class="upload-card"
            action="string"
            :http-request="upload"
            list-type="text/csv"
            :limit="1"
          >
            <el-button size="small" type="primary">{{ this.$t('action.Upload') }}</el-button>
            <div slot="tip" class="el-upload__tip">{{ this.$t('action.OnlyCSVFilesCanBeUploaded') }}</div>
          </el-upload>
        </div>
      </Dialog>
    </IBox>
  </div>
</template>

<script>
import AutoDataTable from '../AutoDataTable'
import Dialog from '../Dialog'
import IBox from '../IBox'
import TableAction from './TableAction'
import { createSourceIdCache } from '@/api/common'
import Emitter from '@/mixins/emitter'

export default {
  name: 'ListTable',
  components: {
    AutoDataTable,
    TableAction,
    Dialog,
    IBox
  },
  mixins: [Emitter],
  props: {
    // 定义 table 的配置
    tableConfig: {
      type: Object,
      default: () => ({})
    },
    // 是否显示table左侧的action
    headerActions: {
      type: Object,
      default: () => ({})
    }

  },
  data() {
    return {
      selectedRows: [],
      showExportDialog: false,
      showImportDialog: false,
      importOption: '1',
      exportOption: '1',
      meta: {}
    }
  },
  computed: {
    hasSelected() {
      return this.selectedRows.length > 0
    },
    upLoadUrl() {
      return this.tableConfig.url
    },
    importTitle() {
      if (this.importOption === '1') { return this.$tc('Import') } else { return this.$tc('Update') }
    },
    downloadImportTempUrl() {
      return process.env.VUE_APP_BASE_API + this.tableConfig.url + '?format=csv&template=import&limit=1'
    }
  },
  mounted() {
    this.$eventBus.$on('showExportDialog', (row) => {
      this.showExportDialog = true
    })
    this.$eventBus.$on('showImportDialog', (row) => {
      this.showImportDialog = true
    })
  },
  methods: {
    upload(item) {
      this.$axios.put(
        this.upLoadUrl,
        item.file
      ).then((res) => {
        console.log('')
      })
    },
    downloadCsv(url) {
      const a = document.createElement('a')
      a.href = url
      a.click()
      window.URL.revokeObjectURL(url)
    },
    handleSelectionChange(val) {
      this.selectedRows = val
      var obj = {}
      val.forEach((item, index) => { obj[index] = item })
      // 已知Bug，必须避免数组扁平化
      this.dispatch('AssetSelect', 'SelectionChange', obj)
    },
    reloadTable() {
      this.$refs.dataTable.$refs.dataTable.getList()
    },
    search(attrs) {
      return this.$refs.dataTable.$refs.dataTable.search(attrs)
    },
    async handleExport() {
      let data
      var resources = []
      if (this.exportOption === '1') {
        data = this.$refs.dataTable.$refs.dataTable.getData()
      } else if (this.exportOption === '2') {
        data = this.selectedRows
      } else {
        data = []
      }
      for (let index = 0; index < data.length; index++) {
        resources.push(data[index].id)
      }
      const spm = await createSourceIdCache(resources)
      const url = process.env.VUE_APP_BASE_API + `${this.tableConfig.url}?format=csv&?spm=` + spm.spm
      return this.downloadCsv(url)
    },
    handleImport() {
    },
    async downloadUpdateTempUrl() {
      var resources = []
      const data = this.$refs.dataTable.$refs.dataTable.getData()
      for (let index = 0; index < data.length; index++) {
        resources.push(data[index].id)
      }
      const spm = await createSourceIdCache(resources)
      const url = process.env.VUE_APP_BASE_API + `${this.tableConfig.url}?format=csv&template=update&spm=` + spm.spm
      return this.downloadCsv(url)
    },
    async handleDialogConfirm(val) {
      switch (val) {
        case 'export':
          await this.handleExport()
          this.showExportDialog = false
          break
        case 'import':
          await this.handleImport()
          this.showImportDialog = false
          break
        default:
          break
      }
    },
    handleDialogCancel(val) {
      switch (val) {
        case 'export':
          this.showExportDialog = false
          break
        case 'import':
          this.showImportDialog = false
          break
        default:
          break
      }
    }
  }
}
</script>

<style lang="scss" scoped>

  .table-content {
    margin-top: 10px;

    & >>> .el-card__body {
      padding: 0;
    }
    & >>> .el-table__header thead > tr > th {
      background-color: white;
    }

    /*& >>> .el-table--striped .el-table__body tr.el-table__row--striped td {*/
      /*background: white;*/
    /*}*/

    /*& >>> .el-table th, .el-table tr  {*/
      /*background-color: red;*/
      /*!*background-color: #FAFAFA;*!*/
    /*}*/
  }

  //修改颜色
  // .el-button--text{
  //   color: #409EFF;
  // }
</style>
