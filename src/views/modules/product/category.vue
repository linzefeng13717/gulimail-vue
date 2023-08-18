<template>
  <div>
    <el-switch
      v-model="draggable"
      active-text="开启拖拽"
      inactive-text="关闭拖拽"
    >
    </el-switch>
    <el-button v-if="draggable" @click="batchSave">批量保存</el-button>
    <el-button type="warning" @click="batchDelete"
      >批量删除</el-button
    >
    <el-tree
      show-checkbox
      :expand-on-click-node="false"
      :data="data"
      :props="defaultProps"
      @node-click="handleNodeClick"
      node-key="catId"
      :default-expanded-keys="expandedKeys"
      :draggable="draggable"
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
      ref="menuTree"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level <= 2"
            type="text"
            size="mini"
            @click="() => append(data)"
          >
            Append
          </el-button>
          <el-button
            v-if="node.childNodes.length == 0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >
            Delete
          </el-button>
          <el-button type="text" size="mini" @click="() => edit(data)">
            Edit
          </el-button>
        </span>
      </span>
    </el-tree>
    <el-dialog :title="dialogTitle" :visible.sync="dialogVisible" width="30%">
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input
            v-model="category.productUnit"
            autocomplete="off"
          ></el-input>
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
export default {
  data() {
    return {
      updateNodes: [],
      maxLength: "1",
      draggable: false,
      dialogTitle: "",
      dialogType: "", //append or edit
      category: {
        name: "",
        parentCid: 0,
        productUnit: "",
        icon: "",
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        catId: null,
      },
      dialogVisible: false,
      expandedKeys: [],
      data: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
    };
  },
  activated() {
    this.getDataList();
  },
  methods: {
    handleNodeClick(data) {
      console.log(data);
    },
    batchDelete() {
      var cIds = [];
      let checkedNodes = this.$refs.menuTree.getCheckedNodes();
      console.log("被选中的节点", checkedNodes);
      for(let i = 0; i < checkedNodes.length; i++){
        cIds.push(checkedNodes[i].catId);
      }
      console.log("cIds",cIds);
      this.$confirm(`是否批量删除选择的节点?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
           this.$http({
           url: this.$http.adornUrl('/product/category/delete'),
           method: 'post',
           data: this.$http.adornData(cIds, false)
           }).then(({ data }) => {
            this.$message({
                message: `菜单删除成功`,
                type: "success",
              });
            });
          }).then(() => {
              this.getDataList();
              this.getDataList();
              //设置默认张开的菜单
            }).catch(() => {});
      
    },
    batchSave() {
      this.$http({
        url: this.$http.adornUrl("/product/category/update/sort"),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false),
      })
        .then(({ data }) => {
          this.$message({
            message: `菜单顺序修改成功`,
            type: "success",
          });
        })
        .then(() => {
          this.getDataList();
        });
      this.updateNodes = [];
      this.maxLength = 1;

      //设置默认张开的菜单
      // this.expandedKeys = [this.pCid];
      // this.pCid = 0;
    },
    handleDrop(draggingNode, dropNode, dropType, ev) {
      let pCid = 0;
      let sibling = null;
      if (dropType == "before" || dropType == "after") {
        pCid =
          dropNode.parent.data.catId == undefined
            ? 0
            : dropNode.parent.data.catId;
        sibling = dropNode.parent.childNodes;
      } else {
        pCid = dropNode.data.catId;
        sibling = dropNode.childNodes;
      }

      for (let i = 0; i < sibling.length; i++) {
        if (sibling[i].data.catId == draggingNode.data.catId) {
          let catLevel = draggingNode.level;
          if (sibling[i].level != draggingNode.level) {
            catLevel = sibling[i].level;
            this.updateChildNodeLevel(sibling[i]);
          }
          this.updateNodes.push({
            catId: sibling[i].data.catId,
            sort: i,
            parentCid: pCid,
            catLevel: catLevel,
          });
        } else {
          this.updateNodes.push({ catId: sibling[i].data.catId, sort: i });
        }
      }
      this.expandedKeys.push(pCid);

      console.log("handleDrop", this.updateNodes);
    },
    updateChildNodeLevel(node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var cNode = node.childNodes[i].data;
          this.updateNodes.push({
            catId: cNode.catId,
            catLevel: node.childNodes[i].level,
          });
          this.updateChildNodeLevel(node.childNodes[i]);
        }
      }
    },
    submitData() {
      if (this.dialogType == "edit") this.editCategory();
      if (this.dialogType == "append") this.addCategory();
    },
    countNodeLevel(node) {
      if (node.childrenNodes != null && node.childrenNodes.length > 0) {
        for (let i = 0; i < node.childrenNodes.length; i++) {
          if (node.childrenNodes[i].level > this.maxLength) {
            this.maxLength = node.childrenNodes[i].level;
          }
          this.countNodeLevel(node.childrenNodes[i]);
        }
      }
    },
    allowDrop(draggingNode, dropNode, type) {
      // console.log("allowDrop", draggingNode, dropNode, type);
      var level = this.countNodeLevel(draggingNode.data);
      let deep = Math.abs(this.maxLength - draggingNode.level + 1);
      console.log("deep", deep);
      if (type == "inner") return deep + dropNode.level <= 3;
      else return deep + dropNode.parent.level <= 3;
    },
    editCategory() {
      var { catId, name, productUnit, icon } = this.category;
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData({ catId, name, productUnit, icon }, false),
      })
        .then(({ data }) => {
          this.$message({
            message: `菜单修改成功`,
            type: "success",
          });
        })
        .then(() => {
          this.getDataList();
        });
      this.dialogVisible = false;

      //设置默认张开的菜单
      this.expandedKeys = [this.category.parentCid];
    },

    edit(data) {
      this.dialogTitle = "修改分类";
      this.dialogType = "edit";
      this.dialogVisible = true;

      /*
      *存在风险：若两人同时操作该数据，一个将该数据删除了，那么另一个人不应该可以修改该数据
      *解决方式：重新获取一遍表单数据
      this.category.name = data.name;
      this.category.catId = data.catId;
    */
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
      }).then(({ data }) => {
        console.log("返回的数据", data);
        this.category.name = data.data.name;
        this.category.catId = data.data.catId;
        this.category.icon = data.data.icon;
        this.category.productUnit = data.data.productUnit;
        this.category.parentCid = data.data.parentCid;
        this.category.catLevel = data.data.catLevel;
        this.category.sort = data.data.sort;
        this.category.showStatus = data.data.showStatus;
      });
    },
    addCategory() {
      console.log("提交数据", this.category);
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      })
        .then(({ data }) => {
          this.$message({
            message: `菜单保存成功`,
            type: "success",
          });
        })
        .then(() => {
          this.dialogVisible = false;
        })
        .then(() => {
          this.getDataList();
          //设置默认张开的菜单
          this.expandedKeys = [this.category.parentCid];
        });
    },
    getDataList() {
      this.dataListLoading = true;
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        console.log("成功获取数据", data.data);
        this.data = data.data;
      });
    },
    append(data) {
      this.category.name = "";
      this.dialogTitle = "添加分类";
      this.dialogType = "append";
      console.log("append", data);
      this.dialogVisible = true;
      this.category.parentCid = data.catId;
      this.category.catLevel = data.catLevel + 1;
    },

    remove(node, data) {
      this.$confirm(`是否删除【${data.name}】菜单, 是否继续?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          var ids = [data.catId];
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false),
          })
            .then(({ data }) => {
              this.$message({
                message: `菜单删除成功`,
                type: "success",
              });
            })
            .then(() => {
              this.getDataList();
              //设置默认张开的菜单
              this.expandedKeys = [node.parent.data.catId];
            });
        })
        .catch(() => {});
    },
  },
};
</script>

<style>
</style>