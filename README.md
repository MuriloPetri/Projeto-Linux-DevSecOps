# Projeto-Linux-DevSecOps
## 🧱 Etapa 0 – Criação da Infraestrutura AWS (VPC, Sub-redes e Roteamento)

Antes de criar a instância EC2, foi configurada uma infraestrutura de rede personalizada na AWS:

### 🔸 VPC

- **Nome**: `projeto-vpc`
- **ID**: `vpc-08705c8b08f62169b`

### 🔸 Sub-redes 
-- Sub-redes Públicas
- `projeto-subnet-public1-us-east-2a`- – Zona de disponibilidade: us-east-2a
- `projeto-subnet-public2-us-east-2b`- – Zona de disponibilidade: us-east-2b

- `projeto-subnet-private2-us-east-2b`– Zona de disponibilidade: us-east-2b
- `projeto-subnet-private2-us-east-2a`– Zona de disponibilidade: us-east-2a

### 🔸 Internet Gateway (IGW)

- Criado um **Internet Gateway**
- Associado à VPC `projeto-vpc`

### 🔸 Tabela de Rotas

- Tabela de rotas `projeto-rtb-public` configurada com:
  - Rota `0.0.0.0/0` apontando para o Internet Gateway
- Associada às sub-redes públicas

### 🔸 Grupo de Segurança (Security Group)

- **Porta 22 (SSH)**: liberada temporariamente para `0.0.0.0/0` (apenas para testes)
- **Porta 80 (HTTP)**: liberada para `0.0.0.0/0`

### 🔸 Par de Chaves

- Nome do par de chaves: `.pem` (CRIE O SEU NOME DO ARQUIVO)
- Usado para acesso SSH via terminal WSL (UBUNTU 24.04.1 LTS)



# ETAPA 2 -- INSTÂNCIA EC2

# 🔧 Monitoramento de Servidor EC2 com Nginx + Alerta via Telegram

Este projeto implementa um servidor web com monitoramento automático. Em caso de falha, um alerta é enviado por um bot através do Telegram.

## ✅ Etapa 1 – Instância EC2 

1. Criei uma instância EC2.
2. Usando Tags fornecidas nas dailys (Name - PB - JUN 2025) (CoastCenter - C092000024) (Project - PB - JUN 2025)
3. Gerei uma chave `.pem` para conexão SSH.
4. Liberei a porta **80 (HTTP)** no **Security Group**.
   ![Image](https://github.com/user-attachments/assets/e479c968-368d-4351-83cc-3040910601c9)

Conecte-se à instância via WSL (UBUNTU 24.04.1 LTS), através do comando abaixo:
```bash
ssh -i "(SUA-CHAVE).pem" ec2-user@ec2-user@<IP-Público>
````
![Image](https://github.com/user-attachments/assets/e41a2ab7-6668-4668-870b-d6a145ab0582)

## 🌐 Etapa 3 – Instalação e Configuração do Nginx

Instalar o Nginx dentro da instância EC2. Ele que será responsável por servir a página HTML de status monitorado.

### 🔧 Passo a passo – Instalando o Nginx

```bash
sudo yum update -y
sudo amazon-linux-extras enable nginx1
sudo yum install -y nginx
````

## Iniciando o Nginx e habilitando
````bash
sudo systemctl start nginx
sudo systemctl enable nginx
````


## 🛠️ Etapa 3 – Página HTML Personalizada

Substituímos a página padrão do Nginx por uma página simples que informa o status do servidor.

## Etapa 4 - Criação de Script de Monitoramento (Telegram)

Criação do arquivo
````bash
nano ~/monitor.sh
````

Script de Monitorar
````bash
#!/bin/bash

# Carregar variáveis do arquivo .env (TOKEN e CHAT_ID)
source ~/.env

# URL a monitorar (servidor Nginx local)
URL="http://localhost"

# Caminho do arquivo de log
LOG_FILE="/var/log/meu_script.log"

# Função para enviar alerta via Telegram
send_alert() {
    local MESSAGE=$1
    curl -s -X POST "https://api.telegram.org/bot${TOKEN}/sendMessage" \
        -d chat_id="${CHAT_ID}" \
        -d text="${MESSAGE}" > /dev/null
}

# Loop infinito
while true; do
    HTTP_STATUS=$(curl -o /dev/null -s -w "%{http_code}" "$URL")
    NGINX_STATUS=$(systemctl is-active nginx)
    TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

    if [ "$HTTP_STATUS" -ne 200 ] || [ "$NGINX_STATUS" != "active" ]; then
        MESSAGE="⚠️ ALERTA ($TIMESTAMP): Problema detectado!
Status HTTP: $HTTP_STATUS
Status do NGINX: $NGINX_STATUS"
    else
        MESSAGE="✅ OK ($TIMESTAMP): Servidor NGINX funcionando normalmente.
Status HTTP: $HTTP_STATUS
Status do NGINX: $NGINX_STATUS"
    fi

    echo "$MESSAGE" >> "$LOG_FILE"
    send_alert "$MESSAGE"

    # Aguarda 30 segundos antes de repetir
    sleep 30
done

````

Logo em seguida, permitir a execução do script
````bash
chmod +x ~/monitor.sh
````

## ETAPA 5 - Criação do arquivo .env (Esconder dados sensíveis)

````bash
nano ~/.env
````

````ini
TOKEN=seu_token_do_bot_telegram
CHAT_ID=seu_chat_id_do_telegram
````
Consiga o seu token e o id do seu chat através de dois bots.
BotFather (Conseguir usar a API, criar seu bot, e o token para receber a mensagem de monitoramento)
Userinfobot (Conseguir o id do seu chat do Telegram)

- Após isso proteja os dados sensíveis, para que apenas voce tenha acesso:
  ````bash
  chmod 600 ~/.env
  ````

## ETAPA 6 - Criação do arquivo para armazenar log.

````bash
sudo touch /var/log/meu_script.log
````

````bash
sudo chown ec2-user:ec2-user /var/log/meu_script.log
````

## ETAPA 7 - Instalar, habilitar e configurar o serviço Cronie

Instalando o Cronie
````bash
sudo yum install -y cronie
sudo systemctl enable crond
sudo systemctl start crond
````

Edite o arquivo para receber o comando que voce deseja:
````bash
EDITOR=nano crontab -e
````

Neste repositório, estou usando a cada 1 minuto, ele me retorna mensagem de erro, logo:

````bash
* * * * * /home/ec2-user/monitor.sh >/dev/null 2>&1
````

## ETAPA 8 - Testando 

Conecte-se à instância via WSL (UBUNTU 24.04.1 LTS), através do comando abaixo:
```bash
ssh -i "(SUA-CHAVE).pem" ec2-user@ec2-user@<IP-Público>
````

Para o servidor iniciar, usa:
````bash
sudo systemctl start nginx
````

Você receberá essa mensagem quando reiniciar o nginx
![Image](https://github.com/user-attachments/assets/4d4d71df-224e-4584-8ebf-33a6c27f419e)


Para testar o erro, e o bot avisar você no seu chat do telegram, usamos o comando do Nginx:

````bash
sudo systemctl stop nginx
````
Você receberá essa mensagem quando parar o nginx 
![Image](https://github.com/user-attachments/assets/93a142f8-4bc8-4c87-bc64-22bf667b0412)










