<template>
  <div>
    <el-row type="flex">
      <el-col :span="codeSpan" class="script-content">
        <ms-code-edit v-if="isCodeEditAlive" :mode="codeEditModeMap[jsr223ProcessorData.scriptLanguage]"
                      :read-only="isReadOnly"
                      height="90%"
                      :data.sync="jsr223ProcessorData.script" theme="eclipse" :modes="['java','python']"
                      ref="codeEdit"/>
      </el-col>
      <div style="width: 14px;margin-right: 5px;">
        <div style="height: 12px;width: 12px; line-height:12px;">
          <i :class="showMenu ? 'el-icon-remove-outline' : 'el-icon-circle-plus-outline'"
             class="show-menu"
             @click="switchMenu"></i>
        </div>
      </div>
      <el-col :span="menuSpan" style="width: 200px" class="script-index">
        <ms-dropdown :default-command="jsr223ProcessorData.scriptLanguage" :commands="languages" style="margin-bottom: 5px;margin-left: 15px;"
                     @command="languageChange"/>
        <mock-script-nav-menu ref="scriptNavMenu" style="width: 90%" :language="jsr223ProcessorData.scriptLanguage" :menus="baseCodeTemplates"
                         @handleCode="handleCodeTemplate"/>
      </el-col>
    </el-row>
  </div>
</template>

<script>

import MsCodeEdit from "@/business/components/api/definition/components/MsCodeEdit";
import MsDropdown from "@/business/components/common/components/MsDropdown";
import CustomFunctionRelate from "@/business/components/project/menu/function/CustomFunctionRelate";
import ApiFuncRelevance from "@/business/components/project/menu/function/ApiFuncRelevance";
import MockScriptNavMenu from "@/business/components/api/definition/components/mock/Components/MockScriptNavMenu";

export default {
  name: "MockApiScriptEditor",
  components: {MsDropdown, MsCodeEdit, CustomFunctionRelate, ApiFuncRelevance, MockScriptNavMenu},
  data() {
    return {
      jsr223ProcessorData: {},
      baseCodeTemplates: [],
      httpCodeTemplates: [
        {
          title: "API"+this.$t('api_test.definition.document.request_info'),
          children: [
            {
              title: this.$t('api_test.request.address'),
              value: 'var returnMsg = requestParams.get("address");\nreturn returnMsg;',
            },
            {
              title: "Header "+this.$t('api_test.definition.document.request_param'),
              value: 'var returnMsg = requestParams.get("header.${param}");\nreturn returnMsg;',
            },
            {
              title: this.$t('api_test.request.body')+this.$t('api_test.variable'),
              value: 'var returnMsg = requestParams.get("body.${param}");\nreturn returnMsg;\n' +
                "\n"+
                '//如果对象是多层JSON，需要引入fastjson协助解析:\n' +
                '// 以"{\"name\":\"user\",\"rows\":[{\"type\":1}]}" 为demo,取rows第1个的type数据:\n' +
                'import com.alibaba.fastjson.JSON;\n'+
                'import com.alibaba.fastjson.JSONArray;\n'+
                'import com.alibaba.fastjson.JSONObject;\n'+
                '\n'+
                'var jsonParam = requestParams.get("body.json");\n' +
                'JSONObject jsonObject = JSONObject.parseObject(jsonParam);\n' +
                'var returnMsg = jsonObject.getJSONArray("rows").getJSONObject(0).getString("type");\n' +
                'return returnMsg;\n',
            },
            {
              title: this.$t('api_test.request.body')+this.$t('api_test.variable')+" (Raw)",
              value: 'var returnMsg = requestParams.get("bodyRaw");\nreturn returnMsg;',
            },
            {
              title: "Query "+this.$t('api_test.definition.document.request_param'),
              value: 'var returnMsg = requestParams.get("query.${param}");\nreturn returnMsg;',
            },
            {
              title: "Rest "+this.$t('api_test.definition.document.request_param'),
              value: 'var returnMsg = requestParams.get("rest.${param}");\nreturn returnMsg;',
            },

          ]
        },
        {
          title: this.$t('project.code_segment.code_segment'),
          children: [
            {
              title: this.$t('project.code_segment.insert_segment'),
              command: "custom_function",
            }
          ]
        },
      ],
      tcpCodeTemplates: [
        {
          title: this.$t('project.code_segment.code_segment'),
          children: [
            {
              title: this.$t('project.code_segment.insert_segment'),
              command: "custom_function",
            }
          ]
        },
      ],
      isCodeEditAlive: true,
      languages: [
        'beanshell'
        // , "python",  "nashornScript", "rhinoScript"
      ],
      codeEditModeMap: {
        beanshell: 'java',
        python: 'python',
        groovy: 'java',
        nashornScript: 'javascript',
        rhinoScript: 'javascript',
      },
      codeSpan: 20,
      menuSpan: 4,
      showMenu: true,
    }
  },
  created() {
    this.jsr223ProcessorData = this.jsr223Processor;
    if(this.showApi){
      this.baseCodeTemplates = this.httpCodeTemplates;
    }else {
      this.baseCodeTemplates = this.tcpCodeTemplates;
    }
  },
  props: {
    isReadOnly: {
      type: Boolean,
      default:
        false
    },
    jsr223Processor: {
      type: Object,
    },
    showApi:{
      type:Boolean,
      default:true,
    },
    node: {},
  },
  watch: {
    jsr223Processor() {
      this.reload();
    }
  }
  ,
  methods: {
    addTemplate(template) {
      if (!this.jsr223ProcessorData.script) {
        this.jsr223ProcessorData.script = "";
      }
      this.jsr223ProcessorData.script += template.value;
      if (this.jsr223ProcessorData.scriptLanguage === 'beanshell') {
        this.jsr223ProcessorData.script += ';';
      }
      this.reload();
    },
    reload() {
      this.isCodeEditAlive = false;
      this.$nextTick(() => (this.isCodeEditAlive = true));
    },
    languageChange(language) {
      this.jsr223ProcessorData.scriptLanguage = language;
      this.$emit("languageChange");
    },
    addCustomFuncScript(script) {
      this.jsr223ProcessorData.script = this.jsr223ProcessorData.script ?
        this.jsr223ProcessorData.script + '\n\n' + script : script;
      this.reload();
    },
    switchMenu() {
      this.showMenu = !this.showMenu;
      if (this.showMenu) {
        this.codeSpan = 20;
        this.menuSpan = 4;
      } else {
        this.codeSpan = 24;
        this.menuSpan = 0;
      }
    },
    handleCodeTemplate(code) {
      if (!this.jsr223ProcessorData.script) {
        this.jsr223ProcessorData.script = code;
      } else {
        this.jsr223ProcessorData.script = this.jsr223ProcessorData.script + '\n' + code;
      }
      this.reload();
    },
  }
}
</script>

<style scoped>
.ace_editor {
  border-radius: 5px;
}

.script-content {
  /*height: calc(100vh - 570px);*/
  height: 185px;
  min-height: 170px;
}

.script-index {
  padding: 0 20px;
}

.template-title {
  margin-bottom: 5px;
  font-weight: bold;
  font-size: 15px;
}

.document-url {
  margin-top: 10px;
}

.instructions-icon {
  margin-left: 5px;
}

.ms-dropdown {
  margin-bottom: 20px;
}

.show-menu {
  text-align:center;
  font-weight: bold;
  color:#935aa1;
  font-size: 18px;
  cursor: pointer;
}
.show-menu:hover {
  color:#935aa1;
}

</style>
