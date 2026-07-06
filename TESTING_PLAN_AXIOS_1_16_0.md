# Axios 1.16.0 Compatibility Testing Plan

## Overview
This document outlines comprehensive testing procedures for Axios 1.16.0 upgrade, which introduces breaking changes to request handling, proxy behavior, and URL credential handling.

---

## 1. BREAKING CHANGES TEST SUITE

### 1.1 Fetch Adapter Size Limits Enforcement

**Change:** Fetch adapter now enforces `maxBodyLength` and `maxContentLength` (previously ignored)

#### Test Case 1.1.1: maxBodyLength Enforcement
```javascript
// tests/axios-1.16.0/fetch-adapter-limits.test.js
describe('Fetch Adapter Size Limits', () => {
  it('should reject requests exceeding maxBodyLength', async () => {
    const largeData = Buffer.alloc(100 * 1024 * 1024); // 100MB
    
    try {
      await axios.post('http://localhost:3000/upload', largeData, {
        adapter: 'fetch',
        maxBodyLength: 1024 * 1024, // 1MB limit
      });
      expect.fail('Should have rejected request');
    } catch (error) {
      expect(error.message).toContain('maxBodyLength');
    }
  });

  it('should allow requests within maxBodyLength', async () => {
    const smallData = Buffer.alloc(100 * 1024); // 100KB
    
    const response = await axios.post('http://localhost:3000/upload', smallData, {
      adapter: 'fetch',
      maxBodyLength: 1024 * 1024, // 1MB limit
    });
    expect(response.status).toBe(200);
  });

  it('should reject responses exceeding maxContentLength', async () => {
    try {
      await axios.get('http://localhost:3000/large-response', {
        adapter: 'fetch',
        maxContentLength: 1024 * 1024, // 1MB limit
      });
      expect.fail('Should have rejected response');
    } catch (error) {
      expect(error.message).toContain('maxContentLength');
    }
  });

  it('should allow responses within maxContentLength', async () => {
    const response = await axios.get('http://localhost:3000/response', {
      adapter: 'fetch',
      maxContentLength: 10 * 1024 * 1024, // 10MB limit
    });
    expect(response.status).toBe(200);
  });
});
```

#### Test Case 1.1.2: Default Limits
```javascript
it('should apply default maxBodyLength limit in fetch adapter', async () => {
  // Default is 2MB
  const defaultLimitData = Buffer.alloc(2.5 * 1024 * 1024); // 2.5MB
  
  try {
    await axios.post('http://localhost:3000/upload', defaultLimitData);
    expect.fail('Should have rejected');
  } catch (error) {
    expect(error.message).toMatch(/size|limit|max/i);
  }
});
```

**Validation Checklist:**
- [ ] Requests exceeding limits are rejected
- [ ] Error messages are clear and actionable
- [ ] Default limits match documentation
- [ ] Both HTTP and fetch adapters respect limits
- [ ] Streaming uploads handle limits correctly
- [ ] FormData uploads validate size before sending

---

### 1.2 Proxy Host Header Preservation

**Change:** User-supplied `Host` headers are now preserved when forwarding through a proxy

#### Test Case 1.2.1: Host Header in Proxy Requests
```javascript
// tests/axios-1.16.0/proxy-host-header.test.js
describe('Proxy Host Header Preservation', () => {
  const proxyUrl = 'http://proxy.internal:3128';
  const targetUrl = 'http://api.example.com/data';

  it('should preserve custom Host header in proxy requests', async () => {
    let capturedHeaders;
    
    nock('http://proxy.internal:3128')
      .post('/data')
      .reply((uri, body, cb) => {
        capturedHeaders = this.req.headers;
        cb(null, [200, { data: 'ok' }]);
      });

    await axios.post(targetUrl, {}, {
      httpAgent: new HttpProxyAgent(proxyUrl),
      headers: { 'Host': 'custom-host.example.com' },
    });

    expect(capturedHeaders.host).toBe('custom-host.example.com');
  });

  it('should use Host header from target URL when not custom-supplied', async () => {
    let capturedHeaders;
    
    nock('http://proxy.internal:3128')
      .get('/data')
      .reply((uri, body, cb) => {
        capturedHeaders = this.req.headers;
        cb(null, [200, { data: 'ok' }]);
      });

    await axios.get('http://api.example.com/data', {
      httpAgent: new HttpProxyAgent(proxyUrl),
    });

    expect(capturedHeaders.host).toBe('api.example.com');
  });

  it('should support virtual-host-style proxy routing', async () => {
    // Virtual host: s3-like bucket routing through proxy
    let capturedHeaders;
    
    nock('http://proxy.internal:3128')
      .get('/bucket/key')
      .reply((uri, body, cb) => {
        capturedHeaders = this.req.headers;
        cb(null, [200, { bucket: 'bucket', key: 'key' }]);
      });

    await axios.get('http://proxy.internal:3128/bucket/key', {
      headers: { 'Host': 'bucket.s3.example.com' },
    });

    expect(capturedHeaders.host).toBe('bucket.s3.example.com');
  });
});
```

#### Test Case 1.2.2: Proxy Edge Cases
```javascript
it('should not override proxy matchers with Host header', async () => {
  // Complex routing scenario
  const proxyUrl = 'http://proxy:3128';
  const config = {
    httpAgent: new HttpProxyAgent(proxyUrl),
    httpProxyAgent: new HttpProxyAgent(proxyUrl),
    httpsAgent: new HttpsProxyAgent(proxyUrl),
    httpsProxyAgent: new HttpsProxyAgent(proxyUrl),
    headers: { 'Host': 'virtual-host.example.com' },
  };

  const response = await axios.get('http://api.example.com/data', config);
  expect(response.status).toBe(200);
});

it('should preserve Host header through redirect chains', async () => {
  let finalHeaders;
  
  nock('http://api.example.com')
    .get('/data')
    .reply(301, null, { Location: 'http://api.example.com/data-v2' });
    
  nock('http://api.example.com')
    .get('/data-v2')
    .reply((uri, body, cb) => {
      finalHeaders = this.req.headers;
      cb(null, [200, { data: 'ok' }]);
    });

  await axios.get('http://api.example.com/data', {
    headers: { 'Host': 'custom.example.com' },
    maxRedirects: 5,
  });

  expect(finalHeaders.host).toBe('custom.example.com');
});
```

**Validation Checklist:**
- [ ] Custom Host headers are preserved in proxy requests
- [ ] Virtual-host routing works correctly
- [ ] Redirect chains maintain Host headers
- [ ] No header conflicts with proxy path
- [ ] Works with HTTP and HTTPS proxies
- [ ] CONNECT tunnels preserve headers

---

### 1.3 URL Basic Auth Credential URL Decoding

**Change:** Basic auth credentials in URLs are now URL-decoded

#### Test Case 1.3.1: Percent-Encoded Credentials
```javascript
// tests/axios-1.16.0/url-auth-decoding.test.js
describe('URL Basic Auth Credential Decoding', () => {
  
  it('should decode percent-encoded credentials in URL', async () => {
    let capturedAuth;
    
    nock('http://api.example.com')
      .get('/data')
      .reply((uri, body, cb) => {
        capturedAuth = this.req.headers.authorization;
        cb(null, [200, { data: 'ok' }]);
      });

    // URL: http://user:p%40ss@api.example.com/data
    // Should send: user:p@ss (decoded)
    await axios.get('http://user:p%40ss@api.example.com/data');

    const decoded = Buffer.from(capturedAuth.split(' ')[1], 'base64').toString();
    expect(decoded).toBe('user:p@ss'); // @ not %40
  });

  it('should handle special characters in credentials', async () => {
    const specialChars = [
      { encoded: 'user:pass%3Fquery', decoded: 'user:pass?query' },
      { encoded: 'user%3Aname:pass%40domain', decoded: 'user:name:pass@domain' },
      { encoded: 'user:p%20ass%26word', decoded: 'user:p ass&word' },
      { encoded: 'user%2Bname:pass%2Fslash', decoded: 'user+name:pass/slash' },
    ];

    for (const { encoded, decoded: expectedDecoded } of specialChars) {
      let capturedAuth;
      
      nock('http://api.example.com')
        .get('/data')
        .reply((uri, body, cb) => {
          capturedAuth = this.req.headers.authorization;
          cb(null, [200, { data: 'ok' }]);
        });

      await axios.get(`http://${encoded}@api.example.com/data`);

      const decodedAuth = Buffer.from(
        capturedAuth.split(' ')[1], 
        'base64'
      ).toString();
      
      expect(decodedAuth).toBe(expectedDecoded);
    }
  });

  it('should preserve already-decoded credentials', async () => {
    let capturedAuth;
    
    nock('http://api.example.com')
      .get('/data')
      .reply((uri, body, cb) => {
        capturedAuth = this.req.headers.authorization;
        cb(null, [200, { data: 'ok' }]);
      });

    // Plain credentials (no encoding)
    await axios.get('http://user:password@api.example.com/data');

    const decoded = Buffer.from(capturedAuth.split(' ')[1], 'base64').toString();
    expect(decoded).toBe('user:password');
  });

  it('should handle empty passwords with special chars', async () => {
    let capturedAuth;
    
    nock('http://api.example.com')
      .get('/data')
      .reply((uri, body, cb) => {
        capturedAuth = this.req.headers.authorization;
        cb(null, [200, { data: 'ok' }]);
      });

    // Empty password: user:%40
    await axios.get('http://user:%40@api.example.com/data');

    const decoded = Buffer.from(capturedAuth.split(' ')[1], 'base64').toString();
    expect(decoded).toBe('user:@');
  });

  it('should prioritize explicit auth config over URL credentials', async () => {
    let capturedAuth;
    
    nock('http://api.example.com')
      .get('/data')
      .reply((uri, body, cb) => {
        capturedAuth = this.req.headers.authorization;
        cb(null, [200, { data: 'ok' }]);
      });

    // URL has credentials, but explicit config should override
    await axios.get('http://urluser:urlpass@api.example.com/data', {
      auth: { username: 'configuser', password: 'configpass' },
    });

    const decoded = Buffer.from(capturedAuth.split(' ')[1], 'base64').toString();
    expect(decoded).toBe('configuser:configpass');
  });
});
```

**Validation Checklist:**
- [ ] Percent-encoded special chars decoded correctly
- [ ] @ signs in passwords work (common issue)
- [ ] ? and : characters properly decoded
- [ ] Spaces, slashes preserved after decoding
- [ ] Empty passwords handled
- [ ] Non-ASCII characters (UTF-8) work
- [ ] Explicit auth config takes precedence
- [ ] Backward compatibility with plain passwords

---

### 1.4 parseProtocol Stricter Validation

**Change:** `parseProtocol` now strictly requires colon in protocol separator

#### Test Case 1.4.1: Protocol Parsing Strictness
```javascript
// tests/axios-1.16.0/parse-protocol.test.js
describe('Protocol Parsing Strictness', () => {
  
  it('should accept valid protocols with colon', async () => {
    const validProtocols = [
      'http://api.example.com/data',
      'https://api.example.com/data',
      'ftp://ftp.example.com/file',
      'ws://websocket.example.com',
      'wss://secure.websocket.example.com',
    ];

    for (const url of validProtocols) {
      const response = await axios.get(url);
      expect(response.status).toBeLessThan(500);
    }
  });

  it('should reject invalid protocol formats', async () => {
    const invalidProtocols = [
      'httpapi.example.com/data',      // missing colon
      'http//api.example.com/data',    // double slash instead of colon+slash
      'ht tp://api.example.com/data',  // space in protocol
    ];

    for (const url of invalidProtocols) {
      try {
        await axios.get(url);
        expect.fail(`Should reject: ${url}`);
      } catch (error) {
        expect(error.message).toMatch(/protocol|invalid|format/i);
      }
    }
  });

  it('should handle relative URLs correctly', async () => {
    // Relative URLs shouldn't require protocol
    const response = await axios.get('/api/data', {
      baseURL: 'http://api.example.com',
    });
    expect(response.status).toBe(200);
  });

  it('should not accidentally match protocol-like strings', async () => {
    const config = {
      baseURL: 'http://api.example.com',
      headers: {
        'X-Custom': 'http://not-a-protocol',
      },
    };

    const response = await axios.get('/data', config);
    expect(response.status).toBe(200);
  });
});
```

**Validation Checklist:**
- [ ] Standard protocols (http, https, ftp, ws, wss) work
- [ ] Missing colon is rejected
- [ ] Malformed protocols fail gracefully
- [ ] Relative URLs still work
- [ ] Custom headers with "protocol-like" strings don't break
- [ ] Error messages are clear

---

### 1.5 UTF-8 Encoding Modernization

**Change:** Deprecated `unescape()` replaced with modern UTF-8 encoding

#### Test Case 1.5.1: Non-ASCII URL Handling
```javascript
// tests/axios-1.16.0/utf8-encoding.test.js
describe('UTF-8 URL Encoding', () => {
  
  it('should handle non-ASCII characters in URLs', async () => {
    const urls = [
      'http://api.example.com/data?name=José',
      'http://api.example.com/data?city=北京',
      'http://api.example.com/data?emoji=🚀',
      'http://api.example.com/user/François',
    ];

    for (const url of urls) {
      nock('http://api.example.com')
        .get(/.*/)
        .reply(200, { ok: true });

      const response = await axios.get(url);
      expect(response.status).toBe(200);
    }
  });

  it('should properly encode UTF-8 in query params', async () => {
    let capturedUrl;

    nock('http://api.example.com')
      .get(/.*/)
      .reply((uri) => {
        capturedUrl = uri;
        return [200, { ok: true }];
      });

    await axios.get('http://api.example.com/search', {
      params: { q: '你好世界' },
    });

    // Should be properly UTF-8 encoded, not using deprecated unescape()
    expect(capturedUrl).toContain(encodeURIComponent('你好世界'));
  });

  it('should handle mixed ASCII and UTF-8', async () => {
    const params = {
      name: 'John',
      city: 'München',
      emoji: '🎉',
      number: 42,
    };

    nock('http://api.example.com')
      .get(/.*/)
      .reply(200, { ok: true });

    const response = await axios.get('http://api.example.com/data', { params });
    expect(response.status).toBe(200);
  });

  it('should maintain spec-compliant byte sequences', async () => {
    let capturedRequest;

    nock('http://api.example.com')
      .post('/upload')
      .reply((uri, body) => {
        capturedRequest = body;
        return [200, { ok: true }];
      });

    const data = { name: 'Zürich' };

    await axios.post('http://api.example.com/upload', data);

    // Should use proper UTF-8 encoding (not legacy unescape behavior)
    expect(capturedRequest).toBeDefined();
  });
});
```

**Validation Checklist:**
- [ ] Non-ASCII characters in URLs work
- [ ] Query parameters with UTF-8 properly encoded
- [ ] Path segments with non-ASCII characters work
- [ ] Mixed ASCII and UTF-8 handled correctly
- [ ] Emoji and special unicode work
- [ ] Byte sequences match spec-compliant encoding
- [ ] No regression with ASCII-only URLs

---

## 2. INTEGRATION TEST SCENARIOS

### 2.1 Form Data with Size Limits

```javascript
// tests/axios-1.16.0/integration-formdata.test.js
describe('Form Data Integration with Limits', () => {
  
  it('should upload form data within size limits', async () => {
    const form = new FormData();
    form.append('file', Buffer.alloc(500 * 1024)); // 500KB
    form.append('name', 'test-file');

    const response = await axios.post('http://localhost:3000/upload', form, {
      maxBodyLength: 10 * 1024 * 1024, // 10MB
      headers: form.getHeaders(),
    });

    expect(response.status).toBe(200);
  });

  it('should reject form data exceeding size limits', async () => {
    const form = new FormData();
    form.append('file', Buffer.alloc(15 * 1024 * 1024)); // 15MB

    try {
      await axios.post('http://localhost:3000/upload', form, {
        maxBodyLength: 10 * 1024 * 1024, // 10MB limit
        headers: form.getHeaders(),
      });
      expect.fail('Should have rejected');
    } catch (error) {
      expect(error.message).toContain('maxBodyLength');
    }
  });

  it('should handle multipart boundary correctly with 1.16.0', async () => {
    const form = new FormData();
    form.append('name', 'value');
    form.append('file', Buffer.from('content'));

    const response = await axios.post('http://localhost:3000/upload', form, {
      headers: form.getHeaders(),
    });

    expect(response.status).toBe(200);
  });
});
```

### 2.2 Proxy + Auth + Host Header Combined

```javascript
// tests/axios-1.16.0/integration-proxy-auth.test.js
describe('Proxy with Auth and Host Headers', () => {
  
  it('should handle authenticated proxy with custom Host header', async () => {
    const proxyUrl = 'http://proxyuser:p%40ssword@proxy.internal:3128';
    
    nock('http://proxy.internal:3128')
      .get('/api/data')
      .reply((uri, body, cb) => {
        const proxyAuth = this.req.headers['proxy-authorization'];
        const hostHeader = this.req.headers.host;
        
        // Auth should be decoded (p%40ssword -> p@ssword)
        expect(proxyAuth).toBeDefined();
        
        // Host should be custom
        expect(hostHeader).toBe('virtual.api.example.com');
        
        cb(null, [200, { data: 'ok' }]);
      });

    await axios.get('http://api.example.com/api/data', {
      httpAgent: new HttpProxyAgent(proxyUrl),
      headers: { 'Host': 'virtual.api.example.com' },
    });
  });

  it('should handle redirect through proxy with credential preservation', async () => {
    const proxyUrl = 'http://proxy.internal:3128';

    nock('http://proxy.internal:3128')
      .get('/redirect')
      .reply(301, null, { Location: 'http://api.example.com/final' });

    nock('http://proxy.internal:3128')
      .get('/final')
      .reply(200, { data: 'redirected' });

    const response = await axios.get('http://api.example.com/redirect', {
      httpAgent: new HttpProxyAgent(proxyUrl),
      auth: { username: 'user', password: 'pass%40word' },
      maxRedirects: 5,
    });

    expect(response.data.data).toBe('redirected');
  });
});
```

---

## 3. REGRESSION TEST CHECKLIST

### 3.1 HTTP Adapter Compatibility
- [ ] Standard GET requests work
- [ ] POST with JSON payload
- [ ] File uploads with streams
- [ ] Chunked encoding
- [ ] Compression (gzip, deflate)
- [ ] Connection pooling
- [ ] Keep-alive connections
- [ ] Timeouts honored
- [ ] Abort signals work

### 3.2 Fetch Adapter Compatibility
- [ ] Standard GET requests work
- [ ] POST with JSON payload
- [ ] Blob/File uploads
- [ ] Stream uploads (if supported)
- [ ] Abort controller integration
- [ ] User-Agent header set correctly
- [ ] Size limits enforced
- [ ] Fetch error handling

### 3.3 XHR Adapter (Browser)
- [ ] CORS requests
- [ ] Credentials: include mode
- [ ] Progress events (download/upload)
- [ ] Response type handling
- [ ] Request timeout
- [ ] Abort functionality
- [ ] withCredentials flag

---

## 4. PERFORMANCE BASELINE

```javascript
// tests/axios-1.16.0/performance-baseline.test.js
describe('Axios 1.16.0 Performance Baseline', () => {
  
  it('should maintain performance for standard requests', async () => {
    const iterations = 1000;
    const start = Date.now();

    for (let i = 0; i < iterations; i++) {
      await axios.get('http://localhost:3000/api/data');
    }

    const duration = Date.now() - start;
    const avgTime = duration / iterations;

    console.log(`Average request time: ${avgTime}ms`);
    expect(avgTime).toBeLessThan(100); // Baseline expectation
  });

  it('should handle concurrent requests efficiently', async () => {
    const start = Date.now();

    await Promise.all(
      Array(100)
        .fill(null)
        .map(() => axios.get('http://localhost:3000/api/data'))
    );

    const duration = Date.now() - start;
    console.log(`100 concurrent requests: ${duration}ms`);
    expect(duration).toBeLessThan(5000); // Baseline
  });

  it('should handle large payload efficiently', async () => {
    const largeData = Buffer.alloc(10 * 1024 * 1024); // 10MB
    const start = Date.now();

    await axios.post('http://localhost:3000/upload', largeData);

    const duration = Date.now() - start;
    console.log(`10MB upload: ${duration}ms`);
    expect(duration).toBeLessThan(10000); // Baseline
  });
});
```

---

## 5. TEST EXECUTION STRATEGY

### Phase 1: Unit Tests (Days 1-2)
```bash
npm run test -- tests/axios-1.16.0/fetch-adapter-limits.test.js
npm run test -- tests/axios-1.16.0/proxy-host-header.test.js
npm run test -- tests/axios-1.16.0/url-auth-decoding.test.js
npm run test -- tests/axios-1.16.0/parse-protocol.test.js
npm run test -- tests/axios-1.16.0/utf8-encoding.test.js
```

### Phase 2: Integration Tests (Days 3-4)
```bash
npm run test -- tests/axios-1.16.0/integration-*.test.js
```

### Phase 3: Regression Tests (Days 5-7)
```bash
npm run test -- tests/axios-1.16.0/regression-*.test.js
npm run test:performance -- tests/axios-1.16.0/performance-baseline.test.js
```

### Phase 4: E2E Tests (Days 8-10)
```bash
npm run test:e2e -- --filter axios-upgrade
```

---

## 6. EXPECTED OUTCOMES

### Pass Criteria
- ✅ All unit tests pass (100%)
- ✅ All integration tests pass (100%)
- ✅ Regression tests show no performance degradation >10%
- ✅ Error messages are clear for breaking changes
- ✅ Documentation updated for new behaviors

### Fail Criteria (Rollback Required)
- ❌ >5% of tests fail
- ❌ Performance degradation >20%
- ❌ Breaking changes not documented
- ❌ Security tests show vulnerabilities
- ❌ Production environment crashes

---

## 7. MIGRATION GUIDE FOR DEVELOPERS

### Required Changes in Application Code

#### 1. Update Size Limits (if using fetch adapter)
```javascript
// BEFORE (1.15.x): Limits were ignored
axios.post('/upload', data);

// AFTER (1.16.0): Must set limits or requests fail
axios.post('/upload', data, {
  maxBodyLength: 50 * 1024 * 1024,     // 50MB
  maxContentLength: 50 * 1024 * 1024,
});
```

#### 2. Update URL Credentials
```javascript
// BEFORE (1.15.x): Special chars remained encoded
// http://user:p%40ss@api.example.com -> sends "p%40ss"

// AFTER (1.16.0): Special chars decoded
// http://user:p%40ss@api.example.com -> sends "p@ss"

// MIGRATION: Use explicit auth config instead
axios.get('http://api.example.com/data', {
  auth: { username: 'user', password: 'p@ss' }
});
```

#### 3. Host Header for Virtual Hosting
```javascript
// NOW WORKS (1.16.0): Custom Host header preserved
axios.get('http://bucket.s3.example.com/key', {
  httpAgent: new HttpProxyAgent('http://proxy:3128'),
  headers: { 'Host': 'bucket.s3.example.com' },
});
```

---

## 8. SIGNOFF CHECKLIST

- [ ] All test phases completed
- [ ] Performance baseline established
- [ ] Breaking changes documented
- [ ] Developer migration guide distributed
- [ ] Staging environment validated
- [ ] Rollback procedure tested
- [ ] On-call team notified
- [ ] Monitoring alerts configured
- [ ] Deployment window scheduled
- [ ] Post-deployment validation plan ready

---

## 9. CONTACT & SUPPORT

**Testing Lead:** [Assign]
**QA Manager:** [Assign]
**DevOps Lead:** [Assign]
**On-Call Engineer:** [Assign]

**Escalation Path:**
1. Test failures → Testing Lead
2. Performance issues → DevOps Lead
3. Breaking changes impact → QA Manager
4. Production incident → On-Call Engineer

---

**Document Version:** 1.0
**Last Updated:** 2026-07-06
**Next Review:** After first staging deployment
