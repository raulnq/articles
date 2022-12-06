# AWS: Single Purpose Lambda vs Monolithic Lambda

AWS Lambda can be a great hosting option, providing us with several advantages, such as cost-effectiveness(we'll only pay for what we use) and its natural ability to scale without intervention. But, there are new challenges, such as answering the question: **How to organize the code into Lambda functions?** Today we will review two options to answer this question: Single Purpose Lambda and Monolithic Lambda. Let's define them first.

## Single Purpose Lambda

Under this option, every Lambda function is focused on accomplishing a single action. All routing logic lives inside the API Gateway:

![Single Purpose Lambda II.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1664637413046/4uzUYZBqF.png align="left")

## Monolithic Lambda

On the other side, the Lambda function can handle more than one action. The routing logic is divided into the API Gateway and the Lambda function itself:

![Monolithic Lambda II.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1664637569288/CIq3Xdqqv.png align="left")

## Comparison

### Isolation

When each Lambda function focuses on a specific task, in most cases, those functions can work independently. Changes in one of them should not affect the others, allowing us to develop and deploy each Lambda function in a high degree of isolation. Here, the clear winner is the Single Purpose Lambda structure.

### Control

Each Lambda function has its own set of settings (memory, security, scaling, routing, etc.). In addition, there are logs, metrics, traces, and other resources related to it. Thus, every feature in a system built under a Single Purpose Lambda structure could take advantage of this flexibility.

### Cold Starts

More Lambda functions usually mean more cold starts. Having everything in a single Lambda function might make sense here. But there is a downside, longer cold starts (because a larger Lambda function usually means more initialization code).

### Package Size

With the Single Purpose Lambda structure will be difficult to exceed the 250Mb per package (50Mb zipped) that we have as a size limit. On the other hand, it is something to keep in mind when using the Monolithic Lambda structure. 

### Configuration

When the number of Lambda functions grows, a new set of problems arise, such as: 

- How to deploy all the functions? 
- Should we have a naming/tagging strategy? 

Usually, to solve these problems, we introduce one or more tools (which we must learn first). And those tools are not alone. They come with several configuration files to create and maintain. Don't forget the routing logic that lives in the API Gateway that needs to be maintained somehow. Here, the clear winner is the Monolithic Lambda structure.

### Development

Monolithic Lambda structure feels more like building a traditional application. For example, in the routing logic, usually, our preference is to have code to maintain instead of configuration to maintain.

## Conclusions

The Monolithic Lambda structure favors developer productivity over operational complexity. The Single Purpose Lambda structure provides fine control but requires a precise operational configuration. But what is missing here is the context, the problem we are trying to solve. We have to choose the structure that best fits our requirements. You would be surprised how many times the Monolithic Lambda structure can be a better fit for your solution, but if you see that some parts of your system need the benefits provided by the Single Purpose Lambda structure, move only those parts there. Both structures can be combined to give you the right mix according to the requirements of your system.


