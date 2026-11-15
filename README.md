# Megadados

Anotações de estudo sobre **bancos de dados relacionais** e **processamento distribuído de dados**, em português.

É um material de leitura e prática, não um software. Cada texto parte de um problema concreto, apresenta os conceitos e os comandos que o resolvem e fecha com um resumo.

## Conteúdo

```
Megadados/
├── sgbd/                    # Bancos de dados relacionais com MySQL e SQL
│   └── data/                # Banco de exemplo Sakila (estrutura, dados e diagrama)
└── sistemas-distribuidos/   # Partições, MapReduce e Spark
```

### SGBD e SQL

Um roteiro em cinco etapas, em ordem crescente de dificuldade. Cada etapa supõe as anteriores.

| Etapa | Arquivo | Tema |
|:-----:|---------|------|
| 0 | [my_sql_basics.md](sgbd/my_sql_basics.md) | Banco de dados e SGBD, modelo cliente-servidor, MySQL com Docker, Workbench |
| 1 | [sql_comandos_1.md](sgbd/sql_comandos_1.md) | Importar o banco, investigar tabelas, `CREATE`, `INSERT`, `UPDATE`, `DELETE`, `ALTER`, `DROP` |
| 2 | [sql_comandos_2.md](sgbd/sql_comandos_2.md) | Consultas em uma tabela: `SELECT`, `DISTINCT`, `WHERE`, `LIKE`, `ORDER BY`, `LIMIT` |
| 3 | [sql_comandos_3.md](sgbd/sql_comandos_3.md) | Agregações: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `GROUP BY`, `HAVING` |
| 4 | [sql_comandos_4.md](sgbd/sql_comandos_4.md) | Combinação de tabelas: `INNER`, `LEFT`, `RIGHT` e `CROSS JOIN` |

Os exemplos usam o **Sakila**, banco de exemplo de uma locadora de filmes, com dados fictícios. O diagrama das tabelas está em [sakila.pdf](sgbd/data/sakila.pdf).

### Sistemas distribuídos

[1_sistemas_distribuidos.md](sistemas-distribuidos/1_sistemas_distribuidos.md) trata do que muda quando os dados não cabem em uma máquina só: partições, o modelo MapReduce (*map*, *shuffle*, *reduce*) e o Spark (DataFrame, PySpark, avaliação preguiçosa, cache e tolerância a falhas).

## Como usar

Para acompanhar a parte de SQL, você precisa de Docker e de um terminal bash (Linux, macOS ou WSL). Os comandos abaixo sobem o MySQL e importam o Sakila. O passo a passo comentado está na etapa 0 e na seção 3 da etapa 1.

```bash
# Subir o servidor MySQL (só na primeira vez; depois, use: docker start sgbd-mysql)
docker run --name sgbd-mysql \
  -e MYSQL_ROOT_PASSWORD=senha_local \
  -p 127.0.0.1:3306:3306 \
  -v sgbd-mysql-dados:/var/lib/mysql \
  -d mysql:8.0.45

# Importar o Sakila, a partir da pasta sgbd/
cd sgbd
docker exec -i sgbd-mysql mysql --default-character-set=utf8mb4 -uroot -psenha_local < data/sakila-schema.sql
docker exec -i sgbd-mysql mysql --default-character-set=utf8mb4 -uroot -psenha_local < data/sakile.sql

# Abrir o cliente MySQL
docker exec -it sgbd-mysql mysql --default-character-set=utf8mb4 -uroot -p
```

> `senha_local` é uma senha de exemplo, para uso só no seu computador. A importação apaga e recria o banco `sakila`, se ele já existir.

Nos textos, os blocos de código vêm marcados:

* **▶ Executar**: faz parte do roteiro; rode na ordem.
* **✖ Erro intencional**: deve falhar, para você conhecer a mensagem.
* **◇ Ilustrativo**: não rode; mostra só uma ideia ou um formato.

Cada etapa de SQL termina com exercícios. As respostas ficam recolhidas, para você tentar antes de conferir.

Os comandos foram testados com a imagem `mysql:8.0.45`.
