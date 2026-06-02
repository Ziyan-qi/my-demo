<template>
  <div class="admin-container">
    <h1>后台管理 Demo</h1>

    <!-- 搜索区域 -->
    <div class="search-box">
      <el-input
        v-model="searchKey"
        placeholder="输入用户名搜索"
        style="width: 300px"
      />
      <el-button type="primary" @click="getList">搜索</el-button>
      <el-button @click="openAddDialog">新增用户</el-button>
    </div>

    <!-- 数据表格（支持排序） -->
    <el-table
      :data="userList"
      border
      style="width: 100%"
      row-key="id"
      @sort-change="handleSortChange"
    >
      <el-table-column label="ID" prop="id" sortable />
      <el-table-column label="用户名" prop="name" />
      <el-table-column label="年龄" prop="age" />
      <el-table-column label="操作">
        <template #default="scope">
          <el-button
            type="success"
            size="small"
            @click="openEditDialog(scope.row)"
          >
            编辑
          </el-button>
          <el-button
            type="danger"
            size="small"
            @click="deleteUser(scope.row.id)"
          >
            删除
          </el-button>
        </template>
      </el-table-column>
    </el-table>

    <!-- 分页 -->
    <el-pagination
      @size-change="handleSizeChange"
      @current-change="handleCurrentChange"
      :current-page="page"
      :page-sizes="[5,10,20]"
      :page-size="limit"
      layout="total, sizes, prev, pager, next, jumper"
      :total="total"
      style="margin-top:20px; text-align:right"
    />

    <!-- 新增/编辑弹窗 -->
    <el-dialog v-model="dialogVisible" :title="isEdit ? '编辑用户' : '新增用户'" @close="resetForm">
      <el-form ref="formRef" :model="form" :rules="rules" label-width="80px">
        <el-form-item label="用户名" prop="name">
          <el-input v-model="form.name" placeholder="请输入用户名" />
        </el-form-item>
        <el-form-item label="年龄" prop="age">
          <el-input v-model.number="form.age" placeholder="请输入年龄" type="number" />
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button @click="dialogVisible = false">取消</el-button>
        <el-button type="primary" @click="confirmSubmit">确认</el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import { ref, onMounted, nextTick } from 'vue'
import { ElMessage, ElMessageBox } from 'element-plus'

// 搜索
const searchKey = ref('')

// 分页
const userList = ref([])
const page = ref(1)
const limit = ref(5)
const total = ref(0)

// 排序状态
const sortProp = ref('id')
const sortOrder = ref('ascending') // 默认正序

// 全部数据
const allMockData = [
  { id: 1, name: '张三', age: 22 },
  { id: 2, name: '李四', age: 25 },
  { id: 3, name: '王五', age: 28 },
  { id: 4, name: '赵六', age: 23 },
  { id: 5, name: '钱七', age: 30 }
]

const sourceList = ref([...allMockData])
let maxId = 5

// 弹窗
const dialogVisible = ref(false)
const isEdit = ref(false)
const formRef = ref(null)
const form = ref({ id: '', name: '', age: '' })

const rules = ref({
  name: [{ required: true, message: '请输入用户名', trigger: 'blur' }],
  age: [
    { required: true, message: '请输入年龄', trigger: 'blur' },
    { type: 'number', message: '年龄必须为数字' }
  ]
})

// 打开新增
const openAddDialog = () => {
  isEdit.value = false
  resetForm()
  dialogVisible.value = true
}

// 打开编辑
const openEditDialog = (row) => {
  isEdit.value = true
  form.value = { ...row }
  dialogVisible.value = true
}

// 重置表单
const resetForm = () => {
  form.value = { id: '', name: '', age: '' }
  formRef.value?.clearValidate()
}

// 提交
const confirmSubmit = async () => {
  const valid = await formRef.value.validate()
  if (!valid) return

  let newId = null
  if (isEdit.value) {
    const idx = sourceList.value.findIndex(x => x.id === form.value.id)
    if (idx !== -1) {
      sourceList.value[idx].name = form.value.name
      sourceList.value[idx].age = form.value.age
    }
    ElMessage.success('编辑成功')
  } else {
    newId = ++maxId
    // 新增数据放在最前面
    sourceList.value.unshift({
      id: newId,
      name: form.value.name,
      age: form.value.age
    })
    ElMessage.success('新增成功')
  }

  dialogVisible.value = false
  getList()

  // 新增后自动跳转到所在分页
  if (!isEdit.value && newId) {
    await nextTick()
    jumpToPageById(newId)
  }
}

// 排序事件（关键：不重置页码）
const handleSortChange = ({ prop, order }) => {
  sortProp.value = prop
  sortOrder.value = order
  // 这里不写 page.value = 1，就不会跳页
  getList()
}

// 获取列表（搜索 + 全局排序 + 分页）
const getList = () => {
  let list = sourceList.value.filter(item =>
    item.name.includes(searchKey.value)
  )

  // 全局跨分页排序（所有数据一起排序）
  if (sortProp.value === 'id') {
    list = list.sort((a, b) => {
      return sortOrder.value === 'ascending' ? a.id - b.id : b.id - a.id
    })
  }

  total.value = list.length
  const start = (page.value - 1) * limit.value
  const end = start + limit.value
  userList.value = list.slice(start, end)
}

// 根据ID跳转到所在分页
const jumpToPageById = (id) => {
  let list = [...sourceList.value].sort((a, b) => {
    return sortOrder.value === 'ascending' ? a.id - b.id : b.id - a.id
  })
  const index = list.findIndex(item => item.id === id)
  if (index === -1) return
  page.value = Math.floor(index / limit.value) + 1
  getList()
}

// 删除
const deleteUser = (id) => {
  ElMessageBox.confirm('确定删除？', '提示', { type: 'warning' }).then(() => {
    sourceList.value = sourceList.value.filter(x => x.id !== id)
    ElMessage.success('删除成功')
    getList()
  })
}

const handleSizeChange = (val) => {
  limit.value = val
  page.value = 1
  getList()
}

const handleCurrentChange = (val) => {
  page.value = val
  getList()
}

onMounted(() => getList())
</script>

<style>
.admin-container {
  width: 92%;
  margin: 30px auto;
}
.search-box {
  display: flex;
  gap: 10px;
  margin-bottom: 20px;
}
</style>