<template>
  <el-dialog
    :close-on-click-modal="false"
    :title="title"
    :type="type"
    :visible.sync="isVisible"
    :width="width"
    top="50px"
  >
    <el-form ref="form" :model="role" :rules="rules" label-position="right" label-width="100px">
      <el-form-item :label="$t('table.role.code')" prop="code">
        <el-input v-model="role.code" :disabled="type==='edit'" />
      </el-form-item>
      <el-form-item :label="$t('table.role.name')" prop="name">
        <el-input v-model="role.name" />
      </el-form-item>
      <el-form-item :label="$t('table.role.status')" prop="status">
        <el-radio-group v-model="role.status">
          <el-radio-button :label="true">{{ $t('common.status.valid') }}</el-radio-button>
          <el-radio-button :label="false">{{ $t('common.status.invalid') }}</el-radio-button>
        </el-radio-group>
      </el-form-item>
      <el-form-item :label="$t('table.role.describe')" prop="describe">
        <el-input v-model="role.describe" />
      </el-form-item>
	  <!--
      <el-form-item :label="$t('table.role.dsType')" prop="dsType">
        <el-radio-group v-model="role.dsType.code" @change="dsTypeChange">
          <el-radio-button
            v-for="(item, key, index) in enums.DataScopeType"
            :key="index"
            :label="key"
            :value="key"
          >{{ item }}</el-radio-button>
        </el-radio-group>
      </el-form-item>
	  -->
      <el-form-item :label="$t('table.role.orgList')" prop="orgList">
        <el-tree
          ref="orgTree"
          :check-strictly="true"
          :data="orgList"
          :default-checked-keys="role.orgList"
          :default-expanded-keys="role.orgList"
          :expand-on-click-node="false"
          highlight-current
          node-key="id"
          show-checkbox
        />
      </el-form-item>
    </el-form>
    <div slot="footer" class="dialog-footer">
      <el-button plain type="warning" @click="isVisible = false">{{ $t('common.cancel') }}</el-button>
      <el-button plain type="primary" @click="submitForm">{{ $t('common.confirm') }}</el-button>
    </div>
  </el-dialog>
</template>
<script>
import roleApi from '@/api/Role.js'
import orgApi from '@/api/Org.js'

export default {
  name: 'RoleEdit',
  components: {},
  props: {
    dialogVisible: {
      type: Boolean,
      default: false
    },
    type: {
      type: String,
      default: 'add'
    }
  },
  data() {
    return {
      role: this.initRole(),
      screenWidth: 0,
      width: this.initWidth(),
      orgList: [],
      orgHidden: true,
      rules: {
        name: [
          {
            required: true,
            message: this.$t('rules.require'),
            trigger: 'blur'
          },
          {
            min: 1,
            max: 255,
            message: this.$t('rules.range4to10'),
            trigger: 'blur'
          },
          {
            validator: (rule, value, callback) => {
              if (!this.type === 'add' && value.trim().length > 0) {
                roleApi.check(value).then(response => {
                  const res = response.data
                  if (res.data) {
                    callback('编码重复')
                  } else {
                    callback()
                  }
                })
              } else {
                callback()
              }
              callback()
            },
            trigger: 'blur'
          }
        ],
        status: {
          required: true,
          message: this.$t('rules.require'),
          trigger: 'blur'
        },
        orgList: {
          validator: (rule, value, callback) => {
            if (true) {
              if (this.$refs.orgTree.getCheckedKeys().length > 0) {
                callback()
              } else {
                callback('请至少选择一个单位或部门')
              }
            } else {
              callback()
            }
          }
        }
      }
    }
  },
  computed: {
    isVisible: {
      get() {
        return this.dialogVisible
      },
      set() {
        this.close()
        this.reset()
      }
    },
    title() {
      return this.type === 'add'
        ? this.$t('common.add')
        : this.$t('common.edit')
    },
    enums() {
      return this.$store.state.common.enums
    }
  },
  watch: {},
  mounted() {
    this.initOrg()
    window.onresize = () => {
      return (() => {
        this.width = this.initWidth()
      })()
    }
  },
  methods: {
    initRole() {
      return {
        id: '',
        code: '',
        name: '',
        orgList: [],
        status: true,
        describe: '',
        dsType: {
          code: 'ALL',
          desc: ''
        }
      }
    },
    initWidth() {
      this.screenWidth = document.body.clientWidth
      if (this.screenWidth < 991) {
        return '90%'
      } else if (this.screenWidth < 1400) {
        return '45%'
      } else {
        return '800px'
      }
    },
    initOrg() {
      orgApi.allTree({ status: true }).then(response => {
        const res = response.data

        this.orgList = res.data
      })
    },
    loadListOptions({ callback }) {
      callback()
    },
    setRole(val) {
      const vm = this
      if (val) {
        vm.role = { ...val }
        //this.orgHidden = val.dsType.code !== 'CUSTOMIZE'
		this.orgHidden = false
        if (!this.orgHidden) {
          roleApi.get(val.id).then(response => {
            const res = response.data
            if (res.isSuccess) {
              this.role.orgList = res.data.orgList
              this.$refs.orgTree.setCheckedKeys(res.data.orgList)
            }
          })
        }
      }
    },
    close() {
      this.$emit('close')
    },
    reset() {
      // 先清除校验，再清除表单，不然有奇怪的bug
      this.$refs.form.clearValidate()
      this.$refs.form.resetFields()
      this.role = this.initRole()
      this.orgHidden = true
      this.$refs.orgTree.setCheckedKeys([])
    },
    submitForm() {
      const vm = this
      this.$refs.form.validate(valid => {
        if (valid) {
          vm.editSubmit()
        } else {
          return false
        }
      })
    },
    editSubmit() {
      const vm = this
      if (this.orgHidden && this.role.orgList) {
        this.role.orgList.length = 0
      } else {
        this.role.orgList = this.$refs.orgTree.getCheckedKeys()
      }

      if (vm.type === 'add') {
        vm.save()
      } else {
        vm.update()
      }
    },
    save() {
      const vm = this
      roleApi.save(this.role).then(response => {
        const res = response.data
        if (res.isSuccess) {
          vm.isVisible = false
          vm.$message({
            message: vm.$t('tips.createSuccess'),
            type: 'success'
          })
          vm.$emit('success')
        }
      })
    },
    update() {
      roleApi.update(this.role).then(response => {
        const res = response.data
        if (res.isSuccess) {
          this.isVisible = false
          this.$message({
            message: this.$t('tips.updateSuccess'),
            type: 'success'
          })
          this.$emit('success')
        }
      })
    },
    dsTypeChange(value) {
      this.orgHidden = value !== 'CUSTOMIZE'
    }
  }
}
</script>
<style lang="scss" scoped>
</style>
