<template>
  <component :is="dialogType"
             lock-scroll
             :wrapperClosable="crud.tableOption.dialogClickModal"
             :direction="direction"
             v-dialogDrag="vaildData(crud.tableOption.dialogDrag,config.dialogDrag)"
             :class="b('dialog')"
             :custom-class="vaildData(crud.tableOption.customClass,config.customClass)"
             :fullscreen="fullscreen?fullscreen:(crud.isMobile?true:crud.tableOption.dialogFullscreen)"
             :modal-append-to-body="false"
             append-to-body
             :top="setPx(crud.tableOption.dialogTop,100)"
             :title="dialogTitle"
             :close-on-press-escape="crud.tableOption.dialogEscape"
             :close-on-click-modal="crud.tableOption.dialogClickModal"
             :modal="crud.tableOption.dialogModal"
             :show-close="crud.tableOption.dialogCloseBtn"
             :visible.sync="boxVisible"
             :size="size"
             :width="width"
             @close="closeDialog">
    <div slot="title"
         :class="b('dialog__menu')">
      <span class="el-dialog__title">{{dialogTitle}}</span>
      <div class="menu">
        <i v-if="fullscreen"
           @click="handleFullScreen"
           class="el-dialog__close el-icon-full-screen"></i>
        <i v-else
           @click="handleFullScreen"
           class="el-dialog__close el-icon-full-screen"></i>

      </div>
    </div>
    <div :style="{height:dialogHeight,overflow:'hidden'}"
         ref="content">
      <el-scrollbar style="height:100%">
        <avue-form v-model="tableForm"
                   v-if="boxVisible"
                   ref="tableForm"
                   @submit="handleSubmit"
                   @reset-change="handleReset"
                   :upload-preview="crud.uploadPreview"
                   :upload-delete="crud.uploadDelete"
                   :upload-before="crud.uploadBefore"
                   :upload-after="crud.uploadAfter"
                   :option="formOption">
          <template slot-scope="scope"
                    v-for="item in columnFormOption"
                    :slot="item.prop">
            <slot :value="scope.value"
                  :column="scope.column"
                  :dic="scope.dic"
                  :size="scope.size"
                  :label="scope.label"
                  :disabled="scope.disabled"
                  :row="tableForm"
                  :index="tableIndex"
                  :name="item.prop"
                  v-if="item.formslot"></slot>
          </template>
          <template slot="menuForm"
                    slot-scope="{size}">
            <slot name="menuForm"
                  :type="boxType"
                  :size="size"></slot>
          </template>
        </avue-form>
      </el-scrollbar>
    </div>
  </component>
</template>

<script>
import { filterDefaultParams } from 'utils/util'
import create from "core/create";
import locale from "../../core/common/locale";
import config from "./config";
export default create({
  name: "crud",
  mixins: [locale],
  inject: ["crud"],
  data () {
    return {
      config: config,
      boxType: "",
      fullscreen: false,
      boxVisible: false,
      boxHeight: 0,
      tableForm: {},
      index: -1
    };
  },
  props: {
    columnFormOption: {},
    value: {
      type: Object,
      default: () => {
        return {};
      }
    }
  },
  watch: {
    boxVisible (val) {
      if (val) {
        this.$nextTick(() => {
          this.initFun()
        })
      }
    },
    value: {
      handler () {
        this.formVal();
      },
      deep: true
    },
    tableForm: {
      handler () {
        this.$emit("input", this.tableForm);
      },
      deep: true
    }
  },
  mounted () {

  },
  computed: {
    isView () {
      return this.boxType === 'view'
    },
    isAdd () {
      return this.boxType === 'add'
    },
    isEdit () {
      return this.boxType === 'edit'
    },
    direction () {
      return this.crud.tableOption.dialogDirection
    },
    width () {
      return this.setPx(this.vaildData(this.crud.tableOption.dialogWidth, this.crud.isMobile ? '100%' : config.dialogWidth))
    },
    size () {
      return this.vaildData(this.crud.tableOption.dialogWidth + '', this.crud.isMobile ? '100%' : config.dialogWidth + '')
    },
    dialogType () {
      return this.isDrawer ? 'elDrawer' : 'elDialog'
    },
    isDrawer () {
      return this.crud.tableOption.dialogType === 'drawer';
    },
    dialogHeight () {
      if (this.isDrawer) {
        return 'calc(100% - 100px)';
      }
      return this.setPx(this.crud.tableOption.dialogHeight || config.dialogHeight);
    },
    formOption () {
      let option = this.deepClone(this.crud.tableOption);
      option.boxType = this.boxType;
      option.column = this.crud.propOption;
      if (this.isView) {
        option.menuBtn = false;
      } else {
        if (!option.menuPosition) option.menuPosition = 'right'
        if (this.isAdd) {
          option.submitBtn = option.saveBtn;
          option.submitText = option.saveBtnTitle || this.t('crud.saveBtn')
          option.submitIcon = option.saveBtnIcon || 'el-icon-circle-plus-outline';
        } else if (this.isEdit) {
          option.submitBtn = option.updateBtn;
          option.submitText = option.updateBtnTitle || this.t('crud.updateBtn')
          option.submitIcon = option.updateBtnIcon || 'el-icon-circle-check';
        }
        option.emptyBtn = option.cancelBtn;
        option.emptyIcon = 'el-icon-circle-close';
        option.emptyText = option.cancelBtnTitle || this.t('crud.cancelBtn')
      }
      //不分组的表单不加载字典
      if (!this.crud.isGroup) {
        option.dicFlag = false;
        option.dicData = this.crud.DIC;
      }
      return option;
    },
    dialogTitle () {
      const key = `${this.boxType}`;
      if (!this.validatenull(this.boxType)) {
        return this.crud.tableOption[key + 'Title'] || this.t(`crud.${key}Title`);
      }
    }
  },
  methods: {
    handleFullScreen () {
      if (this.fullscreen) {
        this.fullscreen = false;
      } else {
        this.fullscreen = true;
      }
    },
    handleReset () {
      this.closeDialog();
    },
    handleSubmit () {
      if (this.isAdd) {
        this.rowSave();
      } else if (this.isEdit) {
        this.rowUpdate();
      }
    },
    initFun () {
      this.crud.clearValidate = this.$refs.tableForm.clearValidate
      this.crud.validate = this.$refs.tableForm.validate
    },
    updateDic (prop, list) {
      this.$refs.tableForm.updateDic(prop, list);
    },
    formVal () {
      Object.keys(this.value).forEach(ele => {
        this.tableForm[ele] = this.value[ele];
      });
      this.$emit("input", this.tableForm);
    },
    //清空表单
    resetForm () {
      this.$refs["tableForm"].resetForm();
      this.$emit("input", this.tableForm);
    },
    // 保存
    rowSave () {
      this.$refs["tableForm"].validate(vaild => {
        if (!vaild) return;
        this.crud.$emit(
          "row-save",
          filterDefaultParams(this.tableForm, this.crud.tableOption.translate),
          this.closeDialog,
          this.$refs.tableForm.hide
        );
      });
    },
    // 更新
    rowUpdate () {
      this.$refs["tableForm"].validate(vaild => {
        if (!vaild) return;
        const index = this.tableIndex;
        this.crud.$emit(
          "row-update",
          filterDefaultParams(this.tableForm, this.crud.tableOption.translate),
          this.index,
          this.closeDialog,
          this.$refs.tableForm.hide
        );
      });
    },
    closeDialog () {
      this.tableIndex = -1;
      this.tableForm = {};
      this.hide();
    },
    // 隐藏表单
    hide () {
      const callback = () => {
        this.$nextTick(() => {
          this.boxVisible = false;
        });
      };
      if (typeof this.crud.beforeClose === "function") {
        this.crud.beforeClose(callback, this.boxType);
      } else {
        callback();
      }
    },
    // 显示表单
    show (type, index = -1) {
      this.index = index;
      this.boxType = type;
      const callback = () => {
        this.$nextTick(() => {
          this.boxVisible = true;
        });
      };
      if (typeof this.crud.beforeOpen === "function") {
        this.crud.beforeOpen(callback, this.boxType);
      } else {
        callback();
      }
    }
  }
});
</script>