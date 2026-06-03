# Snowy 框架 — AI 即读开发规范（源码级）

## 技术栈
- Spring Boot 3.5.9 / JDK 17+ (jakarta.*)
- MyBatis-Plus (ORM, LambdaQueryWrapper)
- Sa-Token v1.40+ (登录/鉴权)
- HuTool 5.8.25 (工具库)
- EasyTrans (字段翻译 @Trans)
- SpringDoc OpenAPI (Swagger)
- Lombok (@Getter @Setter)
- 前端: Vue 3 + AntdV 4.2 + Vite 5 + Pinia + Axios

## 后端结构
```
snowy-plugin/snowy-plugin-{模块}/
└── src/main/java/vip/xiaonuo/{模块}/
    └── modular/{业务名}/
        ├── controller/     ← @RestController @Validated
        ├── service/        ← 接口
        │   └── impl/       ← @Service extends ServiceImpl<Mapper, Entity>
        ├── entity/         ← extends CommonEntity
        ├── mapper/         ← extends BaseMapper
        ├── param/          ← @Getter @Setter 请求DTO
        └── result/         ← 响应DTO
```

## 编码规则（AI 严格遵循）

### Entity
```java
@Getter @Setter
@TableName("MY_BUSINESS")    // 表名全大写_下划线
public class MyBusiness extends CommonEntity {  // 必须有 extends CommonEntity
    @Schema(description = "id")
    private String id;       // String 类型主键，非 Long

    @Schema(description = "名称")
    private String name;

    @Schema(description = "排序", requiredMode = Schema.RequiredMode.REQUIRED)
    private Integer sortCode;

    @Schema(description = "状态")
    private String status;
}
```
- 继承 `CommonEntity`（自带：deleteFlag, createTime, createUser, createUserName, updateTime, updateUser, updateUserName）
- 必须 `@Schema(description = "xxx")` 每个字段
- 表名大写：`@TableName("MY_BUSINESS")`
- 字段用小驼峰：`sortCode`, `createTime`
- 控制器返回 Entity 时直接返回，不需要转 VO

### Mapper
```java
public interface MyBusinessMapper extends BaseMapper<MyBusiness> {}
// 无需 XML，无需 @Mapper 注解（MP 自动扫描）
// 复杂查询在 Service 用 LambdaQueryWrapper
```

### Param（请求参数）
```java
@Getter @Setter
public class MyBusinessPageParam {           // 分页查询
    private Integer current = 1;             // 当前页码
    private Integer size = 20;               // 每页条数
    private String sortField;                // 排序字段（可选）
    @Schema(description = "名称关键词")
    private String searchKey;
}

@Getter @Setter
public class MyBusinessAddParam {             // 新增
    @NotBlank(message = "名称不能为空")
    @Schema(description = "名称")
    private String name;
    @Schema(description = "排序")
    private Integer sortCode;
}

@Getter @Setter
public class MyBusinessEditParam {            // 编辑（必须传 id）
    @NotBlank(message = "id不能为空")
    @Schema(description = "id")
    private String id;
    @NotBlank(message = "名称不能为空")
    @Schema(description = "名称")
    private String name;
    @Schema(description = "排序")
    private Integer sortCode;
}

@Getter @Setter
public class MyBusinessIdParam {              // 批量删除/查详情
    @Schema(description = "id")
    private String id;
}
```

### Service + Impl
```java
// === 接口文件 ===
public interface MyBusinessService {
    Page<MyBusiness> page(MyBusinessPageParam param);
    MyBusiness add(MyBusinessAddParam param);
    MyBusiness edit(MyBusinessEditParam param);
    void delete(List<MyBusinessIdParam> params);
    MyBusiness detail(MyBusinessIdParam param);
}

// === 实现文件 ===
@Service
public class MyBusinessServiceImpl extends ServiceImpl<MyBusinessMapper, MyBusiness>
        implements MyBusinessService {

    @Resource
    private MyBusinessMapper myBusinessMapper;  // ⚠️ 用 @Resource，不用 @Autowired

    @Override
    public Page<MyBusiness> page(MyBusinessPageParam p) {
        LambdaQueryWrapper<MyBusiness> qw = new LambdaQueryWrapper<>();
        qw.like(StrUtil.isNotBlank(p.getSearchKey()), MyBusiness::getName, p.getSearchKey());
        // 按排序字段：sortCode 升序，createTime 降序
        qw.orderByAsc(MyBusiness::getSortCode).orderByDesc(MyBusiness::getCreateTime);
        return this.page(PageUtil.getPage(p), qw);   // ⚠️ PageUtil，不是 new Page()
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public MyBusiness add(MyBusinessAddParam p) {
        MyBusiness entity = BeanUtil.toBean(p, MyBusiness.class);  // ⚠️ HuTool 转换
        entity.setId(IdUtil.fastUUID());      // ⚠️ id 用 UUID，非自增
        this.save(entity);
        return entity;
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public MyBusiness edit(MyBusinessEditParam p) {
        MyBusiness entity = this.query().getById(p.getId());  // 或 this.getById()
        BeanUtil.copyProperties(p, entity);   // ⚠️ HuTool copy
        this.updateById(entity);
        return entity;
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public void delete(List<MyBusinessIdParam> params) {
        this.removeByIds(params.stream().map(MyBusinessIdParam::getId).toList());
    }

    @Override
    public MyBusiness detail(MyBusinessIdParam p) {
        return this.getById(p.getId());
    }
}
```

### Controller（关键⚠️）
```java
@Tag(name = "业务控制器")                     // Swagger 标签
@RestController                                 // REST 控制器
@Validated                                      // 类级别的参数校验
public class MyBusinessController {

    @Resource                                    // ⚠️ 用 @Resource，不是 @Autowired
    private MyBusinessService myBusinessService;

    /**
     * 获取业务分页
     */
    @Operation(summary = "获取业务分页")         // Swagger 方法描述
    @GetMapping("/myBusiness/page")             // ⚠️ URL: /{模块}/{业务}/{动作}
    public CommonResult<Page<MyBusiness>> page(MyBusinessPageParam p) {
        return CommonResult.data(myBusinessService.page(p));  // ⚠️ .data()
    }

    /**
     * 添加业务
     */
    @Operation(summary = "添加业务")
    @CommonLog("添加业务")                       // ⚠️ 操作日志 增删改必须加
    @PostMapping("/myBusiness/add")
    public CommonResult<String> add(@RequestBody @Valid MyBusinessAddParam p) {
        myBusinessService.add(p);
        return CommonResult.ok();                // ⚠️ 无需返回数据用 .ok()
    }

    /**
     * 编辑业务
     */
    @Operation(summary = "编辑业务")
    @CommonLog("编辑业务")
    @PostMapping("/myBusiness/edit")
    public CommonResult<String> edit(@RequestBody @Valid MyBusinessEditParam p) {
        myBusinessService.edit(p);
        return CommonResult.ok();
    }

    /**
     * 删除业务
     */
    @Operation(summary = "删除业务")
    @CommonLog("删除业务")
    @PostMapping("/myBusiness/delete")
    public CommonResult<String> delete(
        @RequestBody @Valid @NotEmpty(message = "集合不能为空")    // ⚠️ 批量删除校验
        List<MyBusinessIdParam> params
    ) {
        myBusinessService.delete(params);
        return CommonResult.ok();
    }

    /**
     * 获取业务详情
     */
    @Operation(summary = "获取业务详情")
    @GetMapping("/myBusiness/detail")
    public CommonResult<MyBusiness> detail(@Valid MyBusinessIdParam p) {
        return CommonResult.data(myBusinessService.detail(p));
    }
}
```

### Controller 注解速查
| 注解 | 位置 | 说明 |
|------|------|------|
| `@Tag(name = "XX控制器")` | 类 | Swagger 分组 |
| `@RestController` | 类 | RESTful |
| `@Validated` | 类 | 参数校验 |
| `@Resource` | 字段 | 注入 （不是 @Autowired） |
| `@Operation(summary = "XX")` | 方法 | Swagger 描述 |
| `@CommonLog("XX")` | 方法 | 操作日志（增删改） |
| `@GetMapping("/path")` | 方法 | GET 查询 |
| `@PostMapping("/path")` | 方法 | POST 操作 |
| `@RequestBody @Valid` | 参数 | JSON 请求体校验 |
| `@NotEmpty(message = "集合不能为空")` | 参数 | 集合非空校验 |

### 常用 import（AI 自动用）
```java
// HuTool
import cn.hutool.core.util.StrUtil;
import cn.hutool.core.util.ObjectUtil;
import cn.hutool.core.collection.CollectionUtil;
import cn.hutool.core.bean.BeanUtil;
import cn.hutool.core.util.IdUtil;
import cn.hutool.json.JSONUtil;

// MP
import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import com.baomidou.mybatisplus.extension.plugins.pagination.Page;
import vip.xiaonuo.common.pojo.PageUtil;      // ⚠️ 分页用 PageUtil，不是 new Page()

// Snowy
import vip.xiaonuo.common.pojo.CommonResult;
import vip.xiaonuo.common.pojo.CommonEntity;
import vip.xiaonuo.common.exception.CommonException;
import vip.xiaonuo.common.annotation.CommonLog;
```

### 常见查询写法
```java
// 等值查询
qw.eq(ObjectUtil.isNotEmpty(p.getXxx()), MyBusiness::getType, p.getType());

// LIKE 查询
qw.like(StrUtil.isNotBlank(p.getSearchKey()), MyBusiness::getName, p.getSearchKey());

// 范围查询
qw.apply(StrUtil.isNotBlank(p.getStartDate()), "date_format(create_time,'%Y%m%d') >= {0}", p.getStartDate());

// IN 查询
qw.in(CollectionUtil.isNotEmpty(p.getIds()), MyBusiness::getId, p.getIds());

// 排序
qw.orderByAsc(MyBusiness::getSortCode).orderByDesc(MyBusiness::getCreateTime);
```

## 前端结构（完整 CRUD）

### API 层 (src/api/biz/xxxApi.js)
```javascript
import { baseRequest } from '@/utils/request'

const request = (url, ...arg) => baseRequest(`/myBusiness/` + url, ...arg)

/**
 * 业务模块
 */
export default {
    // 获取分页
    businessPage(data) {
        return request('page', data, 'get')
    },
    // 提交表单 edit为true时为编辑
    businessSubmitForm(data, edit = false) {
        return request(edit ? 'edit' : 'add', data)
    },
    // 删除
    businessDelete(data) {
        return request('delete', data)
    },
    // 获取详情
    businessDetail(data) {
        return request('detail', data, 'get')
    },
}
```

### 页面 (src/views/biz/xxx/index.vue)
```vue
<template>
    <a-form :model="searchFormState">
        <a-row :gutter="10">
            <a-col :xs="24" :sm="8" :md="8" :lg="8" :xl="8">
                <a-form-item label="关键词" name="searchKey">
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
    <s-table ref="tableRef" :columns="columns" :data="loadData" bordered
        :row-key="(record) => record.id" :tool-config="toolConfig"
        :row-selection="options.rowSelection">
        <template #operator>
            <a-button type="primary" @click="formRef.onOpen()" v-if="hasPerm('xxxAdd')">新增</a-button>
        </template>
        <template #bodyCell="{ column, record }">
            <template v-if="column.dataIndex === 'action'">
                <a @click="formRef.onOpen(record)" v-if="hasPerm('xxxEdit')">编辑</a>
                <a-divider type="vertical" v-if="hasPerm(['xxxEdit','xxxDelete'],'and')" />
                <a-popconfirm title="确定删除？" @confirm="remove(record)">
                    <a-button type="link" danger size="small" v-if="hasPerm('xxxDelete')">删除</a-button>
                </a-popconfirm>
            </template>
        </template>
    </s-table>
    <Form ref="formRef" @successful="tableRef.refresh(true)" />
</template>
<script setup name="bizXxx">
    import xxxApi from '@/api/biz/xxxApi'
    const searchFormState = ref({})
    const tableRef = ref()
    const formRef = ref()
    const columns = [
        { title: '名称', dataIndex: 'name' },
        { title: '状态', dataIndex: 'status' },
        { title: '操作', dataIndex: 'action', width: 150 },
    ]
    const loadData = (params) => xxxApi.page({ ...params, ...searchFormState.value })
    const reset = () => { searchFormState.value = {}; tableRef.value.refresh(true) }
    const remove = (record) => {
        xxxApi.delete([{ id: record.id }]).then(() => tableRef.value.refresh(true))
    }
</script>
```

### 表单弹窗 (src/views/biz/xxx/form.vue)
```vue
<template>
    <xn-form-container :title="formData.id ? '编辑' : '新增'" :width="550"
        :visible="visible" :destroy-on-close="true" @close="onClose">
        <a-form ref="formRef" :model="formData" :rules="formRules" layout="vertical">
            <a-form-item label="名称：" name="name">
                <a-input v-model:value="formData.name" placeholder="请输入" allow-clear />
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
            api(formData.value).finally(() => { submitLoading.value = false })
                .then(() => { emit('successful'); onClose() })
        })
    }
    const formRules = { name: [required('请输入名称')] }
</script>
```

## CRUD 开发流程

### 后端（6步）
1. `CREATE TABLE MY_BUSINESS (id VARCHAR...)`（大写表名，id VARCHAR(64)）
2. `MyBusiness extends CommonEntity`
3. `MyBusinessMapper extends BaseMapper<MyBusiness>`
4. `MyBusinessPageParam` + `MyBusinessAddParam` + `MyBusinessEditParam` + `MyBusinessIdParam`
5. `MyBusinessServiceImpl extends ServiceImpl<...> implements MyBusinessService`
6. `MyBusinessController`（page/add/edit/delete/detail，5个方法）