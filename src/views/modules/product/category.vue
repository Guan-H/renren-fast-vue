<template>
  <div>
    <el-button type="primary" @click="appendOne" size="medium" round>添加一级菜单</el-button>
    <el-switch v-model="draggable" active-text="开启拖拽" inactive-text="关闭拖拽">
    </el-switch>
    <el-button v-if="draggable" @click="batchSave" size="mini" type="primary" round :loading="loading">批量保存</el-button>
    <el-button type="danger" size="mini" @click="batchDelete">批量删除</el-button>
    <el-tree :data="menus" :props="defaultProps" show-checkbox :expand-on-click-node="false" node-key="catId"
      :default-expanded-keys="expandedKey" :draggable="draggable" :allow-drop="allowDrop" @node-drop="handleDrop"
      ref="menuTree">
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button type="text" v-if="node.level <= 2" size="mini" @click="() => append(data)">
            Append
          </el-button>
          <el-button type="text" v-if="node.childNodes.length == 0" size="mini" @click="() => remove(node, data)">
            Delete
          </el-button>
          <el-button type="text" size="mini" @click="() => edit(data)">
            Edit
          </el-button>
        </span>
      </span>
    </el-tree>

    <!--添加菜单节点-->
    <el-dialog :title="title" :visible.sync="dialogVisible" width="30%" :close-on-click-modal="false">
      <el-form :model="category">
        <el-form-item label="活动名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input v-model="category.productUnit" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>

        <el-button type="primary" @click="submitDate">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
//这里可以导入其他文件（比如：组件，工具js，第三方插件js，json文件，图片文件等等）
//例如：import 《组件名称》from '《组件路径》';

import { addListener } from "process";
import Vue from "vue";
export default {
  //import 引入的组件需要注入到对象中才能使用
  components: {},
  props: {},
  data() {
    return {
      loading: false,
      draggable: false,
      pCid: [],
      updateNodes: [],
      maxLevel: 0,
      menus: [],
      expandedKey: [],
      dialogVisible: false,
      dialogType: "",
      title: "",
      category: {
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        catId: null,
        productUnit: "",
        icon: "",
      },
      defaultProps: {
        children: "children",
        label: "name",
      },
    };
  },
  methods: {
    submitDate() {
      if (this.dialogType == "addCategory") {
        this.addCategory();
      }
      if (this.dialogType == "edit") {
        this.title = "修改";
        this.editCategory();
      }
    },
    getMenu() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        console.log("获取数据....", data.data);
        this.menus = data.data;
      });
    },
    batchDelete() {
      let catIds = [];
      let menuName =[];
      let size = 0;
      var data = this.$refs.menuTree.getCheckedNodes();
      for (let i = 0; i < data.length; i++) {
        catIds.push(data[i].catId);
        menuName.push(data[i].name);
        size ++;
      }

      this.$confirm(`是否批量删除【${menuName}】共${size}个菜单?`, {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(catIds, false),
          }).then(({ data }) => {
            this.getMenu();
            this.$message({
              type: "success",
              message: "删除成功!",
            });
          });
        })
        .catch(() => { });

      console.log("拿到的数据:", data);
      console.log("拿到的ID:", catIds);

    },
    //------------------------------------------------------------------------
    handleDrop(draggingNode, dropNode, dropType, ev) {
      console.log("handleDrop: ", draggingNode, dropNode, dropType);
      //1、当前节点最新的父节点id
      let pCid = 0;
      let siblings = null;
      if (dropType == "before" || dropType == "after") {
        pCid =
          dropNode.parent.data.catId == undefined
            ? 0
            : dropNode.parent.data.catId;
        siblings = dropNode.parent.childNodes;
      } else {
        pCid = dropNode.data.catId;
        siblings = dropNode.childNodes;
      }
      this.pCid.push(pCid);

      //2、当前拖拽节点的最新顺序，
      for (let i = 0; i < siblings.length; i++) {
        if (siblings[i].data.catId == draggingNode.data.catId) {
          //如果遍历的是当前正在拖拽的节点
          let catLevel = draggingNode.level;
          if (siblings[i].level != draggingNode.level) {
            //当前节点的层级发生变化
            catLevel = siblings[i].level;
            //修改他子节点的层级
            this.updateChildNodelvel(siblings[i]);
          }
          this.updateNodes.push({
            catId: siblings[i].data.catId,
            sort: i,
            parentCid: pCid,
            catLevel: catLevel,
          });
        } else {
          this.updateNodes.push({ catId: siblings[i].data.catId, sort: i });
        }
      }

      //3、当前拖拽节点的最新层级
      console.log("updateNodes", this.updateNodes);
    },
    batchSave() {
      this.loading = true;
      this.$http({
        url: this.$http.adornUrl("/product/category/update/sort"),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单顺序等修改成功",
          type: "success",
        });
        //刷新出新的菜单
        this.getMenu();
        //设置需要默认展开的菜单
        this.expandedKey = this.pCid;
        this.updateNodes = [];
        this.maxLevel = 0;
        this.loading = false;
        // this.pCid = 0;
      });
    },

    updateChildNodelvel(node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var cNode = node.childNodes[i].data;
          this.updateNodes.push({
            catId: cNode.catId,
            catLevel: node.childNodes[i].level,
          });
          this.updateChildNodelvel(node.childNodes[i]);
        }
      }
    },
    //---------------------------------------------------------------------
    allowDrop(draggingNode, dropNode, type) {
      //1、被拖动的当前节点以及所在的父节点总层数不能大于3

      //1）、被拖动的当前节点总层数
      console.log("allowDrop:", draggingNode, dropNode, type);
      //
      this.countNodeLevel(draggingNode);
      //当前正在拖动的节点+父节点所在的深度不大于3即可
      let deep = Math.abs(this.maxLevel - draggingNode.level) + 1;
      console.log("深度：", deep);

      //   this.maxLevel
      if (type == "inner") {
        return deep + dropNode.level <= 3;
      } else {
        return deep + dropNode.parent.level <= 3;
      }
    },
    countNodeLevel(node) {
      //找到所有子节点，求出最大深度
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level;
          }
          this.countNodeLevel(node.childNodes[i]);
        }
      }
    },
    //修改操作
    edit(data) {
      this.dialogVisible = !this.dialogVisible;
      this.title = "修改分类";
      this.dialogType = "edit";
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
      }).then(({ data }) => {
        console.log("点击edit后，刷新重新获取到要回显的数据", data);
        this.category.name = data.data.name;
        this.category.catId = data.data.catId;
        this.category.icon = data.data.icon;
        this.category.productUnit = data.data.productUnit;
        this.category.parentCid = data.data.parentCid;
        this.category.catLevel = data.data.catLevel;
        this.category.showStatus = data.data.showStatus;
        this.category.sort = data.data.sort;
        /*
          parentCid: 0,
          catLevel: 0,
          showStatus: 1,
          sort: 0,
         */
      });
    },
    editCategory() {
      //解构获取元素
      var { catId, name, icon, productUnit } = this.category;
      //发送post请求到后端
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData({ catId, name, icon, productUnit }, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单修改成功",
          type: "success",
        });
        //关闭对话框
        this.dialogVisible = false;
        //刷新菜单
        this.getMenu();
        //设置需要默认展开的菜单
        this.expandedKey = [this.category.parentCid];
      });
    },
    //添加菜单
    //添加一级菜单
    appendOne() {
      this.dialogVisible = !this.dialogVisible;
      this.title = "添加一级菜单";
      this.dialogType = "addCategory";
      this.category.catLevel = 1;
      this.category.parentCid = 0;
      this.category.name = "";
      this.category.catId = null;
      console.log("添加一级菜单 ...");
    },
    append(data) {
      this.dialogVisible = true;
      this.title = "添加";
      this.dialogType = "addCategory";
      this.category.parentCid = data.catId;
      this.category.catLevel = data.catLevel * 1 + 1;
      this.category.name = "";
      this.category.catId = null;
      this.category.icon = "";
      this.category.productUnit = "";
      this.category.showStatus = 1;
      this.category.sort = 0;
    },
    //提交添加菜单
    addCategory() {
      console.log("添加数据 ...", this.category);
      this.dialogVisible = !this.dialogVisible;
      //发送post请求
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      }).then(({ data }) => {
        //发送请求成功后
        this.getMenu();
        this.expandedKey = [this.category.parentCid];
        this.$message({
          message: "添加成功",
          type: "success",
        });
      });
    },
    //删除
    remove(node, data) {
      var ids = [data.catId];
      this.$confirm(`是否删除【${data.name}】菜单?`, {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$message({
            type: "success",
            message: "删除成功!",
          });
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false),
          }).then(({ data }) => {
            this.getMenu();
            this.expandedKey = [node.parent.data.catId];
            console.log("expandedKey ==  " + this.expandedKey);
          });
        })
        .catch(() => { });

      console.log("remove", node, data);
    },
  },
  //计算属性类似于data 概念
  computed: {},
  //监控data 中的数据变化
  watch: {},
  //生命周期- 创建完成（可以访问当前this 实例）
  created() {
    this.getMenu();
  },
  //生命周期- 挂载完成（可以访问DOM 元素）
  mounted() { },
  beforeCreate() { }, //生命周期- 创建之前
  beforeMount() { }, //生命周期- 挂载之前
  beforeUpdate() { }, //生命周期- 更新之前
  updated() { }, //生命周期- 更新之后
  beforeDestroy() { }, //生命周期- 销毁之前
  destroyed() { }, //生命周期- 销毁完成
  activated() { }, //如果页面有keep-alive 缓存功能，这个函数会触发
};
</script>
<style  scoped>
</style>