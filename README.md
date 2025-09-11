# Projeto WordPress Escalável e Resiliente na AWS

## 1. Resumo do Projeto

1.2. Diagrama da Arquitetura

<img width="680" height="319" alt="Captura de Tela 2025-09-04 às 21 51 01" src="https://github.com/user-attachments/assets/6759767b-6158-4ba2-b5a1-54b0ad3dba51" />
<br> <br>

Este documento detalha a arquitetura e o processo de implantação de uma infraestrutura completa para hospedar um site WordPress na nuvem AWS. A solução foi projetada para ser altamente disponível, escalável e segura, utilizando as melhores práticas de nuvem e automação com Infraestrutura como Código (IaC) através do AWS CloudFormation.

## 2. Arquitetura da Solução

A arquitetura é desacoplada em múltiplas camadas (rede, dados, aplicação) para garantir segurança e escalabilidade independentes.

### 2.1. Diagrama do Fluxo de Tráfego

`Visitante (Internet)` -> `Application Load Balancer (ALB)` -> `Instâncias EC2 (WordPress em Docker)` -> `Amazon RDS (Banco de Dados)` & `Amazon EFS (Arquivos)`

### 2.2. Componentes Principais

* **VPC (Virtual Private Cloud):**
    * **Função:** Cria uma rede privada e isolada na AWS para hospedar todos os recursos.
    * **Detalhes:** CIDR `10.0.0.0/16`, dividida em 4 sub-redes para alta disponibilidade em duas Zonas de Disponibilidade (AZs).
    * **Sub-redes Públicas:** Duas sub-redes (`10.0.1.0/24`, `10.0.2.0/24`) para recursos que precisam de acesso direto à internet, como o Application Load Balancer.
    * **Sub-redes Privadas:** Duas sub-redes (`10.0.10.0/24`, `10.0.11.0/24`) para proteger os recursos principais (Instâncias EC2 e Banco de Dados RDS), que não devem ser acessíveis publicamente.
    * **Internet Gateway (IGW):** Permite a comunicação entre a VPC e a internet.
    * **NAT Gateway:** Permite que as instâncias nas sub-redes privadas iniciem conexões com a internet (para atualizações de software e download de imagens Docker) sem serem diretamente acessíveis.

* **Amazon RDS (Relational Database Service):**
    * **Função:** Prover um banco de dados MySQL gerenciado, seguro e escalável para armazenar todos os dados do WordPress (posts, páginas, usuários, configurações).
    * **Detalhes:** Instância `db.t3.micro` rodando em sub-redes privadas e protegida por um Security Group que só permite conexões a partir das instâncias EC2.

* **Amazon EFS (Elastic File System):**
    * **Função:** Prover um sistema de arquivos de rede (NFS) compartilhado e elástico. É usado para armazenar todos os arquivos de mídia do WordPress (imagens, vídeos, uploads), temas e plugins.
    * **Detalhes:** O EFS é montado no diretório `/var/www/html` de todas as instâncias EC2, garantindo que os arquivos sejam consistentes e persistentes, mesmo que instâncias sejam adicionadas ou removidas.

* **EC2 Auto Scaling Group & Launch Template:**
    * **Função:** Gerenciar automaticamente o número de instâncias EC2 para garantir a disponibilidade e escalar de acordo com a demanda.
    * **Launch Template:** Define a "receita" para novas instâncias, incluindo o tipo de instância (`t2.micro`), a AMI, o par de chaves, o Security Group e o script `UserData`.
    * **UserData Script:** Script de inicialização que automatiza a configuração de cada nova instância: instala e inicia o Docker, monta o EFS, cria o `docker-compose.yml` com as credenciais do banco e inicia o container do WordPress.
    * **Escalabilidade:** A configuração inicial mantém um mínimo de 1 e um máximo de 4 instâncias, com 2 instâncias desejadas.

* **Application Load Balancer (ALB):**
    * **Função:** Distribuir o tráfego de entrada dos visitantes de forma equilibrada entre as instâncias EC2 saudáveis.
    * **Detalhes:** O ALB fica nas sub-redes públicas. Ele realiza verificações de saúde (`Health Checks`) contínuas nas instâncias EC2 para garantir que o tráfego só seja enviado para servidores que estão funcionando corretamente.

* **Bastion Host:**
    * **Função:** Servir como um "ponto de pulo" seguro para acesso administrativo (via SSH) às instâncias EC2 que estão na rede privada. (nosso projeto nao esta funcionando por erro interno no "meu" endereco ipv4, mas pode ser que funciona normalmente em testes em sua maquina, sera solicitado o seu ip na importacao do cloudformation para essa conexao, mas iremos usar o Gerenciador de sessões para logar no bastion via console)
    * **Detalhes:** É uma pequena instância EC2 na sub-rede pública com um Security Group que restringe o acesso SSH apenas a um IP específico.

## 3. Infraestrutura como Código (CloudFormation)

Toda a infraestrutura descrita acima é definida no arquivo `template.yaml`, permitindo a criação e exclusão de todo o ambiente de forma automatizada e consistente.

### 3.1. Parâmetros de Entrada

* `DBPassword`: A senha para o usuário master do banco de dados RDS.
* `KeyName`: O nome de um par de chaves EC2 existente na conta para permitir o acesso SSH.
* `YourIPForSSH`: O endereço IP (em formato CIDR) a partir do qual o acesso ao Bastion Host será permitido.

### 3.2. Saídas (Outputs) da Stack

* `WebsiteURL`: O endereço DNS público do Application Load Balancer, usado para acessar o site WordPress.
* `BastionEIP`: O IP público (Elastic IP) do Bastion Host, usado para o acesso administrativo via SSH.

## 4. Guia de Implantação e Gerenciamento

* **Pré-requisitos**
  * Conta na AWS com as devidas permissões.
  * Um par de chaves EC2 já criado na região de implantação.

* **Passos para Implantação**
  * Entre na sua conta AWS, procure pelo serviço CloudFormation e clique em "Criar pilha".
  * Na seção "Especificar modelo", selecione "Modelo está pronto" e depois "Carregar um arquivo de modelo".
  * Clique em "Escolher arquivo", selecione o arquivo wordpress-full-stack.yaml e clique em "Avançar".
  * Dê um nome para a sua pilha e insira os valores para os parâmetros DBPassword, KeyName e YourIPForSSH.
  * Avance para a próxima tela, revise as configurações, e na parte inferior, selecione "Eu reconheço que o AWS CloudFormation  pode criar recursos do IAM com nomes lógicos personalizados.".
  * Clique em "Criar pilha" para iniciar a implantação.

### 4.3. Acesso Administrativo (SSH)

Para acessar uma instância EC2 privada:
1.  Encontre o IP público do Bastion Host na saída da stack.
2.  Encontre o IP privado de uma das instâncias EC2 no console.
3.  Use o seguinte comando no seu terminal:

    ```bash
    ssh -J ec2-user@IP_DO_BASTION ec2-user@IP_PRIVADO_DA_EC2
    ```

    -J significa que o comando acima ira fazer um jumping, onde ele ira conectar no bastion e ira "pular" direto pra ec2 privada.

### 4.4. Exclusão da Stack

Para remover todos os recursos e evitar custos, entre na sua conta da aws > cloudformation, selecione a pilha e clique em "Excluir pilha".

Template do cloudformation
### wordpress-full-stack.yaml

```
AWSTemplateFormatVersion: '2010-09-09'
Description: >
  Stack completa para implantar um WordPress escalavel e resiliente na AWS, incluindo
  VPC, Sub-redes, Gateways, Security Groups, Bastion Host, EFS, RDS, ALB e Auto Scaling Group.

Parameters:
  DBPassword:
    Type: String
    Description: Senha para o banco de dados RDS. Minimo 8 caracteres alfanumericos.
    NoEcho: true
    MinLength: 8
    AllowedPattern: "[a-zA-Z0-9]+"
    ConstraintDescription: Deve conter apenas letras e números.

  KeyName:
    Type: AWS::EC2::KeyPair::KeyName
    Description: Nome de um par de chaves EC2 existente para permitir acesso SSH.

  YourIPForSSH:
    Type: String
    Description: Seu endereço IP público para acesso SSH ao Bastion Host.
    AllowedPattern: '(\d{1,3})\.(\d{1,3})\.(\d{1,3})\.(\d{1,3})/(\d{1,2})'

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - { Key: Name, Value: "PB - JUL 2025" }
        - { Key: CostCenter, Value: "CO92000024" }
        - { Key: Project, Value: "PB - JUL 2025" }

  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - { Key: Name, Value: "PB - JUL 2025" }
        - { Key: CostCenter, Value: "CO92000024" }
        - { Key: Project, Value: "PB - JUL 2025" }

  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties: { VpcId: !Ref VPC, InternetGatewayId: !Ref InternetGateway }

  PublicSubnetA:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: !Select [0, !GetAZs '']
      MapPublicIpOnLaunch: true
      Tags:
        - { Key: Name, Value: "PB - JUL 2025" }
        - { Key: CostCenter, Value: "CO92000024" }
        - { Key: Project, Value: "PB - JUL 2025" }

  PublicSubnetB:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.2.0/24
      AvailabilityZone: !Select [1, !GetAZs '']
      MapPublicIpOnLaunch: true
      Tags:
        - { Key: Name, Value: "PB - JUL 2025" }
        - { Key: CostCenter, Value: "CO92000024" }
        - { Key: Project, Value: "PB - JUL 2025" }

  PrivateSubnetA:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.10.0/24
      AvailabilityZone: !Select [0, !GetAZs '']
      Tags:
        - { Key: Name, Value: "PB - JUL 2025" }
        - { Key: CostCenter, Value: "CO92000024" }
        - { Key: Project, Value: "PB - JUL 2025" }

  PrivateSubnetB:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.11.0/24
      AvailabilityZone: !Select [1, !GetAZs '']
      Tags:
        - { Key: Name, Value: "PB - JUL 2025" }
        - { Key: CostCenter, Value: "CO92000024" }
        - { Key: Project, Value: "PB - JUL 2025" }

  NatGatewayEIP:
    Type: AWS::EC2::EIP
    Properties:
      Domain: vpc
      Tags:
        - { Key: Name, Value: "PB - JUL 2025" }
        - { Key: CostCenter, Value: "CO92000024" }
        - { Key: Project, Value: "PB - JUL 2025" }

  NatGateway:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatGatewayEIP.AllocationId
      SubnetId: !Ref PublicSubnetA
      Tags:
        - { Key: Name, Value: "PB - JUL 2025" }
        - { Key: CostCenter, Value: "CO92000024" }
        - { Key: Project, Value: "PB - JUL 2025" }

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - { Key: Name, Value: "PB - JUL 2025" }
        - { Key: CostCenter, Value: "CO92000024" }
        - { Key: Project, Value: "PB - JUL 2025" }

  PublicRoute: { Type: AWS::EC2::Route, DependsOn: AttachGateway, Properties: { RouteTableId: !Ref PublicRouteTable, DestinationCidrBlock: 0.0.0.0/0, GatewayId: !Ref InternetGateway } }
  AssociatePublicRouteTableA: { Type: AWS::EC2::SubnetRouteTableAssociation, Properties: { SubnetId: !Ref PublicSubnetA, RouteTableId: !Ref PublicRouteTable } }
  AssociatePublicRouteTableB: { Type: AWS::EC2::SubnetRouteTableAssociation, Properties: { SubnetId: !Ref PublicSubnetB, RouteTableId: !Ref PublicRouteTable } }

  PrivateRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - { Key: Name, Value: "PB - JUL 2025" }
        - { Key: CostCenter, Value: "CO92000024" }
        - { Key: Project, Value: "PB - JUL 2025" }

  PrivateRoute: { Type: AWS::EC2::Route, Properties: { RouteTableId: !Ref PrivateRouteTable, DestinationCidrBlock: 0.0.0.0/0, NatGatewayId: !Ref NatGateway } }
  AssociatePrivateRouteTableA: { Type: AWS::EC2::SubnetRouteTableAssociation, Properties: { SubnetId: !Ref PrivateSubnetA, RouteTableId: !Ref PrivateRouteTable } }
  AssociatePrivateRouteTableB: { Type: AWS::EC2::SubnetRouteTableAssociation, Properties: { SubnetId: !Ref PrivateSubnetB, RouteTableId: !Ref PrivateRouteTable } }

  ALBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: "alb-sg"
      GroupDescription: "Permite trafego da internet para o ALB"
      VpcId: !Ref VPC
      Tags:
        - { Key: Name, Value: "PB - JUL 2025" }
        - { Key: CostCenter, Value: "CO92000024" }
        - { Key: Project, Value: "PB - JUL 2025" }

  BastionSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: "bastion-sg"
      GroupDescription: "Permite acesso SSH ao Bastion Host"
      VpcId: !Ref VPC
      Tags:
        - { Key: Name, Value: "PB - JUL 2025" }
        - { Key: CostCenter, Value: "CO92000024" }
        - { Key: Project, Value: "PB - JUL 2025" }
          
  WebServerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: "webserver-sg"
      GroupDescription: "Regras para os servidores WordPress"
      VpcId: !Ref VPC
      Tags:
        - { Key: Name, Value: "PB - JUL 2025" }
        - { Key: CostCenter, Value: "CO92000024" }
        - { Key: Project, Value: "PB - JUL 2025" }

  DatabaseSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: "database-sg"
      GroupDescription: "Permite acesso ao banco de dados"
      VpcId: !Ref VPC
      Tags:
        - { Key: Name, Value: "PB - JUL 2025" }
        - { Key: CostCenter, Value: "CO92000024" }
        - { Key: Project, Value: "PB - JUL 2025" }

  ALBIngressRule: { Type: AWS::EC2::SecurityGroupIngress, Properties: { GroupId: !Ref ALBSecurityGroup, IpProtocol: tcp, FromPort: 80, ToPort: 80, CidrIp: 0.0.0.0/0 } }
  BastionIngressRule: { Type: AWS::EC2::SecurityGroupIngress, Properties: { GroupId: !Ref BastionSecurityGroup, IpProtocol: tcp, FromPort: 22, ToPort: 22, CidrIp: !Ref YourIPForSSH } }
  WebServerIngressFromALB: { Type: AWS::EC2::SecurityGroupIngress, Properties: { GroupId: !Ref WebServerSecurityGroup, IpProtocol: tcp, FromPort: 80, ToPort: 80, SourceSecurityGroupId: !Ref ALBSecurityGroup } }
  WebServerIngressFromBastion: { Type: AWS::EC2::SecurityGroupIngress, Properties: { GroupId: !Ref WebServerSecurityGroup, IpProtocol: tcp, FromPort: 22, ToPort: 22, SourceSecurityGroupId: !Ref BastionSecurityGroup } }
  WebServerIngressFromSelf: { Type: AWS::EC2::SecurityGroupIngress, Properties: { GroupId: !Ref WebServerSecurityGroup, IpProtocol: tcp, FromPort: 2049, ToPort: 2049, SourceSecurityGroupId: !Ref WebServerSecurityGroup } }
  DatabaseIngressFromWebServer: { Type: AWS::EC2::SecurityGroupIngress, Properties: { GroupId: !Ref DatabaseSecurityGroup, IpProtocol: tcp, FromPort: 3306, ToPort: 3306, SourceSecurityGroupId: !Ref WebServerSecurityGroup } }

  EFSFileSystem:
    Type: AWS::EFS::FileSystem
    DeletionPolicy: Delete
    Properties:
      Encrypted: true
      FileSystemTags:
        - { Key: Name, Value: "PB - JUL 2025" }
        - { Key: CostCenter, Value: "CO92000024" }
        - { Key: Project, Value: "PB - JUL 2025" }

  EFSMountTargetA:
    Type: AWS::EFS::MountTarget
    Properties:
      FileSystemId: !Ref EFSFileSystem
      SubnetId: !Ref PrivateSubnetA
      SecurityGroups: [ !Ref WebServerSecurityGroup ]

  EFSMountTargetB:
    Type: AWS::EFS::MountTarget
    Properties:
      FileSystemId: !Ref EFSFileSystem
      SubnetId: !Ref PrivateSubnetB
      SecurityGroups: [ !Ref WebServerSecurityGroup ]

  DBSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: "Subnet group for RDS"
      SubnetIds: [ !Ref PrivateSubnetA, !Ref PrivateSubnetB ]

  RDSInstance:
    Type: AWS::RDS::DBInstance
    Properties:
      DBName: wordpressdb
      Engine: mysql
      EngineVersion: '8.0'
      MasterUsername: admin
      MasterUserPassword: !Ref DBPassword
      DBInstanceClass: db.t3.micro
      AllocatedStorage: '20'
      DBSubnetGroupName: !Ref DBSubnetGroup
      VPCSecurityGroups: [ !Ref DatabaseSecurityGroup ]
      MultiAZ: false
      Tags:
        - { Key: Name, Value: "PB - JUL 2025" }
        - { Key: CostCenter, Value: "CO92000024" }
        - { Key: Project, Value: "PB - JUL 2025" }

  BastionHost:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t2.micro
      ImageId: ami-00ca32bbc84273381
      KeyName: !Ref KeyName
      SubnetId: !Ref PublicSubnetA
      SecurityGroupIds: [ !Ref BastionSecurityGroup ]
      IamInstanceProfile: !Ref EC2InstanceProfile
      BlockDeviceMappings:
        - DeviceName: /dev/xvda
          Ebs:
            VolumeSize: 8
            VolumeType: gp3
            Encrypted: true
      PropagateTagsToVolumeOnCreation: true
      Tags:
        - { Key: Name, Value: "PB - JUL 2025" }
        - { Key: CostCenter, Value: "CO92000024" }
        - { Key: Project, Value: "PB - JUL 2025" }
  
  BastionEIP:
    Type: AWS::EC2::EIP
    Properties:
      InstanceId: !Ref BastionHost
      Tags:
        - { Key: Name, Value: "PB - JUL 2025" }
        - { Key: CostCenter, Value: "CO92000024" }
        - { Key: Project, Value: "PB - JUL 2025" }

  AppLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: wordpress-alb
      Scheme: internet-facing
      Subnets: [ !Ref PublicSubnetA, !Ref PublicSubnetB ]
      SecurityGroups: [ !Ref ALBSecurityGroup ]
      Tags:
        - { Key: Name, Value: "PB - JUL 2025" }
        - { Key: CostCenter, Value: "CO92000024" }
        - { Key: Project, Value: "PB - JUL 2025" }

  ALBTargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Name: wordpress-tg
      VpcId: !Ref VPC
      Protocol: HTTP
      Port: 80
      HealthCheckPath: /
      TargetType: instance
      Tags:
        - { Key: Name, Value: "PB - JUL 2025" }
        - { Key: CostCenter, Value: "CO92000024" }
        - { Key: Project, Value: "PB - JUL 2025" }

  ALBListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref AppLoadBalancer
      Protocol: HTTP
      Port: 80
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref ALBTargetGroup
  
  EC2IAMRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement: [{ Effect: Allow, Principal: { Service: ec2.amazonaws.com }, Action: sts:AssumeRole }]
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

  EC2InstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Roles: [ !Ref EC2IAMRole ]

  AppLaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateName: wordpress-launch-template
      LaunchTemplateData:
        ImageId: ami-00ca32bbc84273381
        InstanceType: t2.micro
        KeyName: !Ref KeyName
        IamInstanceProfile:
          Arn: !GetAtt EC2InstanceProfile.Arn
        SecurityGroupIds: [ !Ref WebServerSecurityGroup ]
        BlockDeviceMappings:
          - DeviceName: /dev/xvda
            Ebs:
              VolumeSize: 8
              VolumeType: gp3
              Encrypted: true
        TagSpecifications:
          - ResourceType: instance
            Tags:
              - { Key: Name, Value: "PB - JUL 2025" }
              - { Key: CostCenter, Value: "CO92000024" }
              - { Key: Project, Value: "PB - JUL 2025" }
          - ResourceType: volume
            Tags:
              - { Key: Name, Value: "PB - JUL 2025" }
              - { Key: CostCenter, Value: "CO92000024" }
              - { Key: Project, Value: "PB - JUL 2025" }
        UserData:
          Fn::Base64: !Sub |
            exec > >(tee /var/log/user-data.log|logger -t user-data -s 2>/dev/console) 2>&1

            dnf update -y
            dnf install -y docker amazon-efs-utils mariadb105-server

            systemctl start docker
            systemctl enable docker
            usermod -a -G docker ec2-user
            curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
            chmod +x /usr/local/bin/docker-compose

            mkdir -p /var/www/html
            echo "${EFSFileSystem}:/ /var/www/html efs _netdev,tls 0 0" >> /etc/fstab
            mount -a -t efs defaults
            chown ec2-user:ec2-user /var/www/html

            cat << EOF > /home/ec2-user/docker-compose.yml
            version: '3.8'
            services:
              wordpress:
                image: wordpress:latest
                container_name: wordpress
                restart: always
                ports: ["80:80"]
                environment:
                  WORDPRESS_DB_HOST: ${RDSInstance.Endpoint.Address}
                  WORDPRESS_DB_NAME: wordpressdb
                  WORDPRESS_DB_USER: admin
                  WORDPRESS_DB_PASSWORD: ${DBPassword}
                volumes:
                   - "/var/www/html:/var/www/html" 
            EOF
            chown ec2-user:ec2-user /home/ec2-user/docker-compose.yml
            sudo -u ec2-user /usr/local/bin/docker-compose -f /home/ec2-user/docker-compose.yml up -d

  AppAutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    DependsOn: RDSInstance
    Properties:
      AutoScalingGroupName: wordpress-asg
      VPCZoneIdentifier: [ !Ref PrivateSubnetA, !Ref PrivateSubnetB ]
      LaunchTemplate:
        LaunchTemplateId: !Ref AppLaunchTemplate
        Version: !GetAtt AppLaunchTemplate.LatestVersionNumber
      MinSize: '1'
      MaxSize: '4'
      DesiredCapacity: '2'
      TargetGroupARNs: [ !Ref ALBTargetGroup ]
      HealthCheckType: ELB
      HealthCheckGracePeriod: 300
      Tags:
        - { Key: Name, Value: "PB - JUL 2025", PropagateAtLaunch: true }
        - { Key: CostCenter, Value: "CO92000024", PropagateAtLaunch: true }
        - { Key: Project, Value: "PB - JUL 2025", PropagateAtLaunch: true }

Outputs:
  WebsiteURL:
    Description: URL do site WordPress. Copie e cole no seu navegador.
    Value: !Sub http://${AppLoadBalancer.DNSName}
  BastionEIP:
    Description: IP Publico do Bastion Host para acesso via SSH.
    Value: !Ref BastionEIP
    
```



## Explicacao do projeto para leigos totais

A Planta Baixa do Nosso "Shopping Center" (Tradução do Template)

Parameters (O Formulário de Pedido)

Antes de começar a construir, precisamos de algumas informações suas. Esta seção é como um formulário que você preenche:

DBPassword: "Qual será a senha secreta para o cofre principal (o banco de dados)?"

KeyName: "Qual é o nome da sua chave-mestra para entrar nas salas de serviço?"

YourIPForSSH: "Qual é o seu endereço de casa? Só vamos permitir que você entre na sala de segurança."

```
+-------------------------------------------------+
|               FORMULÁRIO DE PEDIDO              |
+-------------------------------------------------+
| Senha do Cofre: [ ********* ]                   |
| Nome da Chave:  [ MinhaChaveSecreta ]           |
| Seu Endereço:   [ 1.2.3.4/32 ]                  |
+-------------------------------------------------+
```

Resources (A Lista de Construção)

Esta é a parte principal, onde listamos tudo o que precisamos construir.

1. A Fundação e o Terreno (VPC, Subnets, Gateways)

Primeiro, preparamos o terreno e a estrutura principal do shopping.

VPC: Compramos um grande terreno cercado. Ninguém entra ou sai sem nossa permissão.

PublicSubnetA / PublicSubnetB: Dentro do terreno, criamos duas "Áreas Públicas", como o estacionamento e a praça de entrada. Elas ficam em dois prédios diferentes (A e B) para que, se um tiver um problema, o outro continue funcionando.

PrivateSubnetA / PrivateSubnetB: Também criamos duas "Áreas Restritas" e seguras, onde as lojas de verdade ficarão. Elas também ficam nos prédios A e B.

InternetGateway: Construímos o portão principal do estacionamento, conectando nosso shopping à rua (internet).

NatGateway: Construímos uma porta de saída segura para os funcionários. As lojas na área restrita podem buscar coisas na rua (como atualizações de software), mas ninguém da rua pode entrar por essa porta.

RouteTables: Desenhamos as placas de trânsito que dizem aos carros como chegar ao portão principal (se estiverem na área pública) ou como usar a porta de saída de serviço (se estiverem na área restrita).

```
          +--------------------------------------+
          |           TERRENO CERCADO (VPC)      |
          |                                      |
---> RUA  |  +------------+     +------------+   |
(Internet)|  | Área Pública A |     | Área Privada A |   |
<---      |  +------------+     +------------+   |
(IGW)     |                                      |
          |  +------------+     +------------+   |
          |  | Área Pública B |     | Área Privada B |   |
          |  +------------+     +------------+   |
          +--------------------------------------+
```


2. Os Seguranças (Security Groups)

Agora, colocamos seguranças em cada porta, com regras claras. Cada segurança tem um "crachá" diferente.

ALBSecurityGroup: O segurança da entrada principal do shopping. Ele tem uma ordem: "Deixe qualquer cliente (tráfego da porta 80) entrar".

BastionSecurityGroup: O segurança da sala de controle. A ordem é: "Só deixe entrar a pessoa que mora no endereço X (o seu IP) e que tenha a chave-mestra".

WebServerSecurityGroup: Os seguranças das lojas. Eles têm três ordens:

"Deixe entrar os clientes que o segurança da porta principal (ALBSecurityGroup) mandar."

"Deixe a pessoa da sala de controle (BastionSecurityGroup) entrar para manutenção."

"Lojas que usam este mesmo crachá podem conversar entre si (para o EFS)."

DatabaseSecurityGroup: O segurança do cofre principal. A ordem é simples: "Só deixe entrar quem tiver o crachá de funcionário da loja (WebServerSecurityGroup)".

3. O Cofre e o Almoxarifado (RDS e EFS)

As partes mais importantes do nosso shopping.

EFSFileSystem: Construímos um almoxarifado central e compartilhado. Todas as lojas podem guardar e pegar produtos (imagens, temas, plugins) aqui.

RDSInstance: Construímos o cofre principal e super seguro. É aqui que todo o dinheiro e os registros importantes (posts, usuários) são guardados. Ele fica na área restrita, protegido pelo seu segurança.

4. O Porteiro e a Sala de Controle (Bastion Host)

BastionHost: Construímos uma sala de controle segura na área pública. É a partir daqui que o gerente (você) pode acessar as áreas restritas para fazer manutenção, usando sua chave-mestra.

5. A Loja e a Fábrica (ALB, Launch Template, ASG)

Agora, montamos a loja do WordPress.

AppLoadBalancer (ALB): É o recepcionista inteligente na porta principal do shopping. Ele recebe todos os clientes e os direciona para o caixa (servidor) que estiver mais livre e funcionando bem.

AppLaunchTemplate: É a planta de montagem de um quiosque do WordPress. Ela tem todas as instruções: o tamanho, o sistema, as ferramentas necessárias e um manual de instruções (UserData) que ensina o quiosque a se conectar ao cofre e ao almoxarifado assim que for construído.

AppAutoScalingGroup (ASG): É a fábrica de quiosques. O gerente da fábrica (ASG) tem ordens simples:

"Sempre mantenha 2 quiosques funcionando."

"Se a fila de clientes ficar muito grande (CPU acima de 50%), construa mais um quiosque usando a planta (Launch Template)."

"Se a loja ficar vazia, desmonte um quiosque para economizar espaço."

"Todo quiosque novo, avise ao recepcionista (ALB) que ele está pronto para receber clientes."

Outputs (O Painel de Informações)

Depois que a construção termina, esta seção nos entrega um painel com os endereços mais importantes:

WebsiteURL: "Aqui está o endereço principal do seu shopping para você divulgar aos clientes."

BastionEIP: "E aqui está o endereço secreto da sua sala de controle."

Em resumo, este template é a receita completa e detalhada para construir, do zero, um shopping center digital, seguro e que cresce sozinho de acordo com o movimento
