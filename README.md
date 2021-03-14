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


Faça deploy do seu gateway. Durante o primeiro deploy vai ser necessário criar um 'stage'. O 'stage' é o local onde os seus recursos realmente ficam disponíveis para uso. Toda vez que uma rota é criado / alterada, se não fizer o Deploy em 'stage' sua mudança não será refletida. 

![image](https://user-images.githubusercontent.com/22084402/111054067-26e53780-8448-11eb-81e6-3c3ffbe704d9.png)

Quando vc for fazer o deploy do API Gateway encontrará a seguinte mensagem de erro: 'Private REST API doesn't have a resource policy attached to it'.

Pelo fato de ser um gateway privado temos que definir a Policy de uso antes de criarmos o stage.

No menu da esquerda abra a opção 'Resource Policy'.

Cole a Policy abaixo (é parecida com o exemplo que a AWS mostra se vc clicar no botão 'Source VPC Allowlist'.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "execute-api:Invoke",
            "Resource": "arn:aws:execute-api:<region>:<account_id>:<id_gateway>/*"
        },
        {
            "Effect": "Deny",
            "Principal": "*",
            "Action": "execute-api:Invoke",
            "Resource": "arn:aws:execute-api:<region>:<account_id>:<id_gateway>/*",
            "Condition": {
                "StringNotEquals": {
                    "aws:SourceVpc": "<vpc_id>"
                }
            }
        }
    ]
}
```   

Caso não saiba como pegar os valores acima:

**Region:** fica no canto superior direito

![image](https://user-images.githubusercontent.com/22084402/111071653-f0460600-84b5-11eb-9183-c9cb84bb13d9.png)

Nesse caso o region vai ser 'us-east-1'.

**ID do Gateway:**

![image](https://user-images.githubusercontent.com/22084402/111071673-0653c680-84b6-11eb-9d3d-f207e7f1457c.png)


**Account ID:**
Clique no menu superior (perto da region) onde fica o seu usuário, uma das opções que serão exibida é o account ID ('My Account').


**VPC_ID**:
Abra o menu da VPC e veja o ID da mesma.


Volte no menu Resources do gateway e faça o deploy novamente.

Como é um gateway de testes, eu tenho custome de alterar a capacidade de Throttling:

![image](https://user-images.githubusercontent.com/22084402/111071630-d1477400-84b5-11eb-802f-c74e244ab84d.png)



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


**7. Testando o VPC Endpoint e verificando SG (security group)**

Conecte na sua EC2.
Lembre-se que se vc reiniciou ela, provavelmente ela subiu com outro IP interno.
Lembre-se que se vc reiniciou seu PC, provavelmente ele trocou seu IP local, portanto, é necessário reconfigurar o SG de acesso na EC2.

No exemplo abaixo, vamos utilizar o Postman para montarmos os requests. Facilitará nossa vida para copiarmos o request pronto durante os testes.

Lembrando que por tratarmos de um API Gateway Privado não conseguirmos acessar o enedreço do gateway diretamente. Por isso, temos que acessá-lo através do endereço VPC Endpoint.

Monte a url no postman e depois clique em Code.

![image](https://user-images.githubusercontent.com/22084402/111054103-8ba09200-8448-11eb-84f4-b4ab79c4f47e.png)

Repare que a URL abaixo é composta do endpoint é composta da seguinte maneira:

https://<vpc_endpoint>/<stg_api_gateway>/<rota_api_gateway>

https://vpce-032ae4eee4be64031-vatu67qs.execute-api.us-east-1.vpce.amazonaws.com/dev/store

Copie o comando e execute de dentro da sua EC2.

Repare que vc vai ver um erro de timeout. Isso ocorre porque o Security Group do VPC Link não permite acesso.

Abra o Security Group que vc colocou durante a criação do VPC Link e adicione a seguinte regra:

![image](https://user-images.githubusercontent.com/22084402/111054157-e33efd80-8448-11eb-9c44-27780b277dc2.png)

Estamos liberando a porta HTTPS onde a origem seja o Security Group que a EC2 utiliza. Portanto, só quem está dentro desse Security Group conseguirá acesasr o VPC Endpoint.

Repita o teste e perceba que a mensagem de erro agora é 'Forbidden'. Isso significa que o VPC Endpoint está redirecionando a chamada para o Gateway, porém, agora o Gateway está dando acesso negado.

**8. Definindo plano de uso do APIGateway**


Abra o gateway e vá até o menu 'Usage Plans'.
























