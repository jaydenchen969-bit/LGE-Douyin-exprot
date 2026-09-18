# 抖音销帮 UI 完整性复核：未通过

## 结论
不能确认“没有遗漏”。本次重新检查对话交付的 xiaobang_all_ui_workpack.zip，发现原页面总表有明确漏登记路由。G0 保持 in_progress，scope_closed=false，total_business_screens=null。目录、组件和入口数量均不能充当全量界面完成率。

## 实际核验
1. 重算 SHA256SUMS.json 所列 462 个文件：缺失 0、哈希不符 0。
2. 复核 source-index.json 所列 308 个资源：缺失 0、哈希不符 0。
3. 重新静态解析 219 个符合原过滤条件的 JS 文件，复现 213 条字面量路由候选，解析错误 0。
4. 加查非字面量 path/route，发现 124 条此前该扫描器未采集的路由形态声明。这些含框架代码、图表树节点、重复宿主声明，以及已经由 CRM 专用解析补齐的路径，不能当作 124 个新增页面。

## 明确漏登记的路由
以下在原源码中有明确声明，而当前 screen-registry.json 没有对应记录。只确认源码声明，运行时可达性和视觉仍待验证。

| 模块 | 漏登记内容 | 常量或表达式 |
|---|---|---|
| 消息 | 消息列表、消息设置、页面未找到、根入口重定向 | u.ActivityList=/list；u.Setting=/setting；u.NotFound=/404；u.RootPath=/ |
| 日报 | 日报列表、我的日报、编辑日报、新建日报、日报详情、同组日报 | y.List=/list；y.MyDaily=/my_daily；y.Edit=/edit；y.Create=/create；y.Detail=/detail；y.Group=/group |
| 首页 | 首页根入口、Deprecated 旧版入口 | tt.Root=/；tt.Deprecated=/deprecated |
| 工具 | 商品管理、品牌管理、直播计划、商品榜单、解决方案、热点雷达/视频效果、库存巡检等 | 路由表中 44 个对象使用 ee/et/ea 前缀或拼接表达式，被原扫描器漏采集；含容器和重定向，不能等同 44 个页面 |

消息、日报、首页合计 12 条去重后确认漏登记的路由声明；包含重定向、兜底、旧版入口、模式复用，不能换算为 12 个不同业务界面。

## 根因与其他覆盖缺口
- tools/census.cjs 的 lit() 只接受 Literal。path:u.Setting、path:y.Detail、''.concat(ee,'/stock_check') 都被跳过。
- 父路由使用常量或拼接时，parent_paths 同样可能漏取。工具模块 list/search/detail 等子路由缺少完整父级上下文。
- tools/extract_crm_routes.py 只枚举 Food、General、Travel、Takeaway、Self 五种 businessType，固定 grayScales=[]；未验证全部业务类型、权限与灰度组合。
- tools/sweep_crm.py 按 entry_module 去重，每个共享组件只选一条路径试运行；不能代替所有路径、参数、模式及数据状态验证。
- 原登记表共 396 条混合证据记录：372 条 not-tested、15 条 browsable-state、7 条 blocked、2 条 fixture-incomplete。15 条可浏览记录都在 CRM，有空态和不可查看协议等提示态。
- 87 条 CRM 记录的 states_pending 仍是同一句待枚举说明，其他 309 条未列逐页状态清单。
- 所有 396 条 visual_status=reference-missing、design_status=not-created；24 个已试运行组件 full_page_complete 均为 false。
- all-in-one 动态配置、非 CRM 深层懒加载、原生壳、同版本同角色参考仍未闭合。
- llm_agent_runs=[]。独立 LLM 子代理启动数为 0；已有 3 个并行浏览器 worker 不能作为子代理执行证明。

## 补漏任务
| ID | 主责 | 要求 |
|---|---|---|
| COV-01 | SA01 | 补常量、枚举、拼接及 JSX 路由解析；124 条线索逐项归类并记录排除理由 |
| COV-02 | SA04、SA06 | 消息、日报、首页、工具漏登记路由归一化，恢复父路由与宿主路径 |
| COV-03 | SA01、SA03 | 业务类型、权限与灰度取值矩阵；未知配置保留为缺口 |
| COV-04 | 各模块、SA08 | 页面×参数/模式×角色×状态逐项登记和测试，不能按组件复用跳过 |
| COV-05 | SA01、SA05 | 动态页面目录与懒加载依赖闭合；缺资源明确标记 |
| COV-06 | SA06、SA07、SA08 | 原生壳、字体、可编辑设计和同版本视觉验收 |

## 验收与更改边界
只有来源路由、原生声明、动态配置、导航入口完成交叉对账，且差集每项都有登记或可复核排除理由，角色和状态明确，才可评估范围完整性。有未知项就不签“全量无遗漏”。
本次提交只记录审计结论与补漏任务，尚未修复主扫描器、更新原工作包或新增页面实现。未登录、未调用业务 API、未修改原服务。

## 可复现来源
全部路径相对于当前工作包的 xiaobang_full_ui/：
- 消息：source-supplement/assets/e4e839caebc818fc4251.js，常量在约 800 字符附近，遗漏路由声明 offset 1094、1211、1290、1397。
- 日报：source-supplement/assets/92b12bff607cf6782f0b.js，module 46013 的枚举与 6 条路由，offset 1216660 至 1217180；module 41116 有带宿主前缀的重复声明，需去重。
- 首页：source/assets/010.js，tt 枚举及 offset 78601、78638 的路由声明。
- 工具：source/assets/006.js，路由表 offset 1255074 至 1261544，ee/et/ea 为宿主相关前缀。
- 原盘点规则：tools/census.cjs；CRM 业务类型枚举：tools/extract_crm_routes.py；按组件去重运行：tools/sweep_crm.py。
- 机器可读补漏证据与完整脚本保存在本轮对话附件 xiaobang_verification/，原始源码保持不变。
