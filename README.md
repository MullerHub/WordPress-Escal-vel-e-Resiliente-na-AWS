# WordPress-Escalavel-e-Resiliente-na-AWS

Documentação do Projeto: WordPress Escalável e Resiliente na AWS
Data de Criação: 4 de Setembro de 2025

1. Visão Geral e Arquitetura

1.1. Objetivo

O objetivo deste projeto é implantar uma aplicação WordPress na nuvem da AWS de forma segura, escalável, resiliente e automatizada. A arquitetura é projetada para lidar com variações de tráfego, se recuperar de falhas e automatizar a configuração de novos servidores web.

1.2. Diagrama da Arquitetura

<img width="680" height="319" alt="Captura de Tela 2025-09-04 às 21 51 01" src="https://github.com/user-attachments/assets/6759767b-6158-4ba2-b5a1-54b0ad3dba51" />


O projeto segue o padrão de uma aplicação web de três camadas (web, dados, armazenamento), utilizando os seguintes componentes principais:

1.3. Componentes Principais

VPC (Virtual Private Cloud): Rede privada e isolada na AWS, dividida em sub-redes públicas e privadas para segurança.

Application Load Balancer (ALB): A porta de entrada pública. Distribui o tráfego de entrada entre os servidores web.

Auto Scaling Group (ASG): O cérebro da automação. Cria e destrói instâncias EC2 com base na demanda (uso de CPU).

EC2 (Elastic Compute Cloud): Servidores virtuais que rodam a aplicação WordPress dentro de contêineres Docker.

Launch Template: O "blueprint" ou "molde" que o ASG usa para criar novas instâncias EC2.

RDS (Relational Database Service): O banco de dados MySQL gerenciado, onde ficam armazenados posts, páginas, usuários, etc.

EFS (Elastic File System): O sistema de arquivos de rede compartilhado, onde ficam os arquivos do WordPress (uploads, temas, plugins).

Bastion Host: Uma instância EC2 na sub-rede pública que serve como um ponto de acesso seguro para gerenciar os recursos na sub-rede privada.

2. Guia de Implementação Passo a Passo

Etapa 1: Fundação da Rede (VPC)

Criar VPC: No console da VPC, use a opção "VPC e mais".

Configurações:

Nome: wordpress-vpc

Bloco CIDR: 10.0.0.0/16

Zonas de Disponibilidade (AZs): 2

Sub-redes Públicas: 2

Sub-redes Privadas: 2

Gateways NAT: Em 1 AZ

Etapa 2: Configuração de Segurança (Security Groups)

Crie 4 Security Groups (SGs) baseados em função:

alb-sg (Para o Load Balancer):

Regra de Entrada: Permitir HTTP (porta 80) de Qualquer Lugar (0.0.0.0/0).

bastion-sg (Para o Bastion Host):

Regra de Entrada: Permitir SSH (porta 22) de Meu IP.

webserver-sg (Para as Instâncias WordPress e EFS):

Regra de Entrada 1: Permitir HTTP (porta 80) da Origem alb-sg.

Regra de Entrada 2: Permitir SSH (porta 22) da Origem bastion-sg.

Regra de Entrada 3 (Crucial): Permitir NFS (porta 2049) da Origem webserver-sg (regra de auto-referência).

database-sg (Para o Banco de Dados RDS):

Regra de Entrada: Permitir MySQL (porta 3306) da Origem webserver-sg.

Etapa 3: Camada de Dados (EFS e RDS)

Criar EFS:

Crie um sistema de arquivos EFS na wordpress-vpc.

Na configuração de rede, crie "Pontos de Montagem" nas duas sub-redes privadas, associando o Security Group webserver-sg a eles.

Crie um "Ponto de Acesso" com User ID 33 e Group ID 33.

Criar RDS:

Crie uma instância de banco de dados MySQL.

Modelo: Produção para Multi-AZ (recomendado) ou Nível Gratuito para Single-AZ.

Coloque a instância na wordpress-vpc, nas sub-redes privadas.

Defina "Acesso Público" como Não.

Associe o Security Group database-sg.

Anote as credenciais: Endpoint, Nome do DB, Usuário e Senha.

Etapa 4: Acesso de Gerenciamento (Bastion Host)

Lance uma instância EC2 t2.micro com Amazon Linux 2023.

Coloque-a em uma sub-rede pública da wordpress-vpc.

Associe o Security Group bastion-sg.

Crie e associe um Elastic IP a ela.

Etapa 5: O Blueprint da Aplicação (Launch Template)

Crie um novo Launch Template (wordpress-launch-template).

AMI: Amazon Linux 2023.

Tipo de Instância: t2.micro.

Par de Chaves: A mesma chave usada no Bastion.

Configurações de Rede: Deixe a Sub-rede em branco. Associe o Security Group webserver-sg.

Detalhes Avançados:

Crie e associe um Perfil IAM (ec2-wordpress-role) com a permissão AmazonSSMManagedInstanceCore.

No campo User Data, cole o script abaixo, substituindo os placeholders:

Bash
#!/bin/bash
# Update and install necessary packages
yum update -y
yum install -y docker
yum install -y amazon-efs-utils

# Start and enable Docker service
systemctl start docker
systemctl enable docker
usermod -a -G docker ec2-user

# Add a custom prompt to the .bashrc for easier identification
echo "export PS1='\[\e[0;31m\][\u@\h] WORDPRESS> \[\e[0m\]'" >> /home/ec2-user/.bashrc

# --- PREENCHA SUAS INFORMAÇÕES AQUI ---
EFS_FILE_SYSTEM_ID="<SEU_EFS_ID>"
RDS_DB_HOST="<SEU_RDS_ENDPOINT>"
RDS_DB_NAME="<SEU_NOME_DO_DB>"
RDS_DB_USER="<SEU_USUARIO_DO_DB>"
RDS_DB_PASSWORD="<SUA_SENHA_DO_DB>"
# --- FIM DA SEÇÃO DE INFORMAÇÕES ---

# Create a directory to mount EFS
mkdir -p /var/www/html

# Mount EFS file system
mount -t efs -o tls ${EFS_FILE_SYSTEM_ID}:/ /var/www/html

# Create docker-compose.yml file
cat << EOF > /home/ec2-user/docker-compose.yml
version: '3'
services:
  wordpress:
    image: wordpress:latest
    container_name: wordpress
    restart: always
    ports:
      - "80:80"
    environment:
      WORDPRESS_DB_HOST: ${RDS_DB_HOST}
      WORDPRESS_DB_NAME: ${RDS_DB_NAME}
      WORDPRESS_DB_USER: ${RDS_DB_USER}
      WORDPRESS_DB_PASSWORD: ${RDS_DB_PASSWORD}
    volumes:
      - /var/www/html:/var/www/html
EOF

# Change ownership of the docker-compose file
chown ec2-user:ec2-user /home/ec2-user/docker-compose.yml

# Install docker-compose
curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# Start WordPress container using docker-compose
cd /home/ec2-user/
docker-compose up -d
Adicione as Tags corporativas obrigatórias ao template.

Etapa 6: A Porta de Entrada (Application Load Balancer)

Crie um Target Group (wordpress-tg) para instâncias na porta HTTP 80.

Crie um Application Load Balancer (wordpress-alb) do tipo "Internet-facing".

Coloque-o nas duas sub-redes públicas.

Associe o Security Group alb-sg.

Configure o "Listener" da porta 80 para encaminhar o tráfego para o wordpress-tg.

Etapa 7: O Motor da Automação (Auto Scaling Group)

Crie um Auto Scaling Group (wordpress-asg).

Associe-o ao wordpress-launch-template (versão mais recente).

Configure-o para usar as duas sub-redes privadas.

Anexe-o ao Target Group wordpress-tg e habilite a verificação de saúde ELB.

Defina a capacidade: Desejada 2, Mínima 1, Máxima 4.

Defina a política de escalabilidade: Utilização média da CPU com alvo em 50%.

3. Lições Aprendidas e Solução de Problemas

Problema 1: Erro de Permissão (You are not authorized...) na Criação do ASG.

Causa Raiz: Políticas de IAM corporativas que exigem tags em recursos.

Solução: Adicionar as tags (Name, CostCenter, Project) ao Launch Template (para Instâncias e Volumes) e também às Sub-redes Privadas.

Problema 2: Erro de Operation timed out ao Conectar via SSH ao Bastion.

Causa Raiz: O IP público do usuário mudou, ou a rede do usuário usa CGNAT, fazendo com que o IP visto pela AWS seja diferente do IP visto pelo usuário.

Solução: Conectar-se temporariamente com a regra do SG aberta para 0.0.0.0/0. De dentro da instância, executar o comando who para descobrir o IP de origem real. Atualizar a regra do SG bastion-sg com este IP exato.

Problema 3: Loop de Instâncias "Não Saudáveis" (Unhealthy).

Causa Raiz: Falha na montagem do EFS (mount timeout) devido a um erro de conectividade de rede.

Diagnóstico: Análise do log /var/log/cloud-init-output.log.

Solução: O erro estava na associação de Security Groups. A instância EC2 estava sendo criada com o SG default, enquanto o EFS esperava uma conexão do webserver-sg. A correção foi modificar o Launch Template para que ele atribuísse o webserver-sg corretamente às instâncias no momento da criação.

4. Próximos Passos e Melhorias

DNS e Domínio: Usar o Amazon Route 53 para apontar um domínio personalizado para o DNS do Application Load Balancer.

HTTPS: Usar o AWS Certificate Manager (ACM) para gerar um certificado SSL gratuito e associá-lo ao ALB, criando um Listener na porta 443.

Segurança de Credenciais: Remover a senha do banco de dados do script user-data e armazená-la de forma segura no AWS Secrets Manager ou SSM Parameter Store. O IAM Role da instância seria usado para dar permissão de leitura.

Automação da Infraestrutura (IaC): Traduzir todo este processo manual para um template do AWS CloudFormation ou Terraform, permitindo que toda a arquitetura seja criada e destruída com um único comando.

