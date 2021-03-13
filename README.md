# API Gateway Privado + VPC Endpoint
Essa branch é referente a uma POC de API Gateway Privado com acesso através de um VPC Endpoint.

Existem alguns tipos de APIGateway da AWS. A idéia dessa POC é exibir a integração entre API Gateway Privado + VPC Endpoint

## API Gateway Privado
Quando temos um API Gateway Privado (AWS) não conseguimos acessar ele pela internet. Para isso, temos que configurar uma rota de comunicação um VPC e nosso gateway. Essa comunicação chama-se VPC Endpoint.

## Quais serviços também serão utilizados nessa POC ?
* **EC2**: Precisaremos nos conectar em uma EC2 para nos comunicar com o gateway privado.
* **VPC**: Precisamos definir uma VPC para acesso.
* **IAM**: Precisamos liberar acesso de conexão para a nossa EC2 e também para que a comunicação entre API Gateway e VPC funcione.
* **APIGateway**: É o nosso serviço principal nessa POC.
* **Lambda**: Faremos uma integração com Lambda para verificarmos a diferença de respostas

**1. Vamos começar com a VPC e EC2**

Crie uma VPC e dê um nome para ela. Se ela não for sua Default VPC pode ser que ela não tenha configuração de saída para internet.
Para essa POC acredito não ser um problema (porque a comunicação com o api gateway será privada), mas lembre-se disso durante os testes.

**2. Crie uma EC2**
No exemplo abaixo criamos uma EC2 com imagem 'Amazon Linux 2 AMI'.
Aqui precisamos ter alguns pontos de atenção:
- Essa EC2 precisa ter IP Publico (para que vc consiga conectar-se nela)

Obs.: Se vc esqueceu de criar ela para ter IP Público, terá que criar outra EC2.
      Caso tenha criado errado, lembre-se de usar o comando 'Terminate' para matar a EC2 criada errada. Se vc apensar usar o 'Stop', vc pagará pelo EBS (disco) da EC2 mesmo com ela desligada.

- Vc vai precisar de uma chave (.pem) para conectar-se na EC2.
- Security Group pode deixar padrão, ou criar um. Alteraremos ele na sequência.

Após baixar o arquivo (.pem) precisamos alterar a permissão dele. Nesse exemplo, estamos usando GitBash no Windows.
- Navegue até o local do arquivo e execute o comando **chmod 400 nome_do_seu_arquivo.pem**
- Abra as configurações da sua EC2 (Console da AWS) e espera ela estar com status 'Running'.
- Pegue o IP Public dela.
- Tente conectar na EC2 através do seguinte comando **ssh -i <nome_do_seu_arquivo.pem> ec2-user@<ip_publico_da_ec2>**
Obs.: Proavelmente você vai tomar erro de timeout devido a falta de configuração do Security Group.

**3. Vamos liberar acesso nessa EC2 para nosso SG (Security Group)**
- Dentro do menu EC2 vc vai encontrar a opção 'Security Groups'. Abra o seu Security Group e veja as regras de Inbound.

![image](https://user-images.githubusercontent.com/22084402/111014742-f9cf5100-8383-11eb-80bc-6449596d33c9.png)

- Adicione a regra de Inboud na porta 22 (SSH) apenas do seu IP (por questões de segurança).
- Repita o comando para conectar na máquina: **ssh -i <nome_do_seu_arquivo.pem> ec2-user@<ip_publico_da_ec2>
Obs.: Se o ssh perguntar sobre a chave fingerprint digite 'yes'

Nesse momento vc deve estar conectado na máquina:

![image](https://user-images.githubusercontent.com/22084402/111014832-69ddd700-8384-11eb-894e-448411ba8f65.png)

**4. Crie um APIGateway Privado**

![image](https://user-images.githubusercontent.com/22084402/111029027-39745800-83d9-11eb-80f4-a11e092359e2.png)


Preencha as informações necessárias para criar o APIGateway e nesse momento, vamos deixar o APIGateway criar 'api de exemplo' para nós.

![image](https://user-images.githubusercontent.com/22084402/111029060-632d7f00-83d9-11eb-98d4-411652a12c0f.png)

**5. Crie um VPC Endpoint**

No serviço VPC, vc encontrará o menu "Endpoints".

Crie um Endpoint do tipo API Gateway 'com.amazonaws.<region>.execute-api':
![image](https://user-images.githubusercontent.com/22084402/111029339-029f4180-83db-11eb-9dc0-71c850ddf211.png)

Preencha as informações. Sugiro criar um outro Security Group, explicaremos o motivo depois. Anote o nome do Security Group.

Referente a policy, pode deixar 'Full Access'. Não é o ideal, mas como ele será privado (acesso através da VPC) e isso é uma POC, podemos deixar assim.

Obs.: Se vc não conseguir criar o VPC Endpoint porque a sua VPC não permite resolução de DNS, volte para a VPC.
Habilite a opção 'DNS Hostnames'
![image](https://user-images.githubusercontent.com/22084402/111029594-42b2f400-83dc-11eb-98cb-d5e0fd6a5b79.png)

Habiliate a opção 'DNS Resolution'
![image](https://user-images.githubusercontent.com/22084402/111029606-5a8a7800-83dc-11eb-9948-738405fd015f.png)


Volte no VPC Endpoint e tente criá-lo novamente.

Após criar o VPC Endpoint, anote a url de DNS:
![image](https://user-images.githubusercontent.com/22084402/111029653-9291bb00-83dc-11eb-8607-8c186b5e4dc3.png)



**6. Testando o VPC Endpoint e verificando SG (security group)**

Conecte na sua EC2.
Lembre-se que se vc reiniciou ela, provavelmente ela subiu com outro IP interno.
Lembre-se que se vc reiniciou seu PC, provavelmente ele trocou seu IP local, portanto, é necessário reconfigurar o SG de acesso na EC2.

No exemplo abaixo, vamos utilizar o Postman para montarmos os requests. Facilitará nossa vida para copiarmos o request pronto durante os testes.




















