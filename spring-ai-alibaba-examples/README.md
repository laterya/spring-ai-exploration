## Spring AI Alibaba 组件概览

### 核心组件
- `spring-ai-alibaba-bom` - 统一版本管理
- `spring-ai-alibaba-starter-dashscope` - 百炼模型服务适配
- `spring-ai-alibaba-graph-core` - 智能体 Graph 框架核心

### 功能组件
- `spring-ai-alibaba-starter-nl2sql` - 自然语言到 SQL 转换
- `spring-ai-alibaba-starter-memory` - 会话记忆管理
- `spring-ai-alibaba-starter-arms-observation` - ARMS 可观测性

### Nacos 集成组件
- `spring-ai-alibaba-starter-nacos-mcp-client/server` - Nacos MCP 客户端/服务端（推荐 Nacos 3.0.1）
- `spring-ai-alibaba-starter-nacos2-mcp-client/server` - Nacos 2.x 兼容版本
- `spring-ai-alibaba-starter-nacos-prompt` - Nacos Prompt 管理

### 扩展组件
- `spring-ai-alibaba-starter-tool-calling-*` - 工具调用组件
- `spring-ai-alibaba-starter-document-reader-*` - 文档读取组件

## ChatClient 详解

`ChatClient` 提供了与 AI 模型通信的流畅 API，支持同步和反应式编程模型。

### 创建方式

#### 自动配置方式
使用默认的 `ChatClient.Builder` bean（如 [HelloworldController](file:///Users/yupan/Project/spring-ai-exploration/spring-ai-alibaba-examples/src/main/java/top/laterya/spring/ai/alibaba/examples/controller/HelloworldController.java#L18-L67) 示例）

#### 手动编程方式
```java
// 禁用自动配置
// spring.ai.chat.client.enabled=false

ChatClient chatClient = ChatClient.builder(myChatModel).build();
// 或
ChatClient chatClient = ChatClient.create(myChatModel);
```


### 响应处理

#### ChatResponse
```java
ChatResponse response = chatClient.prompt()
    .user("Tell me a joke")
    .call()
    .chatResponse();
```


#### 实体类映射
```java
record ActorFilms(String actor, List<String> movies) {}

ActorFilms films = chatClient.prompt()
    .user("Generate filmography")
    .call()
    .entity(ActorFilms.class);
```


#### 流式响应
```java
Flux<String> stream = chatClient.prompt()
    .user("Tell me a joke")
    .stream()
    .content();
```


## Advisors 机制

### 检索增强生成 (RAG)
```java
// 基础 RAG 实现
ChatResponse response = ChatClient.builder(chatModel)
    .build()
    .prompt()
    .advisors(new QuestionAnswerAdvisor(vectorStore, SearchRequest.defaults()))
    .user(userText)
    .call()
    .chatResponse();

// 运行时过滤
String content = chatClient.prompt()
    .user("Please answer my question")
    .advisors(a -> a.param(QuestionAnswerAdvisor.FILTER_EXPRESSION, "type == 'Spring'"))
    .call()
    .content();
```


### 聊天记忆管理
提供两种实现：
- `InMemoryChatMemory` - 内存存储
- `CassandraChatMemory` - 带 TTL 的持久化存储

```java
CassandraChatMemory memory = CassandraChatMemory.create(
    CassandraChatMemoryConfig.builder()
        .withTimeToLive(Duration.ofDays(1))
        .build()
);
```


三种 Advisor 实现：
- `MessageChatMemoryAdvisor` - 作为消息集合添加到提示中
- `PromptChatMemoryAdvisor` - 添加到系统文本中
- `VectorStoreChatMemoryAdvisor` - 从向量存储检索历史记录

### 日志记录
```java
// 启用日志记录
ChatResponse response = ChatClient.create(chatModel)
    .prompt()
    .advisors(new SimpleLoggerAdvisor())
    .user("Tell me a joke?")
    .call()
    .chatResponse();

// 设置日志级别
// org.springframework.ai.chat.client.advisor=DEBUG

// 自定义日志格式
SimpleLoggerAdvisor customLogger = new SimpleLoggerAdvisor(
    request -> "Custom request: " + request.userText,
    response -> "Custom response: " + response.getResult()
);
```


### 完整示例
```java
@Service
public class CustomerSupportAssistant {
    private final ChatClient chatClient;

    public CustomerSupportAssistant(ChatClient.Builder builder, 
                                  VectorStore vectorStore, 
                                  ChatMemory chatMemory) {
        this.chatClient = builder
            .defaultSystem("You are a customer support agent...")
            .defaultAdvisors(
                new PromptChatMemoryAdvisor(chatMemory),
                new QuestionAnswerAdvisor(vectorStore, SearchRequest.defaults()),
                new LoggingAdvisor())
            .defaultFunctions("getBookingDetails", "changeBooking", "cancelBooking")
            .build();
    }

    public Flux<String> chat(String chatId, String userMessageContent) {
        return this.chatClient.prompt()
            .user(userMessageContent)
            .advisors(a -> a.param(CONVERSATION_ID, conversationId))
            .stream()
            .content();
    }
}
```
