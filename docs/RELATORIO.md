# A3 - Relatório técnico

O ambiente executa a aplicação Achados & Perdidos com Flask/Gunicorn e PostgreSQL, usando volumes para preservar os registros e as fotos. Os arquivos fornecidos pelo professor foram mantidos. Este relatório descreve a configuração atual, incluindo os bônus Nginx e Adminer.

## (a) Dockerfile: instruções e ordem escolhida

Arquivo: [app/Dockerfile](../app/Dockerfile).

| Instrução | Explicação e motivo |
|---|---|
| `FROM python:3.12-slim` | Usa a imagem oficial e enxuta do Python 3.12, conforme exigido na Tarefa 2. |
| `WORKDIR /app` | Define `/app` como diretório de trabalho. Foi escolhido para organizar o código e combinar com o caminho de uploads da aplicação. Os destinos `.` dos comandos COPY referem-se a esse diretório. |
| `COPY requirements.txt .` | Copia a lista de dependências para `/app/requirements.txt` antes de copiar o código, seguindo a ordem exigida para aproveitar o cache. |
| `RUN pip install --no-cache-dir -r requirements.txt` | Instala as dependências e versões fornecidas: Flask, psycopg e Gunicorn. A opção `--no-cache-dir` evita guardar o cache de downloads do pip na imagem, como pede a tarefa. |
| `RUN groupadd appuser` | Cria o grupo `appuser`. O nome é uma escolha de implementação, e o Linux atribui seu identificador numérico automaticamente. |
| `&& useradd --gid appuser --no-create-home appuser` | Cria o usuário `appuser` no grupo anterior, sem pasta pessoal, pois a aplicação não precisa dela. A criação de um usuário sem privilégios atende à tarefa; o nome e a opção sem home foram escolhas de implementação. |
| `&& mkdir -p /app/uploads` | Cria o diretório em que o código fornecido grava as fotos. `-p` também cria diretórios intermediários e aceita que o caminho já exista. |
| `&& chown appuser:appuser /app/uploads` | Define o usuário e o grupo como proprietários do diretório, permitindo gravar fotos sem executar a aplicação como root. Isso atende ao requisito de permissões da Tarefa 2. |
| `COPY --chown=appuser:appuser app.py .` | Copia a aplicação para `/app/app.py` depois de instalar as dependências. A ordem é exigida; definir também o proprietário do código com `--chown` é uma escolha adicional. |
| `USER appuser` | Determina que o processo final do contêiner execute como `appuser`, cumprindo RNF04. Criar o usuário sem selecioná-lo não seria suficiente. |
| `EXPOSE 5000` | Documenta a porta interna da aplicação, conforme exigido. Essa instrução não publica a porta no host. |
| `CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--access-logfile", "-", "app:app"]` | Define o comando de início do contêiner na forma exec, com colchetes. Gunicorn e o bind em `0.0.0.0:5000` são exigidos; `--access-logfile -` foi acrescentado para visualizar os acessos HTTP nos logs. |

O `RUN` prepara a imagem durante o build; o `CMD` define o programa iniciado quando o contêiner é executado. As barras `\` continuam uma instrução na linha seguinte, e `&&` só executa o próximo comando se o anterior tiver sucesso. Assim, a criação do grupo, do usuário e do diretório faz parte de uma única instrução RUN.

No CMD, `0.0.0.0` permite receber conexões pelas interfaces de rede do contêiner, inclusive as vindas do Nginx. Em `app:app`, o primeiro nome identifica o módulo `app.py`, e o segundo é o objeto Flask definido nele. O access log vai para a saída padrão, acessível por `docker compose logs web`; não foram configurados workers extras, portanto o Gunicorn usa sua quantidade padrão de um worker.

### Por que instalar as dependências antes do código?

Se somente `app.py` mudar, o Docker pode reutilizar a etapa que instalou as dependências. Se `requirements.txt` mudar, a cópia desse arquivo e as etapas seguintes serão refeitas. Isso evita reinstalar pacotes a cada alteração do código. O cache de etapas do Docker é diferente do cache de downloads do pip: usar `--no-cache-dir` não impede o reaproveitamento das etapas do build.

O arquivo [app/.dockerignore](../app/.dockerignore) contém somente `.env`, `__pycache__/` e `*.pyc`, as exclusões exigidas. Como o contexto de build é `./app`, o `.env` da raiz também fica fora do contexto.

## (b) Docker Compose: explicação dos blocos

Arquivo: [docker-compose.yml](../docker-compose.yml).

### Identificação do projeto

`name: achados-perdidos` define o nome do projeto. Foi uma escolha para manter nomes previsíveis nos recursos, como `achados-perdidos_fotos` e `achados-perdidos_rede_interna`. O bloco `services` reúne os quatro serviços descritos a seguir; o Adminer é opcional.

### Serviço db

| Bloco | Função |
|---|---|
| `image: postgres:16-alpine` | Usa a imagem oficial exigida para o PostgreSQL. |
| `environment` | Passa `POSTGRES_DB`, `POSTGRES_USER` e `POSTGRES_PASSWORD` a partir das referências `${POSTGRES_DB}`, `${POSTGRES_USER}` e `${POSTGRES_PASSWORD}`, preenchidas pelo `.env` local. |
| `volumes` | Monta `dados_banco` em `/var/lib/postgresql/data` para persistir o banco e disponibiliza `./db/init.sql` em `/docker-entrypoint-initdb.d/init.sql:ro` como bind mount somente leitura. |
| `healthcheck` | Usa `pg_isready` para verificar se o PostgreSQL aceita conexões TCP em `127.0.0.1`. |
| `networks` | Conecta o banco à bridge `rede_interna`, permitindo comunicação pelos nomes dos serviços. |
| `restart: unless-stopped` | Configura reinício automático do processo, respeitando uma parada manual. |

O healthcheck do banco usa intervalo de **5 segundos**, timeout de **3 segundos**, **12 tentativas consecutivas com falha** e período inicial de tolerância de **10 segundos**. `CMD-SHELL` permite expandir as variáveis dentro do contêiner; os dois cifrões em `$$POSTGRES_USER` e `$$POSTGRES_DB` impedem que o Compose faça essa expansão antes de o comando chegar ao shell.

O teste TCP evita liberar o web apenas porque o PostgreSQL temporário da inicialização está disponível por socket. `pg_isready` verifica disponibilidade do servidor; a consulta autenticada feita pela aplicação complementa essa checagem.

Não há `ports` no serviço db. A porta 5432 é usada pela rede Docker, sem publicação no host. O entrypoint da imagem executa `init.sql` na inicialização de um diretório de dados vazio; não repete essa carga em um banco já inicializado.

### Serviço web

| Bloco | Função |
|---|---|
| `build: ./app` | Constrói a imagem pelo Dockerfile da pasta app. |
| `image: achados-perdidos-web:local` | Dá nome e tag à imagem construída. Foi uma escolha para facilitar sua identificação; `local` é apenas a tag. |
| `environment` | Configura `DB_HOST=db`, `DB_PORT=5432`, nome do banco, usuário e senha a partir das mesmas variáveis do PostgreSQL, além de `UPLOAD_DIR=/app/uploads`. |
| `volumes` | Monta o volume nomeado `fotos` em `/app/uploads`, separando as imagens da camada gravável do contêiner. |
| `depends_on` | Com `db: condition: service_healthy`, faz o Compose aguardar o banco saudável antes de iniciar o web, cumprindo RNF05. |
| `healthcheck` | Executa Python para consultar `http://127.0.0.1:5000/health` e exigir `status=ok` e `banco=conectado`. |
| `networks` | Coloca o web na mesma bridge do banco e do proxy. |
| `restart: unless-stopped` | Mantém a política de reinício exigida para o web. |

O código usa o host `db`, resolvido pelo DNS interno do Docker, sem precisar fixar o endereço IP do banco. O `.env` fornece os valores de configuração, mas não é copiado para a imagem nem montado como arquivo nos contêineres. O modelo [.env.example](../.env.example) contém valores fictícios, e [.gitignore](../.gitignore) exclui o `.env` do versionamento.

O healthcheck do web foi uma escolha adicional, pois a tarefa exige explicitamente o do banco. Seu intervalo é de **30 segundos**, com timeout de **3 segundos**, **12 tentativas** e período inicial de tolerância de **5 segundos**. O endpoint verifica a conexão e executa `SELECT 1`; não testa a gravação de fotos. O intervalo vale apenas para a checagem automática: a API continua atendendo os usuários normalmente entre essas chamadas.

### Serviço nginx

| Bloco | Função |
|---|---|
| `image: nginx:1.28-alpine` | Usa uma imagem pronta para o proxy reverso; essa versão/variante foi uma escolha de implementação. |
| `ports: 8080:80` | Encaminha a porta 8080 do host para a porta 80 do Nginx. |
| `volumes` | Monta `./nginx/nginx.conf` em `/etc/nginx/conf.d/default.conf:ro`, mantendo a configuração somente leitura no contêiner. |
| `depends_on` | Aguarda o web saudável antes de iniciar o proxy. Quem verifica essa condição é o Compose. |
| `networks` | Permite encaminhar HTTP ao serviço `web` pela bridge. |
| `restart: unless-stopped` | Aplica ao proxy a mesma política de reinício do ambiente. |

O bônus de proxy altera a publicação básica prevista na prova: o web deixa de publicar `8080:5000`, e o caminho passa a ser **host:8080 → nginx:80 → web:5000**. A aplicação continua usando a porta 5000 internamente.

Em [nginx/nginx.conf](../nginx/nginx.conf), o upstream `aplicacao` aponta para `web:5000`. O resolver `127.0.0.11`, com validade de 5 segundos, usa o DNS Docker; `zone` e `resolve` permitem acompanhar os endereços do serviço. Essa validade de DNS é independente do intervalo de 30 segundos do healthcheck do web. O bloco `location /` encaminha as rotas, preserva informações nos cabeçalhos HTTP e limita o corpo da requisição a 10 MB por `client_max_body_size 10m`.

### Serviço adminer

| Bloco | Função |
|---|---|
| `image: adminer:5-standalone` | Fornece o painel web do banco na versão principal 5, em uma variante que já atende HTTP. A escolha da tag não foi imposta pelo professor. |
| `profiles: ["debug"]` | Faz o painel participar da subida quando o perfil debug é ativado, conforme o bônus. |
| `environment` | `ADMINER_DEFAULT_SERVER=db` indica o servidor padrão no painel; não fornece a senha. |
| `ports: 127.0.0.1:8081:8080` | Publica o painel na porta 8081 somente no loopback do host, como escolha para limitar seu acesso à própria máquina. |
| `depends_on` | Aguarda o banco saudável antes de iniciar o painel. |
| `networks` | Permite que o Adminer alcance `db:5432`. |
| `restart: unless-stopped` | Define sua política de reinício quando ativado. |

O comando `docker compose up -d --build` sobe o ambiente normal sem criar o Adminer. Para incluí-lo, usa-se `docker compose --profile debug up -d`, que também aceita `--build`. Depois de iniciado, o painel precisa ser parado explicitamente; executar outro up sem o perfil não o desliga automaticamente.

### Declarações de rede e volumes

`networks: rede_interna: driver: bridge` cria a rede definida pelo usuário e fornece descoberta por nomes entre os serviços. O nome `rede_interna` não equivale à opção `internal: true`, que não foi habilitada. A [inspeção da rede](evidencias/11-inspecao-rede.png) mostra db, web e nginx conectados.

O bloco de topo `volumes` declara `dados_banco` e `fotos`. Eles são gerenciados pelo Docker e independem da existência dos contêineres. O bind mount de `init.sql` tem outra finalidade: disponibilizar um arquivo do repositório em um caminho específico do banco, em vez de guardar dados produzidos em execução.

As condições de saúde organizam a subida, mas não garantem disponibilidade permanente. Se uma dependência falhar depois, a aplicação poderá apresentar erro; `restart: unless-stopped` não reinicia automaticamente um processo apenas porque o healthcheck passou a indicar `unhealthy`.

## (c) Caminho de POST /itens com foto: do curl ao disco

Exemplo para repetir o cadastro da Tarefa 4. Substitua `C:\caminho\para\sua\foto.jpg` pelo caminho completo de uma imagem JPG ou PNG do seu computador:

```powershell
curl.exe -F "descricao=Oculos de grau" -F "local=Bloco B" -F "foto=@C:\caminho\para\sua\foto.jpg" http://localhost:8080/itens
```

1. **Leitura no host:** o curl lê a imagem do caminho indicado após `@` e monta uma requisição `multipart/form-data` com descrição, local e bytes da imagem.
2. **Entrada HTTP:** a conexão chega a `localhost:8080`. A publicação do Docker a encaminha à porta 80 do Nginx.
3. **Proxy:** o Nginx encontra `web` pelo DNS da bridge e encaminha o corpo e o caminho `/itens` para `web:5000`.
4. **Aplicação:** Gunicorn recebe a requisição e chama o objeto Flask. A rota `cadastrar()` lê o formulário e verifica `descricao` e `local`; se faltar um desses campos, retorna HTTP 400.
5. **Nome do arquivo:** havendo foto, o código preserva a extensão e gera um nome com UUID, reduzindo colisões entre arquivos com o mesmo nome original.
6. **Gravação da foto:** `foto.save()` escreve em `/app/uploads/<nome-gerado>.jpg`. Como o caminho está montado no volume `fotos`, os bytes ficam nesse volume, não apenas na camada do contêiner.
7. **Conexão SQL:** psycopg conecta ao PostgreSQL em `db:5432`, usando as credenciais recebidas por variáveis de ambiente.
8. **Registro no banco:** um INSERT parametrizado grava descrição, local e nome da foto na tabela `itens`, retornando o ID. A imagem não é guardada como bytes na tabela.
9. **Persistência do banco:** ao sair com sucesso do contexto `with conectar()`, a transação é confirmada. O PostgreSQL mantém seus arquivos de dados em `/var/lib/postgresql/data`, montado no volume `dados_banco`.
10. **Disco:** os dois volumes são armazenados pelo daemon Linux do Docker Desktop no disco virtual da máquina. A inspeção mostrou os caminhos `/var/lib/docker/volumes/achados-perdidos_fotos/_data` e `/var/lib/docker/volumes/achados-perdidos_dados_banco/_data` nesse ambiente Linux.
11. **Resposta:** Flask retorna HTTP 201 com ID e nome da foto; a resposta volta pelo Gunicorn, Nginx e publicação Docker até o curl. O access log registra o POST.
12. **Consulta posterior:** GET `/itens` devolve o caminho `/fotos/<nome>`. Ao abrir essa URL, a rota de fotos lê o arquivo do volume e o entrega ao navegador pelo proxy.

As [evidências da Tarefa 4](evidencias/README.md) mostram o cadastro, o aumento das contagens, a listagem, a foto no navegador e o POST com HTTP 201. O arquivo é salvo antes do INSERT: se o banco falhar depois da gravação, pode restar uma foto sem registro. O código fornecido não implementa uma transação conjunta entre banco e sistema de arquivos.

## (d) Ciclo de vida dos volumes: resultados da Tarefa 5

### Criação e uso

Na primeira subida com volumes novos, Docker cria os dois armazenamentos. O banco inicializa seu diretório e executa `init.sql`, criando a tabela e os três exemplos sem foto. Os cadastros posteriores aumentam a quantidade de registros e, quando incluem imagem, também a quantidade de arquivos no volume `fotos`.

O diretório de uploads é preparado com proprietário `appuser` na imagem. Um volume novo é inicializado a partir desse diretório; um volume existente conserva suas próprias permissões. Por isso, mudar os IDs do usuário da imagem pode exigir ajustar a propriedade do volume existente, sem apagar seu conteúdo.

### Resultados observados

| Situação | Antes | Depois | Evidências |
|---|---|---|---|
| `docker compose down` e `docker compose up -d` | 7 itens e 4 fotos. | Permaneceram 7 itens e 4 fotos, com os registros listados e as imagens acessíveis. | [Antes](evidencias/tarefa-5/01-antes-recriacao.png), [depois](evidencias/tarefa-5/03-depois-recriacao.png) e [fotos](evidencias/tarefa-5/04-fotos-apos-recriacao.png). |
| Inspeção dos volumes | 7 itens e 4 fotos. | Dois volumes identificados; quatro arquivos de imagem listados em `/app/uploads`, pertencentes a appuser. | [Inspeção](evidencias/tarefa-5/06b-inspecao-volumes.png) e [arquivos](evidencias/tarefa-5/07-arquivos-volume-fotos.png). |
| `docker compose down -v` e nova subida | 7 itens e 4 fotos. | Restaram somente 3 exemplos e 0 fotos, após criação de novos volumes. | [Antes](evidencias/tarefa-5/08-antes-remocao-volumes.png), [remoção](evidencias/tarefa-5/09-remocao-volumes.png), [nova subida](evidencias/tarefa-5/10a-subida-apos-remocao.png) e [resultado](evidencias/tarefa-5/10b-dados-apos-remocao.png). |

### Por que os resultados são diferentes?

`down` remove os contêineres e a rede, mas preserva os volumes nomeados. Na próxima subida, os novos contêineres montam os mesmos volumes e recuperam o acesso aos dados existentes.

Já `down -v` também remove os dois volumes declarados no projeto. A nova subida usa armazenamento vazio: os cadastros enviados e as fotos desaparecem, e o PostgreSQL executa novamente o `init.sql`. Os três exemplos são **reinseridos pelo script**, não preservados após a remoção. As novas datas de criação observadas na listagem também indicam essa reinicialização.

Os caminhos mostrados em `Mountpoint` pertencem à VM Linux do Docker Desktop; não são pastas comuns do Windows com os mesmos nomes. Os volumes mantêm dados entre recriações de contêineres, mas sua remoção explícita elimina esse estado. Portanto, persistência em volume não substitui backup.
