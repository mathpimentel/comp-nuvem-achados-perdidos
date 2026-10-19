# A4 - Evidências

Saídas reais das Tarefas 4 e 5 executadas por Matheus. Cada linha abaixo contém a legenda da respectiva evidência. As capturas foram preservadas sem edição.

## Tarefa 4 - Subida e testes funcionais

| Evidência | Legenda |
|---|---|
| [01-subida-build.png](tarefa-4/01-subida-build.png) | Build da imagem web simplificada e subida dos serviços pelo Compose. |
| [02-servicos.png](tarefa-4/02-servicos.png) | Serviços ativos: db e web saudáveis; Nginx publicando 8080 e banco sem porta publicada. |
| [03-health-listagem-filtro.png](tarefa-4/03-health-listagem-filtro.png) | API com banco conectado, cinco itens listados e filtro biblio retornando Biblioteca. |
| [04-cadastro-com-foto.png](tarefa-4/04-cadastro-com-foto.png) | POST com zuko.jpg criou o objeto de ID 6 e devolveu o nome da foto armazenada. |
| [05-estatisticas-terminal.png](tarefa-4/05-estatisticas-terminal.png) | Após o cadastro de ID 6, a API retornou seis itens e três fotos no terminal. |
| [06-estatisticas-navegador.png](tarefa-4/06-estatisticas-navegador.png) | Endpoint estatisticas no navegador confirmou seis itens e três fotos. |
| [07-listagem-navegador.png](tarefa-4/07-listagem-navegador.png) | Endpoint itens no navegador mostrou o novo cadastro e o caminho da foto. |
| [08-foto-navegador.png](tarefa-4/08-foto-navegador.png) | GET da foto do cadastro de ID 6 exibiu a imagem do Zuko no navegador. |
| [09-logs-web.png](tarefa-4/09-logs-web.png) | Logs do web registraram GET health com HTTP 200 e POST itens do cadastro seguinte com HTTP 201. |

Os prints 04 a 08 registram o cadastro de ID 6 e o estado de seis itens e três fotos. Depois foi feito outro cadastro com hiei.jpg, cujo POST 201 aparece no print 09; por isso, a Tarefa 5 começa com sete itens e quatro fotos. O access log foi reativado no CMD antes da captura 09; o intervalo periódico do healthcheck do web é de 30 segundos.

## Tarefa 5 - Prova de persistência

| Evidência | Legenda |
|---|---|
| [01-antes-recriacao.png](tarefa-5/01-antes-recriacao.png) | Antes de down: sete itens e quatro fotos, com listagem dos registros existentes. |
| [02-down-sem-remover-volumes.png](tarefa-5/02-down-sem-remover-volumes.png) | Compose down removeu os contêineres e a rede, preservando os volumes nomeados. |
| [03-depois-recriacao.png](tarefa-5/03-depois-recriacao.png) | Após up: serviços saudáveis, mesmos sete itens e quatro fotos e registros preservados. |
| [04-fotos-apos-recriacao.png](tarefa-5/04-fotos-apos-recriacao.png) | Após a recriação, a listagem e as fotos do Zuko e Hiei continuaram acessíveis no navegador. |
| [05-antes-inspecao-volumes.png](tarefa-5/05-antes-inspecao-volumes.png) | Antes da inspeção: estatisticas registrou sete itens e quatro fotos. |
| [06a-listagem-volumes.png](tarefa-5/06a-listagem-volumes.png) | Docker volume ls mostrou os volumes dados_banco e fotos do projeto. |
| [06b-inspecao-volumes.png](tarefa-5/06b-inspecao-volumes.png) | Docker volume inspect mostrou nomes, driver e Mountpoint dos dois volumes no daemon Linux. |
| [07-arquivos-volume-fotos.png](tarefa-5/07-arquivos-volume-fotos.png) | Listagem de /app/uploads mostrou quatro imagens com proprietário e grupo appuser. |
| [08-antes-remocao-volumes.png](tarefa-5/08-antes-remocao-volumes.png) | Antes de down -v: estatisticas registrou sete itens e quatro fotos. |
| [09-remocao-volumes.png](tarefa-5/09-remocao-volumes.png) | Compose down -v removeu contêineres, rede e os volumes fotos e dados_banco. |
| [10a-subida-apos-remocao.png](tarefa-5/10a-subida-apos-remocao.png) | Nova subida recriou os volumes e a rede, com db e web saudáveis e Nginx iniciado. |
| [10b-dados-apos-remocao.png](tarefa-5/10b-dados-apos-remocao.png) | Após down -v e up, a API retornou três exemplos do init.sql e zero fotos, sem os cadastros enviados anteriormente. |

A comparação dos prints 01 e 03 registra sete itens e quatro fotos antes/depois de down/up. O print 04 mostra duas imagens acessíveis após a recriação. Os prints 05 a 07 documentam a inspeção dos volumes. Os prints 08 e 09 registram o estado anterior e a remoção explícita dos volumes. Os prints 10a e 10b comprovam a criação de novos volumes e a reinicialização com três exemplos e nenhuma foto. Os registros novos têm datas de criação diferentes das listagens anteriores: os exemplos foram inseridos novamente pelo init.sql, não preservados após a exclusão dos volumes.

## Evidências adicionais exigidas pelo A4

| Evidência | Legenda |
|---|---|
| [11-inspecao-rede.png](11-inspecao-rede.png) | Network inspect mostrou a bridge rede_interna e os contêineres db, web e nginx conectados. |
| [12-tamanho-imagem.png](12-tamanho-imagem.png) | Image ls mostrou achados-perdidos-web:local com 219 MB de uso em disco e 53,8 MB de conteúdo. |


Não foram reaproveitadas as inspeções ou medidas de tamanho da versão antiga.

## Resultado da Tarefa 5

| Situação | Antes | Depois | Evidências |
|---|---|---|---|
| Recriação sem remover volumes | 7 itens e 4 fotos | 7 itens e 4 fotos, com imagens acessíveis | Tarefa 5: prints 01 a 04. |
| Inspeção | 7 itens e 4 fotos | Dois volumes identificados e quatro arquivos listados | Tarefa 5: prints 05 a 07. |
| Remoção dos volumes e nova subida | 7 itens e 4 fotos | 3 itens de exemplo e 0 fotos | Tarefa 5: prints 08 a 10b. |

As capturas cobrem as três situações exigidas, além das saídas funcionais da Tarefa 4, da inspeção da rede e do tamanho da imagem.

## Evidência da reflexão Q7 - Escala

| Evidência | Legenda |
|---|---|
| [13-escala-web-tres-replicas.png](13-escala-web-tres-replicas.png) | Compose scale web=3 resultou em três réplicas web saudáveis, com db e nginx ativos e sem publicação de portas do web no host. |

## Evidências do bônus Adminer

As sete capturas de ativação, login, consulta SQL e retirada do painel estão no [índice do bônus Adminer](bonus-adminer/README.md), cada uma com legenda de uma linha.

## Evidências do bônus de backup

As quatro capturas e o arquivo fotos.tar.gz estão no [índice do bônus de backup](bonus-backup/README.md), com legendas para a preparação, cópia, restauração em volume novo e comparação dos hashes.

## Respostas da Tarefa 5

**1. Os itens e as fotos continuam após recriar os contêineres?**

Sim. Após `docker compose down` e `docker compose up -d`, permaneceram os **7 itens e 4 fotos**. Os volumes não foram removidos, e os novos contêineres voltaram a acessar os mesmos dados.

**2. Onde os dados ficam fisicamente?**

Nos volumes gerenciados pelo Docker, nos caminhos abaixo do ambiente Linux do Docker Desktop, armazenado no disco virtual da máquina:

- Banco: `/var/lib/docker/volumes/achados-perdidos_dados_banco/_data`.
- Fotos: `/var/lib/docker/volumes/achados-perdidos_fotos/_data`.

Dentro do web, o volume de fotos aparece em `/app/uploads`; a listagem mostrou **quatro arquivos de imagem**.

**3. O que aconteceu após remover os volumes e subir novamente?**

Os cadastros enviados e as fotos foram apagados, pois `down -v` removeu os dois volumes. Na nova subida, o banco vazio executou novamente o `init.sql`, recriando os **3 itens de exemplo, sem fotos**. Eles foram reinseridos pelo script, não preservados após a exclusão.
