# INOVE - Plataforma Educacional Online para Apoio ao Ensino (Back-end)

O **INOVE (Inovação Online para Vivências Educacionais)** é uma plataforma educacional online destinada exclusivamente a docentes, com o objetivo de apoiá-los no aprendizado e na aplicação de ferramentas digitais e metodologias de ensino inovadoras. O back-end foi desenvolvido em Java com Spring Boot, expondo os serviços por meio de uma API RESTful. A autenticação é baseada em JWT, dispensando gerenciamento de sessões no servidor; os arquivos de conteúdo (vídeos e PDFs) são armazenados na AWS S3 e o envio de e-mails transacionais é realizado via Resend API.
Entre as funcionalidades implementadas destacam-se autenticação baseada em JWT com refresh token, gerenciamento de cursos, seções e conteúdos com upload de arquivos via AWS S3, controle de progresso dos discentes, sistema de feedback por curso, gerenciamento de escolas, solicitação e aprovação de instrutores, além de recuperação de senha por e-mail. A arquitetura suporta três perfis de usuário - Administrador, Instrutor e Discente - com controle de acesso baseado em papéis (RBAC).

---

## Tecnologias Utilizadas

- Java 17
- Spring Boot 3.2.7
- Spring Security (JWT com Auth0 java-jwt 4.4.0)
- Spring Data JPA / Hibernate
- PostgreSQL 15+
- Flyway (migrations)
- AWS S3 (upload e streaming de arquivos)
- Resend API (envio de e-mails transacionais)
- Springdoc OpenAPI / Swagger UI (2.6.0)
- ModelMapper (2.3.5)
- Lombok
- Maven
- Docker

---

## Perfis de Usuários

O sistema contempla três perfis com responsabilidades distintas.

**Discente (Usuário):** usuário final, professores que consomem os cursos da plataforma. Realiza matrícula em cursos disponíveis, acessa seções e conteúdos (vídeos e PDFs), acompanha seu próprio progresso por conteúdo, deixa feedbacks sobre os cursos e cancela sua matrícula.

**Instrutor:** responsável pela criação e gestão do conteúdo educacional. Precisa solicitar o papel de instrutor, aguardando aprovação do administrador. Após aprovado, gerencia os cursos que ministra, cria e atualiza seções e conteúdos (com upload de arquivos), e visualiza os cursos disponíveis na plataforma.

**Administrador:** responsável pela administração geral da plataforma. Gerencia todos os usuários cadastrados, aprova ou recusa solicitações de instrutores, administra escolas cadastradas e tem controle total sobre os cursos da plataforma.

---

## Estrutura do Sistema

### Arquitetura

```
src/main/java/br/edu/ifgoiano/inove/
├── InoveApplication.java             # Classe principal Spring Boot
├── config/                           # Configurações de integrações
│   ├── AmazonS3Config.java
│   ├── MailConfig.java
│   └── OpenApiConfig.java
├── controller/                       # Camada REST (9 controllers)
│   ├── AuthenticationController.java
│   ├── CourseController.java
│   ├── SectionController.java
│   ├── ContentController.java
│   ├── UserController.java
│   ├── FeedBackController.java
│   ├── SchoolController.java
│   ├── ImageCourseController.java
│   ├── FileController.java
│   ├── dto/                          # Data Transfer Objects (request/response/mapper)
│   └── exceptions/                   # Handlers globais de exceções HTTP
├── domain/
│   ├── model/                        # Entidades JPA (8 entidades + enums)
│   │   └── enums/                    # UserRole, ContentType, RequestStatus
│   ├── repository/                   # Repositórios Spring Data JPA
│   ├── service/                      # Interfaces de serviço (10 serviços)
│   │   └── implementation/           # Implementações concretas
│   └── utils/                        # Classes utilitárias
└── security/                         # Autenticação e autorização
    ├── SecurityConfiguration.java
    ├── SecurityFilter.java
    └── TokenService.java
```

### Camada de Controllers

| Controller                  | Rota base                                              | Ator           | Responsabilidade                                              |
|-----------------------------|--------------------------------------------------------|----------------|---------------------------------------------------------------|
| `AuthenticationController`  | `/api/inove/auth`                                      | Público        | Login, registro, refresh token, logout e recuperação de senha |
| `UserController`            | `/api/inove/usuarios`                                  | Admin/Instrutor| CRUD de usuários, matrícula e solicitação de instrutor        |
| `SchoolController`          | `/api/inove/escolas`                                   | Admin          | CRUD de escolas cadastradas na plataforma                     |
| `CourseController`          | `/api/inove/cursos`                                    | Admin/Instrutor| CRUD de cursos e acompanhamento de progresso                  |
| `ImageCourseController`     | `/api/inove/cursos`                                    | Admin/Instrutor| Upload e preview de imagem de capa do curso                   |
| `SectionController`         | `/api/inove/cursos/{courseId}/secoes`                  | Instrutor      | CRUD de seções dentro de um curso                             |
| `ContentController`         | `/api/inove/cursos/{courseId}/secoes/{sectionId}/conteudos` | Instrutor | CRUD de conteúdos, upload de arquivos e progresso do discente |
| `FeedBackController`        | `/api/inove/feedbacks`                                 | Discente       | Criar, editar, excluir e listar feedbacks de cursos           |
| `FileController`            | `/api/inove/cursos/secoes/conteudos`                   | Discente       | Streaming e identificação de tipo de arquivo                  |

### Entidades e Relacionamentos

| Entidade               | Tabela                       | Campos principais                                                  | Relacionamentos                                                          |
|------------------------|------------------------------|--------------------------------------------------------------------|--------------------------------------------------------------------------|
| `User`                 | `tb_user`                    | nome, cpf, email, senha, dataNasc, motivacao, role                 | ManyToMany → Course; OneToMany → FeedBack, UserCompletedContent          |
| `School`               | `tb_school`                  | nome, cidade, estado, email, senha, unidadeFederativa              | OneToOne → User                                                          |
| `Course`               | `tb_course`                  | nome, descricao, dataCriacao, dataAtualizacao, imageUrl            | ManyToMany → User; OneToMany → Section, FeedBack                         |
| `Section`              | `tb_section`                 | titulo, descricao                                                  | ManyToOne → Course; OneToMany → Content                                  |
| `Content`              | `tb_content`                 | titulo, descricao, contentType, fileUrl, fileName                  | ManyToOne → Section; OneToMany → UserCompletedContent                    |
| `FeedBack`             | `tb_feedback`                | comentario                                                         | ManyToOne → User, Course                                                 |
| `UserCompletedContent` | `tb_user_completed_content`  | -                                                                  | ManyToOne → User, Content                                                |
| `InstructorRequest`    | `instructor_requests`        | status, dataExpiracao                                              | ManyToOne → User                                                         |

### Enumerações

| Enum             | Valores                                    | Uso                                    |
|------------------|--------------------------------------------|----------------------------------------|
| `UserRole`       | `STUDENT`, `INSTRUCTOR`, `ADMINISTRATOR`   | Papel do usuário no sistema            |
| `ContentType`    | `VIDEO`, `PDF`                             | Tipo do arquivo de conteúdo            |
| `RequestStatus`  | `PENDING`, `APPROVED`, `REJECTED`, `EXPIRED` | Status da solicitação de instrutor   |

### Segurança e Autenticação

- **Autenticação:** JWT Bearer token gerado no login e validado em cada requisição via `SecurityFilter`
- **Refresh Token:** token de renovação com validade de 8 horas; access token com validade de 1 hora
- **RBAC:** controle de acesso baseado em papéis - `STUDENT`, `INSTRUCTOR`, `ADMINISTRATOR`
- **Sessão:** STATELESS - sem estado no servidor; toda informação de contexto é extraída do token JWT
- **Senhas:** armazenadas com hash `BCryptPasswordEncoder`
- **Claims JWT:** id, role, subject (email)

---

## Endpoints da API

### Autenticação (`/api/inove/auth`)

| Método | Rota                       | Ator    | Descrição                              |
|--------|----------------------------|---------|----------------------------------------|
| POST   | `/login`                   | Público | Autenticação e obtenção do token JWT   |
| POST   | `/register`                | Público | Cadastro de novo usuário (discente)    |
| POST   | `/refresh-token`           | Público | Renovar access token via refresh token |
| POST   | `/logout`                  | Autenticado | Invalidar sessão do usuário         |
| POST   | `/forgot-password/{email}` | Público | Solicitar código de recuperação de senha |
| POST   | `/verify-code`             | Público | Verificar código de recuperação        |
| POST   | `/reset-password`          | Público | Redefinir senha com código verificado  |

### Usuários (`/api/inove/usuarios`)

| Método | Rota                                  | Ator           | Descrição                                  |
|--------|---------------------------------------|----------------|--------------------------------------------|
| GET    | `/`                                   | Admin          | Listar todos os usuários                   |
| GET    | `/admin`                              | Admin          | Listar administradores                     |
| GET    | `/discente`                           | Admin          | Listar discentes                           |
| GET    | `/instrutor`                          | Admin          | Listar instrutores                         |
| GET    | `/{userId}`                           | Admin          | Buscar usuário por ID                      |
| POST   | `/admin`                              | Admin          | Criar usuário administrador                |
| POST   | `/discente`                           | Público        | Criar usuário discente                     |
| POST   | `/instrutor`                          | Discente       | Solicitar papel de instrutor               |
| PUT    | `/{userId}`                           | Admin          | Atualizar dados do usuário                 |
| DELETE | `/{userId}`                           | Admin          | Remover usuário                            |
| POST   | `/{userId}/inscreverse/{courseId}`    | Discente       | Matricular discente em um curso            |
| GET    | `/{userId}/cursos`                    | Autenticado    | Listar cursos do usuário                   |
| DELETE | `/{userId}/cursos/{courseId}`         | Discente       | Cancelar matrícula em um curso             |
| GET    | `/instrutor/confirmar`                | Admin          | Confirmar cadastro de instrutor            |
| GET    | `/{id}/school`                        | Admin          | Buscar escola vinculada ao usuário         |

### Escolas (`/api/inove/escolas`)

| Método | Rota         | Ator  | Descrição                   |
|--------|--------------|-------|-----------------------------|
| GET    | `/`          | Admin | Listar todas as escolas      |
| GET    | `/{schoolId}`| Admin | Buscar escola por ID         |
| POST   | `/`          | Admin | Cadastrar nova escola        |
| PUT    | `/{schoolId}`| Admin | Atualizar escola             |
| DELETE | `/{schoolId}`| Admin | Remover escola               |

### Cursos (`/api/inove/cursos`)

| Método | Rota                                           | Ator           | Descrição                                       |
|--------|------------------------------------------------|----------------|-------------------------------------------------|
| POST   | `/`                                            | Admin/Instrutor| Criar novo curso                                |
| GET    | `/`                                            | Autenticado    | Listar todos os cursos                          |
| GET    | `/{id}`                                        | Autenticado    | Buscar curso por ID                             |
| PUT    | `/{id}`                                        | Admin/Instrutor| Atualizar curso                                 |
| DELETE | `/{id}`                                        | Admin/Instrutor| Remover curso                                   |
| GET    | `/instrutor/{instructorId}`                    | Autenticado    | Listar cursos de um instrutor                   |
| GET    | `/{courseId}/discente/{userId}/progresso`      | Discente       | Consultar progresso do discente no curso        |
| POST   | `/{courseId}/upload-imagem-curso`              | Admin/Instrutor| Fazer upload da imagem de capa do curso         |
| GET    | `/{courseId}/preview-imagem`                   | Autenticado    | Visualizar imagem de capa do curso              |

### Seções (`/api/inove/cursos/{courseId}/secoes`)

| Método | Rota            | Ator           | Descrição                   |
|--------|-----------------|----------------|-----------------------------|
| GET    | `/`             | Autenticado    | Listar seções do curso       |
| GET    | `/{sectionId}`  | Autenticado    | Buscar seção por ID          |
| POST   | `/`             | Admin/Instrutor| Criar nova seção             |
| PUT    | `/{sectionId}`  | Admin/Instrutor| Atualizar seção              |
| DELETE | `/{sectionId}`  | Admin/Instrutor| Remover seção                |

### Conteúdos (`/api/inove/cursos/{courseId}/secoes/{sectionId}/conteudos`)

| Método | Rota                                    | Ator           | Descrição                                     |
|--------|-----------------------------------------|----------------|-----------------------------------------------|
| GET    | `/`                                     | Autenticado    | Listar conteúdos da seção                     |
| GET    | `/{contentId}`                          | Autenticado    | Buscar conteúdo por ID                        |
| POST   | `/`                                     | Admin/Instrutor| Criar novo conteúdo                           |
| POST   | `/upload`                               | Admin/Instrutor| Fazer upload de arquivo (vídeo ou PDF) via S3 |
| PUT    | `/{contentId}`                          | Admin/Instrutor| Atualizar conteúdo                            |
| PUT    | `/{contentId}/upload`                   | Admin/Instrutor| Atualizar arquivo do conteúdo                 |
| DELETE | `/{contentId}`                          | Admin/Instrutor| Remover conteúdo                              |
| POST   | `/{contentId}/discente/{userId}/progresso` | Discente    | Marcar conteúdo como concluído                |

### Feedbacks (`/api/inove/feedbacks`)

| Método | Rota                    | Ator     | Descrição                            |
|--------|-------------------------|----------|--------------------------------------|
| POST   | `/`                     | Discente | Criar feedback para um curso         |
| PUT    | `/{feedbackId}`         | Discente | Atualizar feedback                   |
| DELETE | `/{feedbackId}`         | Discente | Excluir feedback                     |
| GET    | `/course/{courseId}`    | Autenticado | Listar feedbacks de um curso      |
| GET    | `/user`                 | Discente | Listar feedbacks do usuário autenticado |

### Arquivos (`/api/inove/cursos/secoes/conteudos`)

| Método | Rota          | Ator        | Descrição                              |
|--------|---------------|-------------|----------------------------------------|
| GET    | `/stream/**`  | Autenticado | Fazer streaming do arquivo de conteúdo |
| GET    | `/type/**`    | Autenticado | Identificar tipo do arquivo            |

---

## Como Executar

### Pré-requisitos

- Java 17+
- Maven 3.9+
- PostgreSQL 15+
- Conta AWS com bucket S3 configurado (`inove-bucket-streaming`)
- Conta Resend com API key habilitada (para envio de e-mails)

### Configuração do banco de dados

```sql
CREATE DATABASE inove;
```

### Configuração (`application.yml`)

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/inove
    username: seu_usuario
    password: sua_senha

jwt:
  secret: seu_secret_jwt
  expiration: 3600000       # 1 hora (access token)
  refresh-expiration: 28800000  # 8 horas (refresh token)

aws:
  accessKeyId: sua_access_key
  secretKey: sua_secret_key
  region: us-east-1
  bucketName: inove-bucket-streaming

resend:
  api-key: sua_api_key_resend
  from: seu_email@dominio.com
```

### Instalação e execução

```bash
# Clonar o repositório
git clone https://github.com/seu-usuario/inove-api.git
cd inove-api

# Compilar e executar
./mvnw spring-boot:run

# Build para produção
./mvnw clean package -DskipTests

# Executar o JAR gerado
java -jar target/inove-*.jar
```

### Executar com Docker

```bash
# Build da imagem
docker build -t inove-api .

# Executar o container
docker run -p 8080:8080 \
  -e SPRING_DATASOURCE_URL=jdbc:postgresql://host:5432/inove \
  -e SPRING_DATASOURCE_USERNAME=usuario \
  -e SPRING_DATASOURCE_PASSWORD=senha \
  -e JWT_SECRET=seu_secret \
  inove-api
```

As migrations do Flyway são executadas automaticamente na inicialização. A API ficará disponível em `http://localhost:8080`.

A documentação interativa Swagger UI pode ser acessada em `http://localhost:8080/swagger-ui/index.html`.

### Exemplos de uso

- Autenticação
- POST
- http://localhost:8080/api/inove/auth/login
- Body JSON: `{ "email": "professor@email.com", "password": "senha123" }`
- Resposta esperada: `{ "token": "eyJhbGciOi...", "refreshToken": "eyJhbGciOi...", "userId": 1 }`
-------------------------------------------------------------
- Cadastro de discente
- POST
- http://localhost:8080/api/inove/usuarios/discente
- Body JSON: `{ "nome": "João Silva", "cpf": "000.000.000-00", "email": "joao@email.com", "password": "senha123", "dataNasc": "1990-05-15" }`
- Resposta esperada: `{ "id": 2, "nome": "João Silva", "email": "joao@email.com", "role": "STUDENT" }`
-------------------------------------------------------------
- Matrícula em curso
- POST
- http://localhost:8080/api/inove/usuarios/2/inscreverse/1
- Headers: `Authorization: Bearer {token}`
- Resposta esperada: `{ "id": 2, "cursos": [{ "id": 1, "nome": "Metodologias Ativas" }] }`
-------------------------------------------------------------
- Marcar conteúdo como concluído
- POST
- http://localhost:8080/api/inove/cursos/1/secoes/1/conteudos/1/discente/2/progresso
- Headers: `Authorization: Bearer {token}`
- Resposta esperada: `{ "contentId": 1, "userId": 2, "concluido": true }`
-------------------------------------------------------------
- Listar feedbacks de um curso
- GET
- http://localhost:8080/api/inove/feedbacks/course/1
- Headers: `Authorization: Bearer {token}`
- Resposta esperada: `[{ "id": 1, "comentario": "Excelente curso!", "usuario": "João Silva" }]`

---

## Membros do Projeto

```
Diego Ribeiro Araújo
Flávio Diniz de Sousa
Ítalo Gonçalves Meireles Faria
João Gabriel de Oliveira Meireles
José Antonio Ribeiro Souto
Pedro Henrique Marques Rocha
```

Projeto Integrador IV - Bacharelado em Sistemas de Informação
Instituto Federal Goiano - Campus Urutaí
