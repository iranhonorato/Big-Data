# Megadados

Material de estudo sobre **bancos de dados relacionais** e **processamento distribuído de dados**, escrito em Markdown e em português.

São textos didáticos, não um software: nada aqui precisa ser compilado ou instalado. Cada arquivo parte de um problema concreto, apresenta os conceitos e os comandos necessários para resolvê-lo e termina com resumo e, quando é o caso, exercícios com gabarito.

---

## Estrutura

```
Megadados/
├── README.md
├── sgbd/                          # Bancos de dados relacionais com MySQL|
|
└── sistemas-distribuidos/
    └── 1_sistemas_distribuidos.md # Partições, MapReduce e Spark
```

---

## Trilha 1: SGBD e SQL (`sgbd/`)

Um roteiro progressivo: cada etapa supõe as anteriores e deixa o banco no estado esperado pela seguinte.

| Etapa | Arquivo | O que se aprende |
|-------|---------|------------------|
| 0 | [my_sql_basics.md](sgbd/my_sql_basics.md) | Banco de dados × SGBD, modelo cliente-servidor, MySQL em Docker, Workbench, persistência e transações |
| 1 | [sql_comandos_1.md](sgbd/sql_comandos_1.md) | Importar o Sakila; `SHOW` e `DESCRIBE`; tipos e restrições; `CREATE`, `INSERT`, `UPDATE`, `DELETE`, `ALTER`, `DROP`; transações |
| 2 | [sql_comandos_2.md](sgbd/sql_comandos_2.md) | `SELECT`, `DISTINCT`, `WHERE`, `LIKE`, `NULL`, `ORDER BY`, `LIMIT` e a ordem lógica de processamento |
| 3 | [sql_comandos_3.md](sgbd/sql_comandos_3.md) | `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `GROUP BY`, `HAVING` e granularidade |
| 4 | [sql_comandos_4.md](sgbd/sql_comandos_4.md) | `INNER`, `LEFT`, `RIGHT` e `CROSS JOIN`, multiplicidade, self join e `FULL OUTER JOIN` simulado |

Todos os exemplos usam o **Sakila**, um banco de exemplo de uma locadora de filmes (filmes, cópias, clientes, aluguéis e pagamentos). O diagrama em [sakila.pdf](sgbd/data/sakila.pdf) ajuda a se localizar entre as tabelas.

### Como os blocos de código estão marcados

| Rótulo | Significado |
|--------|-------------|
| **▶ Executar** | Faz parte do roteiro; execute na ordem |
| **✖ Erro intencional** | Deve falhar; serve para conhecer a mensagem de erro |
| **◇ Ilustrativo** | Não execute; mostra uma ideia ou um formato |
| *(sem rótulo)* | Saída esperada |

Cada bloco também indica onde rodar: no terminal do host, dentro do container ou no cliente MySQL.

### Começo rápido

Pré-requisito: Docker instalado. Os comandos são para bash (Linux, macOS ou WSL). A explicação completa de cada passo está na etapa 0 e na seção 3 da etapa 1.

```bash
# 1. Subir o servidor MySQL (uma única vez)
docker run --name sgbd-mysql \
  -e MYSQL_ROOT_PASSWORD=senha_local \
  -p 127.0.0.1:3306:3306 \
  -v sgbd-mysql-dados:/var/lib/mysql \
  -d mysql:8.0.45

# 2. A partir da pasta sgbd/, importar o Sakila (estrutura primeiro, depois os dados)
cd sgbd
docker exec -i sgbd-mysql mysql --default-character-set=utf8mb4 -uroot -psenha_local < data/sakila-schema.sql
docker exec -i sgbd-mysql mysql --default-character-set=utf8mb4 -uroot -psenha_local < data/sakile.sql

# 3. Abrir o cliente
docker exec -it sgbd-mysql mysql --default-character-set=utf8mb4 -uroot -p
```

> **Atenção:**
> * Se já existir um container chamado `sgbd-mysql`, não rode o passo 1: use `docker start sgbd-mysql`.
> * `sakila-schema.sql` apaga e recria o banco `sakila`. Qualquer alteração feita nele antes se perde.
> * A senha `senha_local` serve só para um ambiente local de estudo. A porta fica restrita a `127.0.0.1`.

### Versão de referência e validação

Os materiais foram escritos e testados com a imagem oficial `mysql:8.0.45`. Os blocos **▶ Executar** das etapas 1 a 4 rodaram em sequência, sem erros, num Sakila recém-importado, e as saídas mostradas vêm dessas execuções. O que foi executado, o que foi conferido só por leitura e o que ficou pendente está em [revisao_materiais.md](sgbd/revisao_materiais.md).

---

## Trilha 2: Sistemas distribuídos (`sistemas-distribuidos/`)

[1_sistemas_distribuidos.md](sistemas-distribuidos/1_sistemas_distribuidos.md) trata do que acontece quando os dados não cabem mais em uma única máquina:

1. **Problema:** os limites de uma consulta SQL em um servidor só e os novos problemas que a distribuição traz.
2. **Partições:** dividir os dados entre máquinas e as limitações dessa divisão.
3. **MapReduce:** as etapas *map*, *shuffle* e *reduce*, com um exemplo passo a passo.
4. **Spark:** cluster, DataFrame, PySpark, avaliação preguiçosa, agregação parcial antes do shuffle, jobs/stages/tasks, cache e tolerância a falhas.

A trilha de SQL ajuda a ler esta: separar linhas por chave e calcular um resultado por grupo, como no `GROUP BY` da etapa 3, lembra o *shuffle* e o *reduce*. É uma analogia de ideias, não uma descrição de como o MySQL executa a consulta.

---

## Como ler

* **Do zero:** siga a trilha de SGBD na ordem (0 → 4), executando os blocos **▶**, e depois leia sistemas distribuídos.
* **Para consulta:** cada arquivo abre com um "contrato" (objetivos, conhecimentos prévios, estado inicial e final) e fecha com um resumo, o que permite ir direto ao tópico.
* **Para praticar:** os exercícios ficam no fim de cada etapa de SQL. Os gabaritos estão recolhidos (`<details>`) para você tentar antes de conferir.
