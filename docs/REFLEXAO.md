# A5 - Reflexão e equivalência em nuvem

## Seção 5 - Questões de reflexão

Respostas construídas a partir da discussão e das evidências da atividade. A seção 5.1 apresenta uma proposta de equivalência com serviços gerenciados; esses recursos de nuvem não foram provisionados nesta atividade.

### Q1. Por que o web consegue acessar o banco pelo nome db? O que muda na bridge padrão?

O web consegue usar `db` porque ambos os serviços participam da rede definida pelo usuário `rede_interna`.  
O DNS interno do Docker resolve esse nome para o IP atual do contêiner do banco; a bridge permite a comunicação entre eles.  
Assim, não precisamos fixar um IP, que pode mudar quando o contêiner é recriado.  
Na bridge padrão do `docker run`, não existe essa descoberta automática por nome; uma solução seria criar uma rede definida pelo usuário.  
A configuração está no [docker-compose.yml](../docker-compose.yml), e a [inspeção da rede](evidencias/11-inspecao-rede.png) mostra os três serviços normais conectados a ela.

### Q2. Qual a diferença entre volume nomeado e bind mount? Por que cada um foi escolhido?

Um bind mount disponibiliza um arquivo ou diretório específico do host dentro do contêiner; um volume nomeado tem seu armazenamento gerenciado pelo Docker.  
O `init.sql` usa bind mount porque é um arquivo de inicialização mantido no repositório e montado somente para leitura.  
Os dados do PostgreSQL usam `dados_banco` para permanecerem independentes da existência do contêiner e serem administrados como volume pelo Docker.  
Os dois tipos podem preservar arquivos após remover contêineres; a diferença principal é como a origem do armazenamento é escolhida e gerenciada.  
O [Compose](../docker-compose.yml) define as montagens; a [inspeção dos volumes](evidencias/tarefa-5/06b-inspecao-volumes.png) e a comparação [antes](evidencias/tarefa-5/01-antes-recriacao.png) e [depois](evidencias/tarefa-5/03-depois-recriacao.png) demonstram a persistência dos dados.

### Q3. Alterar init.sql e executar up novamente aplica a mudança? Como alterar sem perder dados?

Não. A imagem do PostgreSQL executa `init.sql` apenas na inicialização de um diretório de dados vazio.  
Como o volume já contém um banco inicializado, repetir `up -d`, mesmo com build, não reaplica o script.  
Para alterar o esquema sem perder os registros, faria um backup e aplicaria uma migração no banco existente, por exemplo `ALTER TABLE itens ADD COLUMN observacao TEXT`.  
O volume mantém os dados, mas não aplica automaticamente mudanças no esquema; o comando SQL citado é uma proposta, não uma alteração realizada nesta atividade.  
Não usaria `down -v` para migrar: nossas evidências mostram a [remoção dos volumes](evidencias/tarefa-5/09-remocao-volumes.png) e a [reinicialização com somente os três exemplos](evidencias/tarefa-5/10b-dados-apos-remocao.png).

### Q4. Por que depends_on sozinho não garante a primeira conexão com o banco?

O `depends_on` simples organiza a ordem de início, mas não garante que o PostgreSQL já esteja pronto para aceitar conexões.  
O contêiner pode estar rodando enquanto o banco ainda prepara seus dados e executa o `init.sql`.  
Se o web tentar conectar nesse intervalo, a primeira tentativa pode falhar.  
Por isso, o [Compose](../docker-compose.yml) usa `condition: service_healthy` junto ao healthcheck do banco com `pg_isready`.  
A [evidência da subida](evidencias/tarefa-4/01-subida-build.png) mostra o banco saudável antes da liberação do web; essa espera inicial não garante que o banco nunca falhará depois.

### Q5. Como a ordem de COPY e pip install influencia o cache e o tempo de build?

A ordem permite aproveitar o cache de camadas do Docker, reutilizando etapas cujas entradas não mudaram.  
No [Dockerfile](../app/Dockerfile), `requirements.txt` é copiado e instalado antes de `app.py`; se somente o código mudar, a instalação dos pacotes pode ser reutilizada.  
Nesse caso, a cópia do código e as etapas seguintes são refeitas; as etapas anteriores continuam elegíveis ao cache.  
Se o código fosse copiado antes da instalação, suas mudanças também invalidariam essa etapa, aumentando o trabalho do build.  
O pip instala a lista do requirements, não procura dependências no app; `--no-cache-dir` controla seu cache de downloads, não o cache de build do Docker.  
A [evidência da construção](evidencias/tarefa-4/01-subida-build.png) mostra etapas marcadas como `CACHED`, indicando reaproveitamento.

### Q6. Por que não publicar 5432? Qual a relação com a LGPD e quais medidas acrescentar?

Não publicar 5432 evita uma entrada adicional pelo host para o banco, pois o web já consegue acessá-lo pela rede Docker.  
A ausência dessa publicação está no [Compose](../docker-compose.yml) e na [listagem dos serviços](evidencias/tarefa-4/02-servicos.png); isso reduz a exposição, mas não protege sozinho todos os acessos pela API.  
Fotos de documentos podem identificar pessoas, e o art. 46 da [LGPD](https://www.planalto.gov.br/ccivil_03/_ato2015-2018/2018/lei/l13709.htm) exige medidas técnicas e administrativas contra acessos e tratamentos inadequados.  
Como primeira medida, exigiria autenticação e autorização: login para identificar o usuário e permissões para limitar quem pode visualizar fotos de documentos.  
Como segunda medida, ocultaria nomes, números de documentos e outros detalhes desnecessários antes de disponibilizar as imagens, reduzindo a exposição de dados pessoais.  
Essas medidas são propostas para uma próxima etapa e não foram implementadas na aplicação fornecida.

### Q7. O que acontece com scale web=3? O que é necessário e o volume pode ser compartilhado?

O comando `docker compose up -d --scale web=3` manteve web-1 e iniciou web-2 e web-3, resultando em três contêineres saudáveis do mesmo serviço, como mostra a [evidência da escala](evidencias/13-escala-web-tres-replicas.png).  
Isso é possível porque o web não publica portas no host, e o [Nginx](../nginx/nginx.conf) pode distribuir as requisições entre as réplicas; no modelo básico com 8080 publicada em cada web, haveria conflito de porta.  
São necessários recursos de CPU e memória suficientes, distribuição do tráfego e acesso ao mesmo banco e armazenamento, como preparado no [Compose](../docker-compose.yml).  
No mesmo Docker Engine, as réplicas podem montar o volume fotos; o [app.py](../app/app.py) gera nomes UUID, reduzindo colisões entre uploads de arquivos diferentes.  
Operações concorrentes sobre o mesmo arquivo exigiriam coordenação, e a gravação da foto e do registro no banco não é atômica; o volume local também não é compartilhado automaticamente entre máquinas.  
A captura confirma cinco contêineres ativos: três web, um db e um nginx; ela comprova a subida das réplicas, mas não mede a distribuição de requisições nem testa uploads simultâneos.

## Seção 5.1 - Do ambiente local para a nuvem

As equivalências abaixo representam a responsabilidade de cada peça em uma arquitetura com serviços gerenciados. Não são substituições automáticas das linhas do Compose; cada serviço exige configuração e, em alguns casos, adaptação do código.

| Peça do ambiente local | Microsoft Azure | AWS | Google Cloud |
|---|---|---|---|
| Contêiner web | Azure Container Apps | Amazon ECS com AWS Fargate | Cloud Run |
| Contêiner db + volume dados_banco | Azure Database for PostgreSQL | Amazon RDS for PostgreSQL | Cloud SQL for PostgreSQL |
| Volume fotos | Azure Blob Storage | Amazon S3 | Cloud Storage |
| rede_interna | Azure Virtual Network | Amazon VPC | VPC |
| Arquivo .env (segredos) | Azure Key Vault | AWS Secrets Manager | Secret Manager |
| Imagem construída localmente | Azure Container Registry | Amazon ECR | Artifact Registry |

Na arquitetura proposta, a peça que mais exigiria adaptação seria o armazenamento das fotos. Atualmente, a aplicação grava e lê arquivos diretamente em `/app/uploads`, montado no volume nomeado `fotos`, como mostram o [código fornecido](../app/app.py) e a [listagem dos arquivos na Tarefa 5](evidencias/tarefa-5/07-arquivos-volume-fotos.png). Ao migrar para Amazon S3, Google Cloud Storage ou Azure Blob Storage, as imagens passariam a ser armazenadas como objetos acessados por APIs, exigindo adaptar o upload e a recuperação das fotos. Também seria necessário configurar a identidade e as permissões da aplicação, manter o armazenamento privado e verificar a autorização do usuário antes de entregar a imagem, preservando as medidas propostas na Q6, inclusive a ocultação de informações pessoais desnecessárias. Esse armazenamento permitiria que réplicas em diferentes máquinas acessassem as mesmas imagens sem depender do volume local de um único Docker Engine. O banco também teria uma mudança operacional importante ao se tornar gerenciado pelo provedor, mas continuaria utilizando PostgreSQL, preservando grande parte das consultas SQL existentes.
