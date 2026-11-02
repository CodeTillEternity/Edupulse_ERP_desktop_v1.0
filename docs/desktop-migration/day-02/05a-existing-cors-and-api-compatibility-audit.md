# EduPulse ERP — Existing CORS and API Compatibility Audit
## Work Package 05A — Baseline Architectural Audit of CORS, Security Filters & REST APIs

**Project:** EduPulse ERP — Desktop Transition Sprint (Week 00 / Day 02)
**Target Platform:** Windows 10 / Windows 11 Enterprise (Tauri v2 + Spring Boot 3.3.4)
**Author:** Antigravity AI Engineering Assistant (Architecture Baseline)
**Date:** 2 November 2026
**Document Status:** Approved Engineering Architecture Audit

---

## 1. Executive Summary & Audit Scope

This document provides an empirical technical audit of the existing Cross-Origin Resource Sharing (CORS) configuration, Spring Security filter chain integration, REST API endpoint topology, and client-side HTTP request pipelines in EduPulse ERP.

The objective is to establish the baseline compatibility posture of the Spring Boot 3.3.4 backend before introducing the Tauri v2 Windows desktop runtime, identifying architectural differences between standard web browser execution and Microsoft Edge WebView2 execution.

```
+----------------------------------------------------------------------------------------------------+
|                                    EMPIRICAL AUDIT SUMMARY SCORECARD                               |
+------------------------------------+-----------------------+---------------------------------------+
| Architectural Component            | Implementation State  | Compatibility Assessment              |
+------------------------------------+-----------------------+---------------------------------------+
| Backend CORS Configuration Bean    | CorsConfig.java       | OVERLY PERMISSIVE (Wildcard Pattern)  |
| Spring Security Filter Integration | SecurityConfig.java   | PROPERLY ORDERED (CorsFilter First)   |
| API Versioning Conventions         | Controllers (39)      | DUAL (/auth/* vs /api/v1/*)           |
| Frontend HTTP Client Engine        | Axios (api.ts)        | COUPLED TO BROWSER (Relative Fallback)|
| Preflight OPTIONS Handling         | SecurityConfig.java:45| PERMITTED (OPTIONS /** .permitAll())  |
| Error Response Standardization     | GlobalExceptionHandler| STANDARDIZED (RFC 7807-style JSON)    |
+------------------------------------+-----------------------+---------------------------------------+
```

---

## 2. Empirical Audit of Backend CORS Configuration

### 2.1 Active Configuration Source Code (`CorsConfig.java`)
The backend CORS configuration bean is located at `backend/src/main/java/com/erp/config/CorsConfig.java`:

```java
@Configuration
public class CorsConfig {

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowCredentials(true);
        config.setAllowedOriginPatterns(Arrays.asList("*"));
        config.setAllowedHeaders(Arrays.asList("Origin", "Content-Type", "Accept", "Authorization", "X-Requested-With", "Idempotency-Key", "idempotency-key", "*"));
        config.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
        config.setExposedHeaders(Arrays.asList("Authorization", "Content-Disposition"));

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```

### 2.2 Analysis of Active CORS Parameters
1. **Allowed Origins (`setAllowedOriginPatterns(Arrays.asList("*"))`):**
   - **Current State:** The backend currently accepts requests from **any origin** using wildcard matching.
   - **Tauri Compatibility:** Because `*` matches everything, requests from Tauri's custom Windows scheme (`https://tauri.localhost` or `http://tauri.localhost`) are accepted by default.
   - **Security Risk:** Combining `allowCredentials(true)` with broad wildcard origin matching is considered an anti-pattern in production enterprise systems. While modern browsers forbid `Access-Control-Allow-Origin: *` with credentials, Spring's `allowedOriginPatterns("*")` dynamically echoes back the incoming `Origin` header, effectively permitting any malicious website running in the user's browser to make authenticated cross-origin requests if cookies or ambient credentials are used.
2. **Allowed Methods:**
   - Explicitly permits `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, and `OPTIONS`. Fully satisfies all 259 planned ERP modules.
3. **Allowed Headers:**
   - Explicitly allows `Origin`, `Content-Type`, `Accept`, `Authorization`, `X-Requested-With`, `Idempotency-Key`, and wildcard `*`.
   - Critical for desktop: `Authorization` (Bearer JWT) and `Idempotency-Key` (network retry protection) are explicitly whitelisted.
4. **Exposed Headers:**
   - Exposes `Authorization` and `Content-Disposition`.
   - Essential for the frontend to read rotated token headers and parse file download names for student grade sheets and fee receipt PDFs.

---

## 3. Spring Security Filter Chain Integration

### 3.1 Filter Order & Execution Flow (`SecurityConfig.java`)
In `backend/src/main/java/com/erp/config/SecurityConfig.java`:

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .cors(org.springframework.security.config.Customizer.withDefaults())
        .csrf(AbstractHttpConfigurer::disable)
        .exceptionHandling(exception -> exception
            .authenticationEntryPoint(jwtAuthenticationEntryPoint)
            .accessDeniedHandler(customAccessDeniedHandler)
        )
        .authorizeHttpRequests(auth -> auth
            .requestMatchers(org.springframework.http.HttpMethod.OPTIONS, "/**").permitAll()
            .requestMatchers("/api/health", "/api/v1/health", "/actuator/health").permitAll()
            .requestMatchers("/auth/**").permitAll()
            .requestMatchers("/v3/api-docs/**", "/swagger-ui/**", "/swagger-ui.html").permitAll()
            .requestMatchers("/api/password/**").permitAll()
            .requestMatchers(org.springframework.http.HttpMethod.GET, "/api/v1/admissions/public/programs", "/api/v1/admissions/public/academic-years").permitAll()
            .requestMatchers(org.springframework.http.HttpMethod.POST, "/api/v1/admissions/public/register").permitAll()
            .requestMatchers("/api/v1/admissions/public/**").authenticated()
            .requestMatchers(org.springframework.http.HttpMethod.GET, "/api/v1/students/*/photo").permitAll()
            .anyRequest().authenticated()
        )
        .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);

    return http.build();
}
```

### 3.2 Key Architectural Verifications:
- **Preflight Handling:** `http.cors(Customizer.withDefaults())` automatically places Spring's `CorsFilter` at the very front of the security filter chain. Preflight `OPTIONS` requests are intercepted and resolved with `HTTP 204 No Content` / `HTTP 200 OK` before reaching `JwtAuthenticationFilter`.
- **Redundant Permissive Rule:** Line 45 (`.requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()`) provides a defensive fallback ensuring that unauthenticated `OPTIONS` requests never trigger `401 Unauthorized`.
- **Stateless Session Policy:** `SessionCreationPolicy.STATELESS` ensures Spring Security does not generate `JSESSIONID` cookies, confirming that both Web and Desktop rely exclusively on stateless Bearer tokens.

---

## 4. API Versioning & Endpoint Topology

The backend exhibits two distinct endpoint naming conventions across its 39 controllers:

```
+-----------------------------------+-----------------------+---------------------------------------------------+
| Endpoint Category                 | Base Route Pattern    | Controller Mapping Example                        |
+-----------------------------------+-----------------------+---------------------------------------------------+
| Authentication & Session          | `/auth/*`             | `AuthController.java` (/auth/login, /auth/refresh)|
| System Health & Monitoring        | `/api/v1/health`      | `HealthController.java` (/api/health, /actuator)  |
| Password Self-Service             | `/api/password/*`     | `PasswordResetController.java`                    |
| Academic & Student Domain         | `/api/v1/students/*`  | `StudentController.java`                          |
| Departmental & Academic Units     | `/api/v1/departments/*`| `DepartmentController.java`                       |
| Financial Ledger & Fee Collection | `/api/v1/finance/*`   | `FinanceController.java`                          |
| Institutional Multi-Tenancy       | `/api/v1/institutions`| `InstitutionController.java`                      |
+-----------------------------------+-----------------------+---------------------------------------------------+
```

*Architectural Implication:* The desktop client must communicate with both unversioned `/auth/*` endpoints and versioned `/api/v1/*` business logic endpoints under the same configured base URL.

---

## 5. Current Frontend Request Behavior & Desktop Compatibility Risks

### 5.1 Active Client Architecture (`frontend-v2/src/config/api.ts`)
```typescript
const envBaseUrl = import.meta.env.VITE_API_BASE_URL;
export const BASE_URL = envBaseUrl || (import.meta.env.DEV ? 'http://localhost:8081' : '/api');

export const apiClient = axios.create({
  baseURL: BASE_URL,
  timeout: 15000,
  headers: { 'Content-Type': 'application/json' },
});
```

### 5.2 Identified Compatibility Problems for Tauri WebView2

```
+-----------------------------------+-----------------------------------+-----------------------------------+
| Problem / Risk Area               | Web Browser Execution             | Tauri WebView2 Execution          |
+-----------------------------------+-----------------------------------+-----------------------------------+
| Origin Header Value               | `http://localhost:5173` or Domain | `https://tauri.localhost`         |
| Relative Base URL (`/api`)        | Resolves to host web server       | FAILS: Resolves to local files    |
| Protocol Scheme                   | Standard HTTP / HTTPS             | Windows custom URI scheme         |
| Direct IPC Bypass vs Axios        | Standard browser network stack    | Can use Axios or Tauri HTTP       |
| Dynamic Server Address Injection  | Hardcoded at build time           | Must be dynamically configurable  |
| Token Persistence Mechanism       | Plaintext `localStorage`          | Encrypted Windows DPAPI Vault     |
+-----------------------------------+-----------------------------------+-----------------------------------+
```

### Critical Risk Summary:
1. **The Relative URL Trap:** The fallback `'/api'` in `api.ts:6` breaks completely inside WebView2. The desktop edition must be supplied with an absolute URL (`https://<hostname>:<port>`).
2. **Wildcard Origin Hardening:** In production, `setAllowedOriginPatterns(Arrays.asList("*"))` must be tightened to recognize `https://tauri.localhost` alongside institutional web domains while avoiding unrestricted wildcards.

Proceed to **Work Package 05B: Tauri WebView2 CORS Architecture**.
