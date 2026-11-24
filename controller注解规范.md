# OpenAPI 接口文档注解规范（供 Agent 执行）

## 1. 适用范围与执行原则

适用于使用 Spring MVC、springdoc-openapi 和 `io.swagger.v3.oas.annotations` 的 Java 项目。本文件定义接口文档的编写规则，不授权修改业务行为、接口协议或项目依赖。

Agent 收到“添加或完善接口注解”的任务时，必须遵循：

1. 先阅读目标 Controller、请求与响应 DTO、直接调用的业务逻辑，以及相关的统一响应、异常处理、认证和校验配置。
2. 查阅项目依赖版本和附近已有的合规示例，再决定使用哪些注解属性。
3. 以实际路由、签名、校验、响应和业务实现为事实依据。已有文档与实现不一致时，应指出差异，不得为了文档完整而编造行为。
4. 只修改完成任务所需的注解和说明，不顺带调整路由、字段、方法签名、校验规则、权限或返回结构。
5. 无法确认且不影响基本描述的信息可以省略并在交付时说明；影响接口含义或契约的关键问题必须提出，不能猜测。

“必须”表示本规范的团队要求；“按需”表示有实际信息需要表达时才添加。不要为了凑齐注解而堆叠注解。

## 2. 注解体系与放置位置

| 注解 | 放置位置 | 要求 |
| --- | --- | --- |
| `@Tag` | Controller 类 | 必须，按业务能力分组 |
| `@Operation` | 对外 API 的 Controller 映射方法 | 必须，描述具体操作 |
| `@Parameter` | 查询、路径、请求头等简单参数 | 按需，补充业务含义、示例、单位或取值说明 |
| `@Schema` | 请求/响应 DTO 类型及属性 | DTO 应有用途说明；自有业务字段必须有清楚的含义说明 |
| `@ApiResponse` / `@ApiResponses` | Controller 方法或项目统一文档配置 | 生成文档必须反映真实响应，不要求每个方法重复公共响应 |
| `@SecurityRequirement` | Controller 类或方法 | 按项目已有安全方案和真实认证要求使用 |
| OpenAPI 的 `@RequestBody` | 请求体参数或 `@Operation` 配置 | 仅在确需补充请求体文档时使用 |

- 普通 Service、Repository 不添加上述接口文档注解。DTO 的说明放在 DTO，不在业务层重复维护。
- 如果项目采用 Controller 接口承载注解，沿用现有承载位置，避免接口与实现重复、冲突。
- 新增 OpenAPI 3 文档代码统一使用 `io.swagger.v3.oas.annotations`，不新增 Swagger 2 的 `@Api`、`@ApiOperation`、`@ApiParam`、`@ApiModelProperty`。
- 如果目标项目仍使用 Swagger 2，不得直接混入 OpenAPI 3；说明现状并确认迁移范围。
- 不为使用某个注解属性擅自升级 Spring Boot、springdoc 或 swagger-annotations。

## 3. `@Tag`：按业务能力分组

```java
@Tag(name = "邮箱认证", description = "邮箱验证码发送与登录")
```

- `name` 必须填写，使用稳定、简短的业务名称，例如“邮箱认证”“订单管理”。
- 禁止使用 `EmailAuthController`、接口类全名、“其他接口”等技术性或模糊名称。
- `description` 按需补充分组范围；只是重复 `name` 时省略。
- 一般只在类上声明，方法上不重复设置 `@Operation(tags = ...)`。确有跨业务分组需求时沿用项目约定。
- 不因方法新增而随意改名已有分组。

## 4. `@Operation`：准确描述单个操作

### 4.1 `summary` 必须填写

采用“动词 + 业务对象”，简短明确，不加句号：

```java
@Operation(summary = "发送邮箱验证码")
@Operation(summary = "使用邮箱验证码登录")
@Operation(summary = "查询当前用户信息")
```

禁止使用“处理请求”“查询接口”“用户相关操作”等无法区分业务用途的标题。

### 4.2 `description` 按需填写

用于补充调用方必须了解、且已经确认的规则：前置条件、数据范围、关键限制、幂等行为、异步处理或其他业务副作用。

- 不重复 `summary`，不复述方法名，不描述 Redis、SQL、Service 等内部实现。
- 有效期、发送频率、次数、金额、权限、验证码使用规则等必须从实现或明确需求确认。
- 不写“由服务控制”“详见代码”“按业务规则处理”这类不能帮助调用方的信息。
- 动态配置的规则只能按真实机制描述；不要把当前环境的值写成永久契约。
- 没有值得补充的信息时，省略 `description` 完全合规。

例如，原文“验证码有效期、发送频率及单次使用规则由验证码服务控制”应当改为已核实的外部规则；无法核实时先省略，不得填入臆测的分钟数或发送间隔。

### 4.3 `operationId` 按需维护

- 项目已有显式 `operationId` 约定或 SDK 生成需求时，沿用其命名规则。
- 显式设置时，必须在生成的 OpenAPI 文档中唯一，并在接口语义不变时保持稳定。
- 项目没有相关要求时，不为了完整而手写。

## 5. 参数与 DTO

### 5.1 简单参数使用 `@Parameter`

```java
@Parameter(description = "订单编号", example = "ORD202611170001")
@PathVariable String orderNo
```

- 补充参数的业务含义；有歧义时说明单位、格式、允许值或默认行为。
- 路径、位置、类型和必填信息优先由实际 Spring 注解、类型及校验约束推导。
- 不重复声明能够正确推导的信息；确需覆盖时，必须与实际绑定规则一致。
- `example` 是合法、虚构、可公开的示例，不是默认值，不会自动传给接口。
- 不使用真实手机号、邮箱、密码、令牌、内部地址或其他敏感数据作为示例。
- 分页对象等复杂查询对象沿用项目现有约定；确需使用 `@ParameterObject` 时，先核实当前 springdoc 版本中的包名和行为。

### 5.2 DTO 使用 `@Schema`

- DTO 类型说明其用途，例如“发送邮箱验证码请求”。
- 字段或项目已有的属性承载位置说明业务含义。沿用字段、getter 或 record component 的现有风格，不在多处重复维护。
- 金额说明单位；时间说明格式及适用时区，时间戳说明秒或毫秒；状态说明各取值含义；分页说明起始页及已实现的限制。
- 枚举、长度、范围、必填、只读和只写等信息必须与实现一致。
- 不给第三方类型或整个 DTO 图无边界补注解，只处理本次接口实际涉及的自有类型。

```java
@Schema(description = "接收登录验证码的邮箱地址", example = "user@example.com")
private String email;
```

### 5.3 文档与校验分别负责

- `@Schema`、`@Parameter` 只描述契约，不会执行格式校验、非空校验、限流或权限检查。
- 必填和范围以实际绑定行为、Bean Validation 分组及自定义校验逻辑为准，不是看到某个注解就直接下结论。
- 不为了让文档显示“必填”而添加 `@NotBlank` 等会改变运行行为的注解。
- 若现有约束不能正确生成文档，才按当前依赖版本补充文档属性；支持 `requiredMode` 时优先使用该属性，不为此升级依赖。
- `@Schema(accessMode = WRITE_ONLY)` 等文档属性不会自动阻止实际响应序列化敏感字段。

### 5.4 区分两个 `@RequestBody`

- `org.springframework.web.bind.annotation.RequestBody` 负责 Spring 请求体绑定。
- `io.swagger.v3.oas.annotations.parameters.RequestBody` 负责请求体文档。
- 不得把前者替换成后者。需要同时使用时，采用项目既有的消歧方式或对其中一个使用全限定名。

## 6. 响应与错误说明

生成的文档必须能看出成功状态码、实际响应结构，以及调用方需要处理的主要失败情况。

- 先查看返回类型、`ResponseEntity`、`@ResponseStatus`、统一响应包装、`ResponseBodyAdvice` 和全局异常处理。
- 状态码必须与实际 HTTP 状态一致。不能把响应体里的业务码当成 HTTP 状态码。
- 公共错误响应优先复用项目现有统一配置，接口特有响应在方法上补充。
- 若实际返回 `Result<UserResponse>`，文档必须保留包装层和业务数据类型，不能只声明 `UserResponse` 或粗略改成 `Object`。
- 分页、集合、文件下载、流式响应及媒体类型按实际协议描述；不要套用普通 JSON 模板。
- 没有响应体时不得声明虚构的 JSON DTO；HTTP 204 的无响应体可用 `content = @Content` 明确表达。
- 不机械添加 `400`、`401`、`403`、`404`、`429`、`500`。每个状态必须有真实依据；例如存在限流逻辑，也要核实最终 HTTP 状态再声明 `429`。
- 错误体的 `code`、`message` 等含义应复用实际公共错误模型，不在注解中创建另一套协议。

## 7. 认证与隐藏

- `@SecurityRequirement(name = "...")` 只能引用项目已定义的安全方案，名称必须一致。
- 按真实配置区分公开接口、登录接口和需要认证的接口；不要给所有 Controller 统一套用同一个认证要求。
- 类级别或全局安全配置存在时，必须核实公开方法的最终生成结果，不能假定“不加注解就会取消继承”。
- 额外的角色、权限或数据范围要求，可以写入已核实的操作说明。
- `@SecurityRequirement` 不执行访问控制；`hidden = true` 只隐藏文档，不会关闭接口。
- 隐藏接口或参数必须有明确依据，不能为了减少文档工作量而隐藏。

## 8. 一致性示例

以下示例仅说明注解组织方式，假设 Java 17、Spring Boot 3，以及接口原本就采用 HTTP 204 成功响应、已有相应业务方法和校验规则。它不是修改现有接口的授权，也不是要求引入 record。

示例省略 import；文档注解来自 `io.swagger.v3.oas.annotations` 相应子包，Spring 注解来自 Spring MVC，校验注解沿用项目的 `jakarta.validation` 或 `javax.validation` 版本。

```java
@Tag(name = "邮箱认证", description = "邮箱验证码发送与登录")
@RestController
@RequestMapping("/auth/email")
public class EmailAuthController {

    private final EmailCodeService emailCodeService;

    public EmailAuthController(EmailCodeService emailCodeService) {
        this.emailCodeService = emailCodeService;
    }

    @Operation(summary = "发送邮箱验证码")
    @ApiResponse(
        responseCode = "204",
        description = "验证码发送成功",
        content = @Content
    )
    @PostMapping("/code")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void sendCode(@Valid @RequestBody SendEmailCodeRequest request) {
        emailCodeService.sendLoginCode(request.email());
    }
}
```

```java
@Schema(description = "发送邮箱验证码请求")
public record SendEmailCodeRequest(

    @NotBlank(message = "邮箱不能为空")
    @Email(message = "邮箱格式不正确")
    @Schema(
        description = "接收登录验证码的邮箱地址",
        example = "user@example.com"
    )
    String email
) {
}
```

该示例没有编造验证码有效期、发送间隔、单次使用机制或错误状态码。实际任务中应先核实，再补充有依据的说明。

## 9. Agent 执行与验收清单

### 修改前

- [ ] 已读取目标接口、直接调用逻辑和涉及的 DTO。
- [ ] 已核实注解体系、Java/框架版本、现有注解承载位置。
- [ ] 已核实实际路由、参数绑定、校验、响应包装、错误处理和认证规则。
- [ ] 已识别不确定信息；没有用常识或示例值代替事实。

### 修改后

- [ ] `@Tag` 是业务分组，`@Operation.summary` 能准确区分操作。
- [ ] `description` 有实际增量信息；没有重复标题、内部实现说明或空泛表述。
- [ ] 参数和字段说明中的单位、格式、允许值、必填信息与实现一致。
- [ ] 示例合法且不包含敏感数据；数值和业务限制都有明确来源。
- [ ] 响应状态、包装层、泛型、媒体类型和安全声明与实际一致。
- [ ] 没有 Swagger 2/3 混用、错误 import、重复注解或不存在的属性。
- [ ] 没有改动任务范围外的路由、业务代码、校验、权限、依赖和响应协议。

### 验证与交付

1. 运行项目现有的编译或相关检查，确认注解和属性适配当前依赖。
2. 能运行应用时，检查实际配置的 OpenAPI 输出；常见默认路径为 `/v3/api-docs`，不能假定项目未修改路径。
3. 在输出中核实分组、操作说明、参数、DTO Schema、响应和安全要求；可用时再检查 Swagger UI 的可读性。
4. 不为纯文档注解修改添加没有业务价值的测试；项目已有文档契约检查时按约定执行。
5. 交付时明确说明修改范围、已执行的检查、未确认信息及未执行检查的原因。未运行应用时，不得声称已验证生成文档。

## 10. 给 Agent 的执行指令

> 请按本文件规范维护接口文档注解。先读取实现与项目约定，只添加有事实依据的信息。默认不修改业务行为、依赖和响应协议；可推导的信息避免重复声明。缺少可选信息时省略并说明，缺少关键事实时提出问题。完成后检查 diff、执行可用验证，并如实报告未验证内容。

