# 代码评审结论

**建议修改后再合入。** 此次变更主要调整了 GitHub Actions 的触发范围，并新增了一个从 GitHub Release 下载 JAR、携带多个密钥执行代码评审的工作流。主要风险集中在：

1. **提交信息进入 shell 命令，存在命令注入风险。**
2. **未经完整性校验的远程 JAR 可读取工作流中的敏感凭据。**
3. **PR 分支、提交信息和 diff 范围可能不准确。**
4. **新增工作流没有构建或测试当前变更，无法替代常规 CI。**

以下评审仅基于给出的 diff；由于没有看到 SDK 实现及完整 `pom.xml`，涉及 JAR 内部 diff 算法、异常处理和 Shade 插件原始配置的部分，需要进一步验证。

---

# 一、必须优先处理的问题

## P0：提交信息直接插入 shell，可能造成命令注入

**位置：** `.github/workflows/remote.yml`，`Print repository, branch name, commit author, and commit message` 步骤。

```yaml
run: |
  echo "Commit author is ${{ env.COMMIT_AUTHOR }}"
  echo "Commit message is ${{ env.COMMIT_MESSAGE }}"
```

`COMMIT_AUTHOR` 和 `COMMIT_MESSAGE` 来自 Git 提交元数据，不能视为可信内容。GitHub Actions 会先替换 `${{ ... }}`，再把结果交给 shell 解析。如果提交信息包含双引号、命令替换等 shell 特殊字符，可能改变原本的脚本语义并执行额外命令。

这是安全边界问题：同一个 job 后续还会运行带有多个 secrets 的 JAR。即使注入发生在传入 secrets 的步骤之前，攻击者也可能尝试修改后续执行环境、文件或 JAR。

### 修改建议

不要把不可信内容通过 `${{ }}` 直接拼入 `run` 脚本。若这些值已经安全写入环境变量，可由 shell **直接展开变量并使用 `printf`**：

```yaml
- name: Print metadata
  run: |
    printf 'Repository name is %s\n' "$REPO_NAME"
    printf 'Branch name is %s\n' "$BRANCH_NAME"
    printf 'Commit author is %s\n' "$COMMIT_AUTHOR"
    printf 'Commit message is %s\n' "$COMMIT_MESSAGE"
```

同时，前面的 `$GITHUB_ENV` 写入也应考虑特殊字符和换行。当前的：

```yaml
echo "COMMIT_MESSAGE=$(git log -1 --pretty=format:'%s')" >> $GITHUB_ENV
```

采用的是单行环境文件格式，不能作为任意文本的通用编码方式。建议对自由文本使用随机分隔符的多行格式，或先编码后传递；同时对长度和控制字符做限制。GitHub 官方文档也建议，当值完全不可控时优先使用文件传递，避免分隔符碰撞。

另外，考虑该工作流是否确实需要打印作者邮箱和完整提交信息；不需要时直接删除打印步骤，可减少日志中的个人信息暴露。

---

## P0：远程 JAR 未校验，且执行时持有多个 secrets

**位置：** `.github/workflows/remote.yml`

```yaml
- name: Download openai-code-review-sdk JAR
  run: wget -O ./libs/openai-code-review-sdk-1.0.jar https://github.com/zdkk619/openai-code-review/releases/download/v1.0/openai-code-review-sdk-1.0.jar
```

随后运行该 JAR 时，注入了 GitHub token、微信密钥和模型 API key。任何被执行的 JAR 都可以读取这些环境变量，并访问 runner 能访问的网络。

这里的风险不是 `wget` 本身，而是**远程制品来源、制品完整性以及制品所获得的权限之间缺少约束**。例如，Release 资产或发布账户一旦被替换或入侵，该工作流可能执行恶意制品。

### 修改建议

1. **优先使用可信构建产物**：在受保护的构建/发布流程中构建 SDK，并固定制品版本。
2. **校验 SHA-256 或制品签名**：期望值应通过受保护、独立可信的渠道维护，不能仅与 JAR 从同一位置同时下载。
3. **固定并审核发布来源**：对发布权限、Release 资产修改权限建立控制。
4. **最小化密钥权限**：
   - 日志仓库 token 仅授予目标仓库必要的写权限；
   - 不需要微信通知的场景不要传入微信密钥；
   - 模型 key 设置额度、用途和供应商侧访问限制。
5. 考虑使用短期凭证、制品溯源或签名验证，而不仅依赖版本号。

示意：

```yaml
- name: Download and verify SDK
  env:
    SDK_URL: https://github.com/zdkk619/openai-code-review/releases/download/v1.0/openai-code-review-sdk-1.0.jar
    # 应维护为实际可信摘要，而不是占位值
    SDK_SHA256: "<trusted-sha256>"
  run: |
    set -euo pipefail
    mkdir -p libs
    curl --fail --location --retry 3 \
      "$SDK_URL" \
      --output libs/openai-code-review-sdk-1.0.jar
    printf '%s  %s\n' "$SDK_SHA256" \
      libs/openai-code-review-sdk-1.0.jar | sha256sum --check
```

---

## P1：工作流可能没有评审到预期的代码范围

**位置：**

```yaml
with:
  fetch-depth: 2
```

以及：

```yaml
run: java -jar ./libs/openai-code-review-sdk-1.0.jar
```

当前工作流没有显式传递 base/head SHA，评审范围依赖 SDK 的内部实现。仅获取两个提交，通常不足以覆盖以下场景：

- 一次 push 包含多个提交；
- PR 包含多个提交；
- PR 的共同祖先不在浅克隆历史中；
- 首次提交或强制推送；
- PR 的默认 checkout 是 GitHub 生成的合并提交，而不是开发者最后一个原始提交。

因此，**不能仅凭该 diff 确定评审一定错误**，但当前工作流无法保证 SDK 拿到正确范围，需要检查 SDK 如何计算 diff。

### 建议明确评审语义

| 事件 | 建议范围 | 注意事项 |
|---|---|---|
| `push` | 事件中的 `before` 到 `after` | 首次 push、强推、提交对象不可用需单独处理 |
| `pull_request` | PR base/head 的共同祖先到 head | 需确保相关对象和历史已获取 |
| PR 合并结果检查 | 合并结果相对目标分支 | 与“只评审 PR 原始变更”语义不同，应明确选择 |

可优先使用：

```yaml
with:
  fetch-depth: 0
```

对于大型仓库，也可以按明确的 base/head SHA 定向 fetch，但仍需确保计算 merge-base 所需的历史可用。更重要的是，**SDK 应接收明确的比较端点，或工作流生成 diff 文件并交给 SDK**；仅调整 `fetch-depth` 不足以保证评审范围正确。

建议增加测试：单提交 push、多提交 push、多提交 PR、目标分支前进、首次 push、强推及浅克隆历史不足。

---

## P1：新增工作流没有验证当前代码能否构建和通过测试

**位置：** `.github/workflows/remote.yml`

新增工作流设置了 JDK，但随后下载并运行的是 Release 中的固定 JAR：

```yaml
run: java -jar ./libs/openai-code-review-sdk-1.0.jar
```

从本次 diff 看，工作流没有执行当前提交的 Maven 构建或测试。因此，即使自动评审成功，也不能说明：

- 当前 SDK 源码可以编译；
- 当前变更的单元测试通过；
- 当前变更打出的 JAR 可以运行；
- Release JAR 与本次变更代码一致。

`main.yml` 的其余内容没有给出，不能断言仓库完全没有构建测试；但应确认仍有独立的 CI 覆盖所有需要的分支和 PR。

### 修改建议

将**构建测试**与**自动代码评审**拆成两个职责清晰的 job 或 workflow。例如，在独立构建 job 中执行：

```yaml
- name: Build and test
  run: mvn --batch-mode --no-transfer-progress verify
```

然后根据产品意图，明确评审 job 使用的是“经过发布验证的 SDK”还是“当前变更构建的 SDK”。如果执行来自 PR 的产物，不应同时暴露高权限 secrets。

---

# 二、触发条件与 PR 元数据问题

## P1：PR 分支名获取不正确

```yaml
echo "BRANCH_NAME=${GITHUB_REF#refs/heads/}" >> $GITHUB_ENV
```

对普通分支 push，`GITHUB_REF` 通常形如 `refs/heads/main`。但在 `pull_request` 事件下，通常形如：

```text
refs/pull/123/merge
```

因此替换后得到的仍可能是 `refs/pull/123/merge`，而不是 PR 源分支名。如果 SDK 用 `COMMIT_BRANCH` 生成日志路径或报告名称，结果会与预期不符。

建议区分事件：

```yaml
- name: Set branch name
  env:
    PR_HEAD_REF: ${{ github.head_ref }}
    REF_NAME: ${{ github.ref_name }}
  run: |
    set -euo pipefail
    branch="${PR_HEAD_REF:-$REF_NAME}"
    printf 'BRANCH_NAME=%s\n' "$branch" >> "$GITHUB_ENV"
```

如需支持特殊字符的通用安全传递，仍应采用前述多行格式或文件传递。对于 PR，还建议同时记录目标分支、PR 编号、head SHA 和 base SHA，不要仅靠分支名标识一次评审。

## P1：PR 的作者和提交信息可能取到了合并提交

```yaml
git log -1 --pretty=format:'%an <%ae>'
git log -1 --pretty=format:'%s'
```

`pull_request` 默认 checkout 的对象可能是合并提交，因此 `git log -1` 返回的作者和标题不一定是 PR 源分支最后一次提交的信息。

建议先明确业务含义：

- 要记录 **PR 信息**：使用 PR 编号、标题、发起者和 head/base SHA；
- 要记录 **head 提交信息**：基于 PR head SHA 读取；
- 要评审 **全部变更**：基于正确 diff 范围，而不是只看 `git log -1`。

## P1：外部 Fork PR 的 secrets 通常不可用

新增工作流会监听 PR，但从 fork 发起的 PR 在常见配置下不会获得仓库 secrets。这可能导致 JAR 因缺少模型 key、日志 token 或微信配置而失败；实际结果取决于 SDK 对缺失配置的处理。

建议明确产品策略：

- 对不具备凭据的 PR，跳过需要 secrets 的评审步骤，并在日志中给出原因；
- 或通过受保护、需人工批准的可信流程处理外部贡献；
- 不要为了解决 secrets 不可用，直接切换到 `pull_request_target` 并执行不可信 PR 代码。

另外，即使 PR 来自同一仓库，也应根据贡献者和分支权限判断是否可信；不能仅凭“非 fork”就认为可以安全地执行任意代码并提供 secrets。

---

# 三、触发范围与重复执行

## P1：`['*']` 不匹配包含 `/` 的分支名

```yaml
push:
  branches: ['*']
pull_request:
  branches: ['*']
```

GitHub Actions 分支过滤中的 `*` 不匹配 `/`。例如：

- `main`：匹配；
- `feature/login`：不匹配。

如果预期覆盖所有分支，应考虑 `['**']`，或直接删除不需要的分支过滤器。

还需注意：`pull_request.branches` 过滤的是 **PR 的目标分支**，不是源分支。

## P2：`main-close` 可能同时触发两个工作流

本次变更后：

- `main.yml` 仅监听 `main-close`；
- `remote.yml` 监听匹配 `*` 的分支，也包括 `main-close`。

因此针对 `main-close` 的 push 或目标为 `main-close` 的 PR，两个工作流可能同时运行。是否重复评审，要看未展示的 `main.yml` 剩余内容，但至少会产生两个 workflow run。

此外，两个工作流名称完全相同：

```yaml
name: Run Java Git Diff By Local
```

在 Actions 页面和分支保护规则中容易混淆。

建议：

1. 分别命名，例如 `Local Code Review`、`Released SDK Code Review`；
2. 明确两者是否需要并行；
3. 不需要重复运行时，调整过滤条件或合并为一个 workflow；
4. 对同一 PR 的过期运行配置适当的 `concurrency`，减少重复模型调用和通知。

---

# 四、凭据和数据治理

## P1：默认 checkout 可能保留 Git 凭据

`actions/checkout` 默认会配置凭据以供后续 Git 操作使用。如果评审流程不需要用 checkout 凭据推送当前仓库，建议关闭：

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0
    persist-credentials: false
```

同时显式声明工作流权限，例如：

```yaml
permissions:
  contents: read
```

这只是降低默认 `GITHUB_TOKEN` 和 checkout 凭据的暴露面，**并不能替代对 `secrets.CODE_TOKEN` 的最小权限控制**；后者是显式传给 JAR 的独立凭据。

## P1：源码可能发送到外部模型和日志服务

工作流配置了模型服务、日志仓库和微信通知。若 SDK 将 diff 或评审结果发送到这些系统，可能涉及源码、密钥误提交、客户信息等敏感内容。

建议在 SDK 和使用规范中明确：

- 实际发送哪些文件、diff 和元数据；
- 是否排除密钥文件、证书、生产配置、生成文件及大文件；
- 外部日志仓库的访问范围和保留周期；
- 模型服务商的数据使用及留存策略；
- 微信通知是否只发送摘要和受控链接；
- 外部服务失败是否阻断 CI，以及超时、重试和费用上限。

这些属于条件性风险：具体数据流仍需检查 SDK 实现。

---

# 五、`dependency-reduced-pom.xml` 评审

## P2：不建议提交自动生成的 reduced POM

新增文件：

```text
openai-code-review-sdk/dependency-reduced-pom.xml
```

该文件通常是 `maven-shade-plugin` 的构建产物。把它提交到源码仓库容易产生：

- 与真实 `pom.xml` 不同步；
- 构建后出现无关 diff；
- 开发者误以为它是需要手工维护的主要构建配置。

建议先确认项目是否**有意依赖该文件发布或安装制品**。如果没有，移除并加入 `.gitignore`：

```gitignore
dependency-reduced-pom.xml
```

如果构建确实不需要生成它，可在**原始 `pom.xml`** 的 Shade 插件配置中评估：

```xml
<createDependencyReducedPom>false</createDependencyReducedPom>
```

该设置可能影响发布到 Maven 仓库时的依赖元数据，需结合发布方式验证，不能只为消除文件就直接修改。

## P2：不要仅凭 `provided` 判断可执行 JAR 缺少依赖

reduced POM 中多个依赖被标记为：

```xml
<scope>provided</scope>
```

这是需要关注的元数据，但**不能仅凭它断定 shaded JAR 运行时缺依赖**：被打包进 fat JAR 的依赖与 reduced POM 中的声明有不同用途。

建议在干净环境中对最终制品做冒烟测试，确认：

- `java -jar` 可以启动；
- Main-Class 正确；
- 服务加载器资源合并正确；
- 日志实现、JGit、OpenAI 客户端等在实际使用路径下可加载；
- 发布后的 POM 元数据符合下游使用预期。

## P2：生成文件中 transformer 信息不完整，需核对原始配置

文件中出现：

```xml
<transformers>
  <transformer>
    <mainClass>com.zdkk.middleware.sdk.OpenAiCodeReview</mainClass>
  </transformer>
  <transformer />
</transformers>
```

其中看不到 transformer 的具体实现类型。由于这是生成的 reduced POM，**不应直接据此断言当前构建失败**；但若它被当作独立构建配置使用，可能无法正确执行。

建议检查原始 `pom.xml` 中是否正确配置了所需 transformer，例如 Manifest 和服务资源合并，并以实际构建和运行测试为准。

## P2：测试资源可能被打入生产 JAR

新增 reduced POM 的主构建资源中包含：

```xml
<resource>
  <directory>src/test/resources</directory>
  ...
</resource>
```

这可能导致测试配置、样例数据或测试凭据进入生产制品。由于还需确认原始 POM 及实际构建行为，建议检查：

```bash
jar tf path/to/sdk.jar
```

测试资源通常应配置在 `<testResources>`，仅供测试使用，而不是放入生产 JAR。

---

# 六、建议的落地顺序

1. **立即修复命令注入**：删除 `${{ env.COMMIT_* }}` 在 `run` 中的直接插值，并安全传递自由文本。
2. **收紧执行边界**：校验远程 JAR、限制 secrets 权限、关闭不必要的 checkout 凭据持久化。
3. **明确并验证评审范围**：区分 push 和 PR 的 base/head；测试多提交、合并提交、首次 push 和强推。
4. **恢复或确认常规 CI 覆盖**：确保当前提交会执行构建、测试及制品冒烟测试。
5. **修正触发条件和元数据**：处理包含 `/` 的分支名、PR 源/目标分支及重复运行。
6. **清理生成文件**：核实 reduced POM 的用途、Shade 配置和生产 JAR 内容。
7. **建立数据治理规则**：明确源码发送范围、外部服务权限、日志保留、失败策略和费用控制。

其中，第 1、2 项属于安全阻塞项；第 3、4 项关系到评审结果和 CI 结果是否可信，应在上线自动评审前完成。