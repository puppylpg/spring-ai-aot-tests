# Spring AI MistralAI AOT Demo Application

## 流程
**从ApplicationContext里找bean作为函数（function callback）。bean name就是函数名。**

function callback:
- biFunction: lambda
- name: retrievePaymentStatus
- description: Get payment status of a transaction
- inputType: Transaction
- inputTypeSchema: 
```
{
  "$schema" : "https://json-schema.org/draft/2020-12/schema",
  "type" : "object",
  "properties" : {
  "transactionId" : {
  "type" : "string"
  }
  }
  }
```

## request function推荐
问了两个id：
```
ChatCompletionRequest[model=mistral-small-latest, messages=[ChatCompletionMessage[content=What's the status of my transaction with id T1005 and T1001?, role=USER, name=null, toolCalls=null, toolCallId=null]], tools=[org.springframework.ai.mistralai.api.MistralAiApi$FunctionTool@cedee22], toolChoice=null, temperature=0.7, topP=1.0, maxTokens=null, stream=false, safePrompt=false, stop=null, randomSeed=null, responseFormat=null]
```

## response
推荐了两个ToolCall，其实是同一个，只是因为没有batch：
```
ChatCompletion[id=307e8dd2e6f7489e992872638288931d, object=chat.completion, created=1731847111, model=mistral-small-latest, choices=[Choice[index=0, message=ChatCompletionMessage[content=, role=ASSISTANT, name=null, toolCalls=[ToolCall[id=m1YWnMkVD, type=function, function=ChatCompletionFunction[name=retrievePaymentStatus, arguments={"transactionId": "T1005"}]], ToolCall[id=VqJruhgQO, type=function, function=ChatCompletionFunction[name=retrievePaymentStatus, arguments={"transactionId": "T1001"}]]], toolCallId=null], finishReason=TOOL_CALLS, logprobs=null]], usage=Usage[promptTokens=107, totalTokens=158, completionTokens=51]]
```

获取tool call里的function name和arguments，从registry里找函数调用，然后调用函数：`AbstractFunctionCallback<I, O>`
```java
	@Override
	public String call(String functionInput, ToolContext toolContext) {
		I request = fromJson(functionInput, this.inputType);
		O response = this.apply(request, toolContext);
		return this.responseConverter.apply(response);
	}
```
**入参是string转I，结果是O转string。**

调用了两次。每一次的结果为了继续发给llm生成答案，要记下来id和name、data：
```
ToolResponse[id=m1YWnMkVD, name=retrievePaymentStatus, responseData={"status":"Pending"}]
```

```
ToolResponseMessage{responses=[ToolResponse[id=m1YWnMkVD, name=retrievePaymentStatus, responseData={"status":"Pending"}], ToolResponse[id=VqJruhgQO, name=retrievePaymentStatus, responseData={"status":"Paid"}]], messageType=TOOL, metadata={messageType=TOOL}}
```

根据返回里的状态判断没结束，继续调用llm做答案生产。

## request 答案生成
把上面的结果作为prompt再发给llm：
```
ChatCompletionRequest[model=mistral-small-latest, messages=[ChatCompletionMessage[content=What's the status of my transaction with id T1005 and T1001?, role=USER, name=null, toolCalls=null, toolCallId=null], ChatCompletionMessage[content=, role=ASSISTANT, name=null, toolCalls=[ToolCall[id=m1YWnMkVD, type=function, function=ChatCompletionFunction[name=retrievePaymentStatus, arguments={"transactionId": "T1005"}]], ToolCall[id=VqJruhgQO, type=function, function=ChatCompletionFunction[name=retrievePaymentStatus, arguments={"transactionId": "T1001"}]]], toolCallId=null], ChatCompletionMessage[content={"status":"Pending"}, role=TOOL, name=retrievePaymentStatus, toolCalls=null, toolCallId=m1YWnMkVD], ChatCompletionMessage[content={"status":"Paid"}, role=TOOL, name=retrievePaymentStatus, toolCalls=null, toolCallId=VqJruhgQO]], tools=[org.springframework.ai.mistralai.api.MistralAiApi$FunctionTool@483f286e], toolChoice=null, temperature=0.7, topP=1.0, maxTokens=null, stream=false, safePrompt=false, stop=null, randomSeed=null, responseFormat=null]
```
看起来上下文也发回去了。

## response 答案生成
`The status of transaction T1005 is Pending, and the status of transaction T1001 is Paid.`

```
ChatResponse [metadata={ id: 14b9a4bc75c7467eb689d4959a36edee, usage: Usage[promptTokens=236, totalTokens=263, completionTokens=27], rateLimit: org.springframework.ai.chat.metadata.EmptyRateLimit@58f2466c }, generations=[Generation[assistantMessage=AssistantMessage [messageType=ASSISTANT, toolCalls=[], textContent=The status of transaction T1005 is Pending, and the status of transaction T1001 is Paid., metadata={finishReason=STOP, index=0, role=ASSISTANT, id=14b9a4bc75c7467eb689d4959a36edee, messageType=ASSISTANT}], chatGenerationMetadata=ChatGenerationMetadata{finishReason=STOP,contentFilterMetadata=null}]]]
```

这次根据response里的状态判断结束了，所以就不需要继续call了。

## run
```
export JAVA_HOME=/Library/Java/JavaVirtualMachines/graalvm-jdk-21.0.2+13.1/Contents/Home

./mvnw clean install -Pnative native:compile
```

```
./target/mistralai-aot-demo
```

## Generate GraalVM configs

```
java -Dspring.aot.enabled=true \
    -agentlib:native-image-agent=config-output-dir=target/config-dir/ \
    -jar target/mistralai-aot-demo-0.0.1-SNAPSHOT.jar
```