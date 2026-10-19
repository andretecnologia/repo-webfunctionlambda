# repo-webfunctionlambda

Especificação Técnica — SSR Next.js no AWS Lambda Web Functions
1. Produto e referências
Produto: AWS Lambda Web Functions — um formato de execução (form factor) da Lambda para rodar servidores HTTP nativos, sem handler de evento e sem adaptador.
Data de anúncio: outubro de 2026.
Documentação oficial: o produto tem duas fontes principais — o blog de lançamento da AWS ("Announcing AWS Lambda Web Functions") e a página de produto em aws.amazon.com/lambda/lambda-web-functions, além da referência de comandos da AWS CLI (lambda-web) e do Developer Guide da Lambda (seção Web Functions).
2. Pré-requisitos para funcionamento ponta a ponta
Ferramentas:

AWS CLI mínima: 2.37.9 — o conjunto de comandos aws lambda-web só existe a partir dessa versão. Abaixo disso o comando não é reconhecido.
Node.js 24 — é o runtime da função (nodejs24.x) e o alvo de build da app.
Terraform (para a camada de borda/DNS) e credenciais AWS com as permissões da seção 6.
Regiões disponíveis (no lançamento):

US East (N. Virginia) — us-east-1
US East (Ohio) — us-east-2
US West (Oregon) — us-west-2
Asia Pacific (Tokyo) — ap-northeast-1
Europe (Ireland) — eu-west-1
Esta solução usa us-east-1.

Runtime no lançamento: Node.js 24 em Arm64, pacote de deploy de até 512 MB, com suporte a conectividade VPC.

3. Componentes da solução
Componente	Função
Lambda Web Function	Roda a app Next.js SSR (servidor HTTP nativo)
Amazon CloudFront	CDN à frente: cache de estáticos + encaminha dinâmico
AWS Certificate Manager	Certificado TLS do subdomínio (us-east-1, grátis)
Amazon Route 53	Registro DNS do subdomínio → CloudFront
Amazon S3	Bucket gerenciado do pacote de deploy + bucket de state do Terraform
IAM	Role de execução da função + policy do usuário de CI
CloudWatch Logs	Logs da função
GitHub Actions	Orquestra build, deploy e destroy
4. Desenho de arquitetura mínimo

          Usuário (HTTPS)
                │
                ▼
        Amazon Route 53
   (subdominio.dominio → alias)
                │
                ▼
         Amazon CloudFront
   ┌──────────────┴───────────────┐
   │ /_next/static/*     resto (/) │
   │ (cache de borda)    (dinâmico)│
   └──────────────┬───────────────┘
                  ▼
     Lambda Web Function (us-east-1)
     Node.js 24 / Arm64 / HTTP nativo
     ┌─────────────────────────────┐
     │ lambda-entry.js (wrapper)   │
     │   → server.js (Next standalone)
     │   → app Next.js (SSR)       │
     └─────────────────────────────┘
                  │
         CloudWatch Logs
Recursos de suporte: S3 (pacote de deploy) e IAM role de execução, ambos criados automaticamente pelo aws lambda-web deploy.

5. Role IAM da função (execução)
Criada automaticamente pelo deploy (nome awscli-lambdaweb-<função>). Trust policy para o serviço Lambda e, no mínimo, permissão de logs:

json

{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "lambda.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
Permissões anexadas: AWSLambdaBasicExecutionRole (CloudWatch Logs). Se a app acessar outros serviços (DynamoDB, S3, Bedrock), adiciona-se o acesso correspondente nessa role.

6. Policy do usuário/CI (deploy)
As ações do Web Functions ficam no namespace lambda: (ex.: CreateWebFunction). A policy de CI precisa de:

json

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "LambdaWebFunctions",
      "Effect": "Allow",
      "Action": [
        "lambda:CreateWebFunction", "lambda:DeleteWebFunction",
        "lambda:GetWebFunction", "lambda:ListWebFunctions",
        "lambda:CreateWebFunctionRevision", "lambda:CreateWebFunctionEndpoint",
        "lambda:DeleteWebFunctionEndpoint", "lambda:GetWebFunctionEndpoint",
        "lambda:UpdateWebFunctionEndpoint", "lambda:TagResource"
      ],
      "Resource": "*"
    },
    {
      "Sid": "CloudFront",
      "Effect": "Allow",
      "Action": [
        "cloudfront:CreateDistribution", "cloudfront:GetDistribution",
        "cloudfront:UpdateDistribution", "cloudfront:DeleteDistribution",
        "cloudfront:TagResource", "cloudfront:ListDistributions"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DeployBucket",
      "Effect": "Allow",
      "Action": [
        "s3:CreateBucket", "s3:GetObject", "s3:GetObjectVersion",
        "s3:PutObject", "s3:DeleteObject", "s3:ListBucket"
      ],
      "Resource": ["arn:aws:s3:::awscli-lambdaweb-*", "arn:aws:s3:::awscli-lambdaweb-*/*"]
    },
    {
      "Sid": "ExecRole",
      "Effect": "Allow",
      "Action": ["iam:CreateRole", "iam:AttachRolePolicy", "iam:PassRole", "iam:GetRole"],
      "Resource": "arn:aws:iam::<conta>:role/awscli-lambdaweb-*"
    },
    {
      "Sid": "DnsAndCert",
      "Effect": "Allow",
      "Action": ["route53:*", "acm:RequestCertificate", "acm:DescribeCertificate", "acm:DeleteCertificate"],
      "Resource": "*"
    }
  ]
}
Dois pontos aprendidos na prática: o deploy referencia o pacote no S3 por version ID, logo s3:GetObjectVersion é obrigatório; e CloudFront exige permissões próprias que não vinham na policy original.

7. Comportamento com 0 requisições
A função escala a zero: sem requisição, nenhum ambiente de execução fica rodando.
Não há cobrança de CPU nem de memória nesse estado.
Permanecem ativos apenas recursos de custo quase nulo: o endpoint registrado, o pacote no S3, o certificado ACM (grátis) e o registro DNS.
A primeira requisição após o período ocioso provisiona um ambiente — cold start.
8. Comportamento com centenas de milhares de requisições
Cada ambiente de execução atende até 128 requisições simultâneas, então o tráfego compartilha ambientes quentes em vez de mapear 1 requisição por ambiente.
Quando a demanda supera a capacidade dos ambientes ativos, a plataforma cria mais ambientes automaticamente (scale out horizontal), sem provisionamento manual.
A cobrança segue o CPU ativo: tempo em espera de I/O (banco, APIs) quase não é faturado, o que mantém o custo proporcional ao processamento real sob alto volume.
Opcionalmente, com endpoint multi-região, as requisições são roteadas para a região de menor latência sob um único domínio global.
9. Caso de uso: aplicações full-stack
Para full-stack JavaScript (React/Next com SSR + rotas de API no mesmo deploy), o encaixe é direto:

Uma única entrega serve HTML renderizado no servidor, assets e rotas de API, sem separar frontend e backend em infraestruturas distintas.
SSR por requisição entrega conteúdo personalizado/dinâmico com desempenho consistente, enquanto o CloudFront cacheia os estáticos na borda.
Multi-concorrência absorve picos sem a penalidade de cold start de 1-por-requisição, adequado a audiências globais.
Cobrança por CPU ativo reduz custo nas chamadas I/O-bound típicas de SSR (fetch de dados, render).
Framework roda sem adaptador: o servidor de produção do Next é o mesmo local e em produção.
10. Como foi feita esta implementação
App: Next.js (App Router) com página em modo dinâmico, renderizada no servidor a cada acesso, exibindo um carimbo de tempo que muda por requisição — prova visível do SSR.
Build: modo de servidor autossuficiente (output: standalone), gerando o servidor com apenas as dependências de runtime.
Ponto de entrada: um wrapper fino (lambda-entry.js) traduz a variável de bind do ambiente para as variáveis que o servidor gerado pelo Next espera e delega a inicialização a ele. Isso evita o erro de inicializar o framework de forma programática no pacote otimizado (ausência de módulos de build).
Deploy: aws lambda-web deploy --create (não interativo), em us-east-1, com tag de alocação de custo. A CLI cria bucket S3, role de execução, função, revisão e endpoint, e retorna a URL HTTPS.
Borda/DNS: Terraform cria CloudFront (cache de estáticos + origem dinâmica), certificado ACM validado por DNS e registro Route 53 para o subdomínio, recebendo o host do endpoint capturado no deploy.
CI/CD (up e down): dois workflows no GitHub Actions usando os secrets AWS do repositório. O de subida instala a CLI ≥ 2.37.9, builda, empacota, publica a função, captura o endpoint e aplica a infra. O de destruição remove CloudFront/ACM/Route 53 via Terraform e apaga a função com revisões e endpoints. Há ainda um workflow de FinOps que reporta custo por tag e recursos ativos.
Verificação: feita ao vivo — endpoint e domínio custom respondendo HTTP 200, carimbo de tempo distinto entre requisições (SSR confirmado) e ciclo completo de destroy→create validado pelos pipelines.
