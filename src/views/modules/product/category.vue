<template>
  <div>
    <el-switch
      v-model="draggable"
      active-text="开启拖拽"
      inactive-text="关闭拖拽"
    >
    </el-switch>
    <el-button type="danger" @click="batchDelete">批量删除</el-button>
    <el-tree
      :data="menus"
      show-checkbox
      :draggable="draggable"
      node-key="catId"
      :props="defaultProps"
      :expand-on-click-node="false"
      :default-expanded-keys="expand"
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
      ref="tree"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level < 3"
            type="text"
            size="mini"
            @click="() => append(data)"
          >
            Append
          </el-button>
          <el-button type="text" size="mini" @click="edit(data)">
            Edit
          </el-button>
          <el-button
            v-if="node.childNodes.length == 0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >
            Delete
          </el-button>
        </span>
      </span>
    </el-tree>
    <el-dialog
      :title="title"
      :visible.sync="dialogVisible"
      :close-on-click-modal="false"
      width="30%"
    >
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="单位">
          <el-input
            v-model="category.productUnit"
            autocomplete="off"
          ></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="sumbitData">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
/* eslint-disable */
export default {
  data() {
    return {
      draggable: false,
      updateNodes: [],
      menus: [],
      expand: [],
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
      dialogVisible: false,
      defaultProps: {
        children: "children",
        label: "name",
      },
    };
  },
  methods: {
    getTips(nodes, maxLen) {
      var tips = nodes.map((x) => x.name).join(",");
      var n = tips.length;
      if (n > maxLen) {
        tips = tips.substring(0, maxLen);
        tips = `${tips} 等${n}个`;
      }
      return tips;
    },
    batchDelete() {
      let nodes = this.$refs.tree.getCheckedNodes();
      this.$confirm(`是否删除 ${this.getTips(nodes, 40)} 菜单`, "警告", {
        confirmButtonText: "删除",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(
              nodes.map((x) => x.catId),
              false
            ),
          }).then(({ data }) => {
            if (data && data.code == 0) {
              this.$message({
                type: "success",
                message: "菜单删除成功",
              });
            }
            this.getMenus();
          });
        })
        .catch(() => {});

      // console.log(nodes.map((x) => x.name));
    },

    handleDrop(draggingNode, dropNode, dropType, ev) {
      let pNode = dropType == "inner" ? dropNode : dropNode.parent;
      if (pNode == draggingNode.parent) return;
      let pCid = pNode.data.catId || 0,
        sibilings = pNode.childNodes;
      this.updateNodes = [];
      for (var i = 0; i < sibilings.length; i++) {
        var x = sibilings[i];
        if (x.data.catId == draggingNode.data.catId) {
          this.updateNodes.push({
            catId: x.data.catId,
            sort: i,
            parentCid: pCid,
            catLevel: pNode.level + 1,
          });
          if (pNode.level + 1 != x.data.catLevel)
            this.updateLevel(x, pNode.level + 1);
        } else {
          this.updateNodes.push({ catId: x.data.catId, sort: i });
        }
      }
      this.$http({
        url: this.$http.adornUrl("/product/category/update/sort"),
        method: "post",
        data: JSON.stringify(this.updateNodes),
      }).then(({ data }) => {
        if (data && data.code == 0) {
          this.$message({
            type: "success",
            message: "菜单更新成功",
          });
        }
        this.getMenus();
        this.expand = [pNode.data.catId];
      });
    },

    updateLevel(node, level) {
      node.childNodes.forEach((x) => {
        this.updateNodes.push({ catId: x.data.catId, catLevel: level + 1 });
        this.updateLevel(x, level + 1);
      });
    },

    allowDrop(draggingNode, dropNode, type) {
      // console.log(dropNode.level, this.depth(draggingNode));
      if (type == "inner") {
        return this.depth(draggingNode) + dropNode.level <= 3;
      }
      return this.depth(draggingNode) + dropNode.level <= 4;
    },

    depth(node) {
      var d = 1;
      node.childNodes.forEach((x) => {
        d = Math.max(d, 1 + this.depth(x));
      });
      return d;
    },

    sumbitData() {
      if (this.category.catId) {
        this.updateCategory();
      } else {
        this.addCategory();
      }
    },
    updateCategory() {
      var { icon, catId, productUnit, name } = this.category;
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData(
          {
            catId, //自动获取
            name,
            productUnit,
            icon, // 用户填写
          },
          false
        ),
      }).then(({ data }) => {
        if (data && data.code == 0) {
          this.$message({
            type: "success",
            message: "菜单修改成功!",
          });
          this.getMenus();

          this.expand = [this.category.parentCid];
        }
      });
      this.dialogVisible = false;
    },
    addCategory() {
      // 确定添加
      var { name, parentCid, catLevel, productUnit, icon } = this.category;
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(
          {
            name,
            productUnit,
            icon, // 用户填写
            parentCid,
            catLevel, // 自动获取
            sort: 0,
            showStatus: 1, // 默认
            productCount: 0,
          },
          false
        ),
      }).then(({ data }) => {
        if (data && data.code == 0) {
          this.$message({
            type: "success",
            message: "菜单添加成功!",
          });
          this.getMenus();
          this.expand = [this.category.parentCid];
        }
      });

      this.dialogVisible = false;
    },
    getMenus() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        // console.log(data);
        this.menus = data.data;
      });
    },
    append(data) {
      this.title = "添加分类";
      this.dialogVisible = true;
      this.category.parentCid = data.catId;
      this.category.catId = this.category.icon = this.category.name = null;
      this.category.catLevel = data.catLevel * 1 + 1;
    },
    edit(data) {
      this.title = "编辑分类";
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
        // params: this.$http.adornParams({}),
      }).then(({ data }) => {
        if (data && data.code == 0) {
          data = data.data;
          this.category.parentCid = data.parentCid;
          this.category.name = data.name;
          this.category.catId = data.catId;
          this.category.productUnit = data.productUnit;
          this.category.icon = data.icon;
          this.dialogVisible = true;
        }
      });
    },
    remove(node, data) {
      var ids = [data.catId];
      this.$confirm(`是否删除【${data.name}】分类菜单`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false),
          }).then(({ data }) => {
            if (data && data.code == 0) {
              this.$message({
                type: "success",
                message: "删除成功!",
              });
              this.getMenus();
              this.expand = [node.parent.data.catId];
            }
          });
        })
        .catch(() => {
          this.$message({
            type: "info",
            message: "已取消删除",
          });
        });
    },
  },

  created() {
    this.getMenus();
  },
};
</script>

