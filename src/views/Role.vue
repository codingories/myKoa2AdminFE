<template>
  <div class="role-manage">
    <!--    <h3>角色管理</h3>-->
    <!--    {{actionMap}}-->
    <div class="query-form">
      <el-form :inline="true" :model="queryForm" ref="queryForm">
        <el-form-item label="角色名称" prop="roleName">
          <el-input v-model="queryForm.roleName" placeholder="请输入角色名称"></el-input>
        </el-form-item>
        <el-form-item>
          <el-button type="primary" @click="getRoleList">查询</el-button>
          <el-button @click="handleReset('queryForm')">重置</el-button>
        </el-form-item>
      </el-form>
    </div>
    <div class="base-table">
      <div class="action">
        <el-button type="primary" @click="handleAdd">创建</el-button>
      </div>
      <el-table
          :data="roleList"
      >
        <el-table-column
            v-for="(item) in columns"
            :key="item.prop"
            :prop="item.prop"
            :label="item.label"
            :width="item.width"
            :formatter="item.formatter"
        >
        </el-table-column>
        <el-table-column
            label="操作"
            width="220">
          <template #default="scope">
            <el-button @click="handleEdit(scope.row)">编辑</el-button>
            <el-button type="primary" @click="handleOpenPermission(scope.row)">设置权限</el-button>
            <el-button type="danger" size="small" @click="handleDel(scope.row._id)">删除</el-button>
          </template>
        </el-table-column>
      </el-table>
      <el-pagination
          class="pagination"
          background
          layout="prev, pager, next, jumper"
          :total="pager.total"
          :page-size="pager.pageSize"
          @current-change="handleCurrentChange"
      />
    </div>
    <el-dialog title="用户新增" v-model="showModal">
      <el-form
          ref="dialogForm"
          :model="roleForm"
          label-width="100px"
          :rules="rules"
      >
        <el-form-item label="角色名称" prop="roleName">
          <el-input v-model="roleForm.roleName" placeholder="请输入角色名称"/>
        </el-form-item>
        <el-form-item label="备注" prop="remark">
          <el-input
              type="textarea"
              :rows="2"
              v-model="roleForm.remark"
              placeholder="请输入备注"
          />
        </el-form-item>
      </el-form>
      <template #footer>
        <span class="dialog-footer">
          <el-button @click="handleClose">取 消</el-button>
          <el-button type="primary" @click="handleSubmit">确 定</el-button>
        </span>
      </template>
    </el-dialog>
    <!--权限弹框-->
    <el-dialog v-model="showPermission" title="用户新增">
      <el-form ref="dialogForm" label-width="100px">
        <el-form-item label="角色名称">
          {{ curRoleName }}
        </el-form-item>
        <el-form-item label="选择权限">
          <el-tree :data="menuList"
                   ref="tree"
                   default-expand-all
                   show-checkbox
                   node-key="_id"
                   :props="{label: 'menuName'}"
          />

        </el-form-item>
      </el-form>
      <template #footer>
          <span class="dialog-footer">
              <el-button @click="handlePermissionClose">取 消</el-button>
              <el-button type="primary" @click="handlePermissionSubmit">确 定</el-button>
          </span>
      </template>
    </el-dialog>
  </div>
</template>

<script lang=js>
import utils from "../utils/utils.js"

export default {
  name: 'role',
  data() {
    return {
      pager: {
        pageNum: 1,
        pageSize: 10,
        total: 0
      },
      curRoleName: '',
      showPermission: false,
      queryForm: {
        roleName: ""
      },
      roleForm: {
        roleName: ''
      },
      showModal: false,
      roleList: [],
      rules: {
        roleName: [
          {
            required: true,
            message: '请输入角色的名称'
          }
        ]
      },
      curRoleId: "",
      action: '',
      actionMap: {},
      columns: [
        {
          label: '角色名称',
          prop: 'roleName',
          width: 150
        },
        {
          label: '备注',
          prop: 'remark',
        },
        {
          label: '权限列表',
          prop: 'permissionList',
          formatter: (row, column, value) => {
            let names = []
            let list = value.halfCheckedKeys || []
            list.map(key => {
              let name = this.actionMap[key]
              if (key && name) names.push(this.actionMap[key])
            })
            return names.join(',')
          }
        },
        {
          label: '创建时间',
          prop: 'createTime',
          width: 180,
          formatter(row, column, value) {
            return utils.formatDate(new Date(value));
          }
        },
      ],
      // 菜单映射表

    }
  },
  mounted() {
    this.getRoleList()
    this.getMenuList();
  },
  methods: {
    handleAdd() {
      this.action = 'create'
      this.showModal = true
    },
    // 获取列表初始化
    async getMenuList() {
      try {
        let list = await this.$api.getMenuList()
        this.menuList = list
        this.getActionMap(list)
      } catch (e) {
        console.error(e)
        throw new Error(e)
      }
    },
    getActionMap(list) {
      let actionMap = {}
      const deep = (arr) => {
        while (arr.length) {
          let item = arr.pop()
          if (item.children && item.action) {
            actionMap[item._id] = item.menuName
          }
          if (item.children && !item.action) {
            deep(item.children)
          }
        }
      }
      deep(JSON.parse(JSON.stringify(list)))
      this.actionMap = actionMap
    },
    handleOpenPermission(row) {
      this.curRolName = row.roleName
      this.curRoleId = row._id
      this.showPermission = true
      let {checkedKeys} = row.permissionList
      console.log('checkedKeys => ', checkedKeys)
      setTimeout(() => {
        this.$refs.tree.setCheckedKeys(checkedKeys);
      })
    },
    handleEdit(row) {
      this.action = 'edit'
      this.showModal = true
      this.$nextTick(() => {
        this.roleForm = {
          _id: row._id,
          roleName: row.roleName,
          remark: row.remark
        }
      })
    },
    handlePermissionClose() {
      this.showPermission = false;
      this.handleReset('dialogForm');
    },
    async getRoleList() {
      try {
        let {list, page} = await this.$api.getRoleList({...this.queryForm, ...this.pager})
        this.roleList = list
        this.pager.total = page.total
      } catch (e) {
        console.error(e)
        throw new Error(e)
      }
    },
    handleQuery() {
      this.getMenuList()
    },
    handleReset(form) {
      this.$refs[form].resetFields()
    },
    handleCurrentChange(current) {
      this.pager.pageNum = current
      this.getRoleList()
    },
    async handleDel(_id) {
      await this.$api.roleOperate({_id, action: 'delete'})
      this.$message.success('删除成功')
      this.getRoleList()
    },
    // 角色提交
    handleSubmit() {
      this.$refs.dialogForm.validate(async (valid) => {
        if (valid) {
          let {roleForm, action} = this
          let params = {...roleForm, action}
          let res = await this.$api.roleOperate(params)
          if (res) {
            this.showModal = false
            this.$message.success('创建成功')
            this.handleReset('dialogForm')
            this.getRoleList()
          }
        }
      })
    },
    async handlePermissionSubmit() {
      let nodes = this.$refs.tree.getCheckedNodes();
      let halfKeys = this.$refs.tree.getHalfCheckedKeys();
      let checkedKeys = []
      let parentKeys = []
      nodes.map(node => {
        if (!node.children) {
          checkedKeys.push(node._id);
        } else {
          parentKeys.push(node._id)
        }
      })
      let params = {
        _id: this.curRoleId,
        permissionList: {
          checkedKeys,
          halfCheckedKeys: parentKeys.concat(halfKeys)
        }
      }
      await this.$api.updatePermission(params)
      this.showPermission = false
      this.$message.success('设置成功')
      this.getRoleList();
    },
    handleClose() {
      this.showModal = false;
      this.handleReset('dialogForm');
    },

  }
}
</script>

<style>

</style>
