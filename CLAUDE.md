# Snowy 框架 — AI 即读开发规范（源码精确版）

## 技术栈
- Spring Boot 3.5.9 / JDK 17+（jakarta.*）
- MyBatis-Plus（ORM，QueryWrapper + LambdaQueryWrapper）
- Sa-Token（认证授权）
- HuTool 5.8.25（工具库）
- EasyTrans（字段翻译 @Trans）
- SpringDoc OpenAPI（Swagger）
- Lombok（@Getter @Setter）
- 前端：Vue 3 + AntdV 4.2 + Vite 5 + Pinia + Axios

## 后端结构
```
vip.xiaonuo.{模块}.modular.{业务}/
├── controller/     ← @RestController @Validated
├── service/        ← 接口
│   └── impl/       ← @Service
├── entity/         ← extends CommonEntity
├── mapper/         ← extends BaseMapper（无 XML）
├── param/          ← @Getter @Setter 请求DTO
└── result/         ← 响应DTO
```

## 精确编码规则（每条都从源码验证过）

### Entity 实体
```java
@Getter @Setter
@TableName("MY_BUSINESS")      // ⚠️ 表名全大写_下划线
public class MyBusiness extends CommonEntity {  // ⚠️ 必须继承
    @Schema(description = "id")
    private String id;

    @Schema(description = "名称")
    private String name;

    @Schema(description = "排序")
    private Integer sortCode;
}
```
- 必须继承 `CommonEntity`（自动有：deleteFlag, createTime, createUser, createUserName, updateTime, updateUser, updateUserName）
- 每个字段必须 `@Schema(description = "xxx")`
- 表名必须大写：`MY_BUSINESS`（不是 `my_business`）
- ID 为雪花算法自动生成，不需要在 Service 里手动 set

### Mapper
- 只需继承 `BaseMapper<Entity>`，无 XML
- 无需 `@Mapper` 注解（MP 自动扫描）

### Param 请求参数
```java
@Getter @Setter
public class MyBusinessPageParam {           // 分页
    private Integer current;                  // 当前页
    private Integer size;                     // 每页条数
    private String sortField;                 // 排序字段
    private String sortOrder;                 // 排序方向 ASC/DESC
    @Schema(description = "关键词")
    private String searchKey;
}

@Getter @Setter
public class MyBusinessAddParam {             // 新增
    @NotBlank(message = "名称不能为空")
    @Schema(description = "名称")
    private String name;
}

@Getter @Setter
public class MyBusinessEditParam {            // 编辑
    @NotBlank(message = "id不能为空")
    @Schema(description = "id")
    private String id;
    @NotBlank(message = "名称不能为空")
    @Schema(description = "名称")
    private String name;
}

@Getter @Setter
public class MyBusinessIdParam {              // 删除/详情
    @Schema(description = "id")
    private String id;
}
```

### Service + Implement
```java
// === 接口 ===
public interface MyBusinessService {
    Page<MyBusiness> page(MyBusinessPageParam param);
    void add(MyBusinessAddParam param);
    void edit(MyBusinessEditParam param);
    void delete(List<MyBusinessIdParam> params);
    MyBusiness detail(MyBusinessIdParam param);
}

// === 实现（关键⚠️）===
@Service
public class MyBusinessServiceImpl extends ServiceImpl<MyBusinessMapper, MyBusiness>
        implements MyBusinessService {

    @Resource                              // ⚠️ 用 @Resource，不用 @Autowired
    private MyBusinessMapper myBusinessMapper;

    @Override
    public Page<MyBusiness> page(MyBusinessPageParam p) {
        // ⚠️ 第一种写法：QueryWrapper（推荐，有 SQL 注入检查）
        QueryWrapper<MyBusiness> qw = new QueryWrapper<MyBusiness>().checkSqlInjection();
        qw.lambda()
            .like(StrUtil.isNotBlank(p.getSearchKey()), MyBusiness::getName, p.getSearchKey())
            .orderByAsc(MyBusiness::getSortCode);
        return this.page(CommonPageRequest.defaultPage(), qw);  // ⚠️ 分页用 CommonPageRequest
    }

    @Override
    @Transactional(rollbackFor = Exception.class)     // ⚠️ 必须加事务
    public void add(MyBusinessAddParam p) {
        MyBusiness entity = BeanUtil.toBean(p, MyBusiness.class);  // ⚠️ HuTool 转换
        this.save(entity);                           // ⚠️ 不需要手动生成 ID
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public void edit(MyBusinessEditParam p) {
        MyBusiness entity = this.getById(p.getId());
        BeanUtil.copyProperties(p, entity);          // ⚠️ HuTool 拷贝
        this.updateById(entity);
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public void delete(List<MyBusinessIdParam> params) {
        List<String> ids = CollStreamUtil.toList(params, MyBusinessIdParam::getId); // ⚠️ 或用 stream
        this.removeByIds(ids);
    }

    @Override
    public MyBusiness detail(MyBusinessIdParam p) {
        return this.getById(p.getId());
    }
}
```

### Controller（最重要⚠️ 每条都准确）
```java
@Tag(name = "业务控制器")                // Swagger 标签
@RestController                          // REST
@Validated                              // 参数校验
public class MyBusinessController {

    @Resource                           // ⚠️ @Resource 不是 @Autowired
    private MyBusinessService myBusinessService;

    /**
     * 获取分页
     */
    @Operation(summary = "获取业务分页")
    @GetMapping("/myBusiness/page")     // ⚠️ 路径: /{模块}/{业务}/{方法}
    public CommonResult<Page<MyBusiness>> page(MyBusinessPageParam p) {
        return CommonResult.data(myBusinessService.page(p));  // ⚠️ data() 包装
    }

    /**
     * 新增
     */
    @Operation(summary = "新增业务")
    @CommonLog("新增业务")               // ⚠️ 增删改必须有 @CommonLog
    @PostMapping("/myBusiness/add")
    public CommonResult<String> add(@RequestBody @Valid MyBusinessAddParam p) {
        myBusinessService.add(p);
        return CommonResult.ok();        // ⚠️ 无数据返回用 ok()
    }

    /**
     * 编辑
     */
    @Operation(summary = "编辑业务")
    @CommonLog("编辑业务")
    @PostMapping("/myBusiness/edit")
    public CommonResult<String> edit(@RequestBody @Valid MyBusinessEditParam p) {
        myBusinessService.edit(p);
        return CommonResult.ok();
    }

    /**
     * 删除
     */
    @Operation(summary = "删除业务")
    @CommonLog("删除业务")
    @PostMapping("/myBusiness/delete")
    public CommonResult<String> delete(
            @RequestBody @Valid @NotEmpty(message = "集合不能为空")  // ⚠️ 批量删除
            List<MyBusinessIdParam> params) {
        myBusinessService.delete(params);
        return CommonResult.ok();
    }

    /**
     * 详情
     */
    @Operation(summary = "获取业务详情")
    @GetMapping("/myBusiness/detail")
    public CommonResult<MyBusiness> detail(@Valid MyBusinessIdParam p) {
        return CommonResult.data(myBusinessService.detail(p));
    }
}
```

### 精确速查表
| 组件 | 写法 | 验证 |
|------|------|------|
| 依赖注入 | `@Resource` | 源码 13 处 vs @Autowired 2 处 |
| 分页 | `CommonPageRequest.defaultPage()` | PageUtil 不存在 |
| Service 继承 | `extends ServiceImpl<Mapper, Entity>` | 源码确认 |
| ID 生成 | 雪花算法自动，不手动 set | 源码确认 |
| Bean 转换 | `BeanUtil.toBean()` / `copyProperties()` | 源码确认 |
| 事务 | `@Transactional(rollbackFor = Exception.class)` | 源码确认 |
| Controller 返回(数据) | `CommonResult.data(xxx)` | 源码确认 |
| Controller 返回(空) | `CommonResult.ok()` 或 `CommonResult<String>` | 源码确认 |
| 日志 | `@CommonLog("xxx")` 仅增删改 | 源码确认 |
| Swagger | `@Tag(name="")` + `@Operation(summary="")` | 源码确认 |
| 批量删除 | `@NotEmpty(message="集合不能为空") List<XxxIdParam>` | 源码确认 |
| 批量/Stream | 不用 stream，用 `CollStreamUtil.toList()` | 源码确认 |
| 查询条件 | `new QueryWrapper<>().checkSqlInjection()` | Snowy 自定义 |
| URL 路径 | `/{module}/{business}/{action}` | 如 `/sys/group/page` |
| 表名 | `大写_下划线` 如 `MY_BUSINESS` | 源码确认 |
| 继承 | `extends CommonEntity` | 必含通用字段 |

## 常见查询速查
```java
// 查询 + 检查 SQL 注入
QueryWrapper<MyBusiness> qw = new QueryWrapper<MyBusiness>().checkSqlInjection();
qw.lambda()
    .eq(ObjectUtil.isNotEmpty(p.getStatus()), Entity::getStatus, p.getStatus())
    .like(StrUtil.isNotBlank(p.getSearchKey()), Entity::getName, p.getSearchKey())
    .in(CollectionUtil.isNotEmpty(p.getIds()), Entity::getId, p.getIds())
    .orderByAsc(Entity::getSortCode)
    .orderByDesc(Entity::getCreateTime);

// 带排序参数处理
if(ObjectUtil.isAllNotEmpty(p.getSortField(), p.getSortOrder())) {
    CommonSortOrderEnum.validate(p.getSortOrder());
    qw.orderBy(true, p.getSortOrder().equals("ASC"),
        StrUtil.toUnderlineCase(p.getSortField()));
} else {
    qw.lambda().orderByAsc(Entity::getSortCode);
}
```

## 前端

### API (src/api/biz/xxxApi.js)
```javascript
import { baseRequest } from '@/utils/request'

const request = (url, ...arg) => baseRequest(`/myBusiness/` + url, ...arg)

export default {
    // 获取分页
    businessPage(data) { return request('page', data, 'get') },
    // 提交表单（edit=true 为编辑）
    businessSubmitForm(data, edit = false) { return request(edit ? 'edit' : 'add', data) },
    // 删除
    businessDelete(data) { return request('delete', data) },
    // 详情
    businessDetail(data) { return request('detail', data, 'get') },
}
```

### 列表页 (src/views/biz/xxx/index.vue)
参考 `src/views/biz/position/index.vue`：s-table + tree 面板布局

### 表单 (src/views/biz/xxx/form.vue)
参考 `src/views/biz/position/form.vue`：xn-form-container + 校验规则

## CRUD 开发步骤
1. 建 SQL（大写表名）
2. Entity（extends CommonEntity）
3. Mapper（extends BaseMapper）
4. Param（PageParam / AddParam / EditParam / IdParam）
5. Service + Impl（extends ServiceImpl）
6. Controller（page/add/edit/delete/detail）

## 精确 import 列表
```java
import cn.hutool.core.util.StrUtil, ObjectUtil, IdUtil;
import cn.hutool.core.collection.CollectionUtil;
import cn.hutool.core.bean.BeanUtil;
import com.baomidou.mybatisplus.core.conditions.query.QueryWrapper;
import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import com.baomidou.mybatisplus.extension.plugins.pagination.Page;
import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;
import vip.xiaonuo.common.page.CommonPageRequest;
import vip.xiaonuo.common.pojo.CommonResult, CommonEntity;
import vip.xiaonuo.common.annotation.CommonLog;
import vip.xiaonuo.common.enums.CommonSortOrderEnum;
import jakarta.annotation.Resource;
import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank, NotEmpty;
```