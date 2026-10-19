# Desafios bônus

## 1. Proxy reverso com Nginx (+0,4)

### O que foi solicitado e implementado

O bônus pede um Nginx na frente da aplicação, com publicação da porta 8080 somente no proxy e nenhuma porta publicada pelo web. Essa configuração foi implementada no [docker-compose.yml](../docker-compose.yml) e representada no [diagrama de arquitetura](ARQUITETURA.md).

```yaml
nginx:
  image: nginx:1.28-alpine
  ports:
    - "8080:80"
  volumes:
    - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
  depends_on:
    web:
      condition: service_healthy
  networks:
    - rede_interna
  restart: unless-stopped
```

O serviço web não possui bloco `ports`. Ele continua atendendo na porta 5000 dentro da rede Docker; o `EXPOSE 5000` do Dockerfile apenas documenta essa porta, sem publicá-la no host. O banco também permanece sem porta publicada. O Adminer é um painel opcional em outra porta, 8081, e não publica 8080 no host.

### Como funciona

```text
Navegador/curl
     ↓ HTTP localhost:8080
Nginx:80
     ↓ HTTP web:5000 pela rede_interna
Gunicorn/Flask
     ↓ SQL db:5432, quando a rota consulta o banco
PostgreSQL
```

1. O cliente envia a requisição para `http://localhost:8080`, por exemplo GET `/health` ou POST `/itens`.
2. A publicação `8080:80` encaminha a conexão do host para a porta 80 do contêiner Nginx.
3. O Nginx lê sua configuração e encaminha a requisição para `web:5000`, pela rede `rede_interna`.
4. O Gunicorn entrega a chamada ao Flask, que executa a rota, consulta o banco ou acessa o volume de fotos conforme necessário.
5. A resposta retorna da aplicação pelo Nginx até o cliente, mantendo a URL externa na porta 8080.

O proxy recebe e encaminha o HTTP; as regras de cadastro e acesso ao banco continuam no código Flask. As fotos também são entregues pela rota da aplicação, pois o Nginx não monta diretamente o volume `fotos` nesta configuração.

### Configuração do proxy

O arquivo [nginx/nginx.conf](../nginx/nginx.conf) fica no projeto e é disponibilizado em `/etc/nginx/conf.d/default.conf` por bind mount somente leitura. Seus trechos principais são:

| Trecho | Função |
|---|---|
| `listen 80` | Recebe as conexões na porta interna do Nginx. |
| `upstream aplicacao` e `server web:5000 resolve` | Definem o serviço web como destino das requisições. |
| `resolver 127.0.0.11 valid=5s ipv6=off` | Consulta o DNS interno do Docker para acompanhar os endereços do web. A validade do DNS é independente do healthcheck de 30 segundos. |
| `zone aplicacao 64k` | Mantém o estado compartilhado do upstream, usado na resolução dinâmica dos destinos. |
| `location /` e `proxy_pass http://aplicacao` | Encaminham os caminhos da API ao grupo de destinos configurado. |
| `proxy_set_header` | Encaminha informações como Host, endereço do cliente e protocolo da requisição. |
| `client_max_body_size 10m` | Limita o tamanho do corpo HTTP, incluindo uploads, a 10 MB. |

O Compose aguarda o web saudável antes de iniciar o Nginx. A imagem escolhida, a organização do arquivo na pasta `nginx`, o limite de upload e a resolução dinâmica são decisões de implementação; colocar o proxy na frente, retirar as portas do web e atualizar o diagrama são exigências do bônus.

### Evidências utilizadas

| Evidência | O que demonstra |
|---|---|
| [Serviços da Tarefa 4](evidencias/tarefa-4/02-servicos.png) | O Nginx publica 8080 no host, enquanto o web apresenta apenas 5000/tcp, sem publicação. |
| [Health, listagem e filtro](evidencias/tarefa-4/03-health-listagem-filtro.png) | A API responde pelas URLs externas na porta 8080, conforme o caminho configurado pelo proxy. |
| [Cadastro com foto](evidencias/tarefa-4/04-cadastro-com-foto.png) e [foto no navegador](evidencias/tarefa-4/08-foto-navegador.png) | O acesso pela porta 8080 permite cadastrar e recuperar imagens. |
| [Inspeção da rede](evidencias/11-inspecao-rede.png) | Nginx, web e db estão conectados à mesma bridge. |
| [Três réplicas do web](evidencias/13-escala-web-tres-replicas.png) | O web foi escalado sem disputar a porta 8080, que permaneceu publicada pelo Nginx. |

As evidências existentes cobrem a publicação das portas, o funcionamento da API e a conectividade configurada. A captura de escala comprova três réplicas saudáveis, mas não mede quantas requisições cada uma recebeu.

### Conferência complementar opcional

Para verificar a sintaxe e visualizar acessos diretamente no proxy, podem ser executados:

```powershell
docker compose exec nginx nginx -t
curl.exe http://localhost:8080/health
docker compose logs nginx --tail 20
```

O resultado esperado é a validação da configuração e uma linha de acesso a `/health` no log do Nginx. Esses comandos são uma sugestão de evidência complementar; sua execução ainda não foi registrada nesta seção. O bônus não acrescenta HTTPS ou autenticação: o proxy atual recebe HTTP, e essas proteções permanecem propostas na reflexão.

## 2. Painel do banco com Adminer (+0,3)

### O que foi solicitado e implementado

O bônus pede um serviço Adminer associado ao perfil `debug`, para que o painel não inicie no comando normal de subida. O [docker-compose.yml](../docker-compose.yml) contém:

```yaml
adminer:
  image: adminer:5-standalone
  profiles: ["debug"]
  environment:
    ADMINER_DEFAULT_SERVER: db
  ports:
    - "127.0.0.1:8081:8080"
  depends_on:
    db:
      condition: service_healthy
  networks:
    - rede_interna
  restart: unless-stopped
```

O perfil é exigido pelo bônus. A tag `5-standalone` foi escolhida por fornecer o painel com atendimento HTTP próprio; a publicação em `127.0.0.1` limita o acesso à própria máquina. A porta 8081 evita conflito com a porta 8080 do Nginx.

### Como funciona

O navegador acessa `http://127.0.0.1:8081`, que é encaminhado para a porta 8080 do contêiner Adminer. O painel se conecta ao PostgreSQL usando `db:5432` pela rede `rede_interna`; esse acesso não passa pelo Nginx e não exige publicar a porta do banco no host.

`ADMINER_DEFAULT_SERVER=db` indica o servidor padrão no formulário. Para entrar, foram selecionados PostgreSQL, servidor `db`, usuário e banco configurados no `.env` e a senha local. A captura do formulário mantém a senha mascarada; seu valor não foi registrado neste documento.

### Validação realizada

1. Executou-se `docker compose ps`, mostrando o ambiente sem Adminer ativo.
2. Executou-se `docker compose --profile debug up -d` e depois `docker compose --profile debug ps`, confirmando a subida do painel com a publicação local `127.0.0.1:8081:8080`.
3. Foi realizado o login pelo navegador e visualizado o esquema `public`, com a tabela `itens`.
4. No editor SQL, foi executada a consulta abaixo, que retornou os três itens de exemplo, todos com `foto` igual a `NULL`.
5. O painel foi parado e removido; a listagem final manteve db, web e Nginx funcionando.

```sql
SELECT id, descricao, local, foto, criado_em
FROM itens
ORDER BY id;
```

Na ativação do perfil, o ambiente ainda tinha três réplicas web do teste da Q7. A saída registrou a remoção de web-2 e web-3, voltando à quantidade padrão de uma réplica porque a chamada de up não repetiu `--scale web=3`. Essa redução de contêineres não remove os volumes.

### Evidências

| Evidência | Legenda |
|---|---|
| [Ativação do perfil](evidencias/bonus-adminer/01-ativacao-perfil-debug.png) | Adminer ausente antes da ativação e iniciado pelo comando com perfil debug. |
| [Painel em execução](evidencias/bonus-adminer/02-adminer-em-execucao.png) | Serviço Adminer rodando com publicação restrita a 127.0.0.1:8081. |
| [Formulário de login](evidencias/bonus-adminer/03-formulario-login.png) | Conexão configurada para PostgreSQL em db, com senha mascarada. |
| [Acesso autenticado](evidencias/bonus-adminer/04-acesso-autenticado-banco.png) | Esquema public acessível, exibindo a tabela itens e sua sequência. |
| [Consulta SQL](evidencias/bonus-adminer/05-consulta-sql.png) | Consulta de leitura preparada no editor do painel. |
| [Resultado da consulta](evidencias/bonus-adminer/06-resultado-consulta.png) | SELECT executado com sucesso, retornando os três registros de exemplo. |
| [Retirada do painel](evidencias/bonus-adminer/07-painel-removido.png) | Adminer parado e removido, mantendo os três serviços normais ativos. |

### Como retirar o painel após o uso

```powershell
docker compose --profile debug stop adminer
docker compose --profile debug rm -f adminer
docker compose ps
```

Executar um novo up sem o perfil não desliga automaticamente um Adminer já iniciado, por isso a retirada foi feita explicitamente. Esses comandos removem somente o contêiner do painel, sem apagar os volumes do projeto. O resultado observado confirma a ativação opcional, o acesso autenticado ao banco e a consulta dos registros.

## 3. Backup do volume fotos e restauração (+0,3)

### O que foi solicitado e demonstrado

O bônus pede um backup do volume `fotos` em um arquivo `.tar.gz`, usando `docker run --rm` com o volume montado, e a restauração em um volume novo. A demonstração foi realizada com duas fotos, sem apagar o volume original nem alterar a montagem usada pelo web.

Antes do teste, havia três itens de exemplo e nenhuma foto. Foram cadastrados dois objetos com as imagens Zuko e Hiei, passando a **5 itens e 2 fotos**. Usar duas imagens foi uma escolha para facilitar a comparação; o bônus não determina essa quantidade.

### 1. Criação do backup

Dentro da pasta `achados-perdidos`, a pasta de evidências foi usada como destino. O web foi parado temporariamente para evitar gravar uma foto enquanto ela estivesse sendo copiada:

```powershell
$backupFotosDir = (Resolve-Path docs/evidencias/bonus-backup).Path
docker compose stop web
docker run --rm --mount "type=volume,source=achados-perdidos_fotos,target=/origem,readonly" --mount "type=bind,source=$backupFotosDir,target=/backup" alpine:3.22 tar -czpf /backup/fotos.tar.gz -C /origem .
docker compose start web
```

O primeiro mount disponibilizou o volume original em `/origem`, somente leitura. O segundo apresentou a pasta do Windows em `/backup`, permitindo salvar o arquivo fora do contêiner temporário. O comando tar compactou o conteúdo do volume, e `--rm` removeu o contêiner ao terminar, preservando os volumes e o arquivo no host.

O web foi iniciado novamente após a cópia. Para comprovar a existência do arquivo e listar seu conteúdo, foram executados:

```powershell
Get-Item "$backupFotosDir\fotos.tar.gz"
docker run --rm --mount "type=bind,source=$backupFotosDir,target=/backup,readonly" alpine:3.22 tar -tzvf /backup/fotos.tar.gz
```

O arquivo criado foi [fotos.tar.gz](evidencias/bonus-backup/fotos.tar.gz), com **167.705 bytes**, contendo as duas imagens JPG.

### 2. Restauração em um volume novo

Foi escolhido um nome com data e hora para evitar reutilizar um volume anterior. O volume criado na demonstração foi `achados-perdidos_fotos_restauradas_20261019_170532`.

```powershell
$volumeFotosNovo = "achados-perdidos_fotos_restauradas_" + (Get-Date -Format "yyyyMMdd_HHmmss")
docker volume create $volumeFotosNovo
$backupFotosDir = (Resolve-Path docs/evidencias/bonus-backup).Path
docker run --rm --mount "type=volume,source=$volumeFotosNovo,target=/destino" --mount "type=bind,source=$backupFotosDir,target=/backup,readonly" alpine:3.22 tar -xzpf /backup/fotos.tar.gz -C /destino
docker run --rm --mount "type=volume,source=$volumeFotosNovo,target=/destino,readonly" alpine:3.22 ls -ln /destino
```

O contêiner temporário leu o arquivo de backup e extraiu as fotos em `/destino`, caminho montado no volume novo. A listagem mostrou os dois arquivos com os mesmos nomes e tamanhos do backup. Na repetição do procedimento, o nome do novo volume será diferente por incluir a data e hora do momento.

### 3. Comparação dos arquivos e resultado

Foram calculados os hashes SHA256 nos dois volumes, usando o mesmo caminho interno `/dados` para facilitar a comparação das saídas:

```powershell
docker run --rm --mount "type=volume,source=achados-perdidos_fotos,target=/dados,readonly" alpine:3.22 sh -c 'cd /dados && sha256sum * | sort'
docker run --rm --mount "type=volume,source=$volumeFotosNovo,target=/dados,readonly" alpine:3.22 sh -c 'cd /dados && sha256sum * | sort'
curl.exe http://localhost:8080/estatisticas
```

Os nomes e hashes das duas fotos coincidiram, comprovando a integridade do conteúdo restaurado. A API continuou retornando **5 itens e 2 fotos**, sem alteração das contagens do ambiente original.

| Evidência | Legenda |
|---|---|
| [Estado anterior](evidencias/bonus-backup/01-estado-antes-backup.png) | Dois cadastros com foto prepararam o volume, resultando em cinco itens e duas imagens. |
| [Criação do backup](evidencias/bonus-backup/02-criacao-backup.png) | Contêiner temporário criou fotos.tar.gz e a listagem do arquivo mostrou as duas fotos. |
| [Restauração](evidencias/bonus-backup/03-restauracao-volume-novo.png) | Backup extraído em um volume novo, com duas imagens listadas no destino. |
| [Comparação dos hashes](evidencias/bonus-backup/04-comparacao-hashes.png) | Hashes SHA256 e nomes coincidiram nos dois volumes; a API manteve cinco itens e duas fotos. |

### Explicação das opções e alcance do backup

No tar, `-c` cria o arquivo, `-z` usa gzip, `-f` indica o nome e `-C` escolhe o diretório de origem ou destino. Na restauração, `-x` extrai e `-p` preserva as permissões. `-t` lista o conteúdo sem extrair, e `-v` detalha essa listagem. `readonly` protege a origem contra gravações pelo contêiner auxiliar; a parada do web evita a escrita simultânea pela aplicação durante a cópia.

Esse backup contém **somente as fotos**, não os registros do PostgreSQL. O volume restaurado ficou separado e disponível para inspeção; ele não substituiu o volume usado pela aplicação e não foi necessário editar o Compose. Para recuperar todo o sistema em uma situação real, também seria preciso um backup do banco compatível com o estado das fotos.
