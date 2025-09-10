Documentação: WordPress Escalável e Resiliente na AWS
Data de Criação: 4 de Setembro de 2025

1. Visão Geral e Arquitetura
1.1. Objetivo

O objetivo deste projeto é implantar uma aplicação WordPress na nuvem da AWS de forma segura, escalável, resiliente e automatizada. A arquitetura é projetada para lidar com variações de tráfego, se recuperar de falhas e automatizar a configuração de novos servidores web.

1.2. Diagrama da Arquitetura

<img width="680" height="319" alt="Captura de Tela 2025-09-04 às 21 51 01" src="https://github.com/user-attachments/assets/6759767b-6158-4ba2-b5a1-54b0ad3dba51" />


O projeto segue o padrão de uma aplicação web de três camadas (web, dados, armazenamento), utilizando os seguintes componentes principais:

Explicacao do projeto para leigos totais

A Planta Baixa do Nosso "Shopping Center" (Tradução do Template)

Parameters (O Formulário de Pedido)

Antes de começar a construir, precisamos de algumas informações suas. Esta seção é como um formulário que você preenche:

DBPassword: "Qual será a senha secreta para o cofre principal (o banco de dados)?"

KeyName: "Qual é o nome da sua chave-mestra para entrar nas salas de serviço?"

YourIPForSSH: "Qual é o seu endereço de casa? Só vamos permitir que você entre na sala de segurança."

+-------------------------------------------------+
|               FORMULÁRIO DE PEDIDO              |
+-------------------------------------------------+
| Senha do Cofre: [ ********* ]                   |
| Nome da Chave:  [ MinhaChaveSecreta ]           |
| Seu Endereço:   [ 1.2.3.4/32 ]                  |
+-------------------------------------------------+
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
