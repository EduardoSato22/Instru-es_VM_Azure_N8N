# Instruções para VM Azure + n8n

Este repositório reúne instruções práticas para criar uma VM no Microsoft Azure e instalar o n8n (automação de workflows) de forma confiável, segura e reproduzível.

Sumário
- Sobre
- Pré-requisitos
- Criar VM no Azure (Azure CLI)
- Acessos de rede (NSG / portas)
- Instalar Docker e Docker Compose
- Exemplo: docker-compose para n8n
- Variáveis de ambiente importantes
- Iniciar, parar e logs
- Segurança e boas práticas
- Backup e restauração
- Troubleshooting
- Como contribuir
- Licença

---

## Sobre
O n8n é uma ferramenta de automação (workflow automation) que pode ser executada em sua própria infraestrutura. Essas instruções focam em uma VM Linux (Ubuntu) no Azure com n8n em Docker Compose.

## Pré-requisitos
- Conta e assinatura do Azure ativa.
- Azure CLI instalado e autenticado (az login).
- Chave SSH pública (para acessar a VM).
- Conhecimento básico de terminal / SSH.
- (Opcional, recomendado) um nome de domínio para configurar TLS/HTTPS.

## 1) Criar VM no Azure (exemplo com Azure CLI)
Altere `MY_RG`, `MY_VM`, `MY_REGION` e `MY_SSH_KEY` conforme necessário.

```bash
# Variáveis de exemplo
MY_RG="rg-n8n"
MY_VM="vm-n8n"
MY_REGION="eastus"
MY_IMAGE="UbuntuLTS"
MY_SIZE="Standard_B1ms"
MY_ADMIN_USERNAME="azureuser"
MY_SSH_KEY="~/.ssh/id_rsa.pub"

# Criar resource group
az group create -n $MY_RG -l $MY_REGION

# Criar VM (abre porta 22)
az vm create \
  -g $MY_RG -n $MY_VM \
  --image $MY_IMAGE \
  --size $MY_SIZE \
  --admin-username $MY_ADMIN_USERNAME \
  --ssh-key-value "$(< $MY_SSH_KEY)" \
  --no-wait
```

Depois, abra as portas necessárias (ex.: 80, 443, 5678). Recomendo usar Network Security Group (NSG) para restringir IPs quando possível.

```bash
# Exemplo: abrir porta 22, 80, 443, 5678 (n8n default)
az vm open-port -g $MY_RG -n $MY_VM --port 22
az vm open-port -g $MY_RG -n $MY_VM --port 80
az vm open-port -g $MY_RG -n $MY_VM --port 443
az vm open-port -g $MY_RG -n $MY_VM --port 5678
```

Observação: para produção, prefira expor n8n por trás de um reverse proxy com TLS (ex.: Traefik, nginx + Certbot) e não diretamente na porta 5678 sem autenticação.

## 2) Instalar Docker + Docker Compose (Ubuntu)
Conecte-se por SSH e execute:

```bash
# Atualizar
sudo apt update && sudo apt upgrade -y

# Instalar dependências
sudo apt install -y ca-certificates curl gnupg lsb-release

# Adicionar repositório Docker
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Dar permissão ao usuário (opcional)
sudo usermod -aG docker $USER
# Faça logout/login ou use `newgrp docker` para aplicar sem logout
```

## 3) Exemplo docker-compose.yml para n8n
Arquivo de exemplo mínimo em `/home/azureuser/n8n/docker-compose.yml`. Ajuste secrets e volumes conforme necessário.

```yaml
version: "3.8"
services:
  n8n:
    image: n8nio/n8n:latest
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=meu-dominio.com       # opcional se usar proxy reverso
      - N8N_PORT=5678
      - NODE_ENV=production
      - GENERIC_TIMEZONE=America/Sao_Paulo
      - DB_TYPE=sqlite                 # para produção, prefira postgres
      - DB_SQLITE_VACUUM_ON_STARTUP=true
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=seu_usuario
      - N8N_BASIC_AUTH_PASSWORD=seu_password_seguro
      # - WEBHOOK_TUNNEL_URL=https://meu-dominio.com
    volumes:
      - ./n8n-data:/home/node/.n8n
```

Observações:
- Para produção, recomendo usar Postgres (DB_TYPE=postgresdb) e armazenar credenciais via secrets.
- Se usar um proxy reverso (nginx/Traefik), deixe N8N_HOST alinhado ao host público e exponha apenas internamente.

## 4) Variáveis de ambiente importantes
- N8N_BASIC_AUTH_ACTIVE=true: ativa autenticação básica.
- N8N_BASIC_AUTH_USER / N8N_BASIC_AUTH_PASSWORD: credenciais.
- WEBHOOK_TUNNEL_URL: URL pública para webhooks quando atrás de proxy.
- DB_TYPE / DB_*: configure seu banco (sqlite, postgres, etc).
- GENERIC_TIMEZONE: ajuste o fuso horário.

## 5) Iniciar / Parar / Logs
No diretório com docker-compose.yml:

```bash
# Iniciar
docker compose up -d

# Parar
docker compose down

# Ver logs
docker compose logs -f
```

## 6) Segurança e boas práticas
- Não exponha a porta 5678 diretamente sem autenticação e TLS.
- Use um reverse proxy (Traefik, nginx) com Let's Encrypt para TLS automático.
- Use banco de dados externo (Postgres) para resiliência e backups.
- Mantenha a máquina e containers atualizados.
- Restrinja acesso SSH por IP quando possível e use chaves SSH (desative senha).
- Considere habilitar firewall (ufw) na VM e limitar portas.

## 7) Backup e restauração
- Se usar SQLite: faça backup do arquivo `./n8n-data` (volume montado).
- Se usar Postgres: faça backup regular com `pg_dump`.
- Para exportar workflows: no n8n (UI) exporte/import ou use ferramentas CLI (depende da versão).
- Teste restaurações periodicamente em ambiente isolado.

## 8) Troubleshooting (problemas comuns)
- Problema: 502/timeout em webhooks — verifique reverse proxy e variável WEBHOOK_TUNNEL_URL.
- Problema: permissões de volume — verifique UID/GID do container e proprietário do diretório.
- Problema: banco corrompido (SQLite) — recupere a partir de backup.

## 9) Como contribuir
1. Faça um fork do repositório.
2. Abra uma branch com a mudança: `git checkout -b melhoria/readme`
3. Edite o README e submeta um Pull Request explicando as alterações.

## Licença
Adicione aqui a licença do projeto (por exemplo, MIT) ou mantenha conforme a política do repositório.

---

Se quiser, posso:
- adaptar esse README a um formato mais curto ou mais técnico;
- gerar um docker-compose com Postgres + Traefik já configurado;
- criar o arquivo README.md no repositório para você (posso fazer o push).

Diga qual opção prefere.
