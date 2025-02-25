<template>
  <ms-container v-if="renderComponent" v-loading="loading">

    <ms-aside-container>
      <test-case-node-tree
        @nodeSelectEvent="nodeChange"
        @refreshTable="refresh"
        @setTreeNodes="setTreeNodes"
        @exportTestCase="exportTestCase"
        @saveAsEdit="editTestCase"
        :show-operator="true"
        @createCase="handleCaseSimpleCreate($event, 'add')"
        @refreshAll="refreshAll"
        @enableTrash="enableTrash"
        :type="'edit'"
        :total='total'
        ref="nodeTree"
      />
    </ms-aside-container>

    <ms-main-container>
      <el-tabs v-model="activeName" @tab-click="addTab" @tab-remove="closeConfirm">
        <el-tab-pane name="trash" v-if="trashEnable" :label="$t('commons.trash')">
          <test-case-list
            :checkRedirectID="checkRedirectID"
            :isRedirectEdit="isRedirectEdit"
            :tree-nodes="treeNodes"
            :trash-enable="true"
            @refreshTable="refresh"
            @testCaseEdit="editTestCase"
            @testCaseCopy="copyTestCase"
            @testCaseDetail="showTestCaseDetail"
            @getTrashList="getTrashList"
            @refresh="refresh"
            @refreshAll="refreshAll"
            @setCondition="setCondition"
            ref="testCaseTrashList">
          </test-case-list>
        </el-tab-pane>
        <el-tab-pane name="default" :label="$t('api_test.definition.case_title')">
          <ms-tab-button
            :active-dom="activeDom"
            @update:activeDom="updateActiveDom"
            :left-tip="$t('test_track.case.list')"
            :left-content="$t('test_track.case.list')"
            :right-tip="$t('test_track.case.minder')"
            :right-content="$t('test_track.case.minder')"
            :middle-button-enable="false">
            <test-case-list
              v-if="activeDom === 'left'"
              :checkRedirectID="checkRedirectID"
              :isRedirectEdit="isRedirectEdit"
              :tree-nodes="treeNodes"
              :trash-enable="false"
              @refreshTable="refresh"
              @testCaseEdit="editTestCase"
              @testCaseCopy="copyTestCase"
              @testCaseDetail="showTestCaseDetail"
              @getTrashList="getTrashList"
              @refresh="refresh"
              @refreshAll="refreshAll"
              @setCondition="setCondition"
              @decrease="decrease"
              ref="testCaseList">
            </test-case-list>
            <test-case-minder
              :tree-nodes="treeNodes"
              :project-id="projectId"
              :condition="condition"
              v-if="activeDom === 'right'"
              @refresh="refreshTable"
              ref="minder"/>
          </ms-tab-button>
        </el-tab-pane>
        <el-tab-pane
          :key="item.name"
          v-for="(item) in tabs"
          :label="item.label"
          :name="item.name"
          closable>
          <div class="ms-api-scenario-div">
            <test-case-edit
              :currentTestCaseInfo="item.testCaseInfo"
              @refresh="refreshTable"
              @caseEdit="handleCaseCreateOrEdit($event,'edit')"
              @caseCreate="handleCaseCreateOrEdit($event,'add')"
              :read-only="testCaseReadOnly"
              :tree-nodes="treeNodes"
              :select-node="selectNode"
              :select-condition="condition"
              :type="type"
              @addTab="addTab"
              ref="testCaseEdit">
            </test-case-edit>
          </div>
        </el-tab-pane>
        <el-tab-pane name="add" v-if="hasPermission('PROJECT_TRACK_CASE:READ+CREATE')">
          <template v-slot:label>
            <el-dropdown @command="handleCommand" v-permission="['PROJECT_TRACK_CASE:READ+CREATE']">
              <el-button type="primary" plain icon="el-icon-plus" size="mini"/>
              <el-dropdown-menu slot="dropdown">
                <el-dropdown-item command="ADD" v-permission="['PROJECT_TRACK_CASE:READ+CREATE']">
                  {{ $t('test_track.case.create') }}
                </el-dropdown-item>
                <el-dropdown-item command="CLOSE_ALL">{{ $t('api_test.definition.request.close_all_label') }}
                </el-dropdown-item>
              </el-dropdown-menu>
            </el-dropdown>
          </template>
        </el-tab-pane>

      </el-tabs>

      <is-change-confirm
        :title="'请保存脑图'"
        :tip="'脑图未保存，确认保存脑图吗？'"
        @confirm="changeConfirm"
        ref="isChangeConfirm"/>
    </ms-main-container>


  </ms-container>

</template>

<script>

import NodeTree from '../common/NodeTree';
import TestCaseEdit from './components/TestCaseEdit';
import TestCaseList from "./components/TestCaseList";
import SelectMenu from "../common/SelectMenu";
import MsContainer from "../../common/components/MsContainer";
import MsAsideContainer from "../../common/components/MsAsideContainer";
import MsMainContainer from "../../common/components/MsMainContainer";
import {getCurrentProjectID, getUUID, hasPermission, setCurTabId} from "@/common/js/utils";
import TestCaseNodeTree from "../common/TestCaseNodeTree";

import MsTabButton from "@/business/components/common/components/MsTabButton";
import TestCaseMinder from "@/business/components/track/common/minder/TestCaseMinder";
import IsChangeConfirm from "@/business/components/common/components/IsChangeConfirm";
import {openMinderConfirm, saveMinderConfirm} from "@/business/components/track/common/minder/minderUtils";

export default {
  name: "TestCase",
  components: {
    IsChangeConfirm,
    TestCaseMinder,
    MsTabButton,
    TestCaseNodeTree,
    MsMainContainer,
    MsAsideContainer, MsContainer, TestCaseList, NodeTree, TestCaseEdit, SelectMenu
  },
  comments: {},
  data() {
    return {
      result: {},
      projects: [],
      treeNodes: [],
      testCaseReadOnly: true,
      trashEnable: false,
      condition: {},
      activeName: 'default',
      tabs: [],
      renderComponent: true,
      loading: false,
      type: '',
      activeDom: 'left',
      tmpActiveDom: null,
      total: 0
    };
  },
  mounted() {
    this.getProject();
    let routeTestCase = this.$route.params.testCase;
    if(routeTestCase && routeTestCase.add===true){
      this.addTab({name: 'add'});
    }else {
      this.init(this.$route);
    }
  },
  watch: {
    redirectID() {
      this.renderComponent = false;
      this.$nextTick(() => {
        // 在 DOM 中添加 my-component 组件
        this.renderComponent = true;
      });
    },
    '$route'(to) {
      this.init(to);
    },
    activeName(newVal, oldVal) {
      if (oldVal !== 'default' && newVal === 'default' && this.$refs.minder) {
        this.$refs.minder.refresh();
      }
    },
    activeDom(newVal, oldVal) {
      this.$nextTick(() => {
        if (oldVal !== 'left' && newVal === 'left' && this.$refs.testCaseList) {
          this.$refs.testCaseList.getTemplateField();
        }
      });
    },
    trashEnable(){
      if(this.trashEnable){
        this.activeName = 'trash';
      }else {
        this.activeName = 'default';
      }
    }
  },
  computed: {
    checkRedirectID: function () {
      let redirectIDParam = this.$route.params.redirectID;
      this.changeRedirectParam(redirectIDParam);
      return redirectIDParam;
    },
    isRedirectEdit: function () {
      let redirectParam = this.$route.params.dataSelectRange;
      return redirectParam;
    },

    projectId() {
      return getCurrentProjectID();
    },
    selectNodeIds() {
      return this.$store.state.testCaseSelectNodeIds;
    },
    selectNode() {
      return this.$store.state.testCaseSelectNode;
    },
    moduleOptions() {
      return this.$store.state.testCaseModuleOptions;
    }
  },
  methods: {
    hasPermission,
    handleCommand(e) {
      switch (e) {
        case "ADD":
          this.addTab({name: 'add'});
          break;
        case "CLOSE_ALL":
          this.handleTabClose();
          break;
        default:
          this.addTab({name: 'add'});
          break;
      }
    },
    getTrashList(){
      this.$get("/case/node/trashCount/"+this.projectId , response => {
        this.total = response.data;
      });
    },
    updateActiveDom(activeDom) {
      openMinderConfirm(this, activeDom);
    },
    changeConfirm(isSave) {
      saveMinderConfirm(this, isSave);
    },
    changeRedirectParam(redirectIDParam) {
      this.redirectID = redirectIDParam;
      if (redirectIDParam != null) {
        if (this.redirectFlag == "none") {
          this.activeName = "default";
          this.redirectFlag = "redirected";
        }
      } else {
        this.redirectFlag = "none";
      }
    },
    addTab(tab) {
      if (!this.projectId) {
        this.$warning(this.$t('commons.check_project_tip'));
        return;
      }
      if (tab.name === 'add') {
        let label = this.$t('test_track.case.create');
        let name = getUUID().substring(0, 8);
        this.activeName = name;
        this.type = 'add';
        this.tabs.push({label: label, name: name, testCaseInfo: {testCaseModuleId: "", id: getUUID()}});
      }
      if (tab.name === 'edit') {
        let label = this.$t('test_track.case.create');
        let name = getUUID().substring(0, 8);
        this.activeName = name;
        label = tab.testCaseInfo.name;
        this.tabs.push({label: label, name: name, testCaseInfo: tab.testCaseInfo});
      }

      setCurTabId(this, tab, 'testCaseEdit');
    },
    handleTabClose() {
      let message = "";
      this.tabs.forEach(t => {
        if (t && this.$store.state.testCaseMap.has(t.testCaseInfo.id) && this.$store.state.testCaseMap.get(t.testCaseInfo.id) > 1) {
          message += t.testCaseInfo.name + "，";
        }
      })
      if (message !== "") {
        this.$alert("用例[ " + message.substr(0, message.length - 1) + " ]未保存，是否确认关闭全部？", '', {
          confirmButtonText: this.$t('commons.confirm'),
          cancelButtonText: this.$t('commons.cancel'),
          callback: (action) => {
            if (action === 'confirm') {
              this.$store.state.testCaseMap.clear();
              this.tabs = [];
              this.activeName = "default";
              this.refresh();
            }
          }
        });
      } else {
        this.tabs = [];
        this.activeName = "default";
        this.refresh();
      }
    },
    closeConfirm(targetName) {
      let t = this.tabs.filter(tab => tab.name === targetName);
      if (t && this.$store.state.testCaseMap.has(t[0].testCaseInfo.id) && this.$store.state.testCaseMap.get(t[0].testCaseInfo.id) > 1) {
        this.$alert("用例[ " + t[0].testCaseInfo.name + " ]未保存，是否确认关闭？", '', {
          confirmButtonText: this.$t('commons.confirm'),
          cancelButtonText: this.$t('commons.cancel'),
          callback: (action) => {
            if (action === 'confirm') {
              this.$store.state.testCaseMap.delete(t[0].testCaseInfo.id);
              this.removeTab(targetName);
            }
          }
        });
      } else {
        this.$store.state.testCaseMap.delete(t[0].testCaseInfo.id);
        this.removeTab(targetName);
      }
    },
    removeTab(targetName) {
      this.tabs = this.tabs.filter(tab => tab.name !== targetName);
      if (this.tabs.length > 0) {
        this.activeName = this.tabs[this.tabs.length - 1].name;
      } else {
        this.activeName = "default";
      }
    },
    exportTestCase(type) {
      if (this.activeDom !== 'left') {
        this.$warning(this.$t('test_track.case.export.export_tip'));
        return;
      }
      this.$refs.testCaseList.exportTestCase(type);
    },
    init(route) {
      let path = route.path;
      if (path.indexOf("/track/case/edit") >= 0 || path.indexOf("/track/case/create") >= 0) {
        this.testCaseReadOnly = false;
        let caseId = this.$route.params.caseId;
        let routeTestCase = this.$route.params.testCase;
        if (!this.projectId) {
          this.$warning(this.$t('commons.check_project_tip'));
          return;
        }
        if(routeTestCase){
          this.editTestCase(routeTestCase);
        }else if (caseId) {
          this.$get('test/case/get/' + caseId, response => {
            let testCase = response.data;
            this.editTestCase(testCase);
          });
        } else {
          this.addTab({name: 'add'});
        }
        this.$router.push('/track/case/all');
      }
    },
    nodeChange(node) {
      this.condition.trashEnable = false;
      this.trashEnable = false;
      this.activeName = "default";
    },
    refreshTable(data) {
      if (this.$refs.testCaseList) {
        this.$refs.testCaseList.initTableData();
      }
      this.$refs.nodeTree.list();
      this.setTable(data);
    },
    increase(id) {
      this.$refs.nodeTree.increase(id);
    },
    decrease(id) {
      this.$refs.nodeTree.decrease(id);
    },
    editTestCase(testCase) {
      const index = this.tabs.find(p => p.testCaseInfo && p.testCaseInfo.id === testCase.id);
      if (!index) {
        this.type = "edit";
        this.testCaseReadOnly = false;
        if (testCase.label !== "redirect") {
          if (this.treeNodes.length < 1) {
            this.$warning(this.$t('test_track.case.create_module_first'));
            return;
          }
        }
        let hasEditPermission = hasPermission('PROJECT_TRACK_CASE:READ+EDIT');
        this.$set(testCase, 'rowClickHasPermission', hasEditPermission);
        this.addTab({name: 'edit', testCaseInfo: testCase});
      } else {
        this.activeName = index.name;
      }
    },
    handleCaseCreateOrEdit(data, type) {
      if (this.$refs.minder) {
        this.$refs.minder.addCase(data, type);
      }
    },
    handleCaseSimpleCreate(data, type) {
      if ('default-module' === data.nodeId) {
        for (let i = 0; i < this.moduleOptions.length; i++) {
          let item = this.moduleOptions[i];
          if (item.path.indexOf('未规划用例') > -1) {
            data.nodeId = item.id;
            break;
          }
        }
      }
      this.handleCaseCreateOrEdit(data, type);
      if (this.$refs.minder) {
        this.$refs.minder.refresh();
      }
    },
    copyTestCase(testCase) {
      this.type = "copy";
      this.testCaseReadOnly = false;
      testCase.isCopy = true;
      this.addTab({name: 'edit', testCaseInfo: testCase});
    },
    showTestCaseDetail(testCase) {
      this.testCaseReadOnly = true;
    },
    refresh(data) {
      this.$store.commit('setTestCaseSelectNode', {});
      this.$store.commit('setTestCaseSelectNodeIds', []);
      this.refreshTable(data);
    },
    setTable(data) {
      if (data) {
        for (let index in this.tabs) {
          let tab = this.tabs[index];
          if (tab.name === this.activeName) {
            tab.label = data.name;
            break;
          }
        }
      }
    },
    refreshAll() {
      this.$refs.nodeTree.list();
      this.refresh();
    },
    openRecentTestCaseEditDialog(caseId) {
      if (caseId) {
        // this.getProjectByCaseId(caseId);
        this.$get('/test/case/get/' + caseId, response => {
          if (response.data) {
            /*
                        this.$refs.testCaseEditDialog.open(response.data);
            */
          }
        });
      } else {
        /*
                this.$refs.testCaseEditDialog.open();
        */
      }
    },
    setTreeNodes(data) {
      this.treeNodes = data;
    },
    setCondition(data) {
      this.condition = data;
    },
    getProject() {
      this.$get("/project/get/" + this.projectId, result => {
        let data = result.data;
        if (data) {
          this.$store.commit('setCurrentProjectIsCustomNum',  data.customNum);
        }
      });
    },
    enableTrash(data) {
      this.initApiTableOpretion = "trashEnable";
      this.trashEnable = data;
    },
  }
};
</script>

<style scoped>

.el-main {
  padding: 5px 10px;
}

/deep/ .el-button-group > .el-button:first-child {
  padding: 4px 1px !important;
}

/deep/ .el-tabs__header {
  margin: 0 0 0px;
  /*width: calc(100% - 90px);*/
}

/deep/ .el-table__empty-block {
  width: 100%;
  min-width: 100%;
  max-width: 100%;
  padding-right: 100%;
}

</style>
