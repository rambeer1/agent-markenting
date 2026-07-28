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

