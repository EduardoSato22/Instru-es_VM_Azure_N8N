Guia Completo: n8n na Nuvem com Azure, Docker, Nginx e IP Dinâmico na Cloudflare
Este guia detalha o processo completo para hospedar uma pilha de automação (n8n, Evolution API, Redis) em uma Máquina Virtual (VM) 'Free Tier' da Azure.

O objetivo é criar uma solução robusta e com custo-zero, superando o desafio do IP dinâmico da VM (que muda a cada reinicialização) usando um script que atualiza automaticamente o DNS na Cloudflare.

🚀 Tech Stack
Cloud: Microsoft Azure (VM Nível Gratuito)

SO: Ubuntu Server 22.04 LTS

Containerização: Docker & Docker Compose

Proxy Reverso: Nginx

Gerenciador de DNS: Cloudflare

Pilha de Aplicação: n8n, Evolution API, Redis

Automação: Bash Script, systemd

📋 Pré-requisitos
Antes de começar, você precisará de:

Uma conta no Microsoft Azure.

Uma conta na Cloudflare.

Um domínio (ex: seusite.com) já configurado e gerenciado pela Cloudflare.

Um cliente SSH (como Terminal, PowerShell ou PuTTY) para se conectar à VM.

⚙️ Passo 1: Criar e Configurar a VM no Azure
O primeiro passo é criar a máquina virtual que hospedará nossos serviços.

Criar a VM:

No portal do Azure, vá em "Máquinas Virtuais" > "Criar".

Importante: Em "Tamanho", escolha uma opção do "Nível Gratuito", como a B1s (1 vcpu, 1 GiB de memória).

Em "Imagem", selecione Ubuntu Server 22.04 LTS.

Em "Autenticação", escolha "Chave pública SSH" (recomendado) ou "Senha". Guarde bem essas credenciais.

Configurar o Firewall (Grupo de Segurança de Rede):

Durante a criação, na aba "Rede", certifique-se de permitir as portas de entrada:

22 (SSH) - Para você se conectar.

80 (HTTP) - Para o Nginx.

443 (HTTPS) - Para o Nginx (futuramente com SSL).

Configurar Desligamento Automático (Economia):

Após a VM ser criada, vá até ela e, no menu lateral, clique em "Desligamento automático".

Ative o desligamento para um horário em que você não usará a máquina (ex: 22:00).

Nota: Para ligar a VM agendadamente, você pode usar um "Aplicativo Lógico" ou "Automation Account" no Azure, que são um pouco mais complexos, ou simplesmente ligá-la manualmente pelo portal.

Configurar Backups Diários:

Na página da VM, no menu lateral, clique em "Backup".

Crie um novo "Cofre dos Serviços de Recuperação".

Defina uma política de backup (ex: "DailyBackup" às 02:00) e salve.

🛠️ Passo 2: Configurar o Ambiente do Servidor
Conecte-se à sua VM via SSH para instalar o software necessário.

Bash

# Substitua pelo seu usuário e IP da VM
ssh seu_usuario@IP_PUBLICO_DA_VM
Atualizar o Sistema:

Bash

sudo apt update && sudo apt upgrade -y
Instalar Docker:

Bash

# Baixa o script oficial de instalação
curl -fsSL https://get.docker.com -o get-docker.sh

# Executa o script
sudo sh get-docker.sh

# Adiciona seu usuário ao grupo do Docker (para não precisar usar 'sudo')
sudo usermod -aG docker $USER

# REINICIE SUA SESSÃO SSH para que a permissão acima tenha efeito.
# (Saia do SSH e conecte-se novamente)
Instalar Docker Compose:

Bash

sudo apt install docker-compose -y
Instalar Nginx e Ferramentas:

Vamos instalar o Nginx (que rodará fora do Docker) e as ferramentas jq (para ler JSON) e curl, que são necessárias para o script de IP.

Bash

sudo apt install nginx jq curl -y
Verificar Nginx:

Abra seu navegador e acesse http://IP_PUBLICO_DA_VM. Você deve ver a página "Welcome to nginx!".

☁️ Passo 3: Configurar a Cloudflare (DNS e API)
Precisamos de um registro DNS para apontar para a VM e uma chave de API para o script poder alterá-lo.

Crie um Registro DNS (Placeholder):

Na Cloudflare, vá para o seu domínio > "DNS".

Clique em "Adicionar registro".

Tipo: A

Nome: n8n (ou o subdomínio que você quer, ex: automacao)

Endereço IPv4: Coloque o IP público atual da sua VM Azure.

Proxy: Deixe DESATIVADO (Cinza). O script de atualização não funciona bem com o proxy laranja.

Salve.

Crie um Token de API da Cloudflare:

No menu da Cloudflare, vá em "Meus Tokens" > "Tokens de API" > "Criar Token".

Use o modelo "Editar DNS de zona".

Permissões:

Zone - Zone - Read

Zone - DNS - Edit

Recursos de Zona:

Incluir - Zona Específica - (Selecione seu domínio)

Continue e crie o token. Copie o token gerado e salve-o em um local seguro. Você não o verá novamente.

🏗️ Passo 4: Configurar o Projeto (Docker Compose e Nginx)
Agora vamos criar os arquivos de configuração para rodar nossa pilha.

Crie o Diretório do Projeto:

Bash

# Crie um diretório em sua home
mkdir ~/n8n-stack
cd ~/n8n-stack
Crie o docker-compose.yaml:

Este arquivo define quais containers queremos rodar.

Bash

nano docker-compose.yaml
Cole o conteúdo abaixo. Preste atenção nos comentários:

YAML

version: '3.7'

services:
  n8n:
    image: n8nio/n8n
    container_name: n8n
    restart: unless-stopped
    ports:
      # Expõe o n8n APENAS para o localhost (127.0.0.1)
      # O Nginx irá acessar o n8n por esta porta.
      - "127.0.0.1:5678:5678"
    environment:
      # Define o fuso horário para o n8n
      - GENERIC_TIMEZONE=America/Sao_Paulo
      # Descomente a linha abaixo se quiser usar o Redis para fila
      # - QUEUE_BULL_REDIS_URL=redis://redis:6379
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      - redis # Garante que o Redis inicie antes do n8n

  evolution_api:
    image: atende/evolution-api
    container_name: evolution_api
    restart: unless-stopped
    ports:
      # Expõe a API APENAS para o localhost
      - "127.0.0.1:8080:8080"
    volumes:
      - evolution_data:/evolution/instances
    depends_on:
      - redis

  redis:
    image: redis:alpine
    container_name: redis
    restart: unless-stopped
    volumes:
      - redis_data:/data

volumes:
  n8n_data:
  evolution_data:
  redis_data:
Configurar o Nginx como Proxy Reverso:

Precisamos dizer ao Nginx para redirecionar o tráfego do seu domínio (ex: n8n.seusite.com) para o container do n8n (que está em localhost:5678).

Crie um arquivo de configuração do Nginx:

Bash

sudo nano /etc/nginx/sites-available/n8n.conf
Cole o conteúdo abaixo, substituindo n8n.seusite.com pelo seu domínio:

Nginx

server {
    listen 80;
    server_name n8n.seusite.com; # <-- SUBSTITUA AQUI

    location / {
        proxy_pass http://127.0.0.1:5678; # Redireciona para o n8n
        proxy_set_header Connection '';
        proxy_http_version 1.1;
        proxy_set_header host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;

        # Configuração necessária para WebSockets (o n8n usa)
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
    }
}
Ativar o Site no Nginx:

O Nginx precisa saber que este novo arquivo de configuração deve ser usado.

Bash

# Cria um link simbólico do arquivo para a pasta 'sites-enabled'
sudo ln -s /etc/nginx/sites-available/n8n.conf /etc/nginx/sites-enabled/

# Remove o link de boas-vindas padrão
sudo rm /etc/nginx/sites-enabled/default

# Testa a configuração do Nginx
sudo nginx -t
# Se disser "syntax is ok" e "test is successful", você está pronto.

# Recarrega o Nginx
sudo systemctl reload nginx
🔄 Passo 5: Criar o Script de IP Dinâmico
Este é o script que mantém seu DNS atualizado.

Encontre seu ZONE_ID e RECORD_ID:

O script precisa saber qual zona e qual registro específico ele deve atualizar.

ZONE_ID: Na página principal do seu domínio na Cloudflare, role para baixo. Você verá o "ID da Zona" no lado direito. Copie-o.

RECORD_ID: Você precisa usar a API para encontrar o ID do registro n8n.seusite.com que você criou. Execute este comando no terminal da sua VM, substituindo as variáveis:

Bash

CF_TOKEN="SEU_TOKEN_DE_API"
ZONE_ID="SEU_ID_DE_ZONA"
RECORD_NAME="n8n.seusite.com"

curl -X GET "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records?name=$RECORD_NAME" \
     -H "Authorization: Bearer $CF_TOKEN" \
     -H "Content-Type: application/json" | jq
A saída será um JSON. Procure pelo "id" dentro do objeto do seu registro. Copie esse RECORD_ID.

Crie o Script:

No diretório ~/n8n-stack, crie o script:

Bash

nano update_ip.sh
Cole o script abaixo, substituindo os 4 placeholders no topo com seus dados:

Bash

#!/bin/bash

# --- CONFIGURE SUAS VARIÁVEIS AQUI ---
CF_TOKEN="SEU_TOKEN_DE_API_CLOUDFLARE"
ZONE_ID="SEU_ID_DE_ZONA"
RECORD_ID="SEU_ID_DO_REGISTRO_DNS"
RECORD_NAME="n8n.seusite.com" # O nome exato do seu registro DNS
# ------------------------------------

# Log para registrar a execução (opcional, mas útil)
LOG_FILE="/var/log/update_cloudflare_ip.log"
echo "--- $(date) ---" >> $LOG_FILE

# 1. Pega o IP público atual da VM
CURRENT_IP=$(curl -s http://ipv4.icanhazip.com)

if [ -z "$CURRENT_IP" ]; then
    echo "Erro: Não foi possível obter o IP público atual." >> $LOG_FILE
    exit 1
fi

# 2. Pega o IP salvo na Cloudflare
SAVED_IP=$(curl -s -X GET "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records/$RECORD_ID" \
     -H "Authorization: Bearer $CF_TOKEN" \
     -H "Content-Type: application/json" | jq -r '.result.content')

if [ -z "$SAVED_IP" ]; then
    echo "Erro: Não foi possível obter o IP salvo na Cloudflare." >> $LOG_FILE
    exit 1
fi

# 3. Compara os IPs
if [ "$CURRENT_IP" == "$SAVED_IP" ]; then
    echo "IPs são iguais ($CURRENT_IP). Nenhuma atualização necessária." >> $LOG_FILE
else
    echo "IP mudou. Atual: $CURRENT_IP. Salvo: $SAVED_IP. Atualizando..." >> $LOG_FILE

    # 4. Atualiza o IP na Cloudflare
    RESPONSE=$(curl -s -X PUT "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records/$RECORD_ID" \
         -H "Authorization: Bearer $CF_TOKEN" \
         -H "Content-Type: application/json" \
         --data '{"type":"A","name":"'"$RECORD_NAME"'","content":"'"$CURRENT_IP"'","ttl":1,"proxied":false}')

    if [ "$(echo $RESPONSE | jq -r '.success')" == "true" ]; then
        echo "Sucesso: IP atualizado para $CURRENT_IP" >> $LOG_FILE
    else
        echo "Erro ao atualizar IP: $RESPONSE" >> $LOG_FILE
    fi
fi
Torne o Script Executável:

Bash

chmod +x update_ip.sh
🔌 Passo 6: Automação da Inicialização com systemd
Queremos que tudo inicie automaticamente quando a VM ligar. Vamos usar o systemd para isso.

Crie o Serviço do Docker Compose:

Este serviço irá rodar o docker-compose up -d.

Bash

sudo nano /etc/systemd/system/n8n-stack.service
Cole o conteúdo, substituindo seu_usuario pelo seu nome de usuário na VM:

Ini, TOML

[Unit]
Description=N8N Stack Docker Compose Service
Requires=docker.service
After=docker.service network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
User=seu_usuario # <-- SUBSTITUA PELO SEU USUÁRIO
WorkingDirectory=/home/seu_usuario/n8n-stack # <-- SUBSTITUA PELO SEU USUÁRIO
ExecStart=/usr/bin/docker-compose up -d
ExecStop=/usr/bin/docker-compose down

[Install]
WantedBy=multi-user.target
Crie o Serviço do Script de IP:

Este serviço rodará o script de atualização de IP.

Bash

sudo nano /etc/systemd/system/update-ip.service
Cole o conteúdo, substituindo seu_usuario pelo seu nome de usuário na VM:

Ini, TOML

[Unit]
Description=Update Cloudflare Dynamic IP
After=network-online.target n8n-stack.service

[Service]
Type=oneshot
User=seu_usuario # <-- SUBSTITUA PELO SEU USUÁRIO
ExecStart=/home/seu_usuario/n8n-stack/update_ip.sh # <-- SUBSTITUA PELO SEU USUÁRIO

[Install]
WantedBy=multi-user.target
Ative os Serviços:

Isso fará com que eles sejam executados toda vez que a VM ligar.

Bash

sudo systemctl daemon-reload # Recarrega o systemd
sudo systemctl enable n8n-stack.service # Ativa o serviço do Docker
sudo systemctl enable update-ip.service # Ativa o serviço do Script
🏁 Finalização e Teste
Seu sistema está pronto!

Reinicie a VM:

Pelo portal do Azure ou com o comando sudo reboot.

Aguarde 2-3 minutos:

Dê tempo para a VM ligar, os serviços systemd rodarem, o Docker subir os containers e o script atualizar o IP da Cloudflare.

Acesse seu n8n:

Abra seu navegador e acesse http://n8n.seusite.com.

Você verá a tela de setup do n8n!

Seu n8n agora está rodando na nuvem, de graça, e seu domínio será atualizado automaticamente toda vez que a VM for reiniciada.