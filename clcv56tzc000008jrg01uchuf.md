# Performance comparison: Amazon Aurora vs. Amazon Aurora Serverless v2

Serverless technologies are everywhere these days, and databases are no exception. [AWS Aurora Serverless V2](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.html) has been with us for some time and is already a viable option for use in more varied production environments than its predecessor. The goal of the comparison is to know if there is any difference from the performance perspective between an Amazon Aurora or an Amazon Aurora Serverless v2 with the same amount of memory, both PostgreSQL compatible.

## Design

We will use two AWS Lambda Functions that share the same [code base](https://github.com/raulnq/aws-aurora-serverless). The first points to an Amazon Aurora database, and the other to an Amazon Aurora Serverless v2 database. We will consider the following scenarios during this performance comparison:

* **db.t3.medium vs. 2 ACU**: Using 1, 2, 5, 10, 50,100, 150, 200, and 250 concurrent connections.
    
* **db.t3.large vs. 4 ACU**: Using 25, 50, 100, 150, 300, and 400 concurrent connections.
    
* **db.r5.large vs. 8 ACU**: Using 300, 450, 600, 750, and 1000 concurrent connections.
    

Every test will send requests to the AWS Lambda Functions for ten minutes.

## Results

### Response Time Distribution

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1673648118481/cb082d1f-4fa8-4017-a429-4edd30b76894.png align="center")

Let's explain the chart above. The first row shows that using one concurrent user, 49.4% of the request to the provisioned instance are in the interval of 100ms-150ms, while only 42% of the requests to the serverless instance are in the same interval. Moving up in the chart, we notice that the response time distributions are similar until the 50 concurrent connections.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1673647726005/35ab1530-4b24-49b8-a4d5-fb72540b01c3.png align="center")

In this scenario, the response time distributions are even until the 100 concurrent connections. Between 150 and 450 concurrent connections, the provisioned instance performs much better.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1673648227539/ca9aa524-8ea8-4fd0-acb1-1cce60918216.png align="center")

The last scene presents the same behavior. This time 750 concurrent connections are the limit where both instances have similar response time distributions.

### Average Response Time

The following charts will confirm the behavior previously detected:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1673649771669/b809d693-3592-4e86-a7cd-2dfca8dc5e2f.png align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1673649803244/b1685c2e-eed1-47b2-8bb9-b5677e89f90a.png align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1673649869298/57371590-de12-4990-b84e-ee6e31e7d8e7.png align="center")

## Conclusions

This comparison only covers three types of provisioned instances but we think that is enough to suspect that those perform better than their corresponding serverless version in the presence of a high number of concurrent connections. That could mean that under a load of 1000 concurrent connections, a db.r5.large instance with its 16 GB of memory could perform as a 9 or 10 ACU serverless instance and not as an 8 ACU instance (remember that every ACU is around 2 GB of memory). Thanks, and happy coding.