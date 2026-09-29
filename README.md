<div align="center">

# 💈 Barbearia Network — Arquitetura de Microsserviços

<p align="center">
  <b>Plataforma distribuída para gestão de agendamentos, controle de acesso e notificações transacionais em tempo real.</b>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Java-17-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white" alt="Java 17" />
  <img src="https://img.shields.io/badge/Spring_Boot-4.0.5-6DB33F?style=for-the-badge&logo=spring-boot&logoColor=white" alt="Spring Boot" />
  <img src="https://img.shields.io/badge/Spring_Cloud-2025.1.1-6DB33F?style=for-the-badge&logo=spring&logoColor=white" alt="Spring Cloud" />
  <img src="https://img.shields.io/badge/Docker_Compose-2496ED?style=for-the-badge&logo=docker&logoColor=white" alt="Docker Compose" />
  <img src="https://img.shields.io/badge/MySQL-8.0-4479A1?style=for-the-badge&logo=mysql&logoColor=white" alt="MySQL" />
  <img src="https://img.shields.io/badge/RabbitMQ-AMQP-FF6600?style=for-the-badge&logo=rabbitmq&logoColor=white" alt="RabbitMQ" />
  <img src="https://img.shields.io/badge/OpenAPI_3-Swagger-85EA2D?style=for-the-badge&logo=swagger&logoColor=black" alt="Swagger" />
</p>

</div>

---

## 📌 Visão Geral & Problemas Resolvidos

O **Barbearia Network** é um ecossistema backend construído com arquitetura orientada a serviços (SOA) e eventos (EDA). O projeto resolve os gargalos clássicos de sistemas monolíticos de agendamento:

* **Conflitos de Agenda e Concorrência:** Algoritmos dedicados e queries com travas otimizadas para impedir que dois clientes reservem o mesmo barbeiro no mesmo horário.
* **Sobrecarga Síncrona em Disparos Externos:** O fluxo de confirmação de reservas não trava esperando a resposta de provedores lentos de e-mail; o agendamento é persistido imediatamente e delegado via mensageria assíncrona.
* **Segurança e Isolamento Perimetral:** Microsserviços internos de negócio não ficam expostos à internet pública. Toda autenticação, validação de token JWT e enriquecimento de headers ocorrem na borda (*API Gateway*).
* **Escalabilidade e Desacoplamento:** Cada domínio (identidade, reservas, notificações) possui suas próprias tabelas e seus próprios schemas de banco de dados versionados via Flyway.

---

## 🏛️ Topologia dos Microsserviços

O ecossistema é composto por 5 microsserviços principais integrados sob uma malha de infraestrutura:

```text
                        [ Requisições Externas / Clientes HTTP ]
                                           │
                                           ▼
                            ┌──────────────────────────────┐
                            │    API Gateway (:8080)       │
                            │  (Perímetro, JWT, WebFlux)   │
                            └──────────────┬───────────────┘
                                           │
                    ┌──────────────────────┼──────────────────────┐
                    │                      │                      │
                    ▼                      ▼                      ▼
        ┌──────────────────────┐ ┌──────────────────┐ ┌──────────────────────┐
        │      identity-ms     │ │    booking-ms    │ │     eureka-server    │
        │  (Auth, Users, RBAC) │ │(Regras & Agenda) │ │ (Service Discovery)  │
        └──────────┬───────────┘ └─────────┬────────┘ └──────────────────────┘
                   │                       │
             MySQL │                       ├─── OpenFeign ──▶ (identity-ms)
                   ▼                       │
        ┌──────────────────────┐           ▼ [Eventos AMQP]
        │   Bases de Dados     │     ┌────────────┐
        │  identity_ms (MySQL) │     │  RabbitMQ  │
        │  booking_ms  (MySQL) │     └─────┬──────┘
        └──────────────────────┘           │
                                           ▼
                             ┌───────────────────────────┐
                             │      notification-ms      │
                             │ (Consumer SMTP / JavaMail)│
                             └───────────────────────────┘
```

1. **`eureka-server`:** O *Service Registry* central. Permite que serviços descubram os IPs e portas uns dos outros dinamicamente pelo nome lógico (`lb://`).
2. **`api-gateway`:** A porta de entrada pública reativa (Spring Cloud Gateway com WebFlux). Valida assinaturas JWT, bloqueia requisições não autorizadas (`401`) e injeta os headers `X-User-Id`, `X-User-Email` e `X-User-Roles` para os serviços internos.
3. **`identity-ms`:** Provedor de identidade oficial. Gerencia cadastro, hashing BCrypt, login local, suporte a OAuth2 e emissão de tokens JWT com expiração.
4. **`booking-ms`:** Motor de regras de agendamento, catálogo de serviços e validações de concorrência. Utiliza **OpenFeign + Resilience4j** para chamadas síncronas tolerantes a falhas e emite eventos para o RabbitMQ.
5. **`notification-ms`:** Serviço consumidor de mensageria orientado a eventos (EDA). Ouve a fila do RabbitMQ e processa envios de e-mails transacionais via SMTP/JavaMail com suporte a retentativas automáticas.

---

## 🛠️ Tecnologias Utilizadas

* **Linguagem:** Java 17
* **Framework Principal:** Spring Boot 4.x
* **Ecossistema Distribuído:** Spring Cloud (Netflix Eureka Server/Client, Spring Cloud Gateway, OpenFeign, Resilience4j)
* **Segurança:** Spring Security, Auth0 Java JWT
* **Persistência & Migrations:** Spring Data JPA, Hibernate, MySQL 8.0, Flyway Migration
* **Mensageria & Resiliência:** RabbitMQ (Spring AMQP com políticas de Retry)
* **Envio Transacional:** Spring Boot Starter Mail (JavaMailSender / TLS)
* **Documentação & Contratos:** SpringDoc OpenAPI 3 / Swagger UI
* **Orquestração de Contêineres:** Docker & Docker Compose
* **Qualidade & Testes:** JUnit 5 e Mockito (cobertura unitária nas regras de negócio e validações)
* **CI/CD:** GitHub Actions (pipeline automatizado executando build e suíte de testes a cada push na branch `main`)

---

## 🚀 Como Executar e Testar Localmente

Siga o passo a passo abaixo para inicializar o ecossistema inteiro de uma única vez em contêineres Docker.

### 📋 Pré-requisitos
* **Git** instalado no seu sistema.
* **Docker Desktop** (ou Docker Engine) instalado e **em execução**.

---

### Passo 1: Clonar o Repositório Central com os Submódulos

Como o repositório central gerencia os microsserviços via **Git Submodules**, execute o clone com a flag `--recurse-submodules` para baixar o código-fonte de cada serviço:

```bash
git clone --recurse-submodules https://github.com/yamamotolucas/barbearia-network-main.git
cd barbearia-network-main
```

> 💡 *Se já clonou o projeto sem a flag, basta rodar na raiz do projeto:*  
> `git submodule update --init --recursive`

---

### Passo 2: Configurar os Arquivos `application.properties` de Cada Microsserviço

Para segurança e boas práticas de versionamento, os arquivos de configuração nos repositórios estão salvos como modelos (`application-example.properties`). 

Antes de subir os contêineres, é **obrigatório** acessar a pasta `src/main/resources` de cada microsserviço e renomear (ou copiar) o arquivo para `application.properties`:

* **`eureka-server/src/main/resources/`** $\rightarrow$ renomear `application-example.properties` para `application.properties`
* **`api-gateway/src/main/resources/`** $\rightarrow$ renomear `application-example.properties` para `application.properties`
* **`identity-ms/src/main/resources/`** $\rightarrow$ renomear `application-example.properties` para `application.properties`
* **`booking-ms/src/main/resources/`** $\rightarrow$ renomear `application-example.properties` para `application.properties`
* **`notification-ms/src/main/resources/`** $\rightarrow$ renomear `application-example.properties` para `application.properties`

> 💡 *Dica rápida via terminal na raiz do projeto:*
>
> **Linux / macOS (Bash):**
> ```bash
> for dir in eureka-server api-gateway identity-ms booking-ms notification-ms; do
>   cp $dir/src/main/resources/application-example.properties $dir/src/main/resources/application.properties
> done
> ```
>
> **Windows (PowerShell):**
> ```powershell
> "eureka-server", "api-gateway", "identity-ms", "booking-ms", "notification-ms" | ForEach-Object {
>   Copy-Item "$_\src\main\resources\application-example.properties" "$_\src\main\resources\application.properties"
> }
> ```

---

### Passo 3: Configurar o Arquivo de Variáveis Global (`.env`)

Na raiz do repositório, faça uma cópia do arquivo de exemplo `.env.example` para `.env`:

```bash
# No Linux / macOS:
cp .env.example .env

# No Windows (PowerShell):
copy .env.example .env
```

Abra o arquivo `.env` gerado. As configurações padrão já vêm preparadas para testes locais imediatos (bancos de dados e RabbitMQ).

---

### Passo 4: Subir a Infraestrutura e Microsserviços

Certifique-se de que o **Docker Desktop** está aberto. No terminal (no mesmo nivel que o docker compose se encontra), execute o comando de compilação e inicialização:

```bash
docker compose up --build -d
```

O Docker irá compilar os JARs de cada microsserviço, provisionar os contêineres do MySQL e RabbitMQ, e conectar tudo sob a mesma rede virtual bridge.

Para acompanhar os logs de inicialização de todos os contêineres:

```bash
docker compose logs -f
```

---

## 🌐 Endpoints Úteis & Acesso aos Painéis

Após a subida dos contêineres, as seguintes interfaces ficam disponíveis no seu navegador:

| Serviço / Recurso | URL | Descrição |
| :--- | :--- | :--- |
| **API Gateway** | `http://localhost:8080` | Ponto de entrada único para todas as requisições HTTP |
| **Documentação Swagger (Gateway)** | `http://localhost:8080/webjars/swagger-ui/index.html` | Interface viva para testes e inspeção dos contratos de rotas |
| **Eureka Service Discovery** | `http://localhost:8081` | Painel visual com o catálogo de instâncias registradas |
| **RabbitMQ Management** | `http://localhost:15672` | Painel das filas e exchanges (Usuário: `guest` \| Senha: `guest`) |

---

## 🔑 Contas Pré-cadastradas para Testes (Seed Data)

Ao iniciar o ecossistema, as migrações do **Flyway** (`identity-ms`) realizam a inserção de usuários padrão com diferentes perfis de acesso (RBAC) para testes imediatos de autenticação e agendamento via API Gateway:

| Nome | E-mail | Senha Padrão | Perfil (`user_role`) | Detalhes / Especialidade |
| :--- | :--- | :--- | :--- | :--- |
| **Administrador** | `admin@barbearia.com` | `123456` | `ADMIN` | Acesso administrativo irrestrito |
| **Barbeiro Master** | `barbeiro@barbearia.com` | `123456` | `BARBER` | Cortes Modernos, Degradê e Barboterapia (Rating: 5.0) |
| **Cliente Teste** | `cliente@gmail.com` | `123456` | `CUSTOMER` | Perfil padrão para simulação de agendamentos |

> 🔒 *Todas as senhas foram geradas e persistidas no banco utilizando hash seguro **BCrypt**.*

---

## 🛑 Como Parar a Aplicação

Para pausar e derrubar todos os contêineres liberando memória do seu sistema:

```bash
docker compose down
```

Se desejar apagar também os volumes persistidos (bancos de dados limpos do zero):

```bash
docker compose down -v
```
