<div align="center">
  
📦 AWS S3 Event Notifications + SNS

Configurei um bucket S3 pra me mandar email toda vez que alguém subir ou deletar um arquivo. 
Depois tentei tornar um objeto público só pra ver o que acontecia.
Spoiler: o AWS não deixou.

https://aws.amazon.com/
https://aws.amazon.com/s3/
https://aws.amazon.com/sns/
https://aws.amazon.com/iam/
https://aws.amazon.com/cli/
</div>


<p align="center">

<img src="https://img.shields.io/badge/AWS-Cloud-orange?style=for-the-badge&logo=amazonaws&logoColor=white">

<img src="https://img.shields.io/badge/Amazon-S3-569A31?style=for-the-badge&logo=amazons3&logoColor=white">

<img src="https://img.shields.io/badge/Amazon-SNS-FF4F8B?style=for-the-badge&logo=amazonsimpleemailservice&logoColor=white">

<img src="https://img.shields.io/badge/AWS-CLI-232F3E?style=for-the-badge&logo=amazonaws&logoColor=white">

<img src="https://img.shields.io/badge/IAM-Security-DD344C?style=for-the-badge&logo=amazoniam&logoColor=white">

<img src="https://img.shields.io/badge/EC2-Compute-FF9900?style=for-the-badge&logo=amazonec2&logoColor=white">

</p> 




<p align="center">
  <img src="./assets/Gemini_Generated_Image_lycedlycedlycedl.png" width="650">
</p>


## O que é isso

Lab prático de event-driven architecture na AWS. A ideia: toda vez que um arquivo é criado ou deletado num bucket S3, o bucket dispara um evento que chega no SNS e chega no meu email. Sem polling. Sem ficar atualizando a página. O S3 avisa sozinho.

Região: us-east-1
Bucket: cafe-eliana20260806
Tópico SNS: s3NotificationTopic



## Arquitetura

### Etapa 1: Criar o tópico SNS e o inferno da confirmação por email.

Criei o tópico s3NotificationTopic no console AWS. Tipo Standard.
Adicionei minha assinatura por email. O SNS disse que mandou o email de confirmação.
Esperei. Nada. Esperei mais. Nada.
Fui verificar o status da assinatura no console: PendingConfirmation.

Tentativa 2: Deletei a assinatura, criei de novo. Email chegou. Cliquei em confirmar. Voltei pro console: ainda PendingConfirmation.

Tentativa 3: Deletei de novo, criei de novo. Email chegou. Cliquei. Console: Confirmed.
O que aconteceu: O email do SNS demorando muito. E às vezes demora muito mesmo. Além disso, se você clicar no link de confirmação depois de já ter deletado a assinatura original, o link é inválido — por isso a primeira confirmação não funcionou.

Lição: Não delete a assinatura antes de confirmar. Espere. Vai no spam. Vai na aba "Promoções". Vai em todo lugar. 
O email do SNS vem de no-reply@sns.amazonaws.com e alguns provedores filtram como lixo.

### Etapa 2: Conectar o S3 ao SNS

Fui nas propriedades do bucket cafe-eliana20260806 e configurei Event Notifications:

Eventos selecionados:
ObjectCreated:Put — quando alguém faz upload
ObjectRemoved:Delete — quando alguém deleta

Destino: Tópico SNS s3NotificationTopic

O S3 precisa de permissão pra publicar no SNS. O console AWS cuidou disso automaticamente criando a policy no tópico. Se fosse via CLI, eu teria que adicionar manualmente — o que me lembra: sempre verifique a policy do SNS quando não receber notificação.

### Etapa 3: AWS CLI — configurando e testando

Entrei numa instância EC2 (CLI Host) e configurei o AWS CLI:

aws configure
### Access Key ID: [minha chave]
### Secret Access Key: [minha secret]
### Region: us-east-1


### Validei quem eu sou
aws sts get-caller-identity

Tudo certo. Hora de bagunçar o bucket.

### Etapa 4: Upload de objeto e notificação

Subi uma imagem pro bucket:

aws s3api put-object \
  --bucket cafe-eliana20260806 \
  --key images/Caramel-Delight.jpg \
  --body ~/new-images/Caramel-Delight.jpg

  Resposta:

 {
  "ETag": ""31ac30da619244b0ce786f106e4f3df7"",
  "ServerSideEncryption": "AES256"
}

Arquivo no bucket - Ok
Email no inbox Ok (com assunto do SNS e JSON do evento)

O evento chegou assim:

{
  "Records": [{
    "eventName": "ObjectCreated:Put",
    "s3": {
      "bucket": { "name": "cafe-eliana20260806" },
      "object": { "key": "images/Caramel-Delight.jpg" }
    }
  }]
}

Funcionou de primeira. Fiquei surpresa.

### Etapa 5: Download de objeto — e o silêncio

Baixei um arquivo do bucket pra testar:

aws s3api get-object \
  --bucket cafe-eliana20260806 \
  --key images/Donuts.jpg \
  Donuts.jpg

  Arquivo baixou. Mas nenhum email chegou.
  
Fiquei 2 minutos achando que tinha quebrado algo. Aí lembrei: eu configurei eventos só para Put e Delete. Get não gera evento no S3 por padrão.

### O que eu aprendi: Só recebe notificação do que você configurou. Não adianta esperar email de download se você não pediu.

### Etapa 6: Deleção de objeto e notificação

Deletei um arquivo pra testar o outro evento:

aws s3api delete-object \
  --bucket cafe-eliana20260806 \
  --key images/Strawberry-Tarts.jpg
  
Resposta:

{
  "DeleteMarker": true,
  "VersionId": "..."
}

Email chegou em segundos. Evento ObjectRemoved:Delete. Pipeline completo.

### Etapa 7: Teste de segurança — "e se eu tentar tornar público?"

Tive uma ideia: e se alguém tentar tornar um objeto público via ACL? O que o AWS faz?

aws s3api put-object-acl \
  --bucket cafe-eliana20260806 \
  --key images/Donuts.jpg \
  --acl public-read

Resposta:

An error occurred (AccessDenied) when calling the PutObjectAcl operation:
Access Denied

Por que? O bucket tinha S3 Block Public Access ativado no nível da conta. Mesmo que eu tivesse permissão IAM pra mudar ACL, o Block Public Access bloqueia antes.
Isso me mostrou duas camadas de segurança trabalhando juntas:
IAM — controla quem pode fazer o quê
Block Public Access — guarda-rail que impede exposição pública independente de IAM
O que eu aprendi: Segurança na AWS não é só permissão. É também prevenção. O AccessDenied foi a melhor resposta que eu poderia receber.


<p align="center">
  <img src="./assets/Gemini_Generated_Image_t499fat499fat499.png" width="650">
</p>


## Evidências

| O que tá acontecendo                                   | Print                                   |
| ------------------------------------------------------ | --------------------------------------- |
| Arquitetura do pipeline S3 → SNS → Email               | `images/s3-sns-architecture.png`        |
| Email de confirmação do SNS (demorou, mas chegou)      | `images/sns-email-confirmation.png`     |
| Assinatura no console: PendingConfirmation → Confirmed | `images/sns-subscription-confirmed.png` |
| Evento `ObjectCreated:Put` no email                    | `images/s3-put-event-email.png`         |
| Evento `ObjectRemoved:Delete` no email                 | `images/s3-delete-event-email.png`      |
| Erro `AccessDenied` no terminal                        | `images/s3-access-denied.png`           |


## Tech Stack

Amazon S3 — storage e geração de eventos
Amazon SNS — entrega de notificações por email
AWS CLI — automação de operações no bucket
IAM — credenciais e permissões
S3 Block Public Access — guarda-rail de segurança
Server-side encryption (AES-256) — criptografia padrão do bucket



## O que esse lab realmente me ensinou

1.Event-driven é diferente de polling. Eu não precisei ficar perguntando pro S3 "tem arquivo novo?". O S3 me avisou. Isso escala infinitamente melhor.
2.SNS email tem delay. E às vezes trauma. O email demorou, chegou 3 vezes, e eu quase desisti. Em produção, você usaria Lambda ou SQS pra processar o evento em tempo real. Email é só pra teste e alertas simples.
3.Block Public Access salva. Eu tentei tornar público e falhei. Em um bucket sem isso, eu teria exposto dados sem querer. Sempre ative.
4.GET não é evento. Parece óbvio depois que você sabe. Mas quando o email não chegou, eu achei que tinha quebrado tudo.
5.Confirmação de assinatura SNS é delicada. Não delete antes de confirmar. Espere. Olhe o spam. E tenha paciência.


## 🚧 Status

[x] Tópico SNS criado
[x] Assinatura por email confirmada (depois de 3 tentativas e muita paciência)
[x] Eventos S3 configurados (Put + Delete)
[x] Upload testado com notificação
[x] Download testado (sem notificação, como esperado)
[x] Deleção testada com notificação
[x] Teste de segurança: AccessDenied no put-object-acl
[ ] Conectar isso a uma Lambda pra processar eventos em vez de só email

## 🌐 Contato
💼 LinkedIn: linkedin.com/in/eliana-diniz
📧 E-mail: eliana.dinizsilva@gmail.com

"Tentei tornar um arquivo público. O AWS me deu um AccessDenied. Foi o melhor 'não' que eu já recebi."





