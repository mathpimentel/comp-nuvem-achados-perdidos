# Achados & Perdidos

## Objetivo

Empacotar a aplicação de achados e perdidos em contêineres, permitindo subir a API e o banco com `docker compose up -d --build`. Os cadastros e as fotos ficam em volumes nomeados, preservados quando os contêineres são recriados sem remover os volumes.

A aplicação Flask fornecida é executada pelo Gunicorn e usa PostgreSQL 16. O Nginx encaminha as requisições recebidas na porta 8080 para a API.

Os arquivos fornecidos pelo professor, mantidos sem alterações, são:

- `app/app.py`: código da API Flask.
- `app/requirements.txt`: dependências Python da aplicação.
- `db/init.sql`: criação da tabela e inserção dos três itens de exemplo.

## Estrutura atual

```text
achados-perdidos/
|-- app/
|   |-- app.py                 # fornecido pelo professor
|   |-- requirements.txt       # fornecido pelo professor
|   |-- Dockerfile
|   `-- .dockerignore
|-- db/
|   `-- init.sql               # fornecido pelo professor
|-- nginx/
|   `-- nginx.conf
|-- docs/
|   |-- ARQUITETURA.md
|   |-- RELATORIO.md
|   |-- REFLEXAO.md
|   |-- BONUS.md
|   `-- evidencias/            # capturas, legendas e backup demonstrado
|-- .env                       # configuração local; não entregar
|-- .env.example               # modelo com valores fictícios
|-- .gitignore
|-- docker-compose.yml
`-- README.md
```

As capturas e suas legendas estão organizadas no [índice das evidências](docs/evidencias/README.md). A pasta `docs/` reúne os artefatos explicativos do projeto.

## Pré-requisitos

- Docker Desktop em execução, configurado para contêineres Linux e com Docker Compose disponível. Em Linux, Docker Engine e o plugin Compose equivalentes.
- Internet na primeira construção para baixar imagens e dependências.
- Porta 8080 disponível; porta 8081 disponível se for ativar o Adminer.
- PowerShell e `curl.exe` para seguir os comandos Windows abaixo. Não é necessário instalar Python ou PostgreSQL no host.

Execute os comandos na pasta `achados-perdidos`, onde está `docker-compose.yml`.

## 1. Configurar o .env

Em uma instalação nova, copie o modelo e edite a cópia:

```powershell
Copy-Item .env.example .env
notepad .env
```

O modelo contém somente valores fictícios:

```dotenv
POSTGRES_DB=nome_db
POSTGRES_USER=user_db
POSTGRES_PASSWORD=senha_exemplo
```

Escolha o nome do banco, o usuário e uma senha própria antes da primeira subida. Mantenha os nomes das três variáveis. Para o código fornecido, use uma senha sem espaços, aspas simples ou barras invertidas, pois a conexão é montada por concatenação de texto.

O Compose usa esses valores tanto no banco quanto na configuração de conexão do web. O host do banco continua sendo o serviço `db`, independentemente do nome escolhido em `POSTGRES_DB`.

O `.env` é local, está no `.gitignore` e não deve ser entregue. Se o seu ambiente já está configurado, mantenha seu `.env` existente: não é necessário sobrescrevê-lo para usar o novo modelo. Alterar essas variáveis depois de inicializar o banco não renomeia automaticamente o banco, não recria usuários e não troca a senha já armazenada no volume.

## 2. Construir e subir

```powershell
docker compose config --quiet
docker compose up -d --build
docker compose ps
```

`--build` constrói a imagem web e `-d` deixa os serviços em segundo plano. O banco precisa ficar saudável antes do início do web; o Nginx aguarda o web saudável. Se ainda aparecer `health: starting`, espere alguns segundos e repita `docker compose ps`.

No comando normal, ficam ativos db, web e nginx. O Adminer só é incluído quando o perfil debug é ativado.

| Serviço | Função | Porta no contêiner | Publicação no host |
|---|---|---|---|
| web | Flask/Gunicorn, usando a imagem própria | 5000 | Nenhuma |
| db | PostgreSQL 16 | 5432 | Nenhuma |
| nginx | Proxy reverso da aplicação | 80 | `8080:80` |
| adminer | Painel opcional do banco | 8080 | `127.0.0.1:8081:8080`, no perfil debug |

O acesso normal segue **navegador/curl → localhost:8080 → Nginx:80 → web:5000**. O banco é acessado em `db:5432` pela rede `rede_interna`. Os bind mounts de configuração são `db/init.sql` e `nginx/nginx.conf`, ambos somente leitura dentro dos respectivos contêineres.

Na primeira inicialização com volume vazio, `init.sql` cria a tabela e insere três objetos sem foto. Subidas posteriores reutilizam os dados existentes e não repetem a carga inicial.

## 3. Testar a aplicação

### Saúde, listagem e filtro

```powershell
curl.exe http://localhost:8080/health
curl.exe http://localhost:8080/itens
curl.exe "http://localhost:8080/itens?local=biblio"
```

O health deve retornar `status: "ok"` e `banco: "conectado"`, além do identificador do contêiner. A listagem mostra os registros existentes; o filtro `biblio` encontra o exemplo da Biblioteca. Também é possível abrir [a listagem no navegador](http://localhost:8080/itens).

### Cadastro com foto

Escolha uma imagem JPG ou PNG do seu computador e substitua `C:\caminho\para\sua\foto.jpg` pelo caminho completo do arquivo antes de executar:

```powershell
curl.exe -i -F "descricao=Oculos de grau" -F "local=Bloco B" -F "foto=@C:\caminho\para\sua\foto.jpg" http://localhost:8080/itens
curl.exe http://localhost:8080/estatisticas
```

`-i` mostra também os cabeçalhos e o status HTTP. O resultado esperado do cadastro é HTTP 201 e JSON com `id` e `foto`. O `@` indica que o curl deve ler e enviar o arquivo local; mantenha-o antes do caminho, dentro das aspas, inclusive quando o caminho tiver espaços.

Cada POST cria um novo registro; com foto, também cria outro arquivo no volume. Em uma base nova, o primeiro cadastro com imagem passa de 3 itens/0 fotos para 4 itens/1 foto. Em um ambiente já usado, compare com as contagens anteriores, pois os cadastros se acumulam.

### Logs e usuário

```powershell
docker compose logs web --tail 20
docker compose exec web whoami
```

O usuário esperado é `appuser`, sem root. O Gunicorn está configurado com access log, portanto podem aparecer GETs, POSTs e seus códigos HTTP. Chamadas internas a `/health` são verificações automáticas do web, configuradas com intervalo de 30 segundos; elas não limitam a frequência das requisições dos usuários.

### Rotas da API

| Método | Rota | Entrada e resultado |
|---|---|---|
| GET | `/health` | Consulta o banco com SELECT 1; retorna HTTP 200 com estado ok/conectado, ou 503 em caso de falha. |
| GET | `/itens` | Lista os objetos cadastrados, ordenados por data de criação decrescente. |
| GET | `/itens?local=texto` | Filtra por trecho do local, sem diferenciar maiúsculas e minúsculas. |
| POST | `/itens` | Recebe formulário multipart com `descricao` e `local` obrigatórios e `foto` opcional; retorna 201 com ID e nome da foto, ou 400 se faltar campo obrigatório. |
| GET | `/fotos/<nome>` | Entrega o arquivo de imagem; retorna 404 se ele não existir. |
| GET | `/estatisticas` | Retorna `itens_no_banco` e `fotos_no_volume`. |

## 4. Parar, retomar e recriar

Para pausar e retomar os contêineres existentes:

```powershell
docker compose stop
docker compose start
```

Para remover os contêineres e a rede, preservando os dados, e depois subir novamente:

```powershell
docker compose down
docker compose up -d
```

| Volume | Caminho montado | Conteúdo |
|---|---|---|
| `dados_banco` | `/var/lib/postgresql/data` no db | Arquivos do PostgreSQL, incluindo os registros da aplicação. |
| `fotos` | `/app/uploads` no web | Imagens enviadas pelos usuários. |

Os nomes físicos têm o prefixo `achados-perdidos_`. Para verificar a persistência, consulte `/estatisticas` e `/itens` antes e depois de down/up e abra novamente as fotos. Para repetir a prova da Tarefa 5, tenha pelo menos dois objetos com foto e registre as estatísticas antes de cada situação. Nas evidências, 7 itens e 4 fotos sobreviveram à recriação.

## 5. Destruir o ambiente e seus dados

**O comando abaixo remove os contêineres, a rede e os dois volumes do projeto, apagando os cadastros e as fotos atuais:**

```powershell
docker compose down -v
```

Para iniciar novamente:

```powershell
docker compose up -d
```

O PostgreSQL inicializa o volume novo e executa `init.sql`: voltam os três exemplos, com zero fotos. Os exemplos são recriados pelo script; os dados enviados antes não sobrevivem à exclusão dos volumes.

Se o Adminer estiver ativo, use `docker compose --profile debug down` para incluir o painel na remoção dos contêineres, ou `docker compose --profile debug down -v` para incluir também a exclusão dos volumes. A imagem construída e os arquivos do projeto permanecem. O arquivo de backup no host e o volume de restauração criado separadamente no bônus também não são apagados por essa remoção dos volumes declarados no Compose.

## Documentação e evidências

| Artefato | Arquivo |
|---|---|
| A1 - Operação e apresentação do projeto | Este README. |
| A2 - Diagrama Mermaid com host, rede, portas, contêineres e volumes | [docs/ARQUITETURA.md](docs/ARQUITETURA.md) |
| A3 - Dockerfile, Compose, fluxo de POST e ciclo de vida dos volumes | [docs/RELATORIO.md](docs/RELATORIO.md) |
| A4 - Saídas das Tarefas 4 e 5, rede, tamanho da imagem e provas complementares | [docs/evidencias/README.md](docs/evidencias/README.md) |
| A5 - Sete reflexões e tabela de equivalência em nuvem | [docs/REFLEXAO.md](docs/REFLEXAO.md) |

As capturas possuem legendas e registram os estados de cada etapa. As contagens mudam durante os testes: novos cadastros aumentam os totais, down/up os preserva e down -v os reinicializa.

## Limites do protótipo

O acesso atual é HTTP e a aplicação fornecida não implementa autenticação nem ocultação de dados nas fotos. Essas proteções foram propostas na reflexão, não acrescentadas ao código original. O nome `rede_interna` identifica uma bridge própria e não significa `internal: true`. A gravação de uma foto e o INSERT no banco não formam uma transação única, e volumes locais não substituem uma estratégia de backup.
