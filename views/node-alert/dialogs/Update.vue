<template>
  <base-dialog @cancel="cancelDialog">
    <div slot="header">更新报警</div>
    <div slot="body">
      <vxe-grid class="mb-2" v-if="params.data && params.columns" :data="params.data" :columns="params.columns.slice(0)" />
      <node-alert-form
        ref="nodeAlertFormRef"
        :metric-opts="params.metricOpts"
        :hypervisor="params.hypervisor"
        :alertType="params.alertType"
        :fd-initail-value="fdInitailValue" />
    </div>
    <div slot="footer">
      <a-button type="primary" @click="handleConfirm" :loading="loading">{{ $t('dialog.ok') }}</a-button>
      <a-button @click="cancelDialog">{{ $t('dialog.cancel') }}</a-button>
    </div>
  </base-dialog>
</template>

<script>
import NodeAlertForm from '../components/Form'
import DialogMixin from '@/mixins/dialog'
import WindowsMixin from '@/mixins/windows'

export default {
  name: 'UpdateNodeAlert',
  components: {
    NodeAlertForm,
  },
  mixins: [DialogMixin, WindowsMixin],
  data () {
    return {
      loading: false,
      fdInitailValue: {
        ...this.params.data[0],
        recipients: this.params.data[0].recipients.split(','),
        window: +this.params.data[0].window.replace('m', ''),
      },
    }
  },
  methods: {
    async handleConfirm () {
      this.loading = true
      try {
        const values = await this.$refs.nodeAlertFormRef.validateForm()
        this.loading = false
        const recipients = values.recipients.join(',')
        const data = {
          metric: values.metric,
          window: `${values.window}m`,
          comparator: values.comparator,
          threshold: values.threshold + '',
          level: values.level,
          channel: values.channel,
          node_id: this.params.nodeId,
          type: this.params.alertType,
          recipients,
        }
        const id = this.params.data[0].id
        if (this.params.list) {
          await this.params.list.onManager('patch', { id, managerArgs: { data } })
        } else {
          await this.params.alertManager.patch({
            id,
            data,
          })
        }
        this.cancelDialog()
        this.$message.success('操作成功')
        this.params.list && this.params.list.refresh()
      } catch (error) {
        this.loading = false
        throw error
      }
    },
  },
}
</script>
