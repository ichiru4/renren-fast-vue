<!--  -->
<template>
    <div>
        <el-switch v-model="draggable" active-text="开启拖拽" inactive-text="关闭拖拽">
        </el-switch>
        <el-button v-if="draggable" @click="batchSave">
            批量保存
        </el-button>
        <el-button type="danger" @click="batchDelete">
            批量删除
        </el-button>
        <el-tree :data="menus" :props="defaultProps" show-checkbox node-key="catId" :expand-on-click-node='false'
            :default-expanded-keys="expandedKey" :draggable="draggable" :allow-drop="allowDrop" @node-drop="handleDrop"
            ref="menuTree">
            <span class="custom-tree-node" slot-scope="{ node, data }">
                <span>{{ node.label }}</span>
                <span>
                    <el-button v-if="node.level <= 2" type="text" size="mini" @click="() => append(data)">
                        Append
                    </el-button>

                    <el-button type="text" size="mini" @click="edit(data)">
                        edit
                    </el-button>

                    <el-button v-if="node.childNodes.length == 0" type="text" size="mini"
                        @click="() => remove(node, data)">
                        Delete
                    </el-button>
                </span>
            </span>
        </el-tree>
        <el-dialog :title="title" :visible.sync="dialogVisible" width="30%" :close-on-click-modal="false">
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


//import { node } from 'webpack';

//这里可以导入其他文件（比如：组件，工具js，第三方插件js，json文件，图片文件等等）
//例如：import 《组件名称》 from '《组件路径》';

export default {
    //import引入的组件需要注入到对象中才能使用
    components: {},
    data() {
        return {
            pCid: [],
            draggable: false,
            updateNodes: [],
            maxlevel: 0,
            title: "",
            dialogType: "",
            category: { name: '', parentCid: 0, catLevel: 0, showStatus: 1, sort: 0, catId: null, icon: "", productUnit: '' },
            dialogVisible: false,
            expandedKey: [],
            menus: [],
            defaultProps: {
                children: 'children',
                label: 'name'
            }
        };
    },
    methods: {
        getMenus() {
            this.$http({
                url: this.$http.adornUrl('/product/category/list/tree'),
                method: 'get'
            }).then(({ data }) => {
        
                this.menus = data.data
            })
        },
        batchDelete() {
            let catId = [];
            let checkedNodes = this.$refs.menuTree.getCheckedNodes();
            for (let i = 0; i < checkedNodes.length; i++) {
                catId.push(checkedNodes[i].catId);
            }
            this.$confirm(`是否批量删除【${catId}】菜单?`, '提示', {
                confirmButtonText: '确定',
                cancelButtonText: '取消',
                type: 'warning'
            }).then(() => {
                this.$http({
                    url: this.$http.adornUrl('/product/category/delete'),
                    method: 'post',
                    data: this.$http.adornData(catId, false)
                }).then(({ data }) => {
                    this.$message({
                        message: '菜单批量删除成功',
                        type: 'success'
                    });
                    this.getMenus();
                });
            }).catch(() => {

            })

        },
        batchSave() {
            this.$http({
                url: this.$http.adornUrl('/product/category/update/sort'),
                method: 'post',
                data: this.$http.adornData(this.updateNodes, false)
            }).then(({ data }) => {
                this.$message({
                    message: '菜单顺序修改成功',
                    type: 'success'
                });
                this.getMenus();
                //设置默认展开的菜单
                this.expandedKey = this.pCid;
                this.updateNodes = [];
                this.maxlevel = 0;
                this.pCid = 0;
            });
        },
        handleDrop(draggingNode, dropNode, dropType, ev) {
            console.log('tree drop: ', draggingNode, dropNode, dropType);
            //当前父节点Id
            let pCid = 0;
            let siblings = null;
            if (dropType == 'before' || dropType == 'after') {
                pCid = dropNode.parent.data.catId == undefined ? 0 : dropNode.parent.data.catId;
                siblings = dropNode.parent.childNodes;
            } else {
                pCid = dropNode.data.catId;
                siblings = dropType.childNodes
            }

            this.pCid.push(pCid);
            //当前拖拽节点的最新顺序
            for (let i = 0; i < siblings.length; i++) {
                //如果遍历的是正在拖拽的节点
                if (siblings[i].data.catId == draggingNode.data.catId) {
                    let catLevel = draggingNode.level;
                    if (siblings[i].level != draggingNode.level) {
                        //当前节点层级发生变化
                        catLevel = siblings[i].level;
                        //修改子节点层级
                        this.updateChildNodelevel(siblings[i]);
                    }
                    this.updateNodes.push({ catId: siblings[i].data.catId, sort: i, parentCid: pCid, catLevel: catLevel });
                } else {
                    this.updateNodes.push({ catId: siblings[i].data.catId, sort: i });
                }

            }

            //当前拖拽节点的最新层级
            console.log("updateNodes", this.updateNodes)

        },
        updateChildNodelevel(node) {
            if (node.childNodes.length > 0) {
                for (let i = 0; i < node.childNodes.length; i++) {
                    var cNode = node.childNodes[i].data;
                    this.updateNodes.push({ catId: cNode.catId, catLevel: node.childNodes[i].level });
                    this.updateChildNodelevel(node.childNodes[i]);
                }
            }

        },
        allowDrop(draggingNode, dropNode, type) {
            //被拖动的当前节点以及所在的父节点总层数不能大于3
            this.countNodeLevel(draggingNode);
            let deep = Math.abs(this.maxlevel - draggingNode.level) + 1;
            if (type == "inner") {
                return (deep + dropNode.level) <= 3;
            } else {
                return (deep + dropNode.parent.level) <= 3;
            }

        },
        countNodeLevel(node) {
            if (node.childNodes != null && node.childNodes.length > 0)
                for (let i = 0; i < node.childNodes.length; i++) {
                    if (node.childNodes[i].Level > this.maxlevel) {
                        this.maxlevel = node.childNodes[i].level;
                    }
                    else {
                        this.maxlevel = node.level;
                    }
                    this.countNodeLevel(node.childNodes[i]);
                }
        },
        edit(data) {
            console.log("要修改的数据", data);
            this.dialogType = "edit";
            this.title = "修改分类";
            this.dialogVisible = true;

            //发送请求获取当前节点最新数据
            this.$http({
                url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
                method: 'get',
            }).then(({ data }) => {
                console.log("回显数据", data)
                this.category.name = data.data.name;
                this.category.catId = data.data.catId;
                this.category.icon = data.data.icon;
                this.category.productUnit = data.data.productUnit;
                this.category.parentCid = data.data.parentCid;
            })

        },

        append(data) {
            console.log("append", data);
            this.dialogType = "add";
            this.title = "添加分类";
            this.dialogVisible = true;
            this.category.parentCid = data.catId;
            this.category.catLevel = data.catLevel * 1 + 1;
            this.category.name = "";
            this.category.catId = null;
            this.category.icon = "";
            this.category.productUnit = "";
            this.category.parentCid = 0;
            this.category.sort = 0;
            this.category.showStatus = 1;
        },
        submitData() {
            if (this.dialogType == 'add') {
                this.addCatogory();
            }
            if (this.dialogType == 'edit') {
                this.editCategory();
            }
        },

        //修改三级分类数据
        editCategory() {
            var { catId, name, icon, productUnit } = this.category;
            this.$http({
                url: this.$http.adornUrl('/product/category/update'),
                method: 'post',
                data: this.$http.adornData({ catId, name, icon, productUnit }, false)
            }).then(({ data }) => {
                this.$message({
                    message: '菜单修改成功',
                    type: 'success'
                })
                //关闭对话框
                this.dialogVisible = false;
                //刷新新的菜单
                this.getMenus();
                //设置默认展开的菜单
                this.expandedKey = [this.category.parentCid];
            });
        },

        //添加三级分类
        addCatogory() {
            console.log("三级数据", this.category);
            this.$http({
                url: this.$http.adornUrl('/product/category/save'),
                method: 'post',
                data: this.$http.adornData(this.category, false)
            }).then(({ data }) => {
                this.$message({
                    message: '菜单保存成功',
                    type: 'success'
                });
                //关闭对话框
                this.dialogVisible = false;
                //刷新新的菜单
                this.getMenus();
                //设置默认展开的菜单
                this.expandedKey = [this.category.parentCid];
            });
        },

        remove(node, data) {
            var ids = [data.catId]

            this.$confirm(`是否删除【${data.name}】菜单?`, '提示', {
                confirmButtonText: '确定',
                cancelButtonText: '取消',
                type: 'warning'
            }).then(() => {
                this.$http({
                    url: this.$http.adornUrl('/product/category/delete'),
                    method: 'post',
                    data: this.$http.adornData(ids, false)
                }).then(({ data }) => {
                    this.$message({
                        message: '菜单删除成功',
                        type: 'success'
                    });
                    this.getMenus();

                    this.expandedKey = [node.parent.data.catId]
                });
            }).catch(() => {

            });


            console.log("remove", node, data)
        }


    },

    //监听属性 类似于data概念
    computed: {},
    //监控data中的数据变化
    watch: {},
    //方法集合

    //生命周期 - 创建完成（可以访问当前this实例）
    created() {
        this.getMenus();
    },
    //生命周期 - 挂载完成（可以访问DOM元素）
    mounted() {

    },
    beforeCreate() { }, //生命周期 - 创建之前
    beforeMount() { }, //生命周期 - 挂载之前
    beforeUpdate() { }, //生命周期 - 更新之前
    updated() { }, //生命周期 - 更新之后
    beforeDestroy() { }, //生命周期 - 销毁之前
    destroyed() { }, //生命周期 - 销毁完成
    activated() { }, //如果页面有keep-alive缓存功能，这个函数会触发
}
</script>
<style scoped></style>