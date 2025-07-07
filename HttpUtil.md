Great — let’s take the same logic, but **wrap it as a Spring `@Service`** so you can inject it anywhere in your Spring Boot app. I’ll also adjust it to use a single, reusable `CloseableHttpClient` bean, which is best practice.

Here’s the **clean, production-grade version**:

```java
package com.example.util;

import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.extern.slf4j.Slf4j;
import org.apache.hc.client5.http.classic.CloseableHttpClient;
import org.apache.hc.client5.http.classic.methods.HttpPost;
import org.apache.hc.core5.http.ContentType;
import org.apache.hc.core5.http.io.entity.StringEntity;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
@Slf4j
public class HttpUtilService {

    private final CloseableHttpClient httpClient;
    private final ObjectMapper objectMapper;

    @Autowired
    public HttpUtilService(CloseableHttpClient httpClient, ObjectMapper objectMapper) {
        this.httpClient = httpClient;
        this.objectMapper = objectMapper;
    }

    /**
     * Makes an HTTP POST call with the given URL, auth token, and request object,
     * then returns a wrapped custom response.
     *
     * @param url The target URL
     * @param authToken The Authorization token
     * @param requestPayload The request payload as a POJO
     * @param responseClass The type of custom response expected
     * @param <T> Type of the response
     * @return custom response object of type T
     * @throws Exception in case of HTTP or JSON errors
     */
    public <T> T post(
            String url,
            String authToken,
            Object requestPayload,
            Class<T> responseClass
    ) throws Exception {

        HttpPost httpPost = new HttpPost(url);

        // Serialize request payload
        String jsonPayload = objectMapper.writeValueAsString(requestPayload);
        StringEntity entity = new StringEntity(jsonPayload, ContentType.APPLICATION_JSON);
        httpPost.setEntity(entity);

        // Set authorization and content-type headers
        httpPost.addHeader("Authorization", "Bearer " + authToken);
        httpPost.addHeader("Content-Type", "application/json");

        // Log request
        log.info("HTTP POST Request to URL: {}", url);
        log.info("Request Payload: {}", jsonPayload);

        try (var response = httpClient.execute(httpPost)) {
            int statusCode = response.getCode();
            String responseBody = new String(response.getEntity().getContent().readAllBytes());

            // Log response
            log.info("HTTP Response Status: {}", statusCode);
            log.info("Response Body: {}", responseBody);

            return objectMapper.readValue(responseBody, responseClass);
        }
    }
}
```

---

## You’ll also need to configure the `CloseableHttpClient` bean

Create this config:

```java
package com.example.config;

import org.apache.hc.client5.http.classic.CloseableHttpClient;
import org.apache.hc.client5.http.impl.classic.HttpClients;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class HttpClientConfig {

    @Bean
    public CloseableHttpClient httpClient() {
        return HttpClients.createDefault();
    }
}
```

---

## Example usage in a controller

Say you have a controller:

```java
@RestController
@RequestMapping("/api/test")
public class TestController {

    private final HttpUtilService httpUtilService;

    @Autowired
    public TestController(HttpUtilService httpUtilService) {
        this.httpUtilService = httpUtilService;
    }

    @GetMapping
    public ResponseEntity<?> callRemote() throws Exception {
        MyRequest req = new MyRequest();
        req.setName("John");
        req.setAge(30);

        MyResponse resp = httpUtilService.post(
            "https://my-remote-service/api",
            "my-auth-token",
            req,
            MyResponse.class
        );

        return ResponseEntity.ok(resp);
    }
}
```

---

✅ **Summary**

* The class is reusable and injectable thanks to `@Service`.
* Uses Spring-managed, reusable Apache `HttpClient` bean.
* Logs both request and response with Lombok `@Slf4j`.
* Very easy to extend for other HTTP verbs (GET/PUT/DELETE) later.

---

**If you want**, I can help you expand this to:
✅ add GET, PUT, DELETE methods
✅ handle errors more gracefully with custom exceptions
✅ support retry / timeout settings

**Just let me know!**

---

## 👉 JDK 1.8 verion

---

```java
package com.example.util;

import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.extern.slf4j.Slf4j;
import org.apache.hc.client5.http.classic.CloseableHttpClient;
import org.apache.hc.client5.http.classic.methods.HttpPost;
import org.apache.hc.client5.http.classic.CloseableHttpResponse;
import org.apache.hc.core5.http.ContentType;
import org.apache.hc.core5.http.io.entity.StringEntity;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.io.InputStream;

@Service
@Slf4j
public class HttpUtilService {

    private final CloseableHttpClient httpClient;
    private final ObjectMapper objectMapper;

    @Autowired
    public HttpUtilService(CloseableHttpClient httpClient, ObjectMapper objectMapper) {
        this.httpClient = httpClient;
        this.objectMapper = objectMapper;
    }

    /**
     * Makes an HTTP POST call with the given URL, auth token, and request object,
     * then returns a wrapped custom response.
     *
     * @param url The target URL
     * @param authToken The Authorization token
     * @param requestPayload The request payload as a POJO
     * @param responseClass The type of custom response expected
     * @param <T> Type of the response
     * @return custom response object of type T
     * @throws Exception in case of HTTP or JSON errors
     */
    public <T> T post(
            String url,
            String authToken,
            Object requestPayload,
            Class<T> responseClass
    ) throws Exception {

        HttpPost httpPost = new HttpPost(url);

        // Serialize request payload
        String jsonPayload = objectMapper.writeValueAsString(requestPayload);
        StringEntity entity = new StringEntity(jsonPayload, ContentType.APPLICATION_JSON);
        httpPost.setEntity(entity);

        // Set authorization and content-type headers
        httpPost.addHeader("Authorization", "Bearer " + authToken);
        httpPost.addHeader("Content-Type", "application/json");

        // Log request
        log.info("HTTP POST Request to URL: {}", url);
        log.info("Request Payload: {}", jsonPayload);

        CloseableHttpResponse response = null;
        try {
            response = httpClient.execute(httpPost);

            int statusCode = response.getCode();
            InputStream inputStream = response.getEntity().getContent();
            byte[] bytes = new byte[inputStream.available()];
            inputStream.read(bytes);
            String responseBody = new String(bytes, "UTF-8");

            // Log response
            log.info("HTTP Response Status: {}", statusCode);
            log.info("Response Body: {}", responseBody);

            return objectMapper.readValue(responseBody, responseClass);
        } finally {
            if (response != null) {
                response.close();
            }
        }
    }
}
```
