<template>
  <page-list
    :list="list"
    :columns="columns"
    :group-actions="groupActions"
    :single-actions="singleActions"
    :export-data-options="exportDataOptions" />
</template>

<script>
import { sizestr } from '@/utils/utils'
import { getStatusTableColumn, getNameDescriptionTableColumn, getProjectTableColumn, isPublicTableColumn, getTimeTableColumn } from '@/utils/common/tableColumn'
import { getTenantFilter, getStatusFilter } from '@/utils/common/tableFilter'
import SystemIcon from '@/sections/SystemIcon'
import BaseDropList from '@/sections/DropList'
import WindowsMixin from '@/mixins/windows'

export default {
  name: 'HostImageList',
  mixins: [WindowsMixin],
  props: {
    id: String,
    getParams: {
      type: Object,
      default: () => ({}),
    },
  },
  data () {
    const validateAction = function (obj) {
      if (obj.is_guest_image === true || obj.is_guest_image === 'true') {
        return false
      }
      return true
    }

    const validateActionTooltip = function (obj) {
      if (obj.is_guest_image === true || obj.is_guest_image === 'true') {
        return '主机镜像的子镜像无法操作'
      }
      return ''
    }

    const ownerDomain = list => this.$store.getters.isAdminMode || this.list.selectedItems.every(obj => obj.domain_id === this.$store.getters.userInfo.projectDomainId)

    const isOwnerProject = project => project === this.$store.getters.userInfo.projectId || project === this.$store.getters.userInfo.projectName

    return {
      isAdminMode: this.$store.getters.isAdminMode,
      isDomainMode: this.$store.getters.isDomainMode,
      userInfo: this.$store.getters.userInfo,
      list: this.$list.createList(this, {
        id: this.id,
        resource: 'guestimages',
        apiVersion: 'v1',
        getParams: this.getParam,
        filterOptions: {
          name: {
            label: '名称',
            filter: true,
            formatter: val => {
              return `name.contains("${val}")`
            },
          },
          status: getStatusFilter('image'),
          // disk_format: {
          //   label: '镜像格式',
          //   dropdown: true,
          //   items: [
          //     { label: 'VMDK', key: 'vmdk' },
          //     { label: 'RAW', key: 'raw' },
          //     { label: 'VHD', key: 'vhd' },
          //     { label: 'QCOW2', key: 'qcow2' },
          //     { label: 'ISO', key: 'iso' },
          //   ],
          // },
          tenant: getTenantFilter(),
          // os_type: getOsTypeFilter(),
        },
      }),
      exportDataOptions: {
        items: [
          { label: 'ID', key: 'id' },
          { label: '名称', key: 'name' },
          { label: '格式', key: 'disk_format' },
          { label: '镜像大小', key: 'size' },
          { label: '项目', key: 'tenant' },
          { label: '镜像类型', key: 'is_standard' },
          { label: '创建时间', key: 'created_at' },
        ],
      },
      columns: [
        getNameDescriptionTableColumn({
          width: 200,
          addLock: true,
          vm: this,
          hideField: true,
          slotCallback: row => {
            return (
              <side-page-trigger onTrigger={ () => this.sidePageTriggerHandle(row.id, 'SystemImageSidePage') }>{ row.name }</side-page-trigger>
            )
          },
        }),
        {
          field: 'child_image',
          title: '子镜像',
          width: 150,
          slots: {
            default: ({ row }) => {
              const list = row.data_images.map(val => ({ value: val.name }))
              list.push({ value: row.root_image.name })
              const dropMsg = [
                {
                  title: `${list.length}个`,
                  list,
                },
              ]
              return [<BaseDropList drop-msg={dropMsg} />]
            },
          },
        },
        {
          field: 'disk_format',
          title: '格式',
          width: 100,
          formatter: ({ cellValue }) => {
            return cellValue && cellValue.toUpperCase()
          },
        },
        {
          field: 'os_type',
          title: '系统',
          width: 60,
          slots: {
            default: ({ row }) => {
              if (!row.properties) return
              let name = row.properties.os_distribution ? decodeURI(row.properties.os_distribution) : row.properties.os_type || ''
              if (name.includes('Windows') || name.includes('windows')) {
                name = 'Windows'
              }
              const tooltip = (row.properties.os_version ? `${name} ${row.properties.os_version}` : name) || '未知'
              return [
                <SystemIcon tooltip={ tooltip } name={ name } />,
              ]
            },
          },
        },
        {
          field: 'size',
          title: '镜像大小',
          minWidth: 100,
          formatter: ({ cellValue }) => {
            return sizestr(cellValue, 'B', 1024)
          },
        },
        getStatusTableColumn({ statusModule: 'image' }),
        getProjectTableColumn(),
        isPublicTableColumn(),
        {
          field: 'is_standard',
          title: '镜像类型',
          width: 100,
          formatter: ({ cellValue }) => {
            if (cellValue) return '公共镜像'
            return '自定义镜像'
          },
        },
        getTimeTableColumn(),
      ],
      groupActions: [
        {
          label: '设置删除保护',
          action: (row) => {
            this.createDialog('ChangeDisableDelete', {
              name: '主机镜像',
              columns: this.columns,
              list: this.list,
              data: this.list.selectedItems,
            })
          },
          meta: () => {
            const validate = this.list.selectedItems.length > 0
            return {
              validate: validate,
              tooltip: !validate && `请选择需要操作的主机镜像`,
            }
          },
        },
        {
          label: '删除',
          permission: 'images_delete',
          action: () => {
            this.createDialog('DeleteResDialog', {
              data: this.list.selectedItems,
              columns: this.columns,
              title: '删除镜像',
              list: this.list,
            })
          },
          meta: () => this.$getDeleteResult(this.list.selectedItems),
        },
      ],
      singleActions: [
        {
          label: '修改属性',
          permission: 'images_update',
          action: obj => {
            this.createDialog('ImageEditAttributesDialog', {
              data: [obj],
              columns: this.columns,
              list: this.list,
            })
          },
          meta: obj => {
            if (!validateAction(obj)) {
              return {
                validate: false,
                tooltip: validateActionTooltip(obj),
              }
            }
            if (this.isAdminMode) {
              return {
                validate: true,
              }
            } else if (this.isDomainMode) {
              return {
                validate: ownerDomain(this.list),
                tooltip: '非当前域下面的镜像无法修改属性',
              }
            }
            return {
              validate: isOwnerProject(obj.tenant_id),
              tooltip: !isOwnerProject(obj.tenant_id) ? '非当前项目下面的镜像无法修改属性' : '',
            }
          },
        },
        {
          label: '更多',
          actions: obj => {
            return [
              {
                label: '设置共享',
                permission: 'images_perform_public',
                action: () => {
                  this.createDialog('SetPublicDialog', {
                    data: [obj],
                    columns: this.columns,
                    list: this.list,
                  })
                },
                meta: () => {
                  function validate (val, tooltip = validateActionTooltip(obj)) {
                    return {
                      validate: val,
                      tooltip,
                    }
                  }
                  if (obj.is_standard) validate(false, '公共镜像不支持设置')
                  if (!validateAction(obj)) validate(false)
                  // 1、管理后台视图可以对所有镜像进行操作；
                  // 2、域管理后台视图只能对该域下的镜像进行操作，不能对其他域共享的镜像进行操作；
                  // 3、项目视图只能对该项目下的镜像进行操作，不能对其他域、其他项目共享的镜像进行操作。
                  if (this.isAdminMode) validate(true)
                  if (!this.isAdminMode && !this.isDomainAdmin) validate(this.userInfo.projectId === obj.tenant_id)
                  if (this.isDomainMode) validate(this.userInfo.projectDomainId === obj.domain_id)
                  return {
                    validate: true,
                  }
                },
              },
              {
                label: '设置删除保护',
                action: (row) => {
                  this.createDialog('ChangeDisableDelete', {
                    name: '系统镜像',
                    columns: this.columns,
                    list: this.list,
                    data: [row],
                  })
                },
              },
              {
                label: '删除',
                permission: 'images_delete',
                action: () => {
                  this.createDialog('DeleteResDialog', {
                    data: [obj],
                    columns: this.columns,
                    title: '删除',
                    list: this.list,
                  })
                },
                meta: () => {
                  if (this.isDomainAdmin && obj.domain_id !== this.userInfo.projectDomainId) {
                    return {
                      validate: false,
                      tooltip: '域管理员只能删除本域下的镜像',
                    }
                  }
                  if (!validateAction(obj)) return { validate: false, tooltip: validateActionTooltip(obj) }
                  return this.$getDeleteResult(obj)
                },
              },
            ]
          },
        },
      ],
    }
  },
  created () {
    this.initSidePageTab('system-image-detail')
    this.list.fetchData()
  },
  methods: {
    getParam () {
      const ret = {
        details: true,
      }
      if (this.cloudEnv) ret.cloud_env = this.cloudEnv
      return ret
    },
    updateStandard (isStandard, selectedItems) {
      let params = {
        is_standard: isStandard,
      }
      if (selectedItems.length > 1) {
        this.list.batchPerformAction('mark-standard', params, this.list.steadyStatus)
      } else {
        params.id = selectedItems[0].id
        this.list.singlePerformAction('mark-standard', params)
      }
    },
  },
}
</script>
