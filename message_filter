"""
消息关键词过滤器

这个插件通过用户消息回调机制，根据用户ID和消息中的正则表达式模式对消息进行过滤，
决定是否触发AI响应。支持私聊和群聊的独立配置，每个用户可以配置独立的正则表达式模式。

## 主要功能

- **私聊/群聊独立配置**: 可分别为私聊和群聊设置不同的过滤规则
- **正则表达式匹配**: 支持强大的正则表达式模式匹配，提供更精细的控制
- **个性化模式过滤**: 为每个用户配置独立的正则表达式模式
- **灵活阻止模式**: 可选择是否保存被过滤的消息记录
- **优雅降级**: 异常情况下默认放行消息，确保系统稳定性

## 配置说明

### 1. 生效范围配置
- **启用私聊过滤 (ENABLE_PRIVATE)**: 控制是否在私聊频道启用过滤功能
- **启用群聊过滤 (ENABLE_GROUP)**: 控制是否在群聊频道启用过滤功能

### 2. 私聊配置
- **私聊目标用户ID列表 (PRIVATE_TARGET_USER_IDS)**: 私聊中需要过滤的用户ID列表
- **私聊用户正则模式列表 (PRIVATE_USER_PATTERNS)**: 与私聊用户ID一一对应的正则表达式

### 3. 群聊配置
- **群聊目标用户ID列表 (GROUP_TARGET_USER_IDS)**: 群聊中需要过滤的用户ID列表
- **群聊用户正则模式列表 (GROUP_USER_PATTERNS)**: 与群聊用户ID一一对应的正则表达式

### 4. 其他配置
- **阻止模式 (BLOCK_MODE)**: 0=完全阻止且不保存记录，1=阻止AI响应但保存记录
- **私聊默认正则模式 (DEFAULT_PRIVATE_PATTERN)**: 私聊中的默认正则表达式
- **群聊默认正则模式 (DEFAULT_GROUP_PATTERN)**: 群聊中的默认正则表达式

## 正则表达式示例

### 基础匹配
- `@AI|@助手`: 包含"@AI"或"@助手"
- `^@.*`: 以"@"开头的消息
- `.*问题.*`: 包含"问题"关键词的消息

### 高级模式
- `^(?!.*@\s*(代码助手|曦)).*$`: 排除包含"@代码助手"或"@曦"的消息
- `^(?=.*@AI)(?=.*请求).*$`: 必须同时包含"@AI"和"请求"
- `^\s*@AI\s+.{5,}$`: "@AI"后跟至少5个字符

## 配置示例

### 示例1: 基础配置
```
启用私聊过滤: True
启用群聊过滤: False
私聊目标用户ID列表: ["user123", "user456"]
私聊用户正则模式列表: ["@助手", "^@AI.*"]
阻止模式: 1
私聊默认正则模式: "@AI"
```

### 示例2: 高级配置
```
启用私聊过滤: True
启用群聊过滤: True
私聊目标用户ID列表: ["user123"]
私聊用户正则模式列表: ["^(?=.*@助手)(?=.*请求).*$"]
群聊目标用户ID列表: ["user456"]
群聊用户正则模式列表: ["^@AI\\s+"]
```

## 使用场景

### 场景1: 私聊专属唤醒词
- 用户在私聊中必须使用特定格式才能唤醒AI
- 例如："@助手 帮我..."才会触发，单独的"你好"不会触发

### 场景2: 群聊@提及过滤
- 群聊中只有正确@提及AI才会响应
- 避免AI对无关对话进行响应

### 场景3: 排除特定模式
- 使用负向匹配排除不想触发的消息
- 例如：排除包含某些敏感词的消息

## 工作原理

1. **频道类型检查**: 根据配置检查是否在当前频道类型启用过滤
2. **用户身份识别**: 获取消息发送者的用户ID
3. **映射构建**: 根据频道类型构建用户到正则模式的映射关系
4. **正则匹配**: 使用正则表达式检查消息内容是否匹配用户模式
5. **执行过滤**: 根据匹配结果和阻止模式决定消息处理方式

## 注意事项

- 正则表达式语法错误会导致该模式被跳过并记录警告
- 私聊用户ID格式通常为：`private_qq号`
- 群聊用户ID格式通常为：`group_群号`
- 正则表达式区分大小写，如需忽略大小写请使用`(?i)`标志
- 异常情况下插件会默认放行消息，确保不会意外阻塞正常对话
- 建议在正式使用前先测试正则表达式的匹配效果

## 版本信息

- 版本: 1.0.0
- 作者: xiaoyu  
- 项目地址: https://github.com/yyl0124/message_filter
"""

import re
from typing import List

from nekro_agent.api import core
from nekro_agent.api.message import ChatMessage
from nekro_agent.api.schemas import AgentCtx
from nekro_agent.api.signal import MsgSignal
from nekro_agent.services.plugin.base import NekroPlugin, ConfigBase
from pydantic import Field

# 1. 插件实例定义
# 创建一个NekroPlugin实例，提供插件的元数据
plugin = NekroPlugin(
    name="消息正则过滤器",
    module_name="message_regex_filter",
    description="通过用户消息回调，根据用户ID和正则表达式模式过滤消息，决定是否触发AI响应。支持私聊和群聊独立配置。",
    version="1.0.0",
    author="xiaoyu",
    url="https://github.com/yyl0124/message_filter",
)


# 2. 插件配置定义
# 使用@plugin.mount_config()装饰器定义插件的配置项
@plugin.mount_config()
class MessageFilterConfig(ConfigBase):
    """消息过滤器配置"""

    # 生效范围配置
    ENABLE_PRIVATE: bool = Field(
        default=True,
        title="启用私聊过滤",
        description="是否在私聊频道启用消息过滤功能。"
    )
    
    ENABLE_GROUP: bool = Field(
        default=False,
        title="启用群聊过滤",
        description="是否在群聊频道启用消息过滤功能。"
    )

    # 私聊配置
    PRIVATE_TARGET_USER_IDS: List[str] = Field(
        default=[],
        title="私聊目标用户ID列表",
        description="私聊中需要应用过滤规则的用户ID列表。请按顺序填写，与下方正则模式列表一一对应。格式：private_qq号"
    )
    
    PRIVATE_USER_PATTERNS: List[str] = Field(
        default=[],
        title="私聊用户正则模式列表",
        description="与上方私聊用户ID一一对应的正则表达式。每个用户的消息必须匹配其对应的正则模式才能触发AI响应。"
    )

    # 群聊配置
    GROUP_TARGET_USER_IDS: List[str] = Field(
        default=[],
        title="群聊目标用户ID列表",
        description="群聊中需要应用过滤规则的用户ID列表。请按顺序填写，与下方正则模式列表一一对应。格式：group_群号_qq号"
    )
    
    GROUP_USER_PATTERNS: List[str] = Field(
        default=[],
        title="群聊用户正则模式列表",
        description="与上方群聊用户ID一一对应的正则表达式。每个用户的消息必须匹配其对应的正则模式才能触发AI响应。"
    )
    
    # 通用配置
    BLOCK_MODE: int = Field(
        default=1,
        title="阻止模式",
        description="当消息被过滤时的处理方式：0 = 完全阻止且不保存消息记录；1 = 阻止AI响应但保存消息记录。"
    )
    
    DEFAULT_PRIVATE_PATTERN: str = Field(
        default="@AI",
        title="私聊默认正则模式（可选）",
        description="如果私聊用户在列表中但未配置专属正则模式时使用的默认正则表达式。留空则严格按照用户-模式映射执行。"
    )
    
    DEFAULT_GROUP_PATTERN: str = Field(
        default="@AI",
        title="群聊默认正则模式（可选）",
        description="如果群聊用户在列表中但未配置专属正则模式时使用的默认正则表达式。留空则严格按照用户-模式映射执行。"
    )


# 获取配置实例，以便在插件方法中使用
config = plugin.get_config(MessageFilterConfig)


# 3. 插件初始化
@plugin.mount_init_method()
async def initialize_plugin():
    """插件初始化方法"""
    core.logger.info(f"插件 '{plugin.name}' 正在初始化...")
    core.logger.info("消息关键词过滤器插件已成功加载")
    
    # 验证私聊配置的合理性
    private_ids = config.PRIVATE_TARGET_USER_IDS
    private_patterns = config.PRIVATE_USER_PATTERNS
    
    if private_ids and private_patterns and len(private_ids) != len(private_patterns):
        core.logger.warning(f"[消息过滤器] 私聊配置警告：用户ID列表长度({len(private_ids)})与正则模式列表长度({len(private_patterns)})不匹配")
    
    # 验证群聊配置的合理性
    group_ids = config.GROUP_TARGET_USER_IDS
    group_patterns = config.GROUP_USER_PATTERNS
    
    if group_ids and group_patterns and len(group_ids) != len(group_patterns):
        core.logger.warning(f"[消息过滤器] 群聊配置警告：用户ID列表长度({len(group_ids)})与正则模式列表长度({len(group_patterns)})不匹配")
    
    # 验证默认正则表达式的有效性
    for pattern_name, pattern in [("私聊默认", config.DEFAULT_PRIVATE_PATTERN), ("群聊默认", config.DEFAULT_GROUP_PATTERN)]:
        if pattern.strip():
            try:
                re.compile(pattern)
            except re.error as e:
                core.logger.error(f"[消息过滤器] {pattern_name}正则表达式语法错误: {pattern} - {e}")
    
    core.logger.success(f"插件 '{plugin.name}' 初始化完成。")


def _build_user_pattern_map(user_ids: List[str], patterns: List[str], default_pattern: str) -> dict:
    """
    构建用户ID到正则模式的映射字典
    
    Args:
        user_ids: 用户ID列表
        patterns: 正则模式列表
        default_pattern: 默认正则模式
        
    Returns:
        dict: 用户ID到编译后正则对象的映射
    """
    user_pattern_map = {}
    
    # 创建用户ID与正则模式的映射
    for i, user_id in enumerate(user_ids):
        pattern_str = None
        
        if i < len(patterns) and patterns[i].strip():
            # 使用对应的正则模式
            pattern_str = patterns[i].strip()
        elif default_pattern.strip():
            # 使用默认正则模式
            pattern_str = default_pattern.strip()
        
        if pattern_str:
            try:
                # 编译正则表达式并存储
                compiled_pattern = re.compile(pattern_str)
                user_pattern_map[user_id] = compiled_pattern
                core.logger.debug(f"[消息过滤器] 为用户 {user_id} 编译正则模式: {pattern_str}")
            except re.error as e:
                core.logger.error(f"[消息过滤器] 用户 {user_id} 的正则表达式语法错误: {pattern_str} - {e}")
                # 正则语法错误时跳过该用户
    
    return user_pattern_map


def _check_message_match(message_text: str, pattern: re.Pattern) -> bool:
    """
    检查消息是否匹配正则模式
    
    Args:
        message_text: 消息文本
        pattern: 编译后的正则表达式对象
        
    Returns:
        bool: 是否匹配
    """
    try:
        return bool(pattern.search(message_text))
    except Exception as e:
        core.logger.error(f"[消息过滤器] 正则匹配时发生错误: {e}")
        return False


# 4. 插件核心实现：用户消息回调
@plugin.mount_on_user_message()
async def handle_user_message(_ctx: AgentCtx, message: ChatMessage) -> MsgSignal | None:
    """
    处理用户消息的回调函数，用于在AI响应前进行过滤。

    该回调函数会根据插件配置执行以下过滤逻辑：
    1. 检查当前频道类型是否启用过滤
    2. 根据频道类型构建用户ID到正则模式的映射关系
    3. 如果消息发送者不在映射中，则不进行过滤，消息正常处理
    4. 如果消息发送者在映射中，检查消息内容是否匹配其专属正则模式
    5. 根据阻止模式决定被过滤消息的处理方式

    Args:
        _ctx (AgentCtx): Agent上下文对象
        message (ChatMessage): 用户发送的消息对象，包含纯文本内容等信息。

    Returns:
        MsgSignal | None:
            - `MsgSignal.BLOCK_TRIGGER`: 阻止AI响应但保存消息记录
            - `MsgSignal.BLOCK_ALL`: 完全阻止且不保存消息记录
            - `None`: 消息通过过滤检查，将继续正常处理
    """
    try:
        # 首先记录回调被触发
        core.logger.info("[消息过滤器] 用户消息回调被触发")
        
        # 检查频道类型
        channel_type = getattr(_ctx, 'channel_type', None)
        core.logger.info(f"[消息过滤器] 频道类型: {channel_type}")
        
        # 根据频道类型和配置决定是否进行过滤
        if channel_type == "private" and not config.ENABLE_PRIVATE:
            core.logger.info("[消息过滤器] 私聊过滤已禁用，直接放行消息")
            return None
        elif channel_type == "group" and not config.ENABLE_GROUP:
            core.logger.info("[消息过滤器] 群聊过滤已禁用，直接放行消息")
            return None
        elif channel_type not in ["private", "group"]:
            core.logger.info(f"[消息过滤器] 未知频道类型 ({channel_type})，直接放行消息")
            return None
        
        # 尝试多种方式获取用户ID
        sender_id = None
        
        # 方式1: 尝试 from_user_id (文档示例中提到的)
        if hasattr(_ctx, 'from_user_id') and _ctx.from_user_id:
            sender_id = _ctx.from_user_id
        # 方式2: 在私聊中，channel_id 就是用户ID；在群聊中可能需要其他方式
        elif _ctx.channel_id:
            sender_id = _ctx.channel_id
        # 方式3: 尝试通过 db_user 获取用户ID
        elif hasattr(_ctx, 'db_user') and _ctx.db_user:
            sender_id = str(_ctx.db_user.id) if hasattr(_ctx.db_user, 'id') else None
        
        if not sender_id:
            core.logger.warning(f"[消息过滤器] 无法获取用户ID。可用属性: channel_id={getattr(_ctx, 'channel_id', None)}")
            return None
            
        plain_text: str = message.content_text
        
        core.logger.info(f"[消息过滤器] 收到消息。发信人 ID: '{sender_id}', 消息内容: '{plain_text[:50]}...'")
        
        # 根据频道类型获取对应的配置项
        if channel_type == "private":
            target_ids = config.PRIVATE_TARGET_USER_IDS
            patterns = config.PRIVATE_USER_PATTERNS
            default_pattern = config.DEFAULT_PRIVATE_PATTERN
            channel_name = "私聊"
        else:  # group
            target_ids = config.GROUP_TARGET_USER_IDS
            patterns = config.GROUP_USER_PATTERNS
            default_pattern = config.DEFAULT_GROUP_PATTERN
            channel_name = "群聊"
        
        block_mode = config.BLOCK_MODE

        core.logger.info(f"[消息过滤器] {channel_name}配置 - 目标用户ID列表: {target_ids}")
        core.logger.info(f"[消息过滤器] {channel_name}配置 - 正则模式列表: {patterns}")
        core.logger.info(f"[消息过滤器] {channel_name}配置 - 默认正则模式: '{default_pattern}'")
        core.logger.info(f"[消息过滤器] 当前配置 - 阻止模式: {block_mode}")

        # 构建用户正则模式映射
        user_pattern_map = _build_user_pattern_map(target_ids, patterns, default_pattern)
        core.logger.info(f"[消息过滤器] 构建的{channel_name}用户正则模式映射: {list(user_pattern_map.keys())}")

        # 如果没有配置任何用户映射，则直接放行
        if not user_pattern_map:
            core.logger.info(f"[消息过滤器] {channel_name}用户正则模式映射为空，直接放行消息。")
            return None

        # 检查发送者是否在映射中
        if sender_id in user_pattern_map:
            pattern = user_pattern_map[sender_id]
            
            # 检查消息是否匹配该用户的专属正则模式
            is_match = _check_message_match(plain_text, pattern)
            core.logger.info(f"[消息过滤器] 用户 '{sender_id}' 是{channel_name}目标用户，正则模式: '{pattern.pattern}', 消息匹配结果: {is_match}")

            if is_match:
                core.logger.info(f"[消息过滤器] 用户 {sender_id} 的消息通过了正则模式 '{pattern.pattern}' 过滤。")
                # 消息符合要求，返回None以继续处理
                return None
            else:
                # 消息来自目标用户，但不匹配其专属正则模式，根据阻止模式决定处理方式
                if block_mode == 0:
                    # 完全阻止且不保存消息
                    core.logger.info(f"[消息过滤器] 已完全阻止用户 {sender_id} 的消息（不保存记录），因为它不匹配专属正则模式 '{pattern.pattern}'。")
                    return MsgSignal.BLOCK_ALL
                else:
                    # 阻止AI响应但保存消息记录
                    core.logger.info(f"[消息过滤器] 已阻止用户 {sender_id} 的消息触发AI（保存记录），因为它不匹配专属正则模式 '{pattern.pattern}'。")
                    return MsgSignal.BLOCK_TRIGGER
        else:
            # 消息发送者不在映射中，直接放行
            core.logger.info(f"[消息过滤器] 发信人 '{sender_id}' 不在{channel_name}用户正则模式映射中，直接放行消息。")
            return None

    except Exception as e:
        # 在过滤逻辑发生异常时，记录错误并默认放行，以避免意外阻塞所有消息
        core.logger.error(f"[消息过滤器] 消息过滤时发生未知错误: {e}", exc_info=True)
        return None


# 5. 插件资源清理
@plugin.mount_cleanup_method()
async def clean_up():
    """清理消息过滤器插件的资源"""
    core.logger.info("[消息过滤器] 插件正在清理资源...")
    core.logger.info("[消息过滤器] 消息过滤器插件无需清理资源。")
