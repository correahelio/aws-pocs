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

1. Vamos começar com a VPC e EC2.

Crie uma VPC e dê um nome para ela. Se ela não for sua Default VPC pode ser que ela não tenha configuração de saída para internet.
Para essa POC acredito não ser um problema (porque a comunicação com o api gateway será privada), mas lembre-se disso durante os testes.

2. Crie uma EC2. No exemplo abaixo criamos uma EC2 com imagem 'Amazon Linux 2 AMI'.
Aqui precisamos ter alguns pontos de atenção:
- Essa EC2 precisa ter IP Publico (para que vc consiga conectar-se nela)
- Vc vai precisar de uma chave (.pem) para conectar-se na EC2.
- Security Group pode deixar padrão, ou criar um. Alteraremos ele na sequência.

Após baixar o arquivo (.pem) precisamos alterar a permissão dele. Nesse exemplo, estamos usando GitBash no Windows.
- Navegue até o local do arquivo e execute o comando **chmod 400 nome_do_seu_arquivo.pem**
- Abra as configurações da sua EC2 (Console da AWS) e espera ela estar com status 'Running'.
- Pegue o IP Public dela.
- Tente conectar na EC2 através do seguinte comando **ssh -i <nome_do_seu_arquivo.pem> ec2-user@<ip_publico_da_ec2>**
Obs.: Proavelmente você vai tomar erro de timeout devido a falta de configuração do Security Group.

3. Vamos liberar acesso nessa EC2 para nosso SG (Security Group)
- Dentro do menu EC2 vc vai encontrar a opção 'Security Groups'. Abra o seu Security Group e veja as regras de Inbound.

![image](https://user-images.githubusercontent.com/22084402/111014742-f9cf5100-8383-11eb-80bc-6449596d33c9.png)

- Adicione a regra de Inboud na porta 22 (SSH) apenas do seu IP (por questões de segurança).
- Repita o comando para conectar na máquina: **ssh -i <nome_do_seu_arquivo.pem> ec2-user@<ip_publico_da_ec2>
Obs.: Se o ssh perguntar sobre a chave fingerprint digite 'yes'

Nesse momento vc deve estar conectado na máquina:

![image](https://user-images.githubusercontent.com/22084402/111014832-69ddd700-8384-11eb-894e-448411ba8f65.png)










