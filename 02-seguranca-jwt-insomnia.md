# Aula 2 — Administrador, JWT e Insomnia

As atividades podem ser continuadas após a aula. Siga a sequência e retome do último checkpoint concluído.

Continue no mesmo projeto da [aula 1](01-api-banco-crud.md). Referência final: [branch JWT](https://github.com/e24Dev/CaixaDeSugestoes/tree/09-jwt-bearer-token-impl), estado atual do projeto-base (incluindo a migration V4). A implementação aqui adapta PostgreSQL para MariaDB do XAMPP, mantendo camadas e contrato HTTP.

## 1. Retomada

Inicie MySQL pelo XAMPP. Confira GET http://localhost:8000/api/courses. Pare a API antes de substituir arquivos.

## Compatibilidade com o projeto de referência

O projeto-base usa PostgreSQL na porta 5432 e API na porta 8080. Esta atividade usa MariaDB do XAMPP na porta 3306 e API na porta 8000. As rotas, os corpos JSON e a autenticação são os mesmos; use a collection deste repositório ou ajuste baseUrl ao trocar de projeto. As migrations de PostgreSQL não devem ser copiadas diretamente para o XAMPP: aqui usamos AUTO_INCREMENT, TIMESTAMP(6), VARCHAR nos nomes únicos e chaves estrangeiras explícitas. O seed V4 contém o mesmo hash BCrypt nos dois projetos.

## 2. Conta administrativa e segurança

O administrador fica no banco com senha BCrypt. A migration V3 cria a tabela e a V4 insere a conta didática `admin` / `Admin@123456` somente quando não há administrador. As credenciais não ficam no main. Os arquivos abaixo acrescentam também JWT; copie todos antes de executar. A etapa Basic da referência é dispensada.

## Como acompanhar a construção

Use a organização de pastas e packages da [aula 1](01-api-banco-crud.md#como-acompanhar-a-construção). Crie os arquivos novos com os blocos completos. Para os arquivos existentes apresentados com blocos completos, substitua o conteúdo inteiro; preserve os arquivos que não aparecem nesta aula. Execute os comandos na raiz do projeto somente após concluir as oito etapas. Se já criou as classes antigas AdminSetupCommand, AdminPasswordHash, AdminProvisioningService ou AdminProvisioningIntegrationTest, exclua esses arquivos: foram substituídos pelo seed V4.

### Índice da construção

- [Etapa 1: Projeto e dependências](#etapa-1)
- [Etapa 2: Conexão e migrations](#etapa-2)
- [Etapa 3: Domínio e contratos](#etapa-3)
- [Etapa 4: Persistência JPA](#etapa-4)
- [Etapa 5: Casos de uso](#etapa-5)
- [Etapa 6: Autenticação e JWT](#etapa-6)
- [Etapa 7: Contrato HTTP e documentação](#etapa-7)
- [Etapa 8: Testes automatizados](#etapa-8)

<a id="etapa-1"></a>

## Etapa 1 — Projeto e dependências

### `.gitignore`

Acrescente estas regras ao final do `.gitignore` gerado na aula 1:

```text
# Configuração local
.env
.env.*
!.env.example

# Credenciais locais carregadas automaticamente pelo Spring Boot
config/application.properties

# Chave JWT local persistente
.local/
```

### `build.gradle`

Substitua o conteúdo inteiro deste arquivo (ou crie-o se ainda não existir). Preserve os demais arquivos.

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '4.1.1'
    id 'io.spring.dependency-management' version '1.1.7'
}
group = 'br.com.etecalbertoferes'
version = '0.0.1-SNAPSHOT'
java { toolchain { languageVersion = JavaLanguageVersion.of(17) } }
repositories { mavenCentral() }
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-webmvc'
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-oauth2-resource-server'
    implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:3.1.1'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-flyway'
    implementation 'org.flywaydb:flyway-mysql'
    runtimeOnly 'org.mariadb.jdbc:mariadb-java-client'
    testImplementation 'org.springframework.boot:spring-boot-starter-webmvc-test'
    testImplementation 'org.springframework.security:spring-security-test'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}
tasks.named('test') { useJUnitPlatform() }
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/CaixaDeSugestoesApplication.java`

Substitua o conteúdo inteiro deste arquivo (ou crie-o se ainda não existir). Preserve os demais arquivos. O main apenas inicia o Spring Boot: não contém credenciais nem comandos de cadastro.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class CaixaDeSugestoesApplication {
    public static void main(String[] args) {
        SpringApplication.run(CaixaDeSugestoesApplication.class, args);
    }
}
```

<a id="etapa-2"></a>

## Etapa 2 — Conexão e migrations

Use os bancos e a pasta `db/migration` da aula 1. Acrescente as migrations V3 e V4; preserve V1, V2 e os dados existentes.

### `src/main/resources/application.yaml`

Substitua o conteúdo inteiro deste arquivo (ou crie-o se ainda não existir). Preserve os demais arquivos.

```yaml
server:
  port: ${SERVER_PORT:8000}
spring:
  application:
    name: CaixaDeSugestoes
  datasource:
    url: ${DB_URL:jdbc:mariadb://localhost:3306/suggestions}
    username: ${DB_USERNAME:root}
    password: ${DB_PASSWORD:}
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        jdbc:
          time_zone: UTC
  flyway:
    enabled: true
springdoc:
  swagger-ui:
    path: /swagger-ui.html
  api-docs:
    path: /v3/api-docs
app:
  jwt:
    secret-base64: ${JWT_SECRET_BASE64:}
    key-file: ${JWT_KEY_FILE:.local/jwt-signing.key}
    ttl-seconds: ${JWT_TTL_SECONDS:900}
```

A V3 abaixo mantém o conteúdo da versão anterior do tutorial para preservar o checksum de quem já a executou. Seu comentário final descreve o fluxo antigo; nesta versão, o cadastro é feito pela nova V4, não por comando interativo. Se você já aplicou V3, preserve seu arquivo e crie somente V4.

### `src/main/resources/db/migration/V3__create_administrators.sql`

Substitua o conteúdo inteiro deste arquivo (ou crie-o se ainda não existir). Preserve os demais arquivos.

```sql
CREATE TABLE administrators (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(100) NOT NULL UNIQUE CHECK (length(trim(username)) > 0),
    password_hash VARCHAR(60) NOT NULL,
    active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP
);
-- A primeira conta é provisionada interativamente, nunca por seed de senha.
```

### `src/main/resources/db/migration/V4__seed_initial_administrator.sql`

Crie este arquivo. Insere a conta didática somente se a tabela administrativa estiver vazia. A senha permanece apenas como hash BCrypt no banco. Esta conta é exclusiva da atividade local.

```sql
-- Conta didática local: admin / Admin@123456. Apenas o hash BCrypt é persistido.
-- Uma base com administrador cadastrado é preservada.
INSERT INTO administrators (username, password_hash, active)
SELECT 'admin', '$2a$12$V1POzMK.w8.31MWqErlWWOqtDKTj2.5rSz9/5tIcFtxtmC.tmHKdG', TRUE
WHERE NOT EXISTS (SELECT 1 FROM administrators);
```

<a id="etapa-3"></a>

## Etapa 3 — Domínio e contratos

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/domain/AdminAccount.java`

Substitua o conteúdo inteiro deste arquivo (ou crie-o se ainda não existir). Preserve os demais arquivos.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.domain;

public record AdminAccount(Long id, String username, String passwordHash, boolean active) {
    @Override public String toString() { return "AdminAccount[id=" + id + ", active=" + active + "]"; }
}
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/domain/AdminAccountRepository.java`

Substitua o conteúdo inteiro deste arquivo (ou crie-o se ainda não existir). Preserve os demais arquivos.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.domain;

import java.util.Optional;
public interface AdminAccountRepository {
    Optional<AdminAccount> findByUsername(String username);
}
```

<a id="etapa-4"></a>

## Etapa 4 — Persistência JPA

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/infrastructure/persistence/AdminAccountEntity.java`

Substitua o conteúdo inteiro deste arquivo (ou crie-o se ainda não existir). Preserve os demais arquivos.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.infrastructure.persistence;

import jakarta.persistence.*;
@Entity
@Table(name = "administrators")
public class AdminAccountEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;
    @Column(nullable = false, unique = true, length = 100) String username;
    @Column(name = "password_hash", nullable = false, length = 60) String passwordHash;
    @Column(nullable = false) boolean active;
    protected AdminAccountEntity() {}
}
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/infrastructure/persistence/AdminAccountJpaRepository.java`

Substitua o conteúdo inteiro deste arquivo (ou crie-o se ainda não existir). Preserve os demais arquivos.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.infrastructure.persistence;

import java.util.Optional;
import org.springframework.data.jpa.repository.JpaRepository;
interface AdminAccountJpaRepository extends JpaRepository<AdminAccountEntity, Long> {
    Optional<AdminAccountEntity> findByUsername(String username);
}
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/infrastructure/persistence/AdminAccountPersistenceAdapter.java`

Substitua o conteúdo inteiro deste arquivo (ou crie-o se ainda não existir). Preserve os demais arquivos.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.infrastructure.persistence;

import java.util.Optional;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;
import br.com.etecalbertoferes.CaixaDeSugestoes.domain.*;
@Repository
@Transactional(readOnly = true)
public class AdminAccountPersistenceAdapter implements AdminAccountRepository {
    private final AdminAccountJpaRepository repository;
    public AdminAccountPersistenceAdapter(AdminAccountJpaRepository repository) {
        this.repository = repository;
    }
    public Optional<AdminAccount> findByUsername(String username) {
        return repository.findByUsername(username).map(e -> new AdminAccount(e.id, e.username, e.passwordHash, e.active));
    }
}
```

<a id="etapa-5"></a>

## Etapa 5 — Casos de uso

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/application/AdminAuthenticationService.java`

Substitua o conteúdo inteiro deste arquivo (ou crie-o se ainda não existir). Preserve os demais arquivos.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.application;

import java.time.Instant;
import java.util.List;
import java.util.UUID;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.security.authentication.*;
import org.springframework.security.oauth2.jwt.*;
import org.springframework.security.oauth2.jose.jws.MacAlgorithm;

@Service
public class AdminAuthenticationService {
    public static final String ISSUER = "caixa-de-sugestoes";
    public static final String AUDIENCE = "caixa-de-sugestoes-api";
    private final AuthenticationManager authenticationManager;
    private final JwtEncoder encoder;
    private final long ttl;
    public AdminAuthenticationService(AuthenticationManager authenticationManager, JwtEncoder encoder,
            @Value("${app.jwt.ttl-seconds:900}") long ttl) {
        if (ttl < 60 || ttl > 3600) throw new IllegalArgumentException("JWT_TTL_SECONDS deve estar entre 60 e 3600");
        this.authenticationManager = authenticationManager; this.encoder = encoder; this.ttl = ttl;
    }
    public AccessToken login(String username, String password) {
        if (password.getBytes(java.nio.charset.StandardCharsets.UTF_8).length > 72)
            throw new BadCredentialsException("Credenciais inválidas");
        var authentication = authenticationManager.authenticate(UsernamePasswordAuthenticationToken.unauthenticated(username, password));
        if (authentication.getAuthorities().stream().noneMatch(a -> a.getAuthority().equals("ROLE_ADMIN")))
            throw new BadCredentialsException("Credenciais inválidas");
        Instant now = Instant.now();
        JwtClaimsSet claims = JwtClaimsSet.builder().issuer(ISSUER).audience(List.of(AUDIENCE))
            .subject(authentication.getName()).issuedAt(now).expiresAt(now.plusSeconds(ttl))
            .id(UUID.randomUUID().toString()).claim("roles", List.of("ADMIN")).build();
        String token = encoder.encode(JwtEncoderParameters.from(JwsHeader.with(MacAlgorithm.HS256).build(), claims)).getTokenValue();
        return new AccessToken(token, "Bearer", ttl);
    }
    public record AccessToken(String accessToken, String tokenType, long expiresIn) {}
}
```


<a id="etapa-6"></a>

## Etapa 6 — Autenticação e JWT

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/infrastructure/security/AdminConfiguration.java`

Substitua o conteúdo inteiro deste arquivo (ou crie-o se ainda não existir). Preserve os demais arquivos. O Spring Security usa estas configurações para autenticar a conta e validar o JWT.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.infrastructure.security;

import org.springframework.context.annotation.*;
import org.springframework.security.authentication.*;
import org.springframework.security.authentication.dao.DaoAuthenticationProvider;
import org.springframework.security.core.userdetails.*;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import br.com.etecalbertoferes.CaixaDeSugestoes.domain.AdminAccountRepository;
@Configuration
public class AdminConfiguration {
    @Bean PasswordEncoder passwordEncoder() { return new BCryptPasswordEncoder(12); }
    @Bean UserDetailsService adminUsers(AdminAccountRepository accounts) {
        return username -> {
            var account = accounts.findByUsername(username)
                .orElseThrow(() -> new UsernameNotFoundException("Credenciais inválidas"));
            return User.withUsername(account.username()).password(account.passwordHash())
                .disabled(!account.active()).roles("ADMIN").build();
        };
    }
    @Bean AuthenticationManager authenticationManager(UserDetailsService users, PasswordEncoder encoder) {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider(users);
        provider.setPasswordEncoder(encoder);
        return new ProviderManager(provider);
    }
}
```



### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/infrastructure/security/JwtConfiguration.java`

Substitua o conteúdo inteiro deste arquivo (ou crie-o se ainda não existir). Preserve os demais arquivos. O Spring Security usa estas configurações para autenticar a conta e validar o JWT.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.infrastructure.security;

import java.time.Duration;
import java.time.Instant;
import java.util.Base64;
import java.util.List;
import javax.crypto.SecretKey;
import javax.crypto.spec.SecretKeySpec;
import org.springframework.context.annotation.*;
import org.springframework.core.env.Environment;
import java.nio.file.Path;
import br.com.etecalbertoferes.CaixaDeSugestoes.domain.AdminAccountRepository;
import org.springframework.security.oauth2.core.*;
import org.springframework.security.oauth2.jose.jws.MacAlgorithm;
import org.springframework.security.oauth2.jwt.*;
import br.com.etecalbertoferes.CaixaDeSugestoes.application.AdminAuthenticationService;

@Configuration
public class JwtConfiguration {
    @Bean
    SecretKey jwtSigningKey(Environment env) {
        byte[] decoded;
        try {
            String configured = env.getProperty("app.jwt.secret-base64", "");
            String secret = configured.isBlank()
                ? JwtKeyStore.loadOrCreate(Path.of(env.getProperty("app.jwt.key-file", ".local/jwt-signing.key")))
                : configured;
            decoded = Base64.getDecoder().decode(secret);
        } catch (IllegalArgumentException ex) {
            throw new IllegalStateException("Configure JWT_SECRET_BASE64 com uma chave Base64 válida");
        }
        if (decoded.length < 32)
            throw new IllegalStateException("JWT_SECRET_BASE64 deve representar pelo menos 32 bytes aleatórios");
        return new SecretKeySpec(decoded, "HmacSHA256");
    }

    @Bean
    JwtEncoder jwtEncoder(SecretKey key) {
        return NimbusJwtEncoder.withSecretKey(key).algorithm(MacAlgorithm.HS256).build();
    }

    @Bean
    JwtDecoder jwtDecoder(SecretKey key, AdminAccountRepository accounts) {
        NimbusJwtDecoder decoder = NimbusJwtDecoder.withSecretKey(key).macAlgorithm(MacAlgorithm.HS256).build();
        OAuth2TokenValidator<Jwt> requiredTimes = jwt -> {
            if (jwt.getExpiresAt() == null || jwt.getIssuedAt() == null
                    || jwt.getIssuedAt().isAfter(Instant.now().plusSeconds(30)))
                return OAuth2TokenValidatorResult
                        .failure(new OAuth2Error("invalid_token", "Claims temporais inválidas", null));
            return OAuth2TokenValidatorResult.success();
        };
        decoder.setJwtValidator(new DelegatingOAuth2TokenValidator<>(
                new JwtTimestampValidator(Duration.ZERO),
                new JwtIssuerValidator(AdminAuthenticationService.ISSUER),
                new JwtClaimValidator<List<String>>(JwtClaimNames.AUD,
                        aud -> aud != null && aud.contains(AdminAuthenticationService.AUDIENCE)),
                new JwtClaimValidator<String>(JwtClaimNames.SUB, subject -> subject != null &&
                    accounts.findByUsername(subject).filter(account -> account.active()).isPresent()),
                requiredTimes));
        return decoder;
    }
}
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/infrastructure/security/JwtKeyStore.java`

Substitua o conteúdo inteiro deste arquivo (ou crie-o se ainda não existir). Preserve os demais arquivos. O Spring Security usa estas configurações para autenticar a conta e validar o JWT.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.infrastructure.security;

import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;
import java.nio.charset.StandardCharsets;
import java.nio.file.*;
import java.nio.file.attribute.PosixFilePermissions;
import java.security.SecureRandom;
import java.util.Base64;
import java.util.Set;

public final class JwtKeyStore {
    private JwtKeyStore() {}
    public static synchronized String loadOrCreate(Path path) {
        try {
            Path absolute = path.toAbsolutePath();
            boolean posix = absolute.getFileSystem().supportedFileAttributeViews().contains("posix");
            if (posix) Files.createDirectories(absolute.getParent(), PosixFilePermissions.asFileAttribute(PosixFilePermissions.fromString("rwx------")));
            else Files.createDirectories(absolute.getParent());
            var options = Set.of(StandardOpenOption.CREATE, StandardOpenOption.READ, StandardOpenOption.WRITE);
            try (FileChannel channel = posix
                    ? FileChannel.open(absolute, options, PosixFilePermissions.asFileAttribute(PosixFilePermissions.fromString("rw-------")))
                    : FileChannel.open(absolute, options);
                 var lock = channel.lock()) {
                if (channel.size() == 0) {
                    byte[] bytes = new byte[32]; new SecureRandom().nextBytes(bytes);
                    byte[] text = Base64.getEncoder().encode(bytes);
                    ByteBuffer buffer = ByteBuffer.wrap(text);
                    while (buffer.hasRemaining()) channel.write(buffer);
                    channel.force(true);
                }
                if (channel.size() > 8192) throw new IllegalStateException("Arquivo de chave JWT inválido");
                channel.position(0);
                ByteBuffer buffer = ByteBuffer.allocate((int) channel.size());
                while (buffer.hasRemaining() && channel.read(buffer) != -1) { }
                return new String(buffer.array(), StandardCharsets.US_ASCII).strip();
            }
        } catch (IOException ex) {
            throw new IllegalStateException("Não foi possível ler ou gravar a chave JWT; configure JWT_KEY_FILE em uma pasta gravável", ex);
        }
    }
}
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/infrastructure/security/SecurityConfiguration.java`

Substitua o conteúdo inteiro deste arquivo (ou crie-o se ainda não existir). Preserve os demais arquivos. O Spring Security usa estas configurações para autenticar a conta e validar o JWT.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.infrastructure.security;

import jakarta.servlet.DispatcherType;
import org.springframework.context.annotation.*;
import org.springframework.http.HttpMethod;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.oauth2.server.resource.authentication.*;
import org.springframework.security.web.SecurityFilterChain;

@org.springframework.boot.autoconfigure.condition.ConditionalOnWebApplication
@Configuration
public class SecurityConfiguration {
    @Bean
    SecurityFilterChain apiSecurity(HttpSecurity http, SecurityErrorWriter errors) throws Exception {
        JwtGrantedAuthoritiesConverter authorities = new JwtGrantedAuthoritiesConverter();
        authorities.setAuthoritiesClaimName("roles");
        authorities.setAuthorityPrefix("ROLE_");
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(authorities);
        var unauthorized = (org.springframework.security.web.AuthenticationEntryPoint) (req, res, cause) -> {
            res.setHeader("WWW-Authenticate", "Bearer");
            errors.write(req, res, 401, "Autenticação necessária ou token inválido");
        };
        var forbidden = (org.springframework.security.web.access.AccessDeniedHandler) (req, res, cause) -> errors
                .write(req, res, 403, "Acesso negado");
        return http
                .csrf(csrf -> csrf.disable())
                .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .requestCache(cache -> cache.disable())
                .authorizeHttpRequests(auth -> auth
                        .dispatcherTypeMatchers(DispatcherType.ERROR).permitAll()
                        .requestMatchers(HttpMethod.PUT, "/api/suggestions/*").hasRole("ADMIN")
                        .requestMatchers(HttpMethod.DELETE, "/api/suggestions/*").hasRole("ADMIN")
                        .anyRequest().permitAll())
                .oauth2ResourceServer(resource -> resource
                        .jwt(jwt -> jwt.jwtAuthenticationConverter(converter))
                        .authenticationEntryPoint(unauthorized).accessDeniedHandler(forbidden))
                .exceptionHandling(ex -> ex.authenticationEntryPoint(unauthorized).accessDeniedHandler(forbidden))
                .build();
    }
}
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/infrastructure/security/SecurityErrorWriter.java`

Substitua o conteúdo inteiro deste arquivo (ou crie-o se ainda não existir). Preserve os demais arquivos. O Spring Security usa estas configurações para autenticar a conta e validar o JWT.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.infrastructure.security;

import java.io.IOException;
import java.time.Instant;
import java.util.Map;
import jakarta.servlet.http.*;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Component;
import tools.jackson.databind.ObjectMapper;
import br.com.etecalbertoferes.CaixaDeSugestoes.interfaces.rest.ApiError;

@Component
public class SecurityErrorWriter {
    private final ObjectMapper mapper;

    public SecurityErrorWriter(ObjectMapper mapper) {
        this.mapper = mapper;
    }

    public void write(HttpServletRequest request, HttpServletResponse response, int status, String message)
            throws IOException {
        response.setStatus(status);
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setCharacterEncoding("UTF-8");
        mapper.writeValue(response.getOutputStream(),
                new ApiError(Instant.now(), status, message, request.getRequestURI(), Map.of()));
    }
}
```

<a id="etapa-7"></a>

## Etapa 7 — Contrato HTTP e documentação

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/interfaces/rest/AdminAuthController.java`

Substitua o conteúdo inteiro deste arquivo (ou crie-o se ainda não existir). Preserve os demais arquivos.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.interfaces.rest;

import jakarta.validation.Valid;
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;
import io.swagger.v3.oas.annotations.*;
import io.swagger.v3.oas.annotations.tags.Tag;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.media.*;
import br.com.etecalbertoferes.CaixaDeSugestoes.application.AdminAuthenticationService;

@RestController
@RequestMapping("/api/auth")
@Tag(name = "Administração", description = "Login exclusivo do administrador; sem cadastro público")
public class AdminAuthController {
    private final AdminAuthenticationService service;

    public AdminAuthController(AdminAuthenticationService service) {
        this.service = service;
    }

    @PostMapping(value = "/login", consumes = MediaType.APPLICATION_JSON_VALUE)
    @Operation(summary = "Autenticar administrador e emitir JWT", description = "Envie username e password. accessToken deve ser apresentado como Authorization: Bearer <token>. Expiração em segundos; não há refresh nem revogação individual nesta etapa.")
    @ApiResponse(responseCode = "200", description = "Token emitido")
    @ApiResponse(responseCode = "400", description = "Corpo inválido", content = @Content(schema = @Schema(implementation = ApiError.class)))
    @ApiResponse(responseCode = "401", description = "Credenciais inválidas", content = @Content(schema = @Schema(implementation = ApiError.class)))
    public ResponseEntity<AdminAuthenticationService.AccessToken> login(@Valid @RequestBody AdminLoginRequest request) {
        return ResponseEntity.ok().cacheControl(CacheControl.noStore()).header(HttpHeaders.PRAGMA, "no-cache")
                .body(service.login(request.username(), request.password()));
    }
}
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/interfaces/rest/AdminLoginRequest.java`

Substitua o conteúdo inteiro deste arquivo (ou crie-o se ainda não existir). Preserve os demais arquivos.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.interfaces.rest;

import jakarta.validation.constraints.*;
import io.swagger.v3.oas.annotations.media.Schema;
public record AdminLoginRequest(
    @NotBlank(message = "Usuário é obrigatório") @Size(max = 100, message = "Usuário deve ter no máximo 100 caracteres")
    @Schema(example = "administrador") String username,
    @NotBlank(message = "Senha é obrigatória") @Size(max = 72, message = "Senha deve ter no máximo 72 caracteres")
    @Schema(format = "password", accessMode = Schema.AccessMode.WRITE_ONLY) String password) {
    @Override public String toString() { return "AdminLoginRequest[credenciais protegidas]"; }
}
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/interfaces/rest/ApiExceptionHandler.java`

Substitua o conteúdo inteiro deste arquivo (ou crie-o se ainda não existir). Preserve os demais arquivos.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.interfaces.rest;

import java.time.Instant;
import java.util.LinkedHashMap;
import java.util.Map;
import jakarta.servlet.http.HttpServletRequest;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.*;
import org.springframework.web.ErrorResponse;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.context.request.*;
import org.springframework.web.method.annotation.HandlerMethodValidationException;
import org.springframework.web.servlet.mvc.method.annotation.ResponseEntityExceptionHandler;
import br.com.etecalbertoferes.CaixaDeSugestoes.domain.ResourceNotFoundException;

@RestControllerAdvice
public class ApiExceptionHandler extends ResponseEntityExceptionHandler {
    private static final Logger log = LoggerFactory.getLogger(ApiExceptionHandler.class);

    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(MethodArgumentNotValidException ex,
            HttpHeaders headers, HttpStatusCode status, WebRequest request) {
        Map<String, String> fields = new LinkedHashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(e -> fields.putIfAbsent(e.getField(), e.getDefaultMessage()));
        return respond(status, "Dados inválidos", fields, headers, request);
    }

    @Override
    protected ResponseEntity<Object> handleHandlerMethodValidationException(HandlerMethodValidationException ex,
            HttpHeaders headers, HttpStatusCode status, WebRequest request) {
        Map<String, String> fields = new LinkedHashMap<>();
        ex.getParameterValidationResults().forEach(result -> {
            String name = result.getMethodParameter().getParameterName();
            result.getResolvableErrors()
                    .forEach(error -> fields.putIfAbsent(name == null ? "id" : name, error.getDefaultMessage()));
        });
        return respond(status, "Dados inválidos", fields, headers, request);
    }

    @Override
    protected ResponseEntity<Object> handleExceptionInternal(Exception ex, Object body, HttpHeaders headers,
            HttpStatusCode status, WebRequest request) {
        String message = switch (status.value()) {
            case 400 -> "Requisição inválida";
            case 404 -> "Recurso não encontrado";
            case 405 -> "Método HTTP não permitido";
            case 406 -> "Formato de resposta não suportado";
            case 415 -> "Tipo de conteúdo não suportado";
            default -> status.is5xxServerError() ? "Erro interno do servidor" : "Requisição não permitida";
        };
        return respond(status, message, Map.of(), headers, request);
    }

    @ExceptionHandler(org.springframework.security.core.AuthenticationException.class)
    public ResponseEntity<Object> invalidCredentials(WebRequest request) {
        HttpHeaders headers = new HttpHeaders();
        headers.set(HttpHeaders.WWW_AUTHENTICATE, "Bearer");
        headers.setCacheControl("no-store");
        return respond(HttpStatus.UNAUTHORIZED, "Credenciais inválidas", Map.of(), headers, request);
    }

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<Object> notFound(ResourceNotFoundException ex, WebRequest request) {
        return respond(HttpStatus.NOT_FOUND, ex.getMessage(), Map.of(), new HttpHeaders(), request);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<Object> unexpected(Exception ex, WebRequest request) {
        if (ex instanceof ErrorResponse error) {
            return respond(error.getStatusCode(), "Requisição inválida", Map.of(), error.getHeaders(), request);
        }
        log.error("Falha inesperada ao processar requisição", ex);
        return respond(HttpStatus.INTERNAL_SERVER_ERROR, "Erro interno do servidor", Map.of(), new HttpHeaders(),
                request);
    }

    private ResponseEntity<Object> respond(HttpStatusCode status, String message, Map<String, String> fields,
            HttpHeaders headers, WebRequest request) {
        HttpServletRequest servlet = ((ServletWebRequest) request).getRequest();
        return new ResponseEntity<>(
                new ApiError(Instant.now(), status.value(), message, servlet.getRequestURI(), fields), headers, status);
    }
}
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/interfaces/rest/OpenApiConfiguration.java`

Substitua o conteúdo inteiro deste arquivo (ou crie-o se ainda não existir). Preserve os demais arquivos.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.interfaces.rest;

import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Info;
import org.springframework.context.annotation.*;

@Configuration
public class OpenApiConfiguration {
    @Bean
    OpenAPI api() {
        return new OpenAPI().components(new io.swagger.v3.oas.models.Components().addSecuritySchemes("bearerAuth",
                new io.swagger.v3.oas.models.security.SecurityScheme()
                        .type(io.swagger.v3.oas.models.security.SecurityScheme.Type.HTTP).scheme(
                                "bearer")
                        .bearerFormat("JWT")))
                .info(new Info().title("Caixa de Sugestões — Etec Prefeito Alberto Feres")
                        .version("2.0.0").description(
                                "API de sugestões para consumo pelo projeto web com Thymeleaf. Catálogos didáticos somente para consulta. Datas em UTC."));
    }
}
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/interfaces/rest/SuggestionController.java`

Substitua o conteúdo inteiro deste arquivo (ou crie-o se ainda não existir). Preserve os demais arquivos.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.interfaces.rest;

import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.tags.Tag;
import io.swagger.v3.oas.annotations.responses.*;
import io.swagger.v3.oas.annotations.media.*;
import java.net.URI;
import jakarta.validation.Valid;
import jakarta.validation.constraints.Positive;
import java.util.List;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import br.com.etecalbertoferes.CaixaDeSugestoes.application.SuggestionService;

@Tag(name = "Sugestões", description = "Cadastro, consulta, substituição e exclusão de sugestões")
@RestController
@RequestMapping("/api/suggestions")
@ApiResponses({
        @ApiResponse(responseCode = "400", description = "Dados, ID ou filtro inválido", content = @Content(schema = @Schema(implementation = ApiError.class))),
        @ApiResponse(responseCode = "404", description = "Sugestão, curso ou categoria não encontrado", content = @Content(schema = @Schema(implementation = ApiError.class))),
        @ApiResponse(responseCode = "500", description = "Erro interno", content = @Content(schema = @Schema(implementation = ApiError.class)))
})
public class SuggestionController {
    private final SuggestionService service;

    public SuggestionController(SuggestionService service) {
        this.service = service;
    }

    @Operation(summary = "Listar sugestões", description = "Array sem paginação. Filtros combinados por AND; q busca título OU conteúdo sem distinguir maiúsculas. Espaços externos são removidos; vazio é ignorado. %, _ e ! são literais. Ordem: createdAt e id decrescentes. IDs positivos inexistentes retornam [].")
    @ApiResponse(responseCode = "200", description = "Lista de sugestões, inclusive vazia")
    @GetMapping
    public List<SuggestionResponse> list(
            @Parameter(name = "courseId", description = "ID positivo do curso; consulte GET /api/courses", example = "1") @RequestParam(name = "courseId", required = false) @Positive(message = "ID deve ser positivo") Long courseId,
            @Parameter(name = "categoryId", description = "ID positivo da categoria; consulte GET /api/categories", example = "1") @RequestParam(name = "categoryId", required = false) @Positive(message = "ID deve ser positivo") Long categoryId,
            @Parameter(name = "q", description = "Trecho literal do título ou conteúdo, sem distinguir maiúsculas; pode ser combinado com curso e categoria", example = "biblioteca") @RequestParam(name = "q", required = false) String q) {
        return service.list(courseId, categoryId, q).stream().map(SuggestionResponse::from).toList();
    }

    @Operation(summary = "Consultar sugestão por ID")
    @ApiResponse(responseCode = "200", description = "Sugestão encontrada")
    @GetMapping("/{id}")
    public SuggestionResponse get(@Parameter(name = "id", description = "ID positivo da sugestão", example = "1") @PathVariable("id") @Positive(message = "ID deve ser positivo") Long id) {
        return SuggestionResponse.from(service.get(id));
    }

    @Operation(summary = "Criar sugestão", description = "O servidor gera id e createdAt. Autor ausente, nulo ou em branco é armazenado como null e devolvido como Anônimo. Consulte os IDs dos catálogos antes de enviar.")
    @ApiResponse(responseCode = "201", description = "Sugestão criada; Location contém /api/suggestions/{id}", content = @Content(schema = @Schema(implementation = SuggestionResponse.class)), headers = @io.swagger.v3.oas.annotations.headers.Header(name = "Location", schema = @Schema(type = "string")))
    @PostMapping
    public ResponseEntity<SuggestionResponse> create(@Valid @RequestBody SuggestionRequest request) {
        SuggestionResponse response = SuggestionResponse.from(service.create(request.command()));
        return ResponseEntity.created(URI.create("/api/suggestions/" + response.id())).body(response);
    }

    @Operation(summary = "Substituir campos editáveis", description = "Preserva id e createdAt. Omitir author remove o nome anterior. Curso e categoria são obrigatórios. Falhas não alteram o registro.")
    @ApiResponse(responseCode = "200", description = "Sugestão atualizada")
    @io.swagger.v3.oas.annotations.security.SecurityRequirement(name = "bearerAuth")
    @ApiResponse(responseCode = "401", description = "Autenticação necessária", content = @Content(schema = @Schema(implementation = ApiError.class)))
    @ApiResponse(responseCode = "403", description = "Acesso negado", content = @Content(schema = @Schema(implementation = ApiError.class)))
    @PutMapping("/{id}")
    public SuggestionResponse update(@Parameter(name = "id", description = "ID positivo da sugestão", example = "1") @PathVariable("id") @Positive(message = "ID deve ser positivo") Long id,
            @Valid @RequestBody SuggestionRequest request) {
        return SuggestionResponse.from(service.update(id, request.command()));
    }

    @Operation(summary = "Excluir sugestão", description = "Exclusão física, preservando curso e categoria. Repetir a exclusão retorna 404.")
    @ApiResponse(responseCode = "204", description = "Excluída, sem corpo", content = @Content)
    @io.swagger.v3.oas.annotations.security.SecurityRequirement(name = "bearerAuth")
    @ApiResponse(responseCode = "401", description = "Autenticação necessária", content = @Content(schema = @Schema(implementation = ApiError.class)))
    @ApiResponse(responseCode = "403", description = "Acesso negado", content = @Content(schema = @Schema(implementation = ApiError.class)))
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@Parameter(name = "id", description = "ID positivo da sugestão", example = "1") @PathVariable("id") @Positive(message = "ID deve ser positivo") Long id) {
        service.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

<a id="etapa-8"></a>

## Etapa 8 — Testes automatizados


### `src/test/java/br/com/etecalbertoferes/CaixaDeSugestoes/ApiIntegrationTest.java`

Substitua o conteúdo inteiro deste arquivo (ou crie-o se ainda não existir). Preserve os demais arquivos. Os testes de integração utilizam somente suggestions_test.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes;

import java.time.Instant;
import br.com.etecalbertoferes.CaixaDeSugestoes.application.*;
import br.com.etecalbertoferes.CaixaDeSugestoes.domain.*;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.webmvc.test.autoconfigure.AutoConfigureMockMvc;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.transaction.annotation.Transactional;
import static org.assertj.core.api.Assertions.*;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.user;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@AutoConfigureMockMvc
@Transactional
class ApiIntegrationTest extends ContextTest {
    @Autowired MockMvc mvc;
    @Autowired jakarta.persistence.EntityManager em;
    @Autowired SuggestionService service;
    @Autowired SuggestionRepository repository;
    @Autowired CatalogService catalogs;
    Long course() { return catalogs.courses().get(0).id(); }
    Long category() { return catalogs.categories().get(0).id(); }
    Suggestion create(String title, String content, String author, Long courseId, Long categoryId) {
        return service.create(new SuggestionCommand(title, content, author, courseId, categoryId));
    }
    @Test void catalogAndHttpLifecycle() throws Exception {
        mvc.perform(get("/api/courses")).andExpect(status().isOk()).andExpect(jsonPath("$.length()").value(4));
        mvc.perform(get("/api/categories")).andExpect(status().isOk()).andExpect(jsonPath("$.length()").value(3));
        String body = "{\"title\":\"Tomadas\",\"content\":\"Biblioteca\",\"author\":\"  Pessoa  \",\"courseId\":"+course()+",\"categoryId\":"+category()+"}";
        String location = mvc.perform(post("/api/suggestions").contentType(MediaType.APPLICATION_JSON).content(body))
            .andExpect(status().isCreated()).andExpect(jsonPath("$.author").value("Pessoa"))
            .andReturn().getResponse().getHeader("Location");
        Long id = Long.valueOf(location.substring(location.lastIndexOf('/')+1));
        Instant time = service.get(id).createdAt();
        mvc.perform(get(location)).andExpect(status().isOk()).andExpect(jsonPath("$.title").value("Tomadas"));
        mvc.perform(put(location).with(user("test-admin").roles("ADMIN")).contentType(MediaType.APPLICATION_JSON).content(body.replace("  Pessoa  ", "  ")))
            .andExpect(status().isOk()).andExpect(jsonPath("$.author").value("Anônimo"));
        assertThat(service.get(id).createdAt()).isEqualTo(time);
        em.flush();
        assertThat(jdbc.queryForObject("select author from suggestions where id=?", String.class, id)).isNull();
        mvc.perform(delete(location).with(user("test-admin").roles("ADMIN"))).andExpect(status().isNoContent()).andExpect(content().string(""));
        mvc.perform(get(location)).andExpect(status().isNotFound());
        mvc.perform(delete(location).with(user("test-admin").roles("ADMIN"))).andExpect(status().isNotFound());
        mvc.perform(put(location).with(user("test-admin").roles("ADMIN")).contentType(MediaType.APPLICATION_JSON).content(body)).andExpect(status().isNotFound());
        assertThat(catalogs.courses()).hasSize(4);
        assertThat(catalogs.categories()).hasSize(3);
    }
    @Test void filtersLiteralCharactersAndOrdering() {
        Long c1 = course(), c2 = catalogs.courses().get(1).id();
        Long t1 = category(), t2 = catalogs.categories().get(1).id();
        Suggestion a = create("Tomadas 100%_!", "biblioteca", null, c1, t1);
        Suggestion b = create("outro", "TOMADAS 100xyz", "", c2, t1);
        Suggestion c = create("Tomadas", "terceiro", "Pessoa", c1, t2);
        assertThat(service.list(null,null,"tomadas")).extracting(Suggestion::id).containsExactly(c.id(), b.id(), a.id());
        assertThat(service.list(c1,null,null)).extracting(Suggestion::id).containsExactly(c.id(), a.id());
        assertThat(service.list(null,t1,null)).extracting(Suggestion::id).containsExactly(b.id(), a.id());
        assertThat(service.list(c1,t1," TOMADAS ")).extracting(Suggestion::id).containsExactly(a.id());
        for (String literal : new String[]{"%", "_", "!", "100%_!"})
            assertThat(service.list(null,null,literal)).extracting(Suggestion::id).containsExactly(a.id());
        assertThat(service.list(999999L,null,null)).isEmpty();
        assertThat(service.list(null,999999L,null)).isEmpty();
        assertThat(service.list(null,null,"inexistente")).isEmpty();
        assertThat(service.list(null,null,"  ")).hasSize(3);
        Instant same = Instant.parse("2026-01-01T00:00:00Z");
        jdbc.update("update suggestions set created_at=?", java.sql.Timestamp.from(same));
        assertThat(service.list(null,null,null)).extracting(Suggestion::id).containsExactly(c.id(), b.id(), a.id());
    }
    @Test void rejectedUpdatesPreserveStoredDataAndTextHasNo255Limit() throws Exception {
        Suggestion item = create("x".repeat(500), "y".repeat(1000), null, course(), category());
        String invalid = "{\"title\":\"change\",\"content\":\"change\",\"courseId\":"+course()+",\"categoryId\":999999}";
        mvc.perform(put("/api/suggestions/"+item.id()).with(user("test-admin").roles("ADMIN")).contentType(MediaType.APPLICATION_JSON).content(invalid)).andExpect(status().isNotFound());
        assertThat(service.get(item.id()).title()).hasSize(500);
        assertThat(service.get(item.id()).content()).hasSize(1000);
        mvc.perform(post("/api/suggestions").contentType(MediaType.APPLICATION_JSON).content(invalid)).andExpect(status().isNotFound());
        assertThat(service.list(null,null,null)).hasSize(1);
    }
    @Test void openApiDocumentsAllOperations() throws Exception {
        mvc.perform(get("/v3/api-docs")).andExpect(status().isOk())
            .andExpect(jsonPath("$.paths['/api/suggestions'].get").exists())
            .andExpect(jsonPath("$.paths['/api/suggestions'].post.responses['201']").exists())
            .andExpect(jsonPath("$.paths['/api/suggestions/{id}'].get").exists())
            .andExpect(jsonPath("$.paths['/api/suggestions/{id}'].put").exists())
            .andExpect(jsonPath("$.paths['/api/suggestions/{id}'].delete.responses['204']").exists())
            .andExpect(jsonPath("$.paths['/api/courses'].get").exists())
            .andExpect(jsonPath("$.paths['/api/categories'].get").exists());
        mvc.perform(get("/swagger-ui.html")).andExpect(status().is3xxRedirection());
    }
}
```

### `src/test/java/br/com/etecalbertoferes/CaixaDeSugestoes/ContextTest.java`

Substitua o conteúdo inteiro deste arquivo (ou crie-o se ainda não existir). Preserve os demais arquivos. Os testes de integração utilizam somente suggestions_test.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.jdbc.core.JdbcTemplate;
import static org.assertj.core.api.Assertions.assertThat;

@org.springframework.test.context.ActiveProfiles("test")
@org.springframework.test.annotation.DirtiesContext(classMode = org.springframework.test.annotation.DirtiesContext.ClassMode.AFTER_CLASS)
@SpringBootTest
class ContextTest {
    static final String ADMIN_PASSWORD = "Test-Admin-2026!";
    @Autowired JdbcTemplate jdbc;
    @Autowired br.com.etecalbertoferes.CaixaDeSugestoes.domain.AdminAccountRepository accounts;
    @Autowired org.springframework.security.crypto.password.PasswordEncoder passwordEncoder;
    @org.junit.jupiter.api.BeforeEach void provisionTestAdmin() {
        if (accounts.findByUsername("test-admin").isEmpty())
            jdbc.update("insert into administrators (username, password_hash) values (?, ?)",
                "test-admin", passwordEncoder.encode(ADMIN_PASSWORD));
    }
    @Test void migrationsAndSeeds() {
        assertThat(jdbc.queryForObject("select count(*) from courses", Long.class)).isEqualTo(4L);
        assertThat(jdbc.queryForObject("select count(*) from categories", Long.class)).isEqualTo(3L);
        assertThat(jdbc.queryForObject("select count(*) from suggestions", Long.class)).isZero();
    }
}
```

### `src/test/java/br/com/etecalbertoferes/CaixaDeSugestoes/JwtSecurityIntegrationTest.java`

Substitua o conteúdo inteiro deste arquivo (ou crie-o se ainda não existir). Preserve os demais arquivos. Os testes de integração utilizam somente suggestions_test.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes;

import java.time.Instant;
import java.util.List;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.webmvc.test.autoconfigure.AutoConfigureMockMvc;
import org.springframework.http.MediaType;
import org.springframework.security.oauth2.jwt.*;
import org.springframework.security.oauth2.jose.jws.MacAlgorithm;
import org.springframework.test.web.servlet.MockMvc;
import tools.jackson.databind.ObjectMapper;
import static org.assertj.core.api.Assertions.*;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.httpBasic;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;
import br.com.etecalbertoferes.CaixaDeSugestoes.application.*;

@AutoConfigureMockMvc
class JwtSecurityIntegrationTest extends ContextTest {
    @Autowired MockMvc mvc;
    @Autowired ObjectMapper mapper;
    @Autowired JwtEncoder encoder;
    @Autowired JwtDecoder decoder;
    @Autowired CatalogService catalogs;
    @Autowired SuggestionService suggestions;
    String login(String username, String password) throws Exception {
        return mvc.perform(post("/api/auth/login").contentType(MediaType.APPLICATION_JSON)
            .content(mapper.writeValueAsString(java.util.Map.of("username", username, "password", password))))
            .andReturn().getResponse().getContentAsString();
    }
    String token() throws Exception { return mapper.readTree(login("test-admin", ADMIN_PASSWORD)).get("accessToken").asText(); }
    String signed(String issuer, String audience, String subject, Instant expiry, List<String> roles) {
        var claims = JwtClaimsSet.builder().issuer(issuer).audience(List.of(audience)).subject(subject)
            .issuedAt(Instant.now().minusSeconds(120)).claim("roles", roles);
        if (expiry != null) claims.expiresAt(expiry);
        return encoder.encode(JwtEncoderParameters.from(JwsHeader.with(MacAlgorithm.HS256).build(), claims.build())).getTokenValue();
    }
    @Test void loginIssuesAdminTokenAndDoesNotCreateSession() throws Exception {
        var result = mvc.perform(post("/api/auth/login").contentType(MediaType.APPLICATION_JSON)
            .content(mapper.writeValueAsString(java.util.Map.of("username", "test-admin", "password", ADMIN_PASSWORD))))
            .andExpect(status().isOk()).andExpect(header().string("Cache-Control", "no-store"))
            .andExpect(header().doesNotExist("Set-Cookie")).andExpect(jsonPath("$.tokenType").value("Bearer"))
            .andExpect(jsonPath("$.expiresIn").value(900)).andReturn();
        String value = mapper.readTree(result.getResponse().getContentAsString()).get("accessToken").asText();
        Jwt jwt = decoder.decode(value);
        assertThat(jwt.getSubject()).isEqualTo("test-admin");
        assertThat(jwt.getClaimAsStringList("roles")).containsExactly("ADMIN");
        assertThat(jwt.getAudience()).containsExactly(AdminAuthenticationService.AUDIENCE);
        assertThat(jwt.getClaims()).doesNotContainKeys("password", "passwordHash");
        assertThat(result.getRequest().getSession(false)).isNull();
    }
    @Test void wrongCredentialsAndUnknownAdminHaveSameFailure() throws Exception {
        for (String username : List.of("test-admin", "unknown")) {
            mvc.perform(post("/api/auth/login").contentType(MediaType.APPLICATION_JSON)
                .content(mapper.writeValueAsString(java.util.Map.of("username", username, "password", "wrong"))))
                .andExpect(status().isUnauthorized()).andExpect(jsonPath("$.message").value("Credenciais inválidas"))
                .andExpect(jsonPath("$.status").value(401)).andExpect(header().string("WWW-Authenticate", "Bearer"));
        }
    }
    @Test void invalidLoginBodyAndUtf8PasswordAreHandled() throws Exception {
        for (String body : List.of("{}", "null", "{", "{\"username\":\" \",\"password\":\"\"}"))
            mvc.perform(post("/api/auth/login").contentType(MediaType.APPLICATION_JSON).content(body))
                .andExpect(status().isBadRequest());
        mvc.perform(post("/api/auth/login").contentType(MediaType.APPLICATION_JSON)
            .content(mapper.writeValueAsString(java.util.Map.of("username", "test-admin", "password", "ç".repeat(50)))))
            .andExpect(status().isUnauthorized());
        mvc.perform(post("/api/auth/register").contentType(MediaType.APPLICATION_JSON).content("{}"))
            .andExpect(status().isNotFound());
    }
    @Test void publicReadAndAnonymousCreateNeedNoToken() throws Exception {
        for (String path : List.of("/api/suggestions", "/api/courses", "/api/categories", "/v3/api-docs"))
            mvc.perform(get(path)).andExpect(status().isOk());
        Long course = catalogs.courses().get(0).id(), category = catalogs.categories().get(0).id();
        var result = mvc.perform(post("/api/suggestions").contentType(MediaType.APPLICATION_JSON)
            .content(mapper.writeValueAsString(java.util.Map.of("title", "Público", "content", "Sem login", "courseId", course, "categoryId", category))))
            .andExpect(status().isCreated()).andExpect(jsonPath("$.author").value("Anônimo")).andReturn();
        Long id = mapper.readTree(result.getResponse().getContentAsString()).get("id").asLong();
        suggestions.delete(id);
    }
    @Test void missingMalformedAndBasicCredentialsCannotMutate() throws Exception {
        mvc.perform(delete("/api/suggestions/999999")).andExpect(status().isUnauthorized())
            .andExpect(header().string("WWW-Authenticate", "Bearer")).andExpect(jsonPath("$.fieldErrors").isEmpty());
        mvc.perform(put("/api/suggestions/999999")).andExpect(status().isUnauthorized());
        mvc.perform(delete("/api/suggestions/999999").header("Authorization", "Bearer invalid"))
            .andExpect(status().isUnauthorized());
        mvc.perform(delete("/api/suggestions/999999").with(httpBasic("test-admin", ADMIN_PASSWORD)))
            .andExpect(status().isUnauthorized());
    }
    @Test void genuineAdminJwtCanMutateAndNoSessionCarriesAuthentication() throws Exception {
        String jwt = token();
        Long course = catalogs.courses().get(0).id(), category = catalogs.categories().get(0).id();
        var item = suggestions.create(new SuggestionCommand("Original", "Conteúdo", null, course, category));
        String body = mapper.writeValueAsString(java.util.Map.of("title", "Alterado", "content", "Atualizado", "courseId", course, "categoryId", category));
        mvc.perform(put("/api/suggestions/"+item.id()).header("Authorization", "Bearer "+jwt)
            .contentType(MediaType.APPLICATION_JSON).content(body)).andExpect(status().isOk())
            .andExpect(jsonPath("$.title").value("Alterado"));
        mvc.perform(delete("/api/suggestions/"+item.id())).andExpect(status().isUnauthorized());
        assertThat(suggestions.get(item.id()).title()).isEqualTo("Alterado");
        mvc.perform(delete("/api/suggestions/"+item.id()).header("Authorization", "Bearer "+jwt))
            .andExpect(status().isNoContent());
        mvc.perform(get("/api/suggestions/"+item.id())).andExpect(status().isNotFound());
    }
    @Test void expiredTamperedWrongIssuerAudienceSubjectAndMissingExpiryAreRejected() throws Exception {
        String valid = token();
        int dot = valid.lastIndexOf('.')+1;
        String tampered = valid.substring(0,dot)+(valid.charAt(dot)=='A'?'B':'A')+valid.substring(dot+1);
        String issuer=AdminAuthenticationService.ISSUER, aud=AdminAuthenticationService.AUDIENCE;
        for (String invalid : List.of(tampered,
            signed(issuer,aud,"test-admin",Instant.now().minusSeconds(1),List.of("ADMIN")),
            signed("wrong",aud,"test-admin",Instant.now().plusSeconds(60),List.of("ADMIN")),
            signed(issuer,"wrong","test-admin",Instant.now().plusSeconds(60),List.of("ADMIN")),
            signed(issuer,aud,"someone-else",Instant.now().plusSeconds(60),List.of("ADMIN")),
            signed(issuer,aud,"test-admin",null,List.of("ADMIN")))) {
            mvc.perform(delete("/api/suggestions/999999").header("Authorization", "Bearer "+invalid))
                .andExpect(status().isUnauthorized()).andExpect(jsonPath("$.status").value(401));
        }
    }
    @Test void signedTokenWithoutAdminRoleIsForbidden() throws Exception {
        String jwt = signed(AdminAuthenticationService.ISSUER,AdminAuthenticationService.AUDIENCE,"test-admin",Instant.now().plusSeconds(60),List.of("READER"));
        mvc.perform(delete("/api/suggestions/999999").header("Authorization", "Bearer "+jwt))
            .andExpect(status().isForbidden()).andExpect(jsonPath("$.status").value(403));
    }
    @Test void disablingAdminRejectsBothLoginAndPreviouslyIssuedJwt() throws Exception {
        String jwt = token();
        jdbc.update("update administrators set active=false where username=?", "test-admin");
        try {
            mvc.perform(post("/api/auth/login").contentType(MediaType.APPLICATION_JSON)
                .content(mapper.writeValueAsString(java.util.Map.of("username", "test-admin", "password", ADMIN_PASSWORD))))
                .andExpect(status().isUnauthorized()).andExpect(jsonPath("$.message").value("Credenciais inválidas"));
            mvc.perform(delete("/api/suggestions/999999").header("Authorization", "Bearer " + jwt))
                .andExpect(status().isUnauthorized());
        } finally { jdbc.update("update administrators set active=true where username=?", "test-admin"); }
    }
    @Test void swaggerExposesAllQueryAndPathInputs() throws Exception {
        var response = mvc.perform(get("/v3/api-docs")).andExpect(status().isOk())
            .andReturn().getResponse().getContentAsString();
        var paths = mapper.readTree(response).get("paths");
        var parameters = paths.get("/api/suggestions").get("get").get("parameters");
        assertThat(parameters.size()).isEqualTo(3);
        var byName = new java.util.HashMap<String, tools.jackson.databind.JsonNode>();
        parameters.forEach(parameter -> byName.put(parameter.get("name").asText(), parameter));
        assertThat(byName.keySet()).containsExactlyInAnyOrder("courseId", "categoryId", "q");
        for (String name : List.of("courseId", "categoryId", "q")) {
            var parameter = byName.get(name);
            assertThat(parameter.get("in").asText()).isEqualTo("query");
            assertThat(parameter.get("required").asBoolean()).isFalse();
            assertThat(parameter.get("example").asText()).isNotBlank();
            assertThat(parameter.get("schema").get("type").asText())
                .isEqualTo(name.equals("q") ? "string" : "integer");
        }
        for (String method : List.of("get", "put", "delete")) {
            var inputs = paths.get("/api/suggestions/{id}").get(method).get("parameters");
            assertThat(inputs.size()).isEqualTo(1);
            assertThat(inputs.get(0).get("name").asText()).isEqualTo("id");
            assertThat(inputs.get(0).get("in").asText()).isEqualTo("path");
            assertThat(inputs.get(0).get("required").asBoolean()).isTrue();
        }
        for (String path : List.of("/api/suggestions", "/api/auth/login"))
            assertThat(paths.get(path).get("post").get("requestBody").get("required").asBoolean()).isTrue();
        assertThat(paths.get("/api/suggestions/{id}").get("put").get("requestBody").get("required").asBoolean()).isTrue();
    }
    @Test void swaggerMarksOnlyMutationsWithBearerSecurity() throws Exception {
        mvc.perform(get("/v3/api-docs")).andExpect(status().isOk())
            .andExpect(jsonPath("$.components.securitySchemes.bearerAuth.scheme").value("bearer"))
            .andExpect(jsonPath("$.paths['/api/auth/login'].post.security").doesNotExist())
            .andExpect(jsonPath("$.paths['/api/suggestions'].post.security").doesNotExist())
            .andExpect(jsonPath("$.paths['/api/suggestions/{id}'].put.security[0].bearerAuth").isArray())
            .andExpect(jsonPath("$.paths['/api/suggestions/{id}'].delete.security[0].bearerAuth").isArray());
    }
}
```

### `src/test/java/br/com/etecalbertoferes/CaixaDeSugestoes/infrastructure/security/SecurityConfigurationTest.java`

Substitua o conteúdo inteiro deste arquivo (ou crie-o se ainda não existir). Preserve os demais arquivos. O Spring Security usa estas configurações para autenticar a conta e validar o JWT. Os testes de integração utilizam somente suggestions_test.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.infrastructure.security;

import java.nio.file.*;
import java.util.Base64;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;
import org.springframework.mock.env.MockEnvironment;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import br.com.etecalbertoferes.CaixaDeSugestoes.domain.*;
import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

class SecurityConfigurationTest {
    @TempDir Path temp;
    @Test void missingSecretCreatesAndReusesPrivateKeyFile() throws Exception {
        Path file = temp.resolve("private/jwt.key");
        var env = new MockEnvironment().withProperty("app.jwt.key-file", file.toString());
        var config = new JwtConfiguration();
        byte[] first = config.jwtSigningKey(env).getEncoded();
        assertThat(first).hasSize(32);
        assertThat(config.jwtSigningKey(env).getEncoded()).isEqualTo(first);
        assertThat(Base64.getDecoder().decode(Files.readString(file))).isEqualTo(first);
        if (file.getFileSystem().supportedFileAttributeViews().contains("posix"))
            assertThat(Files.getPosixFilePermissions(file)).isEqualTo(java.nio.file.attribute.PosixFilePermissions.fromString("rw-------"));
    }
    @Test void invalidOrWeakExplicitKeysFailClosed() {
        var config = new JwtConfiguration();
        for (String secret : new String[]{"not base64!", "c2hvcnQ="})
            assertThatThrownBy(() -> config.jwtSigningKey(new MockEnvironment().withProperty("app.jwt.secret-base64", secret)))
                .isInstanceOf(IllegalStateException.class);
    }
    @Test void invalidExistingKeyIsNotOverwritten() throws Exception {
        Path file = temp.resolve("jwt.key"); Files.writeString(file, "broken key");
        assertThatThrownBy(() -> new JwtConfiguration().jwtSigningKey(new MockEnvironment().withProperty("app.jwt.key-file", file.toString())))
            .isInstanceOf(IllegalStateException.class);
        assertThat(Files.readString(file)).isEqualTo("broken key");
    }
    @Test void explicitKeyDoesNotCreateLocalFile() {
        Path file = temp.resolve("jwt.key");
        var env = new MockEnvironment().withProperty("app.jwt.key-file", file.toString())
            .withProperty("app.jwt.secret-base64", Base64.getEncoder().encodeToString(new byte[32]));
        assertThat(new JwtConfiguration().jwtSigningKey(env).getEncoded()).hasSize(32);
        assertThat(file).doesNotExist();
    }
    @Test void userDetailsComeFromDatabaseAndHonorActiveFlag() {
        AdminAccountRepository accounts = mock(AdminAccountRepository.class);
        when(accounts.findByUsername("missing")).thenReturn(java.util.Optional.empty());
        when(accounts.findByUsername("disabled")).thenReturn(java.util.Optional.of(new AdminAccount(1L,"disabled","hash",false)));
        var users = new AdminConfiguration().adminUsers(accounts);
        assertThatThrownBy(() -> users.loadUserByUsername("missing")).isInstanceOf(UsernameNotFoundException.class);
        assertThat(users.loadUserByUsername("disabled").isEnabled()).isFalse();
    }
}
```

### `src/test/java/br/com/etecalbertoferes/CaixaDeSugestoes/interfaces/rest/SuggestionControllerTest.java`

Substitua o conteúdo inteiro deste arquivo (ou crie-o se ainda não existir). Preserve os demais arquivos. Os testes de integração utilizam somente suggestions_test.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.interfaces.rest;

import br.com.etecalbertoferes.CaixaDeSugestoes.application.*;
import br.com.etecalbertoferes.CaixaDeSugestoes.domain.*;
import java.time.Instant;
import java.util.List;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.web.servlet.MockMvc;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@org.springframework.boot.webmvc.test.autoconfigure.AutoConfigureMockMvc(addFilters = false)
@WebMvcTest(SuggestionController.class)
class SuggestionControllerTest {
    @Autowired MockMvc mvc;
    @MockitoBean SuggestionService service;
    private static final String VALID = """
        {"title":"Biblioteca","content":"Mais tomadas","courseId":1,"categoryId":2}
        """;
    private Suggestion example() {
        return new Suggestion(7L, "Biblioteca", "Mais tomadas", null, Instant.parse("2026-10-08T12:00:00Z"),
            new Course(1L, "Curso"), new Category(2L, "Sugestão"));
    }
    @Test void createReturnsLocationAndAnonymous() throws Exception {
        when(service.create(any())).thenReturn(example());
        mvc.perform(post("/api/suggestions").contentType(MediaType.APPLICATION_JSON).content(VALID))
            .andExpect(status().isCreated()).andExpect(header().string("Location", "/api/suggestions/7"))
            .andExpect(jsonPath("$.author").value("Anônimo")).andExpect(jsonPath("$.course.id").value(1))
            .andExpect(jsonPath("$.createdAt").value("2026-10-08T12:00:00Z"));
    }
    @Test void emptyListIsArray() throws Exception {
        when(service.list(null, null, null)).thenReturn(List.of());
        mvc.perform(get("/api/suggestions")).andExpect(status().isOk()).andExpect(content().json("[]"));
    }
    @Test void combinedQueryParametersReachService() throws Exception {
        when(service.list(1L, 2L, "biblioteca")).thenReturn(List.of(example()));
        mvc.perform(get("/api/suggestions").param("courseId", "1")
                .param("categoryId", "2").param("q", "biblioteca"))
            .andExpect(status().isOk()).andExpect(jsonPath("$[0].id").value(7));
        verify(service).list(1L, 2L, "biblioteca");
    }
    @Test void getUpdateAndDelete() throws Exception {
        when(service.get(7L)).thenReturn(example());
        when(service.update(eq(7L), any())).thenReturn(example());
        mvc.perform(get("/api/suggestions/7")).andExpect(status().isOk()).andExpect(jsonPath("$.id").value(7));
        mvc.perform(put("/api/suggestions/7").contentType(MediaType.APPLICATION_JSON).content(VALID))
            .andExpect(status().isOk());
        mvc.perform(delete("/api/suggestions/7")).andExpect(status().isNoContent()).andExpect(content().string(""));
    }
    @Test void invalidBodyHasFieldErrors() throws Exception {
        mvc.perform(post("/api/suggestions").contentType(MediaType.APPLICATION_JSON)
                .content("{\"title\":\"  \",\"content\":\"\",\"courseId\":0,\"categoryId\":-1}"))
            .andExpect(status().isBadRequest()).andExpect(jsonPath("$.status").value(400))
            .andExpect(jsonPath("$.fieldErrors.title").value("Título é obrigatório"))
            .andExpect(jsonPath("$.fieldErrors.content").exists()).andExpect(jsonPath("$.fieldErrors.courseId").exists())
            .andExpect(jsonPath("$.fieldErrors.categoryId").exists());
        verifyNoInteractions(service);
    }
    @Test void invalidIdsAndFilters() throws Exception {
        for (String path : List.of("/api/suggestions/0", "/api/suggestions/-1", "/api/suggestions/abc",
                "/api/suggestions?courseId=-1", "/api/suggestions?categoryId=0", "/api/suggestions?courseId=abc")) {
            mvc.perform(get(path)).andExpect(status().isBadRequest()).andExpect(jsonPath("$.status").value(400));
        }
        verifyNoInteractions(service);
    }
    @Test void malformedMissingAndNullBodies() throws Exception {
        for (String body : List.of("", "{", "null", "{}")) {
            mvc.perform(post("/api/suggestions").contentType(MediaType.APPLICATION_JSON).content(body))
                .andExpect(status().isBadRequest()).andExpect(jsonPath("$.path").value("/api/suggestions"));
        }
        verifyNoInteractions(service);
    }
    @Test void notFoundAndCatalogNotFound() throws Exception {
        when(service.get(7L)).thenThrow(new ResourceNotFoundException("Sugestão não encontrada"));
        when(service.create(any())).thenThrow(new ResourceNotFoundException("Categoria não encontrada"));
        mvc.perform(get("/api/suggestions/7")).andExpect(status().isNotFound()).andExpect(jsonPath("$.message").value("Sugestão não encontrada"));
        mvc.perform(post("/api/suggestions").contentType(MediaType.APPLICATION_JSON).content(VALID))
            .andExpect(status().isNotFound()).andExpect(jsonPath("$.fieldErrors").isEmpty());
    }
    @Test void protocolStatusesAndGenericError() throws Exception {
        mvc.perform(patch("/api/suggestions/7").contentType(MediaType.APPLICATION_JSON).content(VALID))
            .andExpect(status().isMethodNotAllowed()).andExpect(header().exists("Allow")).andExpect(jsonPath("$.status").value(405));
        mvc.perform(post("/api/suggestions").contentType(MediaType.TEXT_PLAIN).content("texto"))
            .andExpect(status().isUnsupportedMediaType()).andExpect(jsonPath("$.status").value(415));
        when(service.get(7L)).thenThrow(new RuntimeException("SQL password secret"));
        mvc.perform(get("/api/suggestions/7")).andExpect(status().isInternalServerError())
            .andExpect(jsonPath("$.message").value("Erro interno do servidor"));
    }
}
```

### `src/test/resources/application-test.yaml`

Substitua o conteúdo inteiro deste arquivo (ou crie-o se ainda não existir). Preserve os demais arquivos. Os testes de integração utilizam somente suggestions_test.

```yaml
app:
  jwt:
    secret-base64: dGVzdC1vbmx5LXNlY3JldC1ub3QtZm9yLXByb2R1Y3Rpb24h
    ttl-seconds: 900
spring:
  datasource:
    hikari:
      connection-timeout: 3000
    url: jdbc:mariadb://localhost:3306/suggestions_test
    username: root
    password: ""
```

### `src/test/java/br/com/etecalbertoferes/CaixaDeSugestoes/AdminSeedIntegrationTest.java`

Crie este arquivo. Confere se a migration criou a conta ativa com BCrypt compatível com a senha didática.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class AdminSeedIntegrationTest extends ContextTest {
    @Test void migrationCreatesActiveAdministratorWithBcryptPassword() {
        var admin = accounts.findByUsername("admin").orElseThrow();
        assertThat(admin.active()).isTrue();
        assertThat(admin.passwordHash()).isNotEqualTo("Admin@123456");
        assertThat(passwordEncoder.matches("Admin@123456", admin.passwordHash())).isTrue();
    }
}
```

## 3. Executar JWT

Antes de executar a suíte, confira que o perfil test aponta para suggestions_test. Se já utilizou esse banco em uma tentativa anterior, siga o procedimento da aula 1 para recriar somente o banco de testes. Não altere nem apague suggestions.


```powershell
.\gradlew.bat test
.\gradlew.bat bootJar
```

Ao iniciar a API, Flyway aplica V3 e V4. Em banco novo, use usuário `admin` e senha `Admin@123456`. Se já houver uma conta, V4 preserva seus dados: use a credencial existente. Não é necessário comando de cadastro.

```powershell
.\gradlew.bat bootRun
```

A chave é criada em .local/jwt-signing.key e reutilizada. Não versione esse arquivo. JWT dura 900 segundos; faça login novamente quando expirar. Consultas e POST de sugestão são públicos. PUT e DELETE exigem Bearer e administrador ativo.

## 4. Collection final

No Insomnia, use Import > File e escolha [a collection](collections/caixa-de-sugestoes.insomnia.json). No ambiente base, mantenha baseUrl http://localhost:8000 e preencha adminUsername/adminPassword com `admin` / `Admin@123456` (ou a conta já existente). Não compartilhe exportações preenchidas.

Execute as 29 requisições na ordem numérica. 01–02 capturam IDs dos catálogos; 03 faz login e captura accessToken; 06 cria uma sugestão; 07–09 devem retornar 401; 16 atualiza com Bearer; 25 exclui; 26–28 confirmam 404; 29 consulta OpenAPI. Cada requisição contém testes e status esperado. Os casos de erro são resultados esperados, não falhas da atividade. Se o token expirar, repita 03 e continue. Para repetir o fluxo completo, recomece em 01.

### Conferir a atualização do token

Na requisição **03 - Login do administrador**, o script **After-response** valida HTTP 200, tokenType Bearer, accessToken preenchido e expiresIn positivo antes de substituir accessToken no ambiente selecionado. Confira o teste **Contrato e atualização do token** e compare a variável com o accessToken da resposta. Esse uso de scripts segue a [documentação do Insomnia](https://developer.konghq.com/how-to/set-a-value-from-a-response-as-an-environment-variable/).

As requisições 16, 25, 27 e 28 usam `Authorization: Bearer {{ _.accessToken }}`. As requisições 07–08 omitem o token e 09 envia um token inválido de propósito. Os testes de login inválido 04–05 preservam o token obtido em 03. Mantenha o mesmo ambiente durante o fluxo e evite definir accessToken também em pastas, pois essas variáveis têm prioridade sobre o ambiente da collection.

Para obter outro token após a expiração, execute novamente 03; a API não oferece refresh token nesta atividade. A captura ocorre após cada login válido, sem necessidade de copiar o token manualmente.

## Diagnóstico e conclusão

Para falhas de conexão, migrations, porta ou validação do CRUD, consulte o [diagnóstico da aula 1](01-api-banco-crud.md#diagnóstico-e-conclusão). O procedimento para repetir a suíte com `suggestions_test` também está nessa seção.

| Sintoma nesta atividade | Verificação |
| --- | --- |
| Login retorna 401 | Confira a conta criada pela migration V4 e as credenciais no ambiente do Insomnia. |
| PUT ou DELETE retorna 401 | Repita o login 03 e confira `Authorization: Bearer` com o token atualizado no mesmo ambiente. |
| PUT ou DELETE retorna 403 | Confira se o JWT válido contém a role ADMIN. Conta desativada ou token inválido retorna 401. |
| Testes antigos de PUT/DELETE retornam 401 | Confira as adaptações de `ApiIntegrationTest` e `SuggestionControllerTest` na etapa 8. |

- [ ] As migrations V3 e V4 foram aplicadas e o login da conta administrativa foi conferido.
- [ ] A suíte passou, incluindo os novos testes de segurança.
- [ ] O login atualizou accessToken automaticamente no Insomnia.
- [ ] PUT e DELETE exigiram autenticação e funcionaram com o token válido.
- [ ] As 29 requisições retornaram os status esperados.

## Validação desta revisão

Os blocos completos das duas aulas foram extraídos para projetos separados e compilados com Java 17. Os testes passaram com MariaDB 10.4.34: 20 na aula 1 e 40 na aula 2. O projeto-base passou em 40 testes com PostgreSQL 17. As collections Insomnia e Postman executaram as 29 requisições com os scripts de validação em cada banco. Essa execução automatizada valida o contrato HTTP e os scripts exportados; não substitui a conferência visual da importação no aplicativo Insomnia do laboratório.

Os bancos de validação foram executados em Docker no computador do professor. Para os alunos, continuam valendo as instruções de XAMPP desta atividade.

## Guia das 29 requisições

Antes de iniciar, selecione o ambiente importado. Os scripts atualizam courseId, categoryId, suggestionId e accessToken automaticamente; não execute a exclusão antes das consultas e alterações.

| Requisição | HTTP esperado | Dependência |
| --- | --- | --- |
| 01 - Consultar cursos | 200 | Execute após as anteriores; confira variáveis do ambiente. |
| 02 - Consultar categorias | 200 | Execute após as anteriores; confira variáveis do ambiente. |
| 03 - Login do administrador | 200 | Execute após as anteriores; confira variáveis do ambiente. |
| 04 - Rejeitar senha incorreta | 401 | Execute após as anteriores; confira variáveis do ambiente. |
| 05 - Rejeitar login vazio | 400 | Execute após as anteriores; confira variáveis do ambiente. |
| 06 - Criar sugestão identificada | 201 | Execute após as anteriores; confira variáveis do ambiente. |
| 07 - Bloquear alteração sem token | 401 | Execute após as anteriores; confira variáveis do ambiente. |
| 08 - Bloquear exclusão sem token | 401 | Execute após as anteriores; confira variáveis do ambiente. |
| 09 - Bloquear token inválido | 401 | Execute após as anteriores; confira variáveis do ambiente. |
| 10 - Consultar sugestão | 200 | Execute após as anteriores; confira variáveis do ambiente. |
| 11 - Listar sugestões | 200 | Execute após as anteriores; confira variáveis do ambiente. |
| 12 - Filtrar por curso | 200 | Execute após as anteriores; confira variáveis do ambiente. |
| 13 - Filtrar por categoria | 200 | Execute após as anteriores; confira variáveis do ambiente. |
| 14 - Buscar texto literal e combinar filtros | 200 | Execute após as anteriores; confira variáveis do ambiente. |
| 15 - Filtro sem resultado | 200 | Execute após as anteriores; confira variáveis do ambiente. |
| 16 - Atualizar para anônima | 200 | Execute após as anteriores; confira variáveis do ambiente. |
| 17 - Rejeitar campos inválidos | 400 | Execute após as anteriores; confira variáveis do ambiente. |
| 18 - Rejeitar filtro inválido | 400 | Execute após as anteriores; confira variáveis do ambiente. |
| 19 - Rejeitar ID inválido | 400 | Execute após as anteriores; confira variáveis do ambiente. |
| 20 - Categoria inexistente | 404 | Execute após as anteriores; confira variáveis do ambiente. |
| 21 - Curso inexistente | 404 | Execute após as anteriores; confira variáveis do ambiente. |
| 22 - JSON truncado | 400 | Execute após as anteriores; confira variáveis do ambiente. |
| 23 - Método não permitido | 405 | Execute após as anteriores; confira variáveis do ambiente. |
| 24 - Tipo de conteúdo inválido | 415 | Execute após as anteriores; confira variáveis do ambiente. |
| 25 - Excluir sugestão | 204 | Execute após as anteriores; confira variáveis do ambiente. |
| 26 - Confirmar exclusão | 404 | Execute após as anteriores; confira variáveis do ambiente. |
| 27 - Repetir exclusão | 404 | Execute após as anteriores; confira variáveis do ambiente. |
| 28 - Atualizar excluída | 404 | Execute após as anteriores; confira variáveis do ambiente. |
| 29 - OpenAPI JSON | 200 | Execute após as anteriores; confira variáveis do ambiente. |
