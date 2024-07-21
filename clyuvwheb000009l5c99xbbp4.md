---
title: "Kubernetes Pods Resource Sizing"
datePublished: Sun Jul 21 2024 01:33:34 GMT+0000 (Coordinated Universal Time)
cuid: clyuvwheb000009l5c99xbbp4
slug: kubernetes-pods-resource-sizing
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1721426443863/a26de3b6-46f0-455d-9dad-6e7ed76f3c77.png
tags: kubernetes, pods, sizing

---

When deploying applications to Kubernetes, correctly sizing the resources for our pods is crucial for achieving optimal performance and efficient resource utilization. This involves setting the right CPU and memory requests and limits based on our application's needs. In this post, we will share an approach to help define these parameters accurately when the application is based on requests.

### Application Requirements

The first step is to define the goals our application aims to achieve. These are usually one or more of the following:

* **Response Time**: The time it takes for the API to respond to a request.
    
* **Throughput**: The number of requests the API can handle per second.
    
* **Error Rate**: The percentage of failed requests.
    
* **Resource Utilization**: CPU and memory usage on the server.
    

For our practical example, we will use:

* 95% of the requests must have a response time of less than 200 milliseconds.
    
* We need to handle 50 requests per second.
    
* The error rate must be less than 1% of the requests.
    
* The CPU utilization must be less than 90%.
    
* The memory utilization must be less than 50%.
    

### Set Up the Environment

We need an environment that closely resembles production, which is a Kubernetes cluster. Additionally, we need to set up monitoring tools to gather performance metrics, such as CPU and memory usage of our pods.

In this example, we chose the standalone version of [Kubernetes](https://docs.docker.com/desktop/install/windows-install/) included in [Docker Desktop](https://docs.docker.com/desktop/install/windows-install/). To monitor the cluster, we installed the [kube-prometheus](https://github.com/prometheus-operator/kube-prometheus) stack, which includes components like Prometheus and Grafana, among others. The easiest way to install it is by using [Helm](https://helm.sh/docs/intro/install/):

```powershell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace
```

Once installed, we can access Grafana locally by using a simple port forwarding command:

```powershell
kubectl port-forward svc/prometheus-grafana 3000:80 -n monitoring
```

Open [http://localhost:3000/login](http://localhost:3000/login) in a browser, and use `admin` as the username and `prom-operator` as the password.

### Select the Load Testing Tool

Choose a load-testing tool that fits your needs. Many options are available, but we recommend using [K6](https://k6.io/docs/) due to its developer-centric approach.

### Set Initial Resources

Choose initial CPU and memory resources based on your experience; this will be the starting point for the analysis. In our case, we selected `128m` for CPU and `256Mi` for memory.

### Deploy the Application

We built an API to calculate Pi using the [Leibniz series](https://en.wikipedia.org/wiki/Leibniz_formula_for_%CF%80) to test this approach. Download the code [here](https://github.com/raulnq/pod-sizing), and create the image with the following command:

```powershell
docker build -t raulnq/mywebapi:1.0 -f .\MyWebApi\Dockerfile .
```

The `resources.yaml` file defines a `deployment` with the requested resources and a `service` using the `NodePort` type:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
  labels:
    app: api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api-container
          image: raulnq/mywebapi:2.0
          resources:
            requests:
             memory: "128Mi"
             cpu: "256m"
            limits:
             memory: "128Mi"
             cpu: "256m"
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
  labels:
    app: api
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
      nodePort: 30007
  selector:
    app: api
```

Deploy the application with the following command:

```powershell
kubectl apply -f resources.yaml
```

### Define the Load Tests

The next step is to identify the endpoints to be tested. In some cases, we can define user journeys and use fake input data to mimic real-world usage. The `load.js` file defines our test:

```javascript
import http from 'k6/http';
import { sleep } from 'k6';
export const options = {
    stages: [
        { duration: '1m', target: 10 }, // ramp-up
        { duration: '2m', target: 10 }, // stay
        { duration: '1m', target: 0 },  // ramp-down
    ],
    thresholds: {
        http_req_duration: ['p(95)<200'], // 95% of requests must complete below 300ms
        http_req_failed: ['rate<0.01'], // Error rate should be less than 1%
    },
};

export default function () {
    http.get('http://www.api.com/pi?iterations=5000000');
    sleep(1);
}
```

The test above specifies three stages:

* One minute to increase the load from zero to the target VU (virtual users).
    
* Two minutes maintaining the target load.
    
* One minute to decrease the load from the target to zero VU.
    

In addition, we are setting thresholds to determine when our tests exceed our goals. You can run the script using the following command:

```powershell
k6 run load.js
```

### Run the Load Tests and Adjust Resources

Run an initial test to understand the performance under minimal load. Then, based on the results, we will gradually increase the load for the next run, updating the resources if necessary until we achieve the goal defined in the first step. The results of the first run are as follows (data collected using k6 and Grafana):

| Metric | Value |
| --- | --- |
| P95 | 19.77 ms |
| Requests per second | 7.48 |
| CPU | 0.066 Mi |
| % CPU usage | 51.56 |
| Memory | 41.9 mb |
| % memory usage | 16.36 |
| % error | 0 |

For 10 VUs, we met all our goals except for requests per second. We only achieved 7.48, which is far from the target of 50. So, let's increase the VUs to 70 and run the test:

| Metric | Value |
| --- | --- |
| P95 | 4980 ms |
| Requests per second | 16.43 |
| CPU | 0.125 Mi |
| % CPU usage | 97.65 |
| Memory | 54.3 mb |
| % memory usage | 21.21 |
| % error | 0 |

From these results, we can conclude that we need to adjust the amount of CPU requested due to the high CPU usage and the slow response time we are experiencing with this setup. Let's modify the CPU request to `256m` and the memory request to `128Mi`, redeploy the application, and run the test:

| Metric | Value |
| --- | --- |
| P95 | 1090 ms |
| Requests per second | 33.19 |
| CPU | 0.253 Mi |
| % CPU usage | 98.82 |
| Memory | 46.3 mb |
| % memory usage | 36.17 |
| % error | 0 |

Closer to the goals but not there yet, modify the CPU request to `512m`, redeploy the application, and run the test:

| Metric | Value |
| --- | --- |
| P95 | 89.17 ms |
| Requests per second | 51.10 |
| CPU | 0.449 Mi |
| % CPU usage | 87.69 |
| Memory | 45.7 mb |
| % memory usage | 35.70 |
| % error | 0 |

It looks like this final setup meets all our goals.

### When Does It Break?

So far, we have a viable resource setup for our pods, but it's good to know when it will start to fail. To find out, we will stress the API beyond 50 requests per second, using 80, 90, and 100 VUs to see what happens:

| Metric | Value (80 VUs) | Value (90 VUs) | Value (100 VUs) |
| --- | --- | --- | --- |
| P95 | 107.74 ms | 127.63 ms | 204.65 ms |
| Requests per second | 57.30 | 64.22 | 68.01 |
| CPU | 0.475 Mi | 0.491 Mi | 0.506 Mi |
| % CPU usage | 92.77 | 95.89 | 98.82 |
| Memory | 46 mb | 49.1 | 49.5 |
| % memory usage | 35.93 | 38.35 | 38.67 |
| % error | 0 | 0 | 0 |

We can conclude that starting around 57 requests per second, we still respond within the expected time, but the CPU utilization exceeds 90%. At around 68 requests per second, we no longer achieve the expected response time, and the CPU is at its limit.

By defining application requirements, setting up a realistic testing environment, and using appropriate load testing-tools, we can iteratively adjust CPU and memory requests to meet our performance goals. Through systematic testing and monitoring, we can ensure that our application handles the desired load while maintaining acceptable response times, error rates, and resource usage. This approach not only helps in achieving the desired performance but also provides insights into the limits of our setup, allowing for better planning and scaling in production environments. Thank you, and happy coding.