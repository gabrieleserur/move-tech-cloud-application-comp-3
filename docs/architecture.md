1. Estilo Arquitetural
A solução adota o estilo de Monolito em Camadas (Layered Monolith), estruturado logicamente em Apresentação → Serviço/Regra de Negócio → Dados, empacotado como um container único e implantado com 2 réplicas no Kubernetes.

Arquitetura Atual: Monolito em camadas implantado de forma redundante (2 pods) para resiliência básica e distribuição de carga HTTP.

Estilo-Alvo (Evolução do Domínio): Caso o domínio (ex: processamento de notificações/pedidos) cresça em volume ou complexidade, a estratégia arquitetural prevê a extração gradual de microsserviços orientados a eventos.

## Estilo Arquitetural

A solução adota o estilo **Monolito em Camadas (Layered Monolith)**, estruturado de forma clara em três responsabilidades principais:

* **Apresentação:** Tratamento das requisições e rotas HTTP.
* **Serviço:** Regras de negócio da aplicação.
* **Dados:** Comunicação com o banco de dados.

O sistema é implantado como um **container único** operando com **duas réplicas** no Kubernetes para garantir resiliência básica e distribuição de carga. 

> **Evolução da Arquitetura:** Caso o domínio de notificações ou processamento de pedidos cresça em volume ou complexidade, a estratégia arquitetural prevê a extração gradual dessas responsabilidades para um segundo serviço independente.

2. Diagrama de Arquitetura (Mermaid)

Snippet de código
graph TD
    subgraph GitHub ["External: GitHub"]
        GHA["GitHub Actions (CI/CD)"]
    end

    subgraph MGC ["Magalu Cloud (MGC)"]
        subgraph VM ["VM BV2-2-40 (K3s Single-Node)"]
            
            SLB["Klipper ServiceLB\n(Porta 80)"]
            
            subgraph K8s_Default ["Namespace: default"]
                APP1["API Pod 1\n(Cloud Application)"]
                APP2["API Pod 2\n(Cloud Application)"]
                SVC_APP["Service: cloud-application\n(ClusterIP)"]
                SM["ServiceMonitor\n(cloud-application)"]
            end

            subgraph K8s_Monitoring ["Namespace: monitoring"]
                PROM["Prometheus Operator"]
                GRAF["Grafana Dashboard"]
            end

        end

        CR["Container Registry (MGC)"]
        DB["DBaaS PostgreSQL"]
    end

    %% Conexões de Tráfego e CI/CD
    User["Usuário / Cliente HTTP"] -->|HTTP / TCP 80| SLB
    SLB -->|HTTP / TCP internal| SVC_APP
    SVC_APP -->|HTTP / Load Balance| APP1
    SVC_APP -->|HTTP / Load Balance| APP2

    %% Persistência e Imagens
    APP1 -->|SQL / TCP 5432| DB
    APP2 -->|SQL / TCP 5432| DB

    GHA -->|Push Docker Image / HTTPS 443| CR
    GHA -->|kubectl apply / HTTPS 6443| VM
    VM -->|Pull Image / HTTPS 443| CR

    %% Monitoramento e Coleta
    SM -.->|Labels Matching| SVC_APP
    PROM -->|HTTP Scraping /metrics (30s)| APP1
    PROM -->|HTTP Scraping /metrics (30s)| APP2
    GRAF -->|PromQL / HTTP 9090| PROM

3. Componentes da Arquitetura

API K3s (VM single node) K3s (VM single node)
Banco de Dados	DBaaS PostgreSQL	Serviço gerenciado que persiste com segurança os dados de pedidos e itens.
Imagens	Container Registry (MGC)	Armazena e versiona as imagens Docker da aplicação.
Tráfego Externo	Klipper ServiceLB	Escuta no IP público da VM (porta 80) e distribui as requisições entre os Pods da API.
CI/CD	GitHub Actions	Esteira automatizada responsável pela validação, build da imagem e deploy no cluster.
Observabilidade	Prometheus & Grafana	Monitora a saúde do cluster e extrai métricas de negócio/infra via ServiceMonitor.

## Componentes da Arquitetura

| Componente | Serviço MGC | Função |
| :--- | :--- | :--- |
| **API** | K3s (VM single node) — 2 réplicas | Processa as requisições HTTP |
| **Banco de dados** | DBaaS PostgreSQL | Persiste pedidos e itens |
| **Imagens** | Container Registry | Armazena versões da aplicação |
| **Tráfego externo** | Klipper ServiceLB (IP da VM, porta 80) | Distribui entre as réplicas e fornece acesso externo |
| **CI/CD** | GitHub Actions | Automatiza testes, build e deploy |

4. Requisitos Não-Funcionais (NFRs)

Disponibilidade	Probes de Liveness/Readiness no Kubernetes + Proporção de respostas sem erro 5xx no Grafana (sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])))	99,5% mensal
Latência	Cálculo de percentil no Prometheus via métricas /metrics (histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m]) by (le))))	P95 < 500 ms
Escalabilidade	Teste de carga com k6 avaliando a vazão de requisições por segundo (rate(http_requests_total[1m]))	300 req/s sem degradação do P95 ou erros
Custo	Custo mensal somado da VM BV2-2-40, instância DBaaS PostgreSQL e IP público conforme calculadora MGC	Teto definido em ADR de Infraestrutura

## Componentes da Arquitetura

| Componente | Serviço MGC | Função |
| :--- | :--- | :--- |
| **API** | K3s (VM single node) — 2 réplicas | Processa as requisições HTTP |
| **Banco de dados** | DBaaS PostgreSQL | Persiste pedidos e itens |
| **Imagens** | Container Registry | Armazena versões da aplicação |
| **Tráfego externo** | Klipper ServiceLB (IP da VM, porta 80) | Distribui entre as réplicas e fornece acesso externo |
| **CI/CD** | GitHub Actions | Automatiza testes, build e deploy |

## Tabela de Trade-offs Arquiteturais

| Aspecto | Decisão tomada | Alternativa não escolhida | Motivo da escolha |
| :--- | :--- | :--- | :--- |
| **Deploy** | K3s em VM | MKS (Kubernetes Gerenciado) | Custo menor, provisionamento < 2 min, manifests idênticos |
| **Banco** | DBaaS gerenciado | PostgreSQL em container | Backup automático, sem administração |
| **CI/CD** | GitHub Actions | Deploy manual | Consistência e rastreabilidade |
| **Réplicas** | 2 pods | 1 pod | Disponibilidade mínima sem custo excessivo |
| **API** | FastAPI (Python) | Node.js, Go, Java | Curva de aprendizado baixa, alta produtividade |

Escalabilidade
A aplicação é stateless, então escala na horizontal — mais réplicas atrás do balanceador. Hoje são 2 réplicas fixas; o próximo passo natural é o HPA (Horizontal Pod Autoscaler), que ajusta esse número automaticamente pela utilização de CPU (ex.: mínimo 2, máximo 6, alvo de 70%). Vale registrar também que mais réplicas não resolvem um gargalo de banco — o PostgreSQL escala na vertical e costuma saturar primeiro.

## Próximos Passos Naturais

| Melhoria | Por quê |
| :--- | :--- |
| **HTTPS / TLS** | Toda API em produção deve ser acessada por HTTPS |
| **Autoscaler (HPA)** | Escala automaticamente conforme a carga |
| **Versionamento de API** | `/v1/orders` permite evoluir sem quebrar clientes |
| **Rate limiting** | Evita abuso e protege o banco de sobrecargas |
| **Cache (Redis)** | Reduz consultas repetidas ao banco |
| **Migrações de schema (Alembic)** | Controle de versão das mudanças no banco |
| **Testes de carga** | Valida o comportamento sob alto tráfego |
| **Migrar para MKS** | Quando precisar de HA real: basta trocar o kubeconfig — os manifests YAML são idênticos |

## Custo Estimado na Magalu Cloud

| Recurso | Especificação | Observação |
| :--- | :--- | :--- |
| **VM K3s** | BV2-2-40 (2 vCPU, 2 GB) | Cobrada por hora de uso |
| **DBaaS PostgreSQL** | Instância pequena | Cobrado por hora de uso |
| **Container Registry** | Por armazenamento | Baixo para imagens < 500 MB |
