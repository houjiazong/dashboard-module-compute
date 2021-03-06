<template>
  <base-dialog @cancel="cancelDialog">
    <div slot="header">{{ action }}</div>
    <div slot="body">
      <a-alert class="mb-2" type="warning">
        <template v-slot:message>
          <div>新建虚拟机时加入主机组，将按照主机组内规则调度选择宿主机。</div>
          <div class="mt-2">已创建的虚拟机加入主机组后虚拟机所属宿主机不会变化。</div>
        </template>
      </a-alert>
      <dialog-selected-tips :count="params.data.length" :action="action" name="主机组" />
      <vxe-grid class="mb-2" :data="params.data" :columns="params.columns.slice(0, 3)" />
      <a-select
        v-if="instanceGroupsLoaded"
        class="w-100"
        mode="multiple"
        placeholder="请选择要绑定的主机"
        :defaultValue="defaultSelected"
        :loading="instanceGroupsLoading"
        @search="debounceFetchInstanceGroups"
        @change="handleSelectChange">
        <a-select-option
          v-for="item of instanceGroups"
          :key="item.id">{{ item.name }}</a-select-option>
      </a-select>
    </div>
    <div slot="footer">
      <a-button type="primary" @click="handleConfirm" :loading="loading">{{ $t('dialog.ok') }}</a-button>
      <a-button @click="cancelDialog">{{ $t('dialog.cancel') }}</a-button>
    </div>
  </base-dialog>
</template>

<script>
import debounce from 'lodash/debounce'
import { mapGetters } from 'vuex'
import DialogMixin from '@/mixins/dialog'
import WindowsMixin from '@/mixins/windows'

export default {
  name: 'VmBindInstanceGroupDialog',
  mixins: [DialogMixin, WindowsMixin],
  data () {
    return {
      loading: false,
      action: '加入主机组',
      // 获取的主机列表
      instanceGroups: [],
      instanceGroupsLoading: false,
      // 主机列表是否已加载
      instanceGroupsLoaded: false,
      // 已选择的主机
      selectedInstanceGroups: [],
      // 已绑定的主机
      bindedInstanceGroups: [],
      // 默认选择
      defaultSelected: [],
    }
  },
  computed: mapGetters(['scope']),
  created () {
    this.instanceGroupsManager = new this.$Manager('instancegroups', 'v2')
    this.fetchBindedInstanceGroups()
    this.debounceFetchInstanceGroups = debounce(this.fetchInstanceGroups, 300)
  },
  methods: {
    async fetchBindedInstanceGroups () {
      try {
        const { data: { data = [] } } = await this.instanceGroupsManager.list({
          params: {
            scope: this.scope,
            server: this.params.data[0]['id'],
          },
        })
        this.bindedInstanceGroups = data
        this.defaultSelected = data.map(item => item.id)
        this.fetchInstanceGroups()
      } catch (error) {
        throw error
      }
    },
    async fetchInstanceGroups (query) {
      this.instanceGroupsLoading = true
      const params = {
        enabled: true,
        scope: this.scope,
        project: this.params.data[0]['tenant_id'],
      }
      if (query) params.filter = `name.contains("${query}")`
      try {
        const { data: { data = [] } } = await this.instanceGroupsManager.list({
          params,
        })
        this.instanceGroups = data
        this.instanceGroupsLoaded = true
      } finally {
        this.instanceGroupsLoading = false
      }
    },
    doUpdateBindInstanceGroups (action, ids) {
      const data = {}
      for (let i = 0, len = ids.length; i < len; i++) {
        data[`group.${i}`] = ids[i]
      }
      return this.params.list.onManager('performAction', {
        id: this.params.data[0]['id'],
        managerArgs: {
          action,
          data,
        },
      })
    },
    async handleConfirm () {
      this.loading = true
      const unbindIds = this.defaultSelected.filter(item => {
        return !this.selectedInstanceGroups.includes(item)
      }).filter(item => !!item)
      const bindIds = this.selectedInstanceGroups.filter(item => {
        return !this.defaultSelected.includes(item)
      }).filter(item => !!item)
      try {
        if (unbindIds && unbindIds.length) {
          await this.doUpdateBindInstanceGroups('unbind-groups', unbindIds)
        }
        if (bindIds && bindIds.length) {
          await this.doUpdateBindInstanceGroups('bind-groups', bindIds)
        }
        this.params.list.refresh()
        this.cancelDialog()
      } finally {
        this.loading = false
      }
    },
    handleSelectChange (val) {
      this.selectedInstanceGroups = val
    },
  },
}
</script>
