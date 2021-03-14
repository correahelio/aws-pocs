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

Repita o teste e vc verá a seguinte mensagem de erro 'Forbidden'. Isso significa que a requisição chegou até o VPC Endpoint mas o Endpoint não sabe qual gateway deve ser chamado. 

Para isso, será necessário adicionar o id do gateway no header da nossa chamada através do parâmetro 'x-apigw-api-id'

![image](https://user-images.githubusercontent.com/22084402/111075204-9ea57780-84c5-11eb-86e3-362dbd7b4044.png)


Repita o teste e veja que funcionou.

![image](https://user-images.githubusercontent.com/22084402/111075130-3e163a80-84c5-11eb-85cb-cd6853d51e95.png)


**8. Definindo plano de uso do APIGateway**

Nesse momento, não temos ainda garantia de quem está chamando o Gateway. A única restrição que colocamos é que a origem seja da VPC permitada.

Porém, é possível definir plano de uso para o gateway.

Para isso, abra o gateway e vá até o menu 'Usage Plans'. Crie um plano de uso:

![image](https://user-images.githubusercontent.com/22084402/111072176-21bfd100-84b8-11eb-871b-7f5d4a03114b.png)


Na próxima tela escolha o gateway e stage que o plano de uso deve ser vinculado:

![image](https://user-images.githubusercontent.com/22084402/111072205-474cda80-84b8-11eb-964f-60a9d95366f4.png)


E na última tecla, crie uma chave para ser usada através desse plano de uso.

![image](https://user-images.githubusercontent.com/22084402/111072220-5c296e00-84b8-11eb-81ff-cbc4ca882ffc.png)


**9. Testando o VPC Endpoint com plano de uso**

Conecte na ec2, e monte a url no postman.

Reparou que ainda teremos o mesmo cenário de sucesso ?

Legal, mas a chave que nós definimos já não está sendo usada  ?
A resposta é Não, ela só será usada se vc proteger sua rota.

Para isso, abra a rota dentro do menu Resources e abra o Method Request. Altere o combo 'API Key Required' para true.

![image](https://user-images.githubusercontent.com/22084402/111074421-8e8b9900-84c1-11eb-8ad4-da03bf763db8.png)

Não esqueça de fazer deploy do gateway. Lembre-se que após o deploy o gateway demora alguns segundos para refletir as mudanças.

A partir disso, vc verá novamente a mensagem de erro 'Forbidden'.

Adicione a sua chave através do header 'x-api-key' e repita o teste.

![image](https://user-images.githubusercontent.com/22084402/111074454-c5fa4580-84c1-11eb-9392-c9f7dbb05e56.png)

Voltou a funcionar. Portanto, o nosso método GET /pets está protegido através da chave com o plano de uso definido.


**10. Testando o mesmo VPC Endpoint com outro APIGateway**

Vamos criar um outro Gateway para testarmos o mesmo VPC Endpoint. Porém, como já utilizamos o gateway de exemplo, vamos criar um Lambda.

Crie um lambda na AWS que ele já vem com o 'hello-world' de exemplo.

![image](https://user-images.githubusercontent.com/22084402/111074506-0659c380-84c2-11eb-933d-55ac67cacf67.png)


Crie um outro API Gateway Privado só que dessa vez crie um 'New API' para que ele não crie os exemplos.
![image](https://user-images.githubusercontent.com/22084402/111074557-35703500-84c2-11eb-8171-b7d7b4d8641a.png)

Crie um recurso e depois um método GET.

![image](https://user-images.githubusercontent.com/22084402/111074578-53d63080-84c2-11eb-9084-32cae1b8ed55.png)


Lembre-se de alterar a Policy do gateway por ser privado e lembre-se de fazer deploy do gateway. Nesse exemplo criei um stage chamado 'dev2', apenas para facilitar a leitura.

![image](https://user-images.githubusercontent.com/22084402/111074709-04443480-84c3-11eb-8bb7-ba479c2a1f8d.png)

Repare que o que mudou na URL é o stage e a rota do novo gateway que agora chama-se /lambda.

![image](https://user-images.githubusercontent.com/22084402/111075371-808c4700-84c6-11eb-8efc-c66b7de0b1b6.png)

Estamos usando o mesmo VPC Endpoint chamando outro gateway. A única diferença é o ID do Gateway informado no header do request.

Esse segundo gateway não tem plano de uso, portanto, apenas informar o ID do gateway é o suficinet para funcionar!


















 























