<template>
  <div class="container" center>
    <el-alert
      center type="success"
    >
      <p>Account ID: {{ accountId }}</p>
      <p>User Name: {{ userName }}</p>
      <p>Balance: ${{ balance }}</p>
      <p>When a sale, donation or pledge operation is initiated, the guarantee status is TRUE</p>
      <p>The sale, donation or pledge operation can be initiated only when the guarantee is FALSE</p>
    </el-alert>
    <div v-if="realEstateList.length==0" style="text-align: center;">
      <el-alert center
        title="No Data Yet"
        type="warning"
      />
    </div>
    <el-row v-loading="loading" :gutter="20">
      <el-col v-for="(val,index) in realEstateList" :key="index" :span="6" :offset="1">
        <el-card class="realEstate-card">
          <div slot="header" class="clearfix">
            Guarantee status:
            <span style="color: rgb(255, 0, 0);">{{ val.encumbrance }}</span>
          </div>

          <div class="item">
            <el-tag>Real Estate ID: </el-tag>
            <span>{{ val.realEstateId }}</span>
          </div>
          <div class="item">
            <el-tag type="success">Owner ID: </el-tag>
            <span>{{ val.proprietor }}</span>
          </div>
          <div class="item">
            <el-tag type="warning">Total Space: </el-tag>
            <span>{{ val.totalArea }} ㎡</span>
          </div>
          <div class="item">
            <el-tag type="danger">Living Space: </el-tag>
            <span>{{ val.livingSpace }} ㎡</span>
          </div>

          <div v-if="!val.encumbrance&&roles[0] !== 'admin'">
            <el-button type="text" @click="openDialog(val)">Sell</el-button>
            <el-divider direction="vertical" />
            <el-button type="text" @click="openDonatingDialog(val)">Donate</el-button>
          </div>
          <el-rate v-if="roles[0] === 'admin'" />
        </el-card>
      </el-col>
    </el-row>
    <el-dialog v-loading="loadingDialog" :visible.sync="dialogCreateSelling" :close-on-click-modal="false" @close="resetForm('realForm')">
      <el-form ref="realForm" :model="realForm" :rules="rules" label-width="100px">
        <el-form-item label="Price/$" prop="price">
          <el-input-number v-model="realForm.price" :precision="2" :step="10000" :min="0" />
        </el-form-item>
        <el-form-item label="Validity Period/day" prop="salePeriod">
          <el-input-number v-model="realForm.salePeriod" :min="1" />
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button type="primary" @click="createSelling('realForm')">Sell Now</el-button>
        <el-button @click="dialogCreateSelling = false">Cancel</el-button>
      </div>
    </el-dialog>
    <el-dialog v-loading="loadingDialog" :visible.sync="dialogCreateDonating" :close-on-click-modal="false" @close="resetForm('DonatingForm')">
      <el-form ref="DonatingForm" :model="DonatingForm" :rules="rulesDonating" label-width="100px">
        <el-form-item label="Owner" prop="proprietor">
          <el-select v-model="DonatingForm.proprietor" placeholder="Please Select Owner" @change="selectGet">
            <el-option
              v-for="item in accountList"
              :key="item.accountId"
              :label="item.userName"
              :value="item.accountId"
            >
              <span style="float: left">{{ item.userName }}</span>
              <span style="float: right; color: #8492a6; font-size: 13px">{{ item.accountId }}</span>
            </el-option>
          </el-select>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button type="primary" @click="createDonating('DonatingForm')">Donate Now</el-button>
        <el-button @click="dialogCreateDonating = false">Cancel</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import { mapGetters } from 'vuex'
import { queryAccountList } from '@/api/account'
import { queryRealEstateList } from '@/api/realEstate'
import { createSelling } from '@/api/selling'
import { createDonating } from '@/api/donating'

export default {
  name: 'RealeState',
  data() {
    var checkArea = (rule, value, callback) => {
      if (value <= 0) {
        callback(new Error('Must be greater than 0'))
      } else {
        callback()
      }
    }
    return {
      loading: true,
      loadingDialog: false,
      realEstateList: [],
      dialogCreateSelling: false,
      dialogCreateDonating: false,
      realForm: {
        price: 0,
        salePeriod: 0
      },
      rules: {
        price: [
          { validator: checkArea, trigger: 'blur' }
        ],
        salePeriod: [
          { validator: checkArea, trigger: 'blur' }
        ]
      },
      DonatingForm: {
        proprietor: ''
      },
      rulesDonating: {
        proprietor: [
          { required: true, message: 'Please select Owner', trigger: 'change' }
        ]
      },
      accountList: [],
      valItem: {}
    }
  },
  computed: {
    ...mapGetters([
      'accountId',
      'roles',
      'userName',
      'balance'
    ])
  },
  created() {
    if (this.roles[0] === 'admin') {
      queryRealEstateList().then(response => {
        if (response !== null) {
          this.realEstateList = response
        }
        this.loading = false
      }).catch(_ => {
        this.loading = false
      })
    } else {
      queryRealEstateList({ proprietor: this.accountId }).then(response => {
        if (response !== null) {
          this.realEstateList = response
        }
        this.loading = false
      }).catch(_ => {
        this.loading = false
      })
    }
  },
  methods: {
    openDialog(item) {
      this.dialogCreateSelling = true
      this.valItem = item
    },
    openDonatingDialog(item) {
      this.dialogCreateDonating = true
      this.valItem = item
      queryAccountList().then(response => {
        if (response !== null) {
          // 过滤掉管理员和当前用户
          this.accountList = response.filter(item =>
            item.userName !== 'Administrator' && item.accountId !== this.accountId
          )
        }
      })
    },
    createSelling(formName) {
      this.$refs[formName].validate((valid) => {
        if (valid) {
          this.$confirm('Do you want to sell now?', 'Reminder', {
            confirmButtonText: 'Confirm',
            cancelButtonText: 'Cancel',
            type: 'success'
          }).then(() => {
            this.loadingDialog = true
            createSelling({
              objectOfSale: this.valItem.realEstateId,
              seller: this.valItem.proprietor,
              price: this.realForm.price,
              salePeriod: this.realForm.salePeriod
            }).then(response => {
              this.loadingDialog = false
              this.dialogCreateSelling = false
              if (response !== null) {
                this.$message({
                  type: 'success',
                  message: 'Succeeded!'
                })
              } else {
                this.$message({
                  type: 'error',
                  message: 'Failed!'
                })
              }
              setTimeout(() => {
                window.location.reload()
              }, 1000)
            }).catch(_ => {
              this.loadingDialog = false
              this.dialogCreateSelling = false
            })
          }).catch(() => {
            this.loadingDialog = false
            this.dialogCreateSelling = false
            this.$message({
              type: 'info',
              message: 'Cancelled'
            })
          })
        } else {
          return false
        }
      })
    },
    createDonating(formName) {
      this.$refs[formName].validate((valid) => {
        if (valid) {
          this.$confirm('Do you want to donate now?', 'Reminder', {
            confirmButtonText: 'Confirm',
            cancelButtonText: 'Cancel',
            type: 'success'
          }).then(() => {
            this.loadingDialog = true
            createDonating({
              objectOfDonating: this.valItem.realEstateId,
              donor: this.valItem.proprietor,
              grantee: this.DonatingForm.proprietor
            }).then(response => {
              this.loadingDialog = false
              this.dialogCreateDonating = false
              if (response !== null) {
                this.$message({
                  type: 'success',
                  message: 'Succeeded!'
                })
              } else {
                this.$message({
                  type: 'error',
                  message: 'Failed!'
                })
              }
              setTimeout(() => {
                window.location.reload()
              }, 1000)
            }).catch(_ => {
              this.loadingDialog = false
              this.dialogCreateDonating = false
            })
          }).catch(() => {
            this.loadingDialog = false
            this.dialogCreateDonating = false
            this.$message({
              type: 'info',
              message: 'Cancelled'
            })
          })
        } else {
          return false
        }
      })
    },
    resetForm(formName) {
      this.$refs[formName].resetFields()
    },
    selectGet(accountId) {
      this.DonatingForm.proprietor = accountId
    }
  }
}

</script>

<style>
  .container{
    width: 100%;
    text-align: center;
    min-height: 100%;
    overflow: hidden;
  }
  .tag {
    float: left;
  }

  .item {
    font-size: 14px;
    margin-bottom: 18px;
    color: #999;
  }

  .clearfix:before,
  .clearfix:after {
    display: table;
  }
  .clearfix:after {
    clear: both
  }

  .realEstate-card {
    width: 280px;
    height: 340px;
    margin: 18px;
  }
</style>
