---
title: 基于MCU的Agent（二）
date: 2026-06-15 13:20:00
updated: 2026-06-15 13:20:00
categories:
  - 嵌入式
tags:
  - MCU
  - AI
  - 物联网
  - ESP32
  - Agent
description: 从单板 Agent 到四个 ESP32-S3 节点的 Agent Mesh。
cover: /images/covers/anime-cover-01.jpg
toc: true
---

# 基于 MCU 的 Agent（二）

上一篇主要讲的是 ESPAgent 如何把 LLM、工具调用和硬件驱动接起来。那时的重点还是单块 ESP32-S3：用户从飞书发来消息，固件把消息送进 Agent loop，LLM 判断是否需要调用工具，最后由工具层去控制 GPIO、RGB 灯、舵机或者读取传感器。

最近项目往前走了一步。ESPAgent 现在不只是一个“会调用硬件工具的聊天机器人”，而是在往一个小型 Agent Mesh 演进。当前已经有四个 ESP32-S3 角色：Coordinator、Sensor、Control、Display。它们仍然使用同一套代码，但通过不同的 node profile 启动不同的服务。与此同时，项目里也加入了 `spawn_subagent`，让主 Agent 可以临时创建一个受限的子 Agent，去完成独立的搜索、查时间、查天气、读文件和总结任务。

这篇文章就按当前最新进展继续讲：ReAct 链路如何在板端验证，subagent 是怎么实现的，四个 ESP32-S3 如何分工，MQTT Mesh 目前做到哪一步，以及硬件资源到底有没有被吃满。

## 第一章：从单板 Agent 到四个角色

ESPAgent 现在的入口已经从早期集中式启动方式调整成了更清晰的分层启动结构。最外层入口是 `main/espagent.c`，真正的启动编排在 `main/app/espagent_app.c`。这样做的目的很简单：`app_main()` 不再堆满所有初始化细节，而是把启动阶段拆开。

当前启动过程大致可以看成几段：

```text
app_main()
  -> espagent_app_init_core()
  -> espagent_app_init_services()
  -> espagent_app_start_local_services()
  -> wifi_manager_start()
  -> espagent_app_start_network_services()
```

核心初始化里会准备 NVS、SPIFFS、message bus、memory、cache、skills、session、Wi-Fi、HTTP proxy、LLM、工具注册表和四个 role service。代码里四个角色的初始化入口是：

```c
coordinator_node_init();
sensor_node_init();
control_node_init();
display_node_init();
```

这四个函数本身现在还比较轻，主要负责打印当前角色边界。但真正有意义的是 role gating。判断当前板子应该跑哪些服务的代码在 `main/roles/role_config.c`。例如：

```c
bool espagent_role_runs_llm(void)
{
    return espagent_role_is_edge() ||
           espagent_role_is_coordinator() ||
           espagent_node_has_capability("llm");
}

bool espagent_role_runs_sensor_sampling(void)
{
    return espagent_role_is_edge() ||
           espagent_role_is_sensor() ||
           espagent_node_has_capability("telemetry");
}
```

也就是说，四块板并不是四份完全不同的工程，而是同一个固件根据 `ESPAGENT_SECRET_NODE_ROLE` 和 `ESPAGENT_SECRET_NODE_CAPABILITIES` 决定启动哪些服务。

当前四个角色是：

```text
/dev/ttyUSB0  esp32s3-coordinator-01  coordinator_agent
/dev/ttyUSB1  esp32s3-sensor-01       sensor_agent
/dev/ttyUSB2  esp32s3-control-01      control_agent
/dev/ttyUSB3  esp32s3-display-01      display_agent
```

Coordinator 是飞书和 LLM 的入口。Sensor 负责采集环境数据。Control 负责执行器，比如 RGB、GPIO、舵机、继电器、风扇和水泵。Display 负责展示状态、timeline 和告警，未来可以迁移到 ESP32-P4 或 Android App。

当前推荐仍然是“同仓库、同固件、不同 profile”。这样做的好处是协议、工具、消息结构和调试方式统一。缺点也很明显：Flash 还没有按角色裁剪，四块板的 app 镜像基本一样。最新一次构建的 `ESPAgent.bin` 大小是：

```text
0x14eb70
```

最小 app 分区是 2MB，还剩：

```text
0xb1490
```

也就是大约 35% 空间。这个状态不是极限压榨，而是保留了后续扩展余量。

## 第二章：ReAct 在板端的真实验证

ESPAgent 的主 Agent 仍然是一个 ReAct loop。代码在 `main/agent/agent_loop.c`。它不是简单调用一次 LLM 就结束，而是循环处理 `tool_use`：

```c
while (iteration < ESPAGENT_AGENT_MAX_TOOL_ITER) {
    err = llm_chat_tools(system_prompt, messages, tools_json, &resp);

    if (!resp.tool_use) {
        final_text = strdup(resp.text);
        break;
    }

    cJSON_AddItemToObject(asst_msg, "content",
                          build_assistant_content(&resp));

    cJSON *tool_results =
        build_tool_results(&resp, &msg, tool_output,
                           TOOL_OUTPUT_SIZE,
                           tool_fallback,
                           TOOL_OUTPUT_SIZE + 512);

    cJSON_AddItemToObject(result_msg, "content", tool_results);
    cJSON_AddItemToArray(messages, result_msg);

    iteration++;
}
```

这段逻辑说明了 ReAct 的基本形态：

```text
用户消息
  -> LLM
  -> tool_use
  -> 执行工具
  -> tool_result 回填给 LLM
  -> LLM 生成最终回复
```

这一次不是只在代码层面确认，而是在 USB0 的真实 ESP32-S3 上验证过。验证方式是通过串口 CLI 注入一条系统消息：

```text
inject_msg system react_test 请调用get_current_time并回复当前时间
```

串口日志里能看到完整链路：

```text
Processing message from system:react_test
Tool use iteration 1: 1 calls
LLM tool[0]: get_current_time({})
Executing tool: get_current_time
Tool[get_current_time] => 2026-06-15 12:48:18 CST (Monday)
LLM final response: 当前时间：**2026年6月15日（星期一）12:48 CST**
```

这个验证很关键。它证明的不只是 `get_current_time` 这个工具能跑，而是证明主 Agent 的 ReAct 链路可以在真实板端完成一次 LLM 工具调用往返。也就是说，消息进入 Agent loop、LLM 返回 tool call、工具执行、结果回填、LLM 再回答，这条链路是通的。

## 第三章：subagent 是怎么实现的

`spawn_subagent` 是这次新增的一个重要能力。它不是第二块 ESP32，也不是云端进程，而是在同一块 ESP32-S3 上创建一个临时 FreeRTOS task。

它的注册在 `main/tools/tool_registry.c`：

```c
register_tool(&(espagent_tool_t){
    .name = "spawn_subagent",
    .description = "Delegate an independent subtask to a temporary ESPAgent subagent...",
    .input_schema_json =
        "{\"type\":\"object\","
        "\"properties\":{\"task\":{\"type\":\"string\"},"
        "\"context\":{\"type\":\"string\"}},"
        "\"required\":[\"task\"]}",
    .execute = tool_subagent_execute,
});
```

执行入口在 `main/tools/tool_subagent.c`。当主 Agent 或串口 CLI 调用 `spawn_subagent` 时，固件会解析输入 JSON，取出 `task` 和可选的 `context`，然后创建一个新的 FreeRTOS task：

```c
BaseType_t ok = xTaskCreatePinnedToCore(subagent_task,
                                        "subagent",
                                        ESPAGENT_SUBAGENT_STACK,
                                        ctx,
                                        ESPAGENT_SUBAGENT_PRIO,
                                        NULL,
                                        ESPAGENT_SUBAGENT_CORE);
```

主调用方会等待这个子任务完成：

```c
BaseType_t done = xSemaphoreTake(ctx->done_sem,
                                 pdMS_TO_TICKS(ESPAGENT_SUBAGENT_TIMEOUT_MS));
```

所以当前 subagent 是同步等待模式。它不是后台长期任务，也不会返回一个 task id。主 Agent 调用它后，会等它完成或者超时。

子任务内部也有一个短 ReAct loop：

```c
for (int iter = 0; iter < ESPAGENT_SUBAGENT_MAX_TOOL_ITER; iter++) {
    esp_err_t err = llm_chat_tools(system_prompt, messages,
                                   s_subagent_tools_json, &resp);

    if (!resp.tool_use) {
        ctx->result = strdup(resp.text);
        break;
    }

    tool_registry_execute(call->name, call->input, tool_output,
                          ESPAGENT_SUBAGENT_TOOL_BUF_SIZE);
}
```

不过它和主 Agent 有一个重要区别：子 Agent 不能看到所有工具。`tool_subagent.c` 里有一个白名单：

```c
static bool subagent_tool_allowed(const char *name)
{
    return strcmp(name, "web_search") == 0 ||
           strcmp(name, "get_weather") == 0 ||
           strcmp(name, "get_current_time") == 0 ||
           strcmp(name, "read_file") == 0 ||
           strcmp(name, "write_file") == 0 ||
           strcmp(name, "edit_file") == 0 ||
           strcmp(name, "list_dir") == 0;
}
```

这意味着 subagent 适合做独立的信息任务，例如：

```text
查当前时间
查天气
联网搜索
读取 SPIFFS 文件
总结某个 skill 或 memory 文件
编辑简单文本文件
```

它不能做：

```text
GPIO 控制
WS2812 控制
舵机控制
直接读取硬件传感器
发送 mesh_send_command
递归创建新的 subagent
```

这里有两层安全限制。第一层是构造给子 Agent 的 tools JSON 时只放白名单工具。第二层是在真正执行工具前再检查一次 `subagent_tool_allowed()`。如果模型幻觉出了 `gpio_write` 或 `mesh_send_command`，执行侧也会拒绝。

USB0 上已经验证过 subagent。串口命令是：

```text
tool_exec spawn_subagent {"task":"Call_get_current_time_and_return_one_sentence"}
```

日志显示：

```text
Executing tool: spawn_subagent
Spawning subagent
Subagent started
Subagent tool iteration 1
Executing tool: get_current_time
Subagent tool get_current_time result: 32 bytes
Subagent done
Subagent completed
tool_exec status: ESP_OK
```

最后输出：

```text
Current time is **Monday, June 15, 2026, 12:47 PM CST**.
```

这说明 subagent 不只是编译通过，而是在真实 ESP32-S3 上完成了“子 Agent 调 LLM、选择工具、执行工具、再让 LLM 总结”的完整闭环。

## 第四章：MQTT Mesh 与四块 ESP32 的通信

四块 ESP32-S3 之间目前主要通过 MQTT Mesh 组织起来。topic 规划在 `main/espagent_config.h`：

```c
#define ESPAGENT_SENSOR_MQTT_TOPIC_STATE \
    ESPAGENT_MESH_TOPIC_PREFIX "/nodes/" ESPAGENT_NODE_ID "/state"
#define ESPAGENT_SENSOR_MQTT_TOPIC_EVENTS \
    ESPAGENT_MESH_TOPIC_PREFIX "/nodes/" ESPAGENT_NODE_ID "/events"
#define ESPAGENT_SENSOR_MQTT_TOPIC_COMMAND \
    ESPAGENT_MESH_TOPIC_PREFIX "/nodes/" ESPAGENT_NODE_ID "/command"
#define ESPAGENT_MESH_TOPIC_ROLE_COMMAND \
    ESPAGENT_MESH_TOPIC_PREFIX "/roles/" ESPAGENT_NODE_ROLE "/command"
#define ESPAGENT_MESH_TOPIC_TIMELINE \
    ESPAGENT_MESH_TOPIC_PREFIX "/agent/timeline"
```

当前联调使用的 topic prefix 是：

```text
espagent/cube1345
```

所以四个节点会发布类似这样的状态：

```text
espagent/cube1345/nodes/esp32s3-coordinator-01/state
espagent/cube1345/nodes/esp32s3-sensor-01/state
espagent/cube1345/nodes/esp32s3-control-01/state
espagent/cube1345/nodes/esp32s3-display-01/state
```

每个节点还会订阅自己的 node command topic 和 role command topic。例如 Control 节点会关注：

```text
espagent/cube1345/nodes/esp32s3-control-01/command
espagent/cube1345/roles/control_agent/command
```

Coordinator 给其它节点发命令时，走的是 `mesh_send_command` 工具。它的实现入口在 `main/tools/tool_mesh_command.c`。如果用户只是说“读取温湿度”，Coordinator 不需要用户提供 MQTT ID。代码会根据 action 自动补目标角色：

```c
if ((!target_node || !target_node[0]) &&
    (!target_role || !target_role[0]) &&
    strcmp(action, "read_temperature_humidity") == 0) {
    target_role = "sensor_agent";
}
```

控制类 action 也会自动路由到 `control_agent`：

```c
else if ((!target_node || !target_node[0]) &&
         (!target_role || !target_role[0]) &&
         is_control_action(action)) {
    target_role = "control_agent";
}
```

这解决了一个用户体验问题：用户不应该在飞书里说“请调用 mesh_send_command，让 esp32s3-sensor-01 执行 read_temperature_humidity”。正常情况下，用户只需要说“读取温湿度”。Coordinator 应该自动判断交给 Sensor 节点。

Sensor 节点目前已经有一个窄的真实执行路径。代码在 `main/sensors/sensor_mqtt.c`：

```c
static bool handle_sensor_mesh_command(const espagent_mesh_command_t *cmd)
{
    if (!espagent_role_is_sensor()) {
        return false;
    }
    if (strcmp(cmd->action, "read_temperature_humidity") != 0) {
        return false;
    }

    esp_err_t err = tool_aht10_read_temperature_humidity_execute(
        cmd->args_json[0] ? cmd->args_json : "{}",
        result,
        sizeof(result));

    publish_mesh_command_result(cmd, err, result);
    return true;
}
```

也就是说，Sensor 收到 `read_temperature_humidity` 后，可以直接调用 AHT10/AHT20 工具，并发布 `mesh_command_result`。

Control 节点也已经有联调用的直接执行路径。它可以处理：

```text
set_status_light
ws2812_set
servo_write
gpio_write
```

但是这里要强调一个边界：这仍然是联调路径，不是最终的生产安全链路。真正开放远程执行前，还要补 command queue、鉴权、审计、safety interlock、actuator state 和 result correlation。

当前正确的远程控制链路应该演进成：

```text
MQTT command
  -> mesh_protocol validate
  -> command_queue
  -> safety_interlock
  -> actuator_state
  -> driver/tool
  -> result event + timeline event
```

现在还没有完全做到这一点。所以可以说：MQTT Mesh 的状态发布、命令发布、角色路由和部分执行路径已经打通，但还不是完整的多 Agent 调度闭环。

## 第五章：四个角色的资源占用

现在四块 ESP32-S3 并没有真正把硬件资源吃满。更准确的说法是：运行时已经按角色裁剪服务，但固件体积还没有按角色裁剪，资源也保留了较大余量。

Coordinator 是最重的角色。它启动 LLM、Feishu WebSocket、本地 WebSocket server、MQTT、SNTP、cron、heartbeat、proactive、session/context 和工具系统。它还可以临时启动 subagent。USB0 启动日志里可以看到 PSRAM 大约 8MB 可用，完成一次 ReAct 验证后仍然有约 8.25MB PSRAM 可用。

Sensor 角色会跳过 LLM 和 Feishu。它主要运行 sensor sampling、environment monitor、presence monitor、MQTT telemetry 和串口 CLI。当前还没有 sensor cache、滤波统计、阈值事件队列，所以资源占用并不高。

Control 角色也不运行 LLM 和 Feishu。它主要承担 MQTT command receiver、本地执行器工具、控制边界和 boot servo demo。后续真正让它吃资源的部分应该是 command queue、safety interlock、actuator registry、actuator state，以及更多外设，例如继电器、风扇、水泵、加湿器、蜂鸣器或音频提示。

Display 角色目前最空。现在它有 display/state/watchdog 的边界和 MQTT 订阅，但还没有真实屏幕 UI、timeline store、状态聚合和 watchdog 规则。未来如果接 ESP32-P4 或 Android App，这一层才会真正变重。

当前状态可以总结成：

```text
Coordinator: 最重，LLM 和通信入口，已经接近真实使用形态
Sensor: 中等，采样任务已启动，但还缺缓存、滤波、阈值事件
Control: 中等偏低，工具存在，但安全执行闭环还没补完
Display: 最轻，角色边界有了，展示能力还没真正展开
```

所以“把每个 ESP32 都尽量吃满”还没有完成。项目现在保留余量是合理的。因为在 MCU 上，过早把 RAM、任务栈、TLS buffer 和外设都压满，会让调试变得非常困难。更稳妥的方式是先让通信链路和安全边界跑通，再逐步把每个角色的资源吃起来。

下一步更实际的资源扩展方向是：

```text
Sensor:
  sensor_cache
  采样滤波
  telemetry schema
  humidity_low / air_quality_bad / presence_changed 事件

Control:
  command_queue
  safety_interlock
  actuator_state
  relay/fan/pump/humidifier registry
  执行结果审计

Display:
  timeline_store
  node state 聚合
  telemetry 聚合
  alerts/watchdog 规则
  ESP32-P4 或 Android 可视化界面

Coordinator:
  command_id result correlation
  远端执行结果汇总回复飞书
  tool_use/tool_result timeline
  role-based tool exposure
```

## 第六章：当前项目的边界

现在 ESPAgent 已经比第一篇时更接近一个多节点 Agent 系统，但边界仍然要说清楚。

已经跑通的部分：

```text
主 ReAct loop 板端验证通过
spawn_subagent 板端验证通过
USB0 Coordinator 已接入 Feishu / LLM / MQTT
四角色 profile 已经存在
MQTT state / events / command topic 已经存在
Coordinator 可以按自然语言选择 sensor_agent / control_agent
Sensor 有 read_temperature_humidity 的白名单执行路径
Control 有若干执行器 action 的联调执行路径
```

尚未完成的部分：

```text
Coordinator 还不会等待远端 mesh_command_result 并汇总回复用户
Control 还没有完整 command queue / safety interlock / actuator_state
Display 还没有真实 UI 和 timeline store
工具列表还没有按角色裁剪
完整 tool_use / tool_result timeline 还没落地
```

这也是 MCU 上做 Agent 比在云端做 Agent 更麻烦的地方。云端 Agent 可以比较轻松地开线程、开进程、访问数据库和消息队列。MCU 上每一个任务栈、每一个 TLS buffer、每一次 JSON parse 都要算内存。让 LLM 控制硬件也不能只靠 prompt，必须有工具 schema、执行侧校验、GPIO allowlist、role boundary 和安全互锁。

ESPAgent 现在的形态可以概括为：

```text
Feishu / WebSocket / Cron / Serial
        -> message_bus
        -> agent_loop ReAct
        -> tool_registry
        -> 本地工具或 MQTT Mesh command
        -> Sensor / Control / Display 节点
        -> events / timeline / Feishu 回复
```

相比第一篇，项目已经从“单板 LLM 调硬件”推进到了“四板角色分工 + 受限 subagent + MQTT Mesh 雏形”。它还不是完整的分布式多 Agent 系统，但核心方向已经清楚：Coordinator 负责理解和调度，Sensor 负责感知，Control 负责安全执行，Display 负责展示和监督。

接下来最关键的不是继续堆更多工具，而是把远程结果闭环补上。也就是 Coordinator 发出 command 后，要能等待 `mesh_command_result`，根据 `command_id` 关联结果，再把真实执行结果告诉用户。只有做到这一步，用户在飞书里说“读取温湿度”时，才不是只得到“我已经发送指令”，而是真的得到来自 Sensor 节点的温度和湿度。

做到这里，ESPAgent 才会从“能发命令的 Agent”变成“能确认执行结果的 Agent Mesh”。
