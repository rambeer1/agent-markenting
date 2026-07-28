ais: backend: ais-auth-service: src\main\java\com\ais\auth: config
SecurityConfig.java
package com.ais.auth.config;

import com.ais.auth.security.JwtAuthenticationFilter;
import com.ais.auth.security.RestAccessDeniedHandler;
import com.ais.auth.security.RestAuthenticationEntryPoint;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthenticationFilter;
    private final RestAuthenticationEntryPoint restAuthenticationEntryPoint;
    private final RestAccessDeniedHandler restAccessDeniedHandler;

    public SecurityConfig(JwtAuthenticationFilter jwtAuthenticationFilter,
                           RestAuthenticationEntryPoint restAuthenticationEntryPoint,
                           RestAccessDeniedHandler restAccessDeniedHandler) {
        this.jwtAuthenticationFilter = jwtAuthenticationFilter;
        this.restAuthenticationEntryPoint = restAuthenticationEntryPoint;
        this.restAccessDeniedHandler = restAccessDeniedHandler;
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(10);
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
                .csrf(AbstractHttpConfigurer::disable)
                // CORS is handled centrally by ais-gateway-service (see HLD.md Section 5); this
                // service is never called directly by browsers, so it must NOT also add its own
                // Access-Control-Allow-Origin header - doing so duplicates the header through the
                // gateway proxy and browsers reject responses with a duplicated CORS header.
                .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .exceptionHandling(eh -> eh
                        .authenticationEntryPoint(restAuthenticationEntryPoint)
                        .accessDeniedHandler(restAccessDeniedHandler))
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers(
                                "/api/v1/auth/**",
                                "/swagger-ui/**", "/v3/api-docs/**",
                                "/actuator/health")
                        .permitAll()
                        .anyRequest().authenticated())
                .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
----------------------------------------------------
ais: backend: ais-auth-service: src\main\java\com\ais\auth: controller
AuthController.java
package com.ais.auth.controller;

import com.ais.auth.dto.LoginRequest;
import com.ais.auth.dto.LoginResponse;
import com.ais.auth.dto.RefreshRequest;
import com.ais.auth.service.AuthService;
import com.ais.common.api.ApiResponse;
import io.swagger.v3.oas.annotations.Operation;
import jakarta.validation.Valid;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/auth")
public class AuthController {

    private final AuthService authService;

    public AuthController(AuthService authService) {
        this.authService = authService;
    }

    @Operation(summary = "Authenticate with username/password and receive access + refresh JWTs")
    @PostMapping("/login")
    public ResponseEntity<ApiResponse<LoginResponse>> login(@Valid @RequestBody LoginRequest request) {
        LoginResponse response = authService.login(request);
        return ResponseEntity.ok(ApiResponse.success(response, "Login successful"));
    }

    @Operation(summary = "Exchange a valid refresh token for a new access + refresh token pair")
    @PostMapping("/refresh")
    public ResponseEntity<ApiResponse<LoginResponse>> refresh(@Valid @RequestBody RefreshRequest request) {
        LoginResponse response = authService.refresh(request);
        return ResponseEntity.ok(ApiResponse.success(response, "Token refreshed"));
    }

    @Operation(summary = "Logout - JWTs are stateless so this is a client-side token discard signal")
    @PostMapping("/logout")
    public ResponseEntity<ApiResponse<Void>> logout() {
        return ResponseEntity.ok(ApiResponse.success(null, "Logged out successfully"));
    }
}

----------------------------------------------------
ais: backend: ais-auth-service: src\main\java\com\ais\auth: dto
package com.ais.auth.dto;

import jakarta.validation.constraints.NotBlank;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class LoginRequest {

    @NotBlank(message = "Username is required")
    private String username;

    @NotBlank(message = "Password is required")
    private String password;
}

----------------------------------------------------
ais: backend: ais-auth-service: src\main\java\com\ais\auth: dto
package com.ais.auth.dto;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class LoginResponse {

    private String accessToken;
    private String refreshToken;
    private String username;
    private String role;
    private long expiresInMs;
}


----------------------------------------------------
ais: backend: ais-auth-service: src\main\java\com\ais\auth: dto
package com.ais.auth.dto;

import jakarta.validation.constraints.NotBlank;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class RefreshRequest {

    @NotBlank(message = "Refresh token is required")
    private String refreshToken;
}


----------------------------------------------------
ais: backend: ais-auth-service: src\main\java\com\ais\auth: entity
package com.ais.auth.entity;

public enum Role {
    ADMIN,
    EDITOR
}


----------------------------------------------------
ais: backend: ais-auth-service: src\main\java\com\ais\auth: entity
package com.ais.auth.entity;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

@Entity
@Table(name = "users")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "username", nullable = false, unique = true)
    private String username;

    @Column(name = "password", nullable = false)
    private String password;

    @Enumerated(EnumType.STRING)
    @Column(name = "role", nullable = false)
    private Role role;

    @Column(name = "status")
    @Builder.Default
    private String status = "ACTIVE";
}

----------------------------------------------------
ais: backend: ais-auth-service: src\main\java\com\ais\auth: exception
package com.ais.auth.exception;

import com.ais.common.api.ApiErrorResponse;
import io.jsonwebtoken.JwtException;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.List;

@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(InvalidCredentialsException.class)
    public ResponseEntity<ApiErrorResponse> handleInvalidCredentials(InvalidCredentialsException ex) {
        log.warn("Authentication failed: {}", ex.getMessage());
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
                .body(ApiErrorResponse.of("INVALID_CREDENTIALS", "Invalid username or password", List.of()));
    }

    @ExceptionHandler(JwtException.class)
    public ResponseEntity<ApiErrorResponse> handleJwtException(JwtException ex) {
        log.warn("JWT validation failed: {}", ex.getMessage());
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
                .body(ApiErrorResponse.of("INVALID_TOKEN", "Invalid or expired token", List.of()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        List<String> details = ex.getBindingResult().getFieldErrors().stream()
                .map(fe -> fe.getField() + ": " + fe.getDefaultMessage())
                .toList();
        return ResponseEntity.badRequest()
                .body(ApiErrorResponse.of("VALIDATION_FAILED", "Request validation failed", details));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiErrorResponse> handleGeneric(Exception ex) {
        log.error("Unhandled exception in ais-auth-service", ex);
        return ResponseEntity.internalServerError()
                .body(ApiErrorResponse.of("INTERNAL_ERROR", "An unexpected error occurred", List.of()));
    }
}

----------------------------------------------------
ais: backend: ais-auth-service: src\main\java\com\ais\auth: exception
package com.ais.auth.exception;

/**
 * Thrown when username/password do not match, or the account is not ACTIVE.
 * Mapped to HTTP 401 by GlobalExceptionHandler. The message is intentionally generic
 * (never reveals which of username/password was wrong) per SECURITY_DESIGN.md.
 */
public class InvalidCredentialsException extends RuntimeException {

    public InvalidCredentialsException(String message) {
        super(message);
    }
}

----------------------------------------------------
ais: backend: ais-auth-service: src\main\java\com\ais\auth: repository
package com.ais.auth.repository;

import com.ais.auth.entity.User;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.Optional;

public interface UserRepository extends JpaRepository<User, Long> {

    Optional<User> findByUsername(String username);
}

----------------------------------------------------
ais: backend: ais-auth-service: src\main\java\com\ais\auth: security
package com.ais.auth.security;

import com.ais.auth.repository.UserRepository;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Service;

@Service
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    public CustomUserDetailsService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        return userRepository.findByUsername(username)
                .map(UserPrincipal::new)
                .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));
    }
}

----------------------------------------------------
ais: backend: ais-auth-service: src\main\java\com\ais\auth: security
package com.ais.auth.security;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import io.jsonwebtoken.Claims;
import io.jsonwebtoken.JwtException;
import org.springframework.lang.NonNull;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.List;

/**
 * Extracts and validates the Authorization: Bearer <token> header as an ACCESS token,
 * populating the SecurityContext with the roles carried in the "roles" claim.
 */
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenProvider jwtTokenProvider;

    public JwtAuthenticationFilter(JwtTokenProvider jwtTokenProvider) {
        this.jwtTokenProvider = jwtTokenProvider;
    }

    @Override
    protected void doFilterInternal(@NonNull HttpServletRequest request,
                                     @NonNull HttpServletResponse response,
                                     @NonNull FilterChain filterChain) throws ServletException, IOException {
        String header = request.getHeader("Authorization");
        if (header != null && header.startsWith("Bearer ")) {
            String token = header.substring(7);
            try {
                Claims claims = jwtTokenProvider.validateAndParse(token, "access");
                String username = claims.getSubject();
                @SuppressWarnings("unchecked")
                List<String> roles = (List<String>) claims.get("roles", List.class);
                List<GrantedAuthority> authorities = roles == null ? List.of() : roles.stream()
                        .map(SimpleGrantedAuthority::new)
                        .map(GrantedAuthority.class::cast)
                        .toList();

                UsernamePasswordAuthenticationToken authentication =
                        new UsernamePasswordAuthenticationToken(username, null, authorities);
                authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(authentication);
            } catch (JwtException | IllegalArgumentException ex) {
                SecurityContextHolder.clearContext();
            }
        }
        filterChain.doFilter(request, response);
    }
}

----------------------------------------------------
ais: backend: ais-auth-service: src\main\java\com\ais\auth: security
package com.ais.auth.security;

import com.ais.auth.entity.User;
import io.jsonwebtoken.Claims;
import io.jsonwebtoken.JwtException;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import javax.crypto.SecretKey;
import java.nio.charset.StandardCharsets;
import java.util.Date;
import java.util.List;

/**
 * Issues and validates access/refresh JWTs. Only ais-auth-service issues tokens;
 * downstream services only verify them via com.ais.common.security.JwtTokenValidator.
 */
@Component
public class JwtTokenProvider {

    private static final String CLAIM_ROLES = "roles";
    private static final String CLAIM_TOKEN_TYPE = "type";
    private static final String TOKEN_TYPE_ACCESS = "access";
    private static final String TOKEN_TYPE_REFRESH = "refresh";

    private final SecretKey signingKey;
    private final long accessTokenExpirationMs;
    private final long refreshTokenExpirationMs;

    public JwtTokenProvider(
            @Value("${app.jwt.secret}") String secret,
            @Value("${app.jwt.access-token-expiration-ms}") long accessTokenExpirationMs,
            @Value("${app.jwt.refresh-token-expiration-ms}") long refreshTokenExpirationMs) {
        this.signingKey = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
        this.accessTokenExpirationMs = accessTokenExpirationMs;
        this.refreshTokenExpirationMs = refreshTokenExpirationMs;
    }

    public String generateAccessToken(User user) {
        return buildToken(user, TOKEN_TYPE_ACCESS, accessTokenExpirationMs);
    }

    public String generateRefreshToken(User user) {
        return buildToken(user, TOKEN_TYPE_REFRESH, refreshTokenExpirationMs);
    }

    public long getAccessTokenExpirationMs() {
        return accessTokenExpirationMs;
    }

    private String buildToken(User user, String tokenType, long expirationMs) {
        Date now = new Date();
        Date expiry = new Date(now.getTime() + expirationMs);
        return Jwts.builder()
                .subject(user.getUsername())
                .claim(CLAIM_ROLES, List.of("ROLE_" + user.getRole().name()))
                .claim(CLAIM_TOKEN_TYPE, tokenType)
                .issuedAt(now)
                .expiration(expiry)
                .signWith(signingKey)
                .compact();
    }

    /**
     * Validates the token signature/expiry and asserts it is of the expected type
     * (access vs refresh), preventing a refresh token from being used as an access token.
     */
    public Claims validateAndParse(String token, String expectedTokenType) {
        Claims claims = Jwts.parser()
                .verifyWith(signingKey)
                .build()
                .parseSignedClaims(token)
                .getPayload();

        String actualType = claims.get(CLAIM_TOKEN_TYPE, String.class);
        if (!expectedTokenType.equals(actualType)) {
            throw new JwtException("Unexpected token type: expected " + expectedTokenType + " but was " + actualType);
        }
        return claims;
    }
}


----------------------------------------------------
ais: backend: ais-auth-service: src\main\java\com\ais\auth: security
package com.ais.auth.security;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.ais.common.api.ApiErrorResponse;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.http.MediaType;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.security.web.access.AccessDeniedHandler;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.util.List;

/**
 * Writes the standard ApiErrorResponse envelope for authenticated-but-forbidden
 * requests (403). Kept as a real Spring bean (not mocked) in @WebMvcTest slices —
 * see ais_website_instructions.md Section 15A pitfall #1.
 */
@Component
public class RestAccessDeniedHandler implements AccessDeniedHandler {

    private final ObjectMapper objectMapper = new ObjectMapper().findAndRegisterModules();

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, AccessDeniedException accessDeniedException)
            throws IOException {
        response.setStatus(HttpServletResponse.SC_FORBIDDEN);
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        ApiErrorResponse body = ApiErrorResponse.of("FORBIDDEN", "You do not have permission to perform this action", List.of());
        response.getWriter().write(objectMapper.writeValueAsString(body));
    }
}

----------------------------------------------------
ais: backend: ais-auth-service: src\main\java\com\ais\auth: security
package com.ais.auth.security;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.ais.common.api.ApiErrorResponse;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.http.MediaType;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.web.AuthenticationEntryPoint;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.util.List;

/**
 * Writes the standard ApiErrorResponse envelope for unauthenticated requests to
 * protected URLs (401). Kept as a real Spring bean (not mocked) in @WebMvcTest slices —
 * see ais_website_instructions.md Section 15A pitfall #1.
 */
@Component
public class RestAuthenticationEntryPoint implements AuthenticationEntryPoint {

    private final ObjectMapper objectMapper = new ObjectMapper().findAndRegisterModules();

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException authException)
            throws IOException {
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        ApiErrorResponse body = ApiErrorResponse.of("UNAUTHORIZED", "Authentication is required to access this resource", List.of());
        response.getWriter().write(objectMapper.writeValueAsString(body));
    }
}

----------------------------------------------------
ais: backend: ais-auth-service: src\main\java\com\ais\auth: security
package com.ais.auth.security;

import com.ais.auth.entity.User;
import lombok.Getter;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

import java.util.Collection;
import java.util.List;

@Getter
public class UserPrincipal implements UserDetails {

    private final User user;

    public UserPrincipal(User user) {
        this.user = user;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return List.of(new SimpleGrantedAuthority("ROLE_" + user.getRole().name()));
    }

    @Override
    public String getPassword() {
        return user.getPassword();
    }

    @Override
    public String getUsername() {
        return user.getUsername();
    }

    @Override
    public boolean isEnabled() {
        return "ACTIVE".equalsIgnoreCase(user.getStatus());
    }
}

----------------------------------------------------
ais: backend: ais-auth-service: src\main\java\com\ais\auth: service
package com.ais.auth.service;

import com.ais.auth.dto.LoginRequest;
import com.ais.auth.dto.LoginResponse;
import com.ais.auth.dto.RefreshRequest;

public interface AuthService {

    LoginResponse login(LoginRequest request);

    LoginResponse refresh(RefreshRequest request);
}

----------------------------------------------------
ais: backend: ais-auth-service: src\main\java\com\ais\auth: service
package com.ais.auth;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class AisAuthServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(AisAuthServiceApplication.class, args);
    }
}

----------------------------------------------------
ais: backend: ais-auth-service: src\main\java\com\ais\auth: service: impl
package com.ais.auth.service.impl;

import com.ais.auth.dto.LoginRequest;
import com.ais.auth.dto.LoginResponse;
import com.ais.auth.dto.RefreshRequest;
import com.ais.auth.entity.User;
import com.ais.auth.exception.InvalidCredentialsException;
import com.ais.auth.repository.UserRepository;
import com.ais.auth.security.JwtTokenProvider;
import com.ais.auth.service.AuthService;
import io.jsonwebtoken.Claims;
import io.jsonwebtoken.JwtException;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;

@Service
public class AuthServiceImpl implements AuthService {

    private static final Logger log = LoggerFactory.getLogger(AuthServiceImpl.class);

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final JwtTokenProvider jwtTokenProvider;

    public AuthServiceImpl(UserRepository userRepository, PasswordEncoder passwordEncoder, JwtTokenProvider jwtTokenProvider) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
        this.jwtTokenProvider = jwtTokenProvider;
    }

    @Override
    public LoginResponse login(LoginRequest request) {
        User user = userRepository.findByUsername(request.getUsername())
                .orElseThrow(() -> new InvalidCredentialsException("Invalid username or password"));

        if (!"ACTIVE".equalsIgnoreCase(user.getStatus())) {
            throw new InvalidCredentialsException("Invalid username or password");
        }
        if (!passwordEncoder.matches(request.getPassword(), user.getPassword())) {
            throw new InvalidCredentialsException("Invalid username or password");
        }

        log.info("User '{}' authenticated successfully", user.getUsername());
        return buildLoginResponse(user);
    }

    @Override
    public LoginResponse refresh(RefreshRequest request) {
        Claims claims;
        try {
            claims = jwtTokenProvider.validateAndParse(request.getRefreshToken(), "refresh");
        } catch (JwtException | IllegalArgumentException ex) {
            throw new InvalidCredentialsException("Invalid or expired refresh token");
        }

        User user = userRepository.findByUsername(claims.getSubject())
                .orElseThrow(() -> new InvalidCredentialsException("Invalid or expired refresh token"));

        return buildLoginResponse(user);
    }

    private LoginResponse buildLoginResponse(User user) {
        return LoginResponse.builder()
                .accessToken(jwtTokenProvider.generateAccessToken(user))
                .refreshToken(jwtTokenProvider.generateRefreshToken(user))
                .username(user.getUsername())
                .role("ROLE_" + user.getRole().name())
                .expiresInMs(jwtTokenProvider.getAccessTokenExpirationMs())
                .build();
    }
}


----------------------------------------------------
ais: backend: ais-auth-service: src\main\java\com\ais\auth:
package com.ais.auth;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class AisAuthServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(AisAuthServiceApplication.class, args);
    }
}

----------------------------------------------------
ais: backend: ais-auth-service: src\main\: resources: db:migration
CREATE TABLE IF NOT EXISTS users (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    username VARCHAR(255) NOT NULL UNIQUE,
    password VARCHAR(500) NOT NULL,
    role VARCHAR(50) NOT NULL,
    status VARCHAR(20) DEFAULT 'ACTIVE'
);

CREATE INDEX IF NOT EXISTS idx_users_username ON users (username);


----------------------------------------------------
ais: backend: ais-auth-service: src\main\: resources: db:migration
-- Passwords are BCrypt hashes (strength 10) of: admin123 / editor123
-- Generated via bcryptjs (Node) for correctness; Spring Security's BCryptPasswordEncoder
-- accepts both the $2a$ and $2b$ hash prefixes.
INSERT INTO users (username, password, role, status) VALUES
('admin', '$2b$10$RO6fbxFi5y0VW6XjeVO2X.edAx3TSZFxU2hterHektIJvqpQ22wx6', 'ADMIN', 'ACTIVE'),
('editor', '$2b$10$612WWFy7jKg5hnX4hSZ5TuMsm3wFFvPt4DVPqagz1HxOOjVPQsMem', 'EDITOR', 'ACTIVE');


----------------------------------------------------
ais: backend: ais-auth-service: src\main\: resources: 
spring:
  datasource:
    url: jdbc:postgresql://postgres:5432/ais_db
    username: ${DB_USERNAME:ais_auth_user}
    password: ${DB_PASSWORD:ais_auth_pass}


----------------------------------------------------
ais: backend: ais-auth-service: src\main\: resources: 
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  jpa:
    show-sql: false

feature:
  email-notifications-enabled: true
  analytics-enabled: true

logging:
  level:
    com.ais: INFO

----------------------------------------------------
ais: backend: ais-auth-service: src\main\: resources: 
# Local, no-install-required profile: runs against an in-memory H2 database
# (PostgreSQL compatibility mode) instead of a real PostgreSQL server/Docker.
# Start with: mvn spring-boot:run -Dspring-boot.run.profiles=h2
spring:
  datasource:
    url: jdbc:h2:mem:ais_auth;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
    driver-class-name: org.h2.Driver
    username: sa
    password:
  jpa:
    show-sql: true
  flyway:
    schemas: PUBLIC
    default-schema: PUBLIC
    create-schemas: false
  h2:
    console:
      enabled: true
      path: /h2-console

logging:
  level:
    com.ais: DEBUG

----------------------------------------------------
ais: backend: ais-auth-service: src\main\: resources: 
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/ais_db
    username: ais_auth_user
    password: ais_auth_pass
  jpa:
    show-sql: true

logging:
  level:
    com.ais: DEBUG

----------------------------------------------------
ais: backend: ais-auth-service: src\main\: resources: 
server:
  port: 8081

spring:
  application:
    name: ais-auth-service
  jpa:
    hibernate:
      ddl-auto: validate
    open-in-view: false
  flyway:
    enabled: true
    schemas: auth_schema
    default-schema: auth_schema
    create-schemas: true
    locations: classpath:db/migration

app:
  jwt:
    secret: ${JWT_SECRET:local-dev-only-change-me-must-be-32-bytes-minimum}
    access-token-expiration-ms: 3600000
    refresh-token-expiration-ms: 604800000
  cors:
    allowed-origins: ${CORS_ALLOWED_ORIGINS:http://localhost:4200}

management:
  endpoints:
    web:
      exposure:
        include: health,info

springdoc:
  swagger-ui:
    path: /swagger-ui.html

----------------------------------------------------
ais: backend: ais-auth-service: src\main\: resources: 
<configuration>
    <springProfile name="local | default | h2">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{ISO8601} [%thread] %-5level %logger{36} [%X{requestId}] - %msg%n</pattern>
            </encoder>
        </appender>
        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>

    <springProfile name="docker,gcp">
        <appender name="JSON_CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
        </appender>
        <root level="INFO">
            <appender-ref ref="JSON_CONSOLE"/>
        </root>
    </springProfile>
</configuration>

----------------------------------------------------
ais: backend: ais-auth-service: 
# Multi-stage build. Build context is the backend/ reactor root (see docker-compose.yml).
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /workspace
COPY pom.xml .
COPY ais-common ais-common
COPY ais-auth-service ais-auth-service
RUN mvn -q -pl ais-auth-service -am -DskipTests package

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=build /workspace/ais-auth-service/target/ais-auth-service-*.jar app.jar
EXPOSE 8081
ENTRYPOINT ["java", "-jar", "app.jar"]

----------------------------------------------------
ais: backend: ais-auth-service:
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.ais</groupId>
        <artifactId>ais-backend</artifactId>
        <version>1.0.0</version>
    </parent>

    <artifactId>ais-auth-service</artifactId>
    <packaging>jar</packaging>
    <name>AIS Auth Service</name>
    <description>Issues and validates JWT access/refresh tokens for AIS admin users</description>

    <dependencies>
        <dependency>
            <groupId>com.ais</groupId>
            <artifactId>ais-common</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-core</artifactId>
        </dependency>
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-database-postgresql</artifactId>
        </dependency>
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>com.google.cloud.sql</groupId>
            <artifactId>postgres-socket-factory</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-impl</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-jackson</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.springdoc</groupId>
            <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <scope>provided</scope>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>postgresql</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>


============================================================


ais: backend: ais-common: src\main\java\com\ais\common: api
package com.ais.common.api;

import com.fasterxml.jackson.annotation.JsonInclude;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;

import java.time.Instant;
import java.util.List;

/**
 * Standard error response envelope produced by every GlobalExceptionHandler.
 * See ais_website_instructions.md Section 4B (API Contract Standards).
 */
@Getter
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ApiErrorResponse {

    private boolean success;
    private String errorCode;
    private String message;
    private List<String> details;
    private Instant timestamp;

    public static ApiErrorResponse of(String errorCode, String message, List<String> details) {
        return new ApiErrorResponse(false, errorCode, message, details, Instant.now());
    }
}


----------------------------------------------------
ais: backend: ais-common: src\main\java\com\ais\common: api
package com.ais.common.api;

import com.fasterxml.jackson.annotation.JsonInclude;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;

import java.time.Instant;

/**
 * Standard success response envelope used by every AIS microservice.
 * See ais_website_instructions.md Section 4B (API Contract Standards).
 */
@Getter
@NoArgsConstructor
@AllArgsConstructor
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ApiResponse<T> {

    private boolean success;
    private T data;
    private String message;
    private Instant timestamp;

    public static <T> ApiResponse<T> success(T data, String message) {
        return new ApiResponse<>(true, data, message, Instant.now());
    }

    public static <T> ApiResponse<T> success(T data) {
        return success(data, "Operation successful");
    }
}

----------------------------------------------------
ais: backend: ais-common: src\main\java\com\ais\common: api
package com.ais.common.api;

import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;
import org.springframework.data.domain.Page;

import java.util.List;

/**
 * Wraps a Spring Data {@link Page} into the wire shape defined in
 * ais_website_instructions.md Section 4B: { content, totalElements, totalPages, page, size }.
 */
@Getter
@NoArgsConstructor
@AllArgsConstructor
public class PageResponse<T> {

    private List<T> content;
    private long totalElements;
    private int totalPages;
    private int page;
    private int size;

    public static <T> PageResponse<T> of(Page<T> page) {
        return new PageResponse<>(
                page.getContent(),
                page.getTotalElements(),
                page.getTotalPages(),
                page.getNumber(),
                page.getSize());
    }
}

----------------------------------------------------
ais: backend: ais-common: src\main\java\com\ais\common: exception
package com.ais.common.exception;

/**
 * Thrown when a unique constraint would be violated by a create/update operation.
 * Mapped to HTTP 409 by GlobalExceptionHandler.
 */
public class DuplicateResourceException extends RuntimeException {

    public DuplicateResourceException(String message) {
        super(message);
    }
}

----------------------------------------------------
ais: backend: ais-common: src\main\java\com\ais\common: exception
package com.ais.common.exception;

/**
 * Thrown when a requested resource does not exist. Mapped to HTTP 404 by GlobalExceptionHandler.
 */
public class ResourceNotFoundException extends RuntimeException {

    public ResourceNotFoundException(String message) {
        super(message);
    }

    public static ResourceNotFoundException forEntity(String entityName, Object id) {
        return new ResourceNotFoundException(entityName + " with id " + id + " not found");
    }
}

----------------------------------------------------
ais: backend: ais-common: src\main\java\com\ais\common: security
package com.ais.common.security;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.JwtException;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;

import javax.crypto.SecretKey;
import java.nio.charset.StandardCharsets;
import java.util.List;

/**
 * Shared JWT verification utility used by every downstream service (and the gateway) to
 * independently re-validate a token's signature and expiry — defense in depth as described
 * in SECURITY_DESIGN.md Section 3. Only ais-auth-service issues tokens; every other
 * consumer of this class only verifies them.
 */
public class JwtTokenValidator {

    private final SecretKey signingKey;

    public JwtTokenValidator(String secret) {
        this.signingKey = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
    }

    /**
     * Parses and validates the token signature/expiry. Throws {@link JwtException} (or a
     * subclass) if the token is invalid, tampered, or expired.
     */
    public Claims validateAndParse(String token) {
        return Jwts.parser()
                .verifyWith(signingKey)
                .build()
                .parseSignedClaims(token)
                .getPayload();
    }

    public String extractUsername(Claims claims) {
        return claims.getSubject();
    }

    @SuppressWarnings("unchecked")
    public List<String> extractRoles(Claims claims) {
        Object roles = claims.get("roles");
        if (roles instanceof List<?>) {
            return (List<String>) roles;
        }
        return List.of();
    }
}

----------------------------------------------------
ais: backend: ais-common
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.ais</groupId>
        <artifactId>ais-backend</artifactId>
        <version>1.0.0</version>
    </parent>

    <artifactId>ais-common</artifactId>
    <packaging>jar</packaging>
    <name>AIS Common Library</name>
    <description>Shared API envelope, base exceptions, and JWT validation utility used by every AIS microservice</description>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.data</groupId>
            <artifactId>spring-data-commons</artifactId>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-impl</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-jackson</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <scope>provided</scope>
        </dependency>
    </dependencies>
</project>


----------------------------------------------------
ais: backend: ais-consultation-service: com.ais.consultation: config
package com.ais.consultation.config;

import com.ais.consultation.security.JwtAuthenticationFilter;
import com.ais.consultation.security.RestAccessDeniedHandler;
import com.ais.consultation.security.RestAuthenticationEntryPoint;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthenticationFilter;
    private final RestAuthenticationEntryPoint restAuthenticationEntryPoint;
    private final RestAccessDeniedHandler restAccessDeniedHandler;

    public SecurityConfig(JwtAuthenticationFilter jwtAuthenticationFilter,
                           RestAuthenticationEntryPoint restAuthenticationEntryPoint,
                           RestAccessDeniedHandler restAccessDeniedHandler) {
        this.jwtAuthenticationFilter = jwtAuthenticationFilter;
        this.restAuthenticationEntryPoint = restAuthenticationEntryPoint;
        this.restAccessDeniedHandler = restAccessDeniedHandler;
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
                .csrf(AbstractHttpConfigurer::disable)
                // CORS is handled centrally by ais-gateway-service (see HLD.md Section 5); this
                // service is never called directly by browsers, so it must NOT also add its own
                // Access-Control-Allow-Origin header - doing so duplicates the header through the
                // gateway proxy and browsers reject responses with a duplicated CORS header.
                .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .exceptionHandling(eh -> eh
                        .authenticationEntryPoint(restAuthenticationEntryPoint)
                        .accessDeniedHandler(restAccessDeniedHandler))
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers(org.springframework.http.HttpMethod.POST, "/api/v1/consultation/requests").permitAll()
                        .requestMatchers("/swagger-ui/**", "/v3/api-docs/**", "/actuator/health").permitAll()
                        .anyRequest().hasAnyRole("ADMIN", "EDITOR"))
                .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}

----------------------------------------------------
ais: backend: ais-consultation-service: com.ais.consultation: controller
package com.ais.consultation.controller;

import com.ais.common.api.ApiResponse;
import com.ais.common.api.PageResponse;
import com.ais.consultation.dto.ConsultationRequestDto;
import com.ais.consultation.dto.ConsultationResponseDto;
import com.ais.consultation.dto.UpdateStatusDto;
import com.ais.consultation.entity.LeadStatus;
import com.ais.consultation.service.ConsultationService;
import io.swagger.v3.oas.annotations.Operation;
import jakarta.validation.Valid;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/consultation/requests")
public class ConsultationController {

    private final ConsultationService consultationService;

    public ConsultationController(ConsultationService consultationService) {
        this.consultationService = consultationService;
    }

    @Operation(summary = "Submit a consultation request (public lead-generation endpoint)")
    @PostMapping
    public ResponseEntity<ApiResponse<ConsultationResponseDto>> submit(@Valid @RequestBody ConsultationRequestDto dto) {
        ConsultationResponseDto response = consultationService.submitRequest(dto);
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(ApiResponse.success(response, "Consultation request submitted successfully"));
    }

    @Operation(summary = "Get a consultation request by id (Admin/Editor only)")
    @GetMapping("/{id}")
    @PreAuthorize("hasAnyRole('ADMIN', 'EDITOR')")
    public ResponseEntity<ApiResponse<ConsultationResponseDto>> getById(@PathVariable Long id) {
        return ResponseEntity.ok(ApiResponse.success(consultationService.getById(id)));
    }

    @Operation(summary = "List/filter consultation requests, paginated (Admin/Editor only)")
    @GetMapping
    @PreAuthorize("hasAnyRole('ADMIN', 'EDITOR')")
    public ResponseEntity<ApiResponse<PageResponse<ConsultationResponseDto>>> list(
            @RequestParam(required = false) String industry,
            @RequestParam(required = false) LeadStatus status,
            Pageable pageable) {
        Page<ConsultationResponseDto> page = consultationService.list(industry, status, pageable);
        return ResponseEntity.ok(ApiResponse.success(PageResponse.of(page)));
    }

    @Operation(summary = "Update the status/remarks of a consultation request (Admin/Editor only)")
    @PatchMapping("/{id}/status")
    @PreAuthorize("hasAnyRole('ADMIN', 'EDITOR')")
    public ResponseEntity<ApiResponse<ConsultationResponseDto>> updateStatus(
            @PathVariable Long id, @Valid @RequestBody UpdateStatusDto dto) {
        return ResponseEntity.ok(ApiResponse.success(consultationService.updateStatus(id, dto), "Status updated successfully"));
    }

    @Operation(summary = "Export all consultation requests as CSV (Admin/Editor only)")
    @GetMapping("/export")
    @PreAuthorize("hasAnyRole('ADMIN', 'EDITOR')")
    public ResponseEntity<byte[]> exportCsv() {
        byte[] csv = consultationService.exportCsv();
        return ResponseEntity.ok()
                .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=consultation-requests.csv")
                .header(HttpHeaders.CONTENT_TYPE, "text/csv")
                .body(csv);
    }
}

----------------------------------------------------
ais: backend: ais-consultation-service: com.ais.consultation: controller
package com.ais.consultation.controller;

import com.ais.common.api.ApiResponse;
import com.ais.consultation.dto.DashboardStatsDto;
import com.ais.consultation.service.ConsultationService;
import io.swagger.v3.oas.annotations.Operation;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/v1/consultation/dashboard")
public class DashboardController {

    private final ConsultationService consultationService;

    public DashboardController(ConsultationService consultationService) {
        this.consultationService = consultationService;
    }

    @Operation(summary = "Aggregate lead statistics for the Admin Dashboard (Admin/Editor only)")
    @GetMapping("/stats")
    @PreAuthorize("hasAnyRole('ADMIN', 'EDITOR')")
    public ResponseEntity<ApiResponse<DashboardStatsDto>> getStats() {
        return ResponseEntity.ok(ApiResponse.success(consultationService.getDashboardStats()));
    }
}

----------------------------------------------------
ais: backend: ais-consultation-service: com.ais.consultation: dto
package com.ais.consultation.dto;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Pattern;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class ConsultationRequestDto {

    @NotBlank(message = "Organization name is required")
    private String organizationName;

    @NotBlank(message = "Contact person is required")
    private String contactPerson;

    @NotBlank(message = "Mobile number is required")
    @Pattern(regexp = "^\\+?[0-9]{7,15}$", message = "Mobile number format is invalid")
    private String mobile;

    @NotBlank(message = "Email is required")
    @Email(message = "Email format is invalid")
    private String email;

    @NotBlank(message = "Industry is required")
    private String industry;

    private String organizationSize;

    private String requirements;

    private String preferredEngagementStep;
}


----------------------------------------------------
ais: backend: ais-consultation-service: com.ais.consultation: dto
package com.ais.consultation.dto;

import com.ais.consultation.entity.LeadStatus;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

import java.time.Instant;

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class ConsultationResponseDto {

    private Long id;
    private String referenceNumber;
    private String organizationName;
    private String contactPerson;
    private String mobile;
    private String email;
    private String industry;
    private String organizationSize;
    private String requirements;
    private String preferredEngagementStep;
    private LeadStatus status;
    private String internalRemarks;
    private Instant createdDate;
}


----------------------------------------------------
ais: backend: ais-consultation-service: com.ais.consultation: dto
package com.ais.consultation.dto;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

import java.util.Map;

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class DashboardStatsDto {

    private long totalRequests;
    private long newRequestsLast30Days;
    private Map<String, Long> requestsByIndustry;
    private Map<String, Long> requestsByStatus;
}


----------------------------------------------------
ais: backend: ais-consultation-service: com.ais.consultation: dto
package com.ais.consultation.dto;

import com.ais.consultation.entity.LeadStatus;
import jakarta.validation.constraints.NotNull;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class UpdateStatusDto {

    @NotNull(message = "Status is required")
    private LeadStatus status;

    private String internalRemarks;
}


----------------------------------------------------
ais: backend: ais-consultation-service: com.ais.consultation: entity
package com.ais.consultation.entity;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

import java.time.Instant;

@Entity
@Table(name = "consultation_request")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class ConsultationRequest {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "reference_number", nullable = false, unique = true)
    private String referenceNumber;

    @Column(name = "organization_name", nullable = false)
    private String organizationName;

    @Column(name = "contact_person", nullable = false)
    private String contactPerson;

    @Column(name = "mobile", nullable = false)
    private String mobile;

    @Column(name = "email", nullable = false)
    private String email;

    @Column(name = "industry", nullable = false)
    private String industry;

    @Column(name = "organization_size")
    private String organizationSize;

    @Column(name = "requirements")
    private String requirements;

    @Column(name = "preferred_engagement_step")
    private String preferredEngagementStep;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    @Builder.Default
    private LeadStatus status = LeadStatus.NEW;

    @Column(name = "internal_remarks")
    private String internalRemarks;

    @Column(name = "created_date")
    @Builder.Default
    private Instant createdDate = Instant.now();
}


----------------------------------------------------
ais: backend: ais-consultation-service: com.ais.consultation: entity
package com.ais.consultation.entity;

public enum LeadStatus {
    NEW,
    CONTACTED,
    QUALIFIED,
    PROPOSAL_SENT,
    NEGOTIATION,
    WON,
    LOST
}


----------------------------------------------------
ais: backend: ais-consultation-service: com.ais.consultation: exception
package com.ais.consultation.exception;

import com.ais.common.api.ApiErrorResponse;
import com.ais.common.exception.DuplicateResourceException;
import com.ais.common.exception.ResourceNotFoundException;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.List;

@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(404)
                .body(ApiErrorResponse.of("RESOURCE_NOT_FOUND", ex.getMessage(), List.of()));
    }

    @ExceptionHandler(DuplicateResourceException.class)
    public ResponseEntity<ApiErrorResponse> handleDuplicate(DuplicateResourceException ex) {
        return ResponseEntity.status(409)
                .body(ApiErrorResponse.of("DUPLICATE_RESOURCE", ex.getMessage(), List.of()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        List<String> details = ex.getBindingResult().getFieldErrors().stream()
                .map(fe -> fe.getField() + ": " + fe.getDefaultMessage())
                .toList();
        return ResponseEntity.badRequest()
                .body(ApiErrorResponse.of("VALIDATION_FAILED", "Request validation failed", details));
    }

    // Method-level @PreAuthorize denials are resolved here (DispatcherServlet resolves them
    // internally before Spring Security's ExceptionTranslationFilter sees them) —
    // see ais_website_instructions.md Section 15A pitfall #2.
    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ApiErrorResponse> handleAccessDenied(AccessDeniedException ex) {
        return ResponseEntity.status(403)
                .body(ApiErrorResponse.of("FORBIDDEN", "You do not have permission to perform this action", List.of()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiErrorResponse> handleGeneric(Exception ex) {
        log.error("Unhandled exception in ais-consultation-service", ex);
        return ResponseEntity.internalServerError()
                .body(ApiErrorResponse.of("INTERNAL_ERROR", "An unexpected error occurred", List.of()));
    }
}

----------------------------------------------------
ais: backend: ais-consultation-service: com.ais.consultation: mapper
package com.ais.consultation.mapper;

import com.ais.consultation.dto.ConsultationRequestDto;
import com.ais.consultation.dto.ConsultationResponseDto;
import com.ais.consultation.entity.ConsultationRequest;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;

import java.util.List;

@Mapper(componentModel = "spring")
public interface ConsultationMapper {

    @Mapping(target = "id", ignore = true)
    @Mapping(target = "referenceNumber", ignore = true)
    @Mapping(target = "status", ignore = true)
    @Mapping(target = "internalRemarks", ignore = true)
    @Mapping(target = "createdDate", ignore = true)
    ConsultationRequest toEntity(ConsultationRequestDto dto);

    ConsultationResponseDto toResponse(ConsultationRequest entity);

    List<ConsultationResponseDto> toResponseList(List<ConsultationRequest> entities);
}

----------------------------------------------------
ais: backend: ais-consultation-service: com.ais.consultation: repository
package com.ais.consultation.repository;

import com.ais.consultation.entity.ConsultationRequest;
import com.ais.consultation.entity.LeadStatus;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;
import java.util.Optional;

public interface ConsultationRequestRepository extends JpaRepository<ConsultationRequest, Long> {

    Optional<ConsultationRequest> findByReferenceNumber(String referenceNumber);

    boolean existsByReferenceNumber(String referenceNumber);

    long countByCreatedDateAfter(java.time.Instant since);

    Page<ConsultationRequest> findByIndustryAndStatus(String industry, LeadStatus status, Pageable pageable);

    Page<ConsultationRequest> findByIndustry(String industry, Pageable pageable);

    Page<ConsultationRequest> findByStatus(LeadStatus status, Pageable pageable);

    List<ConsultationRequest> findAllByOrderByCreatedDateDesc();

    long countByStatus(LeadStatus status);

    long countByIndustry(String industry);
}

----------------------------------------------------
ais: backend: ais-consultation-service: com.ais.consultation: security
package com.ais.consultation.security;

import com.ais.common.security.JwtTokenValidator;
import io.jsonwebtoken.Claims;
import io.jsonwebtoken.JwtException;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.lang.NonNull;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.List;

/**
 * Re-validates the JWT signature/expiry independently of the gateway (defense in depth,
 * see SECURITY_DESIGN.md Section 3) and populates the SecurityContext with roles.
 */
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenValidator jwtTokenValidator;

    public JwtAuthenticationFilter(JwtTokenValidator jwtTokenValidator) {
        this.jwtTokenValidator = jwtTokenValidator;
    }

    @Override
    protected void doFilterInternal(@NonNull HttpServletRequest request,
                                     @NonNull HttpServletResponse response,
                                     @NonNull FilterChain filterChain) throws ServletException, IOException {
        String header = request.getHeader("Authorization");
        if (header != null && header.startsWith("Bearer ")) {
            String token = header.substring(7);
            try {
                Claims claims = jwtTokenValidator.validateAndParse(token);
                String username = jwtTokenValidator.extractUsername(claims);
                List<GrantedAuthority> authorities = jwtTokenValidator.extractRoles(claims).stream()
                        .map(SimpleGrantedAuthority::new)
                        .map(GrantedAuthority.class::cast)
                        .toList();

                UsernamePasswordAuthenticationToken authentication =
                        new UsernamePasswordAuthenticationToken(username, null, authorities);
                authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(authentication);
            } catch (JwtException | IllegalArgumentException ex) {
                SecurityContextHolder.clearContext();
            }
        }
        filterChain.doFilter(request, response);
    }
}

----------------------------------------------------
ais: backend: ais-consultation-service: com.ais.consultation: security
package com.ais.consultation.security;

import com.ais.common.security.JwtTokenValidator;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class JwtConfig {

    @Bean
    public JwtTokenValidator jwtTokenValidator(@Value("${app.jwt.secret}") String secret) {
        return new JwtTokenValidator(secret);
    }
}

----------------------------------------------------
ais: backend: ais-consultation-service: com.ais.consultation: security
package com.ais.consultation.security;

import com.ais.common.api.ApiErrorResponse;
import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.http.MediaType;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.security.web.access.AccessDeniedHandler;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.util.List;

@Component
public class RestAccessDeniedHandler implements AccessDeniedHandler {

    private final ObjectMapper objectMapper = new ObjectMapper().findAndRegisterModules();

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, AccessDeniedException accessDeniedException)
            throws IOException {
        response.setStatus(HttpServletResponse.SC_FORBIDDEN);
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        ApiErrorResponse body = ApiErrorResponse.of("FORBIDDEN", "You do not have permission to perform this action", List.of());
        response.getWriter().write(objectMapper.writeValueAsString(body));
    }
}

----------------------------------------------------
ais: backend: ais-consultation-service: com.ais.consultation: security
package com.ais.consultation.security;

import com.ais.common.api.ApiErrorResponse;
import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.http.MediaType;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.web.AuthenticationEntryPoint;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.util.List;

@Component
public class RestAuthenticationEntryPoint implements AuthenticationEntryPoint {

    private final ObjectMapper objectMapper = new ObjectMapper().findAndRegisterModules();

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException authException)
            throws IOException {
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        ApiErrorResponse body = ApiErrorResponse.of("UNAUTHORIZED", "Authentication is required to access this resource", List.of());
        response.getWriter().write(objectMapper.writeValueAsString(body));
    }
}

----------------------------------------------------
ais: backend: ais-consultation-service: com.ais.consultation: service
package com.ais.consultation.service;

import com.ais.consultation.dto.ConsultationRequestDto;
import com.ais.consultation.dto.ConsultationResponseDto;
import com.ais.consultation.dto.DashboardStatsDto;
import com.ais.consultation.dto.UpdateStatusDto;
import com.ais.consultation.entity.LeadStatus;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;

public interface ConsultationService {

    ConsultationResponseDto submitRequest(ConsultationRequestDto dto);

    ConsultationResponseDto getById(Long id);

    Page<ConsultationResponseDto> list(String industry, LeadStatus status, Pageable pageable);

    ConsultationResponseDto updateStatus(Long id, UpdateStatusDto dto);

    byte[] exportCsv();

    DashboardStatsDto getDashboardStats();
}

----------------------------------------------------
ais: backend: ais-consultation-service: com.ais.consultation: service: impl
package com.ais.consultation.service.impl;

import com.ais.common.exception.ResourceNotFoundException;
import com.ais.consultation.dto.ConsultationRequestDto;
import com.ais.consultation.dto.ConsultationResponseDto;
import com.ais.consultation.dto.DashboardStatsDto;
import com.ais.consultation.dto.UpdateStatusDto;
import com.ais.consultation.entity.ConsultationRequest;
import com.ais.consultation.entity.LeadStatus;
import com.ais.consultation.mapper.ConsultationMapper;
import com.ais.consultation.repository.ConsultationRequestRepository;
import com.ais.consultation.service.ConsultationService;
import com.ais.consultation.util.ReferenceNumberGenerator;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.io.ByteArrayOutputStream;
import java.io.PrintWriter;
import java.nio.charset.StandardCharsets;
import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.Arrays;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@Service
public class ConsultationServiceImpl implements ConsultationService {

    private static final Logger log = LoggerFactory.getLogger(ConsultationServiceImpl.class);

    private final ConsultationRequestRepository repository;
    private final ConsultationMapper mapper;
    private final ReferenceNumberGenerator referenceNumberGenerator;

    public ConsultationServiceImpl(ConsultationRequestRepository repository,
                                    ConsultationMapper mapper,
                                    ReferenceNumberGenerator referenceNumberGenerator) {
        this.repository = repository;
        this.mapper = mapper;
        this.referenceNumberGenerator = referenceNumberGenerator;
    }

    @Override
    @Transactional
    public ConsultationResponseDto submitRequest(ConsultationRequestDto dto) {
        ConsultationRequest entity = mapper.toEntity(dto);
        entity.setReferenceNumber(referenceNumberGenerator.generate());
        entity.setStatus(LeadStatus.NEW);
        entity.setCreatedDate(Instant.now());

        ConsultationRequest saved = repository.save(entity);
        log.info("Consultation request {} submitted for organization '{}'", saved.getReferenceNumber(), saved.getOrganizationName());
        return mapper.toResponse(saved);
    }

    @Override
    public ConsultationResponseDto getById(Long id) {
        ConsultationRequest entity = repository.findById(id)
                .orElseThrow(() -> ResourceNotFoundException.forEntity("Consultation request", id));
        return mapper.toResponse(entity);
    }

    @Override
    public Page<ConsultationResponseDto> list(String industry, LeadStatus status, Pageable pageable) {
        Page<ConsultationRequest> page;
        if (industry != null && status != null) {
            page = repository.findByIndustryAndStatus(industry, status, pageable);
        } else if (industry != null) {
            page = repository.findByIndustry(industry, pageable);
        } else if (status != null) {
            page = repository.findByStatus(status, pageable);
        } else {
            page = repository.findAll(pageable);
        }
        return page.map(mapper::toResponse);
    }

    @Override
    @Transactional
    public ConsultationResponseDto updateStatus(Long id, UpdateStatusDto dto) {
        ConsultationRequest entity = repository.findById(id)
                .orElseThrow(() -> ResourceNotFoundException.forEntity("Consultation request", id));

        entity.setStatus(dto.getStatus());
        if (dto.getInternalRemarks() != null) {
            entity.setInternalRemarks(dto.getInternalRemarks());
        }
        ConsultationRequest saved = repository.save(entity);
        log.info("Consultation request {} status updated to {}", saved.getReferenceNumber(), saved.getStatus());
        return mapper.toResponse(saved);
    }

    @Override
    public byte[] exportCsv() {
        List<ConsultationRequest> all = repository.findAllByOrderByCreatedDateDesc();
        ByteArrayOutputStream out = new ByteArrayOutputStream();
        try (PrintWriter writer = new PrintWriter(out, true, StandardCharsets.UTF_8)) {
            writer.println("Reference Number,Organization,Contact Person,Mobile,Email,Industry,Status,Created Date");
            for (ConsultationRequest r : all) {
                writer.printf("%s,%s,%s,%s,%s,%s,%s,%s%n",
                        r.getReferenceNumber(), csvEscape(r.getOrganizationName()), csvEscape(r.getContactPerson()),
                        r.getMobile(), r.getEmail(), r.getIndustry(), r.getStatus(), r.getCreatedDate());
            }
        }
        return out.toByteArray();
    }

    @Override
    public DashboardStatsDto getDashboardStats() {
        long total = repository.count();
        long last30Days = repository.countByCreatedDateAfter(Instant.now().minus(30, ChronoUnit.DAYS));

        Map<String, Long> byIndustry = repository.findAll().stream()
                .collect(Collectors.groupingBy(ConsultationRequest::getIndustry, Collectors.counting()));

        Map<String, Long> byStatus = Arrays.stream(LeadStatus.values())
                .collect(Collectors.toMap(Enum::name, repository::countByStatus));

        return DashboardStatsDto.builder()
                .totalRequests(total)
                .newRequestsLast30Days(last30Days)
                .requestsByIndustry(byIndustry)
                .requestsByStatus(byStatus)
                .build();
    }

    private String csvEscape(String value) {
        if (value == null) {
            return "";
        }
        if (value.contains(",") || value.contains("\"")) {
            return "\"" + value.replace("\"", "\"\"") + "\"";
        }
        return value;
    }
}

----------------------------------------------------
ais: backend: ais-consultation-service: com.ais.consultation: util
package com.ais.consultation.util;

import com.ais.consultation.repository.ConsultationRequestRepository;
import org.springframework.stereotype.Component;

import java.time.LocalDate;
import java.time.format.DateTimeFormatter;

/**
 * Generates unique consultation request reference numbers in the format
 * AIS-yyyyMMdd-XXXX (see TECHNICAL_SPEC.md Section 6).
 */
@Component
public class ReferenceNumberGenerator {

    private static final DateTimeFormatter DATE_FORMAT = DateTimeFormatter.BASIC_ISO_DATE;
    private static final int MAX_ATTEMPTS = 1000;

    private final ConsultationRequestRepository repository;

    public ReferenceNumberGenerator(ConsultationRequestRepository repository) {
        this.repository = repository;
    }

    public String generate() {
        String datePart = LocalDate.now().format(DATE_FORMAT);
        for (int sequence = 1; sequence <= MAX_ATTEMPTS; sequence++) {
            String candidate = "AIS-" + datePart + "-" + String.format("%04d", sequence);
            if (!repository.existsByReferenceNumber(candidate)) {
                return candidate;
            }
        }
        throw new IllegalStateException("Unable to generate a unique reference number for " + datePart);
    }
}

----------------------------------------------------
ais: backend: ais-consultation-service:  src: main: java: com.ais.consultation: 
package com.ais.consultation;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class AisConsultationServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(AisConsultationServiceApplication.class, args);
    }
}

----------------------------------------------------
ais: backend: ais-consultation-service: src: main: resources: db:migration
V1__init.sql
CREATE TABLE IF NOT EXISTS consultation_request (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    reference_number VARCHAR(50) NOT NULL UNIQUE,
    organization_name VARCHAR(255) NOT NULL,
    contact_person VARCHAR(255) NOT NULL,
    mobile VARCHAR(20) NOT NULL,
    email VARCHAR(255) NOT NULL,
    industry VARCHAR(100) NOT NULL,
    organization_size VARCHAR(100),
    requirements TEXT,
    preferred_engagement_step VARCHAR(100),
    status VARCHAR(50) NOT NULL DEFAULT 'NEW',
    internal_remarks TEXT,
    created_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_consultation_status ON consultation_request (status);
CREATE INDEX IF NOT EXISTS idx_consultation_industry ON consultation_request (industry);


----------------------------------------------------
ais: backend: ais-consultation-service:  src: main: resources: db:migration
V2__seed_data.sql
INSERT INTO consultation_request
(reference_number, organization_name, contact_person, mobile, email, industry, organization_size, requirements, preferred_engagement_step, status, internal_remarks, created_date)
VALUES
('AIS-20260601-0001', 'Greenfield College', 'Dr. Anita Rao', '+919876500001', 'anita.rao@greenfieldcollege.edu', 'College', '1000-5000 students', 'Need an admission and student helpdesk AI agent to reduce front-office workload.', 'Full End-to-End', 'WON', 'Signed 6-month engagement.', '2026-06-01 10:15:00'),
('AIS-20260603-0002', 'Sunrise Public School', 'Mr. Vikram Shah', '+919876500002', 'vikram.shah@sunriseschool.in', 'School', '500-1000 students', 'Automate attendance and parent notifications.', 'Design Only', 'PROPOSAL_SENT', 'Awaiting budget approval from trustees.', '2026-06-03 09:30:00'),
('AIS-20260605-0003', 'Nexus Auto Components Pvt Ltd', 'Ms. Priya Menon', '+919876500003', 'priya.menon@nexusauto.com', 'Factory', '200-500 employees', 'Predictive maintenance for CNC machines.', 'Full End-to-End', 'NEGOTIATION', 'Comparing with in-house build option.', '2026-06-05 14:00:00'),
('AIS-20260607-0004', 'Skyline Mall Group', 'Mr. Arjun Kapoor', '+919876500004', 'arjun.kapoor@skylinemall.com', 'Shopping Mall', '50-100 tenants', 'Customer assistant chatbot and footfall analytics.', 'Custom', 'CONTACTED', 'Scheduled discovery call for next week.', '2026-06-07 11:45:00'),
('AIS-20260610-0005', 'Bright Future Academy', 'Mrs. Kavita Iyer', '+919876500005', 'kavita.iyer@brightfuture.edu', 'School', '100-500 students', 'Fee reminder automation.', 'Design Only', 'NEW', NULL, '2026-06-10 08:20:00'),
('AIS-20260612-0006', 'Precision Tools Manufacturing', 'Mr. Rohit Verma', '+919876500006', 'rohit.verma@precisiontools.com', 'Factory', '500+ employees', 'Quality inspection agent using computer vision.', 'Full End-to-End', 'QUALIFIED', 'Strong fit, moving to proposal stage.', '2026-06-12 16:10:00'),
('AIS-20260615-0007', 'Metro City Mall', 'Ms. Sneha Kulkarni', '+919876500007', 'sneha.kulkarni@metrocitymall.com', 'Shopping Mall', '100+ tenants', 'Security monitoring agent integration.', 'Custom', 'LOST', 'Went with an in-house team.', '2026-06-15 13:00:00'),
('AIS-20260618-0008', 'Horizon Institute of Technology', 'Dr. Sameer Joshi', '+919876500008', 'sameer.joshi@horizoninstitute.edu', 'College', '2000+ students', 'Placement agent and parent communication agent.', 'Full End-to-End', 'NEW', NULL, '2026-06-18 10:00:00');

----------------------------------------------------
ais: backend: ais-consultation-service: src: main: resources: 
application-docker.yaml
spring:
  datasource:
    url: jdbc:postgresql://postgres:5432/ais_db
    username: ${DB_USERNAME:ais_consultation_user}
    password: ${DB_PASSWORD:ais_consultation_pass}

----------------------------------------------------
ais: backend: ais-consultation-service: src: main: resources: 
application-gcp.yaml
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  jpa:
    show-sql: false

feature:
  csv-export-enabled: true
  email-notifications-enabled: true

logging:
  level:
    com.ais: INFO

----------------------------------------------------
ais: backend: ais-consultation-service: src: main: resources: 
application-h2.yaml
# Local, no-install-required profile: runs against an in-memory H2 database
# (PostgreSQL compatibility mode) instead of a real PostgreSQL server/Docker.
# Start with: mvn spring-boot:run -Dspring-boot.run.profiles=h2
spring:
  datasource:
    url: jdbc:h2:mem:ais_consultation;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
    driver-class-name: org.h2.Driver
    username: sa
    password:
  jpa:
    show-sql: true
  flyway:
    schemas: PUBLIC
    default-schema: PUBLIC
    create-schemas: false
  h2:
    console:
      enabled: true
      path: /h2-console

logging:
  level:
    com.ais: DEBUG

----------------------------------------------------
ais: backend: ais-consultation-service: src: main: resources: 
application-local.yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/ais_db
    username: ais_consultation_user
    password: ais_consultation_pass
  jpa:
    show-sql: true

logging:
  level:
    com.ais: DEBUG

----------------------------------------------------
ais: backend: ais-consultation-service: src: main: resources: 
application.yaml
server:
  port: 8082

spring:
  application:
    name: ais-consultation-service
  jpa:
    hibernate:
      ddl-auto: validate
    open-in-view: false
  flyway:
    enabled: true
    schemas: consultation_schema
    default-schema: consultation_schema
    create-schemas: true
    locations: classpath:db/migration

app:
  jwt:
    secret: ${JWT_SECRET:local-dev-only-change-me-must-be-32-bytes-minimum}
  cors:
    allowed-origins: ${CORS_ALLOWED_ORIGINS:http://localhost:4200}

feature:
  csv-export-enabled: true

management:
  endpoints:
    web:
      exposure:
        include: health,info

springdoc:
  swagger-ui:
    path: /swagger-ui.html

----------------------------------------------------
ais: backend: ais-consultation-service: src: main: resources: 
logback-spring.xml
<configuration>
    <springProfile name="local | default | h2">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{ISO8601} [%thread] %-5level %logger{36} [%X{requestId}] - %msg%n</pattern>
            </encoder>
        </appender>
        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>

    <springProfile name="docker,gcp">
        <appender name="JSON_CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
        </appender>
        <root level="INFO">
            <appender-ref ref="JSON_CONSOLE"/>
        </root>
    </springProfile>
</configuration>

----------------------------------------------------
ais: backend: ais-consultation-service: 
Dockerfile
# Multi-stage build. Build context is the backend/ reactor root (see docker-compose.yml).
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /workspace
COPY pom.xml .
COPY ais-common ais-common
COPY ais-consultation-service ais-consultation-service
RUN mvn -q -pl ais-consultation-service -am -DskipTests package

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=build /workspace/ais-consultation-service/target/ais-consultation-service-*.jar app.jar
EXPOSE 8082
ENTRYPOINT ["java", "-jar", "app.jar"]


----------------------------------------------------
ais: backend: ais-consultation-service:
pom.xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.ais</groupId>
        <artifactId>ais-backend</artifactId>
        <version>1.0.0</version>
    </parent>

    <artifactId>ais-consultation-service</artifactId>
    <packaging>jar</packaging>
    <name>AIS Consultation Service</name>
    <description>Captures and manages consultation requests (leads) - the flagship business feature of the AIS platform</description>

    <dependencies>
        <dependency>
            <groupId>com.ais</groupId>
            <artifactId>ais-common</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        <dependency>
            <groupId>com.google.cloud.sql</groupId>
            <artifactId>postgres-socket-factory</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-core</artifactId>
        </dependency>
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-database-postgresql</artifactId>
        </dependency>
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-impl</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-jackson</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.mapstruct</groupId>
            <artifactId>mapstruct</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springdoc</groupId>
            <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <scope>provided</scope>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>postgresql</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <parameters>true</parameters>
                    <annotationProcessorPaths>
                        <path>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </path>
                        <path>
                            <groupId>org.mapstruct</groupId>
                            <artifactId>mapstruct-processor</artifactId>
                        </path>
                        <path>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok-mapstruct-binding</artifactId>
                            <version>0.2.0</version>
                        </path>
                    </annotationProcessorPaths>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
-------------------------------------


