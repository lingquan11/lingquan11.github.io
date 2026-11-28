---
layout: post
title: "CNB现代开发者实践课复盘（下）：Agent Loop、防假测试与仓库里的NPC"
date: 2026-11-28 23:42:00 +0800
categories: 深研自述
tags: [Agent]
---

## 1. 问题与背景

接上篇的Git和Docker底座，这篇记课程后三章（`03-ai-intro`、`04-vibecoding`、`05-hackathon`）和对应的三组作业（`ai-exercises`、`agent-exercises`、`app-exercises`）。

之前做1×4脉动阵列和C++矩阵乘法时，我已经高频用Claude Code帮着写代码和对拍脚本了，但很多时候全凭手感在调，对底层怎么调工具、为什么测出来全绿还会撞上假测试，其实没系统捋过。这次跟着课程把整条线走了一遍：单次API调用→手写Agent Loop→Coding Agent研发流→仓库里的NPC多智能体→Flask+Docker打包交付，很多之前踩过的体感问题算是对上号了。

第3章的3个Python demo和两套实操、6道`ai-exercises`（`system_prompt_switcher`、`tool_calling_agent_loop`、`memory_multi_turn`、`token_observer`、`rag_retriever`、`tool_guard`）、第4章的贪吃蛇与假测试反例、2道`agent-exercises`，以及第5章预演`01-llm-web-app`，我都在CNB环境里跑通了。这篇只拆跑通的机制、协议细节和源码逻辑。

---

## 2. LLM交互协议：无状态、Token与流式约束

### 2.1 OpenAI兼容接口与CNB网关的流式限制

现在调大模型基本都走OpenAI的Chat Completions格式，换不同厂商或网关通常只改`base_url`、`api_key`和`model`。

CNB内置网关的地址规则是`{CNB_API_ENDPOINT}/{CNB_REPO_SLUG}/-/ai-ide/v2`（默认模型`deepseek-v4.1-flash`）。在云原生开发环境里，平台会自动注入`CNB_API_ENDPOINT`、`CNB_REPO_SLUG`和`CNB_TOKEN`，代码里不用硬编码密钥就能直接跑；本地离线跑时，才在仓库根目录建`.env`填`BASE_URL`/`API_KEY`/`MODEL_ID`，由共用模块`_env.py`里的`load_dotenv()`向上查找读取。

CNB这套接口有个硬限制：**目前只支持流式请求，必须显式传`stream=True`**。发非流式请求会直接吃到`HTTP 400 · code 11101 · "Non-stream chat request is currently not supported"`。所以课程的`_env.py`和`llm_client.py`都是在内部强制开流式，遍历chunk把`delta.content`拼完再返回整段文本。

### 2.2 无状态本质与System Prompt为什么管用

发给模型的不是一句散话，而是一个结构化的`messages`列表，基础对话包含`system`、`user`、`assistant`三种角色。

`ai-exercises/memory_multi_turn.py`验证得很直白：第一轮告诉模型「我叫小明，我最喜欢的颜色是蓝色」，第二轮问「我最喜欢的颜色是什么」。模型本身根本没有状态记忆，每轮能看到的只有这次发过去的`messages`列表——程序必须把第一轮的`user`和模型返回的`assistant`原样追进列表全量重发，它才「记得住」。

至于`system_prompt_switcher.py`里只改一行`system`就能把技术助手切成翻译机，背后也不是玄学，而是两件事叠加：一是`system`固定在序列最开头，而模型对开头和结尾的注意力权重最高，中间长文容易出现Lost in the Middle；二是后训练微调阶段模型见过大量「照着system设定回答user」的样本，照做是训出来的习惯。

### 2.3 Token、BPE与参数对应的底层动作

模型不认字只认数字，文本进去前先要切分成Token再转成ID和向量。按单词切词表会爆炸且遇到新词抓瞎，按单字符切又没语义、序列太长。现在主流大模型走的是子词分词BPE：从UTF-8的256个最小字节起步，统计词内相邻单元共现频率，反复把最高频的一对合并成新token，直到词表达到预设大小（通常3万到20万）。因为有字节级兜底，再冷门的字符也能编码，不会出现`<UNK>`。

课程给的几个中英和代码对照很直观：`"Hello, world!"`13个字符只占4个token（英文大概1词≈1 token）；`"你好，世界！"`6个字符占5～7个token（中文1字≈1～1.5 token）；代码里的缩进和标点都算token，`"def foo(): return 1"`19个字符要占7～9个token。Token数既决定计费（输出单价通常高于输入），也决定上下文窗口上限，超了要么直接报`context_length_exceeded`，要么由上层产品截掉最早的对话。

流式场景下要统计消耗，像`token_observer.py`那样不能直接用藏掉流式细节的封装，得自己遍历chunk，从通常挂在最后一个chunk上的`chunk.usage`里取`prompt_tokens`、`completion_tokens`和`total_tokens`（网关不返回时兜底取0不抛异常）。

拿到token序列后，Transformer靠注意力回看前文，算出词表里每个词作为下一个词的概率分布，按概率采样出一个token再拼回去循环生成——这就是next-token prediction。三个最常用的API参数都能直接对上这一步：
* **`stream=True`**：模型本来就是一个token接一个token吐的，开流式只是不等整段算完，边生成边发。
* **`temperature`**：控制采样有多敢挑低概率候选；设为`0`几乎总选概率最高的，输出稳；调高随机性加大，同一问题每次措辞都会变。
* **`max_tokens`**：硬截断最多生成的token数；设成`1`时，因为输入按`user`角色组织成了对话模板，模型会按回复意图吐第一个字，而不是机械续写上文。

---

### 2.4 RAG的检索环节：先把资料变成向量

第3章还有一节RAG（检索增强生成），流程是切块、检索、拼接三步：先把资料切成片段存好，用户提问时先检索出最相关的几段，拼进`messages`再让模型回答，等于给2.2节那个无状态模型外挂了一份可查的资料。`rag_retriever.py`只练中间的检索，而且不调LLM、不用现成的embedding（向量嵌入）接口，纯用numpy把检索核心手写一遍：

* **分词与词表**：`tokenize`把文本拆成「单字＋相邻两字」的组合，对没有空格分词的中文也能用；词表只用文档集来建，问题里出现词表外的token直接忽略。
* **`embed`**：把文本变成词频计数向量，再做L2归一化（把向量长度缩成1）；一个词都没命中时返回全0向量，不去除以0。
* **`top_k`**：向量都归一化后，余弦相似度就是点积，把全部文档向量堆成矩阵、和问题向量乘一次，就同时算出所有文档的得分，排序取前K个；同分时下标小的在前，K比文档数还大时就返回全部。

这套字面检索的边界也很清楚：它只认字面重合，「汽车」和「轿车」意思相近但一个字都对不上，得分就是0；真embedding靠语义训练出的向量才能把近义词拉近。另一个细节是归一化不能省：不归一化时，长文档词频天然更高、得分虚高，等于按篇幅而不是按相关性排名。

## 3. Chatbot和Agent的差别，就在那个循环

只做一次API调用，模型再强也只是一个问一句答一句的Chatbot。它像关在玻璃罩里，四类信息够不到：实时信息、私域知识、训练截止后的新事、以及代码或API的真实执行结果。

### 3.1 差别不在模型大小，在于循环和Harness

`demo_05_chatbot_vs_agent.py`拿同一道题跑了两组对照：让模型抓取`https://example.com`的内容，算出字节数再除以7。
* **朴素Chatbot**：一次请求直接要答案。模型够不到网络，要么老实说抓不到，要么自信编一个字节数和商。
* **最小Agent**：换同一个模型，只在外面套一个`while`循环，配上`fetch_url`和`calculator`两个工具。第1轮模型输出要抓网页，程序执行后把字节数塞回`messages`；第2轮调计算器算除法；第3轮拿到结果输出最终答案。

说到底，Agent不是模型突然变聪明了，而是外层程序在驱动LLM。业界把这层控制程序叫Harness（马具）——模型是马，外层循环、工具调度和步数边界是缰绳，管着模型什么时候想、什么时候动手、什么时候停下来。

### 3.2 Function Calling协议与核心Agent Loop

`demo_05`留了一个很脆的点：它用的是2022年ReAct论文早期的做法，靠System Prompt口头约定「调工具时只输出一行JSON」，再用`parse_step`去文字里抠大括号。模型多说一句客套话、参数里带引号或者输出被截断，解析当场就挂。

2023年6月OpenAI推出Function Calling，把调工具做成了API层的结构化协议。协议就两件东西：
1. **Tool Schema**：用JSON Schema写明工具的`name`、`description`和`parameters`，通过`tools`参数传给模型（`tool_choice`默认`auto`）。模型看不到你的Python函数体，选得对不对、参数传得准不准，全看Schema里的`description`写得清不清楚。
2. **第4种角色`tool`**：模型要调工具时不再混在正文里说，而是填进结构化字段`assistant.tool_calls`；程序真去执行函数后，再以`role="tool"`把结果塞回`messages`。

把这套协议放进循环，就是`demo_06_function_calling_protocol.py`和`tool_calling_agent_loop.py`的核心骨架：

```python
def run_agent(user_message: str, max_steps: int = 5) -> str:
    messages = [
        {"role": "system", "content": "你是一个会使用工具的助手。需要实时信息或精确计算时，请调用相应工具。"},
        {"role": "user", "content": user_message},
    ]

    for _ in range(max_steps):
        msg = chat(messages, tools=TOOLS_SCHEMA)
        messages.append(msg)  # 必须把带tool_calls的assistant消息原样追加进历史

        if not msg.tool_calls:
            return msg.content or ""

        for tc in msg.tool_calls:
            func = TOOL_REGISTRY[tc.function.name]
            args = json.loads(tc.function.arguments or "{}")
            result = func(**args)
            messages.append({
                "role": "tool",
                "tool_call_id": tc.id,
                "content": str(result),
            })

    return "超过最大步数"
```

这段20行不到的循环里，`TOOL_REGISTRY`负责把字符串工具名映射到真正的Python函数，`max_steps`防模型陷入反复调工具的死循环烧额度；在`calculate`这类求值工具里，还得卡死允许字符集并把`eval`的`__builtins__`置空，防任意代码执行。

写这套协议时有三个地方很容易漏：
* 模型决定调工具时，`assistant.content`经常是`None`，真正的调用请求在`tool_calls`里，看到`None`不是报错。
* 带`tool_calls`的那条`assistant`消息必须原样追回`messages`，不能只塞`tool`结果，否则下一轮模型看不到自己刚发起过调用，会重头再来。
* `tool`消息里的`tool_call_id`必须和上一条`assistant.tool_calls[*].id`严格对齐；而且`tool_calls`是列表，模型完全可能在一轮里并发查多个城市天气，一开始就该按列表遍历。

---

### 3.3 工具安全护栏：不把安全押在模型听不听话上

3.2里`calculate`要卡字符集，根子在第5节《Agent安全》：模型的行动是由它读到的文字决定的，而网页、Issue、PR评论这些文字任何人都能写，提示词注入防不胜防。所以`tool_guard.py`的做法是不指望模型自觉，用纯代码把工具权限收窄到刚好够用，三道关：

* **命令白名单**：`parse_command`先用`shlex.split`拆参数，再逐项拦——命令必须在白名单里（`ls`、`cat`、`echo`、`date`、`wc`、`head`），含`;`、`|`、`$`、反引号等shell元字符的一律拒绝，防`echo hi; env`这类拼接偷渡；连选项都只许「短横线＋单个字母数字」的短选项，防`wc --files0-from=../secret.env`把路径夹带进长选项里。
* **路径沙箱**：`resolve_in_sandbox`把参数路径解析成绝对路径后，必须还在工作目录之内；`..`跳出、绝对路径、指向沙箱外的符号链接，全都拒绝。
* **输出脱敏**：`redact`在结果返回前扫一遍，`cnb_`开头的长串、键名含`token`、`secret`、`password`、`api_key`的键值，值一律换成`***`。

三关串进`run_guarded`才是完整入口：解析、验路径、再用只留`PATH`的干净环境执行（不继承宿主的任何密钥），输出截断到2000字并脱敏后才交回模型。和黑名单比，白名单的思路是反过来的：没被明确允许的，一律不行。

## 4. Vibe Coding、假测试与Harness Engineering

第3章手写的Agent只有查时间和算术两个工具；把工具换成读/写文件、跑命令、搜代码库，再自动喂入项目上下文，就是第4章的Coding Agent（课程拿CNB内置插件形态的CodeBuddy演示，底层逻辑和Cursor、Claude Code一样）。

### 4.1 三种模式、四个能力与规格化约束

在真实代码库里用Coding Agent，不能把所有活全混在一个模式里干：

| 维度 | 具体组件 | 分工与使用边界 |
|---|---|---|
| 三种模式 | `Ask` / `Craft` / `Plan` | `Ask`只答不改，用来读代码和问思路；`Craft`直接动手，适合局部小改和修Bug；`Plan`先澄清需求出方案清单，人审后再改，适合跨文件改造 |
| 斜杠指令 | `/init`、`/explain`、`/fix`、`/tests`、`/cr`、`/rules`、`/summarize` | 快捷触发常用动作，每条消息限用1个，支持自定义 |
| Rules | `.codebuddy/rules/`（或根`CODEBUDDY.md`） | 持久化规范，自动拼在上下文开头；分`always`/`requested`/`manual`，只在新建会话时加载，改完要新开对话才生效 |
| Skills | `.codebuddy/skills/`（`SKILL.md`+脚本/参考） | 打包领域流程和脚本，三层渐进式披露：元数据常驻（约100词）→触发后读正文→需要时再读脚本和参考资料 |
| Subagent | `.codebuddy/agents/` | 独立上下文的专职助手，分`agentic`自动委派与`manual`手动切换；强调单一职责和工具最小化，不污染主会话 |

串成一轮完整研发，顺序是**读→做→修→审**：接手陌生代码先切`Ask`配`@文件`搞清主链路；做功能先走`Plan`让它反问澄清，确认清单再写；修Bug时贴报错堆栈、日志和最小复现，修完必补回归测试；合并前跑`/cr`和`/tests`，保持小步提交。

配套作业`agent-exercises/01-guess-game`专门练「动手前怎么把模糊需求写成合约」。收到一句「做个命令行猜数字游戏，好玩一点」，直接丢给AI它只能靠猜。写`spec.md`要过四个角度：用户故事（谁在什么场景做什么）、**用数字代替形容词的可验证验收标准**（把「机会合理」写成「最多10次、随机数1-100含边界」）、技术与范围边界（仅Python标准库单文件，不做GUI和多人）、以及自动化自测入口（`--self-test`逐条校验、全过退出码0）。

### 4.2 假测试反例：测试全绿不等于测试有效

用Coding Agent写代码时，比语法报错更坑的是AI顺手写出一个「永远全绿的假测试」。`04-vibecoding/demo/fake-test/demo_test_quality.js`用不到百行Node.js把这个反例演得很透：

```javascript
// 无效测试：只在测试里自己重算一遍，压根没调用被测计分器
function checkCopiedCalculation() {
  let score = 0;
  score += 10;
  assert.equal(score, 10);
  score += 10;
  assert.equal(score, 20);
  score = 0;
  assert.equal(score, 0);
}

// 有效测试：调用真实被测工厂，按验收标准断言行为
function checkRealScore(factory) {
  const counter = factory();
  assert.equal(counter.value(), 0, "初始分数应为0");
  counter.eat();
  assert.equal(counter.value(), 10, "吃一个食物应得到10分");
  counter.reset();
  assert.equal(counter.value(), 0, "重新开始应归零");
}

// 在内存中把score += 10;变异为score += 20;构造错误实现
function createWrongVersion() {
  const source = readFileSync(join(__dirname, "score-example.js"), "utf8");
  const sandbox = { module: { exports: {} } };
  runInNewContext(source.replace("score += 10;", "score += 20;"), sandbox);
  return sandbox.module.exports.createScore;
}
```

脚本用`node:vm`在内存沙箱里把`score-example.js`的加分逻辑改成每次加20分，再把两组测试分别跑在正确和错误版本上：

| 测试写法 | 正确实现（吃1个加10分） | 错误实现（吃1个加20分） |
|---|---|---|
| `checkCopiedCalculation`（自己重算答案） | 通过 | 仍然通过 |
| `checkRealScore`（调用真实计分逻辑） | 通过 | 失败 |

这个反例挺醒脑的：AI回一句「测试已补齐并全部通过」不等于真过了，得核对测试到底调没调真实实现、断言是不是按需求清单写的，而不是为了凑全绿把断言改松或者自己重算一遍。

### 4.3 从Vibe Coding到Harness Engineering

4.2进阶选讲里画了一条四层演进：Vibe Coding（交互探索）→SDD规格驱动（先对齐范围和验收标准）→工程化协作（经验沉淀进文档、Rules、Skills和回归测试）→Harness Engineering（把约束升级成系统化交付环境、自动反馈和硬门禁）。

里面有一点我深有同感：AI漏做一次，就往Rules里多加一句「以后必须……」，规则越堆越厚，模型反而连主任务都干不好。Rules只是软约束，必经且失败就会阻断合并的自动化校验才是硬约束。人没必要一直盯着终端当消息搬运工，把获取事实（读代码、跑测试、搜日志）交给机器，人退到决策点做取舍（定范围、判风险、定验收标准）。

---

## 5. 住在仓库里的NPC与综合交付

一个人同时推好几个任务时，IDE本地对话的瓶颈就出来了：结论全散在个人聊天记录里，执行占着本地分支，调好的角色也没法共享。第4.3节的NPC，本质上是把上篇的Git/PR、Docker容器和这篇的Agent Loop拧在了一起。

### 5.1 Subagent推指令，NPC拉上下文

Subagent和NPC都讲单一职责，但上下文来源完全不同。Subagent跑在单次IDE会话里，主Agent把指令和局部材料推给它，干完回传结论，会话一关就消失；NPC住在仓库配置里，靠Issue或PR里的`@`事件唤醒，平台只告诉它「你在哪个仓库、哪个Issue、谁找你」，历史讨论、PR diff和源码都由它自己调工具去留言板上拉，产物也直接留在Issue和PR里。

4.3节里那个「蓝色香蕉」暗号实验把NPC的无状态机制拆得很干净：先在Issue正文里让`@CodeBuddy`记住暗号「蓝色香蕉」，等它回复后，再新发一条评论问「刚才让你记住的暗号是什么」。它答对了，但点开第二次流水线的请求体看原始`messages`，`system`里只有平台准则和Skills简介，`user`里只有这次追问，压根没有「蓝色香蕉」四个字。

翻后面的调用轨迹才看到真相：它先按渐进式披露加载了`cnb-api` Skill，再调`bash`执行`cnb issues get`和`cnb issues list-comments`查了整个Issue的正文和评论，找到暗号后再调`comment`回复。这个实验直接推出两条实用规矩：
1. **每次`@`都是全新会话**：确认好的需求清单、决策和约束必须写在Issue或PR里，只在口头或本地私聊里说过的，NPC永远看不到。
2. **规则文件路径不同**：NPC不读IDE的`.codebuddy/rules/`，想让仓库里的NPC遵守项目约定，要写进仓库根目录的`AGENTS.md`。

### 5.2 为什么NPC能动手：容器隔离+临时令牌

NPC不只是回评论，它能自己建分支、改代码、提PR，靠的正是上篇那两块底座：
* **Docker隔离执行**：每次触发都拉起独立容器（默认镜像`cnbcool/default-npc:latest`，内置git、CNB CLI和基础Skills），跑完即毁。想让NPC用别的语言工具链，自己写Dockerfile建镜像并在`.cnb.yml`里指定就行。
* **Git自动检出+临时委托令牌**：容器启动时已自动checkout好代码（Issue切默认分支，PR切目标分支），同时平台按「谁`@`了它」自动签发临时`CNB_TOKEN`注入容器。这套授权很克制：**限时**（仅本次任务有效）、**限地**（默认只限当前仓库）、**限权**（上限Developer，且默认只读回评论，只有在评论编辑器勾选「替我上班」并具备开发者权限时，才允许建分支提PR）。

### 5.3 三角色闭环与综合交付预演

在仓库`.cnb/settings.yml`里把岗位说明书写进`npc.roles`，就能搭一支仓库里的AI团队（`02-design-your-team`就是自己写这套prompt并拿真实Issue迭代）：
* **产品经理**：收到模糊需求先追问澄清，再输出用数字代替形容词的编号清单。
* **开发**：严格按清单实现，不做范围外改动，提交前自测，收到评审意见逐条改并说明。
* **审查员**：逐条对照清单验收，标出Bug、漏掉的边界测试和超范围改动，每条给文件位置，全过才输出`LGTM`。

走一次完整Issue闭环就是：`提模糊Issue并@产品经理`→`人审数字清单`→`评论@开发并勾选替我上班`→`开发自动建分支、写代码、提PR`→`@审查员出意见并打回返工`→`复审LGTM`→`人按合并键`。也可以在`.cnb.yml`里配内置任务`npc:go`，让每次提PR自动跑审查员（非`@`触发时要手写`userPrompt`）。整个过程里三个角色从不私下传话，全靠读写Issue/PR留言板接力；人只守两关：**审清单、按合并**。

课程最后拿`app-exercises/01-llm-web-app`（AI夸夸机）和第5章迷你黑客松把前面全串成一次交付。`01-llm-web-app`把Flask页面、流式LLM调用和Dockerfile分层打包合在一起：本地云原生开发默认跑`8080`（避开WebIDE占用的`8686`），容器内按黑客松规范统一`ENV PORT=8686`、`EXPOSE 8686`；跑容器时用`-e CNB_API_ENDPOINT -e CNB_REPO_SLUG -e CNB_TOKEN`把宿主三件套透传进去实现零密钥调模型。提PR后CI跑三段阻断检查：`docker build`成功→容器起完`GET /`返回200→`POST /api/compliment`真实调LLM返回非空200。到了黑客松模板`ai-engineering-hackathon-template`，不管选游戏、Agent还是小工具，交付标准都收束到同一套：Git分支协作+Dockerfile打包+监听`0.0.0.0:8686`+PR自动检查+一键预览。

---

## 6. 踩坑、边界与个人收束

后三章跑完，有几个挺容易栽跟头的细节记下来：
* **CNB网关别忘`stream=True`**：发非流式直接报`HTTP 400 / code 11101`，不管是写Agent Loop还是包Flask接口，客户端层都得统一走流式收集。
* **NPC触发的格式限制**：`@NPC`必须写在评论普通段落，塞进列表、引用块、代码块里全不触发；修改已经发出的评论不会重新触发，要重发一条；单Issue或PR评论超100条后不再触发，单条评论最多同时触发10个NPC。
* **容器隔离消不掉Git冲突**：多个NPC并行改仓库，容器互相隔离，但合回主干时改了同一段代码照样撞冲突，拆任务时得把文件边界划清。
* **别搞无上限的自动接力**：如果让审查员自动`@开发`、开发改完再自动`@审查员`，没收敛上限和人工拍板就会死循环烧token。探索性原型和一次性脚本自己直接上手更快，需求清晰的重复交付再交给NPC团队。

说到底，跑完这五章让我对AI编程的体感清醒了不少。大模型写代码的速度确实越来越快，但真要做成能复现、能交付的东西，靠的从来不是把prompt写得多花哨，而是前面那套挺朴素的工程纪律——用数字把验收标准定死，用调真实实现的测试防假绿，用Git、Docker和CI把软约定变成硬门禁。之前做C++矩阵乘法对照实验和1×4脉动阵列上板，我守的是「只信可复现证据，不靠直觉硬猜」；切到AI工程和Agent开发，底层其实还是同一件事。继续把学到的写下来，把做过的摆出来。

---

## 7. 参考来源

* **课程主仓库（第3–5章讲义、Demo与文档）**：`ai-engineering-course`（`03-ai-intro/lecture/01-prompts-and-roles.md`、`02-tool-and-agent-loop.md`、`03-ai-intro/lab/README.md`、`demo_05_chatbot_vs_agent.py`、`demo_06_function_calling_protocol.py`；`04-vibecoding/lecture/01-coding-agent.md`、`02-dev-lifecycle.md`、`03-npc.md`、`demo/fake-test/demo_test_quality.js`与`score-example.js`；`05-hackathon/README.md`、`docs/CNB-LLM接口.md`）。
* **配套作业仓库**：`ai-engineering-exercise`（`ai-exercises/`6个脚本、`agent-exercises/01-guess-game/README.md`、`02-design-your-team/README.md`、`app-exercises/01-llm-web-app/`下的`README.md`、`app.py`、`llm_client.py`与`Dockerfile`）。