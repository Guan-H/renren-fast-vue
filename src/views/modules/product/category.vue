<template>
  <el-tree
    :data="menus"
    :props="defaultProps"
    @node-click="handleNodeClick"
  ></el-tree>
</template>

<script>
//这里可以导入其他文件（比如：组件，工具js，第三方插件js，json文件，图片文件等等）
//例如：import 《组件名称》from '《组件路径》';

export default {
  //import 引入的组件需要注入到对象中才能使用
  components: {},
  props: {},
  data() {
    return{
      menus:[],
      defaultProps:{  
        children:"children",
        label:"name"
      }
    }
  },
  methods: {
    getMenu() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        console.log("获取数据....", data.data);
        this.menus = data.data;
      });
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
  mounted() {},
  beforeCreate() {}, //生命周期- 创建之前
  beforeMount() {}, //生命周期- 挂载之前
  beforeUpdate() {}, //生命周期- 更新之前
  updated() {}, //生命周期- 更新之后
  beforeDestroy() {}, //生命周期- 销毁之前
  destroyed() {}, //生命周期- 销毁完成
  activated() {}, //如果页面有keep-alive 缓存功能，这个函数会触发
};
</script>
<style  scoped>
</style>