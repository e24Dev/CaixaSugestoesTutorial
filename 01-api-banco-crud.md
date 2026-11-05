# Aula 1 — API, banco e CRUD

As atividades podem ser continuadas após a aula. Siga a sequência e retome do último checkpoint concluído.

Prepare Windows, PowerShell, JDK 17, IntelliJ, XAMPP e Insomnia antes da aula. Não é necessário iniciar Apache para a API; ele é usado pelo phpMyAdmin.

## 1. Projeto e ambiente

No Spring Initializr, gere Gradle Groovy, Java 17, JAR, Spring Web, group `br.com.etecalbertoferes`, artifact/name `CaixaDeSugestoes`, package `br.com.etecalbertoferes.CaixaDeSugestoes`. Extraia em `C:\dev\projetos\CaixaDeSugestoes` e abra a pasta do build.gradle no IntelliJ. Configure SDK e Gradle JVM com Java 17. Preserve o Wrapper binário gerado. Apague application.properties e o teste de contexto gerados; serão substituídos pelos arquivos abaixo.

## 2. Banco e persistência

Inicie MySQL e Apache no painel XAMPP. Abra phpMyAdmin e consulte a versão do servidor: este roteiro utiliza o driver MariaDB, adequado ao MariaDB distribuído pelo XAMPP. Confirme porta 3306 e usuário root sem senha na instalação didática local. Se o laboratório tiver senha, ajuste os YAML da aplicação e de testes. Não exponha esse banco na rede.

Execute na aba SQL do phpMyAdmin:

```sql
CREATE DATABASE suggestions CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE suggestions_test CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

Use bancos novos. O Flyway criará tabelas e catálogos; não crie tabelas manualmente. O banco suggestions_test é exclusivo dos testes.

Os blocos abaixo são arquivos completos. Crie as pastas indicadas e copie cada bloco inteiro; quando o arquivo existir, substitua todo o conteúdo. Execute comandos na raiz do projeto. Não clone o gabarito para construir a atividade.

## Como acompanhar a construção

No IntelliJ, clique com o botão direito em src/main/java e use New > Package para os packages indicados. Dentro deles, use New > Java Class; para interfaces e records, substitua o conteúdo pelo bloco completo. Arquivos YAML e SQL ficam em src/main/resources. Testes ficam em src/test/java, nunca em src/main/java.

O caminho acima de cada bloco identifica o arquivo exato. Não cole todos os blocos em uma classe. Os arquivos de um grupo dependem dos demais: mensagens temporárias de símbolo não encontrado são esperadas durante a cópia. Execute apenas depois de terminar o conjunto indicado.

### Índice da construção

- [Etapa 1: Projeto e dependências](#etapa-1)
- [Etapa 2: Conexão e migrations](#etapa-2)
- [Etapa 3: Domínio e contratos](#etapa-3)
- [Etapa 4: Persistência JPA](#etapa-4)
- [Etapa 5: Casos de uso](#etapa-5)
- [Etapa 7: Contrato HTTP e documentação](#etapa-7)
- [Etapa 8: Testes automatizados](#etapa-8)

<a id="etapa-1"></a>

## Etapa 1 — Projeto e dependências

### `.gitignore`

Crie este arquivo. Arquivo de suporte necessário para executar o projeto com a configuração desta aula.

```text
HELP.md
.gradle
build/
!gradle/wrapper/gradle-wrapper.jar
!**/src/main/**/build/
!**/src/test/**/build/

### STS ###
.apt_generated
.classpath
.factorypath
.project
.settings
.springBeans
.sts4-cache
bin/
!**/src/main/**/bin/
!**/src/test/**/bin/

### IntelliJ IDEA ###
.idea
*.iws
*.iml
*.ipr
out/
!**/src/main/**/out/
!**/src/test/**/out/

### NetBeans ###
/nbproject/private/
/nbbuild/
/dist/
/nbdist/
/.nb-gradle/

### VS Code ###
.vscode/

# Configuração local
.env
.env.*
!.env.example
```

### `build.gradle`

Crie este arquivo. Define plugins e bibliotecas. Depois de salvar, clique em Reload All Gradle Projects no IntelliJ.

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
    implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:3.1.1'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-flyway'
    implementation 'org.flywaydb:flyway-mysql'
    runtimeOnly 'org.mariadb.jdbc:mariadb-java-client'
    testImplementation 'org.springframework.boot:spring-boot-starter-webmvc-test'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}
tasks.named('test') { useJUnitPlatform() }
```

### `settings.gradle`

Crie este arquivo. Fixa o nome do projeto e do JAR gerado.

```groovy
rootProject.name = 'CaixaDeSugestoes'
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/CaixaDeSugestoesApplication.java`

Crie este arquivo. Arquivo de suporte necessário para executar o projeto com a configuração desta aula.

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

**Checkpoint de organização:** confira caminhos, nomes e packages deste grupo antes de continuar. A execução completa está no roteiro abaixo.

<a id="etapa-2"></a>

## Etapa 2 — Conexão e migrations

Crie db/migration dentro de resources. Os nomes V1__ e V2__ usam dois underscores. Flyway registra cada versão em flyway_schema_history; após aplicar, crie uma nova migration para evoluir o schema. Hibernate com validate confere as tabelas e não as recria.

### `src/main/resources/application.yaml`

Crie este arquivo. Define porta HTTP, conexão JDBC e validação do schema. Flyway executa as migrations antes da validação do Hibernate.

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
```

### `src/main/resources/db/migration/V1__create_tables.sql`

Crie este arquivo. Cria as tabelas e suas chaves estrangeiras. AUTO_INCREMENT gera os IDs; as referências mantêm sugestões ligadas aos catálogos.

```sql
CREATE TABLE courses (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL UNIQUE
);
CREATE TABLE categories (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL UNIQUE
);
CREATE TABLE suggestions (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    title TEXT NOT NULL,
    content TEXT NOT NULL,
    author TEXT,
    created_at TIMESTAMP(6) NOT NULL,
    course_id BIGINT NOT NULL,
    category_id BIGINT NOT NULL,
    FOREIGN KEY (course_id) REFERENCES courses(id),
    FOREIGN KEY (category_id) REFERENCES categories(id)
);
CREATE INDEX suggestions_course_idx ON suggestions(course_id);
CREATE INDEX suggestions_category_idx ON suggestions(category_id);
CREATE INDEX suggestions_created_idx ON suggestions(created_at DESC, id DESC);
```

### `src/main/resources/db/migration/V2__seed_catalogs.sql`

Crie este arquivo. Insere quatro cursos e três categorias didáticos. Consulte os IDs pela API em vez de presumir valores.

```sql
INSERT INTO courses(name) VALUES
('Desenvolvimento de Software Multiplataforma'),
('Gestão Empresarial'),
('Análise e Desenvolvimento de Sistemas'),
('Administração de Empresas');
INSERT INTO categories(name) VALUES ('Sugestão'), ('Crítica'), ('Elogio');
```

**Checkpoint de organização:** confira caminhos, nomes e packages deste grupo antes de continuar. A execução completa está no roteiro abaixo.

<a id="etapa-3"></a>

## Etapa 3 — Domínio e contratos

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/domain/Category.java`

Crie este arquivo. Define os dados e os contratos da aplicação sem depender de HTTP ou JPA.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.domain;

public record Category(Long id, String name) {}
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/domain/CategoryRepository.java`

Crie este arquivo. Define os dados e os contratos da aplicação sem depender de HTTP ou JPA.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.domain;

import java.util.List;
import java.util.Optional;
public interface CategoryRepository {
    List<Category> findAll();
    Optional<Category> findById(Long id);
}
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/domain/Course.java`

Crie este arquivo. Define os dados e os contratos da aplicação sem depender de HTTP ou JPA.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.domain;

public record Course(Long id, String name) {}
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/domain/CourseRepository.java`

Crie este arquivo. Define os dados e os contratos da aplicação sem depender de HTTP ou JPA.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.domain;

import java.util.List;
import java.util.Optional;
public interface CourseRepository {
    List<Course> findAll();
    Optional<Course> findById(Long id);
}
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/domain/ResourceNotFoundException.java`

Crie este arquivo. Define os dados e os contratos da aplicação sem depender de HTTP ou JPA.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.domain;

public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) { super(message); }
}
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/domain/Suggestion.java`

Crie este arquivo. Define os dados e os contratos da aplicação sem depender de HTTP ou JPA.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.domain;

import java.time.Instant;
public record Suggestion(Long id, String title, String content, String author,
                         Instant createdAt, Course course, Category category) {}
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/domain/SuggestionRepository.java`

Crie este arquivo. Define os dados e os contratos da aplicação sem depender de HTTP ou JPA.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.domain;

import java.util.List;
import java.util.Optional;
public interface SuggestionRepository {
    Suggestion save(Suggestion suggestion);
    Optional<Suggestion> findById(Long id);
    List<Suggestion> search(Long courseId, Long categoryId, String term);
    void delete(Long id);
}
```

**Checkpoint de organização:** confira caminhos, nomes e packages deste grupo antes de continuar. A execução completa está no roteiro abaixo.

<a id="etapa-4"></a>

## Etapa 4 — Persistência JPA

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/infrastructure/persistence/CategoryEntity.java`

Crie este arquivo. Mapeia a tabela para JPA. As anotações descrevem colunas; o schema é criado pelas migrations.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.infrastructure.persistence;

import jakarta.persistence.*;
@Entity
@Table(name = "categories")
public class CategoryEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;
    @Column(nullable = false, unique = true, length = 255)
    String name;
    Long getId() { return id; }
    String getName() { return name; }
    protected CategoryEntity() {}
}
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/infrastructure/persistence/CategoryJpaRepository.java`

Crie este arquivo. O Spring Data fornece operações de banco a partir desta interface, sem implementação manual.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.infrastructure.persistence;

import java.util.List;
import org.springframework.data.jpa.repository.JpaRepository;
interface CategoryJpaRepository extends JpaRepository<CategoryEntity, Long> {
    List<CategoryEntity> findAllByOrderByNameAsc();
}
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/infrastructure/persistence/CategoryPersistenceAdapter.java`

Crie este arquivo. Implementa a interface do domínio, convertendo entidades JPA em objetos usados pelos serviços.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.infrastructure.persistence;

import java.util.List;
import java.util.Optional;
import org.springframework.stereotype.Repository;
import br.com.etecalbertoferes.CaixaDeSugestoes.domain.*;
@Repository
public class CategoryPersistenceAdapter implements CategoryRepository {
    private final CategoryJpaRepository repository;
    public CategoryPersistenceAdapter(CategoryJpaRepository repository) { this.repository = repository; }
    public List<Category> findAll() { return repository.findAllByOrderByNameAsc().stream().map(e -> new Category(e.id, e.name)).toList(); }
    public Optional<Category> findById(Long id) { return repository.findById(id).map(e -> new Category(e.id, e.name)); }
}
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/infrastructure/persistence/CourseEntity.java`

Crie este arquivo. Mapeia a tabela para JPA. As anotações descrevem colunas; o schema é criado pelas migrations.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.infrastructure.persistence;

import jakarta.persistence.*;
@Entity
@Table(name = "courses")
public class CourseEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;
    @Column(nullable = false, unique = true, length = 255)
    String name;
    Long getId() { return id; }
    String getName() { return name; }
    protected CourseEntity() {}
}
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/infrastructure/persistence/CourseJpaRepository.java`

Crie este arquivo. O Spring Data fornece operações de banco a partir desta interface, sem implementação manual.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.infrastructure.persistence;

import java.util.List;
import org.springframework.data.jpa.repository.JpaRepository;
interface CourseJpaRepository extends JpaRepository<CourseEntity, Long> {
    List<CourseEntity> findAllByOrderByNameAsc();
}
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/infrastructure/persistence/CoursePersistenceAdapter.java`

Crie este arquivo. Implementa a interface do domínio, convertendo entidades JPA em objetos usados pelos serviços.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.infrastructure.persistence;

import java.util.List;
import java.util.Optional;
import org.springframework.stereotype.Repository;
import br.com.etecalbertoferes.CaixaDeSugestoes.domain.*;
@Repository
public class CoursePersistenceAdapter implements CourseRepository {
    private final CourseJpaRepository repository;
    public CoursePersistenceAdapter(CourseJpaRepository repository) { this.repository = repository; }
    public List<Course> findAll() { return repository.findAllByOrderByNameAsc().stream().map(e -> new Course(e.id, e.name)).toList(); }
    public Optional<Course> findById(Long id) { return repository.findById(id).map(e -> new Course(e.id, e.name)); }
}
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/infrastructure/persistence/SuggestionEntity.java`

Crie este arquivo. Mapeia a tabela para JPA. As anotações descrevem colunas; o schema é criado pelas migrations.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.infrastructure.persistence;

import java.time.Instant;
import jakarta.persistence.*;
@Entity
@Table(name = "suggestions")
public class SuggestionEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;
    @Column(nullable = false, columnDefinition = "text") String title;
    @Column(nullable = false, columnDefinition = "text") String content;
    @Column(columnDefinition = "text") String author;
    @Column(name = "created_at", nullable = false, updatable = false) Instant createdAt;
    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "course_id", nullable = false) CourseEntity course;
    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "category_id", nullable = false) CategoryEntity category;
    protected SuggestionEntity() {}
}
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/infrastructure/persistence/SuggestionJpaRepository.java`

Crie este arquivo. O Spring Data fornece operações de banco a partir desta interface, sem implementação manual.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.infrastructure.persistence;

import java.util.List;
import java.util.Optional;
import org.springframework.data.jpa.repository.*;

interface SuggestionJpaRepository extends JpaRepository<SuggestionEntity, Long>, JpaSpecificationExecutor<SuggestionEntity> {
    @Override
    @EntityGraph(attributePaths = {"course", "category"})
    Optional<SuggestionEntity> findById(Long id);
    @Override
    @EntityGraph(attributePaths = {"course", "category"})
    List<SuggestionEntity> findAll(org.springframework.data.jpa.domain.Specification<SuggestionEntity> spec, org.springframework.data.domain.Sort sort);
}
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/infrastructure/persistence/SuggestionPersistenceAdapter.java`

Crie este arquivo. Implementa a interface do domínio, convertendo entidades JPA em objetos usados pelos serviços.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.infrastructure.persistence;

import java.util.ArrayList;
import java.util.List;
import java.util.Locale;
import java.util.Optional;
import jakarta.persistence.EntityManager;
import jakarta.persistence.criteria.Predicate;
import org.springframework.data.domain.Sort;
import org.springframework.data.jpa.domain.Specification;
import org.springframework.stereotype.Repository;
import br.com.etecalbertoferes.CaixaDeSugestoes.domain.*;

@Repository
public class SuggestionPersistenceAdapter implements SuggestionRepository {
    private final SuggestionJpaRepository repository;
    private final EntityManager em;
    public SuggestionPersistenceAdapter(SuggestionJpaRepository repository, EntityManager em) {
        this.repository = repository; this.em = em;
    }
    public Suggestion save(Suggestion suggestion) {
        SuggestionEntity entity = suggestion.id() == null ? new SuggestionEntity() : repository.findById(suggestion.id()).orElseThrow(() -> new ResourceNotFoundException("Sugestão não encontrada"));
        entity.title = suggestion.title(); entity.content = suggestion.content(); entity.author = suggestion.author();
        entity.createdAt = suggestion.createdAt();
        entity.course = em.getReference(CourseEntity.class, suggestion.course().id());
        entity.category = em.getReference(CategoryEntity.class, suggestion.category().id());
        return map(repository.save(entity));
    }
    public Optional<Suggestion> findById(Long id) { return repository.findById(id).map(this::map); }
    public List<Suggestion> search(Long courseId, Long categoryId, String term) {
        Specification<SuggestionEntity> spec = (root, query, cb) -> {
            List<Predicate> predicates = new ArrayList<>();
            if (courseId != null) predicates.add(cb.equal(root.get("course").get("id"), courseId));
            if (categoryId != null) predicates.add(cb.equal(root.get("category").get("id"), categoryId));
            if (term != null) {
                String pattern = "%" + term.toLowerCase(Locale.ROOT).replace("!", "!!").replace("%", "!%").replace("_", "!_") + "%";
                predicates.add(cb.or(cb.like(cb.lower(root.get("title")), pattern, '!'), cb.like(cb.lower(root.get("content")), pattern, '!')));
            }
            return cb.and(predicates.toArray(Predicate[]::new));
        };
        return repository.findAll(spec, Sort.by(Sort.Order.desc("createdAt"), Sort.Order.desc("id"))).stream().map(this::map).toList();
    }
    public void delete(Long id) { repository.deleteById(id); }
    private Suggestion map(SuggestionEntity e) {
        return new Suggestion(e.id, e.title, e.content, e.author, e.createdAt,
            new Course(e.course.getId(), e.course.getName()), new Category(e.category.getId(), e.category.getName()));
    }
}
```

**Checkpoint de organização:** confira caminhos, nomes e packages deste grupo antes de continuar. A execução completa está no roteiro abaixo.

<a id="etapa-5"></a>

## Etapa 5 — Casos de uso

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/application/CatalogService.java`

Crie este arquivo. Executa os casos de uso e coordena os contratos de persistência.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.application;

import java.util.List;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import br.com.etecalbertoferes.CaixaDeSugestoes.domain.*;
@Service
@Transactional(readOnly = true)
public class CatalogService {
    private final CourseRepository courses;
    private final CategoryRepository categories;
    public CatalogService(CourseRepository courses, CategoryRepository categories) {
        this.courses = courses; this.categories = categories;
    }
    public List<Course> courses() { return courses.findAll(); }
    public List<Category> categories() { return categories.findAll(); }
}
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/application/SuggestionCommand.java`

Crie este arquivo. Executa os casos de uso e coordena os contratos de persistência.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.application;

public record SuggestionCommand(String title, String content, String author, Long courseId, Long categoryId) {}
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/application/SuggestionService.java`

Crie este arquivo. Executa os casos de uso e coordena os contratos de persistência.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.application;

import java.time.Instant;
import java.util.List;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import br.com.etecalbertoferes.CaixaDeSugestoes.domain.*;

@Service
@Transactional(readOnly = true)
public class SuggestionService {
    private final SuggestionRepository suggestions;
    private final CourseRepository courses;
    private final CategoryRepository categories;
    public SuggestionService(SuggestionRepository suggestions, CourseRepository courses, CategoryRepository categories) {
        this.suggestions = suggestions; this.courses = courses; this.categories = categories;
    }
    public List<Suggestion> list(Long courseId, Long categoryId, String q) {
        String term = q == null ? null : q.strip();
        return suggestions.search(courseId, categoryId, term == null || term.isEmpty() ? null : term);
    }
    public Suggestion get(Long id) {
        return suggestions.findById(id).orElseThrow(() -> new ResourceNotFoundException("Sugestão não encontrada"));
    }
    @Transactional
    public Suggestion create(SuggestionCommand command) {
        return suggestions.save(build(null, Instant.now().truncatedTo(java.time.temporal.ChronoUnit.MICROS), command));
    }
    @Transactional
    public Suggestion update(Long id, SuggestionCommand command) {
        Suggestion current = get(id);
        return suggestions.save(build(current.id(), current.createdAt(), command));
    }
    @Transactional
    public void delete(Long id) { get(id); suggestions.delete(id); }
    private Suggestion build(Long id, Instant createdAt, SuggestionCommand command) {
        Course course = courses.findById(command.courseId()).orElseThrow(() -> new ResourceNotFoundException("Curso não encontrado"));
        Category category = categories.findById(command.categoryId()).orElseThrow(() -> new ResourceNotFoundException("Categoria não encontrada"));
        String author = command.author() == null ? null : command.author().strip();
        if (author != null && author.isBlank()) author = null;
        return new Suggestion(id, command.title(), command.content(), author, createdAt, course, category);
    }
}
```

### `src/test/java/br/com/etecalbertoferes/CaixaDeSugestoes/application/SuggestionServiceTest.java`

Crie este arquivo. Verifica o comportamento da API ou de seus componentes. Copie agora e execute somente após concluir todos os arquivos desta etapa.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.application;

import br.com.etecalbertoferes.CaixaDeSugestoes.domain.*;
import java.time.Instant;
import java.util.Optional;
import org.junit.jupiter.api.*;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.*;
import org.mockito.junit.jupiter.MockitoExtension;
import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class SuggestionServiceTest {
    @Mock SuggestionRepository suggestions;
    @Mock CourseRepository courses;
    @Mock CategoryRepository categories;
    @InjectMocks SuggestionService service;
    final Course course = new Course(10L, "Curso");
    final Category category = new Category(20L, "Sugestão");
    SuggestionCommand command(String author) { return new SuggestionCommand(" título ", " conteúdo ", author, 10L, 20L); }
    void catalogs() {
        when(courses.findById(10L)).thenReturn(Optional.of(course));
        when(categories.findById(20L)).thenReturn(Optional.of(category));
        when(suggestions.save(any())).thenAnswer(i -> i.getArgument(0));
    }
    @Test void createNormalizesAuthorAndAssignsTime() {
        catalogs();
        Instant before = Instant.now().truncatedTo(java.time.temporal.ChronoUnit.MICROS);
        Suggestion created = service.create(command("  Pessoa  "));
        assertThat(created.author()).isEqualTo("Pessoa");
        assertThat(created.title()).isEqualTo(" título ");
        assertThat(created.content()).isEqualTo(" conteúdo ");
        assertThat(created.id()).isNull();
        assertThat(created.createdAt()).isBetween(before, Instant.now());
        assertThat(service.create(command(null)).author()).isNull();
        assertThat(service.create(command(" \t ")).author()).isNull();
    }
    @Test void updatePreservesIdentityAndTimestampAndRemovesAuthor() {
        catalogs();
        Instant time = Instant.parse("2026-01-01T00:00:00Z");
        when(suggestions.findById(7L)).thenReturn(Optional.of(new Suggestion(7L, "old", "old", "Nome", time, course, category)));
        Suggestion updated = service.update(7L, command(null));
        assertThat(updated.id()).isEqualTo(7L);
        assertThat(updated.createdAt()).isEqualTo(time);
        assertThat(updated.author()).isNull();
    }
    @Test void searchNormalizesTermAndPreservesFilters() {
        service.list(10L, 20L, "  100%_!  ");
        verify(suggestions).search(10L, 20L, "100%_!");
        service.list(null, null, "  ");
        verify(suggestions).search(null, null, null);
    }
    @Test void missingSuggestionCannotUpdateOrDelete() {
        when(suggestions.findById(7L)).thenReturn(Optional.empty());
        assertThatThrownBy(() -> service.update(7L, command(null))).isInstanceOf(ResourceNotFoundException.class);
        assertThatThrownBy(() -> service.delete(7L)).isInstanceOf(ResourceNotFoundException.class);
        verify(suggestions, never()).save(any());
        verify(suggestions, never()).delete(any());
    }
    @Test void missingCatalogDoesNotSave() {
        when(courses.findById(10L)).thenReturn(Optional.empty());
        assertThatThrownBy(() -> service.create(command(null))).isInstanceOf(ResourceNotFoundException.class).hasMessage("Curso não encontrado");
        when(courses.findById(10L)).thenReturn(Optional.of(course));
        when(categories.findById(20L)).thenReturn(Optional.empty());
        assertThatThrownBy(() -> service.create(command(null))).isInstanceOf(ResourceNotFoundException.class).hasMessage("Categoria não encontrada");
        verify(suggestions, never()).save(any());
    }
    @Test void deleteChecksExistence() {
        when(suggestions.findById(7L)).thenReturn(Optional.of(new Suggestion(7L, "t", "c", null, Instant.now(), course, category)));
        service.delete(7L);
        verify(suggestions).delete(7L);
    }
}
```

**Checkpoint de organização:** confira caminhos, nomes e packages deste grupo antes de continuar. A execução completa está no roteiro abaixo.

<a id="etapa-7"></a>

## Etapa 7 — Contrato HTTP e documentação

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/interfaces/rest/ApiError.java`

Crie este arquivo. Recebe requisições HTTP e traduz entradas e resultados para JSON.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.interfaces.rest;

import java.time.Instant;
import java.util.Map;
public record ApiError(Instant timestamp, int status, String message, String path, Map<String, String> fieldErrors) {}
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/interfaces/rest/ApiExceptionHandler.java`

Crie este arquivo. Converte exceções em status HTTP e corpos de erro padronizados.

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
            result.getResolvableErrors().forEach(error -> fields.putIfAbsent(name == null ? "id" : name, error.getDefaultMessage()));
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
        return respond(HttpStatus.INTERNAL_SERVER_ERROR, "Erro interno do servidor", Map.of(), new HttpHeaders(), request);
    }
    private ResponseEntity<Object> respond(HttpStatusCode status, String message, Map<String, String> fields,
            HttpHeaders headers, WebRequest request) {
        HttpServletRequest servlet = ((ServletWebRequest) request).getRequest();
        return new ResponseEntity<>(new ApiError(Instant.now(), status.value(), message, servlet.getRequestURI(), fields), headers, status);
    }
}
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/interfaces/rest/CatalogController.java`

Crie este arquivo. Recebe requisições HTTP e traduz entradas e resultados para JSON.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.interfaces.rest;

import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.tags.Tag;
import io.swagger.v3.oas.annotations.responses.*;
import io.swagger.v3.oas.annotations.media.*;
import java.util.List;
import org.springframework.web.bind.annotation.*;
import br.com.etecalbertoferes.CaixaDeSugestoes.application.CatalogService;
import br.com.etecalbertoferes.CaixaDeSugestoes.domain.*;
@Tag(name = "Catálogos", description = "Cursos e categorias didáticos, ordenados por nome; somente consulta")
@RestController
@RequestMapping("/api")
public class CatalogController {
    private final CatalogService service;
    public CatalogController(CatalogService service) { this.service = service; }
    @Operation(summary = "Listar cursos", description = "Array de id/name ordenado por name crescente")
    @GetMapping("/courses") public List<Course> courses() { return service.courses(); }
    @Operation(summary = "Listar categorias", description = "Array de id/name ordenado por name crescente")
    @GetMapping("/categories") public List<Category> categories() { return service.categories(); }
}
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/interfaces/rest/OpenApiConfiguration.java`

Crie este arquivo. Recebe requisições HTTP e traduz entradas e resultados para JSON.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.interfaces.rest;

import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Info;
import org.springframework.context.annotation.*;
@Configuration
public class OpenApiConfiguration {
    @Bean OpenAPI api() {
        return new OpenAPI().info(new Info().title("Caixa de Sugestões — Etec Prefeito Alberto Feres")
            .version("1.0.0").description("API de sugestões para consumo pelo projeto web com Thymeleaf. Catálogos didáticos somente para consulta. Datas em UTC."));
    }
}
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/interfaces/rest/SuggestionController.java`

Crie este arquivo. Recebe requisições HTTP e traduz entradas e resultados para JSON.

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
    public SuggestionController(SuggestionService service) { this.service = service; }
    @Operation(summary = "Listar sugestões", description = "Array sem paginação. Filtros combinados por AND; q busca título OU conteúdo sem distinguir maiúsculas. Espaços externos são removidos; vazio é ignorado. %, _ e ! são literais. Ordem: createdAt e id decrescentes. IDs positivos inexistentes retornam [].")
    @ApiResponse(responseCode = "200", description = "Lista de sugestões, inclusive vazia")
    @GetMapping
    public List<SuggestionResponse> list(@Parameter(description = "ID positivo do curso") @RequestParam(required = false) @Positive(message = "ID deve ser positivo") Long courseId,
            @Parameter(description = "ID positivo da categoria") @RequestParam(required = false) @Positive(message = "ID deve ser positivo") Long categoryId, @Parameter(description = "Trecho literal do título ou conteúdo") @RequestParam(required = false) String q) {
        return service.list(courseId, categoryId, q).stream().map(SuggestionResponse::from).toList();
    }
    @Operation(summary = "Consultar sugestão por ID")
    @ApiResponse(responseCode = "200", description = "Sugestão encontrada")
    @GetMapping("/{id}")
    public SuggestionResponse get(@PathVariable @Positive(message = "ID deve ser positivo") Long id) { return SuggestionResponse.from(service.get(id)); }
    @Operation(summary = "Criar sugestão", description = "O servidor gera id e createdAt. Autor ausente, nulo ou em branco é armazenado como null e devolvido como Anônimo. Consulte os IDs dos catálogos antes de enviar.")
    @ApiResponse(responseCode = "201", description = "Sugestão criada; Location contém /api/suggestions/{id}", content = @Content(schema = @Schema(implementation = SuggestionResponse.class)), headers = @io.swagger.v3.oas.annotations.headers.Header(name = "Location", schema = @Schema(type = "string")))
    @PostMapping
    public ResponseEntity<SuggestionResponse> create(@Valid @RequestBody SuggestionRequest request) {
        SuggestionResponse response = SuggestionResponse.from(service.create(request.command()));
        return ResponseEntity.created(URI.create("/api/suggestions/" + response.id())).body(response);
    }
    @Operation(summary = "Substituir campos editáveis", description = "Preserva id e createdAt. Omitir author remove o nome anterior. Curso e categoria são obrigatórios. Falhas não alteram o registro.")
    @ApiResponse(responseCode = "200", description = "Sugestão atualizada")
    @PutMapping("/{id}")
    public SuggestionResponse update(@PathVariable @Positive(message = "ID deve ser positivo") Long id, @Valid @RequestBody SuggestionRequest request) {
        return SuggestionResponse.from(service.update(id, request.command()));
    }
    @Operation(summary = "Excluir sugestão", description = "Exclusão física, preservando curso e categoria. Repetir a exclusão retorna 404.")
    @ApiResponse(responseCode = "204", description = "Excluída, sem corpo", content = @Content)
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable @Positive(message = "ID deve ser positivo") Long id) { service.delete(id); return ResponseEntity.noContent().build(); }
}
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/interfaces/rest/SuggestionRequest.java`

Crie este arquivo. Valida o JSON de entrada antes de chamar o serviço. DTOs definem o contrato HTTP sem expor entidades JPA.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.interfaces.rest;

import jakarta.validation.constraints.*;
import io.swagger.v3.oas.annotations.media.Schema;
import br.com.etecalbertoferes.CaixaDeSugestoes.application.SuggestionCommand;
public record SuggestionRequest(
    @NotBlank(message = "Título é obrigatório") @Schema(description = "Título obrigatório; texto preservado", example = "Mais tomadas na biblioteca") String title,
    @NotBlank(message = "Conteúdo é obrigatório") @Schema(description = "Conteúdo obrigatório; texto preservado", example = "Instalar tomadas nas mesas de estudo.") String content,
    @Schema(description = "Nome opcional; espaços externos removidos. Ausente, nulo ou em branco torna a sugestão anônima", example = "Pessoa Exemplo") String author,
    @NotNull(message = "Curso é obrigatório") @Positive(message = "Curso deve ser positivo") @Schema(description = "ID de curso existente", example = "1") Long courseId,
    @NotNull(message = "Categoria é obrigatória") @Positive(message = "Categoria deve ser positiva") @Schema(description = "ID de categoria existente", example = "1") Long categoryId) {
    public SuggestionCommand command() { return new SuggestionCommand(title, content, author, courseId, categoryId); }
}
```

### `src/main/java/br/com/etecalbertoferes/CaixaDeSugestoes/interfaces/rest/SuggestionResponse.java`

Crie este arquivo. Recebe requisições HTTP e traduz entradas e resultados para JSON.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes.interfaces.rest;

import java.time.Instant;
import io.swagger.v3.oas.annotations.media.Schema;
import br.com.etecalbertoferes.CaixaDeSugestoes.domain.*;
public record SuggestionResponse(Long id, String title, String content, @Schema(description = "Nome normalizado ou Anônimo") String author, @Schema(description = "Data de criação UTC, preservada no PUT", example = "2026-10-08T12:00:00Z") Instant createdAt, Course course, Category category) {
    public static SuggestionResponse from(Suggestion suggestion) {
        return new SuggestionResponse(suggestion.id(), suggestion.title(), suggestion.content(),
            suggestion.author() == null ? "Anônimo" : suggestion.author(), suggestion.createdAt(), suggestion.course(), suggestion.category());
    }
}
```

**Checkpoint de organização:** confira caminhos, nomes e packages deste grupo antes de continuar. A execução completa está no roteiro abaixo.

<a id="etapa-8"></a>

## Etapa 8 — Testes automatizados

### `src/test/java/br/com/etecalbertoferes/CaixaDeSugestoes/ApiIntegrationTest.java`

Crie este arquivo. Verifica o comportamento da API ou de seus componentes. Copie agora e execute somente após concluir todos os arquivos desta etapa.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes;

import java.time.Instant;
import java.util.Map;
import br.com.etecalbertoferes.CaixaDeSugestoes.application.*;
import br.com.etecalbertoferes.CaixaDeSugestoes.domain.*;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.webmvc.test.autoconfigure.AutoConfigureMockMvc;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.transaction.annotation.Transactional;
import static org.assertj.core.api.Assertions.*;
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
        mvc.perform(put(location).contentType(MediaType.APPLICATION_JSON).content(body.replace("  Pessoa  ", "  ")))
            .andExpect(status().isOk()).andExpect(jsonPath("$.author").value("Anônimo"));
        assertThat(service.get(id).createdAt()).isEqualTo(time);
        em.flush();
        assertThat(jdbc.queryForObject("select author from suggestions where id=?", String.class, id)).isNull();
        mvc.perform(delete(location)).andExpect(status().isNoContent()).andExpect(content().string(""));
        mvc.perform(get(location)).andExpect(status().isNotFound());
        mvc.perform(delete(location)).andExpect(status().isNotFound());
        mvc.perform(put(location).contentType(MediaType.APPLICATION_JSON).content(body)).andExpect(status().isNotFound());
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
        mvc.perform(put("/api/suggestions/"+item.id()).contentType(MediaType.APPLICATION_JSON).content(invalid)).andExpect(status().isNotFound());
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

Crie este arquivo. Verifica o comportamento da API ou de seus componentes. Copie agora e execute somente após concluir todos os arquivos desta etapa.

```java
package br.com.etecalbertoferes.CaixaDeSugestoes;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import static org.assertj.core.api.Assertions.assertThat;

@org.springframework.test.context.ActiveProfiles("test")
@SpringBootTest
class ContextTest {
    @Autowired JdbcTemplate jdbc;
    @Test void migrationsAndSeeds() {
        assertThat(jdbc.queryForObject("select count(*) from courses", Long.class)).isEqualTo(4L);
        assertThat(jdbc.queryForObject("select count(*) from categories", Long.class)).isEqualTo(3L);
        assertThat(jdbc.queryForObject("select count(*) from suggestions", Long.class)).isZero();
    }
}
```

### `src/test/java/br/com/etecalbertoferes/CaixaDeSugestoes/interfaces/rest/SuggestionControllerTest.java`

Crie este arquivo. Verifica o comportamento da API ou de seus componentes. Copie agora e execute somente após concluir todos os arquivos desta etapa.

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

Crie este arquivo. Usa exclusivamente suggestions_test. As credenciais devem corresponder ao servidor do laboratório.

```yaml
spring:
  datasource:
    url: jdbc:mariadb://localhost:3306/suggestions_test
    username: root
    password: ""
```

**Checkpoint de organização:** confira caminhos, nomes e packages deste grupo antes de continuar. A execução completa está no roteiro abaixo.

## 3. CRUD e validação

Revise os arquivos copiados: controllers recebem HTTP, serviços executam regras, adaptadores persistem entidades. DTOs validam entradas; o handler transforma falhas em respostas HTTP. Complete todos os blocos antes da compilação.

```powershell
.\gradlew.bat wrapper --gradle-version 9.7.1
.\gradlew.bat compileJava
.\gradlew.bat test
.\gradlew.bat bootRun
```

## 4. Teste manual

Abra http://localhost:8000/swagger-ui.html para consultar os corpos completos das operações. No Insomnia, crie GET /api/courses e GET /api/categories usando essa base URL. Use os IDs retornados no POST /api/suggestions conforme o schema do Swagger; confira 201. Faça GET pelo ID, PUT e DELETE; confira 200 e 204. Consulte o ID excluído: 404. Envie campos obrigatórios vazios: 400. Nesta aula PUT e DELETE são públicos.

Pare com Ctrl+C. Reinicie e confira que os dados e catálogos continuam no banco. Prossiga para a [aula 2](02-seguranca-jwt-insomnia.md).

## Diagnóstico e conclusão

| Sintoma | Verificação |
| --- | --- |
| Connection refused | MySQL deve estar iniciado no XAMPP; confira porta 3306 no painel e no JDBC. |
| Access denied | Confira usuário e senha nos YAML da API e de testes. |
| Unknown database | Crie o banco vazio pelo phpMyAdmin antes de executar. |
| Migration checksum mismatch | Não edite migrations aplicadas; compare o arquivo com a versão original. |
| Porta 8000 ocupada | Pare outra execução da API antes de iniciar uma nova. |
| HTTP 400 | Leia fieldErrors e corrija os campos indicados no JSON. |
| HTTP 401 na aula 2 | Faça login novamente e confira Authorization: Bearer e o token atual. |
| HTTP 404 | Confira rota e ID; após exclusão, esse status é esperado. |

Para repetir a suíte, pare a API e recrie **somente suggestions_test** no phpMyAdmin antes de executar gradlew test. Esse banco é descartável e exclusivo dos testes; nunca aplique esse procedimento em suggestions. Use o mesmo usuário/senha definidos no perfil test.

- [ ] Todos os arquivos foram criados nos caminhos indicados.
- [ ] A compilação terminou com BUILD SUCCESSFUL.
- [ ] A suíte passou com o banco de testes separado.
- [ ] As respostas HTTP foram conferidas no Insomnia.
