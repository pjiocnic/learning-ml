✅ Absolutely — here’s how to **fully wire up** the `HttpUtilService` before you inject it into `ReportService`.

---

## 1️⃣ Spring config for Apache HttpClient

**(in `HttpClientConfig.java`)**

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
        // you can configure timeouts, retries, etc. here
        return HttpClients.createDefault();
    }
}
```

This registers an Apache `CloseableHttpClient` as a Spring bean.

---

## 2️⃣ `HttpUtilService` setup

**(in `HttpUtilService.java`)**

```java
package com.example.util;

import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.extern.slf4j.Slf4j;
import org.apache.hc.client5.http.classic.CloseableHttpClient;
import org.apache.hc.client5.http.classic.methods.HttpPost;
import org.apache.hc.client5.http.classic.methods.HttpGet;
import org.apache.hc.client5.http.classic.CloseableHttpResponse;
import org.apache.hc.core5.http.ContentType;
import org.apache.hc.core5.http.io.entity.StringEntity;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.io.FileOutputStream;
import java.io.InputStream;

import org.apache.commons.io.IOUtils;

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

    public <T> T post(String url, String authToken, Object requestPayload, Class<T> responseClass) throws Exception {
        HttpPost httpPost = new HttpPost(url);
        String jsonPayload = objectMapper.writeValueAsString(requestPayload);
        StringEntity entity = new StringEntity(jsonPayload, ContentType.APPLICATION_JSON);
        httpPost.setEntity(entity);

        httpPost.addHeader("Authorization", "Bearer " + authToken);
        httpPost.addHeader("Content-Type", "application/json");

        log.info("HTTP POST Request to URL: {}", url);
        log.info("Request Payload: {}", jsonPayload);

        CloseableHttpResponse response = null;
        try {
            response = httpClient.execute(httpPost);

            int statusCode = response.getCode();
            InputStream is = response.getEntity().getContent();
            String responseBody = IOUtils.toString(is, "UTF-8");

            log.info("HTTP Response Status: {}", statusCode);
            log.info("Response Body: {}", responseBody);

            return objectMapper.readValue(responseBody, responseClass);
        } finally {
            if (response != null) {
                response.close();
            }
        }
    }

    public byte[] getPdf(String url, String authToken) throws Exception {
        HttpGet httpGet = new HttpGet(url);
        httpGet.addHeader("Authorization", "Bearer " + authToken);
        httpGet.addHeader("Accept", "application/pdf");

        log.info("HTTP GET Request to URL: {}", url);

        CloseableHttpResponse response = null;
        try {
            response = httpClient.execute(httpGet);
            int statusCode = response.getCode();
            InputStream is = response.getEntity().getContent();
            byte[] pdfBytes = IOUtils.toByteArray(is);

            log.info("HTTP Response Status: {}", statusCode);
            log.info("PDF Content Length: {} bytes", pdfBytes.length);

            return pdfBytes;
        } finally {
            if (response != null) {
                response.close();
            }
        }
    }

    public void savePdfToDisk(String url, String authToken, String outputFilePath) throws Exception {
        HttpGet httpGet = new HttpGet(url);
        httpGet.addHeader("Authorization", "Bearer " + authToken);
        httpGet.addHeader("Accept", "application/pdf");

        log.info("HTTP GET Request to URL: {}", url);

        CloseableHttpResponse response = null;
        try {
            response = httpClient.execute(httpGet);
            int statusCode = response.getCode();
            InputStream is = response.getEntity().getContent();

            FileOutputStream fos = null;
            try {
                fos = new FileOutputStream(outputFilePath);
                byte[] buffer = new byte[4096];
                int read;
                while ((read = is.read(buffer)) != -1) {
                    fos.write(buffer, 0, read);
                }
            } finally {
                if (fos != null) {
                    fos.close();
                }
            }

            log.info("HTTP Response Status: {}", statusCode);
            log.info("PDF saved to: {}", outputFilePath);

        } finally {
            if (response != null) {
                response.close();
            }
        }
    }
}
```

✅ **Key points**

* Spring will **auto-wire** `CloseableHttpClient` from your config
* Spring will **auto-wire** `ObjectMapper` from Jackson’s default configuration
* `HttpUtilService` then is fully ready to inject wherever you want

---

## 3️⃣ Finally in your consumer service

**(ReportService.java)**

```java
package com.example.service;

import com.example.util.HttpUtilService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class ReportService {

    private final HttpUtilService httpUtilService;

    @Autowired
    public ReportService(HttpUtilService httpUtilService) {
        this.httpUtilService = httpUtilService;
    }

    public void downloadReport(String reportId) throws Exception {
        String url = "https://my-remote-service/api/reports/" + reportId;
        String authToken = "my-secret-token";

        String filePath = "/tmp/report_" + reportId + ".pdf";

        // use the save-to-disk variant
        httpUtilService.savePdfToDisk(url, authToken, filePath);

        // or use getPdf:
        /*
        byte[] pdfBytes = httpUtilService.getPdf(url, authToken);
        FileOutputStream fos = new FileOutputStream(filePath);
        fos.write(pdfBytes);
        fos.close();
        */
    }
}
```

---

✅ **Summary**

* `@Configuration` wires up the reusable `HttpClient`
* `HttpUtilService` is a Spring `@Service`
* any other service (like `ReportService`) simply injects `HttpUtilService`
* **ready-to-go** in JDK 1.8 Spring Boot with no manual new-instances needed

---

If you’d like, I can help you:
✅ add timeouts
✅ add connection pooling
✅ or wrap exceptions in custom errors

**Just let me know — happy to extend this for you! 🚀**
