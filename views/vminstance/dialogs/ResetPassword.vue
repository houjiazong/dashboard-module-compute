<template>
  <base-dialog @cancel="cancelDialog">
    <div slot="header">重置密码</div>
    <div slot="body">
      <dialog-selected-tips :count="params.data.length" action="重置密码" />
      <vxe-grid class="mb-2" :data="params.data" :columns="params.columns.slice(0, 3)" />
      <a-form :form="form.fc">
        <a-form-item label="管理员密码" v-bind="formItemLayout">
          <server-password :decorator="decorators.loginConfig" :login-types="loginTypes" />
        </a-form-item>
        <a-form-item label="自动启动" v-bind="formItemLayout" extra="重置密码成功后是否自动启动">
          <a-switch v-decorator="decorators.auto_start" :disabled="form.fi.disableAutoStart" />
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
import ServerPassword from '@Compute/sections/ServerPassword'
import { LOGIN_TYPES_MAP } from '@Compute/constants'
import DialogMixin from '@/mixins/dialog'
import WindowsMixin from '@/mixins/windows'
import { typeClouds } from '@/utils/common/hypervisor'

const hypervisorMap = typeClouds.hypervisorMap

export default {
  name: 'VmResetPasswordDialog',
  components: {
    ServerPassword,
  },
  mixins: [DialogMixin, WindowsMixin],
  data () {
    let autoStartInitialValue = true
    let disableAutoStart = false
    const noAutoStartHyper = [hypervisorMap.azure.key, hypervisorMap.openstack.key]
    const firstData = this.params.data && this.params.data[0]
    if (firstData && firstData.status === 'running') {
      autoStartInitialValue = false
      disableAutoStart = true
    }
    if (firstData && firstData.hypervisor) {
      const hyper = firstData.hypervisor.toLowerCase()
      if (noAutoStartHyper.includes(hyper)) {
        autoStartInitialValue = false
        disableAutoStart = true
      }
    }
    return {
      loading: false,
      loginTypes: [LOGIN_TYPES_MAP.random.key, LOGIN_TYPES_MAP.password.key],
      form: {
        fc: this.$form.createForm(this),
        fi: {
          disableAutoStart,
        },
      },
      decorators: {
        loginConfig: {
          loginType: [
            'loginType',
            {
              initialValue: LOGIN_TYPES_MAP.random.key,
            },
          ],
        },
        auto_start: [
          'auto_start',
          {
            initialValue: autoStartInitialValue,
            valuePropName: 'checked',
          },
        ],
      },
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
  methods: {
    async handleConfirm () {
      this.loading = true
      try {
        const values = await this.form.fc.validateFields()
        const ids = this.params.data.map(item => item.id)
        const data = { reset_password: true, auto_start: values.auto_start }
        if (values.loginType === LOGIN_TYPES_MAP.password.key) {
          data.password = values.password
        }
        await this.params.list.onManager('batchPerformAction', {
          id: ids,
          steadyStatus: ['running', 'ready'],
          managerArgs: {
            action: 'deploy',
            data,
          },
        })
        this.cancelDialog()
      } finally {
        this.loading = false
      }
    },
  },
}
</script>
