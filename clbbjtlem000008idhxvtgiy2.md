# Reducing AWS Lambda Cold Starts with NET. 7 Native AOT

In the [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html) ecosystem, the cold start problem could be a key factor to use it or not, especially if you're developing a customer-facing application that needs to operate in real-time. AWS is continuously releasing new features to minimize the impact on our applications, such as [Provisioned concurrency](https://docs.aws.amazon.com/lambda/latest/dg/provisioned-concurrency.html) or [Lambda SnapStart](https://docs.aws.amazon.com/lambda/latest/dg/snapstart.html) (only available for Java). NET 7 is also helping to reduce the impact of the cold start problem with Native AOT. To understand it, let's check first how JIT (Just-In-Time compilation) works: 

- A language-specific compiler converts the source code to the IL (Intermediate Language), MSIL (Microsoft Intermediate Language), or CIL (Common Intermediate Language). Those are CPU-agnostic sets of instructions that can be converted to native code. 
- IL is then converted into the native code by the JIT compiler. This native code is specific to the computer environment that the JIT compiler runs on.

On the other hand, Native AOT produces a self-contained application that has been Ahead-Of-Time (AOT) compiled into native code at the time of publishing. That improves the performance and reduces startup time as we do not have to execute the JIT compiler when the application runs. But there are some drawbacks, such as:

- No dynamic loading (for example, `Assembly.LoadFile`).
- No runtime code generation (for example, `System.Reflection.Emit`).
- Requires trimming  (when publishing the application, the .NET SDK analyzes the entire application and removes all unused code. However, it can be not easy to determine what is used)
- Should be targeted for console-type applications (not ASP.NET Core).
- Not all the runtime libraries are fully annotated to be native AOT compatible.
- Only Linux and Windows are supported for now.

To measure the improvements in the cold start time. We build two applications, the first one with .NET 6 and the second using .NET 7 with Native AOT (following this [post](https://aws.amazon.com/blogs/compute/building-serverless-net-applications-on-aws-lambda-using-net-7/)). Both Lambda Function are going to do the same (the source code can be found [here](https://github.com/raulnq/aws-lambda-native-aot):

- Receives an HTTP request from API Gateway.
- Store the request in a DynamoDb table.
- Send a message to an SQS queue.
- Return an HTTP response.

For both Lambda Functions, we are using 512 Mb. We run a test (three times, redeploying the applications between runs) to hit the endpoints for 5 minutes (one request per second). Let's take a look at the results:

| Application|Min|Avg|Max|Median|P95|P99|Stddev|
|---|---|---|---|---|---|---|---|
|Native AOT|18ms|29ms|869ms|23ms|46ms|86ms|51ms|
|JIT|17ms|34ms|1417ms|23ms|54ms|81ms|83ms|

| Application|Min|Avg|Max|Median|P95|P99|Stddev|
|---|---|---|---|---|---|---|---|
|Native AOT|19ms|28ms|772ms|23ms|47ms|79ms|46ms|
|JIT|18ms|36ms|1547ms|24ms|65ms|99ms|92ms|

| Application|Min|Avg|Max|Median|P95|P99|Stddev|
|---|---|---|---|---|---|---|---|
|Native AOT|18ms|29ms|851ms|23ms|50ms|91ms|51ms|
|JIT|20ms|37ms|1644ms|25ms|56ms|75ms|97ms|

The cold start time (Max) produced by .NET 7 with Native AOT is almost half of .NET 6. Another thing to notice is that the standard deviation is lower, suggesting that the response times are more stable. Checking the logs, we see another advantage of .NET 7 with Native AOT, the Lambda Function is using less memory (81Mb vs. 98Mb), and we confirm that the cold start time (354.51ms vs. 1015.76ms) is lower (almost three times from the AWS perspective):

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1670289130232/C1Ej2Wq0o.png align="center")

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1670289091487/SeNdLCve5.png align="center")

But nothing could be perfect, and the package size of the Lambda Function produced by .NET 7 with Native AOT is almost four times larger (2.6Mb vs. 10.4Mb). Thanks, and happy coding.


