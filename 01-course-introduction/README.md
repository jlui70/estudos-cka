# Capítulo 01: Basics of Kubernetes

## 1. Por que orquestração é necessária?

Antes do K8s, as empresas rodavam workloads em VMs ou diretamente em servidores físicos. Com a popularização dos containers (Docker, 2013), passou a ser simples empacotar e rodar aplicações de forma isolada. Mas surgiram novos problemas:
Problema	Sem Orquestração	Com Kubernetes
Container cai	Precisa reiniciar manualmente	Auto-restart via controller
Alta demanda	Scale manual	HorizontalPodAutoscaler
Múltiplos hosts	Deploy manual em cada host	Scheduler distribui automaticamente
Atualização	Downtime ou script customizado	Rolling Update nativo
Service discovery	DNS/arquivo manual	CoreDNS integrado

## 2. História e Origem

### Linha do tempo

- **2003–2013 — Google Borg**: Sistema interno do Google para orquestrar dezenas de milhares de jobs. Toda a infraestrutura do Google Search, Gmail etc. rodava no Borg. Era proprietário.
- **2013 — Google Omega**: Segunda geração do Borg, com foco em melhor compartilhamento de recursos. Também proprietário.
- **Junho 2014 — Kubernetes**: Google anuncia o K8s como projeto open-source, construído com as lições aprendidas no Borg. Lançado no GitHub.
- **2015 — CNCF**: K8s torna-se o projeto fundador da **Cloud Native Computing Foundation** (CNCF), parte da Linux Foundation.
- **2016**: K8s supera Docker Swarm e Apache Mesos como padrão de mercado.
- **2018**: K8s se "gradua" na CNCF — primeiro projeto a atingir esse nível de maturidade.
- **Hoje**: Mais de 100 versões lançadas, suporte nativo nos três maiores clouds (GKE, EKS, AKS) e em praticamente todos os provedores de cloud.
