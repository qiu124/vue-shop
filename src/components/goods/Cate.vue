<template>
  <div>
    <!-- 面包屑导航 -->
    <el-breadcrumb separator-class="el-icon-arrow-right">
      <el-breadcrumb-item :to="{ path: '/home' }">首页</el-breadcrumb-item>
      <el-breadcrumb-item>商品管理</el-breadcrumb-item>
      <el-breadcrumb-item>商品分类</el-breadcrumb-item>
    </el-breadcrumb>
    <!-- 卡片试图 -->
    <el-card>
      <!-- 添加按钮 -->
      <el-row>
        <el-col>
          <el-button type="primary" @click="showAddDialog">添加分类</el-button>
        </el-col>
      </el-row>
      <!-- tree-table表格 -->
      <tree-table :data="cateList" :columns="columns" :selection-type="false" :expand-type="false" show-index border>
        <!-- 是否有效 -->
        <template slot="isOk" slot-scope="scope">
          <i class="el-icon-error" style="color: red" v-if="scope.row.cat_deleted"></i>
          <i class="el-icon-success" style="color: lightgreen" v-else></i>
        </template>
        <!-- 排序 -->
        <template slot="order" slot-scope="scope">
          <el-tag size="mini" v-if="scope.row.cat_level === 0">一级</el-tag>
          <el-tag size="mini" type="success" v-else-if="scope.row.cat_level === 1">二级</el-tag>
          <el-tag size="mini" type="warning" v-else>三级</el-tag>
        </template>
        <!-- 操作 -->
        <template slot="operate">
          <el-button size="mini" type="primary" class="el-icon-edit">编辑</el-button>
          <el-button size="mini" type="danger" class="el-icon-delete">删除</el-button>
        </template>
      </tree-table>

      <!-- 分页 -->
      <el-pagination
        @size-change="handleSizeChange"
        @current-change="handleCurrentChange"
        :current-page="queryInfo.pagenum"
        :page-sizes="[3, 5, 10, 15]"
        :page-size="queryInfo.pageisze"
        layout="total, sizes, prev, pager, next, jumper"
        :total="total"
      >
      </el-pagination>
    </el-card>

    <!-- 添加分类的对话框 -->
    <el-dialog title="添加分类" :visible.sync="addDialogVisible" width="50%" @close="addDialogClosed">
      <el-form :model="addForm" :rules="addFormRules" ref="addFormRef" label-width="100px">
        <el-form-item label="分类名称" prop="cat_name">
          <el-input v-model="addForm.cat_name"></el-input>
        </el-form-item>
        <el-form-item label="父级分类">
          <el-cascader v-model="selectedKeys" :options="parentCateList" :props="cateProps" @change="handleChange" clearable></el-cascader>
        </el-form-item>
      </el-form>

      <span slot="footer" class="dialog-footer">
        <el-button @click="addDialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="addCate">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
export default {
  data() {
    return {
      //查询分类列表的参数对象
      queryInfo: {
        type: 3,
        pagenum: 1,
        pagesize: 5,
      },
      //分类列表
      cateList: [],
      //总记录条数
      total: 0,
      //tree-table表格的列定义
      columns: [
        {
          label: '分类名称',
          prop: 'cat_name',
        },
        {
          label: '是否有效',
          //将当前列定义为模板列
          type: 'template',
          //当前使用的模板名称
          template: 'isOK',
        },
        {
          label: '排序',
          //将当前列定义为模板列
          type: 'template',
          template: 'order',
        },
        {
          label: '操作',
          //将当前列定义为模板列
          type: 'template',
          template: 'operate',
        },
      ],
      addDialogVisible: false,
      addForm: {
        cat_name: '', //分类的名称
        cat_pid:0,//父级分类的id
        cat_level:0,//分类的等级,默认为一级

      },
      addFormRules: {
        cat_name: [{ required: true, message: '请输入分类名称', trigger: 'blur' }],
      },
      //父级分类列表
      parentCateList: [],
      //级联选择器的配置对象
      cateProps: {
        label: 'cat_name',
        value: 'cat_id',
        children: 'children',
        expandTrigger: 'hover',
        checkStrictly: true, //选择任意一个选项
      },
      //级联选择器选择的id数组
      selectedKeys: [],
    }
  },
  created() {
    this.getCateList()
  },
  methods: {
    async getCateList() {
      const { data: res } = await this.$http.get('categories', {
        params: this.queryInfo,
      })
      if (res.meta.status !== 200) {
        return this.$message.error('获取商品列表失败')
      }
      // console.log(res)
      this.cateList = res.data.result
      this.total = res.data.total
    },
    handleSizeChange(pagesize) {
      this.queryInfo.pagesize = pagesize
      this.getCateList()
    },
    handleCurrentChange(pagenum) {
      this.queryInfo.pagenum = pagenum
      this.getCateList()
    },
    //获取父级分类的数据列表
    async getParentCateList() {
      const { data: res } = await this.$http.get('categories', {
        params: { type: 2 },
      })
      if (res.meta.status !== 200) {
        return this.$message.error('获取父级分类数据失败')
      }
      //  console.log(res.data)
      this.parentCateList = res.data
    },
    //显示添加分类的对话框
    showAddDialog() {
      this.getParentCateList()
      this.addDialogVisible = true
    },
    //监听级联选择器的改变事件
    handleChange() {
      // console.log(this.selectedKeys, '-------------')
      //判断是否选择了父级分类
      if(this.selectedKeys.length>0){
        this.addForm.cat_pid = this.selectedKeys[this.selectedKeys.length-1]
        this.addForm.cat_level = this.selectedKeys.length
      }else{
         this.addForm.cat_pid = 0
        this.addForm.cat_level = 0
      }
    },
    //添加分类
    addCate(){
      this.$refs.addFormRef.validate(async valid => {
        if(!valid){
          return
        }
        const {data:res} = await this.$http.post('categories',this.addForm)
        if (res.meta.status !== 201) {
           return this.$message.error('获取分类失败');
         }
         this.addDialogVisible = false
         this.getCateList()
         this.$message.success('添加分类成功!')
      })
    },
    //监听添加对话框的关闭事件
    addDialogClosed(){
      this.$refs.addFormRef.resetFields()
      this.selectedKeys = []
      this.addForm.cat_pid = 0
      this.addForm.cat_level = 0
    }

  },
}
</script>

<style lang="less" scoped>
.el-cascader {
  width: 100%;
}
.zk-table {
  margin-top: 15px;
}
</style>