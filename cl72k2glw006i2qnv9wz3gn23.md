## Load Testing with K6

During our development, usually, there is a time when we want to determine how many users our system can handle. Load Testing can help us to answer that:

> Load Testing is a type of Performance Testing used to determine a system's behavior under both normal and peak conditions.

> Load Testing is used to ensure that the application performs satisfactorily when many users access it at the same time.

There are several Load Testing tools like [JMeter](https://jmeter.apache.org/), but today we are going to use [K6](https://k6.io/):

> K6 is a modern load testing tool, building on our years of experience in the load and performance testing industry. It provides a clean, approachable scripting API, local and cloud execution, and flexible configuration.

### Installation

[Here](https://k6.io/docs/getting-started/installation/) you can find how to install K6 in different OS. In our case, we will run:

```
choco install k6
``` 

### Basic script syntax

> K6 works with the concept of virtual users (VUs) that execute scripts - they're essentially glorified, parallel `while(true)` loops. Scripts are written using JavaScript as ES6 modules, which allows you to break larger tests into smaller and more reusable pieces, making it easy to scale tests across an organization.

Create a `script.js` file with the following content:

```js
import http from 'k6/http';
import { sleep } from 'k6';

export default function () {
  http.get('https://test.k6.io');
  sleep(1);
}
``` 

- `default function`: Defines the entry point for our VUs, similar to the `main()` function in many languages.
- `http.get`: Issue an HTTP GET request. [Here](https://k6.io/docs/using-k6/http-requests/) you can check more about HTTP Requests.
- `sleep`: Suspend VU execution for the specified duration. Simulate the interaction of a real user.

### Running the script

Run `k6 run script.js` to see the following metrics:

```powershell
     data_received..................: 17 kB 12 kB/s
     data_sent......................: 438 B 315 B/s
     http_req_blocked...............: avg=273.24ms min=273.24ms med=273.24ms max=273.24ms p(90)=273.24ms p(95)=273.24ms
     http_req_connecting............: avg=99.78ms  min=99.78ms  med=99.78ms  max=99.78ms  p(90)=99.78ms  p(95)=99.78ms
     http_req_duration..............: avg=105.74ms min=105.74ms med=105.74ms max=105.74ms p(90)=105.74ms p(95)=105.74ms
       { expected_response:true }...: avg=105.74ms min=105.74ms med=105.74ms max=105.74ms p(90)=105.74ms p(95)=105.74ms
     http_req_failed................: 0.00% ✓ 0        ✗ 1
     http_req_receiving.............: avg=0s       min=0s       med=0s       max=0s       p(90)=0s       p(95)=0s
     http_req_sending...............: avg=0s       min=0s       med=0s       max=0s       p(90)=0s       p(95)=0s
     http_req_tls_handshaking.......: avg=122.08ms min=122.08ms med=122.08ms max=122.08ms p(90)=122.08ms p(95)=122.08ms
     http_req_waiting...............: avg=105.74ms min=105.74ms med=105.74ms max=105.74ms p(90)=105.74ms p(95)=105.74ms
     http_reqs......................: 1     0.719507/s
     iteration_duration.............: avg=1.38s    min=1.38s    med=1.38s    max=1.38s    p(90)=1.38s    p(95)=1.38s
     iterations.....................: 1     0.719507/s
     vus............................: 1     min=1      max=1
     vus_max........................: 1     min=1      max=1
``` 

> Metrics measure how a system performs under test conditions. By default, k6 automatically collects built-in metrics. Besides built-ins, you can also make custom metrics.

- `data_received`: The amount of received data.
- `data_sent`: The amount of data sent.
- `http_req_blocked`:Time spent blocked (waiting for a free TCP connection slot) before initiating the request. 
- `http_req_connecting`: Time spent establishing TCP connection to the remote host. 
- `http_req_tls_handshaking`: Time spent handshaking TLS session with remote host.
- `http_req_sending`: Time spent sending data to the remote host. 
- `http_req_waiting`: Time spent waiting for response from remote host.
- `http_req_receiving`: Time spent receiving response data from the remote host.
- `http_req_duration`: Total time for the request. It's equal to http_req_sending + http_req_waiting + http_req_receiving.
- `http_reqs`: How many total HTTP requests k6 generated.
- `http_req_failed`: The rate of failed requests.
- `iterations`: The aggregate number of times the VUs executed the JS script (the default function).
- `iteration_duration`: The time it took to complete one full iteration, including time spent in setup and teardown.
- `vus`: Current number of active virtual users.
- `vus_max`: Max possible number of virtual users (VU resources are pre-allocated, ensuring performance will not be affected when scaling up the load level).

### Options

By default, if nothing is specified, K6 runs a script with only 1 VU and for 1 iteration only. Let's see how we can change that, run `k6 run --vus 5 --duration 15s script.js`:

```powershell
     data_received..................: 833 kB 52 kB/s
     data_sent......................: 8.6 kB 536 B/s
     http_req_blocked...............: avg=24.95ms  min=0s      med=0s       max=349.3ms  p(90)=0s       p(95)=349.3ms
     http_req_connecting............: avg=8.2ms    min=0s      med=0s       max=114.96ms p(90)=0s       p(95)=114.83ms
     http_req_duration..............: avg=106.37ms min=96.26ms med=103.61ms max=191.64ms p(90)=113.45ms p(95)=114.6ms
       { expected_response:true }...: avg=106.37ms min=96.26ms med=103.61ms max=191.64ms p(90)=113.45ms p(95)=114.6ms
     http_req_failed................: 0.00%  ✓ 0        ✗ 70
     http_req_receiving.............: avg=140.28µs min=0s      med=0s       max=1.62ms   p(90)=529.34µs p(95)=948.53µs
     http_req_sending...............: avg=10.29µs  min=0s      med=0s       max=508.6µs  p(90)=0s       p(95)=0s
     http_req_tls_handshaking.......: avg=10.1ms   min=0s      med=0s       max=141.91ms p(90)=0s       p(95)=141.07ms
     http_req_waiting...............: avg=106.21ms min=96.26ms med=103.61ms max=191.64ms p(90)=113.45ms p(95)=114.6ms
     http_reqs......................: 70     4.380964/s
     iteration_duration.............: avg=1.13s    min=1.09s   med=1.11s    max=1.45s    p(90)=1.12s    p(95)=1.45s
     iterations.....................: 70     4.380964/s
     vus............................: 5      min=5      max=5
     vus_max........................: 5      min=5      max=5
``` 

As we can notice, now we are using 5 VUs running for 15 seconds. As a result, we have a higher number of iterations. If we do not want to use command line parameters to specify VU and duration, create a `script-options.js` file with the following content:

```js
import http from 'k6/http';
import { sleep } from 'k6';
export const options = {
  vus: 5,
  duration: '15s',
};
export default function () {
  http.get('http://test.k6.io');
  sleep(1);
}
``` 

And run `k6 run script-options.js` to have the same results.

### Stages

In the real world, the number of users is not constant but tends to be variable. We can ramp the number of VUs up and down during the test by using the `options.stages` property. Create a `script-stages.js` file with the following content:

```js
import http from 'k6/http';
import { sleep } from 'k6';
export const options = {
  stages: [
    { duration: '10s', target: 20 },
    { duration: '15s', target: 10 },
    { duration: '5s', target: 0 },
  ],
};

export default function () {
  http.get('http://test.k6.io');
  sleep(1);
}
``` 

And run `k6 run script-stages.js`.

### Checks

> Checks are true/false criteria for your test runtime values. In practice, checks often evaluate whether the system under test responds with a certain value. A check may evaluate:
> 
> - That the system responds with a 200 status
> - That a response body contains certain text
> - That the response body is of a specified size.

Create a `script-checks.js` file with the following content:

```js
import http from 'k6/http';
import { check, sleep } from 'k6';
export const options = {
  vus: 5,
  duration: '15s',
};

export default function () {
  const res = http.get('http://test.k6.io');
  check(res, {
    'is status 200': (r) => r.status === 200,
	'duration was <=' : (r) => r.timings.duration <= 125
  });
  sleep(1);
}
``` 

And run `k6 run script-checks.js`. When a check fails, the script will continue executing successfully and will not return a 'failed' exit status.

```powershell
     ✓ is status 200
     ✗ duration was <=
      ↳  98% — ✓ 60 / ✗ 1

     checks.........................: 99.18% ✓ 121      ✗ 1
     data_received..................: 750 kB 46 kB/s
     data_sent......................: 14 kB  866 B/s
     http_req_blocked...............: avg=15.17ms  min=0s      med=0s       max=218.15ms p(90)=0s       p(95)=152.39ms
     http_req_connecting............: avg=8.77ms   min=0s      med=0s       max=109.53ms p(90)=0s       p(95)=107.94ms
     http_req_duration..............: avg=107.02ms min=96.37ms med=105.62ms max=214.04ms p(90)=111.15ms p(95)=114.71ms
       { expected_response:true }...: avg=107.02ms min=96.37ms med=105.62ms max=214.04ms p(90)=111.15ms p(95)=114.71ms
     http_req_failed................: 0.00%  ✓ 0        ✗ 122
     http_req_receiving.............: avg=958.9µs  min=0s      med=0s       max=102.44ms p(90)=533.94µs p(95)=1ms
     http_req_sending...............: avg=48.17µs  min=0s      med=0s       max=997.8µs  p(90)=0s       p(95)=383.49µs
     http_req_tls_handshaking.......: avg=4.62ms   min=0s      med=0s       max=117.69ms p(90)=0s       p(95)=0s
     http_req_waiting...............: avg=106.02ms min=95.89ms med=105.6ms  max=123.99ms p(90)=111.15ms p(95)=114.2ms
     http_reqs......................: 122    7.537943/s
     iteration_duration.............: avg=1.25s    min=1.2s    med=1.22s    max=1.59s    p(90)=1.24s    p(95)=1.59s
     iterations.....................: 61     3.768971/s
     vus............................: 1      min=1      max=5
     vus_max........................: 5      min=5      max=5
``` 

### Thresholds

> Thresholds are pass/fail criteria for your test metrics. If a test metric does not meet the expectation you defined, the threshold fails. Often, k6 users use thresholds to codify their SLOs.

This is particularly useful in specific contexts, such as integrating K6 into your CI pipelines or receiving alerts when scheduling your performance tests. Create a `script-thresholds.js` file with the following content:

```js
import http from 'k6/http';
import { check, sleep } from 'k6';
export const options = {
  vus: 5,
  duration: '15s',
  thresholds: {
    http_req_duration: ['p(95)<100'], // 95% of requests should be below 100ms
  },
};

export default function () {
  const res = http.get('http://test.k6.io');
  check(res, {
    'is status 200': (r) => r.status === 200
  });
  sleep(1);
}

``` 

And run `k6 run script-thresholds.js`. Checks are nice for codifying assertions, but unlike thresholds, checks do not affect the exit status of K6.

```
     ✓ is status 200

     checks.........................: 100.00% ✓ 65       ✗ 0
     data_received..................: 797 kB  49 kB/s
     data_sent......................: 15 kB   912 B/s
     http_req_blocked...............: avg=14.79ms  min=0s      med=0s       max=229.99ms p(90)=0s       p(95)=162.42ms
     http_req_connecting............: avg=8.32ms   min=0s      med=0s       max=118.95ms p(90)=0s       p(95)=105.57ms
   ✗ http_req_duration..............: avg=104.73ms min=97.11ms med=103.92ms max=132.31ms p(90)=109.49ms p(95)=110.77ms
       { expected_response:true }...: avg=104.73ms min=97.11ms med=103.92ms max=132.31ms p(90)=109.49ms p(95)=110.77ms
     http_req_failed................: 0.00%   ✓ 0        ✗ 130
     http_req_receiving.............: avg=132.96µs min=0s      med=0s       max=1.42ms   p(90)=655.45µs p(95)=999.95µs
     http_req_sending...............: avg=58.54µs  min=0s      med=0s       max=991.9µs  p(90)=10.25µs  p(95)=557.96µs
     http_req_tls_handshaking.......: avg=4.3ms    min=0s      med=0s       max=114.2ms  p(90)=0s       p(95)=0s
     http_req_waiting...............: avg=104.53ms min=95.68ms med=103.83ms max=132.31ms p(90)=109.28ms p(95)=110.32ms
     http_reqs......................: 130     7.995463/s
     iteration_duration.............: avg=1.24s    min=1.19s   med=1.21s    max=1.61s    p(90)=1.23s    p(95)=1.59s
     iterations.....................: 65      3.997732/s
     vus............................: 5       min=5      max=5
     vus_max........................: 5       min=5      max=5

ERRO[0016] some thresholds have failed
``` 

### K6 Cloud

Sign in [K6 Cloud](https://app.k6.io/account/login) and get your API token under `ACCOUNT SETTINGS`. Run the following command:

```powershell
k6 login cloud -t <YOUR API TOKEN>
``` 

And now, we are ready to execute our scripts in the cloud:

```powershell
 k6 cloud script-thresholds.js
``` 

Let's wait for a couple of minutes to see our test results:

![results.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1661038881153/_YgxxH0rw.PNG align="left")

Load Testing is something that everybody must do. K6 is a great tool to achieve this goal (check the official documentation [here](https://k6.io/docs/)). All the scripts are available [here](https://github.com/raulnq/k6-sandbox). Thanks, and happy coding.