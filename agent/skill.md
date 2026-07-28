# **1. Agent Skill 是什么？**
Agent Skill，中文译为 Agent 技能，其本质是一份可供Agent随时查阅与调用的规范化说明文档。一个最简单的Skill结构如下所示：
![skill是什么.png](reference/skill/skill%E6%98%AF%E4%BB%80%E4%B9%88.png)

# **2. 创建 & 配置 & 使用会议总结 skill**
1. 创建 SKILL 文件夹：meeting-minutes-generator
2. 创建和编写 SKILL.md

[meeting-minutes-generator.md](reference/skill/meeting-minutes-generator.md)

## 2.2 给 Codex 配置 Skill
![codex配置skill.png](reference/skill/codex%E9%85%8D%E7%BD%AEskill.png)

## 2.3 使用 Skill
```
王强：我看了一下数据库，目前主要问题是部分查询没有走索引，可以先优化数据库索引。
李四：除了数据库问题，我建议增加缓存机制，把一些热点订单数据放到 Redis 里面。
张三：可以，数据库优化和缓存方案一起推进。
王强：数据库索引优化我负责，本周五之前完成。
李四：Redis 缓存方案我负责，下周一提交技术方案。
张三：好的，下一版本目标是把订单查询接口平均响应时间降低到 1 秒以内。
王强：测试数据需要重新准备一下，方便验证优化效果。
张三：测试数据部分由测试团队负责，具体负责人后续确认。
```
![使用skill.png](reference/skill/%E4%BD%BF%E7%94%A8skill.png)

# **3. Skill 流程**
![skill流程.png](reference/skill/skill%E6%B5%81%E7%A8%8B.png)

# **4. References**
按需加载！
[财务报销.md](reference/skill/%E8%B4%A2%E5%8A%A1%E6%8A%A5%E9%94%80.md)
如果会议中提到报销的内容，有时很需要判断报销流程是否合规，那么怎么判断呢？

最简单的方式就是筛入 SKILL.md，但是这样也有问题。

## 4.1 SKILL.md 修改
按需加载 References，SKILL.md 中增加财务报销的引用：
```
## 六、财务报销
若会议内容涉及资金、预算、采购或费用事项，应参考 `./references/财务报销.md`，判断金额是否符合标准。 
如果没有涉及，就不需要展示这个部分。
```
![财务报销.png](reference/skill/%E8%B4%A2%E5%8A%A1%E6%8A%A5%E9%94%80.png)
```
总结以下会议内容：
张三：老板，我昨晚招待客户，住宿花了一万多，具体做了什么，不方便说。
老板：好的，那你去找财务报销吧。
```

# 5. scripts 的使用
1. 保存会议纪要要桌面：save_to_desktop.py
```python
import sys
from datetime import datetime
from pathlib import Path


def get_desktop_path() -> Path:
    """获取当前用户的桌面目录。"""
    home = Path.home()

    for desktop_name in ("Desktop", "桌面"):
        desktop = home / desktop_name
        if desktop.exists():
            return desktop

    desktop = home / "Desktop"
    desktop.mkdir(parents=True, exist_ok=True)
    return desktop


def save_meeting_minutes(content: str, filename: str | None = None) -> Path:
    """将会议纪要保存到桌面。"""
    content = content.strip()

    if not content:
        raise ValueError("会议纪要内容不能为空")

    if filename is None:
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        filename = f"会议纪要_{timestamp}.md"

    if not filename.lower().endswith(".md"):
        filename += ".md"

    file_path = get_desktop_path() / filename
    file_path.write_text(content, encoding="utf-8")

    return file_path


def main() -> None:
    """
    参数格式：

    python upload.py "会议纪要内容"
    python upload.py "会议纪要内容" "文件名.md"
    """
    if len(sys.argv) < 2:
        print(
            '使用方式：python upload.py "会议纪要内容" ["文件名.md"]',
            file=sys.stderr,
        )
        sys.exit(1)

    # 第一个参数：Codex 传入的会议纪要内容
    meeting_content = sys.argv[1]

    # 第二个参数：可选文件名
    filename = sys.argv[2] if len(sys.argv) >= 3 else None

    try:
        saved_path = save_meeting_minutes(
            content=meeting_content,
            filename=filename,
        )
        print(f"保存成功：{saved_path}")

    except Exception as error:
        print(f"保存失败：{error}", file=sys.stderr)
        sys.exit(1)


if __name__ == "__main__":
    main()
```
2. 添加 SKILL.md 文件内容
```
        ## 脚本规则
        如果用户需要将总结内容"保存"、"上传"等，你需要必须运行 `./scripts/save_to_desktop.py` 将总结内容上传服务器。
        脚本使用方法
        ```python
        python ./scripts/save_to_desktop.py "会议总结内容"
        ```
```


