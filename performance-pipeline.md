# Deploying K6 Performance Tests in GitLab CI/CD

K6 is a powerful open-source load testing tool that integrates seamlessly with GitLab CI/CD pipelines. This guide will walk you through setting up automated performance testing using K6 in your GitLab projects.

## Prerequisites

Before getting started, ensure you have:
- A GitLab project with CI/CD enabled
- Basic understanding of GitLab CI/CD pipelines
- Familiarity with JavaScript (K6 tests are written in JavaScript)

## Setting Up K6 Tests

### Creating Your First K6 Test

Create a `tests/performance` directory in your project root and add a basic K6 test script:

```javascript
// tests/performance/load-test.js
import http from 'k6/http';
import { sleep, check } from 'k6';

export let options = {
  stages: [
    { duration: '2m', target: 100 }, // Ramp up to 100 users
    { duration: '5m', target: 100 }, // Stay at 100 users
    { duration: '2m', target: 0 },   // Ramp down to 0 users
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'], // 95% of requests must complete below 500ms
    http_req_failed: ['rate<0.1'],    // Error rate must be below 10%
  },
};

export default function () {
  const response = http.get('https://your-api-endpoint.com/health');
  
  check(response, {
    'status is 200': (r) => r.status === 200,
    'response time OK': (r) => r.timings.duration < 500,
  });
  
  sleep(1);
}
```

## GitLab CI/CD Configuration

### Basic Pipeline Setup

Create or update your `.gitlab-ci.yml` file to include K6 performance testing:

<img width="635" height="388" alt="Screenshot 2025-07-19 at 08 51 53" src="https://github.com/user-attachments/assets/f2257ede-06b8-4904-86b8-321ec716f25f" />


<img width="397" height="393" alt="Screenshot 2025-07-19 at 08 51 59" src="https://github.com/user-attachments/assets/974619bd-5f6b-4501-8bd2-fe1ecc0b6c34" />


## Integration with K6 Cloud
### Creation of tokens and variables

<img width="1303" height="369" alt="Screenshot 2025-07-19 at 09 00 24" src="https://github.com/user-attachments/assets/af7f0604-e6bc-4be7-bc45-da320e0f4263" />

Creation of Variable in GITLAB:K6_CLOUD_TOKEN AND K6_CLOUD_PROJECT_ID

<img width="1242" height="885" alt="Screenshot 2025-07-19 at 09 01 34" src="https://github.com/user-attachments/assets/8577310b-1f43-4fae-8a21-9dd68da9e76f" />


<img width="123" height="36" alt="Screenshot 2025-07-19 at 09 04 43" src="https://github.com/user-attachments/assets/70601695-0a77-4a9b-84ec-10dc843bf7f6" />


## Visualisation on K6 Cloud

Once the ci/cd job is done you can see your result



<img width="1165" height="895" alt="Screenshot 2025-07-19 at 08 49 38" src="https://github.com/user-attachments/assets/e322de52-dacc-4da9-bf02-0377ac607093" />


## Conclusion

Integrating K6 performance tests into your GitLab CI/CD pipeline provides automated performance validation for your applications. By following these patterns and best practices, you can catch performance regressions early and maintain application quality across different environments.

Remember to start with simple smoke tests and gradually build up to more comprehensive load and stress testing scenarios. Regular performance testing helps ensure your application can handle real-world traffic and provides confidence in your deployment process.


