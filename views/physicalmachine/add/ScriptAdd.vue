<template>
  <a-form-item :wrapper-col="offsetWrapperCol">
    <div class="script-add-wrap">
      <template v-if="loading"><loader loading /></template>
      <template v-else>
        <a-alert message="提示：在一台或多台已安装系统的物理机中运行以下命令" banner />
        <code-mirror :value="value" :options="cmOptions" />
        <div class="text-right">
          <a-button
            type="link"
            icon="copy"
            class="pl-0 pr-0"
            @click="handleCopy">点击复制</a-button>
        </div>
      </template>
    </div>
  </a-form-item>
</template>

<script>
import 'codemirror/mode/shell/shell'

export default {
  name: 'PhysicalmachineScriptAdd',
  props: {
    offsetWrapperCol: {
      type: Object,
      required: true,
    },
  },
  data () {
    return {
      loading: false,
      value: '',
      cmOptions: {
        styleActiveLine: true,
        lineNumbers: true,
        lineWrapping: true,
        line: true,
        readOnly: true,
        theme: 'material',
      },
    }
  },
  destroyed () {
    this.manager = null
  },
  created () {
    this.manager = new this.$Manager('hosts')
    this.fetchScript()
  },
  methods: {
    async fetchScript () {
      this.loading = true
      try {
        const response = await this.manager.get({ id: 'bm-start-register-script' })
        this.value = `sudo sh -c "$(${response.data.script})"`
      } finally {
        this.loading = false
      }
    },
    handleCopy () {
      this.$copyText(this.value).then(() => {
        this.$message.success('复制成功')
      }).catch(() => {
        this.$message.error('复制失败')
      })
    },
  },
}
</script>

<style lang="scss" scoped>
.script-add-wrap {
  line-height: 1.5rem;
  max-width: 600px;
  ::v-deep {
    .CodeMirror {
      height: auto !important;
    }
  }
}
</style>
