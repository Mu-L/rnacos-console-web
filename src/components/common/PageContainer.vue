<template>
  <div class="page-container">
    <div
      class="page-header"
      v-if="slots.header"
    >
      <slot
        name="header"
        :methods="{ createForm, refreshData, onSearch }"
        :param="state.param"
      ></slot>
    </div>
    <div class="page-content">
      <div
        v-if="slots.actions"
        class="page-actions"
      >
        <NCard>
          <div class="action">
            <slot
              name="actions"
              :methods="{ createForm, refreshData, loadData }"
              :param="state.param"
            ></slot>
          </div>
        </NCard>
      </div>
      <div class="page-table">
        <NCard>
          <n-data-table
            :columns="config.columns"
            :data="state.tableData"
            :pagination="config.pagination"
          />
        </NCard>
      </div>
      <div class="page-pager"></div>
    </div>
    <NDrawer
      to="#app"
      :block-scroll="false"
      :trap-focus="false"
      v-model:show="showDrawer"
      default-width="600"
      :width="config.drawer?.width"
      resizable
    >
      <NDrawerContent
        :title="formTitle"
        closable
      >
        <slot
          name="form"
          :operationType="state.operationType"
          :submitLoading="state.submitLoading"
          :formData="state.formData"
          :isReadonly="state.formData.mode === constant.FORM_MODE_DETAIL"
          :isKeyReadonly="state.formData.mode !== constant.FORM_MODE_CREATE"
          :methods="{ onSave, onValidateFail, closeDrawer }"
        ></slot>
        <template #footer>
          <slot
            name="footer"
            v-if="slots.footer"
          ></slot>
          <n-space
            v-else
            align="baseline"
          >
            <n-button
              text
              @click="closeDrawer"
            >
              返回
            </n-button>
            <n-button
              type="primary"
              @click="confirm"
            >
              确认
            </n-button>
          </n-space>
        </template>
      </NDrawerContent>
    </NDrawer>
  </div>
</template>
<script setup lang="ts">
import type { CrudOptions } from '@/types/base'
import { NCard, NDrawerContent, useMessage } from 'naive-ui'
import type { AnyObj } from '@/utils'
import constant from '@/types/constant'
import apis from '@/apis/index'
const slots = useSlots()
const toast = useMessage()
console.log(slots.actions, 'slots')
const emits = defineEmits(['onSave'])
let props = defineProps({
  config: {
    type: Object as PropType<CrudOptions<Object>>,
    default: new Map<string, any>(),
  },
  data: {
    type: Array<any>,
    default: () => [],
  },
})
const showDrawer = ref(false)
const state = reactive({
  operationType: 1,
  submitLoading: false,
  formData: {} as AnyObj,
  param: (props.config.param ? props.config.param : {}) as AnyObj,
  tableData: [],
})

// 动态标题
const formTitle = computed(() => props.config.form?.title + (state.formData.mode == constant.FORM_MODE_CREATE ? '新增' : state.formData.mode == constant.FORM_MODE_UPDATE ? '编辑' : '详情'))

/**
 * 新增
 *
 * @param model 数据字段
 */
const createForm = (model: AnyObj = {}) => {
  state.formData = { ...model, mode: constant.FORM_MODE_CREATE }
  showDrawer.value = true
}

/**
 * 修改
 *
 * @param itemData 变更数据项
 */
const updateForm = (itemData: AnyObj) => {
  state.formData = { ...itemData, mode: constant.FORM_MODE_UPDATE }
  showDrawer.value = true
}

/**
 * 确认
 */
const confirm = () => {
  if (state.formData.mode === 'add' || state.formData.mode === constant.FORM_MODE_CREATE) {
    onSave()
  } else {
    onUpdate()
  }
}

/**
 * 保存表单数据
 */
const onSave = async () => {
  // 调用表单校验
  if (props.config.validator && typeof props.config.validator === 'function') {
    let vr = await props.config.validator()
    if (!vr) {
      return
    }
  }
  let { data } = await apis.postJSON(`${props.config.apis?.create || ''}`, {
    data: state.formData,
    headers: {
      'Content-Type': 'application/x-www-form-urlencoded',
    },
  })
  if (data.code === 200 && data.data === true) {
    toast.success('添加成功')
    refreshData()
  } else {
    toast.error(data.message || '添加失败')
  }
}

/**
 * 更新表单数据
 */
const onUpdate = async () => {
  // 调用表单校验
  if (props.config.validator && typeof props.config.validator === 'function') {
    let vr = await props.config.validator()
    if (!vr) {
      return
    }
  }
  let { data } = await apis.putJSON(`${props.config.apis?.update || ''}`, {
    data: state.formData,
    headers: {
      'Content-Type': 'application/x-www-form-urlencoded',
    },
  })
  if (data.code === 200 && data.data === true) {
    toast.success('修改成功')
    refreshData()
  } else {
    toast.error(data.message || '修改失败')
  }
}

/**
 * 删除数据
 *
 * @param params 删除参数
 */
const onDelete = async (params: AnyObj) => {
  let { data } = await apis.deleteJSON(`${props.config.apis?.delete || ''}`, {
    data: params,
    headers: {
      'Content-Type': 'application/x-www-form-urlencoded',
    },
  })
  if (data.code === 200 && data.data === true) {
    toast.success('删除成功')
    refreshData()
  } else {
    toast.error(data.message || '删除失败')
  }
}

/**
 * 关闭抽屉
 */
const closeDrawer = () => {
  state.formData = {}
  showDrawer.value = false
  state.submitLoading = false
}

/**
 * 加载数据
 */
const loadData = async () => {
  let url = `${props.config.apis?.list}`
  if (url) {
    let resp = await apis.getJSON(url, {
      params: state.param,
    })
    console.log(resp.data, 'response')
    // if (data.code === 200) {
    // 兼容处理
    if (Array.isArray(resp.data.data)) {
      state.tableData = resp.data?.data || []
    } else {
      if (Array.isArray(resp.data.list)) {
        state.tableData = resp.data.list || []
        return
      } else if (Array.isArray(resp.data.data.list)) {
        state.tableData = resp.data.data.list || []
      }
    }
    // }
  } else {
    state.tableData = (props.data || []) as any
  }
}

const refreshData = () => {
  state.param.pageNo = 1
  closeDrawer()
  loadData()
}

const onSearch = () => {
  loadData()
}

onMounted(() => {
  loadData()
})

/**
 * 表单验证
 */
const onValidateFail = () => {
  console.log('onValidateFail')
}

const saveUpdateForm = async (data: AnyObj) => {
  let url = `${props.config?.apis?.update || ''}`
  if (!url) {
    toast.error('请传入update-api')
    return
  }
  let res = await apis.putJSON(`${props.config?.apis?.update || ''}`, {
    data: {
      ...data,
    },
    headers: {
      'Content-Type': 'application/x-www-form-urlencoded',
    },
  })
  if (res.status == 200) {
    if (res.data.code == 200) {
      refreshData()
      toast.success(res.data.message || '修改成功')
    } else {
      toast.error(res.data.message || '修改失败')
    }
  } else {
    toast.error('request err,status code:' + res.status)
  }
}

defineExpose({
  onSave,
  onSearch,
  onUpdate,
  onDelete,
  createForm,
  updateForm,
  closeDrawer,
  refreshData,
  saveUpdateForm,
})
</script>

<style lang="scss" scoped>
.page-container {
  box-sizing: border-box;
  border-radius: 10px;

  .page-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    height: 40px;
    background: #fff;
    box-sizing: border-box;
    padding: 0 20px;
    margin-bottom: 10px;
  }

  .page-actions {
    background-color: #fff;

    .action {
      display: flex;
      justify-content: space-between;
    }
  }

  .page-content {
    margin: 10px;
  }

  .page-table {
    background-color: #fff;
  }
}
</style>