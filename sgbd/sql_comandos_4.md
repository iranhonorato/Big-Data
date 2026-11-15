# SQL — Nível 4: Combinando tabelas com JOIN

| Etapa | Arquivo | Tema |
|-------|---------|------|
| 0 | [my_sql_basics.md](my_sql_basics.md) | Fundamentos: banco, SGBD, cliente/servidor, ambiente com Docker e primeira conexão |
| 1 | [sql_comandos_1.md](sql_comandos_1.md) | Carregar o Sakila; investigar estruturas; criar, alterar e remover tabelas; inserir e editar dados |
| 2 | [sql_comandos_2.md](sql_comandos_2.md) | Consultas em uma tabela: `SELECT`, `DISTINCT`, `WHERE`, `LIKE`, `ORDER BY`, `LIMIT` |
| 3 | [sql_comandos_3.md](sql_comandos_3.md) | Agregações: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `GROUP BY`, `HAVING` |
| **4** | **[sql_comandos_4.md](sql_comandos_4.md)** | **Combinar tabelas: `INNER`, `LEFT`, `RIGHT` e `CROSS JOIN`, multiplicidade, `FULL` simulado ← você está aqui** |

> **Contrato deste arquivo**
>
> * **Objetivo:** combinar tabelas sabendo **quantas linhas** o resultado terá e **o que cada linha representa**.
> * **Conhecimentos prévios:** níveis [2](sql_comandos_2.md) e [3](sql_comandos_3.md): filtros, `NULL`, agregações, `GROUP BY`, `HAVING`, granularidade.
> * **Estado inicial:** `sakila` no estado original. Execute `USE sakila;`.
> * **Ao final, você conseguirá:** (1) escrever `INNER` e `LEFT JOIN` com apelidos e colunas qualificadas; (2) prever quando um JOIN multiplica linhas; (3) contar corretamente com `LEFT JOIN` e encontrar linhas sem correspondência; (4) decidir se uma condição vai no `ON` ou no `WHERE`; (5) evitar somas infladas por JOIN; (6) montar um relatório com várias tabelas e explicar cada linha.
> * **Estado final:** o `sakila` volta ao original; as tabelas `demo_*` criadas na seção 3 são removidas na seção 11.
> * **Próximo passo:** subconsultas, views e índices (fora deste curso).

> Rótulos dos blocos: veja os [Fundamentos](my_sql_basics.md). Ambiente de referência: `mysql:8.0.45`. Todos os comandos rodam **no cliente MySQL**.

---

## 1. Problema

O nível 3 terminou com uma lista de clientes que mais pagaram, identificados só por número: `customer_id = 526`. Quem é ele? A tabela `payment` guarda apenas o **número** do cliente; o nome está em `customer`.

### 1.1 Por que os dados estão separados?

Imagine guardar o nome do cliente em **cada pagamento**. O nome se repetiria dezenas de vezes; para corrigir um sobrenome, seria preciso alterar todas essas linhas, e bastaria esquecer uma para os dados se contradizerem (uma **anomalia de atualização**).

**Normalização** é organizar as tabelas de modo que cada **fato** dependa da chave certa: o nome depende do cliente (`customer`), o valor depende do pagamento (`payment`). As tabelas se ligam por chaves, e a redundância e as anomalias diminuem.

> O preço: para responder perguntas que envolvem vários assuntos, é preciso **juntar** as tabelas na consulta. Para isso existe o `JOIN`.

---

## 2. O primeiro JOIN

Antes de rankings e somas, um passo simples: pôr o nome do cliente ao lado de cada pagamento dele.

**▶ Executar**

```sql
SELECT p.payment_id,
       p.payment_date,
       p.amount,
       c.first_name,
       c.last_name
FROM payment p
INNER JOIN customer c ON c.customer_id = p.customer_id
WHERE p.customer_id = 526
ORDER BY p.payment_id
LIMIT 3;
```

```text
+------------+---------------------+--------+------------+-----------+
| payment_id | payment_date        | amount | first_name | last_name |
+------------+---------------------+--------+------------+-----------+
|      14136 | 2005-05-28 00:40:48 |   4.99 | KARL       | SEAL      |
|      14137 | 2005-05-28 23:24:57 |   4.99 | KARL       | SEAL      |
|      14138 | 2005-05-31 02:44:57 |   2.99 | KARL       | SEAL      |
+------------+---------------------+--------+------------+-----------+
```

O cliente 526 é KARL SEAL. Cada linha do resultado representa **um pagamento**, agora acompanhado dos dados do cliente que o fez.

| Parte | Significado |
|-------|-------------|
| `FROM payment p` | Tabela da **esquerda**, com o apelido `p` |
| `INNER JOIN customer c` | Tabela da **direita**, com o apelido `c` |
| `ON c.customer_id = p.customer_id` | **Condição de correspondência**: quais linhas combinam |
| `p.amount`, `c.first_name` | Colunas **qualificadas** pelo apelido da tabela |

Os **apelidos** (`p`, `c`) encurtam a consulta e resolvem colunas com o mesmo nome em duas tabelas:

**✖ Erro intencional**

```sql
SELECT title FROM film JOIN film_actor ON film_id = film_id;
```

```text
ERROR 1052 (23000): Column 'film_id' in on clause is ambiguous
```

`film` e `film_actor` têm uma coluna `film_id`. Com apelidos, não há dúvida: `ON fa.film_id = f.film_id`.

> Boa prática: em consultas com várias tabelas, qualifique **todas** as colunas.

**Observação:** o `JOIN` compara valores; ele **não exige** uma `FOREIGN KEY` declarada. Neste curso, porém, quase sempre juntamos pela chave estrangeira e pela chave primária correspondente, que é o caso mais comum e o mais seguro.

---

## 3. Multiplicidade: quantas linhas sai de um JOIN?

A imagem "colar colunas lado a lado" engana: um JOIN pode **multiplicar** ou **eliminar** linhas. Isso fica claro em tabelas pequenas.

### 3.1 Três tabelas de demonstração

**▶ Executar** (tabelas comuns, removidas na seção 11)

```sql
CREATE TABLE demo_cliente   (cliente_id INT PRIMARY KEY, nome VARCHAR(20) NOT NULL);
CREATE TABLE demo_pagamento (pagamento_id INT PRIMARY KEY, cliente_id INT NOT NULL, valor DECIMAL(5,2) NOT NULL);
CREATE TABLE demo_telefone  (telefone_id INT PRIMARY KEY, cliente_id INT NOT NULL, numero VARCHAR(20) NOT NULL);

INSERT INTO demo_cliente   VALUES (1, 'Ana'), (2, 'Bruno'), (3, 'Carla');
INSERT INTO demo_pagamento VALUES (10, 1, 5.00), (11, 1, 5.00), (12, 2, 4.00), (13, 9, 2.00);
INSERT INTO demo_telefone  VALUES (100, 1, '1111-1111'), (101, 1, '2222-2222'), (102, 2, '3333-3333');
```

| `demo_cliente` | | `demo_pagamento` | | | `demo_telefone` | | |
|---|---|---|---|---|---|---|---|
| **cliente_id** | **nome** | **pagamento_id** | **cliente_id** | **valor** | **telefone_id** | **cliente_id** | **numero** |
| 1 | Ana | 10 | 1 | 5.00 | 100 | 1 | 1111-1111 |
| 2 | Bruno | 11 | 1 | 5.00 | 101 | 1 | 2222-2222 |
| 3 | Carla | 12 | 2 | 4.00 | 102 | 2 | 3333-3333 |
| | | 13 | **9** | 2.00 | | | |

Repare: Ana tem **dois** pagamentos; Bruno, **um**; Carla, **nenhum**; e o pagamento 13 é de um cliente 9 que não está cadastrado. (Sem chave estrangeira declarada, o MySQL aceitou esse 9. É de propósito, para o exemplo.)

### 3.2 `INNER JOIN`: só os pares que combinam

**▶ Executar**

```sql
SELECT c.cliente_id, c.nome, p.pagamento_id, p.valor
FROM demo_cliente c
INNER JOIN demo_pagamento p ON p.cliente_id = c.cliente_id
ORDER BY c.cliente_id, p.pagamento_id;
```

```text
+------------+-------+--------------+-------+
| cliente_id | nome  | pagamento_id | valor |
+------------+-------+--------------+-------+
|          1 | Ana   |           10 |  5.00 |
|          1 | Ana   |           11 |  5.00 |
|          2 | Bruno |           12 |  4.00 |
+------------+-------+--------------+-------+
```

| Cliente | Correspondências | Linhas no resultado |
|---------|------------------|---------------------|
| Ana | 2 | 2 (Ana **aparece duas vezes**) |
| Bruno | 1 | 1 |
| Carla | 0 | 0 (**some**) |
| Pagamento 13 (cliente 9) | 0 | 0 (**some**) |

Cada linha representa **um par (cliente, pagamento) que combina**. São 3 clientes e 4 pagamentos, e o resultado tem 3 linhas.

### 3.3 `LEFT JOIN`: todas as linhas da esquerda

**▶ Executar**

```sql
SELECT c.cliente_id, c.nome, p.pagamento_id, p.valor
FROM demo_cliente c
LEFT JOIN demo_pagamento p ON p.cliente_id = c.cliente_id
ORDER BY c.cliente_id, p.pagamento_id;
```

```text
+------------+-------+--------------+-------+
| cliente_id | nome  | pagamento_id | valor |
+------------+-------+--------------+-------+
|          1 | Ana   |           10 |  5.00 |
|          1 | Ana   |           11 |  5.00 |
|          2 | Bruno |           12 |  4.00 |
|          3 | Carla |         NULL |  NULL |
+------------+-------+--------------+-------+
```

Carla agora aparece, com as colunas da direita preenchidas com `NULL`. "Manter todas as linhas da esquerda" **não** significa manter a mesma quantidade: 3 clientes viraram 4 linhas, porque Ana continua aparecendo uma vez por pagamento.

### 3.4 Contando com `LEFT JOIN`

**▶ Executar**

```sql
SELECT c.cliente_id, c.nome,
       COUNT(*)              AS linhas_do_resultado,
       COUNT(p.pagamento_id) AS pagamentos
FROM demo_cliente c
LEFT JOIN demo_pagamento p ON p.cliente_id = c.cliente_id
GROUP BY c.cliente_id, c.nome
ORDER BY c.cliente_id;
```

```text
+------------+-------+---------------------+------------+
| cliente_id | nome  | linhas_do_resultado | pagamentos |
+------------+-------+---------------------+------------+
|          1 | Ana   |                   2 |          2 |
|          2 | Bruno |                   1 |          1 |
|          3 | Carla |                   1 |          0 |
+------------+-------+---------------------+------------+
```

`COUNT(*)` conta **linhas do resultado**: Carla tem uma (a linha com `NULL`). `COUNT(p.pagamento_id)` conta **pagamentos correspondentes**: zero, porque a chave da direita é `NULL`.

> Para contar correspondências à direita, conte uma coluna da direita que nunca é `NULL` em uma correspondência real (a chave primária é a escolha natural). `COUNT(*)` continua certo quando o que você quer são as linhas do resultado.

E para encontrar **quem não tem par** (um *anti-join*):

**▶ Executar**

```sql
SELECT c.cliente_id, c.nome
FROM demo_cliente c
LEFT JOIN demo_pagamento p ON p.cliente_id = c.cliente_id
WHERE p.pagamento_id IS NULL;
```

```text
+------------+-------+
| cliente_id | nome  |
+------------+-------+
|          3 | Carla |
+------------+-------+
```

Teste `IS NULL` em uma coluna que **nunca** é nula quando há correspondência, como a chave primária. Se testasse uma coluna opcional (que pode ser `NULL` mesmo em pagamentos reais), você confundiria "não tem pagamento" com "tem pagamento, mas sem esse dado".

### 3.5 Cuidado: JOIN pode inflar somas

Quanto cada cliente pagou? Se, sem necessidade, juntarmos também os telefones:

**✖ Erro intencional** (de lógica: executa, mas o resultado está errado)

```sql
SELECT c.cliente_id, c.nome,
       COUNT(*)                       AS linhas,
       SUM(p.valor)                   AS soma_inflada,
       SUM(DISTINCT p.valor)          AS soma_distinct,
       COUNT(DISTINCT p.pagamento_id) AS pagamentos
FROM demo_cliente c
INNER JOIN demo_pagamento p ON p.cliente_id = c.cliente_id
INNER JOIN demo_telefone t  ON t.cliente_id = c.cliente_id
GROUP BY c.cliente_id, c.nome
ORDER BY c.cliente_id;
```

```text
+------------+-------+--------+--------------+---------------+------------+
| cliente_id | nome  | linhas | soma_inflada | soma_distinct | pagamentos |
+------------+-------+--------+--------------+---------------+------------+
|          1 | Ana   |      4 |        20.00 |          5.00 |          2 |
|          2 | Bruno |      1 |         4.00 |          4.00 |          1 |
+------------+-------+--------+--------------+---------------+------------+
```

Ana pagou 10.00 (5.00 + 5.00), mas a soma deu **20.00**. Por quê? Ela tem 2 pagamentos **e** 2 telefones, e o JOIN gerou todas as combinações: 2 × 2 = 4 linhas, cada pagamento repetido uma vez por telefone.

| Tentativa de correção | Resultado para Ana | Funciona? |
|-----------------------|--------------------|-----------|
| `SUM(p.valor)` | 20.00 | Não: conta cada pagamento duas vezes |
| `SUM(DISTINCT p.valor)` | 5.00 | **Não**: os dois pagamentos têm o mesmo valor e viraram um só |
| `COUNT(DISTINCT p.pagamento_id)` | 2 | Sim, para **contar** pagamentos (o ID distingue cada um) |
| Não juntar a tabela de telefones | 10.00 | Sim: ela não participa da pergunta |

**▶ Executar** — a forma correta

```sql
SELECT c.cliente_id, c.nome, SUM(p.valor) AS total
FROM demo_cliente c
INNER JOIN demo_pagamento p ON p.cliente_id = c.cliente_id
GROUP BY c.cliente_id, c.nome
ORDER BY c.cliente_id;
```

```text
+------------+-------+-------+
| cliente_id | nome  | total |
+------------+-------+-------+
|          1 | Ana   | 10.00 |
|          2 | Bruno |  4.00 |
+------------+-------+-------+
```

> Antes de somar depois de um JOIN, pergunte: **cada linha de pagamento aparece uma única vez?** Juntar duas tabelas "filhas" (1:N) do mesmo pai multiplica as linhas. Quando as duas informações são necessárias, cada uma deve ser agregada separadamente, o que exige subconsultas (fora deste curso).

> **✅ Verifique.** Quantas linhas tem `demo_cliente LEFT JOIN demo_telefone`? *Resposta:* 4 (Ana 2, Bruno 1, Carla 1 com `NULL`).

---

## 4. `INNER JOIN` no Sakila

### 4.1 O ranking do nível 3, com nomes

**▶ Executar**

```sql
SELECT c.first_name,
       c.last_name,
       COUNT(*)      AS pagamentos_positivos,
       SUM(p.amount) AS total
FROM payment p
INNER JOIN customer c ON c.customer_id = p.customer_id
WHERE p.amount > 0
GROUP BY c.customer_id, c.first_name, c.last_name
HAVING COUNT(*) >= 40
ORDER BY total DESC
LIMIT 5;
```

```text
+------------+-----------+----------------------+--------+
| first_name | last_name | pagamentos_positivos | total  |
+------------+-----------+----------------------+--------+
| KARL       | SEAL      |                   45 | 221.55 |
| ELEANOR    | HUNT      |                   46 | 216.54 |
| CLARA      | SHAW      |                   42 | 195.58 |
| WESLEY     | BULL      |                   40 | 177.60 |
| MARCIA     | DEAN      |                   42 | 175.58 |
+------------+-----------+----------------------+--------+
```

Tudo do nível 3 continua valendo; o JOIN acontece no `FROM`, **antes** do `WHERE`. Aqui, cada pagamento combina com exatamente **um** cliente, então nenhum valor se repete e a soma está correta.

Agrupamos por `c.customer_id` **e** pelo nome. Agrupar só pelo nome juntaria clientes diferentes que tivessem o mesmo nome. O padrão do resto do arquivo é este: **agrupar pelo identificador** e incluir o nome para exibição.

### 4.2 Muitos-para-muitos: atores e filmes

Um ator atua em vários filmes, e um filme tem vários atores. A ligação fica em `film_actor`, cuja chave primária composta `(actor_id, film_id)` garante que cada par aparece uma vez (nível 1, seção 4.3).

```mermaid
flowchart LR
    A["actor"] -->|"actor_id"| FA["film_actor"]
    FA -->|"film_id"| F["film"]
```

**▶ Executar** — filmes de PENELOPE GUINESS

```sql
SELECT f.film_id, f.title
FROM actor a
INNER JOIN film_actor fa ON fa.actor_id = a.actor_id
INNER JOIN film f        ON f.film_id   = fa.film_id
WHERE a.first_name = 'PENELOPE' AND a.last_name = 'GUINESS'
ORDER BY f.title
LIMIT 5;
```

```text
+---------+-----------------------+
| film_id | title                 |
+---------+-----------------------+
|       1 | ACADEMY DINOSAUR      |
|      23 | ANACONDA CONFESSIONS  |
|      25 | ANGELS LIFE           |
|     106 | BULWORTH COMMANDMENTS |
|     140 | CHEAPER CLYDE         |
+---------+-----------------------+
```

(Sem o `LIMIT`, são 19 filmes: uma linha por par ator–filme.)

Se uma consulta assim devolver linhas repetidas, a causa quase sempre é uma condição de `ON` errada ou faltando. **Não use `DISTINCT` para esconder isso**: corrija a condição.

### 4.3 Filmes por categoria

**▶ Executar**

```sql
SELECT cat.category_id, cat.name AS categoria, COUNT(*) AS filmes
FROM category cat
INNER JOIN film_category fc ON fc.category_id = cat.category_id
GROUP BY cat.category_id, cat.name
ORDER BY filmes DESC, cat.category_id
LIMIT 5;
```

```text
+-------------+-------------+--------+
| category_id | categoria   | filmes |
+-------------+-------------+--------+
|          15 | Sports      |     74 |
|           9 | Foreign     |     73 |
|           8 | Family      |     69 |
|           6 | Documentary |     68 |
|           2 | Animation   |     66 |
+-------------+-------------+--------+
```

O que está sendo contado são **vínculos** em `film_category` (linhas filme–categoria). Isso equivale a contar filmes porque, no schema, a chave `(film_id, category_id)` impede o mesmo vínculo duas vezes, e, nos dados carregados, cada filme tem exatamente **uma** categoria (1000 vínculos para 1000 filmes). Se um filme pudesse ter duas categorias, ele seria contado em ambas, e o total geral passaria de 1000; para contar filmes distintos, use `COUNT(DISTINCT fc.film_id)`.

Não foi preciso juntar `film`: nenhuma coluna ou filtro dela é usado.

### 4.4 Um caminho longo: cliente → endereço → cidade → país

```mermaid
flowchart LR
    C["customer"] -->|"address_id"| A["address"]
    A -->|"city_id"| CI["city"]
    CI -->|"country_id"| CO["country"]
```

**▶ Executar** — países com mais clientes

```sql
SELECT co.country_id, co.country, COUNT(*) AS clientes
FROM customer c
INNER JOIN address a  ON a.address_id  = c.address_id
INNER JOIN city ci    ON ci.city_id    = a.city_id
INNER JOIN country co ON co.country_id = ci.country_id
GROUP BY co.country_id, co.country
ORDER BY clientes DESC, co.country_id
LIMIT 5;
```

```text
+------------+---------------+----------+
| country_id | country       | clientes |
+------------+---------------+----------+
|         44 | India         |       60 |
|         23 | China         |       53 |
|        103 | United States |       36 |
|         50 | Japan         |       31 |
|         60 | Mexico        |       30 |
+------------+---------------+----------+
```

Cada cliente tem um endereço, cada endereço uma cidade, cada cidade um país: o caminho não multiplica linhas, e `COUNT(*)` conta clientes.

---

## 5. `LEFT JOIN` no Sakila

### 5.1 Idiomas sem filmes

**▶ Executar**

```sql
SELECT l.language_id, l.name AS idioma,
       COUNT(*)         AS linhas_do_resultado,
       COUNT(f.film_id) AS filmes
FROM language l
LEFT JOIN film f ON f.language_id = l.language_id
GROUP BY l.language_id, l.name
ORDER BY l.language_id;
```

```text
+-------------+----------+---------------------+--------+
| language_id | idioma   | linhas_do_resultado | filmes |
+-------------+----------+---------------------+--------+
|           1 | English  |                1000 |   1000 |
|           2 | Italian  |                   1 |      0 |
|           3 | Japanese |                   1 |      0 |
|           4 | Mandarin |                   1 |      0 |
|           5 | French   |                   1 |      0 |
|           6 | German   |                   1 |      0 |
+-------------+----------+---------------------+--------+
```

Com `INNER JOIN`, só English apareceria. O `LEFT JOIN` mantém os 6 idiomas, e `COUNT(f.film_id)` dá o zero correto (seção 3.4).

### 5.2 Filmes sem nenhuma cópia em estoque

**▶ Executar**

```sql
SELECT f.film_id, f.title
FROM film f
LEFT JOIN inventory i ON i.film_id = f.film_id
WHERE i.inventory_id IS NULL
ORDER BY f.title
LIMIT 5;
```

```text
+---------+----------------------+
| film_id | title                |
+---------+----------------------+
|      14 | ALICE FANTASIA       |
|      33 | APOLLO TEEN          |
|      36 | ARGONAUTS TOWN       |
|      38 | ARK RIDGEMONT        |
|      41 | ARSENIC INDEPENDENCE |
+---------+----------------------+
```

Ao todo, 42 filmes estão no catálogo, mas sem nenhuma cópia (`inventory`). O teste usa `i.inventory_id`, a chave primária de `inventory`, que nunca é nula em uma cópia real.

### 5.3 Condição no `ON` ou no `WHERE`?

Pergunta: "Quantos filmes **com mais de 180 minutos** cada idioma tem, **inclusive zero**?"

**✖ Erro intencional** (de lógica) — condição no `WHERE`

```sql
SELECT l.language_id, l.name AS idioma, COUNT(f.film_id) AS filmes_longos
FROM language l
LEFT JOIN film f ON f.language_id = l.language_id
WHERE f.length > 180
GROUP BY l.language_id, l.name;
```

```text
+-------------+---------+---------------+
| language_id | idioma  | filmes_longos |
+-------------+---------+---------------+
|           1 | English |            39 |
+-------------+---------+---------------+
```

Os idiomas com zero sumiram. O `LEFT JOIN` gerou as linhas de Italian, Japanese... com `f.length = NULL`, mas o `WHERE` roda depois e testa `NULL > 180`, que é desconhecido: essas linhas são descartadas.

**▶ Executar** — condição no `ON`

```sql
SELECT l.language_id, l.name AS idioma, COUNT(f.film_id) AS filmes_longos
FROM language l
LEFT JOIN film f ON f.language_id = l.language_id AND f.length > 180
GROUP BY l.language_id, l.name
ORDER BY l.language_id;
```

```text
+-------------+----------+---------------+
| language_id | idioma   | filmes_longos |
+-------------+----------+---------------+
|           1 | English  |            39 |
|           2 | Italian  |             0 |
|           3 | Japanese |             0 |
|           4 | Mandarin |             0 |
|           5 | French   |             0 |
|           6 | German   |             0 |
+-------------+----------+---------------+
```

Agora o filtro faz parte da **regra de correspondência**: "junte cada idioma aos seus filmes longos". Quem não tem filmes longos fica com `NULL` e é contado como zero.

| Onde fica a condição | Papel | Efeito sobre as linhas da esquerda sem par |
|----------------------|-------|--------------------------------------------|
| No `ON` | Define **o que combina** | Permanecem, com `NULL` à direita |
| No `WHERE`, rejeitando `NULL` (`f.length > 180`, `f.rating = 'G'`...) | Filtra **o resultado** | São eliminadas: o `LEFT` passa a se comportar como `INNER` |
| No `WHERE`, aceitando `NULL` (`i.inventory_id IS NULL`) | Filtra o resultado | **Só elas** permanecem (anti-join, seção 5.2) |

A regra não é "condição sobre a direita no `WHERE` transforma em `INNER`": depende de a condição rejeitar ou aceitar as linhas completadas com `NULL`.

---

## 6. `RIGHT JOIN`

O `RIGHT JOIN` mantém todas as linhas da **direita**. `A LEFT JOIN B` dá o mesmo resultado que `B RIGHT JOIN A`, com a mesma lista explícita de colunas.

**▶ Executar** — o pagamento 13, sem cliente cadastrado, aparece

```sql
SELECT c.cliente_id, c.nome, p.pagamento_id, p.cliente_id AS cliente_do_pagamento, p.valor
FROM demo_cliente c
RIGHT JOIN demo_pagamento p ON p.cliente_id = c.cliente_id
ORDER BY p.pagamento_id;
```

```text
+------------+-------+--------------+----------------------+-------+
| cliente_id | nome  | pagamento_id | cliente_do_pagamento | valor |
+------------+-------+--------------+----------------------+-------+
|          1 | Ana   |           10 |                    1 |  5.00 |
|          1 | Ana   |           11 |                    1 |  5.00 |
|          2 | Bruno |           12 |                    2 |  4.00 |
|       NULL | NULL  |           13 |                    9 |  2.00 |
+------------+-------+--------------+----------------------+-------+
```

Na prática, a maioria das pessoas usa só `LEFT JOIN` e escolhe qual tabela vem primeiro.

---

## 7. `CROSS JOIN`: todas as combinações

O `CROSS JOIN` combina **cada** linha de uma tabela com **cada** linha da outra, sem condição. O resultado tem (linhas de A) × (linhas de B) linhas:

| Consulta | Cálculo | Linhas |
|----------|---------|--------|
| `demo_cliente CROSS JOIN demo_telefone` | 3 × 3 | 9 |
| `store CROSS JOIN category` | 2 × 16 | 32 |

**▶ Executar**

```sql
SELECT s.store_id, cat.name AS categoria
FROM store s
CROSS JOIN category cat
ORDER BY s.store_id, cat.name
LIMIT 4;
```

```text
+----------+-----------+
| store_id | categoria |
+----------+-----------+
|        1 | Action    |
|        1 | Animation |
|        1 | Children  |
|        1 | Classics  |
+----------+-----------+
```

Uso típico: montar uma grade de todas as combinações (por exemplo, uma meta por loja e por categoria).

**Cuidado:** a sintaxe antiga com vírgula e sem condição também gera produto cartesiano. `SELECT COUNT(*) FROM store, category;` devolve 32. Entre tabelas grandes, como `rental, payment`, seriam 16044 × 16044 ≈ 257 milhões de linhas.

---

## 8. A mesma tabela duas vezes (*self join*)

Um *self join* não é um tipo novo de JOIN nem uma palavra-chave: é o **padrão** de juntar uma tabela com ela mesma, usando dois apelidos.

**▶ Executar** — pares de atores com o mesmo sobrenome (filtrando os Zellweger)

```sql
SELECT a.first_name AS ator_1, b.first_name AS ator_2, a.last_name
FROM actor a
INNER JOIN actor b
        ON b.last_name = a.last_name
       AND a.actor_id < b.actor_id
WHERE a.last_name = 'ZELLWEGER'
ORDER BY a.actor_id, b.actor_id;
```

```text
+---------+---------+-----------+
| ator_1  | ator_2  | last_name |
+---------+---------+-----------+
| MINNIE  | CAMERON | ZELLWEGER |
| MINNIE  | JULIA   | ZELLWEGER |
| CAMERON | JULIA   | ZELLWEGER |
+---------+---------+-----------+
```

Sem `a.actor_id < b.actor_id`, cada ator formaria par **consigo mesmo** (MINNIE–MINNIE), e cada par apareceria **duas vezes** (MINNIE–JULIA e JULIA–MINNIE). O `<` mantém um único par por dupla.

---

## 9. Relatórios com várias tabelas

### 9.1 Total recebido por categoria de filme

O caminho do pagamento até a categoria:

```mermaid
flowchart LR
    P["payment"] -->|"rental_id"| R["rental"]
    R -->|"inventory_id"| I["inventory"]
    I -->|"film_id"| FC["film_category"]
    FC -->|"category_id"| CAT["category"]
```

**Premissas** que tornam a soma interpretável **nos dados carregados**:

| Premissa | Verificação | Se fosse falsa... |
|----------|-------------|-------------------|
| Todo pagamento está ligado a um aluguel | `SELECT COUNT(*) FROM payment WHERE rental_id IS NULL;` → 0 | O `INNER JOIN` descartaria esses pagamentos, e o total ficaria menor |
| Cada filme tem exatamente uma categoria | 1000 vínculos em `film_category` para 1000 filmes | Um pagamento seria somado em mais de uma categoria (seção 3.5) |

**▶ Executar**

```sql
SELECT cat.category_id, cat.name AS categoria, SUM(p.amount) AS total_recebido
FROM payment p
INNER JOIN rental r         ON r.rental_id     = p.rental_id
INNER JOIN inventory i      ON i.inventory_id  = r.inventory_id
INNER JOIN film_category fc ON fc.film_id      = i.film_id
INNER JOIN category cat     ON cat.category_id = fc.category_id
GROUP BY cat.category_id, cat.name
ORDER BY total_recebido DESC, cat.category_id
LIMIT 5;
```

```text
+-------------+-----------+----------------+
| category_id | categoria | total_recebido |
+-------------+-----------+----------------+
|          15 | Sports    |        5314.21 |
|          14 | Sci-Fi    |        4756.98 |
|           2 | Animation |        4656.30 |
|           7 | Drama     |        4587.39 |
|           5 | Comedy    |        4383.58 |
+-------------+-----------+----------------+
```

O Sakila tem uma view, `sales_by_film_category`, que calcula **a mesma métrica**. Pelo `SHOW CREATE VIEW sales_by_film_category`, ela faz um JOIN a mais (com `film`), agrupa só pelo nome da categoria, chama as colunas de `category` e `total_sales`, ordena por `total_sales DESC` e não tem `LIMIT`. Não é a mesma consulta, mas, nestes dados, as 16 categorias têm os mesmos totais, e as 5 primeiras linhas da view coincidem com as acima.

### 9.2 Por categoria e por loja da cópia

**▶ Executar**

```sql
SELECT cat.category_id,
       cat.name   AS categoria,
       i.store_id AS loja_da_copia,
       SUM(p.amount) AS total_recebido
FROM payment p
INNER JOIN rental r         ON r.rental_id     = p.rental_id
INNER JOIN inventory i      ON i.inventory_id  = r.inventory_id
INNER JOIN film_category fc ON fc.film_id      = i.film_id
INNER JOIN category cat     ON cat.category_id = fc.category_id
GROUP BY cat.category_id, cat.name, i.store_id
HAVING SUM(p.amount) > 2500
ORDER BY total_recebido DESC;
```

```text
+-------------+-----------+---------------+----------------+
| category_id | categoria | loja_da_copia | total_recebido |
+-------------+-----------+---------------+----------------+
|          15 | Sports    |             2 |        2825.75 |
|           7 | Drama     |             1 |        2573.24 |
|          14 | Sci-Fi    |             2 |        2553.19 |
+-------------+-----------+---------------+----------------+
```

**O que é "loja" aqui?** É a loja onde está a **cópia** alugada (`inventory.store_id`). É uma **escolha de atribuição**: também se poderia atribuir o pagamento à loja do cliente (`customer.store_id`) ou à loja do funcionário que o registrou (`staff.store_id`), e os números mudariam. Um relatório precisa dizer qual critério usa.

### 9.3 Relatório por país, sem esconder os zeros

"Para alguns países, mostre clientes, pagamentos e total recebido, **mostrando zero** quando não houver nada."

**▶ Executar**

```sql
SELECT co.country_id,
       co.country                    AS pais,
       COUNT(*)                      AS linhas,
       COUNT(DISTINCT c.customer_id) AS clientes,
       COUNT(p.payment_id)           AS pagamentos,
       COALESCE(SUM(p.amount), 0)    AS total_recebido
FROM country co
LEFT JOIN city ci    ON ci.country_id = co.country_id
LEFT JOIN address a  ON a.city_id     = ci.city_id
LEFT JOIN customer c ON c.address_id  = a.address_id
LEFT JOIN payment p  ON p.customer_id = c.customer_id
WHERE co.country IN ('Brazil', 'Argentina', 'Chile', 'Australia')
GROUP BY co.country_id, co.country
ORDER BY total_recebido DESC;
```

```text
+------------+-----------+--------+----------+------------+----------------+
| country_id | pais      | linhas | clientes | pagamentos | total_recebido |
+------------+-----------+--------+----------+------------+----------------+
|         15 | Brazil    |    748 |       28 |        748 |        3200.52 |
|          6 | Argentina |    352 |       13 |        352 |        1434.48 |
|         22 | Chile     |     71 |        3 |         71 |         328.29 |
|          8 | Australia |      2 |        0 |          0 |           0.00 |
+------------+-----------+--------+----------+------------+----------------+
```

Lendo cada coluna:

| Coluna | O que mede | Por quê |
|--------|------------|---------|
| `linhas` | Linhas geradas pelo JOIN para o país | No Brasil, uma por pagamento. Na Austrália, **2**: há dois endereços lá (o da loja 2 e o de um funcionário), nenhum de cliente |
| `clientes` | `COUNT(DISTINCT c.customer_id)` | Cada cliente se repete uma vez por pagamento; o `DISTINCT` conta cada um uma vez |
| `pagamentos` | `COUNT(p.payment_id)` | Conta só correspondências reais; dá 0 na Austrália, enquanto `COUNT(*)` daria 2 |
| `total_recebido` | `COALESCE(SUM(p.amount), 0)` | `SUM` de nenhum valor é `NULL`; zero é a resposta certa no domínio |

**E se algum elo fosse `INNER`?** Depende do elo:

| Troca | A Austrália continua? | Por quê |
|-------|-----------------------|---------|
| `INNER JOIN city` e `INNER JOIN address` | **Sim** | Ela tem cidade (Woodridge) e endereços |
| `INNER JOIN customer` | **Não** | Nenhum endereço australiano é de cliente |
| `INNER JOIN payment` | **Não** | Sem cliente, não há pagamento |

O filtro `WHERE co.country IN (...)` é sobre a tabela da **esquerda**, que nunca vem com `NULL`, então não elimina linhas sem par.

---

## 10. Aprofundamento (opcional): `FULL OUTER JOIN`

### 10.1 A ideia e a limitação do MySQL

O `FULL OUTER JOIN` manteria as linhas **dos dois lados**: os pares que combinam, as linhas só da esquerda e as linhas só da direita. Nas tabelas de demonstração, seriam Ana (2×), Bruno, Carla (sem pagamento) e o pagamento 13 (sem cliente): 5 linhas.

O MySQL 8.0 **não** tem `FULL OUTER JOIN`:

**✖ Erro intencional**

```sql
SELECT * FROM demo_cliente c FULL OUTER JOIN demo_pagamento p ON p.cliente_id = c.cliente_id;
```

```text
ERROR 1064 (42000): You have an error in your SQL syntax; ... near 'FULL OUTER JOIN ...'
```

> **Nota curta:** sem apelido e sem `OUTER`, `FROM t1 FULL JOIN t2 ...` **não** dá erro no MySQL 8.0.45: `FULL` é interpretado como apelido de `t1` e a consulta vira um `JOIN` comum (com um aviso). Se você vir `FULL JOIN` funcionando no MySQL, desconfie.

### 10.2 Três peças necessárias

| Peça | O que é |
|------|---------|
| `UNION` / `UNION ALL` | Empilha os resultados de duas consultas com o mesmo número de colunas, de tipos compatíveis. `UNION` **remove linhas repetidas** do resultado final; `UNION ALL` mantém todas |
| `COALESCE(a, b)` | Devolve o primeiro valor não nulo (visto no nível 3) |
| Tabela derivada | Uma consulta entre parênteses no `FROM`, que funciona como uma tabela dentro da consulta. Precisa de apelido: `FROM (SELECT ...) AS t`. O MySQL decide se a materializa ou não |

### 10.3 A simulação correta: `LEFT JOIN` + `UNION ALL` + "só da direita"

**▶ Executar**

```sql
SELECT c.nome, p.valor
FROM demo_cliente c
LEFT JOIN demo_pagamento p ON p.cliente_id = c.cliente_id

UNION ALL

SELECT c.nome, p.valor
FROM demo_cliente c
RIGHT JOIN demo_pagamento p ON p.cliente_id = c.cliente_id
WHERE c.cliente_id IS NULL;
```

```text
+-------+-------+
| nome  | valor |
+-------+-------+
| Ana   |  5.00 |
| Ana   |  5.00 |
| Bruno |  4.00 |
| Carla |  NULL |
| NULL  |  2.00 |
+-------+-------+
```

A primeira parte traz os pares e as linhas só da esquerda. A segunda traz **apenas** as linhas da direita sem par (`c.cliente_id IS NULL`, a chave da esquerda, que nunca é nula em um par real). Como as duas partes não se sobrepõem, `UNION ALL` não duplica nada e preserva as repetições legítimas.

**✖ Erro intencional** (de lógica) — a versão "LEFT + `UNION` + RIGHT"

```sql
SELECT c.nome, p.valor FROM demo_cliente c LEFT JOIN demo_pagamento p ON p.cliente_id = c.cliente_id
UNION
SELECT c.nome, p.valor FROM demo_cliente c RIGHT JOIN demo_pagamento p ON p.cliente_id = c.cliente_id;
```

```text
+-------+-------+
| nome  | valor |
+-------+-------+
| Ana   |  5.00 |
| Bruno |  4.00 |
| Carla |  NULL |
| NULL  |  2.00 |
+-------+-------+
```

Faltou uma linha! Os dois pagamentos de Ana, ambos de 5.00, viraram uma linha só, porque o `UNION` removeu "duplicatas" que eram, na verdade, pagamentos diferentes. Trocar por `UNION ALL` sem o filtro também erraria: os pares apareceriam duas vezes, uma em cada metade.

### 10.4 Quando o `UNION` simples basta: comparar catálogos

Quais filmes estão só na loja 1, só na loja 2, ou nas duas? Aqui interessam **filmes**, não cópias: cada loja tem várias cópias do mesmo filme, então primeiro fazemos a lista **distinta** de filmes de cada loja (tabelas derivadas):

**▶ Executar**

```sql
SELECT l1.film_id AS loja_1, l2.film_id AS loja_2
FROM (SELECT DISTINCT film_id FROM inventory WHERE store_id = 1) AS l1
LEFT JOIN (SELECT DISTINCT film_id FROM inventory WHERE store_id = 2) AS l2
       ON l2.film_id = l1.film_id

UNION

SELECT l1.film_id, l2.film_id
FROM (SELECT DISTINCT film_id FROM inventory WHERE store_id = 1) AS l1
RIGHT JOIN (SELECT DISTINCT film_id FROM inventory WHERE store_id = 2) AS l2
       ON l2.film_id = l1.film_id

ORDER BY COALESCE(loja_1, loja_2)
LIMIT 8;
```

```text
+--------+--------+
| loja_1 | loja_2 |
+--------+--------+
|      1 |      1 |
|   NULL |      2 |
|   NULL |      3 |
|      4 |      4 |
|   NULL |      5 |
|      6 |      6 |
|      7 |      7 |
|   NULL |      8 |
+--------+--------+
```

Aqui o `UNION` simples funciona **porque cada lista já é distinta**: uma linha repetida só pode ser o mesmo par contado nas duas metades. Lendo: o filme 1 está nas duas lojas; os filmes 2, 3, 5 e 8 só na loja 2.

Contando as 958 linhas do resultado completo:

| Situação | Filmes |
|----------|--------|
| Nas duas lojas | 563 |
| Só na loja 1 | 196 |
| Só na loja 2 | 199 |
| **Total com alguma cópia** | **958** |

Os outros 42 filmes do catálogo são os da seção 5.2, sem cópia em nenhuma loja.

---

## 11. Limpeza

**▶ Executar**

```sql
DROP TABLE demo_telefone, demo_pagamento, demo_cliente;
SHOW TABLES LIKE 'demo%';
```

```text
Empty set
```

> **Estado final:** o `sakila` está igual ao importado.

---

## 12. Resumindo

Comparação usando `demo_cliente` (3 clientes) à esquerda e `demo_pagamento` (4 pagamentos) à direita, com `ON p.cliente_id = c.cliente_id`:

| Tipo | Pares (Ana×2, Bruno) | Carla (sem pagamento) | Pagamento 13 (sem cliente) | Linhas |
|------|----------------------|-----------------------|----------------------------|--------|
| `INNER JOIN` | Sim | Não | Não | 3 |
| `LEFT JOIN` | Sim | Sim, com `NULL` à direita | Não | 4 |
| `RIGHT JOIN` | Sim | Não | Sim, com `NULL` à esquerda | 4 |
| `FULL OUTER` (simulado, seção 10) | Sim | Sim | Sim | 5 |
| `CROSS JOIN` (sem `ON`) | Todas as combinações | — | — | 3 × 4 = 12 |

| Recurso | Lembre |
|---------|--------|
| Multiplicidade | Uma linha da esquerda gera tantas linhas quantas correspondências tiver (0, 1 ou várias) |
| Contagem com `LEFT JOIN` | `COUNT(chave_da_direita)`, não `COUNT(*)` |
| Anti-join | `LEFT JOIN ... WHERE chave_da_direita IS NULL` |
| `ON` × `WHERE` | Condições de correspondência no `ON`; um `WHERE` que rejeita `NULL` elimina as linhas sem par |
| Somas | Duas tabelas 1:N do mesmo pai inflam `SUM`; `SUM(DISTINCT)` não é correção |
| Agrupamento | Pelo identificador (e o nome para exibir) |

O roteiro completo, em uma consulta:

**◇ Ilustrativo**

```sql
SELECT ...                -- nível 2: colunas, expressões, apelidos
FROM t1
JOIN t2 ON ...            -- nível 4: combinar tabelas
WHERE ...                 -- nível 2: filtrar linhas
GROUP BY ...              -- nível 3: formar grupos
HAVING ...                -- nível 3: filtrar grupos
ORDER BY ...              -- nível 2: ordenar com desempate
LIMIT ...;                -- nível 2: limitar
```

---

## 13. Exercícios

1. **Escolha do JOIN.** Liste os 6 idiomas e, para cada um, quantos filmes o têm como **idioma original** (`film.original_language_id`), mostrando zero quando for o caso.
2. **Previsão de linhas.** Sem executar: quantas linhas devolve `SELECT * FROM actor a JOIN film_actor fa ON fa.actor_id = a.actor_id WHERE a.actor_id = 1;`, sabendo que PENELOPE GUINESS atuou em 19 filmes?
3. **Correção de `ON`.** Um colega quer os filmes do ator 1 e escreveu `... JOIN film_actor fa ON fa.film_id = a.actor_id WHERE a.actor_id = 1`. O resultado tem 10 linhas. O que ele obteve, na verdade?
4. **Filtro que elimina zeros.** Esta consulta deveria mostrar todos os idiomas com a quantidade de filmes NC-17, mas mostra só English. Corrija: `SELECT l.name, COUNT(f.film_id) FROM language l LEFT JOIN film f ON f.language_id = l.language_id WHERE f.rating = 'NC-17' GROUP BY l.language_id, l.name;`
5. **Contagem distinta.** O filme BUCKET BROTHERHOOD foi alugado quantas vezes, e por quantos clientes diferentes? (Caminho: `film` → `inventory` → `rental`.)
6. **Soma duplicada.** `SUM(p.amount)` sobre `payment` → `rental` → `inventory` → `film_actor` dá 367559.07, mas o total de pagamentos é 67406.56. Explique.

**Desafio integrador.** Mostre os 3 países com maior total recebido, com o identificador e o nome do país. Depois, explique em uma frase o que cada linha do resultado representa e por que nenhum pagamento é contado duas vezes.

<details>
<summary><strong>Gabarito comentado</strong></summary>

**1.** Como todos os idiomas devem aparecer, `LEFT JOIN`, com a coluna certa no `ON`:

```sql
SELECT l.language_id, l.name, COUNT(f.film_id) AS filmes
FROM language l
LEFT JOIN film f ON f.original_language_id = l.language_id
GROUP BY l.language_id, l.name
ORDER BY l.language_id;
```

Todos ficam com 0: nenhum filme tem idioma original informado. Com `INNER JOIN`, o resultado seria vazio.

**2.** **19 linhas**: uma por par (ator 1, filme), e o `WHERE` fixa o ator.

**3.** A condição compara o **ID do filme** com o **ID do ator**. Com `a.actor_id = 1`, ela junta as linhas de `film_actor` com `film_id = 1`: o resultado é o **elenco do filme 1** (10 atores), não os filmes do ator 1. Correção: `ON fa.actor_id = a.actor_id`.

**4.** O `WHERE f.rating = 'NC-17'` rejeita as linhas com `NULL`. Leve a condição para o `ON`:

```sql
SELECT l.name, COUNT(f.film_id) AS filmes_nc17
FROM language l
LEFT JOIN film f ON f.language_id = l.language_id AND f.rating = 'NC-17'
GROUP BY l.language_id, l.name
ORDER BY l.language_id;
```

English: 210; os demais: 0.

**5.**

```sql
SELECT COUNT(*) AS alugueis, COUNT(DISTINCT r.customer_id) AS clientes
FROM film f
INNER JOIN inventory i ON i.film_id = f.film_id
INNER JOIN rental r    ON r.inventory_id = i.inventory_id
WHERE f.title = 'BUCKET BROTHERHOOD';
```

34 aluguéis, 33 clientes: um cliente alugou o filme duas vezes. Cada linha do JOIN é um aluguel; `COUNT(DISTINCT ...)` conta pessoas.

**6.** Cada pagamento se liga a **um** filme, mas cada filme tem **vários** atores em `film_actor`. O JOIN repete o pagamento uma vez por ator do filme, e a soma infla (seção 3.5). `film_actor` não participa da pergunta "quanto foi recebido"; tirá-la do JOIN resolve.

**Desafio.**

```sql
SELECT co.country_id, co.country, SUM(p.amount) AS total_recebido
FROM payment p
INNER JOIN customer c ON c.customer_id = p.customer_id
INNER JOIN address a  ON a.address_id  = c.address_id
INNER JOIN city ci    ON ci.city_id    = a.city_id
INNER JOIN country co ON co.country_id = ci.country_id
GROUP BY co.country_id, co.country
ORDER BY total_recebido DESC, co.country_id
LIMIT 3;
```

India (6628.28), China (5798.74) e United States (4110.32). Cada linha representa **um país** e a soma dos pagamentos dos clientes que moram nele. Nenhum pagamento se repete porque cada passo do caminho (pagamento → cliente → endereço → cidade → país) liga cada linha a **exatamente uma** linha da tabela seguinte.

</details>

> **Próximos passos:** subconsultas (para agregar antes de juntar, como a seção 3.5 sugere), views (o Sakila tem várias, como `sales_by_film_category` e `customer_list`) e índices, que podem acelerar JOINs quando as colunas da condição estão indexadas.
