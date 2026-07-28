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
========================================================
ais: backend: ais-content-service: src\main\java\com\ais\content: config
package com.ais.content.config;

import com.ais.content.security.JwtAuthenticationFilter;
import com.ais.content.security.RestAccessDeniedHandler;
import com.ais.content.security.RestAuthenticationEntryPoint;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
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
                        .requestMatchers(HttpMethod.GET,
                                "/api/v1/content/home", "/api/v1/content/about",
                                "/api/v1/content/services/**", "/api/v1/content/industries/**",
                                "/api/v1/content/use-cases/**", "/api/v1/content/case-studies/**",
                                "/api/v1/content/testimonials")
                        .permitAll()
                        .requestMatchers(HttpMethod.POST, "/api/v1/content/contact").permitAll()
                        .requestMatchers("/swagger-ui/**", "/v3/api-docs/**", "/actuator/health").permitAll()
                        .anyRequest().hasAnyRole("ADMIN", "EDITOR"))
                .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: controller
package com.ais.content.controller;

import com.ais.common.api.ApiResponse;
import com.ais.content.dto.*;
import com.ais.content.service.CompanyProfileService;
import com.ais.content.service.IndustryCatalogService;
import com.ais.content.service.EngagementFeedbackService;
import io.swagger.v3.oas.annotations.Operation;
import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;

/**
 * Consolidated CRUD controller for all admin-managed content types (services, industries,
 * use cases, case studies, testimonials). ROLE_ADMIN or ROLE_EDITOR may create/update;
 * only ROLE_ADMIN may delete, per SECURITY_DESIGN.md.
 */
@RestController
@RequestMapping("/api/v1/content/admin")
@PreAuthorize("hasAnyRole('ADMIN', 'EDITOR')")
public class AdminContentController {

    private final CompanyProfileService companyProfileService;
    private final IndustryCatalogService industryCatalogService;
    private final EngagementFeedbackService engagementFeedbackService;

    public AdminContentController(CompanyProfileService companyProfileService,
                                   IndustryCatalogService industryCatalogService,
                                   EngagementFeedbackService engagementFeedbackService) {
        this.companyProfileService = companyProfileService;
        this.industryCatalogService = industryCatalogService;
        this.engagementFeedbackService = engagementFeedbackService;
    }

    // ---- Service Offerings ----

    @PostMapping("/services")
    public ResponseEntity<ApiResponse<ServiceOfferingDto>> createService(@Valid @RequestBody ServiceOfferingRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(ApiResponse.success(companyProfileService.createServiceOffering(request), "Service offering created"));
    }

    @PutMapping("/services/{id}")
    public ResponseEntity<ApiResponse<ServiceOfferingDto>> updateService(@PathVariable Long id, @Valid @RequestBody ServiceOfferingRequest request) {
        return ResponseEntity.ok(ApiResponse.success(companyProfileService.updateServiceOffering(id, request), "Service offering updated"));
    }

    @DeleteMapping("/services/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<Void> deleteService(@PathVariable Long id) {
        companyProfileService.deleteServiceOffering(id);
        return ResponseEntity.noContent().build();
    }

    // ---- Industries ----

    @PostMapping("/industries")
    public ResponseEntity<ApiResponse<IndustryDto>> createIndustry(@Valid @RequestBody IndustryRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(ApiResponse.success(industryCatalogService.createIndustry(request), "Industry created"));
    }

    @PutMapping("/industries/{id}")
    public ResponseEntity<ApiResponse<IndustryDto>> updateIndustry(@PathVariable Long id, @Valid @RequestBody IndustryRequest request) {
        return ResponseEntity.ok(ApiResponse.success(industryCatalogService.updateIndustry(id, request), "Industry updated"));
    }

    @DeleteMapping("/industries/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<Void> deleteIndustry(@PathVariable Long id) {
        industryCatalogService.deleteIndustry(id);
        return ResponseEntity.noContent().build();
    }

    // ---- Use Cases ----

    @PostMapping("/use-cases")
    public ResponseEntity<ApiResponse<UseCaseDto>> createUseCase(@Valid @RequestBody UseCaseRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(ApiResponse.success(industryCatalogService.createUseCase(request), "Use case created"));
    }

    @PutMapping("/use-cases/{id}")
    public ResponseEntity<ApiResponse<UseCaseDto>> updateUseCase(@PathVariable Long id, @Valid @RequestBody UseCaseRequest request) {
        return ResponseEntity.ok(ApiResponse.success(industryCatalogService.updateUseCase(id, request), "Use case updated"));
    }

    @DeleteMapping("/use-cases/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<Void> deleteUseCase(@PathVariable Long id) {
        industryCatalogService.deleteUseCase(id);
        return ResponseEntity.noContent().build();
    }

    // ---- Case Studies ----

    @Operation(summary = "Create a case study")
    @PostMapping("/case-studies")
    public ResponseEntity<ApiResponse<CaseStudyDto>> createCaseStudy(@Valid @RequestBody CaseStudyRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(ApiResponse.success(industryCatalogService.createCaseStudy(request), "Case study created"));
    }

    @PutMapping("/case-studies/{id}")
    public ResponseEntity<ApiResponse<CaseStudyDto>> updateCaseStudy(@PathVariable Long id, @Valid @RequestBody CaseStudyRequest request) {
        return ResponseEntity.ok(ApiResponse.success(industryCatalogService.updateCaseStudy(id, request), "Case study updated"));
    }

    @DeleteMapping("/case-studies/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<Void> deleteCaseStudy(@PathVariable Long id) {
        industryCatalogService.deleteCaseStudy(id);
        return ResponseEntity.noContent().build();
    }

    // ---- Testimonials ----

    @PostMapping("/testimonials")
    public ResponseEntity<ApiResponse<TestimonialDto>> createTestimonial(@Valid @RequestBody TestimonialRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(ApiResponse.success(engagementFeedbackService.createTestimonial(request), "Testimonial created"));
    }

    @PutMapping("/testimonials/{id}")
    public ResponseEntity<ApiResponse<TestimonialDto>> updateTestimonial(@PathVariable Long id, @Valid @RequestBody TestimonialRequest request) {
        return ResponseEntity.ok(ApiResponse.success(engagementFeedbackService.updateTestimonial(id, request), "Testimonial updated"));
    }

    @DeleteMapping("/testimonials/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<Void> deleteTestimonial(@PathVariable Long id) {
        engagementFeedbackService.deleteTestimonial(id);
        return ResponseEntity.noContent().build();
    }
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: controller
package com.ais.content.controller;

import com.ais.common.api.ApiResponse;
import com.ais.common.api.PageResponse;
import com.ais.content.dto.ContactMessageDto;
import com.ais.content.dto.ContactMessageRequest;
import com.ais.content.service.EngagementFeedbackService;
import io.swagger.v3.oas.annotations.Operation;
import jakarta.validation.Valid;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/content/contact")
public class ContactController {

    private final EngagementFeedbackService engagementFeedbackService;

    public ContactController(EngagementFeedbackService engagementFeedbackService) {
        this.engagementFeedbackService = engagementFeedbackService;
    }

    @Operation(summary = "Submit a contact form message (public)")
    @PostMapping
    public ResponseEntity<ApiResponse<ContactMessageDto>> submit(@Valid @RequestBody ContactMessageRequest request) {
        ContactMessageDto saved = engagementFeedbackService.submitContactMessage(request);
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(ApiResponse.success(saved, "Message submitted successfully"));
    }

    @Operation(summary = "List contact messages, paginated (Admin/Editor only)")
    @GetMapping
    @PreAuthorize("hasAnyRole('ADMIN', 'EDITOR')")
    public ResponseEntity<ApiResponse<PageResponse<ContactMessageDto>>> list(Pageable pageable) {
        Page<ContactMessageDto> page = engagementFeedbackService.listContactMessages(pageable);
        return ResponseEntity.ok(ApiResponse.success(PageResponse.of(page)));
    }
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: controller
package com.ais.content.controller;

import com.ais.common.api.ApiResponse;
import com.ais.content.dto.*;
import com.ais.content.service.CompanyProfileService;
import com.ais.content.service.IndustryCatalogService;
import com.ais.content.service.EngagementFeedbackService;
import io.swagger.v3.oas.annotations.Operation;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

/**
 * Exposes every public (unauthenticated) content read endpoint. Consolidated into a single
 * controller (time-boxed design decision, see docs/TECHNICAL_SPEC.md) instead of one
 * controller per resource, since these are all simple reads over the same content domain.
 */
@RestController
@RequestMapping("/api/v1/content")
public class PublicContentController {

    private final CompanyProfileService companyProfileService;
    private final IndustryCatalogService industryCatalogService;
    private final EngagementFeedbackService engagementFeedbackService;

    public PublicContentController(CompanyProfileService companyProfileService,
                                    IndustryCatalogService industryCatalogService,
                                    EngagementFeedbackService engagementFeedbackService) {
        this.companyProfileService = companyProfileService;
        this.industryCatalogService = industryCatalogService;
        this.engagementFeedbackService = engagementFeedbackService;
    }

    @Operation(summary = "Aggregated home page content: value prop, industries, services, featured case studies/testimonials")
    @GetMapping("/home")
    public ResponseEntity<ApiResponse<HomeDto>> getHome() {
        CompanyInfoDto companyInfo = companyProfileService.getCompanyInfo();
        List<CaseStudyDto> caseStudies = industryCatalogService.listCaseStudies();
        List<TestimonialDto> testimonials = engagementFeedbackService.listTestimonials();

        HomeDto home = HomeDto.builder()
                .companyName(companyInfo.getName())
                .tagline(companyInfo.getTagline())
                .industries(industryCatalogService.listIndustries())
                .services(companyProfileService.listServiceOfferings())
                .featuredCaseStudies(caseStudies.stream().limit(3).toList())
                .featuredTestimonials(testimonials.stream().limit(3).toList())
                .build();
        return ResponseEntity.ok(ApiResponse.success(home));
    }

    @Operation(summary = "About Us / founder profile")
    @GetMapping("/about")
    public ResponseEntity<ApiResponse<CompanyInfoDto>> getAbout() {
        return ResponseEntity.ok(ApiResponse.success(companyProfileService.getCompanyInfo()));
    }

    @Operation(summary = "List the 4-step engagement / services offering")
    @GetMapping("/services")
    public ResponseEntity<ApiResponse<List<ServiceOfferingDto>>> listServices() {
        return ResponseEntity.ok(ApiResponse.success(companyProfileService.listServiceOfferings()));
    }

    @GetMapping("/services/{id}")
    public ResponseEntity<ApiResponse<ServiceOfferingDto>> getService(@PathVariable Long id) {
        return ResponseEntity.ok(ApiResponse.success(companyProfileService.getServiceOffering(id)));
    }

    @Operation(summary = "List industries AIS serves")
    @GetMapping("/industries")
    public ResponseEntity<ApiResponse<List<IndustryDto>>> listIndustries() {
        return ResponseEntity.ok(ApiResponse.success(industryCatalogService.listIndustries()));
    }

    @GetMapping("/industries/{id}")
    public ResponseEntity<ApiResponse<IndustryDto>> getIndustry(@PathVariable Long id) {
        return ResponseEntity.ok(ApiResponse.success(industryCatalogService.getIndustry(id)));
    }

    @Operation(summary = "List AI agent use cases, optionally filtered by industryId")
    @GetMapping("/use-cases")
    public ResponseEntity<ApiResponse<List<UseCaseDto>>> listUseCases(
            @RequestParam(required = false) Long industryId) {
        return ResponseEntity.ok(ApiResponse.success(industryCatalogService.listUseCases(industryId)));
    }

    @GetMapping("/use-cases/{id}")
    public ResponseEntity<ApiResponse<UseCaseDto>> getUseCase(@PathVariable Long id) {
        return ResponseEntity.ok(ApiResponse.success(industryCatalogService.getUseCase(id)));
    }

    @Operation(summary = "List published case studies / portfolio")
    @GetMapping("/case-studies")
    public ResponseEntity<ApiResponse<List<CaseStudyDto>>> listCaseStudies() {
        return ResponseEntity.ok(ApiResponse.success(industryCatalogService.listCaseStudies()));
    }

    @GetMapping("/case-studies/{id}")
    public ResponseEntity<ApiResponse<CaseStudyDto>> getCaseStudy(@PathVariable Long id) {
        return ResponseEntity.ok(ApiResponse.success(industryCatalogService.getCaseStudy(id)));
    }

    @Operation(summary = "List client testimonials")
    @GetMapping("/testimonials")
    public ResponseEntity<ApiResponse<List<TestimonialDto>>> listTestimonials() {
        return ResponseEntity.ok(ApiResponse.success(engagementFeedbackService.listTestimonials()));
    }
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: dto
package com.ais.content.dto;

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
public class CaseStudyDto {
    private Long id;
    private Long industryId;
    private String industryName;
    private String clientType;
    private String challenge;
    private String solution;
    private String techStack;
    private String results;
    private Instant publishedDate;
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: dto
package com.ais.content.dto;

import jakarta.validation.constraints.NotBlank;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class CaseStudyRequest {

    private Long industryId;

    @NotBlank(message = "Client type is required")
    private String clientType;

    private String challenge;
    private String solution;
    private String techStack;
    private String results;
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: dto
package com.ais.content.dto;

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
public class CompanyInfoDto {
    private Long id;
    private String name;
    private String tagline;
    private String mission;
    private String vision;
    private String founderBio;
    private String address;
    private String phone;
    private String email;
    private String linkedinUrl;
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: dto
package com.ais.content.dto;

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
public class ContactMessageDto {
    private Long id;
    private String name;
    private String email;
    private String mobile;
    private String subject;
    private String message;
    private Instant createdDate;
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: dto
package com.ais.content.dto;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class ContactMessageRequest {

    @NotBlank(message = "Name is required")
    private String name;

    @NotBlank(message = "Email is required")
    @Email(message = "Email format is invalid")
    private String email;

    private String mobile;
    private String subject;

    @NotBlank(message = "Message is required")
    private String message;
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: dto
package com.ais.content.dto;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

import java.util.List;

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class HomeDto {
    private String companyName;
    private String tagline;
    private List<IndustryDto> industries;
    private List<ServiceOfferingDto> services;
    private List<CaseStudyDto> featuredCaseStudies;
    private List<TestimonialDto> featuredTestimonials;
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: dto
package com.ais.content.dto;

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
public class IndustryDto {
    private Long id;
    private String name;
    private String iconUrl;
    private String description;
    private String painPoints;
    private String howAiHelps;
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: dto
package com.ais.content.dto;

import jakarta.validation.constraints.NotBlank;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class IndustryRequest {

    @NotBlank(message = "Name is required")
    private String name;

    private String iconUrl;
    private String description;
    private String painPoints;
    private String howAiHelps;
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: dto
package com.ais.content.dto;

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
public class ServiceOfferingDto {
    private Long id;
    private Integer stepNumber;
    private String title;
    private String description;
    private String deliverables;
    private String estimatedDuration;
    private String status;
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: dto
package com.ais.content.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class ServiceOfferingRequest {

    @NotNull(message = "Step number is required")
    private Integer stepNumber;

    @NotBlank(message = "Title is required")
    private String title;

    private String description;
    private String deliverables;
    private String estimatedDuration;
    private String status;
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: dto
package com.ais.content.dto;

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
public class TestimonialDto {
    private Long id;
    private String clientName;
    private String organization;
    private String designation;
    private String testimonialText;
    private Integer rating;
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: dto
package com.ais.content.dto;

import jakarta.validation.constraints.Max;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class TestimonialRequest {

    @NotBlank(message = "Client name is required")
    private String clientName;

    private String organization;
    private String designation;

    @NotBlank(message = "Testimonial text is required")
    private String testimonialText;

    @Min(value = 1, message = "Rating must be between 1 and 5")
    @Max(value = 5, message = "Rating must be between 1 and 5")
    private Integer rating;
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: dto
package com.ais.content.dto;

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
public class UseCaseDto {
    private Long id;
    private Long industryId;
    private String industryName;
    private String name;
    private String problemStatement;
    private String proposedSolution;
    private String expectedRoi;
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: dto
package com.ais.content.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class UseCaseRequest {

    @NotNull(message = "Industry id is required")
    private Long industryId;

    @NotBlank(message = "Name is required")
    private String name;

    private String problemStatement;
    private String proposedSolution;
    private String expectedRoi;
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: entity
package com.ais.content.entity;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

import java.time.Instant;

@Entity
@Table(name = "case_study")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class CaseStudy {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "industry_id")
    private Industry industry;

    @Column(name = "client_type")
    private String clientType;

    private String challenge;
    private String solution;

    @Column(name = "tech_stack")
    private String techStack;

    private String results;

    @Column(name = "published_date")
    private Instant publishedDate;
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: entity
package com.ais.content.entity;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

@Entity
@Table(name = "company_info")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class CompanyInfo {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    private String tagline;
    private String mission;
    private String vision;

    @Column(name = "founder_bio")
    private String founderBio;

    private String address;
    private String phone;
    private String email;

    @Column(name = "linkedin_url")
    private String linkedinUrl;
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: entity
package com.ais.content.entity;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

import java.time.Instant;

@Entity
@Table(name = "contact_message")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class ContactMessage {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false)
    private String email;

    private String mobile;
    private String subject;

    @Column(nullable = false)
    private String message;

    @Column(name = "created_date")
    @Builder.Default
    private Instant createdDate = Instant.now();
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: entity
package com.ais.content.entity;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

@Entity
@Table(name = "industry")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Industry {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String name;

    @Column(name = "icon_url")
    private String iconUrl;

    private String description;

    @Column(name = "pain_points")
    private String painPoints;

    @Column(name = "how_ai_helps")
    private String howAiHelps;
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: entity
package com.ais.content.entity;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

@Entity
@Table(name = "service_offering")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class ServiceOffering {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "step_number", nullable = false)
    private Integer stepNumber;

    @Column(nullable = false)
    private String title;

    private String description;
    private String deliverables;

    @Column(name = "estimated_duration")
    private String estimatedDuration;

    @Builder.Default
    private String status = "ACTIVE";
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: entity
package com.ais.content.entity;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

@Entity
@Table(name = "testimonial")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Testimonial {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "client_name", nullable = false)
    private String clientName;

    private String organization;
    private String designation;

    @Column(name = "testimonial_text", nullable = false)
    private String testimonialText;

    private Integer rating;
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: entity
package com.ais.content.entity;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

@Entity
@Table(name = "use_case")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class UseCase {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "industry_id", nullable = false)
    private Industry industry;

    @Column(nullable = false)
    private String name;

    @Column(name = "problem_statement")
    private String problemStatement;

    @Column(name = "proposed_solution")
    private String proposedSolution;

    @Column(name = "expected_roi")
    private String expectedRoi;
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: exception
package com.ais.content.exception;

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

    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ApiErrorResponse> handleAccessDenied(AccessDeniedException ex) {
        return ResponseEntity.status(403)
                .body(ApiErrorResponse.of("FORBIDDEN", "You do not have permission to perform this action", List.of()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiErrorResponse> handleGeneric(Exception ex) {
        log.error("Unhandled exception in ais-content-service", ex);
        return ResponseEntity.internalServerError()
                .body(ApiErrorResponse.of("INTERNAL_ERROR", "An unexpected error occurred", List.of()));
    }
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: repository
package com.ais.content.repository;

import com.ais.content.entity.CaseStudy;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;

public interface CaseStudyRepository extends JpaRepository<CaseStudy, Long> {

    List<CaseStudy> findByIndustryId(Long industryId);
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: repository
package com.ais.content.repository;

import com.ais.content.entity.CompanyInfo;
import org.springframework.data.jpa.repository.JpaRepository;

public interface CompanyInfoRepository extends JpaRepository<CompanyInfo, Long> {
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: repository
package com.ais.content.repository;

import com.ais.content.entity.ContactMessage;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;

public interface ContactMessageRepository extends JpaRepository<ContactMessage, Long> {

    Page<ContactMessage> findAllByOrderByCreatedDateDesc(Pageable pageable);
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: repository
package com.ais.content.repository;

import com.ais.content.entity.Industry;
import org.springframework.data.jpa.repository.JpaRepository;

public interface IndustryRepository extends JpaRepository<Industry, Long> {
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: repository
package com.ais.content.repository;

import com.ais.content.entity.ServiceOffering;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;

public interface ServiceOfferingRepository extends JpaRepository<ServiceOffering, Long> {

    List<ServiceOffering> findAllByOrderByStepNumberAsc();
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: repository
package com.ais.content.repository;

import com.ais.content.entity.Testimonial;
import org.springframework.data.jpa.repository.JpaRepository;

public interface TestimonialRepository extends JpaRepository<Testimonial, Long> {
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: repository
package com.ais.content.repository;

import com.ais.content.entity.UseCase;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;

public interface UseCaseRepository extends JpaRepository<UseCase, Long> {

    List<UseCase> findByIndustryId(Long industryId);
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: security
package com.ais.content.security;

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

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: security
package com.ais.content.security;

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

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: security
package com.ais.content.security;

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

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: security
package com.ais.content.security;

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

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: service
package com.ais.content.service;

import com.ais.content.dto.CompanyInfoDto;
import com.ais.content.dto.ServiceOfferingDto;
import com.ais.content.dto.ServiceOfferingRequest;

import java.util.List;

/**
 * Handles the "About Us" (CompanyInfo) and "Services" (ServiceOffering / 4-step
 * engagement) content domains.
 */
public interface CompanyProfileService {

    CompanyInfoDto getCompanyInfo();

    List<ServiceOfferingDto> listServiceOfferings();

    ServiceOfferingDto getServiceOffering(Long id);

    ServiceOfferingDto createServiceOffering(ServiceOfferingRequest request);

    ServiceOfferingDto updateServiceOffering(Long id, ServiceOfferingRequest request);

    void deleteServiceOffering(Long id);
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: service
package com.ais.content.service;

import com.ais.content.dto.ContactMessageDto;
import com.ais.content.dto.ContactMessageRequest;
import com.ais.content.dto.TestimonialDto;
import com.ais.content.dto.TestimonialRequest;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;

import java.util.List;

/**
 * Handles the "Testimonials" and "Contact Us" content domains.
 */
public interface EngagementFeedbackService {

    List<TestimonialDto> listTestimonials();

    TestimonialDto createTestimonial(TestimonialRequest request);

    TestimonialDto updateTestimonial(Long id, TestimonialRequest request);

    void deleteTestimonial(Long id);

    ContactMessageDto submitContactMessage(ContactMessageRequest request);

    Page<ContactMessageDto> listContactMessages(Pageable pageable);
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: service
package com.ais.content.service;

import com.ais.content.dto.CaseStudyDto;
import com.ais.content.dto.CaseStudyRequest;
import com.ais.content.dto.IndustryDto;
import com.ais.content.dto.IndustryRequest;
import com.ais.content.dto.UseCaseDto;
import com.ais.content.dto.UseCaseRequest;

import java.util.List;

/**
 * Handles the "Industries We Serve", "AI Agent Use Cases", and "Case Studies" content domains.
 */
public interface IndustryCatalogService {

    List<IndustryDto> listIndustries();

    IndustryDto getIndustry(Long id);

    IndustryDto createIndustry(IndustryRequest request);

    IndustryDto updateIndustry(Long id, IndustryRequest request);

    void deleteIndustry(Long id);

    List<UseCaseDto> listUseCases(Long industryId);

    UseCaseDto getUseCase(Long id);

    UseCaseDto createUseCase(UseCaseRequest request);

    UseCaseDto updateUseCase(Long id, UseCaseRequest request);

    void deleteUseCase(Long id);

    List<CaseStudyDto> listCaseStudies();

    CaseStudyDto getCaseStudy(Long id);

    CaseStudyDto createCaseStudy(CaseStudyRequest request);

    CaseStudyDto updateCaseStudy(Long id, CaseStudyRequest request);

    void deleteCaseStudy(Long id);
}

-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: service:impl
package com.ais.content.service.impl;

import com.ais.common.exception.ResourceNotFoundException;
import com.ais.content.dto.CompanyInfoDto;
import com.ais.content.dto.ServiceOfferingDto;
import com.ais.content.dto.ServiceOfferingRequest;
import com.ais.content.entity.CompanyInfo;
import com.ais.content.entity.ServiceOffering;
import com.ais.content.repository.CompanyInfoRepository;
import com.ais.content.repository.ServiceOfferingRepository;
import com.ais.content.service.CompanyProfileService;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
public class CompanyProfileServiceImpl implements CompanyProfileService {

    private final CompanyInfoRepository companyInfoRepository;
    private final ServiceOfferingRepository serviceOfferingRepository;

    public CompanyProfileServiceImpl(CompanyInfoRepository companyInfoRepository,
                                      ServiceOfferingRepository serviceOfferingRepository) {
        this.companyInfoRepository = companyInfoRepository;
        this.serviceOfferingRepository = serviceOfferingRepository;
    }

    @Override
    public CompanyInfoDto getCompanyInfo() {
        CompanyInfo info = companyInfoRepository.findAll().stream().findFirst()
                .orElseThrow(() -> new ResourceNotFoundException("Company info has not been configured"));
        return toDto(info);
    }

    @Override
    public List<ServiceOfferingDto> listServiceOfferings() {
        return serviceOfferingRepository.findAllByOrderByStepNumberAsc().stream()
                .map(this::toDto)
                .toList();
    }

    @Override
    public ServiceOfferingDto getServiceOffering(Long id) {
        return toDto(findServiceOffering(id));
    }

    @Override
    @Transactional
    public ServiceOfferingDto createServiceOffering(ServiceOfferingRequest request) {
        ServiceOffering entity = ServiceOffering.builder()
                .stepNumber(request.getStepNumber())
                .title(request.getTitle())
                .description(request.getDescription())
                .deliverables(request.getDeliverables())
                .estimatedDuration(request.getEstimatedDuration())
                .status(request.getStatus() != null ? request.getStatus() : "ACTIVE")
                .build();
        return toDto(serviceOfferingRepository.save(entity));
    }

    @Override
    @Transactional
    public ServiceOfferingDto updateServiceOffering(Long id, ServiceOfferingRequest request) {
        ServiceOffering entity = findServiceOffering(id);
        entity.setStepNumber(request.getStepNumber());
        entity.setTitle(request.getTitle());
        entity.setDescription(request.getDescription());
        entity.setDeliverables(request.getDeliverables());
        entity.setEstimatedDuration(request.getEstimatedDuration());
        if (request.getStatus() != null) {
            entity.setStatus(request.getStatus());
        }
        return toDto(serviceOfferingRepository.save(entity));
    }

    @Override
    @Transactional
    public void deleteServiceOffering(Long id) {
        ServiceOffering entity = findServiceOffering(id);
        serviceOfferingRepository.delete(entity);
    }

    private ServiceOffering findServiceOffering(Long id) {
        return serviceOfferingRepository.findById(id)
                .orElseThrow(() -> ResourceNotFoundException.forEntity("Service offering", id));
    }

    private CompanyInfoDto toDto(CompanyInfo entity) {
        return CompanyInfoDto.builder()
                .id(entity.getId())
                .name(entity.getName())
                .tagline(entity.getTagline())
                .mission(entity.getMission())
                .vision(entity.getVision())
                .founderBio(entity.getFounderBio())
                .address(entity.getAddress())
                .phone(entity.getPhone())
                .email(entity.getEmail())
                .linkedinUrl(entity.getLinkedinUrl())
                .build();
    }

    private ServiceOfferingDto toDto(ServiceOffering entity) {
        return ServiceOfferingDto.builder()
                .id(entity.getId())
                .stepNumber(entity.getStepNumber())
                .title(entity.getTitle())
                .description(entity.getDescription())
                .deliverables(entity.getDeliverables())
                .estimatedDuration(entity.getEstimatedDuration())
                .status(entity.getStatus())
                .build();
    }
}


-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: service:impl
package com.ais.content.service.impl;

import com.ais.common.exception.ResourceNotFoundException;
import com.ais.content.dto.ContactMessageDto;
import com.ais.content.dto.ContactMessageRequest;
import com.ais.content.dto.TestimonialDto;
import com.ais.content.dto.TestimonialRequest;
import com.ais.content.entity.ContactMessage;
import com.ais.content.entity.Testimonial;
import com.ais.content.repository.ContactMessageRepository;
import com.ais.content.repository.TestimonialRepository;
import com.ais.content.service.EngagementFeedbackService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;
import java.util.List;

@Service
public class EngagementFeedbackServiceImpl implements EngagementFeedbackService {

    private static final Logger log = LoggerFactory.getLogger(EngagementFeedbackServiceImpl.class);

    private final TestimonialRepository testimonialRepository;
    private final ContactMessageRepository contactMessageRepository;

    public EngagementFeedbackServiceImpl(TestimonialRepository testimonialRepository,
                                          ContactMessageRepository contactMessageRepository) {
        this.testimonialRepository = testimonialRepository;
        this.contactMessageRepository = contactMessageRepository;
    }

    @Override
    public List<TestimonialDto> listTestimonials() {
        return testimonialRepository.findAll().stream().map(this::toDto).toList();
    }

    @Override
    @Transactional
    public TestimonialDto createTestimonial(TestimonialRequest request) {
        Testimonial entity = Testimonial.builder()
                .clientName(request.getClientName())
                .organization(request.getOrganization())
                .designation(request.getDesignation())
                .testimonialText(request.getTestimonialText())
                .rating(request.getRating())
                .build();
        return toDto(testimonialRepository.save(entity));
    }

    @Override
    @Transactional
    public TestimonialDto updateTestimonial(Long id, TestimonialRequest request) {
        Testimonial entity = testimonialRepository.findById(id)
                .orElseThrow(() -> ResourceNotFoundException.forEntity("Testimonial", id));
        entity.setClientName(request.getClientName());
        entity.setOrganization(request.getOrganization());
        entity.setDesignation(request.getDesignation());
        entity.setTestimonialText(request.getTestimonialText());
        entity.setRating(request.getRating());
        return toDto(testimonialRepository.save(entity));
    }

    @Override
    @Transactional
    public void deleteTestimonial(Long id) {
        Testimonial entity = testimonialRepository.findById(id)
                .orElseThrow(() -> ResourceNotFoundException.forEntity("Testimonial", id));
        testimonialRepository.delete(entity);
    }

    @Override
    @Transactional
    public ContactMessageDto submitContactMessage(ContactMessageRequest request) {
        ContactMessage entity = ContactMessage.builder()
                .name(request.getName())
                .email(request.getEmail())
                .mobile(request.getMobile())
                .subject(request.getSubject())
                .message(request.getMessage())
                .createdDate(Instant.now())
                .build();
        ContactMessage saved = contactMessageRepository.save(entity);
        log.info("Contact message received from '{}'", saved.getEmail());
        return toDto(saved);
    }

    @Override
    public Page<ContactMessageDto> listContactMessages(Pageable pageable) {
        return contactMessageRepository.findAllByOrderByCreatedDateDesc(pageable).map(this::toDto);
    }

    private TestimonialDto toDto(Testimonial entity) {
        return TestimonialDto.builder()
                .id(entity.getId())
                .clientName(entity.getClientName())
                .organization(entity.getOrganization())
                .designation(entity.getDesignation())
                .testimonialText(entity.getTestimonialText())
                .rating(entity.getRating())
                .build();
    }

    private ContactMessageDto toDto(ContactMessage entity) {
        return ContactMessageDto.builder()
                .id(entity.getId())
                .name(entity.getName())
                .email(entity.getEmail())
                .mobile(entity.getMobile())
                .subject(entity.getSubject())
                .message(entity.getMessage())
                .createdDate(entity.getCreatedDate())
                .build();
    }
}


-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: service:impl
package com.ais.content.service.impl;

import com.ais.common.exception.ResourceNotFoundException;
import com.ais.content.dto.*;
import com.ais.content.entity.CaseStudy;
import com.ais.content.entity.Industry;
import com.ais.content.entity.UseCase;
import com.ais.content.repository.CaseStudyRepository;
import com.ais.content.repository.IndustryRepository;
import com.ais.content.repository.UseCaseRepository;
import com.ais.content.service.IndustryCatalogService;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;
import java.util.List;

@Service
public class IndustryCatalogServiceImpl implements IndustryCatalogService {

    private final IndustryRepository industryRepository;
    private final UseCaseRepository useCaseRepository;
    private final CaseStudyRepository caseStudyRepository;

    public IndustryCatalogServiceImpl(IndustryRepository industryRepository,
                                       UseCaseRepository useCaseRepository,
                                       CaseStudyRepository caseStudyRepository) {
        this.industryRepository = industryRepository;
        this.useCaseRepository = useCaseRepository;
        this.caseStudyRepository = caseStudyRepository;
    }

    @Override
    public List<IndustryDto> listIndustries() {
        return industryRepository.findAll().stream().map(this::toDto).toList();
    }

    @Override
    public IndustryDto getIndustry(Long id) {
        return toDto(findIndustry(id));
    }

    @Override
    @Transactional
    public IndustryDto createIndustry(IndustryRequest request) {
        Industry entity = Industry.builder()
                .name(request.getName())
                .iconUrl(request.getIconUrl())
                .description(request.getDescription())
                .painPoints(request.getPainPoints())
                .howAiHelps(request.getHowAiHelps())
                .build();
        return toDto(industryRepository.save(entity));
    }

    @Override
    @Transactional
    public IndustryDto updateIndustry(Long id, IndustryRequest request) {
        Industry entity = findIndustry(id);
        entity.setName(request.getName());
        entity.setIconUrl(request.getIconUrl());
        entity.setDescription(request.getDescription());
        entity.setPainPoints(request.getPainPoints());
        entity.setHowAiHelps(request.getHowAiHelps());
        return toDto(industryRepository.save(entity));
    }

    @Override
    @Transactional
    public void deleteIndustry(Long id) {
        industryRepository.delete(findIndustry(id));
    }

    @Override
    @Transactional(readOnly = true)
    public List<UseCaseDto> listUseCases(Long industryId) {
        List<UseCase> useCases = industryId != null
                ? useCaseRepository.findByIndustryId(industryId)
                : useCaseRepository.findAll();
        return useCases.stream().map(this::toDto).toList();
    }

    @Override
    @Transactional(readOnly = true)
    public UseCaseDto getUseCase(Long id) {
        return toDto(findUseCase(id));
    }

    @Override
    @Transactional
    public UseCaseDto createUseCase(UseCaseRequest request) {
        Industry industry = findIndustry(request.getIndustryId());
        UseCase entity = UseCase.builder()
                .industry(industry)
                .name(request.getName())
                .problemStatement(request.getProblemStatement())
                .proposedSolution(request.getProposedSolution())
                .expectedRoi(request.getExpectedRoi())
                .build();
        return toDto(useCaseRepository.save(entity));
    }

    @Override
    @Transactional
    public UseCaseDto updateUseCase(Long id, UseCaseRequest request) {
        UseCase entity = findUseCase(id);
        entity.setIndustry(findIndustry(request.getIndustryId()));
        entity.setName(request.getName());
        entity.setProblemStatement(request.getProblemStatement());
        entity.setProposedSolution(request.getProposedSolution());
        entity.setExpectedRoi(request.getExpectedRoi());
        return toDto(useCaseRepository.save(entity));
    }

    @Override
    @Transactional
    public void deleteUseCase(Long id) {
        useCaseRepository.delete(findUseCase(id));
    }

    @Override
    @Transactional(readOnly = true)
    public List<CaseStudyDto> listCaseStudies() {
        return caseStudyRepository.findAll().stream().map(this::toDto).toList();
    }

    @Override
    @Transactional(readOnly = true)
    public CaseStudyDto getCaseStudy(Long id) {
        return toDto(findCaseStudy(id));
    }

    @Override
    @Transactional
    public CaseStudyDto createCaseStudy(CaseStudyRequest request) {
        CaseStudy entity = CaseStudy.builder()
                .industry(request.getIndustryId() != null ? findIndustry(request.getIndustryId()) : null)
                .clientType(request.getClientType())
                .challenge(request.getChallenge())
                .solution(request.getSolution())
                .techStack(request.getTechStack())
                .results(request.getResults())
                .publishedDate(Instant.now())
                .build();
        return toDto(caseStudyRepository.save(entity));
    }

    @Override
    @Transactional
    public CaseStudyDto updateCaseStudy(Long id, CaseStudyRequest request) {
        CaseStudy entity = findCaseStudy(id);
        entity.setIndustry(request.getIndustryId() != null ? findIndustry(request.getIndustryId()) : null);
        entity.setClientType(request.getClientType());
        entity.setChallenge(request.getChallenge());
        entity.setSolution(request.getSolution());
        entity.setTechStack(request.getTechStack());
        entity.setResults(request.getResults());
        return toDto(caseStudyRepository.save(entity));
    }

    @Override
    @Transactional
    public void deleteCaseStudy(Long id) {
        caseStudyRepository.delete(findCaseStudy(id));
    }

    private Industry findIndustry(Long id) {
        return industryRepository.findById(id)
                .orElseThrow(() -> ResourceNotFoundException.forEntity("Industry", id));
    }

    private UseCase findUseCase(Long id) {
        return useCaseRepository.findById(id)
                .orElseThrow(() -> ResourceNotFoundException.forEntity("Use case", id));
    }

    private CaseStudy findCaseStudy(Long id) {
        return caseStudyRepository.findById(id)
                .orElseThrow(() -> ResourceNotFoundException.forEntity("Case study", id));
    }

    private IndustryDto toDto(Industry entity) {
        return IndustryDto.builder()
                .id(entity.getId())
                .name(entity.getName())
                .iconUrl(entity.getIconUrl())
                .description(entity.getDescription())
                .painPoints(entity.getPainPoints())
                .howAiHelps(entity.getHowAiHelps())
                .build();
    }

    private UseCaseDto toDto(UseCase entity) {
        return UseCaseDto.builder()
                .id(entity.getId())
                .industryId(entity.getIndustry() != null ? entity.getIndustry().getId() : null)
                .industryName(entity.getIndustry() != null ? entity.getIndustry().getName() : null)
                .name(entity.getName())
                .problemStatement(entity.getProblemStatement())
                .proposedSolution(entity.getProposedSolution())
                .expectedRoi(entity.getExpectedRoi())
                .build();
    }

    private CaseStudyDto toDto(CaseStudy entity) {
        return CaseStudyDto.builder()
                .id(entity.getId())
                .industryId(entity.getIndustry() != null ? entity.getIndustry().getId() : null)
                .industryName(entity.getIndustry() != null ? entity.getIndustry().getName() : null)
                .clientType(entity.getClientType())
                .challenge(entity.getChallenge())
                .solution(entity.getSolution())
                .techStack(entity.getTechStack())
                .results(entity.getResults())
                .publishedDate(entity.getPublishedDate())
                .build();
    }
}



-------------------------------------
ais: backend: ais-content-service: src\main\java\com\ais\content: 
package com.ais.content;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class AisContentServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(AisContentServiceApplication.class, args);
    }
}
-------------------------------------
ais: backend: ais-content-service: src\main\resources: db:migration
V1__init.sql
CREATE TABLE IF NOT EXISTS company_info (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    tagline VARCHAR(500),
    mission VARCHAR(2000),
    vision VARCHAR(2000),
    founder_bio TEXT,
    address VARCHAR(500),
    phone VARCHAR(50),
    email VARCHAR(255),
    linkedin_url VARCHAR(255)
);

CREATE TABLE IF NOT EXISTS service_offering (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    step_number INT NOT NULL,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    deliverables VARCHAR(2000),
    estimated_duration VARCHAR(100),
    status VARCHAR(20) DEFAULT 'ACTIVE'
);

CREATE TABLE IF NOT EXISTS industry (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE,
    icon_url VARCHAR(500),
    description TEXT,
    pain_points TEXT,
    how_ai_helps TEXT
);

CREATE TABLE IF NOT EXISTS use_case (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    industry_id BIGINT NOT NULL REFERENCES industry(id),
    name VARCHAR(255) NOT NULL,
    problem_statement TEXT,
    proposed_solution TEXT,
    expected_roi VARCHAR(500)
);

CREATE TABLE IF NOT EXISTS case_study (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    industry_id BIGINT REFERENCES industry(id),
    client_type VARCHAR(255),
    challenge TEXT,
    solution TEXT,
    tech_stack VARCHAR(500),
    results TEXT,
    published_date TIMESTAMP
);

CREATE TABLE IF NOT EXISTS testimonial (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    client_name VARCHAR(255) NOT NULL,
    organization VARCHAR(255),
    designation VARCHAR(255),
    testimonial_text TEXT NOT NULL,
    rating INT CHECK (rating BETWEEN 1 AND 5)
);

CREATE TABLE IF NOT EXISTS contact_message (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL,
    mobile VARCHAR(20),
    subject VARCHAR(255),
    message TEXT NOT NULL,
    created_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_use_case_industry ON use_case (industry_id);
CREATE INDEX IF NOT EXISTS idx_case_study_industry ON case_study (industry_id);
CREATE INDEX IF NOT EXISTS idx_contact_message_created ON contact_message (created_date);

-------------------------------------
ais: backend: ais-content-service: src\main\resources: db:migration
V2__seed_data.sql
INSERT INTO company_info (name, tagline, mission, vision, founder_bio, address, phone, email, linkedin_url) VALUES
('AIS - Artificial Intelligent Solutions',
 'Building Smart, AI-Enabled Organizations',
 'To help colleges, schools, factories, and shopping malls increase ROI, speed up operations, and reduce dependency on manual workforce through custom AI agents.',
 'A world where every organization has an AI agent workforce that amplifies human potential instead of replacing it.',
 'Founder of AIS is a Java Solution Architect who has designed and built end-to-end web applications, enterprise cloud-based applications, and Data Processing ETL/ELT applications from scratch, now offering Solution Architecture Consulting to organizations of every size.',
 'Bengaluru, India',
 '+91-98765-00000',
 'contact@ais-solutions.example.com',
 'https://www.linkedin.com/company/ais-solutions');

INSERT INTO service_offering (step_number, title, description, deliverables, estimated_duration, status) VALUES
(1, 'Design Phase', 'We start by understanding your organization and produce the architecture blueprint before writing a single line of code.', 'High Level Design (HLD), Low Level Design (LLD), Database Design, Security Design', '1-2 weeks', 'ACTIVE'),
(2, 'Documentation Phase', 'We translate the design into precise functional and technical specifications your team can review and sign off on.', 'Functional Specification, Technical Implementation Document', '1 week', 'ACTIVE'),
(3, 'Backend Development', 'We build production-grade microservices with authentication, authorization, logging, exception handling, and full test coverage.', 'Java 17 / Spring Boot 3.x Microservices, JWT Auth & RBAC, Centralized Logging, Global Exception Handling, JUnit Unit Tests, Integration Tests', '3-6 weeks', 'ACTIVE'),
(4, 'Frontend Development', 'We deliver a responsive Angular single-page application fully integrated with your backend microservices.', 'Angular SPA, Responsive UI, Accessibility, Full Backend Integration', '2-4 weeks', 'ACTIVE');

INSERT INTO industry (name, icon_url, description, pain_points, how_ai_helps) VALUES
('College', '/assets/icons/college.svg', 'Higher-education institutions managing admissions, students, faculty, and placements.', 'Manual admission follow-ups, overloaded helpdesk staff, slow placement coordination.', 'AI agents automate admissions triage, answer routine student queries 24/7, and match students to placement opportunities.'),
('School', '/assets/icons/school.svg', 'K-12 schools managing attendance, fees, and parent communication.', 'Manual attendance registers, delayed fee reminders, inconsistent parent communication.', 'AI agents automate attendance capture, send proactive fee reminders, and keep parents informed in real time.'),
('Factory', '/assets/icons/factory.svg', 'Manufacturing plants managing machinery, inventory, and quality control.', 'Unplanned machine downtime, manual inventory tracking, inconsistent quality inspection.', 'AI agents predict maintenance needs, optimize inventory levels, and automate visual quality inspection.'),
('Shopping Mall', '/assets/icons/mall.svg', 'Retail malls managing tenants, customers, and security operations.', 'Generic customer service, limited footfall insight, reactive security monitoring.', 'AI agents power customer assistant chatbots, surface footfall/sales analytics, and enable proactive security monitoring.');

INSERT INTO use_case (industry_id, name, problem_statement, proposed_solution, expected_roi) VALUES
(1, 'Admission Agent', 'Admission staff spend hours manually triaging inquiries.', 'An AI agent that auto-responds, qualifies leads, and schedules counselor calls.', '30-40% reduction in admission staff workload'),
(1, 'Student Helpdesk Agent', 'Students wait days for answers to routine questions.', 'A 24/7 conversational agent answering FAQs and escalating complex cases.', 'Faster resolution, higher student satisfaction'),
(1, 'Placement Agent', 'Manual matching of students to job openings is slow.', 'An AI agent that matches student profiles to openings and notifies both sides.', 'Higher placement rate, faster turnaround'),
(1, 'Parent Communication Agent', 'Parents lack timely visibility into student progress.', 'An AI agent sending automated, personalized progress updates.', 'Improved parent satisfaction and retention'),
(2, 'Admission Agent', 'Front-office staff overwhelmed during admission season.', 'An AI agent handling initial inquiries and document checklists.', 'Reduced front-office overtime costs'),
(2, 'Attendance & Parent Notification Agent', 'Manual attendance registers and delayed parent alerts.', 'An AI agent automating attendance capture and real-time parent alerts.', 'Near-zero manual attendance effort'),
(2, 'Fee Reminder Agent', 'Delayed fee payments due to inconsistent reminders.', 'An AI agent sending automated, escalating fee reminders.', 'Improved on-time fee collection rate'),
(3, 'Predictive Maintenance Agent', 'Unplanned downtime from unexpected machine failures.', 'An AI agent monitoring sensor data to predict failures before they happen.', '20-30% reduction in unplanned downtime'),
(3, 'Inventory & Supply Chain Agent', 'Manual inventory tracking causes stockouts and overstock.', 'An AI agent forecasting demand and automating reorder points.', 'Lower carrying costs, fewer stockouts'),
(3, 'Quality Inspection Agent', 'Manual visual inspection is slow and inconsistent.', 'A computer-vision AI agent automating defect detection.', 'Higher defect detection accuracy, faster throughput'),
(3, 'Safety Compliance Agent', 'Manual safety audits miss real-time hazards.', 'An AI agent monitoring compliance and flagging hazards in real time.', 'Fewer safety incidents, lower compliance risk'),
(4, 'Customer Assistant Agent', 'Generic customer service does not scale during peak hours.', 'A conversational AI agent guiding shoppers to stores, offers, and events.', 'Higher customer satisfaction and dwell time'),
(4, 'Footfall & Sales Analytics Agent', 'Limited visibility into footfall and tenant sales trends.', 'An AI agent aggregating footfall and sales data into actionable insights.', 'Data-driven leasing and marketing decisions'),
(4, 'Tenant Management Agent', 'Manual coordination with tenants on operations and billing.', 'An AI agent automating tenant communication and billing reminders.', 'Reduced administrative overhead'),
(4, 'Security Monitoring Agent', 'Reactive, manually-monitored security camera feeds.', 'An AI agent providing real-time anomaly detection and alerts.', 'Faster incident response, improved safety');

INSERT INTO case_study (industry_id, client_type, challenge, solution, tech_stack, results, published_date) VALUES
(1, 'College', 'A mid-sized college struggled to keep up with admission inquiries during peak season.', 'AIS designed and built an Admission Agent microservice integrated with the college website and CRM.', 'Java 17, Spring Boot 3.x, PostgreSQL, Angular 20', 'Admission response time reduced from 2 days to under 1 hour; 35% increase in inquiry-to-enrollment conversion.', '2026-05-01 00:00:00'),
(2, 'School', 'A K-12 school had inconsistent fee collection and manual attendance tracking.', 'AIS delivered an Attendance and Fee Reminder Agent with parent notification integration.', 'Java 17, Spring Boot 3.x, PostgreSQL, Angular 20', 'On-time fee collection improved by 25%; attendance capture time reduced by 90%.', '2026-05-10 00:00:00'),
(3, 'Factory', 'A manufacturing plant faced frequent unplanned downtime on critical CNC machines.', 'AIS built a Predictive Maintenance Agent consuming sensor telemetry to flag at-risk machines.', 'Java 17, Spring Boot 3.x, PostgreSQL, Angular 20', 'Unplanned downtime reduced by 28% within the first quarter.', '2026-05-18 00:00:00'),
(4, 'Shopping Mall', 'A shopping mall wanted better customer engagement and footfall insight.', 'AIS delivered a Customer Assistant Agent plus a Footfall Analytics dashboard.', 'Java 17, Spring Boot 3.x, PostgreSQL, Angular 20', 'Average customer dwell time increased by 15%; actionable footfall insights for tenant negotiations.', '2026-05-25 00:00:00'),
(1, 'College', 'A university placement cell manually matched hundreds of students to job openings.', 'AIS built a Placement Agent that automatically matches student profiles to postings.', 'Java 17, Spring Boot 3.x, PostgreSQL, Angular 20', 'Placement cell processing time reduced by 40%.', '2026-06-02 00:00:00'),
(3, 'Factory', 'A components manufacturer needed automated visual quality inspection.', 'AIS delivered a Quality Inspection Agent using computer vision for defect detection.', 'Java 17, Spring Boot 3.x, PostgreSQL, Angular 20', 'Defect detection accuracy improved by 18%; inspection throughput doubled.', '2026-06-08 00:00:00');

INSERT INTO testimonial (client_name, organization, designation, testimonial_text, rating) VALUES
('Dr. Anita Rao', 'Greenfield College', 'Director of Admissions', 'AIS transformed our admission process. What used to take days now takes minutes, and our conversion rate has never been higher.', 5),
('Mr. Vikram Shah', 'Sunrise Public School', 'Principal', 'The attendance and fee reminder agents freed up our front-office staff to focus on students instead of paperwork.', 5),
('Ms. Priya Menon', 'Nexus Auto Components', 'Plant Manager', 'The predictive maintenance agent paid for itself within the first quarter by preventing two major breakdowns.', 5),
('Mr. Arjun Kapoor', 'Skyline Mall Group', 'Operations Head', 'Our customers love the new assistant, and we finally have real data to guide our tenant mix decisions.', 4),
('Dr. Sameer Joshi', 'Horizon Institute of Technology', 'Dean', 'AIS delivered exactly what was promised - on time, well documented, and thoroughly tested.', 5),
('Mrs. Kavita Iyer', 'Bright Future Academy', 'Administrator', 'Professional, responsive, and technically excellent. Highly recommended for any AI agent initiative.', 5),
('Mr. Rohit Verma', 'Precision Tools Manufacturing', 'Quality Head', 'The quality inspection agent has measurably improved our defect detection rate.', 4),
('Ms. Sneha Kulkarni', 'Metro City Mall', 'Marketing Manager', 'Great experience working with the AIS team - clear communication throughout the engagement.', 4);

INSERT INTO contact_message (name, email, mobile, subject, message, created_date) VALUES
('Ramesh Kumar', 'ramesh.kumar@example.com', '+919812345001', 'General Inquiry', 'Interested in learning more about your AI agent services for educational institutions.', '2026-06-01 09:00:00'),
('Sunita Patel', 'sunita.patel@example.com', '+919812345002', 'Partnership Opportunity', 'We run a chain of schools and would like to discuss a partnership.', '2026-06-03 10:30:00'),
('Amit Desai', 'amit.desai@example.com', '+919812345003', 'Factory Automation', 'Looking for predictive maintenance solutions for our manufacturing unit.', '2026-06-05 11:15:00'),
('Neha Gupta', 'neha.gupta@example.com', '+919812345004', 'Mall Customer Experience', 'Interested in a customer assistant chatbot for our shopping mall.', '2026-06-07 14:20:00'),
('Suresh Iyer', 'suresh.iyer@example.com', '+919812345005', 'Career Opportunity', 'Do you have any open Solution Architect positions?', '2026-06-09 16:00:00'),
('Divya Nair', 'divya.nair@example.com', '+919812345006', 'General Inquiry', 'Can you share case studies relevant to higher education?', '2026-06-11 08:45:00'),
('Karan Malhotra', 'karan.malhotra@example.com', '+919812345007', 'Technical Question', 'What technology stack do you use for your microservices?', '2026-06-13 13:10:00'),
('Pooja Sharma', 'pooja.sharma@example.com', '+919812345008', 'General Inquiry', 'Requesting a callback to discuss AI agents for our school.', '2026-06-15 09:50:00'),
('Manoj Tiwari', 'manoj.tiwari@example.com', '+919812345009', 'Factory Automation', 'Interested in an inventory management AI agent.', '2026-06-17 15:30:00'),
('Anjali Reddy', 'anjali.reddy@example.com', '+919812345010', 'Mall Security', 'Can your security monitoring agent integrate with existing CCTV systems?', '2026-06-19 10:00:00'),
('Vivek Choudhary', 'vivek.choudhary@example.com', '+919812345011', 'General Inquiry', 'Would like a demo of your admission agent.', '2026-06-21 12:00:00'),
('Meera Pillai', 'meera.pillai@example.com', '+919812345012', 'Partnership Opportunity', 'Representing an ed-tech company interested in co-developing agents.', '2026-06-23 09:15:00'),
('Sanjay Bhatt', 'sanjay.bhatt@example.com', '+919812345013', 'Technical Question', 'Do you support on-premise deployment in addition to cloud?', '2026-06-25 14:40:00'),
('Rekha Menon', 'rekha.menon@example.com', '+919812345014', 'General Inquiry', 'Interested in the fee reminder agent for our school group.', '2026-06-27 11:25:00'),
('Arvind Rao', 'arvind.rao@example.com', '+919812345015', 'Factory Automation', 'What is the typical ROI timeline for a predictive maintenance agent?', '2026-06-29 16:50:00');

-------------------------------------
ais: backend: ais-content-service: src\main\resources: 
application-docker.yaml
spring:
  datasource:
    url: jdbc:postgresql://postgres:5432/ais_db
    username: ${DB_USERNAME:ais_content_user}
    password: ${DB_PASSWORD:ais_content_pass}

-------------------------------------
ais: backend: ais-content-service: src\main\resources: 
application-gcp.yaml
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  jpa:
    show-sql: false

feature:
  analytics-enabled: true

logging:
  level:
    com.ais: INFO

-------------------------------------
ais: backend: ais-content-service: src\main\resources: 
application-h2.yaml
# Local, no-install-required profile: runs against an in-memory H2 database
# (PostgreSQL compatibility mode) instead of a real PostgreSQL server/Docker.
# Start with: mvn spring-boot:run -Dspring-boot.run.profiles=h2
spring:
  datasource:
    url: jdbc:h2:mem:ais_content;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
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

-------------------------------------
ais: backend: ais-content-service: src\main\resources: 
application-local.yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/ais_db
    username: ais_content_user
    password: ais_content_pass
  jpa:
    show-sql: true

logging:
  level:
    com.ais: DEBUG

-------------------------------------
ais: backend: ais-content-service: src\main\resources: 
application.yaml
server:
  port: 8083

spring:
  application:
    name: ais-content-service
  jpa:
    hibernate:
      ddl-auto: validate
    open-in-view: false
  flyway:
    enabled: true
    schemas: content_schema
    default-schema: content_schema
    create-schemas: true
    locations: classpath:db/migration

app:
  jwt:
    secret: ${JWT_SECRET:local-dev-only-change-me-must-be-32-bytes-minimum}
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

-------------------------------------
ais: backend: ais-content-service: src\main\resources: 
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
-------------------------------------
ais: backend: ais-content-service:
Dockerfile
# Multi-stage build. Build context is the backend/ reactor root (see docker-compose.yml).
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /workspace
COPY pom.xml .
COPY ais-common ais-common
COPY ais-content-service ais-content-service
RUN mvn -q -pl ais-content-service -am -DskipTests package

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=build /workspace/ais-content-service/target/ais-content-service-*.jar app.jar
EXPOSE 8083
ENTRYPOINT ["java", "-jar", "app.jar"]

-------------------------------------
ais: backend: ais-content-service:
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

    <artifactId>ais-content-service</artifactId>
    <packaging>jar</packaging>
    <name>AIS Content Service</name>
    <description>Serves marketing content: company info, services, industries, use cases, case studies, testimonials, contact messages</description>

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
==========================================
ais: backend: ais-gateway-service: src\main\java\com\ais\gateway: config
package com.ais.gateway.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;

import java.util.ArrayList;
import java.util.List;

@Configuration
@ConfigurationProperties(prefix = "app.security")
public class GatewaySecurityProperties {

    private List<String> publicPaths = new ArrayList<>();

    public List<String> getPublicPaths() {
        return publicPaths;
    }

    public void setPublicPaths(List<String> publicPaths) {
        this.publicPaths = publicPaths;
    }
}

------------------------------
ais: backend: ais-gateway-service: src\main\java\com\ais\gateway: filter
package com.ais.gateway.filter;

import com.ais.common.security.JwtTokenValidator;
import com.ais.gateway.config.GatewaySecurityProperties;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.jsonwebtoken.JwtException;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.Ordered;
import org.springframework.core.io.buffer.DataBuffer;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.server.reactive.ServerHttpResponse;
import org.springframework.stereotype.Component;
import org.springframework.util.AntPathMatcher;
import org.springframework.web.server.ServerWebExchange;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import reactor.core.publisher.Mono;

import java.nio.charset.StandardCharsets;
import java.util.Map;

/**
 * Rejects requests to protected routes that lack a valid JWT before they ever reach a
 * downstream service. Downstream services still independently re-validate the JWT
 * (defense in depth, see SECURITY_DESIGN.md Section 3) - this filter is an edge
 * optimization, not the only line of defense.
 */
@Component
public class JwtValidationGlobalFilter implements GlobalFilter, Ordered {

    private final JwtTokenValidator jwtTokenValidator;
    private final GatewaySecurityProperties securityProperties;
    private final AntPathMatcher pathMatcher = new AntPathMatcher();
    private final ObjectMapper objectMapper = new ObjectMapper();

    public JwtValidationGlobalFilter(@Value("${app.jwt.secret}") String secret,
                                      GatewaySecurityProperties securityProperties) {
        this.jwtTokenValidator = new JwtTokenValidator(secret);
        this.securityProperties = securityProperties;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getURI().getPath();

        if (isPublicPath(path)) {
            return chain.filter(exchange);
        }

        String authHeader = exchange.getRequest().getHeaders().getFirst("Authorization");
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            return unauthorized(exchange, "Authentication is required to access this resource");
        }

        try {
            jwtTokenValidator.validateAndParse(authHeader.substring(7));
            return chain.filter(exchange);
        } catch (JwtException | IllegalArgumentException ex) {
            return unauthorized(exchange, "Invalid or expired token");
        }
    }

    private boolean isPublicPath(String path) {
        return securityProperties.getPublicPaths().stream()
                .anyMatch(pattern -> pathMatcher.match(pattern, path));
    }

    private Mono<Void> unauthorized(ServerWebExchange exchange, String message) {
        ServerHttpResponse response = exchange.getResponse();
        response.setStatusCode(HttpStatus.UNAUTHORIZED);
        response.getHeaders().setContentType(MediaType.APPLICATION_JSON);

        Map<String, Object> body = Map.of(
                "success", false,
                "errorCode", "UNAUTHORIZED",
                "message", message,
                "details", java.util.List.of());
        try {
            byte[] bytes = objectMapper.writeValueAsBytes(body);
            DataBuffer buffer = response.bufferFactory().wrap(bytes);
            return response.writeWith(Mono.just(buffer));
        } catch (Exception ex) {
            return response.setComplete();
        }
    }

    @Override
    public int getOrder() {
        return -1;
    }
}

----------------------------------------
ais: backend: ais-gateway-service: src\main\java\com\ais\gateway: 
package com.ais.gateway;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class AisGatewayServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(AisGatewayServiceApplication.class, args);
    }
}

----------------------------------------
ais: backend: ais-gateway-service: src\main\resources:
application-docker.yaml
spring:
  cloud:
    gateway:
      routes:
        - id: ais-auth-service
          uri: http://ais-auth-service:8081
          predicates:
            - Path=/api/v1/auth/**
        - id: ais-content-service
          uri: http://ais-content-service:8083
          predicates:
            - Path=/api/v1/content/**
        - id: ais-consultation-service
          uri: http://ais-consultation-service:8082
          predicates:
            - Path=/api/v1/consultation/**

----------------------------------------
ais: backend: ais-gateway-service: src\main\resources:
application-docker.yaml

----------------------------------------
ais: backend: ais-gateway-service: src\main\resources:
application-gcp.yaml
spring:
  cloud:
    gateway:
      routes:
        - id: ais-auth-service
          uri: ${AUTH_SERVICE_URI}
          predicates:
            - Path=/api/v1/auth/**
        - id: ais-content-service
          uri: ${CONTENT_SERVICE_URI}
          predicates:
            - Path=/api/v1/content/**
        - id: ais-consultation-service
          uri: ${CONSULTATION_SERVICE_URI}
          predicates:
            - Path=/api/v1/consultation/**

logging:
  level:
    com.ais: INFO

----------------------------------------
ais: backend: ais-gateway-service: src\main\resources:
application-local.yaml
logging:
  level:
    com.ais: DEBUG
    org.springframework.cloud.gateway: DEBUG

----------------------------------------
ais: backend: ais-gateway-service: src\main\resources:
application.yaml
server:
  port: 8080

spring:
  application:
    name: ais-gateway-service
  cloud:
    gateway:
      routes:
        - id: ais-auth-service
          uri: ${AUTH_SERVICE_URI:http://localhost:8081}
          predicates:
            - Path=/api/v1/auth/**
        - id: ais-content-service
          uri: ${CONTENT_SERVICE_URI:http://localhost:8083}
          predicates:
            - Path=/api/v1/content/**
        - id: ais-consultation-service
          uri: ${CONSULTATION_SERVICE_URI:http://localhost:8082}
          predicates:
            - Path=/api/v1/consultation/**
      globalcors:
        cors-configurations:
          '[/**]':
            allowedOrigins: ${CORS_ALLOWED_ORIGINS:http://localhost:4200}
            allowedMethods:
              - GET
              - POST
              - PUT
              - PATCH
              - DELETE
              - OPTIONS
            allowedHeaders:
              - Authorization
              - Content-Type
            allowCredentials: true

app:
  jwt:
    secret: ${JWT_SECRET:local-dev-only-change-me-must-be-32-bytes-minimum}
  security:
    # Paths that do not require a valid JWT at the gateway. Downstream services still
    # independently re-validate JWTs for protected endpoints (defense in depth).
    public-paths:
      - /api/v1/auth/**
      - /api/v1/content/home
      - /api/v1/content/about
      - /api/v1/content/services/**
      - /api/v1/content/industries/**
      - /api/v1/content/use-cases/**
      - /api/v1/content/case-studies/**
      - /api/v1/content/testimonials
      - /api/v1/content/contact
      - /api/v1/consultation/requests
      - /actuator/health

management:
  endpoints:
    web:
      exposure:
        include: health,info,gateway

----------------------------------------
ais: backend: ais-gateway-service: 
Dockerfile
# Multi-stage build. Build context is the backend/ reactor root (see docker-compose.yml).
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /workspace
COPY pom.xml .
COPY ais-common ais-common
COPY ais-gateway-service ais-gateway-service
RUN mvn -q -pl ais-gateway-service -am -DskipTests package

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=build /workspace/ais-gateway-service/target/ais-gateway-service-*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]


----------------------------------------
ais: backend: ais-gateway-service: 
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

    <artifactId>ais-gateway-service</artifactId>
    <packaging>jar</packaging>
    <name>AIS Gateway Service</name>
    <description>Single entry point for the AIS platform - routes to downstream microservices and validates JWTs at the edge</description>

    <dependencies>
        <dependency>
            <groupId>com.ais</groupId>
            <artifactId>ais-common</artifactId>
            <exclusions>
                <!-- ais-common pulls in the servlet-based spring-boot-starter-web; the gateway
                     is reactive (WebFlux) so we exclude it to avoid a mixed servlet/reactive
                     classpath, and rely on spring-cloud-starter-gateway's own web stack. -->
                <exclusion>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-web</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
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

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.projectreactor</groupId>
            <artifactId>reactor-test</artifactId>
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
=================================================
ais: backend:
pom.xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.ais</groupId>
    <artifactId>ais-backend</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>
    <name>AIS Backend Parent</name>
    <description>Parent reactor POM for the AIS (Artificial Intelligent Solutions) microservices</description>

    <properties>
        <java.version>17</java.version>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <spring-boot.version>3.3.4</spring-boot.version>
        <spring-cloud.version>2023.0.3</spring-cloud.version>
        <jjwt.version>0.12.6</jjwt.version>
        <mapstruct.version>1.6.2</mapstruct.version>
        <springdoc.version>2.6.0</springdoc.version>
        <testcontainers.version>1.20.1</testcontainers.version>
        <cloud-sql-socket-factory.version>1.19.1</cloud-sql-socket-factory.version>
    </properties>

    <modules>
        <module>ais-common</module>
        <module>ais-auth-service</module>
        <module>ais-content-service</module>
        <module>ais-consultation-service</module>
        <module>ais-gateway-service</module>
    </modules>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>org.testcontainers</groupId>
                <artifactId>testcontainers-bom</artifactId>
                <version>${testcontainers.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>

            <dependency>
                <groupId>com.ais</groupId>
                <artifactId>ais-common</artifactId>
                <version>${project.version}</version>
            </dependency>

            <dependency>
                <groupId>io.jsonwebtoken</groupId>
                <artifactId>jjwt-api</artifactId>
                <version>${jjwt.version}</version>
            </dependency>
            <dependency>
                <groupId>io.jsonwebtoken</groupId>
                <artifactId>jjwt-impl</artifactId>
                <version>${jjwt.version}</version>
                <scope>runtime</scope>
            </dependency>
            <dependency>
                <groupId>io.jsonwebtoken</groupId>
                <artifactId>jjwt-jackson</artifactId>
                <version>${jjwt.version}</version>
                <scope>runtime</scope>
            </dependency>

            <dependency>
                <groupId>org.mapstruct</groupId>
                <artifactId>mapstruct</artifactId>
                <version>${mapstruct.version}</version>
            </dependency>
            <dependency>
                <groupId>org.mapstruct</groupId>
                <artifactId>mapstruct-processor</artifactId>
                <version>${mapstruct.version}</version>
            </dependency>

            <dependency>
                <groupId>org.springdoc</groupId>
                <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
                <version>${springdoc.version}</version>
            </dependency>

            <!-- Lets each service's JDBC URL connect to Cloud SQL via a Unix socket using
                 the "cloudSqlInstance" query parameter, when running with the gcp profile
                 on Cloud Run. Not used by local/docker/h2 profiles. -->
            <dependency>
                <groupId>com.google.cloud.sql</groupId>
                <artifactId>postgres-socket-factory</artifactId>
                <version>${cloud-sql-socket-factory.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-maven-plugin</artifactId>
                    <version>${spring-boot.version}</version>
                </plugin>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <configuration>
                        <source>${java.version}</source>
                        <target>${java.version}</target>
                        <parameters>true</parameters>
                        <annotationProcessorPaths>
                            <path>
                                <groupId>org.projectlombok</groupId>
                                <artifactId>lombok</artifactId>
                            </path>
                            <path>
                                <groupId>org.mapstruct</groupId>
                                <artifactId>mapstruct-processor</artifactId>
                                <version>${mapstruct.version}</version>
                            </path>
                        </annotationProcessorPaths>
                    </configuration>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
</project>
========================================
