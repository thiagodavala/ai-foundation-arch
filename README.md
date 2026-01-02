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

**Implementação Técnica**:
```mermaid
graph TB
    subgraph "Public Subnets"
        AGW[AgentCore Gateway]
        ALB[Application Load Balancer]
    end
    
    subgraph "Private Subnets"
        MCP1[Database MCP]
        MCP2[Cache MCP]
        MCP3[Internal API MCP]
        
        RDS[(Amazon RDS)]
        REDIS[(ElastiCache Redis)]
        API[Internal APIs]
    end
    
    subgraph "VPC Endpoints"
        S3EP[S3 VPC Endpoint]
        SSMEP[SSM VPC Endpoint]
    end
    
    Internet --> ALB
    ALB --> AGW
    AGW --> MCP1
    AGW --> MCP2
    AGW --> MCP3
    
    MCP1 --> RDS
    MCP2 --> REDIS
    MCP3 --> API
    
    MCP1 --> S3EP
    MCP2 --> SSMEP
    
    style AGW fill:#ff9800
    style RDS fill:#4caf50
    style REDIS fill:#f44336
    style API fill:#2196f3
```

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

### Matriz de Decisão Arquitetural

```mermaid
graph TD
    subgraph "Decision Matrix"
        D1{Direct MCP Access Needed?}
        D2{High Volume/Low Latency?}
        D3{Complex Business Logic?}
        D4{External Integration?}
        
        D1 -->|Yes| CLIENT[MCP Client + Keycloak DCR]
        D1 -->|No| D2
        D2 -->|Yes| DIRECT[Direct Agent-MCP via Gateway]
        D2 -->|No| D3
        D3 -->|Yes| A2A[Agent-to-Agent Communication]
        D3 -->|No| D4
        D4 -->|Yes| GATEWAY[Gateway-mediated Integration]
        D4 -->|No| SIMPLE[Simple Agent-MCP Pattern]
    end
    
    style CLIENT fill:#ff9800
    style A2A fill:#4caf50
    style GATEWAY fill:#2196f3
    style SIMPLE fill:#9c27b0
```



## Arquitetura de Alto Nível

```mermaid
graph TB
    subgraph "AI Agents Layer"
        POC[POC Agents<br/>AgentCore Runtime]
        PROD[Production Agents<br/>Amazon EKS]
    end
    
    subgraph "Gateway Layer"
        AGW[AgentCore Gateway<br/>MCP Orchestration]
    end
    
    subgraph "MCP Servers Layer"
        MCP1[Database MCP<br/>AgentCore Runtime]
        MCP2[API Integration MCP<br/>AgentCore Runtime]
        MCP3[Custom MCP<br/>AgentCore Runtime]
    end
    
    subgraph "Authentication Layer"
        KC[Keycloak<br/>DCR]
        COG[Amazon Cognito<br/>M2M]
    end
    
    subgraph "AWS Services"
        EKS[Amazon EKS]
        BR[Amazon Bedrock]
        VPC[Amazon VPC]
        IAM[AWS IAM]
    end
    
    POC --> AGW
    PROD --> AGW
    AGW --> MCP1
    AGW --> MCP2
    AGW --> MCP3
    
    AGW -.-> KC
    AGW --> COG
    
    PROD --> EKS
    AGW --> BR
    EKS --> VPC
    EKS --> IAM
    
    style POC fill:#e1f5fe
    style PROD fill:#f3e5f5
    style AGW fill:#fff3e0
    style KC fill:#e8f5e8
    style COG fill:#e8f5e8
```

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

### 2. MCP Servers (AgentCore Runtime)

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

## Implementação Detalhada

### 5.1 Configuração do AgentCore Gateway

```yaml
# Configuração exemplo do Gateway
gateway_config:
  mcp_servers:
    - name: "database-mcp"
      runtime: "agentcore"
      auth_method: "cognito_m2m"
      scopes: ["read:database", "write:database"]
    
    - name: "api-integration-mcp"
      runtime: "agentcore"
      source_api: "https://api.example.com/openapi.json"
      auth_method: "cognito_m2m"
      scopes: ["read:api"]

  authentication:
    cognito:
      user_pool_id: "us-east-1_XXXXXXXXX"
      client_id: "gateway-client-id"
    
    keycloak:
      realm: "mcp-realm"
      dcr_endpoint: "https://keycloak.example.com/auth/realms/mcp-realm/clients-registrations/openid-connect"
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

## Custos e Otimização

### Modelo de Custos por Ambiente

```mermaid
pie title Distribuição de Custos - Ambiente POC
    "AgentCore Runtime" : 45
    "Amazon Cognito" : 15
    "Data Transfer" : 20
    "CloudWatch" : 10
    "Outros Serviços" : 10
```

```mermaid
pie title Distribuição de Custos - Ambiente Produção
    "EKS Control Plane" : 5
    "EC2 Instances" : 50
    "Fargate" : 25
    "Load Balancers" : 8
    "Storage" : 7
    "Data Transfer" : 5
```

### Estratégia de Otimização de Custos

```mermaid
graph TD
    subgraph "Cost Optimization Strategy"
        SPOT[Spot Instances<br/>70% savings]
        RESERVED[Reserved Instances<br/>40% savings]
        AUTOSCALE[Auto Scaling<br/>30% savings]
        RIGHTSIZING[Right Sizing<br/>25% savings]
    end
    
    subgraph "Monitoring & Alerts"
        BUDGET[AWS Budgets]
        COST[Cost Explorer]
        TAGS[Resource Tagging]
    end
    
    subgraph "Workload Types"
        DEV[Development<br/>Spot Instances]
        TEST[Testing<br/>On-Demand]
        PROD[Production<br/>Reserved + Spot]
    end
    
    SPOT --> DEV
    RESERVED --> PROD
    AUTOSCALE --> TEST
    RIGHTSIZING --> PROD
    
    BUDGET --> COST
    COST --> TAGS
    TAGS --> DEV
    TAGS --> TEST
    TAGS --> PROD
    
    style SPOT fill:#4caf50
    style RESERVED fill:#2196f3
    style AUTOSCALE fill:#ff9800
    style RIGHTSIZING fill:#9c27b0
```

### 9.1 Modelo de Custos

**POC Environment:**
- AgentCore Runtime: $0.10/hora por instância ativa
- Cognito: $0.0055 por MAU (Monthly Active User)
- Data Transfer: $0.09/GB

**Production Environment:**
- EKS Control Plane: $0.10/hora
- EC2 Instances: Variável baseado no tipo de instância
- Fargate: $0.04048/vCPU/hora + $0.004445/GB/hora
- Load Balancer: $0.0225/hora

### 9.2 Estratégias de Otimização

- **Spot Instances**: Uso de instâncias spot para workloads tolerantes a interrupção
- **Reserved Instances**: Reserva de capacidade para cargas previsíveis
- **Auto Scaling**: Dimensionamento automático baseado em demanda
- **Resource Tagging**: Rastreamento detalhado de custos por projeto/ambiente

## Roadmap de Implementação

```mermaid
gantt
    title Roadmap de Implementação - Fundação MCP & AI Agents
    dateFormat  YYYY-MM-DD
    section Fase 1: Fundação
    Setup AgentCore Gateway           :active, f1-1, 2025-01-02, 1w
    Configuração Cognito             :f1-2, after f1-1, 1w
    Deploy MCP Servers               :f1-3, after f1-2, 1w
    Ambiente POC                     :f1-4, after f1-3, 1w
    
    section Fase 2: Produção
    Configuração EKS                 :f2-1, after f1-4, 1w
    Implementação Keycloak           :f2-2, after f2-1, 1w
    Migração Agents EKS              :f2-3, after f2-2, 1w
    Setup Monitoramento              :f2-4, after f2-3, 1w
    
    section Fase 3: Otimização
    Implementação Caching            :f3-1, after f2-4, 1w
    Otimização Performance           :f3-2, after f3-1, 1w
    Multi-região                     :f3-3, after f3-2, 1w
    CI/CD Automação                  :f3-4, after f3-3, 1w
    
    section Fase 4: Governança
    Políticas Segurança              :f4-1, after f3-4, 1w
    Compliance & Auditoria           :f4-2, after f4-1, 1w
    Documentação                     :f4-3, after f4-2, 1w
    Treinamento                      :f4-4, after f4-3, 1w
```

### Detalhamento das Fases

### Fase 1: Fundação (Semanas 1-4)
- [ ] Setup do AgentCore Gateway
- [ ] Configuração básica do Cognito
- [ ] Deploy de MCP servers de exemplo
- [ ] Ambiente de POC funcional

### Fase 2: Produção (Semanas 5-8)
- [ ] Configuração do cluster EKS
- [ ] Implementação do Keycloak
- [ ] Migração de agents para EKS
- [ ] Configuração de monitoramento

### Fase 3: Otimização (Semanas 9-12)
- [ ] Implementação de caching
- [ ] Otimização de performance
- [ ] Configuração de multi-região
- [ ] Automação completa de CI/CD

### Fase 4: Governança (Semanas 13-16)
- [ ] Políticas de segurança avançadas
- [ ] Compliance e auditoria
- [ ] Documentação completa
- [ ] Treinamento de equipes

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