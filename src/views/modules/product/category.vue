<template>
  <div>
    <el-tree
      :data="menus"
      :props="defaultProps"
      show-checkbox
      node-key="catId"
      :expand-on-click-node="false"
      :default-expanded-keys="expandedKey"
      draggable
      :allow-drop="allowDrop"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button v-if="node.level<=2" type="text" size="mini" @click="() => append(data)">Append</el-button>
          <el-button
            v-if="node.childNodes.length==0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >Delete</el-button>
          <el-button type="text" size="mini" @click="() => edit(data)">Edit</el-button>
        </span>
      </span>
    </el-tree>
    <!-- 对话框 -->
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
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input v-model="category.productUnit" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitData">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
//这里可以导入其他文件（比如：组件，工具js，第三方插件js，json文件，图片文件等等）
//例如：import 《组件名称》 from '《组件路径》';

export default {
  data() {
    return {
      maxLevel: 0,
      title: "",
      dialogType: "",
      category: {
        catId: null,
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        productUnit: "",
        icon: ""
      },
      dialogVisible: false,
      expandedKey: [],
      menus: [],
      defaultProps: {
        children: "children",
        label: "name"
      }
    };
  },
  methods: {
    //获取菜单
    getMenus() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get"
      }).then(({ data }) => {
        console.log(data.data);
        this.menus = data.data;
      });
    },
    submitData() {
      if (this.dialogType == "add") {
        this.addCategory();
      }
      if (this.dialogType == "edit") {
        this.editCategory();
      }
    },
    //添加节点
    append(data) {
      this.title = "添加菜单";
      this.dialogType = "add";
      this.dialogVisible = true;

      this.category.parentCid = data.catId;
      this.category.name = "";
      this.category.catLevel = d6ata.catLevel * 1 + 1;
      this.category.showStatus = 1;
      this.category.sort = 0;
      this.category.productUnit = "";
      this.category.icon = "";
    },
    //修改节点
    edit(data) {
      this.title = "编辑菜单";
      this.dialogType = "edit";
      this.dialogVisible = true;
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "GET",
        params: this.$http.adornParams({})
      }).then(({ data }) => {
        console.log(data);
        this.category.catId = data.data.catId;
        this.category.name = data.data.name;
        this.category.icon = data.data.icon;
        this.category.productUnit = data.data.productUnit;
        this.category.parentCid = data.data.parentCid;
      });
    },
    //添加三级分类的方法
    addCategory() {
      console.log("提交的数据", this.category);
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "POST",
        data: this.$http.adornData(this.category, false)
      }).then(({ data }) => {
        this.$message({
          showClose: true,
          message: "菜单保存成功",
          type: "success"
        });
        this.dialogVisible = false;
        this.getMenus();
        this.expandedKey = [this.category.parentCid];
      });
    },
    //修改节点信息
    editCategory() {
      var { catId, name, icon, productUnit } = this.category;
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "POST",
        data: this.$http.adornData({ catId, name, icon, productUnit }, false)
      }).then(({ data }) => {
        this.$message({
          showClose: true,
          message: "菜单修改成功",
          type: "success"
        });
        this.dialogVisible = false;
        this.getMenus();
        this.expandedKey = [this.category.parentCid];
      });
    },
    //删除节点
    remove(node, data) {
      var ids = [data.catId];
      this.$confirm(`是否删除【${data.name}】菜单?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning"
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "POST",
            data: this.$http.adornData(ids, false)
          }).then(({ data }) => {
            this.$message({
              showClose: true,
              message: "菜单删除成功",
              type: "success"
            });

            this.getMenus();
            this.expandedKey = [node.parent.data.catId];
          });
        })
        .catch(() => {});
    },
    allowDrop(draggingNode, dropNode, type) {
      //总层数不能大于3
      console.log("allowDrop" + draggingNode, dropNode, type);
      this.countNodeLevel(draggingNode.data);
      let deep = this.maxLevel - draggingNode.data.catLevel + 1;
      if (type == "inner") {
        return deep + dropNode.level <= 3;
      } else {
        return deep + dropNode.parent.level <= 3;
      }
    },
    //得到菜单最大深度 递归
    countNodeLevel(node) {
      if (node.children != null && node.children.length > 0) {
        for (let i = 0; i < node.children.length; i++) {
          if (node.children[i].catLevel > this.maxLevel) {
            this.maxLevel = node.children[i].catLevel;
          }
          this.countNodeLevel(node.children[i]);
        }
      }
    }
  },
  created() {
    this.getMenus();
  }
};
</script>
