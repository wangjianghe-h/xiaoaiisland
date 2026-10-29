# 背屏课程卡片模板（xiaoai_course_schedule）

配合 [REAREye](https://github.com/killerprojecte/REAREye) 的「组件管理」，把本模块的课程焦点通知渲染成小米妙享背屏卡片。

组件已上架 REARStore（Widget ID `xiaoai_course_schedule`），模板与源码也单独放在
[wangjianghe-h/xiaoai-course-backscreen](https://github.com/wangjianghe-h/xiaoai-course-backscreen)。

## 前置条件

1. 本模块已安装并在 LSPosed 中启用，作用域包含 `com.miui.voiceassist`。
2. REAREye 中开启：更多 → 通知与路由 → **允许焦点通知在背屏显示**。
3. REAREye 中已存在组件 `xiaoai_course_schedule`（商店安装或手动新增）。

## 必须保留一条通知路由

REAREye → 通知与路由 → 通知路由 → 新增：

| 字段 | 值 |
|---|---|
| 目标包名 | `com.miui.voiceassist` |
| 通知规则 | `regex:.*` |
| 组件 | `xiaoai_course_schedule` |

这条路由是 REAREye 对该包的**包级强制放行**来源。删掉后即使「允许焦点通知在背屏显示」是开着的，背屏也会收不到卡片（实测）。`com.android.keyguard` 不需要路由。

## 导入模板

REAREye → 配置 → 背屏中心 → 背屏个性化 → **组件管理** → `xiaoai_course_schedule` → 编辑 →
模板文件路径 → 选择文件 → 选 `xiaoai_course_schedule.zip` → 确定。
然后重启背屏服务让模板重新部署：

```bash
adb shell am force-stop com.xiaomi.subscreencenter
```

核对部署结果（约 19.5 KB 即为本模板）：

```bash
adb shell ls -l /data/system/theme_magic/users/0/subscreencenter/smart_assistant/re_xiaoai_course_schedule
```

## 数据来源

背屏智能助手只把 `miui.rear.param` 原样透传给 MAML 模板（`miui.focus.param` 会被过滤掉），
因此模块把课程字段写进 `rear_param_v1`：

```json
{"rear_param_v1":{
  "business":"xiaoai_course_schedule",
  "title":"高等数学", "content":"08:00-09:40", "hint":"教科A-101",
  "status":"即将开始", "section":"1-2", "teacher":"张三",
  "countdown_label":"距上课", "timer_when":"1793259000000"
}}
```

| 字段 | 含义 | 卡片位置 |
|---|---|---|
| `title` | 课程名（约 9 字折两行） | 标题 |
| `content` | 起止时间 | 第二行（青） |
| `hint` | 教室 | 第三行（薄荷绿） |
| `section` / `teacher` | 节次 / 教师 | 第四行（淡紫） |
| `countdown_label` + `timer_when` | 倒计时前缀与目标时刻（epoch ms 字符串） | 底部橙色 |

右上角为超级小爱图标（占原系统时钟位）。左侧蓝色竖线与标题上沿对齐（v1.1.0）。

## 模板实现要点（踩过的坑）

- **解析必须走 Trigger**：`<ExternalCommands><Trigger action="init">` + `notification_received` 调 `loadData`；直接写在 `<Var expression>` 里的 `strToJson`/`jsonGetString` 链式调用不会被求值。
- **`num()` 只认变量**：`num(jsonGetString(...))` 恒为 0，要先赋给字符串变量再 `num(@var)`。
- **熄屏倒计时**：根标签需 `useVariableUpdater="DateTime.Minute,DateTime.Second"`，`frameRate=1` 足够（每秒 1 帧驱动秒级倒计时，重绘开销约 60 帧方案的 1/60）。
- **多行标题锚点不同**：`multiLine` 文本的 y 与单行不一致，不能依赖 `.text_height`；用字数判断（`len(@c_title) > 9`）+ 固定行高。
- **布局避开摄像头**：背屏左侧约 296px 为遮挡区，内容从 `120*scale` 起排。
- **图标裁边**：按 alpha 包围盒裁切后再设定尺寸，否则视觉上会"显小"。
- **业务 ID 可配置**：模块内 `rear_business_id`（默认 `xiaoai_course_schedule`），换组件时无需改代码。
