# Arquitetura de Fundação para MCPs e AI Agents na AWS

## Contexto e Motivação

Com a crescente adoção de AI Agents e a necessidade de integração padronizada entre LLMs e fontes de dados externas, o Model Context Protocol (MCP) emerge como um padrão fundamental. Esta arquitetura visa estabelecer uma fundação escalável, segura e eficiente para suportar tanto cenários de prova de conceito quanto implementações de produção em larga escala.

## Premissas Arquiteturais

### Premissas Fundamentais

#### 1. Centralização de Acesso via AgentCore Gateway

**Premissa**: Todos os MCPs são acessados exclusivamente via AgentCore Gateway, independentemente do domínio ou contexto de negócio.

**Justificativa Técnica**:
- **Ponto Único de Controle**: Centraliza autenticação, autorização, rate limiting e auditoria
- **Abstração de Complexidade**: Agents não precisam conhecer detalhes de conectividade específicos de cada MCP
- **Observabilidade Unificada**: Métricas, logs e traces centralizados para todos os MCPs
- **Versionamento e Compatibilidade**: Gateway gerencia diferentes versões de MCPs de forma transparente
- **Resiliência**: Circuit breakers, retry policies e failover centralizados

```mermaid
graph LR
    subgraph "Agents Layer"
        A1[Agent A]
        A2[Agent B]
        A3[Agent C]
    end
    
    subgraph "Gateway Layer"
        GW[AgentCore Gateway<br/>Single Point of Access]
    end
    
    subgraph "MCP Layer"
        M1[Finance MCP]
        M2[HR MCP]
        M3[Operations MCP]
        M4[External API MCP]
    end
    
    A1 --> GW
    A2 --> GW
    A3 --> GW
    
    GW --> M1
    GW --> M2
    GW --> M3
    GW --> M4
    
    style GW fill:#ff9800
    style A1 fill:#e3f2fd
    style A2 fill:#e3f2fd
    style A3 fill:#e3f2fd
```

#### 2. Comunicação Agent-to-Agent (A2A) para Extensibilidade
**Premissa**: A comunicação Agent-to-Agent deve ser utilizada para estender capacidades dos agentes através de composição, evitando a criação de agentes monolíticos.

**Justificativa Técnica**:
- **Princípio da Responsabilidade Única**: Cada agent mantém foco em seu domínio específico
- **Composição sobre Herança**: Agents especializados podem ser combinados para resolver problemas complexos
- **Escalabilidade Horizontal**: Agents podem ser escalados independentemente baseado na demanda
- **Manutenibilidade**: Mudanças em um domínio não afetam outros agents
- **Reutilização**: Agents especializados podem ser reutilizados em diferentes contextos

**Padrões de Comunicação A2A**:
- **Orquestração**: Um agent coordenador chama outros agents sequencialmente
- **Coreografia**: Agents colaboram através de eventos assíncronos
- **Pipeline**: Agents processam dados em cadeia (output de um é input do próximo)

```mermaid
graph TB
    subgraph "A2A Communication Patterns"
        subgraph "Orchestration Pattern"
            COORD[Coordinator Agent]
            SPEC1[Specialist Agent A]
            SPEC2[Specialist Agent B]
            COORD --> SPEC1
            COORD --> SPEC2
        end
        
        subgraph "Pipeline Pattern"
            P1[Processing Agent 1]
            P2[Processing Agent 2]
            P3[Processing Agent 3]
            P1 --> P2
            P2 --> P3
        end
        
        subgraph "Event-Driven Pattern"
            E1[Event Producer Agent]
            E2[Event Consumer Agent A]
            E3[Event Consumer Agent B]
            E1 -.-> E2
            E1 -.-> E3
        end
    end
```

#### 3. Integração com Recursos VPC Privada
**Premissa**: MCPs devem suportar integração segura com recursos em VPC privada (RDS, ElastiCache, APIs internas) sem exposição desnecessária.

**Justificativa Técnica**:
- **Segurança por Design**: Recursos sensíveis permanecem em redes privadas
- **Compliance**: Atende requisitos de segurança e conformidade regulatória
- **Performance**: Comunicação interna de baixa latência
- **Controle de Acesso**: Security groups e NACLs para controle granular
#### 4. Acesso Direto via MCP Clients com DCR
**Premissa**: Para casos que requerem conexão direta com MCPs (bypassing agents), utilizar MCP Clients autenticados via Keycloak com Dynamic Client Registration para facilitar configuração.

**Justificativa Técnica**:
- **Flexibilidade**: Suporta casos de uso que não se adequam ao padrão agent-based
- **Configuração Simplificada**: DCR elimina processo manual de registro de clientes
- **Segurança Mantida**: Autenticação e autorização robustas mesmo em acesso direto
- **Auditoria Completa**: Todos os acessos são rastreados e auditados

**Casos de Uso Válidos**:
- Ferramentas de desenvolvimento e debugging
- Integrações legacy que não podem ser refatoradas
- Aplicações batch que processam grandes volumes
- Ferramentas de monitoramento e observabilidade

#### 5. Isolamento de Tenancy
**Premissa**: Cada tenant (organização/projeto) deve ter isolamento lógico completo de dados e configurações.

**Implementação**:
- Namespaces Kubernetes dedicados por tenant
- Políticas de rede restritivas entre tenants
- Encryption keys separadas por tenant
- Métricas e logs segregados

#### 6. Versionamento e Compatibilidade
**Premissa**: MCPs devem suportar versionamento semântico e manter compatibilidade backward por pelo menos 2 versões major.

**Estratégia**:
- Versionamento de APIs via headers ou path
- Deprecation warnings com timeline claro
- Testes automatizados de compatibilidade
- Blue/green deployments para atualizações

#### 7. Observabilidade como Cidadão de Primeira Classe
**Premissa**: Todos os componentes devem implementar observabilidade completa (metrics, logs, traces) desde o primeiro deploy.

**Requisitos**:
- OpenTelemetry para traces distribuídos
- Structured logging em formato JSON
- Métricas de negócio e técnicas
- SLIs/SLOs definidos para cada componente


#### 8. Princípio de Least Privilege
**Premissa**: Todos os componentes operam com permissões mínimas necessárias, com revisão regular de privilégios.

**Controles**:
- IAM roles específicas por função
- Service accounts dedicadas no Kubernetes
- Rotação automática de credenciais
- Auditoria de permissões mensais

### Matriz de Decisão Arquitetural por Cenário

```mermaid
graph TD
    subgraph "Architectural Decision Matrix"
        START{Qual o cenário de uso?}
        
        START -->|Desenvolvimento Interno| INTERNO[Cenário Interno]
        START -->|Business to Consumer| B2C[Cenário B2C]
        START -->|Business to Business| B2B[Cenário B2B]
        
        subgraph "Cenário Interno - Desenvolvimento"
            INTERNO --> DEV_TEAM[Equipes de Desenvolvimento]
            DEV_TEAM --> KEYCLOAK_DCR[Keycloak DCR<br/>Dynamic Client Registration]
            KEYCLOAK_DCR --> DIRECT_GATEWAY[Acesso Direto<br/>AgentCore Gateway]
            DIRECT_GATEWAY --> RUNTIME_MCP[MCPs no<br/>AgentCore Runtime]
        end
        
        subgraph "Cenário B2C - Consumidor Final"
            B2C --> CONSUMER[Aplicação do Consumidor]
            CONSUMER --> BFF_B2C[Backend for Frontend<br/>B2C]
            BFF_B2C --> AGENT_B2C{Agent Location?}
            AGENT_B2C -->|High Scale| EKS_B2C[Agent no EKS]
            AGENT_B2C -->|Prototype/POC| RUNTIME_B2C[Agent no AgentCore Runtime]
            EKS_B2C --> GATEWAY_B2C[AgentCore Gateway]
            RUNTIME_B2C --> GATEWAY_B2C
            GATEWAY_B2C --> MCP_B2C[MCPs no Runtime]
        end
        
        subgraph "Cenário B2B - Business Integration"
            B2B --> PARTNER[Sistema Parceiro/Cliente]
            PARTNER --> BFF_B2B[Backend for Frontend<br/>B2B]
            BFF_B2B --> AGENT_B2B{Agent Complexity?}
            AGENT_B2B -->|Complex Logic| EKS_B2B[Agent no EKS]
            AGENT_B2B -->|Simple Logic| RUNTIME_B2B[Agent no AgentCore Runtime]
            EKS_B2B --> DECISION_B2B{MCP Access Pattern?}
            RUNTIME_B2B --> DECISION_B2B
            DECISION_B2B -->|Via Gateway| GATEWAY_B2B[AgentCore Gateway]
            DECISION_B2B -->|Direct Access| DIRECT_MCP[Direct MCP Access]
            GATEWAY_B2B --> MCP_B2B[MCPs no Runtime]
        end
    end
    
    style INTERNO fill:#4caf50
    style B2C fill:#2196f3
    style B2B fill:#ff9800
    style KEYCLOAK_DCR fill:#9c27b0
    style BFF_B2C fill:#e91e63
    style BFF_B2B fill:#e91e63
```

### Detalhamento dos Cenários Arquiteturais

#### Cenário 1: Interno - Desenvolvimento e Ferramentas

```mermaid
sequenceDiagram
    participant DEV as Equipe de Desenvolvimento
    participant IDE as IDE/Ferramenta
    participant KC as Keycloak
    participant AGW as AgentCore Gateway
    participant MCP as MCP Server
    
    Note over DEV, MCP: Cenário Interno - Acesso Direto
    DEV->>IDE: Configura ferramenta de desenvolvimento
    IDE->>KC: Dynamic Client Registration (DCR)
    KC->>IDE: Client Credentials
    IDE->>KC: Authorization Code Flow + PKCE
    KC->>IDE: Access Token
    IDE->>AGW: Request direto com token
    AGW->>MCP: Processa request
    MCP->>AGW: Response
    AGW->>IDE: Response final
```

**Características do Cenário Interno:**
- **Usuários**: Desenvolvedores, DevOps, QA
- **Autenticação**: Keycloak com DCR para flexibilidade
- **Acesso**: Direto ao AgentCore Gateway
- **MCPs**: Hospedados no AgentCore Runtime
- **Casos de Uso**: Debugging, testes, desenvolvimento de MCPs

#### Cenário 2: B2C - Aplicações Voltadas ao Consumidor

```mermaid
sequenceDiagram
    participant USER as Usuário Final
    participant APP as Aplicação B2C
    participant BFF as BFF B2C
    participant AGENT as AI Agent
    participant AGW as AgentCore Gateway
    participant MCP as MCP Server
    
    Note over USER, MCP: Cenário B2C - Via BFF
    USER->>APP: Interação do usuário
    APP->>BFF: Request via API
    BFF->>BFF: Autenticação/Autorização
    BFF->>AGENT: Request processado
    AGENT->>AGW: MCP Request (se necessário)
    AGW->>MCP: Processa request
    MCP->>AGW: Response
    AGW->>AGENT: Response
    AGENT->>BFF: Response processada
    BFF->>APP: Response final
    APP->>USER: Resultado para usuário
```

**Características do Cenário B2C:**
- **Usuários**: Consumidores finais
- **Arquitetura**: BFF → Agent → Gateway → MCP
- **Agent Location**: EKS (alta escala) ou AgentCore Runtime (POC)
- **Autenticação**: Cognito para M2M entre componentes
- **Casos de Uso**: Chatbots, assistentes virtuais, aplicações móveis

#### Cenário 3: B2B - Integrações Empresariais

```mermaid
sequenceDiagram
    participant PARTNER as Sistema Parceiro
    participant API as API B2B
    participant BFF as BFF B2B
    participant AGENT as AI Agent
    participant DECISION as Decision Point
    participant AGW as AgentCore Gateway
    participant MCP as MCP Server
    
    Note over PARTNER, MCP: Cenário B2B - Flexível
    PARTNER->>API: API Call
    API->>BFF: Request autenticado
    BFF->>AGENT: Business logic request
    AGENT->>DECISION: Avalia necessidade MCP
    
    alt Via Gateway
        DECISION->>AGW: Request via Gateway
        AGW->>MCP: Processa request
        MCP->>AGW: Response
        AGW->>DECISION: Response
    else Direct Access
        DECISION->>MCP: Direct MCP access
        MCP->>DECISION: Direct response
    end
    
    DECISION->>AGENT: Response
    AGENT->>BFF: Processed response
    BFF->>API: Business response
    API->>PARTNER: Final response
```

**Características do Cenário B2B:**
- **Usuários**: Sistemas empresariais, parceiros
- **Arquitetura**: Flexível - com ou sem gateway
- **Agent Location**: EKS (lógica complexa) ou AgentCore Runtime (lógica simples)
- **Padrões de Acesso**: Via Gateway ou acesso direto aos MCPs
- **Casos de Uso**: Integrações ERP, APIs empresariais, automação B2B

### Matriz de Decisão Detalhada

| Critério | Interno | B2C | B2B |
|----------|---------|-----|-----|
| **Autenticação** | Keycloak DCR | Cognito M2M | Cognito M2M |
| **Ponto de Entrada** | Direto no Gateway | BFF | BFF |
| **Agent Hosting** | N/A | EKS/Runtime | EKS/Runtime |
| **MCP Access** | Via Gateway | Via Gateway | Gateway/Direto |
| **Escalabilidade** | Baixa-Média | Alta | Média-Alta |
| **Complexidade** | Baixa | Média | Alta |
| **Latência Alvo** | < 500ms | < 200ms | < 300ms |
| **Casos de Uso** | Dev Tools, Debug | Apps Consumer | Enterprise APIs |

### Configurações por Cenário

#### Configuração Interno
```yaml
internal_scenario:
  authentication:
    provider: "keycloak"
    method: "dcr"
    flow: "authorization_code_pkce"
  
  access_pattern:
    direct_gateway: true
    bff_required: false
  
  mcp_deployment:
    location: "agentcore_runtime"
    scaling: "minimal"
  
  target_users:
    - "developers"
    - "devops"
    - "qa_engineers"
```

#### Configuração B2C
```yaml
b2c_scenario:
  authentication:
    provider: "cognito"
    method: "client_credentials"
    flow: "m2m"
  
  architecture:
    bff_required: true
    agent_location: "eks_or_runtime"
    gateway_access: true
  
  mcp_deployment:
    location: "agentcore_runtime"
    scaling: "auto"
  
  performance_targets:
    latency: "< 200ms"
    throughput: "high"
    availability: "99.9%"
```

#### Configuração B2B
```yaml
b2b_scenario:
  authentication:
    provider: "cognito"
    method: "client_credentials"
    flow: "m2m"
  
  architecture:
    bff_required: true
    agent_location: "eks_or_runtime"
    gateway_access: "flexible"
  
  mcp_deployment:
    location: "agentcore_runtime"
    scaling: "demand_based"
    direct_access: "optional"
  
  integration_patterns:
    - "api_gateway"
    - "event_driven"
    - "batch_processing"
```



## Arquitetura de Alto Nível - Cenários de Uso

```mermaid
graph TB
    subgraph "Cenário Interno - Desenvolvimento"
        DEV_TOOLS[Ferramentas de Desenvolvimento<br/>IDEs, CLIs, Debug Tools]
        KC_INTERNAL[Keycloak<br/>DCR]
        AGW_INTERNAL[AgentCore Gateway<br/>Acesso Direto]
        MCP_INTERNAL[MCP Servers<br/>AgentCore Runtime]
    end
    
    subgraph "Cenário B2C - Consumer Applications"
        CONSUMER_APPS[Aplicações B2C<br/>Mobile, Web, Chatbots]
        BFF_B2C[BFF B2C<br/>Consumer API Layer]
        AGENTS_B2C[AI Agents<br/>EKS ou AgentCore Runtime]
        AGW_B2C[AgentCore Gateway<br/>MCP Orchestration]
        MCP_B2C[MCP Servers<br/>AgentCore Runtime]
    end
    
    subgraph "Cenário B2B - Enterprise Integration"
        ENTERPRISE[Sistemas Empresariais<br/>ERP, CRM, APIs]
        BFF_B2B[BFF B2B<br/>Enterprise API Layer]
        AGENTS_B2B[AI Agents<br/>EKS ou AgentCore Runtime]
        DECISION_B2B{Gateway ou<br/>Acesso Direto?}
        AGW_B2B[AgentCore Gateway<br/>Opcional]
        MCP_B2B[MCP Servers<br/>AgentCore Runtime]
    end
    
    subgraph "Camada de Autenticação"
        KC[Keycloak<br/>DCR para Interno]
        COG[Amazon Cognito<br/>M2M para B2C/B2B]
    end
    
    subgraph "Infraestrutura AWS"
        EKS[Amazon EKS<br/>Agents Produção]
        RUNTIME[AgentCore Runtime<br/>MCPs e Agents POC]
        BR[Amazon Bedrock<br/>LLM Services]
    end
    
    %% Fluxos Cenário Interno
    DEV_TOOLS --> KC_INTERNAL
    KC_INTERNAL --> AGW_INTERNAL
    AGW_INTERNAL --> MCP_INTERNAL
    
    %% Fluxos Cenário B2C
    CONSUMER_APPS --> BFF_B2C
    BFF_B2C --> AGENTS_B2C
    AGENTS_B2C --> AGW_B2C
    AGW_B2C --> MCP_B2C
    
    %% Fluxos Cenário B2B
    ENTERPRISE --> BFF_B2B
    BFF_B2B --> AGENTS_B2B
    AGENTS_B2B --> DECISION_B2B
    DECISION_B2B -->|Via Gateway| AGW_B2B
    DECISION_B2B -->|Direto| MCP_B2B
    AGW_B2B --> MCP_B2B
    
    %% Conexões com Infraestrutura
    KC_INTERNAL -.-> KC
    BFF_B2C -.-> COG
    BFF_B2B -.-> COG
    
    AGENTS_B2C -.-> EKS
    AGENTS_B2B -.-> EKS
    MCP_INTERNAL -.-> RUNTIME
    MCP_B2C -.-> RUNTIME
    MCP_B2B -.-> RUNTIME
    
    AGENTS_B2C -.-> BR
    AGENTS_B2B -.-> BR
    
    style DEV_TOOLS fill:#4caf50
    style CONSUMER_APPS fill:#2196f3
    style ENTERPRISE fill:#ff9800
    style BFF_B2C fill:#e91e63
    style BFF_B2B fill:#e91e63
    style KC_INTERNAL fill:#9c27b0
```

### Comparativo dos Cenários

| Aspecto | Interno | B2C | B2B |
|---------|---------|-----|-----|
| **Complexidade** | Baixa | Média | Alta |
| **Usuários** | Desenvolvedores | Consumidores | Empresas |
| **Autenticação** | Keycloak DCR | Cognito M2M | Cognito M2M |
| **BFF** | Não necessário | Obrigatório | Obrigatório |
| **Agent Location** | N/A | EKS/Runtime | EKS/Runtime |
| **Gateway** | Sempre | Sempre | Opcional |
| **Escalabilidade** | Baixa | Alta | Variável |
| **Latência** | Tolerante | Crítica | Moderada |

## Componentes Principais

### 1. AgentCore Gateway

O **Amazon Bedrock AgentCore Gateway** atua como o componente central da arquitetura, fornecendo:

- **Orquestração de MCPs**: Ponto único de controle para roteamento, autenticação e gerenciamento de ferramentas
- **Transformação de APIs**: Conversão automática de APIs REST existentes em servidores MCP compatíveis
- **Suporte Nativo ao MCP**: Implementação completa do Model Context Protocol conforme especificação oficial
- **Descoberta Inteligente**: Capacidade de descoberta automática de ferramentas disponíveis
- **Infraestrutura Serverless**: Gerenciamento automático de infraestrutura para servidores MCP

**Características Técnicas:**
- Suporte a especificações OpenAPI e modelos Smithy
- Integração nativa com AWS Lambda
- Autorização de entrada e saída integrada
- Isolamento de sessão com microVMs dedicadas

### 2. Backend for Frontend (BFF) - Cenários B2C e B2B

#### Arquitetura BFF para Diferentes Cenários

```mermaid
graph TB
    subgraph "BFF Architecture Pattern"
        subgraph "B2C BFF Layer"
            BFF_MOBILE[Mobile BFF<br/>iOS/Android Optimized]
            BFF_WEB[Web BFF<br/>Browser Optimized]
            BFF_CHAT[Chatbot BFF<br/>Conversation Optimized]
        end
        
        subgraph "B2B BFF Layer"
            BFF_API[API BFF<br/>REST/GraphQL]
            BFF_WEBHOOK[Webhook BFF<br/>Event-Driven]
            BFF_BATCH[Batch BFF<br/>Bulk Operations]
        end
        
        subgraph "Shared Services"
            AUTH_SERVICE[Authentication Service]
            RATE_LIMITER[Rate Limiting]
            CACHE_LAYER[Caching Layer]
            MONITORING[Monitoring & Logging]
        end
        
        subgraph "AI Agents Layer"
            AGENT_CONSUMER[Consumer Agents<br/>Personalization Focus]
            AGENT_BUSINESS[Business Agents<br/>Integration Focus]
        end
    end
    
    %% B2C Connections
    BFF_MOBILE --> AUTH_SERVICE
    BFF_WEB --> AUTH_SERVICE
    BFF_CHAT --> AUTH_SERVICE
    
    BFF_MOBILE --> RATE_LIMITER
    BFF_WEB --> RATE_LIMITER
    BFF_CHAT --> RATE_LIMITER
    
    BFF_MOBILE --> AGENT_CONSUMER
    BFF_WEB --> AGENT_CONSUMER
    BFF_CHAT --> AGENT_CONSUMER
    
    %% B2B Connections
    BFF_API --> AUTH_SERVICE
    BFF_WEBHOOK --> AUTH_SERVICE
    BFF_BATCH --> AUTH_SERVICE
    
    BFF_API --> CACHE_LAYER
    BFF_WEBHOOK --> CACHE_LAYER
    BFF_BATCH --> CACHE_LAYER
    
    BFF_API --> AGENT_BUSINESS
    BFF_WEBHOOK --> AGENT_BUSINESS
    BFF_BATCH --> AGENT_BUSINESS
    
    %% Shared Services
    AUTH_SERVICE --> MONITORING
    RATE_LIMITER --> MONITORING
    CACHE_LAYER --> MONITORING
    
    style BFF_MOBILE fill:#2196f3
    style BFF_WEB fill:#2196f3
    style BFF_CHAT fill:#2196f3
    style BFF_API fill:#ff9800
    style BFF_WEBHOOK fill:#ff9800
    style BFF_BATCH fill:#ff9800
```

#### Responsabilidades dos BFFs

##### BFF B2C - Consumer-Focused
```yaml
b2c_bff_responsibilities:
  authentication:
    - "User session management"
    - "Social login integration"
    - "Token refresh handling"
  
  data_transformation:
    - "Mobile-optimized responses"
    - "Pagination for mobile"
    - "Image resizing/optimization"
  
  personalization:
    - "User preference caching"
    - "Recommendation filtering"
    - "A/B testing integration"
  
  performance:
    - "Response compression"
    - "CDN integration"
    - "Client-specific caching"
```

##### BFF B2B - Enterprise-Focused
```yaml
b2b_bff_responsibilities:
  integration:
    - "Protocol translation (REST/SOAP/GraphQL)"
    - "Data format conversion"
    - "Legacy system adaptation"
  
  security:
    - "API key management"
    - "Rate limiting per tenant"
    - "Audit logging"
  
  business_logic:
    - "Tenant-specific rules"
    - "Business process orchestration"
    - "Compliance validation"
  
  reliability:
    - "Circuit breaker patterns"
    - "Retry mechanisms"
    - "Bulk operation handling"
```

### 3. MCP Servers (AgentCore Runtime)

#### Fluxo de Comunicação MCP

```mermaid
graph LR
    subgraph "MCP Architecture (JSON-RPC 2.0)"
        HOST[Host<br/>LLM Application]
        CLIENT[Client<br/>Connector]
        SERVER[Server<br/>MCP Service]
    end
    
    subgraph "AgentCore Gateway"
        ROUTER[Request Router]
        AUTH[Auth Validator]
        CACHE[Response Cache]
    end
    
    subgraph "MCP Servers"
        DB[Database MCP<br/>PostgreSQL/MySQL]
        API[API Integration MCP<br/>REST APIs]
        FILE[File System MCP<br/>S3/EFS]
        CUSTOM[Custom MCP<br/>Business Logic]
    end
    
    HOST --> CLIENT
    CLIENT --> ROUTER
    ROUTER --> AUTH
    AUTH --> CACHE
    
    CACHE --> DB
    CACHE --> API
    CACHE --> FILE
    CACHE --> CUSTOM
    
    DB --> CACHE
    API --> CACHE
    FILE --> CACHE
    CUSTOM --> CACHE
    
    CACHE --> AUTH
    AUTH --> ROUTER
    ROUTER --> CLIENT
    CLIENT --> HOST
    
    style HOST fill:#e3f2fd
    style CLIENT fill:#f3e5f5
    style SERVER fill:#e8f5e8
    style ROUTER fill:#fff3e0
```

Todos os servidores MCP são executados no **AgentCore Runtime**, proporcionando:

- **Isolamento Seguro**: Cada sessão de usuário é executada em microVMs dedicadas
- **Escalabilidade Automática**: Dimensionamento baseado na demanda
- **Gerenciamento Simplificado**: Abstração da complexidade de infraestrutura
- **Protocolo Padronizado**: Implementação completa do MCP usando JSON-RPC 2.0

**Referências Técnicas:**
- Baseado na especificação MCP v2025-11-25 (modelcontextprotocol.io)
- Comunicação via JSON-RPC 2.0 conforme RFC 7591
- Arquitetura client-host-server padrão do MCP

### 3. AI Agents - Abordagem Dual

#### 3.1 Ambiente de POC (AgentCore Runtime)

Para prototipagem e desenvolvimento inicial:

- **Simplicidade**: Deploy rápido sem configuração de infraestrutura
- **Custo Otimizado**: Modelo serverless com cobrança por uso
- **Desenvolvimento Ágil**: Foco na lógica de negócio sem overhead operacional

#### 3.2 Ambiente de Produção (Amazon EKS)

Para cargas de trabalho críticas e em escala:

- **Alta Disponibilidade**: Distribuição multi-AZ automática
- **Escalabilidade Avançada**: Karpenter e Auto Mode para gerenciamento dinâmico
- **Controle Granular**: Configuração detalhada de recursos e políticas
- **Integração Nativa**: VPC CNI, IRSA, e integração com serviços AWS

**Características do EKS (2025):**
- EKS Auto Mode para gerenciamento automatizado
- EKS Capabilities para orquestração simplificada
- Suporte a EC2, Fargate e nós híbridos
- Integração com EBS, EFS e FSx para armazenamento

## Identificação de Dados do Cliente e Passagem de Contexto

### Arquitetura de Identificação de Cliente

```mermaid
sequenceDiagram
    participant User as Usuário Final
    participant Agent as AI Agent
    participant Gateway as AgentCore Gateway
    participant Cognito as Amazon Cognito
    participant MCP as MCP Server
    participant SM as Secrets Manager
    
    Note over User, MCP: 1. Identificação e Autenticação do Cliente
    User->>Agent: Solicitação com contexto do cliente
    Agent->>SM: Recupera credenciais do service account
    SM->>Agent: Client credentials (client_id, client_secret)
    
    Note over User, MCP: 2. Obtenção de Token com Contexto
    Agent->>Cognito: Client Credentials Grant + Client Context
    Cognito->>Cognito: Valida credenciais e gera JWT
    Cognito->>Agent: Access Token JWT (com claims do cliente)
    
    Note over User, MCP: 3. Passagem de Contexto via Gateway
    Agent->>Gateway: Request + JWT Token + Client Headers
    Gateway->>Gateway: Extrai claims do JWT + headers
    Gateway->>MCP: Request + Contexto do Cliente
    MCP->>Gateway: Response filtrada por cliente
    Gateway->>Agent: Response final
```

### 5.1 Estrutura de Claims JWT para Identificação do Cliente

Baseado na RFC 9068 (JWT Profile for OAuth 2.0 Access Tokens), os tokens JWT devem incluir claims específicos para identificação do cliente:

**Claims Obrigatórios (RFC 9068):**
```json
{
  "iss": "https://cognito-idp.us-east-1.amazonaws.com/us-east-1_POOL",
  "sub": "ai-agent-service-account-id",
  "aud": "https://gateway.agentcore.aws.com",
  "client_id": "finance-agent-001",
  "scope": "mcp:read mcp:write tenant:acme-corp",
  "iat": 1704067200,
  "exp": 1704070800,
  "jti": "unique-token-identifier"
}
```

**Claims Customizados para Contexto do Cliente:**
```json
{
  "tenant_id": "acme-corp-12345",
  "organization_id": "org-finance-division",
  "user_context": {
    "user_id": "user-john-doe",
    "department": "finance",
    "role": "analyst",
    "permissions": ["read:financial-data", "write:reports"]
  },
  "client_metadata": {
    "agent_type": "financial-assistant",
    "version": "2.1.0",
    "deployment_env": "production"
  }
}
```

### 5.2 Passagem de Contexto através do Gateway

#### Headers HTTP Padronizados

O AgentCore Gateway utiliza headers HTTP padronizados para passar informações de contexto:

```http
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
X-Client-ID: finance-agent-001
X-Tenant-ID: acme-corp-12345
X-User-Context: eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
X-Request-ID: req-uuid-12345
X-Correlation-ID: corr-uuid-67890
```

#### Transformação de Contexto no Gateway

```mermaid
graph TB
    subgraph "AgentCore Gateway - Context Processing"
        JWT[JWT Token Validation]
        EXTRACT[Claims Extraction]
        ENRICH[Context Enrichment]
        TRANSFORM[Header Transformation]
        ROUTE[Request Routing]
    end
    
    subgraph "Input Context"
        TOKEN[Access Token JWT]
        HEADERS[Request Headers]
        BODY[Request Body]
    end
    
    subgraph "Output Context"
        MCPHEADERS[MCP Headers]
        MCPBODY[Enhanced Request]
        AUDIT[Audit Trail]
    end
    
    TOKEN --> JWT
    HEADERS --> JWT
    BODY --> JWT
    
    JWT --> EXTRACT
    EXTRACT --> ENRICH
    ENRICH --> TRANSFORM
    TRANSFORM --> ROUTE
    
    ROUTE --> MCPHEADERS
    ROUTE --> MCPBODY
    ROUTE --> AUDIT
    
    style JWT fill:#ff9800
    style EXTRACT fill:#4caf50
    style ENRICH fill:#2196f3
```

### 5.3 Implementação de Context Enrichment

#### Configuração do Gateway para Context Passing

```yaml
# AgentCore Gateway Configuration
context_processing:
  jwt_validation:
    issuer_validation: true
    audience_validation: true
    signature_verification: true
    
  claim_extraction:
    required_claims:
      - "tenant_id"
      - "client_id"
      - "user_context"
    
  header_mapping:
    "tenant_id": "X-Tenant-ID"
    "client_id": "X-Client-ID"
    "user_context": "X-User-Context"
    "organization_id": "X-Org-ID"
    
  context_enrichment:
    tenant_lookup: true
    permission_resolution: true
    audit_logging: true

mcp_routing:
  context_forwarding:
    preserve_original_headers: true
    add_gateway_headers: true
    include_audit_trail: true
```

#### Exemplo de Processamento no MCP Server

```python
# Exemplo de como um MCP Server processa o contexto do cliente
class MCPServerContextHandler:
    def process_request(self, request):
        # Extrai contexto dos headers
        tenant_id = request.headers.get('X-Tenant-ID')
        client_id = request.headers.get('X-Client-ID')
        user_context = self.decode_user_context(
            request.headers.get('X-User-Context')
        )
        
        # Valida permissões baseadas no contexto
        if not self.validate_permissions(tenant_id, user_context):
            raise UnauthorizedError("Insufficient permissions")
        
        # Filtra dados baseado no tenant
        filtered_data = self.filter_by_tenant(
            data=self.get_data(),
            tenant_id=tenant_id,
            user_permissions=user_context.get('permissions', [])
        )
        
        return filtered_data
```

### 5.4 Auditoria e Rastreabilidade

#### Estrutura de Audit Log

```json
{
  "timestamp": "2025-01-02T10:30:00Z",
  "event_type": "mcp_request",
  "correlation_id": "corr-uuid-67890",
  "request_id": "req-uuid-12345",
  "client_context": {
    "agent_id": "finance-agent-001",
    "tenant_id": "acme-corp-12345",
    "user_id": "user-john-doe",
    "client_ip": "10.0.1.100"
  },
  "mcp_context": {
    "server_name": "financial-data-mcp",
    "method": "tools/list",
    "resource_accessed": "/api/v1/financial-reports",
    "data_classification": "confidential"
  },
  "security_context": {
    "token_issued_at": "2025-01-02T10:25:00Z",
    "token_expires_at": "2025-01-02T11:25:00Z",
    "scopes": ["mcp:read", "finance:reports"],
    "permissions_validated": true
  },
  "response_context": {
    "status_code": 200,
    "response_size_bytes": 2048,
    "processing_time_ms": 150,
    "records_returned": 25
  }
}
```

### 5.5 Conformidade com RFCs e Especificações

#### RFC 9068 - JWT Profile for OAuth 2.0 Access Tokens

O gateway implementa validação completa conforme RFC 9068:

- **Validação de Assinatura**: Verificação criptográfica do token
- **Validação de Audience**: Confirmação que o token foi emitido para o gateway
- **Validação de Issuer**: Verificação da origem do token
- **Validação de Tempo**: Verificação de expiração e não-antes

#### RFC 8707 - Resource Indicators for OAuth 2.0

Implementação de Resource Indicators para binding de tokens:

```http
# Durante a solicitação do token
POST /oauth2/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=finance-agent-001
&client_secret=secret
&resource=https://gateway.agentcore.aws.com
&scope=mcp:read mcp:write
```

#### MCP Authorization Specification

Conformidade com a especificação MCP de autorização:

- **Protected Resource Metadata**: Descoberta de servidores de autorização
- **Scope Challenge Handling**: Tratamento de erros de escopo insuficiente
- **Token Audience Binding**: Validação de audience para prevenir token passthrough

### 5.6 Segurança e Isolamento de Dados

#### Princípios de Isolamento

```mermaid
graph TB
    subgraph "Tenant Isolation Architecture"
        subgraph "Tenant A - ACME Corp"
            TA_AGENT[Finance Agent A]
            TA_DATA[Financial Data A]
            TA_MCP[MCP Instance A]
        end
        
        subgraph "Tenant B - XYZ Inc"
            TB_AGENT[HR Agent B]
            TB_DATA[HR Data B]
            TB_MCP[MCP Instance B]
        end
        
        subgraph "Gateway Layer"
            GATEWAY[AgentCore Gateway]
            ROUTER[Context Router]
            VALIDATOR[Token Validator]
        end
        
        subgraph "Security Controls"
            RBAC[Role-Based Access]
            ENCRYPTION[Data Encryption]
            AUDIT[Audit Logging]
        end
    end
    
    TA_AGENT --> GATEWAY
    TB_AGENT --> GATEWAY
    
    GATEWAY --> ROUTER
    ROUTER --> VALIDATOR
    
    VALIDATOR --> TA_MCP
    VALIDATOR --> TB_MCP
    
    TA_MCP --> TA_DATA
    TB_MCP --> TB_DATA
    
    VALIDATOR --> RBAC
    VALIDATOR --> ENCRYPTION
    VALIDATOR --> AUDIT
    
    style GATEWAY fill:#ff9800
    style VALIDATOR fill:#4caf50
    style RBAC fill:#f44336
```

#### Controles de Segurança Implementados

1. **Isolamento de Tenant**: Dados completamente segregados por tenant_id
2. **Validação de Contexto**: Verificação rigorosa de permissões por request
3. **Encryption at Rest**: Dados criptografados com chaves específicas por tenant
4. **Encryption in Transit**: TLS 1.3 para todas as comunicações
5. **Audit Completo**: Log de todas as operações com contexto de segurança

## Estratégia de Autenticação

### Fluxo de Autenticação Completo - Cenários Distintos

#### Cenário 1: MCP Client Direto (ex: Kiro IDE) - DCR Flow

```mermaid
sequenceDiagram
    participant IDE as Kiro IDE
    participant KC as Keycloak
    participant MCP as MCP Server
    
    Note over IDE, MCP: Dynamic Client Registration Flow (RFC 7591)
    IDE->>KC: 1. DCR Request com Client Metadata
    KC->>KC: 2. Valida Software Statement
    KC->>IDE: 3. Client Credentials (client_id, client_secret)
    
    Note over IDE, MCP: Authorization Code Flow com PKCE
    IDE->>KC: 4. Authorization Request + PKCE Challenge
    KC->>KC: 5. User Authentication & Consent
    KC->>IDE: 6. Authorization Code
    IDE->>KC: 7. Token Request + PKCE Verifier
    KC->>IDE: 8. Access Token JWT
    
    Note over IDE, MCP: Direct MCP Access
    IDE->>MCP: 9. MCP Request + Access Token
    MCP->>KC: 10. Token Validation
    KC->>MCP: 11. Token Valid + User Context
    MCP->>IDE: 12. MCP Response
```

#### Cenário 2: Agent-to-Gateway Communication - Client Credentials Flow

```mermaid
sequenceDiagram
    participant Agent as AI Agent
    participant SM as Secrets Manager
    participant COG as Amazon Cognito
    participant AGW as AgentCore Gateway
    participant MCP as MCP Server
    
    Note over Agent, MCP: Service Account Authentication (RFC 6749 Sec 4.4)
    Agent->>SM: 1. Retrieve Service Account Credentials
    SM->>Agent: 2. Client Credentials (client_id, client_secret)
    
    Note over Agent, MCP: M2M Token Request
    Agent->>COG: 3. Client Credentials Grant + Resource Indicator
    COG->>COG: 4. Validate Credentials & Generate JWT
    COG->>Agent: 5. Access Token JWT (RFC 9068)
    
    Note over Agent, MCP: Gateway-Mediated MCP Access
    Agent->>AGW: 6. MCP Request + JWT + Client Context
    AGW->>COG: 7. Token Validation
    COG->>AGW: 8. Token Valid + Claims
    AGW->>AGW: 9. Context Enrichment & Routing
    AGW->>MCP: 10. MCP Request + Enhanced Context
    MCP->>AGW: 11. MCP Response
    AGW->>Agent: 12. Final Response
```

### 4.1 Keycloak para Dynamic Client Registration (DCR) - MCP Clients Diretos

**Casos de Uso Específicos:**
- Conectividade direta via MCP Client (como Kiro IDE)
- Ferramentas de desenvolvimento que precisam de acesso direto aos MCPs
- Cenários que requerem flexibilidade de distribuição e configuração

**Justificativa para DCR:**
O DCR é mais adequado para MCP Clients diretos pois oferece uma estratégia flexível para distribuição, permitindo que ferramentas como IDEs se registrem dinamicamente sem necessidade de pré-configuração manual. Conforme RFC 7591, o DCR permite que aplicações cliente se registrem automaticamente com servidores de autorização.

**Implementação Técnica:**
- Conformidade com RFC 7591 (OAuth 2.0 Dynamic Client Registration)
- Suporte a software statements para validação de clientes
- Client ID Metadata Documents conforme especificação MCP
- Registro protegido com validação de domínio

**Exemplo de Fluxo DCR para MCP Client:**
```json
{
  "software_id": "KIRO-MCP-CLIENT",
  "client_name": "Kiro IDE MCP Client",
  "client_uri": "https://kiro.ai/mcp-client",
  "grant_types": ["authorization_code"],
  "response_types": ["code"],
  "redirect_uris": ["https://localhost:8080/callback"],
  "token_endpoint_auth_method": "none",
  "code_challenge_methods_supported": ["S256"]
}
```

### 4.2 Amazon Cognito para Comunicação Agent-Gateway (M2M)

**Casos de Uso Específicos:**
- Comunicação Agent → AgentCore Gateway
- Comunicação Gateway → MCP Servers
- Autenticação entre serviços internos da arquitetura

**Justificativa para Client Credentials:**
Para a comunicação entre agentes e o gateway, o Client Credentials Grant (RFC 6749, Seção 4.4) é mais apropriado pois:
- Agentes atuam em nome próprio, não de usuários
- Credenciais podem ser gerenciadas via AWS Secrets Manager
- Identificação clara do agente através de service accounts
- Melhor performance para comunicação M2M em alta escala

**Implementação Técnica:**
- OAuth 2.0 Client Credentials Grant (RFC 6749, Seção 4.4)
- Tokens JWT conforme RFC 9068 (JWT Profile for OAuth 2.0 Access Tokens)
- Integração com AWS Secrets Manager para rotação de credenciais
- Service accounts dedicadas por agente

**Exemplo de Token JWT para Agent (RFC 9068):**
```json
{
  "iss": "https://cognito-idp.us-east-1.amazonaws.com/us-east-1_XXXXXXXXX",
  "sub": "ai-agent-finance-001",
  "aud": "https://gateway.agentcore.aws.com",
  "client_id": "finance-agent-service-account",
  "scope": "mcp:read mcp:write finance:access",
  "iat": 1704067200,
  "exp": 1704070800,
  "jti": "unique-token-id"
}
```

**Vantagens da Abordagem Dual:**
- **Separação de Responsabilidades**: DCR para flexibilidade de distribuição, Client Credentials para M2M confiável
- **Segurança Otimizada**: Credenciais gerenciadas centralmente para agentes, registro dinâmico para clientes
- **Escalabilidade**: Cognito otimizado para M2M em alta escala, Keycloak para flexibilidade de registro

## Implementação Detalhada por Cenário

### 5.1 Configuração Cenário Interno - Desenvolvimento

#### Keycloak para DCR (Dynamic Client Registration)
```yaml
# Keycloak Configuration for Internal Scenario
keycloak_internal:
  realm: "development-realm"
  dcr_endpoint: "/auth/realms/development-realm/clients-registrations/openid-connect"
  
  client_registration:
    anonymous_registration: false
    protected_registration: true
    software_statement_required: true
  
  supported_flows:
    - "authorization_code"
    - "client_credentials"
  
  pkce_required: true
  code_challenge_methods:
    - "S256"
  
  scopes:
    - "mcp:read"
    - "mcp:write" 
    - "mcp:debug"
    - "gateway:admin"
```

#### Configuração de Acesso Direto ao Gateway
```yaml
# Direct Gateway Access for Internal Users
internal_gateway_config:
  direct_access: true
  authentication_method: "oauth2_bearer"
  
  rate_limiting:
    requests_per_minute: 1000
    burst_capacity: 100
  
  allowed_operations:
    - "tools/list"
    - "tools/call"
    - "resources/list"
    - "resources/read"
    - "prompts/list"
    - "prompts/get"
  
  debug_features:
    request_tracing: true
    response_logging: true
    performance_metrics: true
```

### 5.2 Configuração Cenário B2C - Consumer Applications

#### BFF B2C Configuration
```yaml
# B2C BFF Configuration
b2c_bff:
  deployment:
    platform: "kubernetes"
    replicas: 3
    auto_scaling:
      min_replicas: 2
      max_replicas: 20
      cpu_threshold: 70
  
  api_gateway:
    rate_limiting:
      anonymous: 100/hour
      authenticated: 1000/hour
    
    cors:
      allowed_origins: ["https://*.myapp.com"]
      allowed_methods: ["GET", "POST"]
      allowed_headers: ["Authorization", "Content-Type"]
  
  caching:
    redis_cluster: true
    ttl_default: 300  # 5 minutes
    ttl_user_data: 3600  # 1 hour
  
  monitoring:
    metrics_enabled: true
    tracing_enabled: true
    log_level: "INFO"
```

#### Agent Configuration for B2C
```yaml
# B2C Agent Configuration
b2c_agents:
  deployment_strategy:
    primary: "agentcore_runtime"  # For POC/MVP
    production: "eks"  # For scale
  
  agent_types:
    - name: "customer-support"
      runtime: "agentcore"
      scaling: "auto"
      max_instances: 10
      
    - name: "product-recommendation"
      runtime: "eks"
      scaling: "predictive"
      max_instances: 50
  
  performance_targets:
    response_time: "< 200ms"
    availability: "99.9%"
    throughput: "1000 rps"
```

### 5.3 Configuração Cenário B2B - Enterprise Integration

#### BFF B2B Configuration
```yaml
# B2B BFF Configuration
b2b_bff:
  deployment:
    platform: "kubernetes"
    replicas: 2
    auto_scaling:
      min_replicas: 1
      max_replicas: 10
      cpu_threshold: 80
  
  api_management:
    versioning: true
    documentation: "openapi_3.0"
    rate_limiting:
      per_tenant: true
      default_limit: "10000/day"
    
    authentication:
      methods: ["api_key", "oauth2", "mutual_tls"]
      token_validation: "cognito"
  
  integration_patterns:
    - "request_response"
    - "event_driven"
    - "batch_processing"
  
  data_transformation:
    formats: ["json", "xml", "csv"]
    validation: "json_schema"
    sanitization: true
```

#### Agent Configuration for B2B
```yaml
# B2B Agent Configuration
b2b_agents:
  deployment_strategy:
    simple_logic: "agentcore_runtime"
    complex_logic: "eks"
  
  agent_types:
    - name: "integration-processor"
      runtime: "eks"
      scaling: "manual"
      resources:
        cpu: "2"
        memory: "4Gi"
      
    - name: "data-transformer"
      runtime: "agentcore"
      scaling: "auto"
      max_instances: 5
  
  mcp_access_patterns:
    gateway_mediated:
      - "external_apis"
      - "database_queries"
    
    direct_access:
      - "file_processing"
      - "batch_operations"
```

### 5.4 Configuração Multi-Cenário do AgentCore Gateway

```yaml
# AgentCore Gateway Multi-Scenario Configuration
gateway_config:
  scenarios:
    internal:
      authentication: "keycloak"
      access_pattern: "direct"
      rate_limiting: "developer_tier"
      
    b2c:
      authentication: "cognito_m2m"
      access_pattern: "via_bff"
      rate_limiting: "consumer_tier"
      
    b2b:
      authentication: "cognito_m2m"
      access_pattern: "flexible"
      rate_limiting: "enterprise_tier"
  
  mcp_servers:
    - name: "database-mcp"
      runtime: "agentcore"
      scenarios: ["internal", "b2c", "b2b"]
      
    - name: "api-integration-mcp"
      runtime: "agentcore"
      scenarios: ["b2c", "b2b"]
      
    - name: "development-mcp"
      runtime: "agentcore"
      scenarios: ["internal"]
  
  routing_rules:
    internal:
      path_prefix: "/internal"
      auth_required: true
      debug_enabled: true
      
    b2c:
      path_prefix: "/consumer"
      auth_required: true
      caching_enabled: true
      
    b2b:
      path_prefix: "/enterprise"
      auth_required: true
      audit_logging: true
```

### 5.2 Configuração do EKS para Produção

#### Arquitetura de Rede EKS

```mermaid
graph TB
    subgraph "AWS Region: us-east-1"
        subgraph "VPC: 10.0.0.0/16"
            subgraph "AZ-1a"
                PUB1[Public Subnet<br/>10.0.1.0/24]
                PRIV1[Private Subnet<br/>10.0.10.0/24]
            end
            
            subgraph "AZ-1b"
                PUB2[Public Subnet<br/>10.0.2.0/24]
                PRIV2[Private Subnet<br/>10.0.20.0/24]
            end
            
            subgraph "AZ-1c"
                PUB3[Public Subnet<br/>10.0.3.0/24]
                PRIV3[Private Subnet<br/>10.0.30.0/24]
            end
            
            IGW[Internet Gateway]
            NAT1[NAT Gateway]
            NAT2[NAT Gateway]
            
            subgraph "EKS Control Plane"
                CP[EKS Control Plane<br/>Multi-AZ]
            end
            
            subgraph "Worker Nodes"
                WN1[AI Agents Nodes<br/>m6i.large]
                WN2[Compute Nodes<br/>c6i.2xlarge]
                WN3[Fargate Pods]
            end
            
            subgraph "Load Balancers"
                ALB[Application Load Balancer]
                NLB[Network Load Balancer]
            end
        end
    end
    
    Internet --> IGW
    IGW --> PUB1
    IGW --> PUB2
    IGW --> PUB3
    
    PUB1 --> NAT1
    PUB2 --> NAT2
    
    NAT1 --> PRIV1
    NAT2 --> PRIV2
    NAT1 --> PRIV3
    
    CP --> PRIV1
    CP --> PRIV2
    CP --> PRIV3
    
    PRIV1 --> WN1
    PRIV2 --> WN2
    PRIV3 --> WN3
    
    ALB --> WN1
    NLB --> WN2
    
    style CP fill:#ff9800
    style WN1 fill:#4caf50
    style WN2 fill:#2196f3
    style WN3 fill:#9c27b0
```

```yaml
# EKS Cluster Configuration
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: ai-agents-production
  region: us-east-1
  version: "1.31"

# EKS Auto Mode habilitado
autoMode:
  enabled: true

# Configuração de nós para diferentes workloads
managedNodeGroups:
  - name: ai-agents-general
    instanceTypes: ["m6i.large", "m6i.xlarge"]
    minSize: 2
    maxSize: 10
    desiredCapacity: 3
    
  - name: ai-agents-compute
    instanceTypes: ["c6i.2xlarge", "c6i.4xlarge"]
    minSize: 0
    maxSize: 20
    desiredCapacity: 0
    taints:
      - key: workload-type
        value: compute-intensive
        effect: NoSchedule

# Integração com serviços AWS
iam:
  withOIDC: true
  serviceAccounts:
    - metadata:
        name: ai-agent-service
        namespace: production
      attachPolicyARNs:
        - arn:aws:iam::aws:policy/AmazonBedrockFullAccess
        - arn:aws:iam::aws:policy/AmazonCognitoPowerUser
```

### 5.3 Configuração de Segurança

```yaml
# Network Policies para isolamento
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ai-agents-isolation
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: ai-agent
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: gateway
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: mcp-servers
    ports:
    - protocol: TCP
      port: 443
```

## Monitoramento e Observabilidade

### Arquitetura de Observabilidade

```mermaid
graph TB
    subgraph "Data Sources"
        AGW[AgentCore Gateway<br/>Metrics & Logs]
        EKS[EKS Cluster<br/>Container Metrics]
        MCP[MCP Servers<br/>Application Metrics]
        AUTH[Auth Services<br/>Security Logs]
    end
    
    subgraph "Collection Layer"
        CW[CloudWatch Agent]
        OTEL[OpenTelemetry Collector]
        FB[Fluent Bit]
    end
    
    subgraph "Storage & Processing"
        CWLOGS[CloudWatch Logs]
        CWMETRICS[CloudWatch Metrics]
        XRAY[AWS X-Ray]
        ES[Amazon OpenSearch]
    end
    
    subgraph "Visualization & Alerting"
        CWDASH[CloudWatch Dashboards]
        GRAFANA[Amazon Managed Grafana]
        SNS[Amazon SNS]
        SLACK[Slack Integration]
    end
    
    AGW --> CW
    AGW --> OTEL
    EKS --> CW
    EKS --> FB
    MCP --> OTEL
    AUTH --> FB
    
    CW --> CWMETRICS
    OTEL --> XRAY
    OTEL --> CWMETRICS
    FB --> CWLOGS
    FB --> ES
    
    CWMETRICS --> CWDASH
    CWLOGS --> CWDASH
    XRAY --> CWDASH
    ES --> GRAFANA
    CWMETRICS --> GRAFANA
    
    CWMETRICS --> SNS
    SNS --> SLACK
    
    style AGW fill:#ff9800
    style EKS fill:#4caf50
    style MCP fill:#2196f3
    style AUTH fill:#9c27b0
```

### 6.1 Métricas Essenciais

- **Gateway Performance**: Latência, throughput, taxa de erro
- **MCP Server Health**: Disponibilidade, tempo de resposta
- **Authentication Metrics**: Taxa de sucesso de autenticação, tokens emitidos
- **Resource Utilization**: CPU, memória, rede por componente

### 6.2 Logging Estruturado

```json
{
  "timestamp": "2025-01-02T10:30:00Z",
  "level": "INFO",
  "component": "agentcore-gateway",
  "event": "mcp_request",
  "details": {
    "client_id": "ai-agent-001",
    "mcp_server": "database-mcp",
    "method": "tools/list",
    "duration_ms": 45,
    "status": "success"
  }
}
```

## Considerações de Segurança

### 7.1 Princípios de Segurança

- **Zero Trust**: Verificação contínua de identidade e autorização
- **Least Privilege**: Permissões mínimas necessárias para cada componente
- **Defense in Depth**: Múltiplas camadas de segurança
- **Encryption Everywhere**: Dados em trânsito e em repouso sempre criptografados

### 7.2 Implementações Específicas

- **mTLS**: Comunicação segura entre todos os componentes
- **JWT Validation**: Validação rigorosa de tokens em todos os endpoints
- **Rate Limiting**: Proteção contra ataques de negação de serviço
- **Audit Logging**: Registro completo de todas as operações sensíveis

## Escalabilidade e Performance

### 8.1 Estratégias de Escalabilidade

- **Horizontal Scaling**: Auto-scaling baseado em métricas de CPU e memória
- **Vertical Scaling**: Ajuste dinâmico de recursos por pod
- **Geographic Distribution**: Deploy multi-região para redução de latência
- **Caching Inteligente**: Cache de respostas MCP frequentes

### 8.2 Otimizações de Performance

- **Connection Pooling**: Reutilização de conexões para MCPs
- **Async Processing**: Processamento assíncrono para operações longas
- **Load Balancing**: Distribuição inteligente de carga entre instâncias
- **Resource Optimization**: Configuração otimizada de recursos por workload

### 8.3 Multi-Region Latency Considerations

#### Arquitetura Multi-Região para AgentCore

```mermaid
graph TB
    subgraph "US-East-1 (Primary)"
        subgraph "Primary Region Components"
            AGW_US[AgentCore Gateway US]
            EKS_US[EKS Cluster US]
            COG_US[Cognito US]
            MCP_US[MCP Servers US]
        end
        
        subgraph "US Data Layer"
            RDS_US[(RDS Primary)]
            S3_US[(S3 Primary)]
            CACHE_US[(ElastiCache US)]
        end
    end
    
    subgraph "EU-West-1 (Secondary)"
        subgraph "European Region Components"
            AGW_EU[AgentCore Gateway EU]
            EKS_EU[EKS Cluster EU]
            COG_EU[Cognito EU]
            MCP_EU[MCP Servers EU]
        end
        
        subgraph "EU Data Layer"
            RDS_EU[(RDS Read Replica)]
            S3_EU[(S3 Cross-Region)]
            CACHE_EU[(ElastiCache EU)]
        end
    end
    
    subgraph "Global Services"
        R53[Route 53]
        CF[CloudFront]
        WAF[AWS WAF]
    end
    
    subgraph "Cross-Region Connectivity"
        TGW[Transit Gateway]
        DX[Direct Connect]
        VPN[VPN Connection]
    end
    
    R53 --> AGW_US
    R53 --> AGW_EU
    CF --> AGW_US
    CF --> AGW_EU
    
    AGW_US --> EKS_US
    AGW_EU --> EKS_EU
    
    EKS_US --> MCP_US
    EKS_EU --> MCP_EU
    
    MCP_US --> RDS_US
    MCP_EU --> RDS_EU
    
    RDS_US -.->|Cross-Region Replication| RDS_EU
    S3_US -.->|Cross-Region Replication| S3_EU
    
    TGW -.->|Low Latency| TGW
    
    style AGW_US fill:#ff9800
    style AGW_EU fill:#ff9800
    style R53 fill:#4caf50
    style CF fill:#2196f3
```

#### Estratégias de Redução de Latência

##### 1. **Roteamento Inteligente Baseado em Geolocalização**

```yaml
# Route 53 Health Check Configuration
health_checks:
  us_east_1:
    endpoint: "https://gateway-us-east-1.agentcore.aws.com/health"
    type: "HTTPS"
    resource_path: "/health"
    failure_threshold: 3
    request_interval: 30
    
  eu_west_1:
    endpoint: "https://gateway-eu-west-1.agentcore.aws.com/health"
    type: "HTTPS"
    resource_path: "/health"
    failure_threshold: 3
    request_interval: 30

routing_policies:
  geolocation:
    - location: "North America"
      target: "gateway-us-east-1.agentcore.aws.com"
      health_check: "us_east_1"
      
    - location: "Europe"
      target: "gateway-eu-west-1.agentcore.aws.com"
      health_check: "eu_west_1"
      
    - location: "Default"
      target: "gateway-us-east-1.agentcore.aws.com"
      health_check: "us_east_1"
```

##### 2. **Cache Distribuído Multi-Região**

```mermaid
graph LR
    subgraph "Multi-Region Caching Strategy"
        subgraph "US Region"
            AGENT_US[AI Agent US]
            CACHE_US[ElastiCache US<br/>Redis Cluster]
            MCP_US[MCP Server US]
        end
        
        subgraph "EU Region"
            AGENT_EU[AI Agent EU]
            CACHE_EU[ElastiCache EU<br/>Redis Cluster]
            MCP_EU[MCP Server EU]
        end
        
        subgraph "Global Cache Layer"
            CDN[CloudFront CDN]
            GLOBAL_CACHE[Global Cache<br/>DynamoDB Global Tables]
        end
    end
    
    AGENT_US --> CACHE_US
    AGENT_EU --> CACHE_EU
    
    CACHE_US -.->|Cache Miss| MCP_US
    CACHE_EU -.->|Cache Miss| MCP_EU
    
    CACHE_US -.->|Cross-Region Sync| CACHE_EU
    CACHE_EU -.->|Cross-Region Sync| CACHE_US
    
    CDN --> CACHE_US
    CDN --> CACHE_EU
    
    GLOBAL_CACHE -.->|Eventual Consistency| CACHE_US
    GLOBAL_CACHE -.->|Eventual Consistency| CACHE_EU
    
    style CDN fill:#ff9800
    style GLOBAL_CACHE fill:#4caf50
```

##### 3. **Otimização de Latência por Componente**

| Componente | Latência Alvo | Estratégia de Otimização |
|------------|---------------|--------------------------|
| **AgentCore Gateway** | < 50ms | Edge locations, connection pooling |
| **Authentication** | < 100ms | Token caching, regional Cognito |
| **MCP Servers** | < 200ms | Regional deployment, warm containers |
| **Database Queries** | < 150ms | Read replicas, query optimization |
| **Cross-Region Sync** | < 500ms | Async replication, eventual consistency |

#### Implementação de Latency Monitoring

##### Métricas de Latência Críticas

```yaml
# CloudWatch Custom Metrics
latency_metrics:
  agent_to_gateway:
    metric_name: "AgentGatewayLatency"
    unit: "Milliseconds"
    dimensions:
      - Region
      - AgentType
      - ClientID
    alarm_threshold: 100
    
  gateway_to_mcp:
    metric_name: "GatewayMCPLatency"
    unit: "Milliseconds"
    dimensions:
      - Region
      - MCPServer
      - Operation
    alarm_threshold: 200
    
  cross_region_sync:
    metric_name: "CrossRegionSyncLatency"
    unit: "Milliseconds"
    dimensions:
      - SourceRegion
      - TargetRegion
      - DataType
    alarm_threshold: 1000

# X-Ray Tracing Configuration
xray_tracing:
  sampling_rate: 0.1
  service_map: true
  trace_segments:
    - "AgentCore-Gateway"
    - "MCP-Servers"
    - "Authentication-Service"
    - "Database-Layer"
```

##### Dashboard de Latência Multi-Região

```mermaid
graph TB
    subgraph "Latency Monitoring Dashboard"
        subgraph "Real-Time Metrics"
            RTM1[Agent → Gateway<br/>P50: 45ms, P99: 120ms]
            RTM2[Gateway → MCP<br/>P50: 80ms, P99: 250ms]
            RTM3[Cross-Region Sync<br/>P50: 300ms, P99: 800ms]
        end
        
        subgraph "Regional Performance"
            RP1[US-East-1<br/>Avg: 65ms]
            RP2[EU-West-1<br/>Avg: 72ms]
            RP3[AP-Southeast-1<br/>Avg: 95ms]
        end
        
        subgraph "Alerts & Actions"
            ALERT1[High Latency Alert<br/>Threshold: 200ms]
            ALERT2[Region Failover<br/>Auto-trigger]
            ALERT3[Scaling Event<br/>Auto-scale]
        end
    end
    
    RTM1 --> ALERT1
    RTM2 --> ALERT2
    RTM3 --> ALERT3
    
    RP1 --> ALERT2
    RP2 --> ALERT2
    RP3 --> ALERT3
    
    style RTM1 fill:#4caf50
    style RTM2 fill:#ff9800
    style RTM3 fill:#f44336
```

#### Estratégias de Failover e Disaster Recovery

##### 1. **Failover Automático Baseado em Latência**

```python
# Exemplo de lógica de failover baseada em latência
class LatencyBasedFailover:
    def __init__(self):
        self.latency_thresholds = {
            'warning': 200,  # ms
            'critical': 500,  # ms
            'failover': 1000  # ms
        }
        self.region_priorities = [
            'us-east-1',
            'eu-west-1', 
            'ap-southeast-1'
        ]
    
    def check_and_failover(self, current_region, latency_ms):
        if latency_ms > self.latency_thresholds['failover']:
            return self.get_next_region(current_region)
        return current_region
    
    def get_next_region(self, current_region):
        try:
            current_index = self.region_priorities.index(current_region)
            next_index = (current_index + 1) % len(self.region_priorities)
            return self.region_priorities[next_index]
        except ValueError:
            return self.region_priorities[0]
```

##### 2. **Circuit Breaker Pattern para MCP Servers**

```yaml
# Circuit Breaker Configuration
circuit_breaker:
  failure_threshold: 5
  timeout: 30000  # 30 seconds
  reset_timeout: 60000  # 60 seconds
  
  per_region_config:
    us_east_1:
      max_concurrent_requests: 100
      latency_threshold: 200
      
    eu_west_1:
      max_concurrent_requests: 80
      latency_threshold: 250
      
    ap_southeast_1:
      max_concurrent_requests: 60
      latency_threshold: 300
```

#### Otimizações Específicas por Região

##### Configuração Regional Otimizada

```yaml
# Regional Configuration Template
regional_config:
  us_east_1:
    instance_types: ["m6i.large", "c6i.xlarge"]
    availability_zones: 3
    auto_scaling:
      min_capacity: 2
      max_capacity: 20
      target_cpu: 70
    
  eu_west_1:
    instance_types: ["m6i.large", "c6i.xlarge"] 
    availability_zones: 3
    auto_scaling:
      min_capacity: 1
      max_capacity: 15
      target_cpu: 70
      
  ap_southeast_1:
    instance_types: ["m6i.medium", "c6i.large"]
    availability_zones: 2
    auto_scaling:
      min_capacity: 1
      max_capacity: 10
      target_cpu: 75

# Network Optimization
network_optimization:
  enhanced_networking: true
  placement_groups: true
  dedicated_tenancy: false
  
  inter_region_connectivity:
    transit_gateway: true
    direct_connect: true
    vpn_backup: true
```

#### Considerações de Compliance Multi-Região

##### Data Residency e Sovereignty

```mermaid
graph TB
    subgraph "Data Residency Compliance"
        subgraph "GDPR - EU Data"
            EU_DATA[EU Customer Data]
            EU_PROCESSING[EU Processing Only]
            EU_STORAGE[EU Storage Only]
        end
        
        subgraph "US Data Regulations"
            US_DATA[US Customer Data]
            US_PROCESSING[US Processing Allowed]
            US_STORAGE[US Storage Allowed]
        end
        
        subgraph "Cross-Border Controls"
            ENCRYPTION[Data Encryption]
            TOKENIZATION[Data Tokenization]
            PSEUDONYMIZATION[Data Pseudonymization]
        end
    end
    
    EU_DATA --> EU_PROCESSING
    EU_DATA --> EU_STORAGE
    EU_DATA --> ENCRYPTION
    
    US_DATA --> US_PROCESSING
    US_DATA --> US_STORAGE
    US_DATA --> TOKENIZATION
    
    ENCRYPTION --> PSEUDONYMIZATION
    TOKENIZATION --> PSEUDONYMIZATION
    
    style EU_DATA fill:#4caf50
    style US_DATA fill:#2196f3
    style ENCRYPTION fill:#ff9800
```

#### Custos Multi-Região

##### Modelo de Custos por Região

```mermaid
pie title Distribuição de Custos Multi-Região
    "US-East-1 (Primary)" : 45
    "EU-West-1 (Secondary)" : 30
    "Cross-Region Data Transfer" : 15
    "Global Services (Route53, CloudFront)" : 10
```

##### Otimização de Custos Cross-Region

- **Data Transfer Optimization**: Compressão e deduplicação
- **Regional Pricing**: Aproveitamento de preços regionais
- **Reserved Instances**: Reserva multi-região para workloads previsíveis
- **Spot Instances**: Uso inteligente para workloads tolerantes a interrupção

#### Melhores Práticas Multi-Região

1. **Design for Failure**: Assumir que falhas regionais vão ocorrer
2. **Eventual Consistency**: Aceitar consistência eventual para melhor performance
3. **Regional Autonomy**: Cada região deve operar independentemente
4. **Data Locality**: Manter dados próximos aos usuários
5. **Monitoring Proativo**: Monitoramento contínuo de latência e disponibilidade
6. **Testing Regular**: Testes regulares de failover e disaster recovery

## Custos e Otimização

### Modelo de Custos por Ambiente e Região

```mermaid
pie title Distribuição de Custos - Ambiente POC (Single Region)
    "AgentCore Runtime" : 45
    "Amazon Cognito" : 15
    "Data Transfer" : 20
    "CloudWatch" : 10
    "Outros Serviços" : 10
```

```mermaid
pie title Distribuição de Custos - Ambiente Produção (Multi-Region)
    "EKS Control Plane (Multi-Region)" : 8
    "EC2 Instances (Primary + Secondary)" : 45
    "Fargate (Multi-Region)" : 20
    "Cross-Region Data Transfer" : 12
    "Load Balancers & Global Services" : 8
    "Storage (Multi-Region)" : 7
```

### Estratégia de Otimização de Custos Multi-Região

```mermaid
graph TD
    subgraph "Cost Optimization Strategy"
        subgraph "Regional Optimization"
            SPOT[Spot Instances<br/>70% savings]
            RESERVED[Reserved Instances<br/>40% savings]
            REGIONAL_PRICING[Regional Pricing<br/>15% variance]
        end
        
        subgraph "Data Transfer Optimization"
            COMPRESSION[Data Compression<br/>60% reduction]
            CACHING[Regional Caching<br/>80% reduction]
            CDN[CloudFront CDN<br/>50% reduction]
        end
        
        subgraph "Monitoring & Control"
            BUDGET[AWS Budgets<br/>Multi-Region]
            COST[Cost Explorer<br/>Regional Analysis]
            TAGS[Resource Tagging<br/>Region + Environment]
        end
    end
    
    subgraph "Workload Distribution"
        PRIMARY[Primary Region<br/>100% Traffic]
        SECONDARY[Secondary Region<br/>Standby + DR]
        TERTIARY[Tertiary Region<br/>Development]
    end
    
    SPOT --> SECONDARY
    RESERVED --> PRIMARY
    REGIONAL_PRICING --> TERTIARY
    
    COMPRESSION --> PRIMARY
    CACHING --> SECONDARY
    CDN --> PRIMARY
    
    BUDGET --> COST
    COST --> TAGS
    TAGS --> PRIMARY
    TAGS --> SECONDARY
    TAGS --> TERTIARY
    
    style COMPRESSION fill:#4caf50
    style RESERVED fill:#2196f3
    style CDN fill:#ff9800
    style CACHING fill:#9c27b0
```

### 9.1 Modelo de Custos Multi-Região

**Região Primária (US-East-1):**
- EKS Control Plane: $0.10/hora
- EC2 Instances: Variável baseado no tipo (m6i.large: $0.0864/hora)
- Fargate: $0.04048/vCPU/hora + $0.004445/GB/hora
- Data Transfer OUT: $0.09/GB (primeiros 10TB)

**Região Secundária (EU-West-1):**
- EKS Control Plane: $0.10/hora
- EC2 Instances: Preço regional (m6i.large: $0.0922/hora - 6.7% mais caro)
- Cross-Region Data Transfer: $0.02/GB (entre regiões AWS)
- Storage Replication: Custos adicionais de sincronização

**Serviços Globais:**
- Route 53: $0.50/hosted zone + $0.40/milhão de queries
- CloudFront: $0.085/GB (primeiros 10TB) + $0.0075/10.000 requests
- AWS WAF: $1.00/web ACL + $0.60/milhão de requests

### 9.2 Estratégias de Otimização Multi-Região

#### Otimização de Data Transfer
- **Compressão**: Redução de 60-80% no volume de dados transferidos
- **Deduplicação**: Eliminação de dados duplicados na sincronização
- **Delta Sync**: Sincronização apenas de mudanças incrementais
- **Regional Caching**: Cache local para reduzir transferências

#### Otimização de Instâncias
- **Spot Instances**: Uso em regiões secundárias para standby (70% economia)
- **Reserved Instances**: Reserva multi-região para workloads previsíveis
- **Right Sizing**: Dimensionamento otimizado por região baseado na demanda
- **Scheduled Scaling**: Escalonamento baseado em padrões de uso regional

#### Otimização de Storage
- **Intelligent Tiering**: S3 Intelligent Tiering para otimização automática
- **Cross-Region Replication**: Replicação seletiva baseada em criticidade
- **Lifecycle Policies**: Políticas de ciclo de vida regionais
- **Compression at Rest**: Compressão de dados em repouso

## Referências Técnicas

### RFCs e Especificações Oficiais

#### OAuth 2.0 e Extensões
- **RFC 6749**: The OAuth 2.0 Authorization Framework - Base para autenticação M2M
- **RFC 7591**: OAuth 2.0 Dynamic Client Registration Protocol - DCR para MCP clients
- **RFC 7636**: Proof Key for Code Exchange by OAuth Public Clients - PKCE obrigatório
- **RFC 8707**: Resource Indicators for OAuth 2.0 - Token audience binding
- **RFC 9068**: JSON Web Token (JWT) Profile for OAuth 2.0 Access Tokens - Formato padronizado de tokens
- **RFC 9728**: OAuth 2.0 Protected Resource Metadata - Descoberta de servidores de autorização

#### Model Context Protocol (MCP)
- **MCP Specification v2025-11-25**: [Model Context Protocol Official](https://modelcontextprotocol.io/specification/2025-11-25)
- **MCP Authorization Specification**: [MCP Authorization](https://modelcontextprotocol.io/specification/draft/basic/authorization)
- **Client ID Metadata Documents**: OAuth Client ID Metadata Document (draft-ietf-oauth-client-id-metadata-document-00)

#### Segurança e Melhores Práticas
- **OAuth 2.1**: OAuth 2.1 Security Best Practices - Versão consolidada com melhorias de segurança
- **OWASP API Security Top 10**: Diretrizes de segurança para APIs
- **NIST Cybersecurity Framework**: Framework de segurança cibernética

### Documentação AWS Oficial

#### Amazon EKS
- [Amazon EKS Best Practices Guide](https://aws.github.io/aws-eks-best-practices/) - Práticas recomendadas para EKS
- [EKS Auto Mode Documentation](https://docs.aws.amazon.com/eks/latest/userguide/auto-mode.html) - Gerenciamento automatizado
- [EKS Security Best Practices](https://aws.github.io/aws-eks-best-practices/security/docs/) - Segurança em EKS

#### Amazon Cognito
- [Amazon Cognito Developer Guide](https://docs.aws.amazon.com/cognito/) - Guia completo do Cognito
- [Cognito User Pool OAuth 2.0 Grants](https://docs.aws.amazon.com/cognito/latest/developerguide/amazon-cognito-user-pools-using-tokens-with-identity-providers.html) - Implementação OAuth
- [Machine-to-Machine Authentication](https://docs.aws.amazon.com/cognito/latest/developerguide/amazon-cognito-user-pools-using-tokens-verifying-a-jwt.html) - M2M com Cognito

#### Amazon Bedrock e AgentCore
- [Amazon Bedrock AgentCore Documentation](https://aws.amazon.com/bedrock/agentcore/) - Documentação oficial do AgentCore
- [Bedrock Agents Developer Guide](https://docs.aws.amazon.com/bedrock/latest/userguide/agents.html) - Desenvolvimento de agentes
- [AgentCore Runtime Guide](https://docs.aws.amazon.com/bedrock/latest/userguide/agentcore-runtime.html) - Runtime para MCPs

#### Segurança AWS
- [AWS Security Reference Architecture](https://docs.aws.amazon.com/prescriptive-guidance/latest/security-reference-architecture/) - Arquitetura de referência
- [AWS Secrets Manager Best Practices](https://docs.aws.amazon.com/secretsmanager/latest/userguide/best-practices.html) - Gerenciamento de credenciais
- [AWS IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html) - Práticas de IAM

### Recursos Técnicos Adicionais

#### Keycloak
- [Keycloak Documentation](https://www.keycloak.org/documentation) - Documentação oficial
- [Keycloak OAuth 2.0 Dynamic Client Registration](https://www.keycloak.org/docs/latest/securing_apps/#_client_registration) - Implementação DCR
- [Keycloak Security Best Practices](https://www.keycloak.org/docs/latest/server_admin/#_security_best_practices) - Práticas de segurança

#### Observabilidade e Monitoramento
- [OpenTelemetry Specification](https://opentelemetry.io/docs/specs/) - Padrão de observabilidade
- [AWS CloudWatch Best Practices](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_architecture.html) - Monitoramento
- [Prometheus Monitoring](https://prometheus.io/docs/introduction/overview/) - Métricas de aplicação

### Implementações de Referência

#### MCP Implementations
- [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk) - SDK oficial TypeScript
- [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk) - SDK oficial Python
- [MCP Servers Examples](https://github.com/modelcontextprotocol/servers) - Exemplos de servidores MCP

#### OAuth 2.0 Libraries
- [OAuth 2.0 Security Best Practices](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics) - Práticas de segurança
- [PKCE Implementation Guide](https://datatracker.ietf.org/doc/html/rfc7636) - Implementação PKCE
- [JWT Best Practices](https://datatracker.ietf.org/doc/html/rfc8725) - Práticas para JWT

### Conformidade e Certificações

#### Padrões de Conformidade
- **SOC 2 Type II**: Controles de segurança organizacional
- **ISO 27001**: Sistema de gestão de segurança da informação
- **PCI DSS**: Padrão de segurança para dados de cartão
- **GDPR**: Regulamento geral de proteção de dados

#### Certificações AWS
- **AWS Well-Architected Framework**: Princípios arquiteturais
- **AWS Security Pillar**: Pilar de segurança do Well-Architected
- **AWS Compliance Programs**: Programas de conformidade AWS

### Atualizações e Versionamento

Este documento segue as seguintes especificações em suas versões mais recentes:

- **MCP Specification**: v2025-11-25 (Novembro 2024)
- **OAuth 2.1**: Draft mais recente (2024)
- **RFC 9068**: Publicado em Outubro 2021
- **RFC 9728**: Publicado em Dezembro 2023
- **AWS Services**: Versões atuais (Janeiro 2025)

**Nota sobre Atualizações**: Este documento é mantido atualizado com as versões mais recentes das especificações. Recomenda-se verificar periodicamente as especificações oficiais para mudanças que possam impactar a implementação.

## Conclusão

Esta arquitetura revisada fornece uma base sólida e escalável para implementação de MCPs e AI Agents na AWS, com distinções claras entre diferentes padrões de autenticação baseados no contexto de uso:

### Principais Melhorias Implementadas

#### 1. **Estratégia de Autenticação Diferenciada**
- **DCR (Keycloak)** para MCP Clients diretos como IDEs, oferecendo flexibilidade de distribuição
- **Client Credentials (Cognito)** para comunicação Agent-Gateway, otimizada para M2M em escala
- **Service Accounts** gerenciadas via AWS Secrets Manager para identificação clara de agentes

#### 2. **Identificação e Contexto de Cliente Robusto**
- Implementação completa de **RFC 9068** para tokens JWT padronizados
- **Context Enrichment** no gateway com passagem estruturada de dados do cliente
- **Tenant Isolation** completo com auditoria granular por cliente
- **Headers HTTP padronizados** para passagem de contexto entre componentes

#### 3. **Conformidade com Especificações Oficiais**
- **RFC 7591** para Dynamic Client Registration
- **RFC 6749 Seção 4.4** para Client Credentials Grant
- **RFC 8707** para Resource Indicators e token audience binding
- **MCP Authorization Specification** para descoberta e validação de servidores

#### 4. **Segurança Aprimorada**
- **Token Audience Binding** para prevenir token passthrough
- **PKCE obrigatório** para MCP clients públicos
- **Validação rigorosa de contexto** em cada request
- **Encryption at rest e in transit** com chaves específicas por tenant

### Benefícios da Arquitetura Revisada

- **Flexibilidade de Distribuição**: DCR permite que ferramentas como Kiro se registrem dinamicamente
- **Escalabilidade M2M**: Client Credentials otimizado para comunicação de alta escala entre agentes
- **Identificação Clara**: Service accounts e context enrichment permitem rastreabilidade completa
- **Conformidade Regulatória**: Implementação baseada em RFCs oficiais e melhores práticas de segurança
- **Isolamento de Dados**: Segregação completa por tenant com controles granulares

### Próximos Passos Recomendados

1. **Implementação Faseada**: Seguir o roadmap proposto com foco inicial na fundação
2. **Testes de Conformidade**: Validar implementação contra especificações RFC
3. **Monitoramento Proativo**: Implementar observabilidade desde o primeiro deploy
4. **Documentação Técnica**: Manter documentação atualizada com as especificações

A implementação faseada permite evolução gradual da solução, minimizando riscos e permitindo aprendizado contínuo. O foco em conformidade com RFCs oficiais, segurança robusta e identificação clara de contexto garante que a solução seja adequada para ambientes empresariais críticos com requisitos rigorosos de auditoria e compliance.

---

**Versão**: 2.0  
**Data**: Janeiro 2025  
**Autor**: Arquitetura de Soluções AWS  
**Status**: Documento Vivo - Atualizado com RFC 9068, MCP Authorization Spec e melhores práticas 2025  
**Principais Atualizações**: Estratégia de autenticação diferenciada, identificação de contexto de cliente, conformidade com especificações oficiais