<template>
  <div class="app-container">
    <div class="filter-container">
      <el-input
        v-model="queryParams.submittedFileName"
        :placeholder="$t('table.attachment.submittedFileName')"
        class="filter-item search-item"
      />
      <el-date-picker
        v-model="queryParams.timeRange"
        :range-separator="null"
        class="filter-item search-item date-range-item"
        end-placeholder="结束日期"
        format="yyyy-MM-dd HH:mm:ss"
        start-placeholder="开始日期"
        type="daterange"
        value-format="yyyy-MM-dd HH:mm:ss"
      />
      <el-button class="filter-item" plain type="primary" @click="search">{{ $t("table.search") }}</el-button>
      <el-button class="filter-item" plain type="warning" @click="reset">{{ $t("table.reset") }}</el-button>
      <el-dropdown
        v-has-any-permission="['file:add', 'file:delete', 'file:download']"
        class="filter-item"
        trigger="click"
      >
        <el-button>
          {{ $t("table.more") }}
          <i class="el-icon-arrow-down el-icon--right" />
        </el-button>
        <el-dropdown-menu slot="dropdown">
          <el-dropdown-item
            v-has-permission="['file:add']"
            @click.native="upload"
          >{{ $t("table.upload") }}</el-dropdown-item>
          <el-dropdown-item
            v-has-permission="['file:delete']"
            @click.native="batchDelete"
          >{{ $t("table.delete") }}</el-dropdown-item>
          <el-dropdown-item
            v-has-permission="['file:download']"
            @click.native="batchDownload"
          >{{ $t("table.download") }}</el-dropdown-item>
        </el-dropdown-menu>
      </el-dropdown>
    </div>

    <el-table
      :key="tableKey"
      ref="table"
      v-loading="loading"
      :data="tableData.records"
      border
      fit
      style="width: 100%;"
      @selection-change="onSelectChange"
      @sort-change="sortChange"
    >
      <el-table-column align="center" type="selection" width="40px" />
      <el-table-column
        :label="$t('table.attachment.submittedFileName')"
        :show-overflow-tooltip="true"
        align="left"
        prop="submittedFileName"
      >
        <template slot-scope="scope">
          <div @click="onView(scope.row)">
            <i :class="scope.row.icon" class="button-list" />
            <span>{{ scope.row.submittedFileName }}</span>
          </div>
        </template>
      </el-table-column>
      <el-table-column
        :label="$t('table.attachment.dataType')"
        :show-overflow-tooltip="true"
        align="center"
        prop="dataType"
        width="100px"
      >
        <template slot-scope="scope">
          <span>{{ scope.row.dataType.desc }}</span>
        </template>
      </el-table-column>
      <el-table-column
        :label="$t('table.attachment.bizType')"
        :show-overflow-tooltip="true"
        align="center"
        width="120px"
      >
        <template slot-scope="scope">
          <span>{{ scope.row.bizType }}</span>
        </template>
      </el-table-column>
      <el-table-column
        :label="$t('table.attachment.bizId')"
        :show-overflow-tooltip="true"
        align="center"
        width="180px"
      >
        <template slot-scope="scope">
          <span>{{ scope.row.bizId }}</span>
        </template>
      </el-table-column>
      <el-table-column
        :label="$t('table.attachment.size')"
        align="center"
        :show-overflow-tooltip="true"
        width="120px"
      >
        <template slot-scope="scope">
          <span>{{ formatSize(scope.row) }}</span>
        </template>
      </el-table-column>
      <el-table-column
        :label="$t('table.createTime')"
        align="center"
        prop="createTime"
        sortable="custom"
        width="170px"
      >
        <template slot-scope="scope">
          <span>{{ scope.row.createTime }}</span>
        </template>
      </el-table-column>
      <el-table-column
        :label="$t('table.operation')"
        align="center"
        class-name="small-padding fixed-width"
        width="100px"
      >
        <template slot-scope="{ row }">
          <i
            v-hasPermission="['file:download']"
            class="el-icon-download table-operation"
            style="color: #f50;"
            @click="singleDownload(row)"
          />
          <i
            v-hasPermission="['file:delete']"
            class="el-icon-delete table-operation"
            style="color: #f50;"
            @click="singleDelete(row)"
          />
          <el-link
            v-has-no-permission="['file:update', 'file:delete']"
            class="no-perm"
          >{{ $t("tips.noPermission") }}</el-link>
        </template>
      </el-table-column>
    </el-table>
    <pagination
      v-show="tableData.total > 0"
      :limit.sync="pagination.size"
      :page.sync="pagination.current"
      :total="Number(tableData.total)"
      @pagination="fetch"
    />
    <attachment-edit
      ref="edit"
      :dialog-visible="dialog.isVisible"
      :type="dialog.type"
      @close="editClose"
      @success="editSuccess"
    />
    <el-dialog :visible.sync="dialogVisible">
      <img
        v-if="dialogImageUrl"
        :key="dialogImageUrl"
        :src="dialogImageUrl"
        alt="图片飞到火星了"
        height="500px"
        width="100%"
        @error="errorImage()"
      />
      <span v-else>图片飞到火星了~</span>
    </el-dialog>
  </div>
</template>

<script>
import Pagination from '@/components/Pagination'
import AttachmentEdit from './Edit'
import attachmentApi from '@/api/Attachment.js'
import { renderSize } from '@/utils/utils'
import { onlinePreview } from '@/settings'

export default {
  name: 'AttachmentManage',
  components: { Pagination, AttachmentEdit },
  filters: {},
  data() {
    return {
      dialogVisible: false,
      dialogImageUrl: '',
      dialog: {
        isVisible: false,
        type: 'add'
      },
      tableKey: 0,
      queryParams: {},
      sort: {},
      selection: [],
      // 以下已修改
      loading: false,
      tableData: {
        records: [],
        total: 0
      },
      pagination: {
        size: 10,
        current: 1
      }
    }
  },
  computed: {},
  mounted() {
    this.fetch()
  },
  methods: {
    errorImage() {
      let img = event.srcElement
      img.src = require('@/assets/404_images/404_image.jpeg')
      img.onerror = null //防止闪图
    },
    formatSize(row) {
      return renderSize(row.size)
    },
    filterStatus(value, row) {
      return row.status === value
    },
    editClose() {
      this.dialog.isVisible = false
    },
    editSuccess() {
      this.search()
    },
    onSelectChange(selection) {
      this.selection = selection
    },
    search() {
      this.fetch({
        ...this.queryParams,
        ...this.sort
      })
    },
    reset() {
      this.queryParams = {}
      this.sort = {}
      this.$refs.table.clearSort()
      this.$refs.table.clearFilter()
      this.search()
    },
    singleDownload(row) {
      this.$refs.table.toggleRowSelection(row, true)
      this.batchDownload()
    },
    singleDelete(row) {
      this.$refs.table.toggleRowSelection(row, true)
      this.batchDelete()
    },
    batchDownload() {
      if (!this.selection.length) {
        this.$message({
          message: this.$t('tips.noDataSelected'),
          type: 'warning'
        })
        return
      }
      this.$confirm('确认下载吗？', this.$t('common.tips'), {
        confirmButtonText: this.$t('common.confirm'),
        cancelButtonText: this.$t('common.cancel'),
        type: 'warning'
      })
        .then(() => {
          const ids = []
          this.selection.forEach(u => {
            ids.push(u.id)
          })
          this.download(ids)
        })
        .catch(() => {
          this.clearSelections()
        })
    },
    batchDelete() {
      if (!this.selection.length) {
        this.$message({
          message: this.$t('tips.noDataSelected'),
          type: 'warning'
        })
        return
      }
      this.$confirm(this.$t('tips.confirmDelete'), this.$t('common.tips'), {
        confirmButtonText: this.$t('common.confirm'),
        cancelButtonText: this.$t('common.cancel'),
        type: 'warning'
      })
        .then(() => {
          const ids = []
          this.selection.forEach(u => {
            ids.push(u.id)
          })
          this.delete(ids)
        })
        .catch(() => {
          this.clearSelections()
        })
    },
    clearSelections() {
      this.$refs.table.clearSelection()
    },
    download(ids) {
      attachmentApi.download({ ids: ids }).then(response => {
        const res = response.data
        const type = res.type
        if (type.includes('application/json')) {
          let reader = new FileReader()
          reader.onload = e => {
            if (e.target.readyState === 2) {
              let data = JSON.parse(e.target.result)
              this.$message({
                message: data.msg,
                type: 'warning'
              })
            }
          }
          reader.readAsText(res)
        } else {
          let disposition = response.headers['content-disposition']
          let fileName = '下载文件.zip'
          if (disposition) {
            let respcds = disposition.split(';')
            for (let i = 0; i < respcds.length; i++) {
              let header = respcds[i]
              if (header !== null && header !== '') {
                let headerValue = header.split('=')
                if (headerValue !== null && headerValue.length > 0) {
                  if (headerValue[0].toLowerCase() === 'filename') {
                    fileName = decodeURI(headerValue[1])
                    break
                  }
                }
              }
            }
          }
          let blob = new Blob([res])
          let link = document.createElement('a')
          link.href = window.URL.createObjectURL(blob)
          link.download = fileName
          link.click()
          window.URL.revokeObjectURL(link.href)
        }

        this.clearSelections()
      })
    },
    delete(ids) {
      attachmentApi.delete({ ids: ids }).then(response => {
        const res = response.data
        if (res.isSuccess) {
          this.$message({
            message: this.$t('tips.deleteSuccess'),
            type: 'success'
          })
        }
        this.search()
      })
    },
    onView(row) {
      if (row.url) {
        window.open(onlinePreview + encodeURIComponent(row.url))
      }
    },
    upload() {
      this.dialog.type = 'upload'
      this.dialog.isVisible = true
      this.$refs.edit.setAttachment(false)
    },
    edit(row) {
      this.$refs.edit.setAttachment(row)
      this.dialog.type = 'edit'
      this.dialog.isVisible = true
    },
    fetch(params = {}) {
      this.loading = true
      params.size = this.pagination.size
      params.current = this.pagination.current
      if (this.queryParams.timeRange) {
        params.startCreateTime = this.queryParams.timeRange[0]
        params.endCreateTime = this.queryParams.timeRange[1]
      }
      attachmentApi
        .page(params)
        .then(response => {
          const res = response.data
          this.loading = false
          if (res.isSuccess) {
            this.tableData = res.data
          }
        })
        .catch(() => {
          this.loading = false
        })
    },
    sortChange(val) {
      this.sort.field = val.prop
      this.sort.order = val.order
      this.search()
    }
  }
}
</script>
<style lang="scss" scoped>
.file-breadcrumb {
  margin: 10px 0px 20px;
}
.page {
  text-align: center;
  margin-top: 5px;
}
.button-list {
  margin-right: 10px;
  font-size: 20px !important;
  color: #22a2ed;
  vertical-align: middle;
}
.title {
  color: #777;
  font-size: 2em;
  border-bottom: 1px solid #e5e5e5;
}
</style>
