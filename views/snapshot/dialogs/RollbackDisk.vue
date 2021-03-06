<template>
  <base-dialog @cancel="cancelDialog">
    <div slot="header">{{action}}</div>
    <div slot="body">
      <a-alert class="mb-2" type="warning">
        <div slot="message">
          系统盘上该时刻之后的数据将被清除。请谨慎操作! 只有已停止的实例和当前磁盘没有创建中的快照才可以回滚磁盘。
        </div>
      </a-alert>
      <dialog-selected-tips :count="params.data.length" :action="action" />
      <vxe-grid class="mb-2" :data="params.data" :columns="columns" />
      <a-form
        :form="form.fc">
        <a-form-item label="自动启动" v-bind="formItemLayout" extra="回滚硬盘后是否自动启动">
          <a-switch v-decorator="decorators.autoStart" :disabled="disabled" />
        </a-form-item>
      </a-form>
    </div>
    <div slot="footer">
      <a-button type="primary" @click="handleConfirm" :loading="loading">{{ $t('dialog.ok') }}</a-button>
      <a-button @click="cancelDialog">{{ $t('dialog.cancel') }}</a-button>
    </div>
  </base-dialog>
</template>

<script>
import DialogMixin from '@/mixins/dialog'
import WindowsMixin from '@/mixins/windows'

export default {
  name: 'RollbackDiskDialog',
  mixins: [DialogMixin, WindowsMixin],
  data () {
    return {
      loading: false,
      action: '回滚硬盘',
      form: {
        fc: this.$form.createForm(this),
      },
      decorators: {
        autoStart: [
          'autoStart',
          {
            valuePropName: 'checked',
            initialValue: true,
          },
        ],
      },
      disabled: false,
      formItemLayout: {
        wrapperCol: {
          span: 21,
        },
        labelCol: {
          span: 3,
        },
      },
    }
  },
  computed: {
    columns () {
      const showFields = ['name', 'brand', 'disk_type']
      return this.params.columns.filter((item) => { return showFields.includes(item.field) })
    },
  },
  watch: {
    'params.data': {
      immediate: true,
      deep: true,
      handler (val) {
        if (val.some(v => !v.guest)) {
          this.disabled = true
          this.$nextTick(() => {
            this.form.fc.setFieldsValue({ autoStart: false })
          })
        } else {
          this.disabled = false
          this.$nextTick(() => {
            this.form.fc.setFieldsValue({ autoStart: true })
          })
        }
      },
    },
  },
  methods: {
    async doRollbackDiskSubmit (values) {
      const params = {
        snapshot_id: this.params.data[0].id,
        auto_start: values.autoStart,
      }
      const ids = this.params.data.map(item => item.disk_id)
      let manager = new this.$Manager('disks')
      return manager.performAction({
        id: ids[0],
        action: 'disk-reset',
        data: params,
      })
    },
    async handleConfirm () {
      this.loading = true
      try {
        const values = await this.form.fc.getFieldsValue()
        await this.doRollbackDiskSubmit(values)
        this.params.list.refresh()
        this.loading = false
        this.cancelDialog()
      } catch (error) {
        this.loading = false
      }
    },
  },
}
</script>
