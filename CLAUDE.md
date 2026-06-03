# Snowy 框架 — AI 即读开发规范（全栈版）

> 将此文件放入 Snowy 项目根目录，AI 编程工具 (Codex/Claude Code/Cursor) 将自动理解框架规范。

---

## 一、技术栈

### 后端
- Spring Boot 3.5.9 / JDK 17+
- MyBatis-Plus (ORM)
- Sa-Token (登录/鉴权)
- HuTool 5.8.25 (工具库)
- EasyTrans (字段翻译 @Trans)
- SpringDoc OpenAPI (swagger)
- Lombok (@Getter @Setter)
- 数据库: MySQL / PostgreSQL / 达梦 / 人大金仓

### 前端
- Vue 3 + AntDesign Vue 4.2
- Vite 5 (构建)
- Pinia (状态管理)
- Axios (HTTP)
- s-table (表格组件)
- xn-form-container (弹窗表单)
- xn-tree-select (树选择)
- @antv/g2plot (图表)

---

## 二、后端规范

### 2.1 项目结构
```
snowy-plugin/snowy-plugin-{模块}/
└── src/main/java/vip/xiaonuo/{模块}/
    └── modular/{业务名}/
        ├── controller/     ← @RestController
        ├── service/        ← 接口
        │   └── impl/       ← @Service 实现
        ├── entity/         ← extends CommonEntity
        ├── mapper/         ← extends BaseMapper
        ├── param/          ← @Getter @Setter DTO
        ├── result/         ← 返回 VO
        ├── enums/          ← 枚举
        └── request/        ← 调用外部 API
```

### 2.2 后端编码规则

**Entity**
```java
@Getter @Setter
@TableName("MY_BUSINESS")  // 表名全大写_分隔
public class MyBusiness extends CommonEntity {
    @Schema(description = "id")
    private String id;          // String 主键，不是 Long
    @Schema(description = "名称")
    private String name;
    @Schema(description = "状态")
    private String status;
}
```
- 必须继承 `CommonEntity`（自动有：id, createTime, createUser, updateTime, updateUser, deleteFlag）
- 表名大写下划线：`MY_BUSINESS`
- `@Schema()` 每个字段必须标注

**Mapper**
```java
public interface MyBusinessMapper extends BaseMapper<MyBusiness> {}
// 不需要 mapper.xml，MP 自动生成 SQL
```

**Service**
```java
@Service
public class MyBusinessServiceImpl extends ServiceImpl<MyBusinessMapper, MyBusiness>
        implements MyBusinessService {
    @Resource  // 不用 @Autowired
    private MyBusinessMapper myBusinessMapper;

    public Page<MyBusiness> page(MyBusinessPageParam p) {
        LambdaQueryWrapper<MyBusiness> qw = new LambdaQueryWrapper<>();
        qw.like(StrUtil.isNotBlank(p.getXxx()), MyBusiness::getName, p.getXxx());
        return this.page(PageUtil.getPage(p), qw);
    }
}
```

**Controller**
```java
@Tag(name = "业务控制器")
@RestController @Validated
public class MyBusinessController {
    @Resource private MyBusinessService myBusinessService;

    @Operation(summary = "分页查询")
    @GetMapping("/myBusiness/page")
    public CommonResult<Page<MyBusiness>> page(@Valid MyBusinessPageParam p) {
        return CommonResult.data(myBusinessService.page(p));
    }

    @Operation(summary = "新增")
    @CommonLog("新增业务")
    @PostMapping("/myBusiness/add")
    public CommonResult<Void> add(@Valid @RequestBody MyBusinessAddParam p) {
        myBusinessService.add(p);
        return CommonResult.ok();
    }

    @Operation(summary = "删除")
    @CommonLog("删除业务")
    @PostMapping("/myBusiness/delete")
    public CommonResult<Void> delete(@Valid @RequestBody List<MyBusinessIdParam> ids) {
        myBusinessService.delete(ids);
        return CommonResult.ok();
    }
}
```

**Param 规则**
```java
@Getter @Setter
public class MyBusinessPageParam {
    private Integer current;
    private Integer size;
    private String sortField;
    @Schema(description = "名称关键词")
    private String searchKey;
}

@Getter @Setter
public class MyBusinessAddParam {
    @NotBlank(message = "名称不能为空")
    @Schema(description = "名称", requiredMode = Schema.RequiredMode.REQUIRED)
    private String name;
}

@Getter @Setter
public class MyBusinessIdParam {
    @Schema(description = "id")
    private String id;
}
```

---

## 三、前端规范

### 3.1 前端目录结构
```
src/
├── api/              ← 接口定义 (Axios)
│   ├── sys/          ← 系统管理
│   ├── biz/          ← 业务模块
│   ├── auth/         ← 认证
│   ├── dev/          ← 开发工具
│   └── gen/          ← 代码生成
├── views/            ← 页面
│   └── {模块}/{业务}/
│       ├── index.vue        ← 列表页(主页面)
│       └── form.vue         ← 表单弹窗
├── router/           ← 路由配置
│   ├── index.js
│   └── systemRouter.js      ← 系统/业务路由
├── store/            ← Pinia 状态
├── components/       ← 公共组件
│   ├── XnUpload, XnTable, TreeSelect, DictSelect...
└── utils/            ← 工具方法
```

### 3.2 API 层 (src/api/)
```javascript
import request from '@/utils/request'
const baseURL = '/biz/xxx'

// CRUD 标准写法
export default {
    page: (data) => { return request.get(baseURL + '/page', { params: data }) },
    add: (data) => { return request.post(baseURL + '/add', data) },
    edit: (data) => { return request.post(baseURL + '/edit', data) },
    delete: (data) => { return request.post(baseURL + '/delete', data) },
    detail: (data) => { return request.get(baseURL + '/detail', { params: data }) },
}
```
- 方法名统一：page / add / edit / delete / detail
- GET 请求用 `params: data`
- POST 请求直接传 data

### 3.3 列表页 (index.vue)
```vue
<template>
    <!-- 搜索栏 -->
    <a-form :model="searchFormState">
        <a-row :gutter="10">
            <a-col :xs="24" :sm="8" :md="8" :lg="8" :xl="8">
                <a-form-item name="searchKey" label="关键词">
                    <a-input v-model:value="searchFormState.searchKey" />
                </a-form-item>
            </a-col>
            <a-col :xs="24" :sm="16" :md="16" :lg="16" :xl="16">
                <a-space>
                    <a-button type="primary" @click="tableRef.refresh(true)">查询</a-button>
                    <a-button @click="reset">重置</a-button>
                </a-space>
            </a-col>
        </a-row>
    </a-form>

    <!-- 表格 -->
    <s-table ref="tableRef" :columns="columns" :data="loadData" bordered
        :row-key="(record) => record.id" :tool-config="toolConfig"
        :row-selection="options.rowSelection">
        <template #operator>
            <a-button type="primary" @click="formRef.onOpen()" v-if="hasPerm('xxxAdd')">新增</a-button>
        </template>
        <template #bodyCell="{ column, record }">
            <template v-if="column.dataIndex === 'action'">
                <a @click="formRef.onOpen(record)" v-if="hasPerm('xxxEdit')">编辑</a>
                <a-divider type="vertical" />
                <a-popconfirm title="确认删除？" @confirm="remove(record)">
                    <a-button type="link" danger size="small" v-if="hasPerm('xxxDelete')">删除</a-button>
                </a-popconfirm>
            </template>
        </template>
    </s-table>
    <Form ref="formRef" @successful="tableRef.refresh(true)" />
</template>

<script setup name="bizXxx">
    import { $TOOL, hasPerm } from '@/utils/tool'

    const searchFormState = ref({ searchKey: '' })
    const tableRef = ref()
    const formRef = ref()
    const selectedRowKeys = ref([])

    const columns = [
        { title: '名称', dataIndex: 'name' },
        { title: '状态', dataIndex: 'status' },
        { title: '操作', dataIndex: 'action', width: 150 },
    ]

    const loadData = (params) => {
        return xxxApi.page({ ...params, ...searchFormState.value })
    }

    const reset = () => {
        searchFormState.value = { searchKey: '' }
        tableRef.value.refresh(true)
    }

    const remove = (record) => {
        xxxApi.delete([{ id: record.id }]).then(() => {
            tableRef.value.refresh(true)
        })
    }
</script>
```

### 3.4 表单弹窗 (form.vue)
```vue
<template>
    <xn-form-container :title="formData.id ? '编辑' : '新增'"
        :width="550" :visible="visible" :destroy-on-close="true" @close="onClose">
        <a-form ref="formRef" :model="formData" :rules="formRules" layout="vertical">
            <a-form-item label="名称：" name="name">
                <a-input v-model:value="formData.name" placeholder="请输入名称" allow-clear />
            </a-form-item>
            <a-form-item label="状态：" name="status">
                <a-select v-model:value="formData.status" :options="$TOOL.dictList('XXX_STATUS')" />
            </a-form-item>
            <a-form-item label="排序：" name="sortCode">
                <a-input-number v-model:value="formData.sortCode" :max="100" />
            </a-form-item>
        </a-form>
        <template #footer>
            <a-button @click="onClose">关闭</a-button>
            <a-button type="primary" :loading="submitLoading" @click="onSubmit">保存</a-button>
        </template>
    </xn-form-container>
</template>

<script setup name="bizXxxForm">
    import { required } from '@/utils/formRules'
    import xxxApi from '@/api/biz/xxxApi'

    const emit = defineEmits({ successful: null })
    const visible = ref(false)
    const formRef = ref()
    const formData = ref({ sortCode: 99 })
    const submitLoading = ref(false)

    const onOpen = (record) => {
        visible.value = true
        formData.value = record ? Object.assign({}, record) : { sortCode: 99 }
    }

    const onClose = () => { visible.value = false }

    const onSubmit = () => {
        formRef.value.validate().then(() => {
            submitLoading.value = true
            const api = formData.value.id ? xxxApi.edit : xxxApi.add
            api(formData.value).then(() => {
                emit('successful')
                onClose()
            }).finally(() => { submitLoading.value = false })
        })
    }

    const formRules = {
        name: [required('请输入名称')],
        status: [required('请选择状态')],
    }
</script>
```

### 3.5 路由配置
编辑 `src/router/systemRouter.js`：

```javascript
module.exports = [
    {
        path: 'myBusiness',
        name: '业务管理',
        component: () => import('@/views/biz/myBusiness/index.vue'),
        meta: {
            title: '业务管理',
            permission: 'myBusiness'
        }
    }
]
```

### 3.6 常用组件
| 组件 | 用途 |
|------|------|
| `<s-table>` | 表格（自带分页、排序、批量选择） |
| `<xn-form-container>` | 弹窗表单 |
| `<xn-tree-select>` | 树形选择（机构等） |
| `<xn-batch-button>` | 批量删除按钮 |
| `<DictSelect>` | 字典下拉 |
| `<TreeSelect>` | 树形下拉 |

---

## 四、CRUD 快速开发步骤

### 后端
1. 创建 SQL：`CREATE TABLE {大写表名}`
2. 创建 `Entity`（extends CommonEntity）
3. 创建 `Mapper`（extends BaseMapper）
4. 创建 `Param`（PageParam / AddParam / IdParam）
5. 创建 `Service` + `ServiceImpl`
6. 创建 `Controller`（分页/新增/编辑/删除/详情 5 个接口）

### 前端
1. 创建 `src/api/biz/xxxApi.ts`（page / add / edit / delete）
2. 创建 `src/views/biz/xxx/index.vue`（列表页）
3. 创建 `src/views/biz/xxx/form.vue`（表单弹窗）
4. 编辑 `src/router/systemRouter.js`（添加路由）

---

## 五、常见查询写法

```java
// 等值查询
qw.eq(ObjectUtil.isNotEmpty(p.getXxx()), MyBusiness::getXxx, p.getXxx());

// LIKE
qw.like(StrUtil.isNotBlank(p.getSearchKey()), MyBusiness::getName, p.getSearchKey());

// 时间范围
qw.apply(StrUtil.isNotBlank(p.getStartDate()), "date_format(create_time,'%Y%m%d') >= {0}", p.getStartDate());

// 排序
qw.orderByAsc(MyBusiness::getSortCode).orderByDesc(MyBusiness::getCreateTime);

// IN
qw.in(CollectionUtil.isNotEmpty(p.getIds()), MyBusiness::getId, p.getIds());
```

---

## 六、附录

### 后端常用 import
```java
import cn.hutool.core.util.StrUtil;
import cn.hutool.core.util.ObjectUtil;
import cn.hutool.core.collection.CollectionUtil;
import cn.hutool.core.bean.BeanUtil;
import cn.hutool.core.util.IdUtil;
import cn.hutool.json.JSONUtil;
import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import com.baomidou.mybatisplus.extension.plugins.pagination.Page;
import vip.xiaonuo.common.pojo.CommonResult;
import vip.xiaonuo.common.exception.CommonException;
import vip.xiaonuo.common.annotation.CommonLog;
```

### Sa-Token 权限验证
```java
// 后端
@SaCheckPermission("xxx:add")
@PreAuth("hasPermission('xxx')")

// 前端
hasPerm('xxxAdd')  // v-if="hasPerm('xxxAdd')"
```