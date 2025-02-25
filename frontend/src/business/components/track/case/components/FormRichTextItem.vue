<template>
  <el-form-item v-loading="result.loading" :disable="true" :label="title" :prop="prop" :label-width="labelWidth">
    <mavon-editor :id="id" v-if="active" :editable="!disabled" @imgAdd="imgAdd" :default-open="defaultOpen" class="mavon-editor"
                  :xss-options="xssOptions"
                  :subfield="false" :toolbars="toolbars" :language="language" :toolbarsFlag="disabled ? false : true" @imgDel="imgDel" v-model="data[prop]"  ref="md"/>
  </el-form-item>
</template>

<script>
import {getCurrentUser, getUUID} from "@/common/js/utils";
export default {
  name: "FormRichTextItem",
  components: {},
  props: ['data', 'title', 'prop', 'disabled', 'labelWidth'],
  data() {
    return {
      result: {loading: false},
      id: getUUID(),
      xssOptions: {
        whiteList: {
          img: ["src", "alt", "width", "height"],
        },
        stripIgnoreTagBody: true
      },
      defaultOpen: 'preview',
      toolbars: {
        bold: true, // 粗体
        italic: true, // 斜体
        header: true, // 标题
        underline: true, // 下划线
        strikethrough: true, // 中划线
        mark: true, // 标记
        superscript: true, // 上角标
        subscript: true, // 下角标
        quote: true, // 引用
        ol: true, // 有序列表
        ul: true, // 无序列表
        link: true, // 链接
        imagelink: true, // 图片链接
        code: true, // code
        table: true, // 表格
        fullscreen: true, // 全屏编辑
        readmodel: true, // 沉浸式阅读
        htmlcode: true, // 展示html源码
        help: true, // 帮助
        /* 1.3.5 */
        undo: true, // 上一步
        redo: true, // 下一步
        trash: true, // 清空
        save: false, // 保存（触发events中的save事件）
        /* 1.4.2 */
        navigation: true, // 导航目录
        /* 2.1.8 */
        alignleft: true, // 左对齐
        aligncenter: true, // 居中
        alignright: true, // 右对齐
        /* 2.2.1 */
        subfield: true, // 单双栏模式
        preview: true, // 预览
      }
    }
  },
  computed: {
    active() {
      if (this.data[this.prop] !== undefined) {
        return true;
      }
      return false;
    },
    language() {
      const user = getCurrentUser();
      const language = user.language;
      switch (language) {
        case 'zh_CN':
          return 'zh-CN';
        case 'zh_TW':
          return 'zh-TW';
        case 'en_US':
          return 'en';
        default:
          return 'zh-CN';
      }
    }
  },
  mounted() {
    // 点击编辑，失去焦点展示
    let el = document.getElementById(this.id);
    if (el) {
      el.addEventListener('click', () => {
        let imagePreview = el.getElementsByClassName('v-note-img-wrapper');
        if (imagePreview.length > 0) { // 图片预览的时候不切换到编辑模式
          this.defaultOpen = 'preview';
        } else {
          this.defaultOpen = null;
        }
      });
      let input = el.getElementsByClassName('auto-textarea-input');
      input[0].addEventListener('blur', () => {
        this.defaultOpen = 'preview';
      });
    }
  },
  methods: {
    imgAdd(pos, file){
      let param = {
        id: getUUID().substring(0, 8)
      };
      file.prefix = param.id;
      this.result.loading = true;
      // 带括号和空格，可能无法展示
      param.fileName = file.name.replace("(", "").replace(")", "").replace(" ", "");
      this.$fileUpload('/resource/md/upload', file, null, param, () => {
        this.$success(this.$t('commons.save_success'));
        this.$refs.md.$img2Url(pos, '/resource/md/get/'  + param.id + '_' + param.fileName);
        this.result.loading = false;
      });
      this.$emit('imgAdd', file);
    },
    imgDel(file) {
      if (file) {
        this.$get('/resource/md/delete/' + file[1].prefix + "_" + file[1].name);
      }
    },
  }
}
</script>

<style scoped>

.mavon-editor {
  min-height: 20px;
}

/deep/ .v-note-wrapper {
  position: initial;
}

</style>
