<template>
  <div class="user-manage">
    <!--    <h3>待审批</h3>-->
    <div class="query-form">
      <el-form :inline="true" :model="queryForm" ref="form">
        <el-form-item label="审批状态" prop="applyState">
          <el-select v-model="queryForm.applyState">
            <el-option :value="''" label="全部"></el-option>
            <el-option :value="1" label="待审批"></el-option>
            <el-option :value="2" label="审批中"></el-option>
            <el-option :value="3" label="审批拒绝"></el-option>
            <el-option :value="4" label="审批通过"></el-option>
            <el-option :value="5" label="作废"></el-option>
          </el-select>
        </el-form-item>
        <el-form-item>
          <el-button type="primary" @click="getApplyList">查询</el-button>
          <el-button @click="handleReset('form')">重置</el-button>
        </el-form-item>
      </el-form>
    </div>
    <div class="base-table">
      <div class="action">
      </div>
      <el-table
          :data="applyList">
        <el-table-column type="selection" width="55"/>
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
            width="150">
          <template #default="scope">
            <el-button type="primary" @click="handleDetail(scope.row)"
                       v-if="scope.row.curAuditUserName === userInfo.userName && [1,2].includes(scope.row.applyState)"
            >审核
            </el-button>
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
    <el-dialog title="审核" width="50%" v-model="showDetailModal" destroy-on-close>
      <el-form ref="dialogForm" :model="auditForm" label-width="120px" label-suffix=":" :rules="rules">
        <el-form-item label="申请人">
          <div>{{ detail.applyUser.userName }}</div>
        </el-form-item>
        <el-form-item label="休假时间">
          <div>{{ detail.time }}</div>
        </el-form-item>
        <el-form-item label="休假时长">
          <div>{{ detail.leaveTime }}</div>
        </el-form-item>
        <el-form-item label="休假原因">
          <div>{{ detail.reasons }}</div>
        </el-form-item>
        <el-form-item label="审批状态">
          <div>{{ detail.applyStateName }}</div>
        </el-form-item>
        <el-form-item label="审批人">
          <div>{{ detail.curAuditUserName }}</div>
        </el-form-item>
        <el-form-item label="备注" prop="remark">
          <el-input type="textarea" :rows="3" placeholder="请输入审核备注" v-model="auditForm.remark">
          </el-input>
        </el-form-item>
      </el-form>
      <template #footer>
        <span class="dialog-footer">
            <el-button @click="handleApprove('pass')">审核通过</el-button>
            <el-button type="primary" @click="handleApprove('refuse')">驳回</el-button>
        </span>
      </template>
    </el-dialog>
  </div>
</template>

<script lang="ts" setup>
import {getCurrentInstance, onMounted, reactive, ref, toRaw} from 'vue';
import api from '../api/index.js';
import utils from '../utils/utils.js';
import {useStore} from 'vuex';

const {proxy} = getCurrentInstance(); // ctx调用全局会有问题, 通过proxy来调用全局方法属性

const queryForm = reactive({
  applyState: 1
});
const showDetailModal = ref(false);
const active = ref(0);
// 初始化分页
const pager = reactive({
  pageNum: 1,
  pageSize: 10,
  total: 100
});
// 初始化用户列表
const userList = ref([]);
const detail = ref({});

// 初始化接口调用
onMounted(() => {
  getApplyList();
});


const userInfo = proxy.$store.state.userInfo;
const store = useStore();
const applyList = ref([]);
const auditForm = reactive({
  remark: ''
});
const getApplyList = async () => {
  let params = {...queryForm, ...pager, type: 'approve'};
  let {list, page} = await api.getApplyList(params);
  applyList.value = list;
  pager.total = page.total;
};

const handleApprove = (action) => {
  proxy.$refs.dialogForm.validate(async (valid) => {
    if (valid) {
      let params = {action, remark: auditForm.remark, _id: detail.value._id};
      try {
        await proxy.$api.leaveApprove(params);
        handleClose();
        proxy.$message.success('处理成功');
        await getApplyList();
        store.commit('saveNoticeCount', store.state.noticeCount - 1);
      } catch (error) {

      }
    }
  });
};

// 用户弹窗关闭
const handleClose = () => {
  showDetailModal.value = false;
  handleReset('dialogForm');
};

// 查询事件，获取用户列表
const handleQuery = () => {

};

// 选中用户列表的对象
const checkedUserIds = ref([]);
// 批量删除
const handlePatchDel = async (row) => {
  if (checkedUserIds.value.length === 0) {
    proxy.$message.error('请选择要删除的用户');
    return;
  }
  const res = await proxy.$api.userDel({
    userIds: checkedUserIds.value
  });
  if (res.nModified > 0) {
    proxy.$message.success('删除成功');
    await getUserList();
  } else {
    proxy.$message.success('修改失败');
  }
};


// 重置查询表单
const handleReset = (form) => {
  proxy.$refs[form].resetFields();
};

// 分页事件处理
const handleCurrentChange = (current) => {
  pager.pageNum = current;
  getUserList();
};
// 新增用户
const userForm = reactive({
  state: 3
});


// 定义表单校验规则
const rules = {
  remark: [
    {required: true, message: '请输入审核备注', trigger: 'change'}
  ],
};

// 所有的部门列表
const deptList = ref([]);
// 所有的角色列表
const roleList = ref([]);
// 获取所有的部门
const getDeptList = async () => {
  let list = await proxy.$api.getDeptList();
  deptList.value = list;
};


const handleDetail = (row) => {
  let data = {...row};
  data.applyTypeName = {
    1: '事假',
    2: '调休',
    3: '年假'
  }[data.applyType];
  data.time = utils.formatDate(new Date(data.startTime), 'yyyy-MM-dd') + '到'
      + utils.formatDate(new Date(data.endTime), 'yyyy-MM-dd');
  data.applyStateName = {
    1: '待审批',
    2: '审批中',
    3: '审批拒绝',
    4: '审批通过',
    5: '作废'
  }[data.applyState];
  detail.value = data;
  showDetailModal.value = true;
};

// 定义动态表格头
const columns = reactive([
  {
    label: '单号', prop: 'orderNo'
  },
  {
    label: '申请人', prop: '',
    formatter(row, column, value) {
      return row.applyUser.userName;
    }
  },
  {
    label: '休假时间', prop: '', formatter(row, column, value) {
      return utils.formatDate(new Date(row.startTime), 'yyyy-MM-dd') + '到'
          + utils.formatDate(new Date(row.endTime), 'yyyy-MM-dd');
    }
  },
  {
    label: '休假时长', prop: 'leaveTime'
  },
  {
    label: '休假类型', prop: 'applyType',
    formatter(row, column, value) {
      return {
        1: '事假',
        2: '调休',
        3: '年假'
      }[value];
    }
  },
  {
    label: '休假原因', prop: 'reasons'
  },
  {
    label: '申请时间', prop: 'createTime', width: 180, formatter(row, column, value) {
      return utils.formatDate(new Date(value));
    }
  },
  {
    label: '审批人', prop: 'auditUsers', width: 180
  },
  {
    label: '当前审批人', prop: 'curAuditUserName', width: 180
  },
  {
    label: '审批状态', prop: 'applyState', width: 180,
    formatter(row, column, value) {
      return {
        1: '待审批',
        2: '审批中',
        3: '审批拒绝',
        4: '审批通过',
        5: '作废'
      }[value];
    }
  },
]);
</script>

<style>
.center {
  display: flex;
  justify-content: center;
}
</style>
