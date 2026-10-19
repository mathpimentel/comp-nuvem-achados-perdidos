# Evidências do bônus Adminer

| Evidência | Legenda |
|---|---|
| [01-ativacao-perfil-debug.png](01-ativacao-perfil-debug.png) | Adminer ausente na listagem inicial e iniciado com o perfil debug, enquanto o web voltou de três réplicas para uma. |
| [02-adminer-em-execucao.png](02-adminer-em-execucao.png) | Serviço Adminer ativo com publicação local 127.0.0.1:8081 para a porta 8080 do contêiner. |
| [03-formulario-login.png](03-formulario-login.png) | Formulário configurado para PostgreSQL, servidor db e banco achados_perdidos, com senha mascarada. |
| [04-acesso-autenticado-banco.png](04-acesso-autenticado-banco.png) | Login concluído no Adminer e esquema public exibindo a tabela itens e a sequência itens_id_seq. |
| [05-consulta-sql.png](05-consulta-sql.png) | Consulta SELECT dos campos de itens preparada no editor SQL do Adminer, ordenada por ID. |
| [06-resultado-consulta.png](06-resultado-consulta.png) | Consulta executada com sucesso, retornando os três itens de exemplo com foto NULL. |
| [07-painel-removido.png](07-painel-removido.png) | Adminer parado e removido; db, uma réplica web e Nginx continuam em execução. |

A demonstração registrou a ausência inicial do Adminer, sua ativação com o perfil debug, login no PostgreSQL, consulta dos três itens de exemplo e retirada do painel ao final.
