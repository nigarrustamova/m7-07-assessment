# Load Test Plan

This document outlines the load testing plan to verify that the `retail-recs-service` can handle the peak production load of 800 RPS within the 120 ms p95 latency budget.

## 1. Tool Selection
We will use **k6** (by Grafana) for load testing.
* **Why**: k6 is written in Go, runs compiled scripts, is extremely light on resource usage, and can easily generate 800+ RPS from a single test runner instance. Test scripts are written in simple JavaScript.

## 2. Test Environment
To ensure realistic results, the load test must run in a staging environment that matches the production configuration:
* **Service replicas**: 12 replicas of `c6i.xlarge` managed in Kubernetes.
* **Feature Store**: 1 primary + 1 replica Redis `cache.m6g.large` cluster pre-loaded with 10 million mock user feature records.
* **Load Balancer**: AWS Application Load Balancer (ALB).
* **Test Runner**: Run k6 on a separate `c6i.2xlarge` instance in the same VPC to eliminate internet network latency from the measurements.

## 3. Test Scenarios and Phases

The test is divided into four distinct phases to simulate realistic user traffic patterns:

```
RPS
 ^
1500|                                   /------- (Stress test to failure)
    |                                  /
 800|                      /----------/
    |                     /    Peak
 300|      /-------------/
    |     /    Sustained
   0+----+----+----+----+----+----+----+----> Time
    0   5m   25m  30m  40m  45m  50m
```

1. **Ramp-Up (Warm-up)**: 0 to 300 RPS over 5 minutes. Warm up the JIT compiler and connection pools.
2. **Sustained Load**: Constant 300 RPS (our estimated average load) for 20 minutes. Check for memory leaks or gradual latency degradation.
3. **Peak Load Spike**: Ramp from 300 to 800 RPS over 5 minutes, hold at 800 RPS for 10 minutes. This validates if we meet the p95 latency budget under peak capacity.
4. **Stress / Saturation Test**: Ramp from 800 to 1,500 RPS over 5 minutes. Find the breaking point of the system (where response times degrade or errors start occurring).

## 4. Sample k6 Test Script

Below is a mock-up of the JS script used by k6 to run the load test:

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { uuidv4 } from 'https://jslib.k6.io/k6-utils/1.4.0/index.js';

export const options = {
  stages: [
    { duration: '5m', target: 300 },  // Ramp-up to average
    { duration: '20m', target: 300 }, // Sustained average load
    { duration: '5m', target: 800 },  // Ramp-up to peak
    { duration: '10m', target: 800 }, // Peak load
    { duration: '5m', target: 0 },   // Cool down
  ],
  thresholds: {
    http_req_failed: ['rate<0.001'],         // Error rate must be < 0.1%
    http_req_duration: ['p(95)<120'],        // 95% of requests must be < 120ms
  },
};

// Generate list of mock user IDs
const users = Array.from({ length: 10000 }, (_, i) => `usr_${i}`);

export default function () {
  const randomUser = users[Math.floor(Math.random() * users.length)];
  const url = 'http://retail-recs-service.staging.local/recommend';
  
  const payload = JSON.stringify({
    user_id: randomUser,
    limit: 10,
    context: {
      device: 'mobile_ios',
      location: 'US-NY'
    }
  });

  const params = {
    headers: {
      'Content-Type': 'application/json',
      'X-Correlation-ID': uuidv4(),
    },
  };

  const res = http.post(url, payload, params);

  check(res, {
    'is status 200': (r) => r.status === 200,
    'correct user returned': (r) => r.json().user_id === randomUser,
  });

  // Short pause to simulate user click delay
  sleep(0.1);
}
```

## 5. Success Criteria
The load test is considered successful if and only if:
1. **p95 Latency**: Stays below 120 ms during the entire 800 RPS peak phase.
2. **Error Rate**: Total failed requests (5xx) represent less than 0.1% of all requests.
3. **Resource Health**:
   - CPU utilization of the serving pods does not exceed 75% on average.
   - Redis memory usage remains stable (no out-of-memory errors).
   - The connection pool does not exhaust.
