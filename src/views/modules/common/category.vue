<template>
  <el-tree
    :data="menus"
    :props="defaultProps"
    node-key="catId"
    :default-expanded-keys="defaultExpand"
    ref="tree"
    @node-click="clickHandle"
  ></el-tree>
</template>
<script>
export default {
  data() {
    return {
      menus: [],
      defaultExpand: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
    };
  },
  created() {
    this.getMenus();
  },
  methods: {
    clickHandle(data, node, tree) {
      this.$emit("node-click", data, node, tree);
    },
    getMenus() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        this.menus = data.data;
        this.$emit("data-refresh", data.data);
      });
    },
  },
};
</script>