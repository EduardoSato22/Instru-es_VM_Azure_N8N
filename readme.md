Infraestrutura de Automação em Nuvem — Azure + Docker + Cloudflare

Este projeto implementa uma plataforma completa de automação hospedada na Azure, com custo zero de operação e total automação de inicialização, DNS e backup.
A solução foi desenvolvida para rodar uma pilha de serviços modernos, mantendo resiliência, escalabilidade e eficiência financeira.

🚀 Objetivo

Hospedar e orquestrar aplicações de automação (como n8n, Evolution API e Redis) em uma infraestrutura gratuita e profissional, com infraestrutura como código e auto-recuperação total.

🧩 Arquitetura da Solução

Serviços Principais:

n8n — Plataforma de automação de fluxos.

Evolution API — Interface para integrações e automação de mensagens.

Redis — Armazenamento em cache e fila de tarefas.

Infraestrutura:

Azure Free Tier (VM Linux) — Servidor base de custo zero.

Docker Compose — Gerenciamento e isolamento dos containers.

Nginx — Proxy reverso com gerenciamento de SSL e roteamento.

Cloudflare API — Atualização dinâmica do DNS.

Systemd (Ubuntu) — Automação da inicialização dos serviços.

⚙️ Automação e Scripts

Agendamento de inicialização/desligamento:
A VM é ligada e desligada automaticamente em horários definidos, evitando custos de processamento ocioso.

Script de atualização de IP dinâmico:

Detecta o novo IP público da VM a cada boot.

Atualiza automaticamente o registro DNS (Tipo A) via Cloudflare API.

Inicialização automática com systemd:

Inicia o Docker e os containers (Nginx, n8n, Evolution API, Redis).

Executa o script de atualização de IP.

Toda a pilha fica disponível em menos de 2 minutos.

🔒 Segurança e Resiliência

Backups diários automáticos via snapshot na Azure.

Containers isolados para cada serviço.

SSL ativo via Nginx com certificados atualizados automaticamente.

Infraestrutura auto-recuperável: todo o ambiente se reconstrói ao ligar a VM.

💡 Resultado

Uma plataforma de automação em nuvem:

Robusta e segura.

Com DNS dinâmico e automação completa.

Custando zero reais em operação.

Ideal para desenvolvedores, entusiastas de DevOps, e profissionais que desejam aprender sobre infraestrutura escalável e eficiente na Azure.

🧠 Tecnologias Utilizadas

Azure · Linux (Ubuntu) · Docker Compose · Nginx · Redis · n8n · Evolution API · Cloudflare API · Systemd · Shell Script

Quer que eu adapte esse texto para o formato ideal de README.md (com badges, seções de instalação e execução, e código de exemplo)?
Posso transformar essa descrição em um modelo de repositório pronto.

Você disse:
350 caracters
O ChatGPT disse:

Perfeito — aqui está uma versão profissional e concisa (≈350 caracteres) para a descrição do seu repositório no GitHub:

Plataforma de automação na Azure com custo zero, utilizando Docker Compose, Nginx, Redis, n8n e Evolution API. Inclui automação de inicialização, atualização dinâmica de DNS via Cloudflare e backups diários, garantindo resiliência, segurança e eficiência operacional.

Você disse:

Quer que eu adapte esse texto para o formato ideal de **README.md** (com badges, seções de instalação e execução, e código de exemplo)?

quero sim e me mande adaptado com tudo pronto
O ChatGPT disse:

Perfeito 👍 Aqui está o README.md completo, já formatado e pronto para uso no GitHub — com seções profissionais, badges, instruções de instalação, execução, e destaque das tecnologias.

# Infraestrutura de Automação na Azure  
![Azure](https://img.shields.io/badge/Azure-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-009639?style=for-the-badge&logo=nginx&logoColor=white)
![Cloudflare](https://img.shields.io/badge/Cloudflare-F38020?style=for-the-badge&logo=cloudflare&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)

> Plataforma de automação na Azure com custo zero, utilizando Docker Compose, Nginx, Redis, n8n e Evolution API.  
> Inclui automação de inicialização, atualização dinâmica de DNS via Cloudflare e backups diários, garantindo resiliência, segurança e eficiência operacional.

---

## 📘 Visão Geral

Este projeto cria uma infraestrutura de automação **100% automatizada** e **de custo zero** na **Azure**, utilizando **containers Docker** para rodar:
- **n8n** (automação de fluxos)
- **Evolution API** (integrações)
- **Redis** (cache e filas)

Toda a plataforma é orquestrada com **Docker Compose**, exposta via **Nginx**, protegida com SSL e com **DNS dinâmico atualizado automaticamente** via **Cloudflare API**.

---

## 🧩 Arquitetura da Solução

```mermaid
flowchart TD
    A[Usuário] -->|HTTPS| B[Nginx Proxy]
    B --> C[n8n]
    B --> D[Evolution API]
    B --> E[Redis]
    F[Cloudflare DNS] --> B
    G[Systemd - Ubuntu] -->|Inicializa| B

⚙️ Automação Implementada

Agendamento de inicialização/desligamento da VM (Azure Free Tier)

Script de IP dinâmico: detecta o IP público e atualiza automaticamente o DNS via Cloudflare

Inicialização automática: o systemd inicia Docker, containers e o script DNS

Backup diário: snapshots automáticos da VM

🧠 Tecnologias Utilizadas
Categoria	Tecnologias
Cloud	Azure Free Tier
Orquestração	Docker Compose
Web Server	Nginx
Banco de Dados	Redis
Automação	n8n, Evolution API
DNS Dinâmico	Cloudflare API
Sistema Operacional	Ubuntu Linux
Scripts	Shell Script + Systemd
🧰 Instalação

Clone o repositório:

git clone https://github.com/usuario/nome-do-repositorio.git
cd nome-do-repositorio


Configure variáveis de ambiente:

cp .env.example .env
# Edite o arquivo com suas credenciais da Cloudflare e URLs


Suba os containers:

docker-compose up -d


Acesse via navegador:

https://seu-dominio.com

🔄 Script de Atualização de IP (exemplo)
#!/bin/bash
# Atualiza automaticamente o IP público no DNS da Cloudflare
IP=$(curl -s https://api.ipify.org)
ZONE_ID="xxxxxxxxxxxx"
RECORD_ID="xxxxxxxxxxxx"
EMAIL="seuemail@dominio.com"
API_KEY="sua_chave_api"

curl -X PUT "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records/$RECORD_ID" \
-H "X-Auth-Email: $EMAIL" \
-H "X-Auth-Key: $API_KEY" \
-H "Content-Type: application/json" \
--data "{\"type\":\"A\",\"name\":\"seu-dominio.com\",\"content\":\"$IP\",\"ttl\":120,\"proxied\":false}"

🔒 Segurança e Resiliência

SSL ativo via Nginx

Containers isolados

Backups automáticos na Azure

Auto-recuperação total em caso de reboot

📈 Resultado

Uma plataforma de automação robusta, auto-gerenciável e gratuita, ideal para estudar ou implantar soluções leves em nuvem, mantendo boas práticas de DevOps e Infra as Code.

📄 Licença

Distribuído sob a licença MIT. Veja LICENSE para mais detalhes.

🧑‍💻 Autor

Eduardo Sato
Desenvolvedor Full Stack | DevOps Enthusiast
