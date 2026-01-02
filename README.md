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
            B2B --> B2B_TYPE{Tipo de Cliente?}
            B2B_TYPE -->|Sistema Parceiro| PARTNER[Sistema Parceiro/Cliente]
            B2B_TYPE -->|Agente de Terceiros| THIRD_PARTY[ChatGPT Apps, Claude, etc]
            
            PARTNER --> BFF_B2B[Backend for Frontend<br/>B2B]
            BFF_B2B --> AGENT_B2B{Agent Complexity?}
            AGENT_B2B -->|Complex Logic| EKS_B2B[Agent no EKS]
            AGENT_B2B -->|Simple Logic| RUNTIME_B2B[Agent no AgentCore Runtime]
            EKS_B2B --> GATEWAY_B2B[AgentCore Gateway]
            RUNTIME_B2B --> GATEWAY_B2B
            
            THIRD_PARTY --> AUTH_THIRD[Autenticação<br/>OAuth/API Key]
            AUTH_THIRD --> GATEWAY_THIRD[Acesso Direto<br/>AgentCore Gateway]
            
            GATEWAY_B2B --> MCP_B2B[MCPs no Runtime]
            GATEWAY_THIRD --> MCP_B2B
        end
    end
    
    style INTERNO fill:#4caf50
    style B2C fill:#2196f3
    style B2B fill:#ff9800
    style THIRD_PARTY fill:#9c27b0
    style KEYCLOAK_DCR fill:#9c27b0
    style BFF_B2C fill:#e91e63
    style BFF_B2B fill:#e91e63
    style AUTH_THIRD fill:#795548
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
    participant THIRD_PARTY as Agente de Terceiros<br/>(ChatGPT, Claude, etc)
    participant API as API B2B
    participant BFF as BFF B2B
    participant AGENT as AI Agent Interno
    participant AGW as AgentCore Gateway
    participant MCP as MCP Server
    
    Note over PARTNER, MCP: Cenário B2B - Múltiplos Pontos de Entrada
    
    rect rgb(200, 230, 255)
        Note over PARTNER, AGENT: Fluxo Sistema Parceiro
        PARTNER->>API: API Call
        API->>BFF: Request autenticado
        BFF->>AGENT: Business logic request
        AGENT->>AGW: Request para AgentCore Gateway
        AGW->>MCP: Processa request
        MCP->>AGW: Response
        AGW->>AGENT: Response
        AGENT->>BFF: Processed response
        BFF->>API: Business response
        API->>PARTNER: Final response
    end
    
    rect rgb(255, 230, 200)
        Note over THIRD_PARTY, MCP: Fluxo Agente de Terceiros
        THIRD_PARTY->>AGW: Direct MCP Request (OAuth/API Key)
        AGW->>AGW: Valida credenciais de terceiros
        AGW->>MCP: Processa request
        MCP->>AGW: Response
        AGW->>THIRD_PARTY: Response direta
    end
```

**Características do Cenário B2B Expandido:**
- **Usuários**: Sistemas empresariais, parceiros, **agentes de terceiros**
- **Arquitetura Principal**: Sistema → BFF → Agent → Gateway → MCP
- **Arquitetura Terceiros**: Agente Terceiro → Gateway → MCP (acesso direto ao gateway)
- **Agent Location**: EKS (lógica complexa) ou AgentCore Runtime (lógica simples)
- **Acesso aos MCPs**: Sempre via AgentCore Gateway
- **Casos de Uso**: Integrações ERP, APIs empresariais, **ChatGPT Apps, Claude Apps, agentes externos**

### Matriz de Decisão Detalhada

| Critério | Interno | B2C | B2B Parceiros | B2B Terceiros |
|----------|---------|-----|---------------|---------------|
| **Autenticação** | Keycloak DCR | Cognito M2M | Cognito M2M | OAuth2/API Key |
| **Ponto de Entrada** | Direto no Gateway | BFF | BFF | Direto no Gateway |
| **Agent Hosting** | N/A | EKS/Runtime | EKS/Runtime | Externo |
| **MCP Access** | Via Gateway | Via Gateway | Via Gateway | Via Gateway |
| **Escalabilidade** | Baixa-Média | Alta | Média-Alta | Alta |
| **Complexidade** | Baixa | Média | Alta | Média |
| **Latência Alvo** | < 500ms | < 200ms | < 300ms | < 400ms |
| **Rate Limiting** | 1000/min | 10000/hour | 10000/day | 1000/hour |
| **Permissões** | Full Access | App Specific | Tenant Specific | Limited Access |
| **Casos de Uso** | Dev Tools, Debug | Apps Consumer | Enterprise APIs | ChatGPT Apps, Claude |

### Configurações por Cenário Expandido

#### Configuração B2B Terceiros
```yaml
b2b_third_party_scenario:
  authentication:
    primary: "oauth2_client_credentials"
    fallback: "api_key_jwt"
    registration: "manual_approval"
  
  access_control:
    permissions: "limited"
    operations: ["read", "limited_write"]
    admin_access: false
  
  rate_limiting:
    default: "1000/hour"
    burst: "50/minute"
    premium_tier: "5000/hour"
  
  monitoring:
    audit_level: "detailed"
    metrics_collection: true
    alert_on_anomalies: true
  
  supported_clients:
    - "openai_chatgpt"
    - "anthropic_claude"
    - "custom_agents"
    - "automation_tools"
```

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
        subgraph "Sistemas Parceiros"
            ENTERPRISE[Sistemas Empresariais<br/>ERP, CRM, APIs]
            BFF_B2B[BFF B2B<br/>Enterprise API Layer]
            AGENTS_B2B[AI Agents<br/>EKS ou AgentCore Runtime]
        end
        
        subgraph "Agentes de Terceiros"
            CHATGPT[ChatGPT Apps]
            CLAUDE[Claude Apps]
            OTHER_AGENTS[Outros Agentes<br/>Externos]
        end
        
        AGW_B2B[AgentCore Gateway<br/>Multi-Client Support]
        MCP_B2B[MCP Servers<br/>AgentCore Runtime]
    end
    
    subgraph "Camada de Autenticação"
        KC[Keycloak<br/>DCR para Interno]
        COG[Amazon Cognito<br/>M2M para B2C/B2B]
        THIRD_PARTY_AUTH[Third-Party Auth<br/>OAuth/API Keys]
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
    
    %% Fluxos Cenário B2B - Sistemas Parceiros
    ENTERPRISE --> BFF_B2B
    BFF_B2B --> AGENTS_B2B
    AGENTS_B2B --> AGW_B2B
    
    %% Fluxos Cenário B2B - Agentes de Terceiros
    CHATGPT --> THIRD_PARTY_AUTH
    CLAUDE --> THIRD_PARTY_AUTH
    OTHER_AGENTS --> THIRD_PARTY_AUTH
    THIRD_PARTY_AUTH --> AGW_B2B
    
    AGW_B2B --> MCP_B2B
    
    %% Conexões com Infraestrutura
    KC_INTERNAL -.-> KC
    BFF_B2C -.-> COG
    BFF_B2B -.-> COG
    THIRD_PARTY_AUTH -.-> COG
    
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
    style CHATGPT fill:#9c27b0
    style CLAUDE fill:#9c27b0
    style OTHER_AGENTS fill:#9c27b0
    style BFF_B2C fill:#e91e63
    style BFF_B2B fill:#e91e63
    style KC_INTERNAL fill:#9c27b0
    style THIRD_PARTY_AUTH fill:#795548
```

### Comparativo dos Cenários Expandido

| Aspecto | Interno | B2C | B2B Parceiros | B2B Terceiros |
|---------|---------|-----|---------------|---------------|
| **Complexidade** | Baixa | Média | Alta | Média |
| **Usuários** | Desenvolvedores | Consumidores | Empresas | Agentes IA |
| **Autenticação** | Keycloak DCR | Cognito M2M | Cognito M2M | OAuth/API Key |
| **BFF** | Não necessário | Obrigatório | Obrigatório | Não necessário |
| **Agent Location** | N/A | EKS/Runtime | EKS/Runtime | Externo |
| **Gateway** | Sempre | Sempre | Sempre | Sempre |
| **Escalabilidade** | Baixa | Alta | Variável | Alta |
| **Latência** | Tolerante | Crítica | Moderada | Moderada |

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

### Arquitetura de Identificação de Cliente com Token Exchange (RFC 8693)

```mermaid
sequenceDiagram
    participant User as Usuário Final
    participant Agent as AI Agent
    participant Gateway as AgentCore Gateway
    participant STS as Security Token Service<br/>(RFC 8693)
    participant Cognito as Amazon Cognito
    participant MCP as MCP Server
    participant SM as Secrets Manager
    
    Note over User, MCP: 1. Token Exchange para Contexto de Cliente (RFC 8693)
    User->>Agent: Solicitação com contexto do cliente
    Agent->>SM: Recupera credenciais do service account
    SM->>Agent: Client credentials (client_id, client_secret)
    
    Note over User, MCP: 2. Token Exchange - Delegation Pattern
    Agent->>STS: Token Exchange Request<br/>subject_token (user context)<br/>actor_token (agent credentials)
    STS->>STS: Valida tokens e aplica políticas
    STS->>Agent: Composite Token JWT (user + agent context)
    
    Note over User, MCP: 3. Passagem de Contexto Enriquecido via Gateway
    Agent->>Gateway: Request + Composite JWT Token
    Gateway->>Gateway: Extrai claims do composite token
    Gateway->>MCP: Request + Contexto Completo (user + agent)
    MCP->>Gateway: Response filtrada por contexto
    Gateway->>Agent: Response final
```

### 5.1 Implementação de Token Exchange (RFC 8693)

#### Token Exchange para Delegation Semantics

A RFC 8693 define um protocolo para Security Token Service (STS) que permite trocar tokens existentes por novos tokens com diferentes contextos, incluindo **delegation** e **impersonation**. Na nossa arquitetura, isso é especialmente útil para:

**Casos de Uso com Token Exchange:**
- **Agent Delegation**: Agent atua em nome do usuário mantendo identidade própria
- **Context Enrichment**: Combinar contexto do usuário com credenciais do agent
- **Cross-Domain Access**: Trocar tokens entre diferentes domínios de segurança
- **Scope Transformation**: Converter escopos de usuário para escopos de sistema

#### Implementação do Token Exchange Request

```http
POST /oauth2/token HTTP/1.1
Host: sts.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&subject_token=eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
&subject_token_type=urn:ietf:params:oauth:token-type:access_token
&actor_token=eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
&actor_token_type=urn:ietf:params:oauth:token-type:access_token
&resource=https://gateway.agentcore.aws.com
&audience=mcp-servers
&scope=mcp:read mcp:write user:context
```

#### Composite Token JWT com Claims RFC 8693

```json
{
  "iss": "https://sts.agentcore.aws.com",
  "sub": "user-12345",
  "aud": "https://gateway.agentcore.aws.com",
  "exp": 1704070800,
  "iat": 1704067200,
  "jti": "composite-token-uuid",
  
  // Claims do usuário (subject_token)
  "user_context": {
    "user_id": "user-john-doe",
    "email": "john.doe@acme-corp.com",
    "department": "finance",
    "role": "analyst",
    "tenant_id": "acme-corp-12345",
    "permissions": ["read:financial-data", "write:reports"]
  },
  
  // Claims do agent (actor_token) - RFC 8693 "act" claim
  "act": {
    "sub": "ai-agent-finance-001",
    "iss": "https://cognito-idp.us-east-1.amazonaws.com/us-east-1_POOL",
    "client_id": "finance-agent-service-account",
    "agent_type": "financial-assistant",
    "version": "2.1.0",
    "deployment_env": "production"
  },
  
  // Delegation chain (se aplicável)
  "delegation_chain": [
    {
      "delegator": "user-john-doe",
      "delegate": "ai-agent-finance-001",
      "scope": ["mcp:read", "mcp:write:limited"],
      "timestamp": "2025-01-02T10:25:00Z"
    }
  ],
  
  // Contexto de negócio
  "business_context": {
    "session_id": "session-uuid-12345",
    "request_origin": "web-app",
    "compliance_level": "gdpr",
    "data_classification": "confidential"
  }
}
```

### 5.2 Security Token Service (STS) Configuration

#### STS para Token Exchange
```yaml
# Security Token Service Configuration (RFC 8693)
sts_config:
  token_exchange:
    enabled: true
    grant_type: "urn:ietf:params:oauth:grant-type:token-exchange"
    
  supported_token_types:
    input:
      - "urn:ietf:params:oauth:token-type:access_token"
      - "urn:ietf:params:oauth:token-type:id_token"
      - "urn:ietf:params:oauth:token-type:jwt"
    
    output:
      - "urn:ietf:params:oauth:token-type:access_token"
      - "urn:ietf:params:oauth:token-type:jwt"
  
  delegation_policies:
    - name: "agent_delegation"
      subject_pattern: "user-*"
      actor_pattern: "ai-agent-*"
      allowed_scopes: ["mcp:read", "mcp:write:limited"]
      max_delegation_depth: 2
      
    - name: "third_party_delegation"
      subject_pattern: "user-*"
      actor_pattern: "third-party-*"
      allowed_scopes: ["mcp:read"]
      max_delegation_depth: 1
  
  token_validation:
    signature_verification: true
    audience_validation: true
    expiration_check: true
    revocation_check: true
```

#### Integration with Amazon Cognito
```yaml
# Cognito Integration for Token Exchange
cognito_sts_integration:
  user_pool_id: "us-east-1_XXXXXXXXX"
  
  token_exchange_client:
    client_id: "sts-token-exchange-client"
    client_secret: "${AWS_SECRET_MANAGER_REF}"
    grant_types: ["urn:ietf:params:oauth:grant-type:token-exchange"]
    
  custom_attributes:
    - "tenant_id"
    - "department"
    - "role"
    - "permissions"
  
  lambda_triggers:
    pre_token_generation: "arn:aws:lambda:us-east-1:123456789012:function:EnrichTokenClaims"
    post_authentication: "arn:aws:lambda:us-east-1:123456789012:function:LogUserContext"
```

### 5.3 Passagem de Contexto Enriquecido através do Gateway

#### Headers HTTP com Contexto RFC 8693

```http
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
X-Subject-Token-Type: urn:ietf:params:oauth:token-type:access_token
X-Actor-Context: eyJzdWIiOiJhaS1hZ2VudC1maW5hbmNlLTAwMSJ9...
X-Delegation-Chain: eyJkZWxlZ2F0b3IiOiJ1c2VyLWpvaG4tZG9lIn0...
X-Tenant-ID: acme-corp-12345
X-User-Context: eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
X-Business-Context: eyJzZXNzaW9uX2lkIjoic2Vzc2lvbi11dWlkIn0...
X-Request-ID: req-uuid-12345
X-Correlation-ID: corr-uuid-67890
```

#### Gateway Context Processing com RFC 8693

```mermaid
graph TB
    subgraph "AgentCore Gateway - RFC 8693 Context Processing"
        TOKEN_EXCHANGE[Token Exchange Validation]
        COMPOSITE_PARSE[Composite Token Parsing]
        DELEGATION_CHECK[Delegation Chain Validation]
        CONTEXT_EXTRACT[Context Extraction]
        POLICY_APPLY[Policy Application]
        HEADER_TRANSFORM[Header Transformation]
    end
    
    subgraph "Input Tokens (RFC 8693)"
        COMPOSITE_TOKEN[Composite JWT Token]
        SUBJECT_CLAIMS[Subject Claims<br/>(User Context)]
        ACTOR_CLAIMS[Actor Claims<br/>(Agent Context)]
    end
    
    subgraph "Output Context"
        MCP_HEADERS[Enhanced MCP Headers]
        AUDIT_CONTEXT[Audit Context]
        FILTERED_CLAIMS[Filtered Claims by Policy]
    end
    
    COMPOSITE_TOKEN --> TOKEN_EXCHANGE
    TOKEN_EXCHANGE --> COMPOSITE_PARSE
    COMPOSITE_PARSE --> SUBJECT_CLAIMS
    COMPOSITE_PARSE --> ACTOR_CLAIMS
    
    SUBJECT_CLAIMS --> DELEGATION_CHECK
    ACTOR_CLAIMS --> DELEGATION_CHECK
    DELEGATION_CHECK --> CONTEXT_EXTRACT
    CONTEXT_EXTRACT --> POLICY_APPLY
    POLICY_APPLY --> HEADER_TRANSFORM
    
    HEADER_TRANSFORM --> MCP_HEADERS
    HEADER_TRANSFORM --> AUDIT_CONTEXT
    HEADER_TRANSFORM --> FILTERED_CLAIMS
    
    style TOKEN_EXCHANGE fill:#ff9800
    style COMPOSITE_PARSE fill:#4caf50
    style DELEGATION_CHECK fill:#2196f3
    style POLICY_APPLY fill:#9c27b0
```

### 5.4 Implementação de Delegation vs Impersonation

#### Delegation Semantics (Recomendado)
```python
# Exemplo de processamento de delegation com RFC 8693
class TokenExchangeProcessor:
    def process_delegation_token(self, composite_token):
        # Parse composite token
        claims = self.parse_jwt(composite_token)
        
        # Extract subject (user) and actor (agent) contexts
        subject_context = claims.get('user_context', {})
        actor_context = claims.get('act', {})
        
        # Validate delegation chain
        delegation_chain = claims.get('delegation_chain', [])
        if not self.validate_delegation_chain(delegation_chain):
            raise UnauthorizedError("Invalid delegation chain")
        
        # Apply delegation policies
        effective_permissions = self.apply_delegation_policies(
            subject_permissions=subject_context.get('permissions', []),
            actor_permissions=actor_context.get('scopes', []),
            delegation_policies=self.get_delegation_policies()
        )
        
        return {
            'subject': subject_context,
            'actor': actor_context,
            'effective_permissions': effective_permissions,
            'delegation_chain': delegation_chain
        }
    
    def validate_delegation_chain(self, chain):
        # Validate each step in delegation chain
        for step in chain:
            if not self.is_valid_delegation_step(step):
                return False
        return True
```

#### MCP Server Context Processing
```python
# Exemplo de como um MCP Server processa contexto RFC 8693
class MCPServerRFC8693Handler:
    def process_request_with_delegation(self, request):
        # Extract delegation context from headers
        composite_token = request.headers.get('Authorization').replace('Bearer ', '')
        actor_context = self.decode_header(request.headers.get('X-Actor-Context'))
        delegation_chain = self.decode_header(request.headers.get('X-Delegation-Chain'))
        
        # Parse composite token
        token_claims = self.parse_composite_token(composite_token)
        
        # Validate delegation authority
        if not self.validate_delegation_authority(token_claims, actor_context):
            raise UnauthorizedError("Actor not authorized for delegation")
        
        # Apply business logic with delegation context
        filtered_data = self.filter_data_by_delegation_context(
            data=self.get_data(),
            subject_context=token_claims.get('user_context'),
            actor_context=token_claims.get('act'),
            delegation_chain=delegation_chain
        )
        
        # Log delegation activity
        self.log_delegation_activity(token_claims, actor_context, delegation_chain)
        
        return filtered_data
```

### 5.5 Auditoria e Compliance com RFC 8693

#### Enhanced Audit Log Structure
```json
{
  "timestamp": "2025-01-02T10:30:00Z",
  "event_type": "mcp_request_with_delegation",
  "correlation_id": "corr-uuid-67890",
  "request_id": "req-uuid-12345",
  
  "token_exchange_context": {
    "grant_type": "urn:ietf:params:oauth:grant-type:token-exchange",
    "subject_token_type": "urn:ietf:params:oauth:token-type:access_token",
    "actor_token_type": "urn:ietf:params:oauth:token-type:access_token",
    "issued_token_type": "urn:ietf:params:oauth:token-type:access_token"
  },
  
  "delegation_context": {
    "delegation_type": "agent_delegation",
    "subject": {
      "user_id": "user-john-doe",
      "tenant_id": "acme-corp-12345",
      "department": "finance"
    },
    "actor": {
      "agent_id": "ai-agent-finance-001",
      "agent_type": "financial-assistant",
      "version": "2.1.0"
    },
    "delegation_chain": [
      {
        "delegator": "user-john-doe",
        "delegate": "ai-agent-finance-001",
        "scope": ["mcp:read", "mcp:write:limited"],
        "timestamp": "2025-01-02T10:25:00Z"
      }
    ]
  },
  
  "mcp_context": {
    "server_name": "financial-data-mcp",
    "method": "tools/call",
    "resource_accessed": "/api/v1/financial-reports",
    "effective_permissions": ["read:financial-data"],
    "data_classification": "confidential"
  },
  
  "compliance_context": {
    "gdpr_applicable": true,
    "data_subject": "user-john-doe",
    "processing_purpose": "financial_analysis",
    "legal_basis": "legitimate_interest",
    "retention_period": "7_years"
  }
}
```

### 5.6 Benefícios da Implementação RFC 8693

#### Vantagens Técnicas
- **Padronização**: Uso de especificação oficial IETF
- **Flexibilidade**: Suporte a delegation e impersonation
- **Segurança**: Validação rigorosa de delegation chains
- **Auditoria**: Rastreamento completo de contexto de delegação
- **Interoperabilidade**: Compatibilidade com outros sistemas OAuth 2.0

#### Casos de Uso Cobertos
- **Agent Delegation**: AI Agent atua em nome do usuário
- **Third-Party Delegation**: Agentes externos com contexto limitado
- **Cross-Domain Access**: Acesso entre diferentes domínios de segurança
- **Compliance**: Atendimento a requisitos de auditoria e governança

#### Conformidade com Especificações
- **RFC 8693**: OAuth 2.0 Token Exchange
- **RFC 9068**: JWT Profile for OAuth 2.0 Access Tokens
- **RFC 8707**: Resource Indicators for OAuth 2.0
- **MCP Authorization Specification**: Especificação oficial do MCP

## Matriz de Decisão: Cognito vs Keycloak

### Arquitetura Inbound/Outbound do AgentCore Gateway

```mermaid
graph TB
    subgraph "Inbound Traffic - Para o AgentCore Gateway"
        subgraph "Keycloak DCR Zone"
            DEV_TOOLS[Ferramentas de Desenvolvimento<br/>IDEs, CLIs, Debug Tools]
            MCP_CLIENTS[MCP Clients Diretos<br/>Kiro, Claude Desktop, etc]
            THIRD_PARTY_DIRECT[Agentes Terceiros Diretos<br/>ChatGPT Apps, Custom Agents]
        end
        
        subgraph "Cognito M2M Zone"
            INTERNAL_AGENTS[AI Agents Internos<br/>via BFF ou Direto]
            BFF_REQUESTS[Requests via BFF<br/>B2C e B2B Partners]
            SERVICE_ACCOUNTS[Service Accounts<br/>Automação Interna]
        end
    end
    
    subgraph "AgentCore Gateway"
        INBOUND_AUTH[Inbound Authentication<br/>Keycloak + Cognito]
        GATEWAY_CORE[Gateway Core<br/>Routing & Processing]
        OUTBOUND_AUTH[Outbound Authentication<br/>Cognito M2M]
    end
    
    subgraph "Outbound Traffic - Do AgentCore Gateway"
        MCP_SERVERS[MCP Servers<br/>AgentCore Runtime]
        EXTERNAL_APIS[APIs Externas<br/>Databases, Services]
        AWS_SERVICES[AWS Services<br/>S3, DynamoDB, etc]
    end
    
    %% Inbound Flows
    DEV_TOOLS -->|Keycloak DCR| INBOUND_AUTH
    MCP_CLIENTS -->|Keycloak DCR| INBOUND_AUTH
    THIRD_PARTY_DIRECT -->|Keycloak DCR| INBOUND_AUTH
    
    INTERNAL_AGENTS -->|Cognito M2M| INBOUND_AUTH
    BFF_REQUESTS -->|Cognito M2M| INBOUND_AUTH
    SERVICE_ACCOUNTS -->|Cognito M2M| INBOUND_AUTH
    
    %% Gateway Processing
    INBOUND_AUTH --> GATEWAY_CORE
    GATEWAY_CORE --> OUTBOUND_AUTH
    
    %% Outbound Flows
    OUTBOUND_AUTH -->|Cognito M2M| MCP_SERVERS
    OUTBOUND_AUTH -->|Cognito M2M| EXTERNAL_APIS
    OUTBOUND_AUTH -->|AWS IAM/Cognito| AWS_SERVICES
    
    style INBOUND_AUTH fill:#ff9800
    style GATEWAY_CORE fill:#4caf50
    style OUTBOUND_AUTH fill:#2196f3
    style DEV_TOOLS fill:#9c27b0
    style MCP_CLIENTS fill:#9c27b0
    style INTERNAL_AGENTS fill:#e91e63
    style BFF_REQUESTS fill:#e91e63
```

### Matriz de Decisão Detalhada: Cognito vs Keycloak

| Critério | Keycloak | Cognito | Justificativa |
|----------|----------|---------|---------------|
| **Direção de Tráfego** | **Inbound** (para Gateway) | **Inbound + Outbound** | Cognito para M2M, Keycloak para flexibilidade |
| **Tipo de Cliente** | Externos, Desenvolvimento | Internos, Produção | Keycloak para clientes diversos, Cognito para controle |
| **Padrão de Autenticação** | DCR, Authorization Code | Client Credentials, M2M | DCR para flexibilidade, M2M para performance |
| **Casos de Uso** | IDEs, MCP Clients, Third-Party | Agents, BFFs, Services | Diferentes necessidades de integração |
| **Escalabilidade** | Média (self-hosted) | Alta (managed service) | Cognito gerenciado pela AWS |
| **Custo** | Infraestrutura própria | Pay-per-use | Modelo de custos diferente |
| **Flexibilidade** | Alta (customização) | Média (AWS managed) | Keycloak mais customizável |
| **Manutenção** | Alta (self-managed) | Baixa (AWS managed) | Cognito gerenciado pela AWS |

### Decisão por Cenário de Uso

#### Cenário 1: Inbound - Use Keycloak Quando:

```yaml
use_keycloak_when:
  client_type:
    - "development_tools"      # IDEs, CLIs, Debug tools
    - "mcp_clients_direct"     # Kiro, Claude Desktop
    - "third_party_agents"     # ChatGPT Apps, Custom agents
    - "external_integrations"  # Sistemas externos não controlados
  
  authentication_pattern:
    - "dynamic_client_registration"  # DCR para flexibilidade
    - "authorization_code_flow"      # Com usuário final
    - "pkce_required"               # Clientes públicos
  
  characteristics:
    - "unknown_clients"        # Clientes não pré-registrados
    - "diverse_client_types"   # Diferentes tipos de aplicação
    - "user_consent_required"  # Necessita consentimento do usuário
    - "flexible_registration"  # Registro dinâmico necessário
  
  traffic_direction: "inbound"  # Sempre para o Gateway
```

**Exemplo de Configuração Keycloak:**
```yaml
keycloak_inbound_config:
  realm: "agentcore-external"
  
  dcr_settings:
    anonymous_registration: false
    protected_registration: true
    software_statement_required: true
    
  supported_flows:
    - "authorization_code"
    - "client_credentials"  # Para alguns third-party
    
  client_policies:
    development_tools:
      pkce_required: true
      redirect_uris: ["http://localhost:*", "https://localhost:*"]
      scopes: ["mcp:read", "mcp:write", "mcp:debug"]
      
    third_party_agents:
      pkce_required: true
      redirect_uris: ["https://*.openai.com/callback", "https://*.anthropic.com/callback"]
      scopes: ["mcp:read", "mcp:write:limited"]
```

#### Cenário 2: Inbound - Use Cognito Quando:

```yaml
use_cognito_when:
  client_type:
    - "internal_ai_agents"     # Agents desenvolvidos internamente
    - "bff_applications"       # Backend for Frontend
    - "service_accounts"       # Contas de serviço
    - "microservices"          # Comunicação entre serviços
  
  authentication_pattern:
    - "client_credentials"     # M2M authentication
    - "service_to_service"     # Comunicação interna
    - "token_exchange"         # RFC 8693 delegation
  
  characteristics:
    - "pre_registered_clients" # Clientes conhecidos
    - "high_volume_traffic"    # Alto volume de requests
    - "predictable_patterns"   # Padrões previsíveis
    - "aws_native_integration" # Integração nativa AWS
  
  traffic_direction: "inbound"  # Para o Gateway
```

**Exemplo de Configuração Cognito Inbound:**
```yaml
cognito_inbound_config:
  user_pool_id: "us-east-1_XXXXXXXXX"
  
  app_clients:
    internal_agents:
      client_name: "internal-ai-agents"
      generate_secret: true
      explicit_auth_flows: ["ALLOW_CLIENT_CREDENTIALS"]
      supported_identity_providers: ["COGNITO"]
      
    bff_applications:
      client_name: "bff-applications"
      generate_secret: true
      explicit_auth_flows: ["ALLOW_CLIENT_CREDENTIALS"]
      token_validity_units:
        access_token: 1  # 1 hour
        
  custom_scopes:
    - "mcp:read"
    - "mcp:write"
    - "agent:execute"
    - "bff:access"
```

#### Cenário 3: Outbound - Use Sempre Cognito:

```yaml
use_cognito_outbound_always:
  reason: "Padronização e integração AWS nativa"
  
  outbound_targets:
    - "mcp_servers"           # MCPs no AgentCore Runtime
    - "external_apis"         # APIs externas via Gateway
    - "aws_services"          # S3, DynamoDB, etc
    - "database_connections"  # RDS, ElastiCache
  
  authentication_pattern:
    - "client_credentials"    # Sempre M2M
    - "service_account_based" # Contas de serviço dedicadas
    - "iam_integration"       # Integração com AWS IAM
  
  characteristics:
    - "high_performance"      # Performance otimizada
    - "aws_managed"           # Gerenciado pela AWS
    - "scalable"              # Escalabilidade automática
    - "cost_effective"        # Custo otimizado
```

**Exemplo de Configuração Cognito Outbound:**
```yaml
cognito_outbound_config:
  user_pool_id: "us-east-1_YYYYYYYYY"  # Pool dedicado para outbound
  
  service_accounts:
    gateway_to_mcp:
      client_name: "gateway-mcp-service"
      generate_secret: true
      explicit_auth_flows: ["ALLOW_CLIENT_CREDENTIALS"]
      custom_scopes: ["mcp:full_access"]
      
    gateway_to_external:
      client_name: "gateway-external-service"
      generate_secret: true
      explicit_auth_flows: ["ALLOW_CLIENT_CREDENTIALS"]
      custom_scopes: ["external:api_access"]
  
  token_settings:
    access_token_validity: 3600  # 1 hour
    refresh_token_validity: 0    # No refresh for M2M
```

### Fluxos de Autenticação por Direção

#### Fluxo Inbound - Keycloak DCR

```mermaid
sequenceDiagram
    participant CLIENT as Cliente Externo<br/>(IDE, Third-Party)
    participant KC as Keycloak
    participant AGW as AgentCore Gateway
    
    Note over CLIENT, AGW: Inbound Authentication - Keycloak DCR
    CLIENT->>KC: 1. Dynamic Client Registration
    KC->>CLIENT: 2. Client Credentials
    CLIENT->>KC: 3. Authorization Code Flow + PKCE
    KC->>CLIENT: 4. Access Token
    CLIENT->>AGW: 5. Request + Access Token
    AGW->>KC: 6. Token Validation
    KC->>AGW: 7. Token Valid + Claims
    AGW->>AGW: 8. Process Request
```

#### Fluxo Inbound - Cognito M2M

```mermaid
sequenceDiagram
    participant AGENT as AI Agent/BFF
    participant COG as Amazon Cognito
    participant AGW as AgentCore Gateway
    
    Note over AGENT, AGW: Inbound Authentication - Cognito M2M
    AGENT->>COG: 1. Client Credentials Grant
    COG->>AGENT: 2. Access Token JWT
    AGENT->>AGW: 3. Request + Access Token
    AGW->>COG: 4. Token Validation
    COG->>AGW: 5. Token Valid + Claims
    AGW->>AGW: 6. Process Request
```

#### Fluxo Outbound - Sempre Cognito

```mermaid
sequenceDiagram
    participant AGW as AgentCore Gateway
    participant COG as Amazon Cognito
    participant MCP as MCP Server/External API
    
    Note over AGW, MCP: Outbound Authentication - Sempre Cognito
    AGW->>COG: 1. Service Account Credentials
    COG->>AGW: 2. Service Access Token
    AGW->>MCP: 3. Outbound Request + Service Token
    MCP->>COG: 4. Token Validation (opcional)
    COG->>MCP: 5. Token Valid
    MCP->>AGW: 6. Response
```

### Configuração do Gateway para Dual Authentication

```yaml
# AgentCore Gateway - Dual Authentication Configuration
gateway_authentication:
  inbound:
    providers:
      keycloak:
        enabled: true
        realm: "agentcore-external"
        discovery_url: "https://keycloak.example.com/auth/realms/agentcore-external/.well-known/openid-configuration"
        client_types: ["development", "third_party", "mcp_clients"]
        
      cognito:
        enabled: true
        user_pool_id: "us-east-1_INBOUND_POOL"
        region: "us-east-1"
        client_types: ["internal_agents", "bff", "services"]
    
    routing_rules:
      - path: "/internal/*"
        auth_provider: "cognito"
        
      - path: "/third-party/*"
        auth_provider: "keycloak"
        
      - path: "/development/*"
        auth_provider: "keycloak"
        
      - path: "/enterprise/*"
        auth_provider: "cognito"
  
  outbound:
    provider: "cognito"  # Sempre Cognito
    user_pool_id: "us-east-1_OUTBOUND_POOL"
    service_accounts:
      - name: "mcp-access"
        client_id: "gateway-mcp-service"
        scopes: ["mcp:full_access"]
        
      - name: "external-api-access"
        client_id: "gateway-external-service"
        scopes: ["external:api_access"]
```

### Benefícios da Abordagem Dual

#### Vantagens Técnicas
- **Especialização**: Cada sistema otimizado para seu caso de uso
- **Performance**: Cognito para M2M, Keycloak para flexibilidade
- **Escalabilidade**: Cognito gerenciado, Keycloak customizável
- **Manutenção**: Redução de overhead operacional

#### Vantagens de Negócio
- **Flexibilidade**: Suporte a diversos tipos de cliente
- **Custo**: Otimização de custos por caso de uso
- **Compliance**: Atendimento a diferentes requisitos
- **Evolução**: Capacidade de evoluir independentemente

### Resumo da Decisão

| Direção | Sistema | Quando Usar | Casos de Uso |
|---------|---------|-------------|--------------|
| **Inbound** | **Keycloak** | Clientes externos, desenvolvimento, third-party | IDEs, MCP Clients, ChatGPT Apps |
| **Inbound** | **Cognito** | Clientes internos, produção, M2M | AI Agents, BFFs, Service Accounts |
| **Outbound** | **Cognito** | Sempre (padronização) | MCPs, APIs Externas, AWS Services |

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

## Estratégia de Autenticação Baseada em Matriz de Decisão

### Resumo da Estratégia Dual

Com base na matriz de decisão Cognito vs Keycloak, nossa estratégia de autenticação segue o princípio de **especialização por caso de uso**:

- **Keycloak**: Inbound traffic de clientes externos e desenvolvimento
- **Cognito**: Inbound traffic interno + Todo outbound traffic

### 4.1 Keycloak para Inbound Traffic Externo (DCR)

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

### 4.2 Amazon Cognito para Inbound Traffic Interno + Todo Outbound Traffic

**Casos de Uso Específicos:**
- **Inbound**: AI Agents internos, BFFs, Service Accounts
- **Outbound**: Comunicação Gateway → MCPs, Gateway → APIs Externas, Gateway → AWS Services
- **Padrão**: Client Credentials Grant (RFC 6749, Seção 4.4) para M2M
- **Token Exchange**: RFC 8693 para delegation scenarios

**Justificativa para Cognito:**
- **Inbound Interno**: Clientes conhecidos e controlados, alta performance M2M
- **Outbound Universal**: Padronização e integração nativa AWS
- **Escalabilidade**: Gerenciado pela AWS, auto-scaling
- **Custo**: Pay-per-use otimizado para alto volume

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

### 4.3 Autenticação para Agentes de Terceiros (B2B)

**Casos de Uso Específicos:**
- ChatGPT Apps que precisam acessar MCPs empresariais
- Claude Apps integradas com sistemas internos
- Agentes de IA de terceiros (Anthropic, OpenAI, outros)
- Ferramentas de automação externas

**Estratégias de Autenticação:**

#### Opção 1: OAuth 2.0 Client Credentials (Recomendado)
```yaml
third_party_oauth:
  grant_type: "client_credentials"
  authentication_method: "client_secret_post"
  token_endpoint: "https://cognito-oauth.amazonaws.com/oauth2/token"
  
  client_registration:
    method: "manual"  # Registro manual por segurança
    approval_process: "admin_approval"
    
  scopes:
    - "mcp:read"
    - "mcp:write:limited"  # Escopo limitado para terceiros
    
  rate_limiting:
    requests_per_hour: 1000
    burst_capacity: 50
```

#### Opção 2: API Keys com JWT
```yaml
third_party_api_keys:
  key_format: "jwt"
  expiration: "30_days"
  rotation: "automatic"
  
  permissions:
    - "mcp_access"
    - "read_only"  # Padrão conservador
    
  validation:
    signature_verification: true
    audience_check: true
    issuer_validation: true
```

#### Fluxo de Autenticação para Terceiros

```mermaid
sequenceDiagram
    participant THIRD as Agente de Terceiros
    participant REG as Registration Service
    participant COG as Amazon Cognito
    participant AGW as AgentCore Gateway
    participant MCP as MCP Server
    
    Note over THIRD, MCP: Registro e Autenticação de Terceiros
    
    rect rgb(255, 240, 240)
        Note over THIRD, COG: Processo de Registro (One-time)
        THIRD->>REG: Solicitação de Registro
        REG->>REG: Validação e Aprovação Manual
        REG->>COG: Cria Client Credentials
        COG->>REG: Client ID + Secret
        REG->>THIRD: Credenciais de Acesso
    end
    
    rect rgb(240, 255, 240)
        Note over THIRD, MCP: Fluxo de Acesso (Runtime)
        THIRD->>COG: Client Credentials Grant
        COG->>COG: Valida Credenciais
        COG->>THIRD: Access Token JWT
        THIRD->>AGW: MCP Request + Token
        AGW->>COG: Token Validation
        COG->>AGW: Token Valid + Claims
        AGW->>MCP: Authorized Request
        MCP->>AGW: Response
        AGW->>THIRD: Final Response
    end
```

### Vantagens da Abordagem Dual Especializada:
- **Inbound Otimizado**: Keycloak para flexibilidade externa, Cognito para performance interna
- **Outbound Padronizado**: Cognito para toda comunicação de saída (MCPs, APIs, AWS)
- **Especialização**: Cada sistema otimizado para seu caso de uso específico
- **Manutenção**: Redução de overhead operacional com Cognito gerenciado

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

#### Third-Party Agents Configuration
```yaml
# Third-Party Agents Access Configuration
third_party_config:
  supported_agents:
    - name: "chatgpt_apps"
      authentication: "oauth2_client_credentials"
      rate_limit: "1000/hour"
      scopes: ["mcp:read", "mcp:write:limited"]
      
    - name: "claude_apps"
      authentication: "oauth2_client_credentials"
      rate_limit: "1500/hour"
      scopes: ["mcp:read", "mcp:write:limited"]
      
    - name: "custom_agents"
      authentication: "api_key_jwt"
      rate_limit: "500/hour"
      scopes: ["mcp:read"]
  
  security_controls:
    ip_whitelist: true
    request_signing: true
    audit_logging: "detailed"
    
  gateway_routing:
    path_prefix: "/third-party"
    cors_enabled: true
    timeout: "30s"
```

#### Agent Configuration for B2B
```yaml
# B2B Agent Configuration (Internal + Third-Party)
b2b_agents:
  internal_agents:
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
  
  third_party_access:
    gateway_endpoint: "agentcore_gateway"
    authentication_required: true
    operations_allowed:
      - "tools/list"
      - "tools/call"
      - "resources/read"
    
    restrictions:
      - "no_admin_operations"
      - "read_only_resources"
      - "limited_tool_execution"
  
  mcp_access:
    method: "via_gateway"
    gateway_endpoint: "agentcore_gateway"
    operations:
      - "external_apis"
      - "database_queries"
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
      
    b2b_partners:
      authentication: "cognito_m2m"
      access_pattern: "via_bff"
      rate_limiting: "enterprise_tier"
      
    b2b_third_party:
      authentication: "oauth2_or_api_key"
      access_pattern: "direct"
      rate_limiting: "third_party_tier"
  
  mcp_servers:
    - name: "database-mcp"
      runtime: "agentcore"
      scenarios: ["internal", "b2c", "b2b_partners", "b2b_third_party"]
      third_party_permissions: "read_only"
      
    - name: "api-integration-mcp"
      runtime: "agentcore"
      scenarios: ["b2c", "b2b_partners", "b2b_third_party"]
      third_party_permissions: "limited_write"
      
    - name: "development-mcp"
      runtime: "agentcore"
      scenarios: ["internal"]
      third_party_permissions: "none"
  
  routing_rules:
    internal:
      path_prefix: "/internal"
      auth_required: true
      debug_enabled: true
      
    b2c:
      path_prefix: "/consumer"
      auth_required: true
      caching_enabled: true
      
    b2b_partners:
      path_prefix: "/enterprise"
      auth_required: true
      audit_logging: true
      
    b2b_third_party:
      path_prefix: "/third-party"
      auth_required: true
      rate_limiting: "strict"
      audit_logging: "detailed"
      cors_enabled: true
      
  third_party_controls:
    allowed_operations:
      - "tools/list"
      - "tools/call"
      - "resources/read"
    
    restricted_operations:
      - "admin/*"
      - "config/*"
      - "debug/*"
    
    security_headers:
      - "X-Third-Party-Client-ID"
      - "X-Request-Source"
      - "X-Rate-Limit-Remaining"
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
- **RFC 8693**: OAuth 2.0 Token Exchange - Delegation e impersonation para contexto de cliente
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

#### 3. **Matriz de Decisão Cognito vs Keycloak**
- **Keycloak para Inbound Externo**: IDEs, MCP Clients, Third-Party Apps com DCR
- **Cognito para Inbound Interno**: AI Agents, BFFs, Service Accounts com M2M
- **Cognito para Todo Outbound**: Padronização para MCPs, APIs e AWS Services
- **Especialização por Caso de Uso**: Cada sistema otimizado para seu contexto

#### 4. **Conformidade com Especificações Oficiais**
- **RFC 7591** para Dynamic Client Registration
- **RFC 6749 Seção 4.4** para Client Credentials Grant
- **RFC 8693** para Token Exchange e delegation scenarios
- **RFC 8707** para Resource Indicators e token audience binding
- **MCP Authorization Specification** para descoberta e validação de servidores
#### 5. **Segurança Aprimorada**
- **Token Audience Binding** para prevenir token passthrough
- **PKCE obrigatório** para MCP clients públicos
- **Dual Authentication Strategy** com especialização por caso de uso
- **Validação rigorosa de contexto** em cada request
- **Encryption at rest e in transit** com chaves específicas por tenant

### Benefícios da Arquitetura Revisada

- **Flexibilidade de Distribuição**: DCR permite que ferramentas como Kiro se registrem dinamicamente
- **Escalabilidade M2M**: Client Credentials otimizado para comunicação de alta escala entre agentes
- **Padronização Outbound**: Cognito para toda comunicação de saída do gateway
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