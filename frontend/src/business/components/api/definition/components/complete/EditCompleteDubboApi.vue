<template>

  <!-- 操作按钮 -->
  <div style="background-color: white;">
    <el-row>
      <el-col>
        <!--操作按钮-->
        <div style="float: right;margin-right: 20px;margin-top: 20px" class="ms-opt-btn">
          <el-link type="primary" style="margin-right: 20px" @click="openHis" v-if="basisData.id">
            {{ $t('operating_log.change_history') }}
          </el-link>
          <el-button type="primary" size="small" @click="saveApi" title="ctrl + s">{{ $t('commons.save') }}</el-button>
        </div>
      </el-col>
    </el-row>
    <!-- 基础信息 -->
    <p class="tip">{{ $t('test_track.plan_view.base_info') }} </p>
    <br/>
    <el-row>
      <el-col>
        <ms-basis-api @createRootModelInTree="createRootModelInTree" :moduleOptions="moduleOptions" :basisData="basisData" ref="basicForm"
                      @callback="callback"/>
      </el-col>
    </el-row>

    <!-- 请求参数 -->
    <p class="tip">{{ $t('api_test.definition.request.req_param') }} </p>
    <ms-basis-parameters :showScript="false" :request="request"/>

    <api-other-info :api="basisData"/>
    <ms-change-history ref="changeHistory"/>
  </div>
</template>

<script>
  import MsBasisApi from "./BasisApi";
  import MsBasisParameters from "../request/dubbo/BasisParameters";
  import MsChangeHistory from "../../../../history/ChangeHistory";
  import ApiOtherInfo from "@/business/components/api/definition/components/complete/ApiOtherInfo";

  export default {
    name: "MsApiDubboRequestForm",
    components: {
      ApiOtherInfo,
      MsBasisApi, MsBasisParameters,MsChangeHistory
    },
    props: {
      request: {},
      basisData: {},
      moduleOptions: Array,
      isReadOnly: {
        type: Boolean,
        default: false
      },
      syncTabs: Array,
    },
    watch: {
      syncTabs() {
        if (this.basisData && this.syncTabs && this.syncTabs.includes(this.basisData.id)) {
          // 标示接口在其他地方更新过，当前页面需要同步
          let url = "/api/definition/get/";
          this.$get(url + this.basisData.id, response => {
            if (response.data) {
              let request = JSON.parse(response.data.request);
              let index = this.syncTabs.findIndex(item => {
                if (item === this.basisData.id) {
                  return true;
                }
              })
              this.syncTabs.splice(index, 1);
              Object.assign(this.request, request);
            }
          });
        }
      }
    },

    data() {
      return {validated: false}
    },
    methods: {
      openHis(){
        this.$refs.changeHistory.open(this.basisData.id,["接口定义" , "接口定義" , "Api definition"]);
      },
      callback() {
        this.validated = true;
      },
      validateApi() {
        this.validated = false;
        this.basisData.method = this.request.protocol;
        this.$refs['basicForm'].validate();
      },
      saveApi() {
        this.validateApi();
        if (this.validated) {
          this.basisData.request = this.request;
          if (this.basisData.tags instanceof Array) {
            this.basisData.tags = JSON.stringify(this.basisData.tags);
          }
          this.$emit('saveApi', this.basisData);
        }
      },
      runTest() {
        this.validateApi();
        if (this.validated) {
          this.basisData.request = this.request;
          if (this.basisData.tags instanceof Array) {
            this.basisData.tags = JSON.stringify(this.basisData.tags);
          }
          this.$emit('runTest', this.basisData);
        }
      },
      createRootModelInTree() {
        this.$emit("createRootModelInTree");
      },
    },

    computed: {}
  }
</script>

<style scoped>

.ms-opt-btn {
  position: fixed;
  right: 50px;
  z-index: 1;
}
</style>
