<template>
  <base-dialog @cancel="cancelDialog">
    <div slot="header">新建</div>
    <div slot="body">
      <a-form
        :form="form.fc">
        <a-form-item label="区域" class="mb-0" v-bind="formItemLayout">
          <cloudregion-zone
            :zone-params="par.zone"
            :cloudregion.sync="currentCloudregion"
            :cloudregion-params="par.region"
            :decorator="decorators.regionZone" />
        </a-form-item>
        <a-form-item label="名称" v-bind="formItemLayout">
          <a-input v-decorator="decorators.name" placeholder="字母开头，数字和字母大小写组合，长度为2-128个字符，不含'.','_','@'" />
        </a-form-item>
        <a-form-item :label="storageLabel" v-bind="formItemLayout">
          <a-select v-decorator="decorators.storage_id">
            <a-select-option v-for="item in storageOpts" :key="item.value">
              {{item.label}}
            </a-select-option>
          </a-select>
        </a-form-item>
        <a-form-item label="容量" v-bind="formItemLayout">
          <a-input-number :min="1" :max="100000" v-decorator="decorators.size" /> GB
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
import * as R from 'ramda'
import * as CommonConstants from '../../../constants'
import CloudregionZone from '@/sections/CloudregionZone'
import DialogMixin from '@/mixins/dialog'
import WindowsMixin from '@/mixins/windows'

export default {
  name: 'CreateDialog',
  components: {
    CloudregionZone,
  },
  mixins: [DialogMixin, WindowsMixin],
  data () {
    return {
      loading: false,
      currentCloudregion: {},
      form: {
        fc: this.$form.createForm(this, {
          onValuesChange: (props, values) => {
            if (values.hasOwnProperty('zone')) {
              if (values.zone.key) {
                this.fetchStorageList(values.zone.key)
              }
            }
          },
        }),
      },
      decorators: {
        name: [
          'name',
          {
            rules: [
              { required: true, message: '字母开头，数字和字母大小写组合，长度为2-128个字符，不含".","_","@"' },
              { validator: this.$validate('snapshotName') },
            ],
          },
        ],
        backend: [
          'backend',
        ],
        storage_id: [
          'storage_id',
          {
            rules: [
              { required: true, message: '请选择存储类型' },
            ],
          },
        ],
        size: [
          'size',
          {
            initialValue: 10,
            rules: [
              { required: true },
            ],
          },
        ],
        regionZone: {
          cloudregion: [
            'cloudregion',
            {
              initialValue: { key: '', label: '' },
              rules: [
                { required: true, message: '请选择区域' },
              ],
            },
          ],
          zone: [
            'zone',
            {
              initialValue: { key: '', label: '' },
              rules: [
                { required: true, message: '请选择可用区' },
              ],
            },
          ],
        },
      },
      formItemLayout: {
        wrapperCol: {
          span: 21,
        },
        labelCol: {
          span: 3,
        },
      },
      param: {
        zone: {},
        region: {
          usable: true,
          cloud_env: 'onpremise',
          // scope: 'system',
          project_domain: 'asdasd',
        },
      },
      storageOpts: [],
    }
  },
  computed: {
    provider () { // 向外提供的，通过 refs 获取
      if (this.currentCloudregion && this.currentCloudregion.provider) {
        return this.currentCloudregion.provider.toLowerCase()
      }
      return ['kvm', 'esxi'] // 没有 provider 肯定是 kvm 或者 esxi 的cloudregion
    },
    diskType () {
      return this.params.diskType
    },
    storageLabel () {
      if (['idc', 'private'].includes(this.diskType)) {
        return '存储类型'
      }
      return '介质类型'
    },
    par () {
      if (this.diskType === 'private') {
        return {
          zone: {
            usable: true,
            show_emulated: true,
            order_by: 'created_at',
            order: 'asc',
          },
          region: {
            usable: true,
            cloud_env: 'private',
            show_emulated: true,
          },
        }
      } else if (this.diskType === 'public') {
        return {
          zone: {
            usable: true,
            show_emulated: true,
            order_by: 'created_at',
            order: 'asc',
          },
          region: {
            usable: true,
            cloud_env: 'public',
          },
        }
      }
      return {
        zone: {},
        region: {
          usable: true,
          cloud_env: 'onpremise',
        },
      }
    },
  },
  provide () {
    return {
      form: this.form,
    }
  },
  methods: {
    fetchStorageList (zoneId) {
      const params = {
        usable: true,
        share: true,
        details: true,
        show_emulated: true,
      }
      this.storageOpts = []
      new this.$Manager('storages').list({ ctx: [['zones', zoneId]], params })
        .then(({ data: { data } = { data: [] } }) => {
          try {
            this.storageOpts = this._translateStorageOps(data)
            if (this.storageOpts.length > 0) {
              this.form.fc.setFieldsValue({ storage_id: this.storageOpts[0].value })
            }
          } catch (error) {
            throw new Error('存储类型解析出错：' + error)
          }
        })
    },
    validateForm () {
      return new Promise((resolve, reject) => {
        this.form.fc.validateFields((err, values) => {
          if (!err) {
            resolve(values)
          } else {
            reject(err)
          }
        })
      })
    },
    doCreate (data) {
      return this.params.list.onManager('create', {
        managerArgs: {
          data,
        },
      })
    },
    async handleConfirm () {
      this.loading = true
      try {
        let values = await this.validateForm()
        const storageOpts = R.indexBy(R.prop('value'), this.storageOpts)
        values = {
          ...values,
          size: values.size * 1024,
          backend: storageOpts[values.storage_id].storage_type,
        }
        Reflect.deleteProperty(values, 'cloudregion')
        Reflect.deleteProperty(values, 'zone')
        this.loading = true
        await this.doCreate(values)
        this.loading = false
        this.cancelDialog()
      } catch (error) {
        this.loading = false
      }
    },
    _translateStorageOps (data) {
      const findStorageProvider = optItem => {
        const storageName = optItem.name ? `(${optItem.name})` : ''
        let storageType = optItem.storage_type
        let storageProvider = {}
        if (R.is(Array, this.provider)) {
          this.provider.forEach(hypervisor => { // 将 kvm 和 esxi 的存储类型合一
            Object.assign(storageProvider, CommonConstants.STORAGE_TYPES[hypervisor])
          })
        } else {
          console.log(this.provider)
          console.log(CommonConstants.STORAGE_TYPES)
          storageProvider = CommonConstants.STORAGE_TYPES[this.provider]
        }
        if (storageProvider[storageType.toLowerCase()]) {
          storageType = storageType.toLowerCase()
        } else if (storageProvider[storageType.toUpperCase()]) {
          storageType = storageType.toUpperCase()
        }
        return {
          storageName,
          storageType,
          storageProvider,
        }
      }
      // 过滤掉不支持创建的云硬盘的存储类型
      const conCreateCloud = data.filter(v => {
        const { storageProvider, storageType } = findStorageProvider(v)
        if (storageType && storageProvider && !R.isEmpty(storageProvider) && storageProvider[storageType]) {
          return !storageProvider[storageType].unCreateCloud
        }
        // 说明支持自定义
        if (CommonConstants.CUSTOM_STORAGE_TYPES.includes(this.provider)) {
          return true
        }
        return false
      })
      return conCreateCloud.map(v => {
        const { storageName, storageType, storageProvider } = findStorageProvider(v)
        let label = v.name
        try {
          if (storageProvider[storageType]) {
            label = storageProvider[storageType].label
          } else {
            label = storageProvider[storageType.toLowerCase()].label
          }
        } catch (error) {
          console.warn(`没有找到 ${this.provider} 下面的 ${storageType}`)
        }
        return {
          label: `${label}${storageName}`,
          value: v.id,
          ...v,
        }
      })
    },
  },
}
</script>
