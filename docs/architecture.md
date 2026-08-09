1. Estilo Arquitetural
A solução adota o estilo de Monolito em Camadas (Layered Monolith), estruturado logicamente em Apresentação → Serviço/Regra de Negócio → Dados, empacotado como um container único e implantado com 2 réplicas no Kubernetes.

Arquitetura Atual: Monolito em camadas implantado de forma redundante (2 pods) para resiliência básica e distribuição de carga HTTP.

Estilo-Alvo (Evolução do Domínio): Caso o domínio (ex: processamento de notificações/pedidos) cresça em volume ou complexidade, a estratégia arquitetural prevê a extração gradual de microsserviços orientados a eventos.


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

4. Requisitos Não-Funcionais (NFRs)

Disponibilidade	Probes de Liveness/Readiness no Kubernetes + Proporção de respostas sem erro 5xx no Grafana (sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])))	99,5% mensal
Latência	Cálculo de percentil no Prometheus via métricas /metrics (histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m]) by (le))))	P95 < 500 ms
Escalabilidade	Teste de carga com k6 avaliando a vazão de requisições por segundo (rate(http_requests_total[1m]))	300 req/s sem degradação do P95 ou erros
Custo	Custo mensal somado da VM BV2-2-40, instância DBaaS PostgreSQL e IP público conforme calculadora MGC	Teto definido em ADR de Infraestrutura
