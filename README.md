# Event Tech - Sistema de Gestão de Eventos

## Visão Geral

Event Tech é uma aplicação Java Spring para gerenciamento completo de eventos, fornecendo funcionalidades para organizadores e participantes. O sistema utiliza PostgreSQL para armazenamento de dados e integra vários serviços AWS para garantir alta disponibilidade, escalabilidade e desempenho.

## Tecnologias Utilizadas

### Backend
- **Java 17**
- **Spring Boot 3.2**
- **Spring Data JPA**
- **Spring Security**
- **Spring Cloud AWS**

### Banco de Dados
- **PostgreSQL** (gerenciado pelo Amazon Aurora)

### Infraestrutura AWS
- **Amazon EC2** - Hospedagem da aplicação
- **Amazon Aurora** - Banco de dados PostgreSQL gerenciado
- **Amazon S3** - Armazenamento de imagens e arquivos
- **Amazon CloudFront** - CDN para entrega de conteúdo estático
- **Amazon IAM** - Gerenciamento de permissões e acesso

## Arquitetura do Sistema

```
                   +-------------+
                   |  Amazon     |
+--------+         | CloudFront  |
| Client |-------->| (CDN)       |
+--------+         +-------------+
    |                    |
    v                    v
+--------+         +-------------+
| API    |         | Amazon S3   |
| Gateway|-------->| (Imagens)   |
+--------+         +-------------+
    |
    v
+--------+         +-------------+
| EC2    |         | Amazon      |
| Spring |-------->| Aurora      |
| Boot   |         | PostgreSQL  |
+--------+         +-------------+
```

## Estrutura do Projeto

```
event-tech/
├── src/
│   ├── main/
│   │   ├── java/com/eventtech/
│   │   │   ├── config/        # Configurações da aplicação
│   │   │   ├── controller/    # Controladores REST
│   │   │   ├── dto/           # Data Transfer Objects
│   │   │   ├── exception/     # Exceções customizadas
│   │   │   ├── model/         # Entidades JPA
│   │   │   ├── repository/    # Interfaces Spring Data
│   │   │   ├── service/       # Serviços de negócio
│   │   │   ├── security/      # Configurações de segurança
│   │   │   ├── util/          # Classes utilitárias
│   │   │   └── EventTechApplication.java
│   │   └── resources/
│   │       ├── application.properties  # Configurações principais
│   │       ├── application-dev.properties
│   │       └── application-prod.properties
│   └── test/
│       └── java/com/eventtech/
│           ├── controller/
│           └── service/
├── pom.xml
├── Dockerfile
├── docker-compose.yml
├── README.md
└── .gitignore
```

## Configuração do Banco de Dados

### application.properties
```properties
# Configuração comum do banco de dados
spring.datasource.driver-class-name=org.postgresql.Driver
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true

# Configuração específica do ambiente será carregada baseada no perfil ativo
```

### application-dev.properties
```properties
# Configuração para desenvolvimento local
spring.datasource.url=jdbc:postgresql://localhost:5432/eventtech
spring.datasource.username=postgres
spring.datasource.password=postgres
```

### application-prod.properties
```properties
# Configuração para AWS Aurora PostgreSQL
spring.datasource.url=${JDBC_DATABASE_URL}
spring.datasource.username=${JDBC_DATABASE_USERNAME}
spring.datasource.password=${JDBC_DATABASE_PASSWORD}
spring.datasource.hikari.maximum-pool-size=5
```

## Integrações AWS

### Configuração S3 para Armazenamento de Imagens

```java
@Configuration
@EnableConfigurationProperties(S3Properties.class)
public class S3Config {
    
    @Bean
    public AmazonS3 amazonS3Client(S3Properties s3Properties) {
        BasicAWSCredentials credentials = new BasicAWSCredentials(
            s3Properties.getAccessKey(), 
            s3Properties.getSecretKey()
        );
        
        return AmazonS3ClientBuilder.standard()
            .withCredentials(new AWSStaticCredentialsProvider(credentials))
            .withRegion(s3Properties.getRegion())
            .build();
    }
}
```

### Propriedades S3

```java
@ConfigurationProperties(prefix = "aws.s3")
public class S3Properties {
    private String accessKey;
    private String secretKey;
    private String bucketName;
    private String region;
    
    // Getters e setters
}
```

## Segurança

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf().disable()
            .authorizeRequests()
            .antMatchers("/api/public/**").permitAll()
            .antMatchers("/api/admin/**").hasRole("ADMIN")
            .anyRequest().authenticated()
            .and()
            .oauth2ResourceServer().jwt();
        
        return http.build();
    }
}
```

## Modelos de Dados

### Evento

```java
@Entity
@Table(name = "eventos")
public class Evento {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String nome;
    
    @Column(nullable = false)
    private String descricao;
    
    private LocalDateTime dataHoraInicio;
    
    private LocalDateTime dataHoraFim;
    
    private String localizacao;
    
    private String imagemUrl;
    
    @ManyToOne
    @JoinColumn(name = "organizador_id")
    private Usuario organizador;
    
    @OneToMany(mappedBy = "evento")
    private List<Inscricao> inscricoes;
    
    // Getters, setters, construtores
}
```

### Usuário

```java
@Entity
@Table(name = "usuarios")
public class Usuario {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, unique = true)
    private String email;
    
    @Column(nullable = false)
    private String nome;
    
    @Column(nullable = false)
    private String senha;
    
    @ElementCollection(fetch = FetchType.EAGER)
    @CollectionTable(name = "usuario_roles", joinColumns = @JoinColumn(name = "usuario_id"))
    @Column(name = "role")
    private Set<String> roles = new HashSet<>();
    
    // Getters, setters, construtores
}
```

## API Endpoints

### Eventos

| Método | Endpoint | Descrição |
|--------|----------|-----------|
| GET    | /api/eventos | Lista todos os eventos |
| GET    | /api/eventos/{id} | Obtém um evento específico |
| POST   | /api/eventos | Cria um novo evento (requer autenticação) |
| PUT    | /api/eventos/{id} | Atualiza um evento (requer ser organizador) |
| DELETE | /api/eventos/{id} | Remove um evento (requer ser organizador) |

### Usuários

| Método | Endpoint | Descrição |
|--------|----------|-----------|
| POST   | /api/public/usuarios | Cria um novo usuário |
| POST   | /api/public/login | Realiza login e retorna token JWT |
| GET    | /api/usuarios/perfil | Obtém perfil do usuário logado |
| PUT    | /api/usuarios/perfil | Atualiza perfil do usuário logado |

## Upload de Imagens para S3

```java
@Service
public class ImagemService {
    
    private final AmazonS3 s3Client;
    private final S3Properties s3Properties;
    
    public ImagemService(AmazonS3 s3Client, S3Properties s3Properties) {
        this.s3Client = s3Client;
        this.s3Properties = s3Properties;
    }
    
    public String uploadImagem(MultipartFile arquivo) throws IOException {
        String fileName = UUID.randomUUID().toString() + "_" + arquivo.getOriginalFilename();
        
        ObjectMetadata metadata = new ObjectMetadata();
        metadata.setContentLength(arquivo.getSize());
        metadata.setContentType(arquivo.getContentType());
        
        s3Client.putObject(
            s3Properties.getBucketName(),
            fileName,
            arquivo.getInputStream(),
            metadata
        );
        
        return s3Client.getUrl(s3Properties.getBucketName(), fileName).toString();
    }
}
```

## Implantação

### Docker Compose para Desenvolvimento Local

```yaml
version: '3'
services:
  postgres:
    image: postgres:14
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: eventtech
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
  
  app:
    build: .
    depends_on:
      - postgres
    environment:
      - SPRING_PROFILES_ACTIVE=dev
    ports:
      - "8080:8080"

volumes:
  postgres-data:
```

### Dockerfile

```dockerfile
FROM openjdk:17-slim
WORKDIR /app
COPY target/event-tech-0.0.1-SNAPSHOT.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

## Configuração para Implantação no EC2

1. Instale o Docker no EC2
2. Configure o security group para permitir tráfego na porta 8080
3. Configure IAM role para o EC2 com permissões para S3 e Aurora
4. Execute o contêiner Docker com variáveis de ambiente para AWS

## Scripts de Implantação

### deploy.sh

```bash
#!/bin/bash

# Pull latest changes
git pull

# Build the project
./mvnw clean package -DskipTests

# Deploy to Docker
docker-compose down
docker-compose build
docker-compose up -d
```

## Próximos Passos

1. Implementar CI/CD com AWS CodePipeline
2. Configurar balanceamento de carga com Elastic Load Balancer
3. Adicionar monitoramento com CloudWatch
4. Implementar backup automatizado com AWS Backup
5. Configurar ambiente de staging separado

## Requisitos de Sistema

- JDK 17 ou superior
- Maven 3.8 ou superior
- Docker e Docker Compose
- Conta AWS com acesso aos serviços EC2, Aurora e S3

## Contribuição

1. Faça fork do repositório
2. Crie uma branch para sua feature (`git checkout -b feature/nova-funcionalidade`)
3. Faça commit das suas alterações (`git commit -m 'Adiciona nova funcionalidade'`)
4. Faça push para a branch (`git push origin feature/nova-funcionalidade`)
5. Abra um Pull Request
