<template>
  <div class="web-user">
    <el-card class="box-card">
      <h2 class="page-title">用户管理</h2>

      <!-- 搜索框和按钮 -->
      <el-row class="search-box" gutter="20" type="flex" justify="start" align="middle">
        <el-col :span="4">
          <el-input v-model="searchQuery.username" placeholder="用户名" size="medium" />
        </el-col>
        <el-col :span="4">
          <el-input v-model="searchQuery.realName" placeholder="真实姓名" size="medium" />
        </el-col>
        <el-col :span="4">
          <el-select v-model="searchQuery.identity" placeholder="选择角色" size="medium">
            <el-option label="校长" value="0" />
            <el-option label="教务处" value="1" />
            <el-option label="班主任" value="2" />
            <el-option label="教师" value="3" />
          </el-select>
        </el-col>
        <el-col :span="4">
          <el-select v-model="searchQuery.gender" placeholder="选择性别" size="medium">
            <el-option label="男" value="1" />
            <el-option label="女" value="0" />
          </el-select>
        </el-col>
        <el-col :span="1.5">
          <el-button type="primary" @click="searchUsers">搜索</el-button>
        </el-col>
        <el-col :span="1.5">
          <el-button @click="resetSearch">重置</el-button>
        </el-col>
        <el-col :span="1.5">
          <el-button type="success" @click="openAddUserDialog">添加用户</el-button>
        </el-col>
      </el-row>

      <!-- 用户列表 -->
      <el-table :data="users.list" border stripe>
        <el-table-column label="用户名" prop="username" />
        <el-table-column label="真实姓名" prop="realName" />
        <el-table-column label="手机号码" prop="phoneNumber" />
        <el-table-column label="性别" prop="gender" :formatter="genderFormatter" />
        <el-table-column label="角色" prop="identity" :formatter="identityFormatter" />
        <el-table-column label="学校" :formatter="schoolNameFormatter" />
        <el-table-column label="操作">
          <template #default="{ row }">
            <el-button size="small" @click="editUser(row)">编辑</el-button>
            <el-button type="danger" size="small" @click="confirmDeleteUser(row.id)">删除</el-button>
          </template>
        </el-table-column>
      </el-table>

      <!-- 分页 -->
      <el-pagination
        :current-page="users.curPage"
        :page-size="users.pageSize"
        :total="users.total"
        layout="prev, pager, next, jumper"
        @current-change="changePage"
      />
    </el-card>

    <!-- 添加/编辑用户弹窗 -->
    <el-dialog :visible.sync="showAddUserDialog" width="50%" @close="closeAddUserDialog">
      <h3>{{ editingUser ? '编辑用户' : '添加用户' }}</h3>
      <el-form :model="newUser" ref="form" label-width="100px" :rules="rules">
        <el-form-item label="用户名" prop="username">
          <el-input v-model="newUser.username" />
        </el-form-item>
        <el-form-item v-if="!editingUser" label="密码" prop="password">
          <el-input v-model="newUser.password" type="password" />
        </el-form-item>
        <el-form-item label="真实姓名" prop="realName">
          <el-input v-model="newUser.realName" />
        </el-form-item>
        <el-form-item label="手机号码" prop="phoneNumber">
          <el-input v-model="newUser.phoneNumber" />
        </el-form-item>
        <el-form-item label="性别">
          <el-select v-model="newUser.gender">
            <el-option v-for="option in genderOptions" :key="option.value" :label="option.label" :value="option.value" />
          </el-select>
        </el-form-item>
        <el-form-item label="角色">
          <el-select v-model="newUser.identity">
            <el-option v-for="option in identityOptions" :key="option.value" :label="option.label" :value="option.value" />
          </el-select>
        </el-form-item>
        <el-form-item label="学校">
          <el-select v-model="newUser.schoolId" placeholder="选择学校">
            <el-option v-for="school in schools" :key="school.id" :label="school.schoolName" :value="school.id" />
          </el-select>
        </el-form-item>
      </el-form>

      <div slot="footer" class="dialog-footer">
        <el-button @click="closeAddUserDialog">取消</el-button>
        <el-button type="primary" @click="validateAndSaveUser">保存</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import axios from "axios";
import { baseUrl } from "@/api/baseapi"; // 引入 baseUrl

export default {
  data() {
    return {
      // 用户信息
      newUser: {
        username: "",
        password: "",
        realName: "",
        phoneNumber: "",
        gender: 1,
        identity: 3,
        schoolId: "",
      },
      rules: {
        username: [
          { required: true, message: "用户名不能为空", trigger: "blur" },
          { pattern: /^.{1,10}$/, message: "用户名不能超过10个字符", trigger: "blur" },
        ],
        password: [
          { required: true, message: "密码不能为空", trigger: "blur" },
          { pattern: /^.{1,10}$/, message: "密码不能超过10个字符", trigger: "blur" },
        ],
        realName: [
          { required: true, message: "真实姓名不能为空", trigger: "blur" },
          { pattern: /^.{1,15}$/, message: "真实姓名不能超过15个字符", trigger: "blur" },
        ],
        phoneNumber: [
          { required: true, message: "手机号不能为空", trigger: "blur" },
          { pattern: /^\d{11}$/, message: "手机号必须是11位数字", trigger: "blur" },
        ],
      },
      editingUser: false, // 是否为编辑模式
      editingUserId: null, // 编辑的用户ID
      searchQuery: {
        username: "",
        realName: "",
        identity: "",
        gender: "",
      },
      allUsers: [],
      users: {
        list: [],
        curPage: 1,
        pageSize: 10,
        pages: 1,
        total: 0,
        isLast: false,
      },
      showAddUserDialog: false, // 弹窗显示状态
      schools: [], // 存储学校列表
      // 性别和角色选项
      genderOptions: [
        { label: "男", value: 1 },
        { label: "女", value: 0 },
      ],
      identityOptions: [
        { label: "校长", value: 0 },
        { label: "教务处", value: 1 },
        { label: "班主任", value: 2 },
        { label: "教师", value: 3 },
      ],
    };
  },
  methods: {
    validateAndSaveUser() {
      this.$refs.form.validate((valid) => {
        if (valid) {
          this.saveUser();
        } else {
          this.$message.error("请检查表单输入是否正确");
        }
      });
    },
    fetchSchools() {
      axios
        .get(`${baseUrl}/webUser/schools`)
        .then((response) => {
          this.schools = response.data;
        })
        .catch((error) => {
          console.error("获取学校列表失败：", error);
        });
    },
    fetchAllUsers() {
      axios
        .get(`${baseUrl}/webUser/all`)
        .then((response) => {
          this.allUsers = response.data; // 假设接口返回的数据为用户列表
        })
        .catch((error) => {
          console.error("获取所有用户失败：", error);
        });
    },
    fetchUsers() {
      const { username, realName, identity, gender } = this.searchQuery;
      axios
        .get(`${baseUrl}/webUser/list`, {
          params: {
            curPage: this.users.curPage,
            pageSize: this.users.pageSize,
            username,
            realName,
            identity,
            gender,
          },
        })
        .then((response) => {
          this.users = response.data;
        })
        .catch((error) => {
          console.error("获取用户列表出错：", error);
        });
    },
    saveUser() {
      // 验证用户名和手机号的唯一性
      const isDuplicateUsername = this.allUsers.some(
        (user) =>
          user.username === this.newUser.username &&
          (!this.editingUser || user.id !== this.editingUserId) // 排除当前编辑的用户
      );

      const isDuplicatePhoneNumber = this.allUsers.some(
        (user) =>
          user.phoneNumber === this.newUser.phoneNumber &&
          (!this.editingUser || user.id !== this.editingUserId) // 排除当前编辑的用户
      );

      if (isDuplicateUsername) {
        this.$message.error("用户名已存在，请重新输入！");
        return;
      }
      if (isDuplicatePhoneNumber) {
        this.$message.error("手机号已存在，请重新输入！");
        return;
      }

      // 如果是编辑模式，调用更新方法
      if (this.editingUser) {
        this.updateWebUser();
      } else {
        // 如果是新增模式，调用新增方法
        this.addWebUser();
      }
    },

    addWebUser() {
      axios
        .post(`${baseUrl}/webUser/add`, this.newUser)
        .then(() => {
          this.$message.success("用户添加成功！");
          this.fetchUsers();
          this.closeAddUserDialog();
        })
        .catch((error) => {
          console.error("添加用户出错：", error);
          this.$message.error("添加用户失败，请重试！");
        });
    },
    confirmDeleteUser(id) {
      this.$confirm(
        "此操作将永久删除该用户, 是否继续?",
        "提示",
        {
          confirmButtonText: "确定",
          cancelButtonText: "取消",
          type: "warning",
        }
      )
        .then(() => {
          // 确认删除
          this.deleteUser(id);
        })
        .catch(() => {
          // 用户取消删除
          this.$message.info("已取消删除");
        });
    },
    deleteUser(id) {
      axios
        .delete(`${baseUrl}/webUser/delete/${id}`)
        .then(() => {
          this.fetchUsers();
        })
        .catch((error) => {
          console.error("删除用户出错：", error);
        });
    },
    changePage(page) {
      if (page > 0 && page <= this.users.pages) {
        this.users.curPage = page;
        this.fetchUsers();
      }
    },
    editUser(user) {
      this.editingUser = true;
      this.editingUserId = user.id; // 设置当前编辑用户的 ID
      this.newUser = { ...user }; // 将用户信息复制到 newUser 中
      this.showAddUserDialog = true;
    },
    updateWebUser() {
      axios
        .put(`${baseUrl}/webUser/update`, this.newUser)
        .then(() => {
          this.$message.success("用户更新成功！");
          this.fetchUsers();
          this.closeAddUserDialog();
        })
        .catch((error) => {
          console.error("更新用户失败：", error);
          this.$message.error("更新用户失败，请重试！");
        });
    },
    searchUsers() {
      this.users.curPage = 1;
      this.fetchUsers();
    },
    closeAddUserDialog() {
      this.showAddUserDialog = false;
      this.editingUser = false;
      this.editingUserId = null; // 清空编辑用户 ID
      this.newUser = {
        username: "",
        password: "",
        realName: "",
        phoneNumber: "",
        gender: 1,
        identity: 3,
        schoolId: "",
      };
    },
    openAddUserDialog() {
      this.showAddUserDialog = true;
    },
    genderFormatter(row) {
      return this.genderOptions.find((option) => option.value === row.gender)?.label || "未知";
    },
    identityFormatter(row) {
      return this.identityOptions.find((option) => option.value === row.identity)?.label || "未知";
    },
    schoolNameFormatter(row) {
      const school = this.schools.find((school) => school.id === row.schoolId);
      return school ? school.schoolName : "未知学校";
    },
    resetSearch() {
      this.searchQuery = {
        username: "",
        realName: "",
        identity: "",
        gender: "",
      };
      this.fetchUsers();
    },
  },
  mounted() {
    this.fetchSchools();
    this.fetchAllUsers();
    this.fetchUsers();
  },
};
</script>

<style scoped>
.page-title {
  font-size: 18px;
  margin-bottom: 20px;
}
.search-box {
  margin-bottom: 20px;
}
</style>
