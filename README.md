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

## Estratégia de Autenticação

### Fluxo de Autenticação Completo

```mermaid
sequenceDiagram
    participant Agent as AI Agent
    participant KC as Keycloak
    participant AGW as AgentCore Gateway
    participant COG as Amazon Cognito
    participant MCP as MCP Server
    
    Note over Agent, MCP: Dynamic Client Registration (DCR) Flow
    Agent->>KC: 1. DCR Request (RFC 7591)
    KC->>KC: 2. Validate Software Statement
    KC->>Agent: 3. Client Credentials (client_id, client_secret)
    
    Note over Agent, MCP: Agent Authentication Flow
    Agent->>AGW: 4. Request with Client Credentials
    AGW->>KC: 5. Validate Client Credentials
    KC->>AGW: 6. Access Token (JWT)
    
    Note over Agent, MCP: M2M Communication Flow
    AGW->>COG: 7. Client Credentials Grant
    COG->>AGW: 8. M2M Access Token
    AGW->>MCP: 9. MCP Request + M2M Token
    MCP->>COG: 10. Token Validation
    COG->>MCP: 11. Token Valid
    MCP->>AGW: 12. MCP Response
    AGW->>Agent: 13. Final Response
```

### 4.1 Keycloak para Dynamic Client Registration (DCR)

**Casos de Uso:**
- Registro dinâmico de clientes MCP
- Cenários que requerem flexibilidade de registro
- Integração com sistemas de identidade existentes

**Implementação Técnica:**
- Conformidade com RFC 7591 (OAuth 2.0 Dynamic Client Registration)
- Suporte a software statements para validação de clientes
- Registro anônimo e protegido conforme necessidade

**Exemplo de Fluxo DCR:**
```json
{
  "software_id": "MCP-CLIENT-001",
  "client_name": "AI Agent MCP Client",
  "client_uri": "https://agent.example.com/",
  "grant_types": ["client_credentials"],
  "token_endpoint_auth_method": "client_secret_basic"
}
```

### 4.2 Amazon Cognito para Machine-to-Machine (M2M)

**Casos de Uso:**
- Comunicação Gateway → MCP Servers
- Autenticação entre serviços internos
- Cenários de alta escala e performance

**Implementação Técnica:**
- OAuth 2.0 Client Credentials Grant
- Tokens JWT com scopes específicos
- Integração nativa com API Gateway para validação

**Vantagens da Abordagem Híbrida:**
- **Distribuição de Carga**: Keycloak para DCR complexo, Cognito para M2M em escala
- **Otimização de Custos**: Uso eficiente de recursos de cada serviço
- **Flexibilidade**: Adaptação a diferentes padrões de uso

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

### RFCs e Especificações
- **RFC 7591**: OAuth 2.0 Dynamic Client Registration Protocol
- **RFC 6749**: The OAuth 2.0 Authorization Framework
- **RFC 7636**: Proof Key for Code Exchange by OAuth Public Clients
- **RFC 8707**: Resource Indicators for OAuth 2.0
- **MCP Specification**: Model Context Protocol v2025-11-25

### Documentação AWS
- [Amazon EKS Best Practices Guide](https://aws.github.io/aws-eks-best-practices/)
- [Amazon Cognito Developer Guide](https://docs.aws.amazon.com/cognito/)
- [Amazon Bedrock AgentCore Documentation](https://aws.amazon.com/bedrock/agentcore/)
- [AWS Security Reference Architecture](https://docs.aws.amazon.com/prescriptive-guidance/latest/security-reference-architecture/)

### Recursos Adicionais
- [Model Context Protocol Official Site](https://modelcontextprotocol.io/)
- [Keycloak Documentation](https://www.keycloak.org/documentation)
- [OAuth 2.0 Security Best Practices](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics)

## Conclusão

Esta arquitetura fornece uma base sólida e escalável para implementação de MCPs e AI Agents na AWS, combinando a simplicidade do AgentCore Runtime para desenvolvimento e POCs com a robustez do Amazon EKS para ambientes de produção. A estratégia de autenticação híbrida (Keycloak + Cognito) oferece flexibilidade e otimização de custos, enquanto o AgentCore Gateway centraliza o gerenciamento e orquestração de MCPs.

A implementação faseada permite evolução gradual da solução, minimizando riscos e permitindo aprendizado contínuo. O foco em segurança, observabilidade e otimização de custos garante que a solução seja adequada para ambientes empresariais críticos.

---

**Versão**: 1.0  
**Data**: Janeiro 2025  
**Autor**: Arquitetura de Soluções AWS  
**Status**: Documento Vivo - Sujeito a atualizações baseadas em feedback e evolução das tecnologias