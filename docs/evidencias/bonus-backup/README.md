# Evidências do bônus de backup do volume

| Evidência | Legenda |
|---|---|
| [01-estado-antes-backup.png](01-estado-antes-backup.png) | Cadastros com Zuko e Hiei deixaram o ambiente com cinco itens e duas fotos antes do backup. |
| [02-criacao-backup.png](02-criacao-backup.png) | Web pausado para cópia, backup criado por docker run --rm e conteúdo do tar.gz listado com duas fotos. |
| [03-restauracao-volume-novo.png](03-restauracao-volume-novo.png) | Volume novo criado e preenchido pela extração do backup, com duas imagens listadas. |
| [04-comparacao-hashes.png](04-comparacao-hashes.png) | Nomes e hashes SHA256 coincidentes no original e na restauração; estatísticas mantidas em cinco itens e duas fotos. |
| [fotos.tar.gz](fotos.tar.gz) | Arquivo real de backup do volume fotos, com as duas imagens demonstradas. |

O volume de restauração é `achados-perdidos_fotos_restauradas_20261019_170532`. Ele foi mantido separado do volume ativo da aplicação. O arquivo tar.gz contém as fotos, sem os registros do PostgreSQL. O procedimento e os comandos estão em [BONUS.md](../../BONUS.md).
