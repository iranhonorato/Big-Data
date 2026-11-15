# SQL — Nível 2: Consultando uma tabela

| Etapa | Arquivo | Tema |
|-------|---------|------|
| 0 | [my_sql_basics.md](my_sql_basics.md) | Fundamentos: banco, SGBD, cliente/servidor, ambiente com Docker e primeira conexão |
| 1 | [sql_comandos_1.md](sql_comandos_1.md) | Carregar o Sakila; investigar estruturas; criar, alterar e remover tabelas; inserir e editar dados |
| **2** | **[sql_comandos_2.md](sql_comandos_2.md)** | **Consultas em uma tabela: `SELECT`, `DISTINCT`, `WHERE`, `LIKE`, `ORDER BY`, `LIMIT` ← você está aqui** |
| 3 | [sql_comandos_3.md](sql_comandos_3.md) | Agregações: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `GROUP BY`, `HAVING` |
| 4 | [sql_comandos_4.md](sql_comandos_4.md) | Combinar tabelas: `INNER`, `LEFT`, `RIGHT` e `CROSS JOIN`, multiplicidade, `FULL` simulado |

> **Contrato deste arquivo**
>
> * **Objetivo:** extrair de **uma** tabela exatamente as linhas e colunas que respondem a uma pergunta.
> * **Conhecimentos prévios:** [nível 1](sql_comandos_1.md): conectar, `USE`, ler a estrutura de uma tabela.
> * **Estado inicial:** banco `sakila` importado e no estado original (seção 3 do nível 1). Execute `USE sakila;`.
> * **Ao final, você conseguirá:** (1) escolher colunas e calcular expressões com apelidos; (2) remover repetições com `DISTINCT`; (3) filtrar com comparações, `AND`/`OR`/`NOT`, `IN`, `BETWEEN` e `LIKE`, usando parênteses; (4) tratar `NULL` corretamente; (5) escrever uma consulta com filtro, ordenação com desempate e limite; (6) corrigir o erro de apelido no `WHERE`.
> * **Estado final:** nenhum dado é alterado neste nível.
> * **Próximo passo:** [nível 3](sql_comandos_3.md).

> Rótulos dos blocos (**▶ Executar**, **✖ Erro intencional**, **◇ Ilustrativo**): veja os [Fundamentos](my_sql_basics.md). Ambiente de referência: `mysql:8.0.45` com o Sakila de `data/`. Todos os comandos rodam **no cliente MySQL**.

---

## 1. Problema

A tabela `film` tem 1000 filmes, cada um com 13 colunas. Um `SELECT * FROM film;` despeja tudo na tela, mas as perguntas reais são mais específicas:

* "Quais filmes têm mais de 3 horas?"
* "Quais classificações indicativas existem?"
* "Quais atores têm o sobrenome começando com Z?"
* "Quais são os 5 filmes mais longos?"

| Operação | Cláusula | Analogia com uma planilha |
|----------|----------|---------------------------|
| Escolher **colunas** | `SELECT` | Esconder as colunas que não interessam |
| Remover **repetições** | `DISTINCT` | "Remover duplicatas" |
| Filtrar **linhas** | `WHERE` | Aplicar um filtro |
| **Ordenar** e **limitar** | `ORDER BY`, `LIMIT` | Classificar e olhar só o topo |

### 1.1 Dicionário das colunas usadas

| Tabela.coluna | Significado | Valores no Sakila |
|---------------|-------------|-------------------|
| `film.title` | Título do filme | Textos em maiúsculas, todos diferentes |
| `film.length` | Duração, em **minutos** | De 46 a 185 |
| `film.rating` | Classificação indicativa do sistema norte-americano (MPAA) | `G` (público geral), `PG`, `PG-13`, `R`, `NC-17` |
| `film.rental_rate` | Preço do aluguel, em unidades monetárias | `0.99`, `2.99`, `4.99` |
| `film.description` | Sinopse (texto em inglês) | Frases geradas automaticamente |
| `actor.first_name`, `actor.last_name` | Nome e sobrenome do ator | Maiúsculas; nomes podem repetir |
| `customer.active` | Cliente ativo? | `1` (sim) ou `0` (não) nos dados carregados |
| `customer.store_id` | Loja do cliente | `1` ou `2` |
| `rental.rental_date` | Data **e hora** do aluguel | De 2005-05-24 a 2006-02-14 |
| `rental.return_date` | Data e hora da devolução | `NULL` se o DVD ainda não voltou |
| `address.address2` | Complemento do endereço | Texto vazio ou `NULL` |

---

## 2. `SELECT`: escolhendo colunas

> O `SELECT` **calcula uma tabela de resultado**. Ele não altera os dados armazenados.

### 2.1 Todas as colunas

**▶ Executar**

```sql
SELECT * FROM language;
```

Devolve as 6 linhas e 3 colunas de `language` (já vistas no nível 1). O `*` é prático para espiar tabelas pequenas. Em consultas de verdade, prefira listar as colunas: o resultado fica menor e não muda se alguém acrescentar colunas à tabela.

### 2.2 Colunas específicas

**▶ Executar**

```sql
SELECT title, length, rating
FROM film
ORDER BY film_id
LIMIT 5;
```

```text
+------------------+--------+--------+
| title            | length | rating |
+------------------+--------+--------+
| ACADEMY DINOSAUR |     86 | PG     |
| ACE GOLDFINGER   |     48 | G      |
| ADAPTATION HOLES |     50 | NC-17  |
| AFFAIR PREJUDICE |    117 | G      |
| AFRICAN EGG      |    130 | G      |
+------------------+--------+--------+
```

**Linha de apoio:** `ORDER BY film_id LIMIT 5` significa "ordene pelo identificador e mostre só as 5 primeiras". Usamos isso por enquanto para não encher a tela e para que a saída seja sempre a mesma. As duas cláusulas são explicadas nas seções 7 e 8.

### 2.3 Expressões e apelidos (`AS`)

O `SELECT` também calcula. Com `AS`, você dá um nome (**apelido**, ou *alias*) à coluna calculada:

**▶ Executar**

```sql
SELECT title,
       length,
       length / 60        AS horas,
       rental_rate,
       rental_rate * 1.10 AS novo_preco
FROM film
ORDER BY film_id
LIMIT 5;
```

```text
+------------------+--------+--------+-------------+------------+
| title            | length | horas  | rental_rate | novo_preco |
+------------------+--------+--------+-------------+------------+
| ACADEMY DINOSAUR |     86 | 1.4333 |        0.99 |     1.0890 |
| ACE GOLDFINGER   |     48 | 0.8000 |        4.99 |     5.4890 |
| ADAPTATION HOLES |     50 | 0.8333 |        2.99 |     3.2890 |
| AFFAIR PREJUDICE |    117 | 1.9500 |        2.99 |     3.2890 |
| AFRICAN EGG      |    130 | 2.1667 |        2.99 |     3.2890 |
+------------------+--------+--------+-------------+------------+
```

| Coluna | Tipo |
|--------|------|
| `title`, `length`, `rental_rate` | **Colunas físicas**: existem na tabela |
| `horas`, `novo_preco` | **Expressões calculadas**: existem só neste resultado |

`CONCAT` junta textos:

**▶ Executar**

```sql
SELECT actor_id, CONCAT(first_name, ' ', last_name) AS nome_completo
FROM actor
ORDER BY actor_id
LIMIT 5;
```

```text
+----------+---------------------+
| actor_id | nome_completo       |
+----------+---------------------+
|        1 | PENELOPE GUINESS    |
|        2 | NICK WAHLBERG       |
|        3 | ED CHASE            |
|        4 | JENNIFER DAVIS      |
|        5 | JOHNNY LOLLOBRIGIDA |
+----------+---------------------+
```

**Observação:** se qualquer argumento de `CONCAT` for `NULL`, o resultado inteiro é `NULL` (`CONCAT('A', NULL, 'B')` devolve `NULL`).

---

## 3. `DISTINCT`: removendo repetições

### 3.1 Com um recorte pequeno

Primeiro, os 8 primeiros filmes, para você conseguir prever o resultado:

**▶ Executar**

```sql
SELECT film_id, rating, rental_rate
FROM film
WHERE film_id BETWEEN 1 AND 8
ORDER BY film_id;
```

```text
+---------+--------+-------------+
| film_id | rating | rental_rate |
+---------+--------+-------------+
|       1 | PG     |        0.99 |
|       2 | G      |        4.99 |
|       3 | NC-17  |        2.99 |
|       4 | G      |        2.99 |
|       5 | G      |        2.99 |
|       6 | PG     |        2.99 |
|       7 | PG-13  |        4.99 |
|       8 | R      |        4.99 |
+---------+--------+-------------+
```

(`WHERE film_id BETWEEN 1 AND 8` filtra os filmes de 1 a 8; o `WHERE` vem na seção 4.)

**Antes de executar, preveja:** quantas linhas terá `SELECT DISTINCT rating` nesse recorte? E `SELECT DISTINCT rating, rental_rate`?

**▶ Executar**

```sql
SELECT DISTINCT rating FROM film WHERE film_id BETWEEN 1 AND 8;
SELECT DISTINCT rating, rental_rate FROM film WHERE film_id BETWEEN 1 AND 8;
```

Saídas (a ordem das linhas pode variar):

```text
+--------+
| rating |
+--------+
| PG     |
| G      |
| NC-17  |
| PG-13  |
| R      |
+--------+

+--------+-------------+
| rating | rental_rate |
+--------+-------------+
| PG     |        0.99 |
| G      |        4.99 |
| NC-17  |        2.99 |
| G      |        2.99 |
| PG     |        2.99 |
| PG-13  |        4.99 |
| R      |        4.99 |
+--------+-------------+
```

* Com **uma coluna**, as 8 linhas viram 5: cada classificação aparece uma vez.
* Com **duas colunas**, o `DISTINCT` considera a **combinação**: `G | 2.99` aparecia nos filmes 4 e 5 e virou uma linha só. Já `G | 4.99` é outra combinação. Resultado: 7 linhas.
* A tabela `film` continua intacta: o `DISTINCT` só afeta o resultado.

### 3.2 Na tabela inteira

| Consulta | Resultado neste dataset | Pergunta respondida |
|----------|-------------------------|---------------------|
| `SELECT DISTINCT rating FROM film;` | 5 classificações | Quais classificações existem? |
| `SELECT DISTINCT rental_rate FROM film;` | `0.99`, `2.99`, `4.99` | Quais são os preços de aluguel? |
| `SELECT DISTINCT rating, rental_rate FROM film;` | 15 combinações | Quais pares classificação/preço existem? |

Aqui, as 15 combinações são todas as possíveis (5 × 3), porque o catálogo do Sakila tem todas elas. Em outro conjunto de dados, poderiam ser menos.

---

## 4. `WHERE`: filtrando linhas

### 4.1 Operadores de comparação

| Operador | Significado | Exemplo |
|----------|-------------|---------|
| `=` | Igual | `rating = 'PG'` |
| `<>` ou `!=` | Diferente | `rating <> 'PG'` |
| `>` / `<` | Maior / menor | `length > 180` |
| `>=` / `<=` | Maior ou igual / menor ou igual | `rental_rate <= 0.99` |

Textos e datas vão entre aspas simples (`'PG'`, `'2005-05-24'`); números, não (`185`).

**▶ Executar** — filmes com a duração máxima, 185 minutos

```sql
SELECT title, length
FROM film
WHERE length = 185;
```

Saída (a ordem das linhas pode variar):

```text
+--------------------+--------+
| title              | length |
+--------------------+--------+
| CHICAGO NORTH      |    185 |
| CONTROL ANTHEM     |    185 |
| DARN FORRESTER     |    185 |
| GANGS PRIDE        |    185 |
| HOME PITY          |    185 |
| MUSCLE BRIGHT      |    185 |
| POND SEATTLE       |    185 |
| SOLDIERS EVOLUTION |    185 |
| SWEET BROTHERHOOD  |    185 |
| WORST BANGER       |    185 |
+--------------------+--------+
```

### 4.2 Combinando condições: `AND`, `OR`, `NOT`

| Operador | A linha entra se... |
|----------|---------------------|
| `A AND B` | As **duas** condições forem verdadeiras |
| `A OR B` | **Pelo menos uma** for verdadeira |
| `NOT (A)` | A condição for falsa |

**▶ Executar** — `AND`: filmes PG-13 com menos de 50 minutos

```sql
SELECT title, length, rating
FROM film
WHERE rating = 'PG-13' AND length < 50;
```

Saída (a ordem das linhas pode variar):

```text
+---------------------+--------+--------+
| title               | length | rating |
+---------------------+--------+--------+
| HALLOWEEN NUTS      |     47 | PG-13  |
| HAWK CHILL          |     47 | PG-13  |
| KWAI HOMEWARD       |     46 | PG-13  |
| LABYRINTH LEAGUE    |     46 | PG-13  |
| NOTTING SPEAKEASY   |     48 | PG-13  |
| PARADISE SABRINA    |     48 | PG-13  |
| RIDGEMONT SUBMARINE |     46 | PG-13  |
| VALENTINE VANISHING |     48 | PG-13  |
+---------------------+--------+--------+
```

**▶ Executar** — `OR`: atores chamados Penelope **ou** com sobrenome Zellweger

```sql
SELECT first_name, last_name
FROM actor
WHERE first_name = 'PENELOPE' OR last_name = 'ZELLWEGER';
```

Saída (a ordem das linhas pode variar):

```text
+------------+-----------+
| first_name | last_name |
+------------+-----------+
| PENELOPE   | GUINESS   |
| PENELOPE   | PINKETT   |
| MINNIE     | ZELLWEGER |
| PENELOPE   | CRONYN    |
| CAMERON    | ZELLWEGER |
| PENELOPE   | MONROE    |
| JULIA      | ZELLWEGER |
+------------+-----------+
```

**▶ Executar** — `NOT`: clientes inativos

```sql
SELECT COUNT(*) FROM customer WHERE NOT (active = 1);
SELECT COUNT(*) FROM customer WHERE active <> 1;
```

As duas devolvem 15. `NOT (active = 1)` e `active <> 1` são equivalentes **em geral**: ambas selecionam as linhas em que `active` é diferente de 1 (e ambas descartam linhas com `active` `NULL`, como a seção 6 explica). Use parênteses depois de `NOT`, para deixar claro a que ele se aplica.

Já `active = 0` só é equivalente a essas duas porque, **nos dados carregados**, `active` vale apenas 0 ou 1. Se existisse uma linha com `active = 2`, `active <> 1` a incluiria e `active = 0`, não. Da mesma forma, `store_id = 2` e `store_id <> 1` só coincidem porque existem apenas duas lojas.

**▶ Executar** — clientes inativos da loja 2

```sql
SELECT customer_id, first_name, last_name
FROM customer
WHERE active = 0 AND store_id = 2
ORDER BY customer_id;
```

```text
+-------------+------------+-----------+
| customer_id | first_name | last_name |
+-------------+------------+-----------+
|          16 | SANDRA     | MARTIN    |
|          64 | JUDITH     | COX       |
|         169 | ERICA      | MATTHEWS  |
|         241 | HEIDI      | LARSON    |
|         315 | KENNETH    | GOODEN    |
|         446 | THEODORE   | CULP      |
|         510 | BEN        | EASTER    |
+-------------+------------+-----------+
```

### 4.3 Cuidado: `AND` vem antes de `OR`

Assim como a multiplicação vem antes da soma, o `AND` é avaliado antes do `OR`.

**Antes de executar, preveja:** quantas linhas cada consulta devolve?

**▶ Executar** — sem parênteses

```sql
SELECT first_name, last_name
FROM actor
WHERE first_name = 'PENELOPE' OR first_name = 'JULIA' AND last_name = 'ZELLWEGER';
```

O MySQL lê: `PENELOPE` **OU** (`JULIA` **E** `ZELLWEGER`). Resultado: as 4 Penelopes e JULIA ZELLWEGER (5 linhas).

**▶ Executar** — com parênteses

```sql
SELECT first_name, last_name
FROM actor
WHERE (first_name = 'PENELOPE' OR first_name = 'JULIA') AND last_name = 'ZELLWEGER';
```

Agora: (`PENELOPE` **OU** `JULIA`) **E** `ZELLWEGER`. Resultado: só JULIA ZELLWEGER (1 linha).

> Sempre que misturar `AND` e `OR`, use parênteses.

### 4.4 `IN`: uma lista de valores

**▶ Executar**

```sql
SELECT first_name, last_name
FROM actor
WHERE last_name IN ('ZELLWEGER', 'MONROE', 'CRUISE');
```

Devolve 6 atores (1 Cruise, 2 Monroe, 3 Zellweger). Equivale a `last_name = 'ZELLWEGER' OR last_name = 'MONROE' OR last_name = 'CRUISE'`. O oposto é `NOT IN`, que tem uma armadilha com `NULL` (seção 6.3).

### 4.5 `BETWEEN` e intervalos de datas

**▶ Executar**

```sql
SELECT title, length
FROM film
WHERE length BETWEEN 46 AND 47;
```

Devolve 12 filmes com 46 **ou** 47 minutos: o `BETWEEN` **inclui** os dois extremos (equivale a `length >= 46 AND length <= 47`).

**Datas e horas.** `rental_date` é `DATETIME` (data **e** hora). Para os aluguéis de um dia inteiro, use um intervalo **semiaberto**: "a partir do início do dia, e antes do início do dia seguinte".

**▶ Executar**

```sql
SELECT rental_id, rental_date, customer_id
FROM rental
WHERE rental_date >= '2005-05-24' AND rental_date < '2005-05-25'
ORDER BY rental_id;
```

```text
+-----------+---------------------+-------------+
| rental_id | rental_date         | customer_id |
+-----------+---------------------+-------------+
|         1 | 2005-05-24 22:53:30 |         130 |
|         2 | 2005-05-24 22:54:33 |         459 |
|         3 | 2005-05-24 23:03:39 |         408 |
|         4 | 2005-05-24 23:04:41 |         333 |
|         5 | 2005-05-24 23:05:21 |         222 |
|         6 | 2005-05-24 23:08:07 |         549 |
|         7 | 2005-05-24 23:11:53 |         269 |
|         8 | 2005-05-24 23:31:46 |         239 |
+-----------+---------------------+-------------+
```

O que acontece nos limites:

| Momento | `>= '2005-05-24' AND < '2005-05-25'` | `BETWEEN '2005-05-24' AND '2005-05-25'` |
|---------|--------------------------------------|-----------------------------------------|
| `2005-05-24 00:00:00` | Entra | Entra |
| `2005-05-24 23:59:59` | Entra | Entra |
| `2005-05-25 00:00:00` | **Não entra** (correto: já é outro dia) | **Entra** (errado: `'2005-05-25'` vale `2005-05-25 00:00:00`) |

Neste dataset não há aluguel exatamente à meia-noite do dia 25, então as duas formas dão as mesmas 8 linhas. É coincidência dos dados; o intervalo semiaberto é correto sempre.

> **✅ Verifique.** Quantos filmes G **ou** PG têm mais de 180 minutos? Escreva com parênteses. *Resposta:* `WHERE (rating = 'G' OR rating = 'PG') AND length > 180` devolve 13 filmes.

---

## 5. `LIKE`: padrões em textos

O `=` compara o texto inteiro. O `LIKE` compara com um **padrão** que usa dois curingas:

| Curinga | Significa | Exemplo | Encontra |
|---------|-----------|---------|----------|
| `%` | Qualquer sequência de caracteres (inclusive nenhuma) | `'Z%'` | Começa com Z |
| `_` | Exatamente **um** caractere qualquer | `'J_N%'` | J, uma letra, N, e qualquer coisa depois |

### 5.1 Começa com, termina com, contém

**▶ Executar**

```sql
SELECT first_name, last_name FROM actor WHERE last_name LIKE 'Z%';
```

Devolve os 3 atores ZELLWEGER (MINNIE, CAMERON, JULIA).

**▶ Executar**

```sql
SELECT title FROM film WHERE title LIKE '%LOVE%' ORDER BY title;
```

```text
+--------------------+
| title              |
+--------------------+
| GRAFFITI LOVE      |
| IDAHO LOVE         |
| IDENTITY LOVER     |
| INDIAN LOVE        |
| LAWRENCE LOVE      |
| LOVE SUICIDES      |
| LOVELY JINGLE      |
| LOVER TRUMAN       |
| LOVERBOY ATTACKS   |
| STRANGELOVE DESIRE |
+--------------------+
```

`'%LOVE%'` casa com o **trecho** `LOVE` em qualquer lugar, inclusive dentro de `LOVER` e `STRANGELOVE`. `LIKE` não entende "palavra inteira".

| Padrão | Significa |
|--------|-----------|
| `'LOVE%'` | Começa com LOVE |
| `'%LOVE'` | Termina com LOVE |
| `'%LOVE%'` | Contém LOVE |

### 5.2 O curinga `_`

**▶ Executar**

```sql
SELECT first_name FROM actor WHERE first_name LIKE '__';
SELECT first_name, last_name FROM actor WHERE first_name LIKE 'J_N%';
```

A primeira devolve os nomes de exatamente **duas** letras: `ED`, `ED`, `AL`, `ED` (com `DISTINCT`, só `ED` e `AL`). A segunda devolve JENNIFER DAVIS, JANE JACKMAN e JON CHASE.

### 5.3 Procurando `%` ou `_` literalmente

E se o texto que você procura contém um `%` de verdade (por exemplo, "50%")? Use `ESCAPE` para escolher um caractere que "desliga" o curinga seguinte:

**▶ Executar**

```sql
SELECT '50%' LIKE '50!%' ESCAPE '!' AS a,
       '500' LIKE '50!%' ESCAPE '!' AS b,
       '500' LIKE '50%'              AS c;
```

```text
+---+---+---+
| a | b | c |
+---+---+---+
| 1 | 0 | 1 |
+---+---+---+
```

Com `!%`, o padrão significa "50 seguido de um `%` literal": casa com `'50%'` (a) e não com `'500'` (b). Sem o escape, `'50%'` é "50 seguido de qualquer coisa" e casa com `'500'` (c). (`1` = verdadeiro, `0` = falso.)

### 5.4 Maiúsculas, minúsculas e acentos

**▶ Executar**

```sql
SELECT first_name, last_name FROM actor WHERE first_name LIKE 'penelope';
```

Encontra as 4 Penelopes, embora os nomes estejam em maiúsculas. Quem decide isso é a **collation** efetiva da coluna; no Sakila, `utf8mb4_0900_ai_ci`, que ignora diferenças de maiúsculas (`ci`) e de acentos (`ai`). Vale também para `=`. Com uma collation sensível (terminada em `_cs` ou `_bin`), a busca não encontraria nada.

---

## 6. `NULL`: a ausência de valor

### 6.1 `NULL` não é zero nem texto vazio

`NULL` significa "**sem valor**": desconhecido ou não aplicável. É diferente de `0` e de `''`. A coluna `address2` mostra a diferença:

**▶ Executar**

```sql
SELECT address_id, address, address2
FROM address
ORDER BY address_id
LIMIT 6;
```

```text
+------------+----------------------+----------+
| address_id | address              | address2 |
+------------+----------------------+----------+
|          1 | 47 MySakila Drive    | NULL     |
|          2 | 28 MySQL Boulevard   | NULL     |
|          3 | 23 Workhaven Lane    | NULL     |
|          4 | 1411 Lillydale Drive | NULL     |
|          5 | 1913 Hanoi Way       |          |
|          6 | 1121 Loja Avenue     |          |
+------------+----------------------+----------+
```

Os endereços 1 a 4 têm `NULL`; os demais (599 endereços) têm texto **vazio**. Nenhum endereço tem complemento de fato preenchido.

### 6.2 Comparações com `NULL` dão "desconhecido"

**✖ Erro intencional** (de lógica: não dá erro, mas não faz o que parece)

```sql
SELECT address_id FROM address WHERE address2 = NULL;
```

```text
Empty set
```

Toda comparação com `NULL` usando `=`, `<>`, `<`, `>` resulta em **desconhecido** (`UNKNOWN`), nem verdadeiro nem falso. O `WHERE` só mantém linhas em que a condição é **verdadeira**. Para testar `NULL`, use `IS NULL` / `IS NOT NULL`:

**▶ Executar**

```sql
SELECT address_id, address FROM address WHERE address2 IS NULL;
```

Devolve os 4 endereços (1 a 4).

| Condição | Resultado para `address2 = NULL` | Resultado para `address2 = ''` | Linhas |
|----------|----------------------------------|--------------------------------|--------|
| `address2 IS NULL` | Verdadeiro | Falso | 4 |
| `address2 = ''` | Desconhecido | Verdadeiro | 599 |
| `address2 = NULL` | Desconhecido | Desconhecido | 0 |

### 6.3 A armadilha do `NOT IN`

`NOT IN` é uma sequência de `<>`. Por isso, herda o comportamento do `NULL`:

**▶ Executar**

```sql
SELECT 3 NOT IN (1, 2)       AS sem_null,
       3 NOT IN (1, 2, NULL) AS com_null,
       1 NOT IN (1, 2, NULL) AS presente;
```

```text
+----------+----------+----------+
| sem_null | com_null | presente |
+----------+----------+----------+
|        1 |     NULL |        0 |
+----------+----------+----------+
```

`3 NOT IN (1, 2, NULL)` não é verdadeiro: como o 3 poderia ser igual ao valor desconhecido, o resultado é desconhecido (`NULL`).

O mesmo acontece quando o `NULL` está **na coluna**:

**▶ Executar** — "endereços cujo complemento não é 'Apto 1'"

```sql
SELECT COUNT(*) FROM address WHERE address2 NOT IN ('Apto 1');
```

Devolve **599**, não 603. Os 4 endereços com `address2` `NULL` sumiram, porque `NULL NOT IN ('Apto 1')` é desconhecido. Para incluí-los: `WHERE address2 NOT IN ('Apto 1') OR address2 IS NULL`.

### 6.4 Um `NULL` com significado de negócio

Em `rental`, `return_date` fica `NULL` enquanto o DVD não volta:

**▶ Executar**

```sql
SELECT rental_id, rental_date, customer_id, return_date
FROM rental
WHERE return_date IS NULL
ORDER BY rental_id
LIMIT 5;
```

```text
+-----------+---------------------+-------------+-------------+
| rental_id | rental_date         | customer_id | return_date |
+-----------+---------------------+-------------+-------------+
|     11496 | 2006-02-14 15:16:03 |         155 | NULL        |
|     11541 | 2006-02-14 15:16:03 |         335 | NULL        |
|     11563 | 2006-02-14 15:16:03 |          83 | NULL        |
|     11577 | 2006-02-14 15:16:03 |         219 | NULL        |
|     11593 | 2006-02-14 15:16:03 |          99 | NULL        |
+-----------+---------------------+-------------+-------------+
```

Aqui, `NULL` significa "ainda não aconteceu". São 183 aluguéis em aberto.

---

## 7. `ORDER BY`: ordenando

### 7.1 Sem `ORDER BY`, a ordem não é garantida

O MySQL devolve as linhas na ordem que for mais conveniente para o plano escolhido. Por isso as saídas das seções 4 e 5 vinham marcadas com "a ordem pode variar". Se a ordem importa, **peça explicitamente**.

| Sintaxe | Ordem |
|---------|-------|
| `ORDER BY length` (ou `ORDER BY length ASC`) | Crescente (padrão) |
| `ORDER BY length DESC` | Decrescente |
| `ORDER BY length DESC, title` | Por duração decrescente; em empate, por título |

Em ordem crescente, o MySQL coloca os `NULL` **primeiro**.

### 7.2 Desempate

Os 5 filmes mais longos:

**◇ Ilustrativo** — incompleto, de propósito

```sql
SELECT title, length FROM film ORDER BY length DESC LIMIT 5;
```

Há **10** filmes com 185 minutos, e a consulta pede só 5. Quais 5 aparecem? Não está definido. Acrescente um critério de desempate:

**▶ Executar**

```sql
SELECT title, length
FROM film
ORDER BY length DESC, title
LIMIT 5;
```

```text
+----------------+--------+
| title          | length |
+----------------+--------+
| CHICAGO NORTH  |    185 |
| CONTROL ANTHEM |    185 |
| DARN FORRESTER |    185 |
| GANGS PRIDE    |    185 |
| HOME PITY      |    185 |
+----------------+--------+
```

O título serve de desempate porque, no Sakila, não há dois filmes com o mesmo título. Quando o critério **pode** repetir, termine com o identificador. Veja os atores DAVIS: há duas SUSAN DAVIS diferentes.

**▶ Executar**

```sql
SELECT actor_id, first_name, last_name
FROM actor
WHERE last_name = 'DAVIS'
ORDER BY first_name, actor_id;
```

```text
+----------+------------+-----------+
| actor_id | first_name | last_name |
+----------+------------+-----------+
|        4 | JENNIFER   | DAVIS     |
|      101 | SUSAN      | DAVIS     |
|      110 | SUSAN      | DAVIS     |
+----------+------------+-----------+
```

Sem o `actor_id` no final, as duas SUSAN poderiam trocar de posição entre execuções.

---

## 8. `LIMIT` e `OFFSET`: limitando

`LIMIT n` devolve no máximo `n` linhas. `OFFSET k` pula as `k` primeiras **posições da sequência ordenada**. Juntos, fazem **paginação**:

**▶ Executar** — página 2, com 5 atores por página

```sql
SELECT actor_id, first_name, last_name
FROM actor
ORDER BY actor_id
LIMIT 5 OFFSET 5;
```

```text
+----------+------------+-----------+
| actor_id | first_name | last_name |
+----------+------------+-----------+
|        6 | BETTE      | NICHOLSON |
|        7 | GRACE      | MOSTEL    |
|        8 | MATTHEW    | JOHANSSON |
|        9 | JOE        | SWANK     |
|       10 | CHRISTIAN  | GABLE     |
+----------+------------+-----------+
```

Aqui os IDs vieram de 6 a 10 porque os atores 1 a 10 existem e estão em sequência. O `OFFSET` conta **posições**, não IDs: se o ator 3 tivesse sido apagado, a página 2 começaria no ator 7.

> `LIMIT` sem `ORDER BY` devolve "quaisquer" `n` linhas. Para "os 5 maiores" ou "a página 2", use sempre os dois juntos, com desempate.

---

## 9. Ordem de escrita × ordem lógica de processamento

Escrevemos as cláusulas em uma ordem fixa:

```sql
SELECT [DISTINCT] colunas   -- 1
FROM tabela                 -- 2
WHERE condição              -- 3
ORDER BY colunas            -- 4
LIMIT n;                    -- 5
```

Para entender o que cada cláusula "enxerga", pense nesta **ordem lógica**:

| Ordem lógica | Etapa | O que acontece |
|--------------|-------|----------------|
| 1 | `FROM` | Escolhe a tabela |
| 2 | `WHERE` | Mantém as linhas em que a condição é verdadeira |
| 3 | `SELECT` | Calcula as colunas e cria os apelidos |
| 4 | `DISTINCT` | Remove repetições do resultado |
| 5 | `ORDER BY` | Ordena |
| 6 | `LIMIT` | Corta |

Essa é uma ordem **lógica**: o otimizador do MySQL pode executar as etapas de outra forma física (usando um índice para já ler as linhas ordenadas, por exemplo), desde que o resultado seja o mesmo.

```mermaid
flowchart LR
    F["FROM"] --> W["WHERE"]
    W --> S["SELECT<br/>cria apelidos"]
    S --> D["DISTINCT"]
    D --> O["ORDER BY<br/>enxerga apelidos"]
    O --> L["LIMIT"]
```

Consequência: um apelido criado no `SELECT` ainda não existe para o `WHERE`.

**✖ Erro intencional**

```sql
SELECT title, length / 60 AS horas
FROM film
WHERE horas > 3;
```

```text
ERROR 1054 (42S22): Unknown column 'horas' in 'where clause'
```

**Correção:** repita a **mesma** expressão no `WHERE`. No `ORDER BY`, que vem depois do `SELECT`, o apelido funciona.

**▶ Executar**

```sql
SELECT title, length / 60 AS horas
FROM film
WHERE length / 60 > 3
ORDER BY horas DESC, title
LIMIT 5;
```

```text
+----------------+--------+
| title          | horas  |
+----------------+--------+
| CHICAGO NORTH  | 3.0833 |
| CONTROL ANTHEM | 3.0833 |
| DARN FORRESTER | 3.0833 |
| GANGS PRIDE    | 3.0833 |
| HOME PITY      | 3.0833 |
+----------------+--------+
```

São 39 filmes com mais de 3 horas (o `LIMIT 5` mostra os primeiros).

---

## 10. Juntando tudo

**Pergunta:** "Quais filmes de classificação **G** (público geral, no sistema norte-americano), com aluguel de **0.99** unidade monetária e **menos de 60 minutos**, existem? Mostre os 5 mais curtos."

**▶ Executar**

```sql
SELECT title, length, rental_rate
FROM film
WHERE rating = 'G'
  AND rental_rate = 0.99
  AND length < 60
ORDER BY length, title
LIMIT 5;
```

```text
+-----------------+--------+-------------+
| title           | length | rental_rate |
+-----------------+--------+-------------+
| DOWNHILL ENOUGH |     47 |        0.99 |
| HOOK CHARIOTS   |     49 |        0.99 |
| EXCITEMENT EVE  |     51 |        0.99 |
| HARPER DYING    |     52 |        0.99 |
| SIDE ARK        |     52 |        0.99 |
+-----------------+--------+-------------+
```

| Etapa lógica | Cláusula | Linhas |
|--------------|----------|--------|
| 1 | `FROM film` | 1000 filmes |
| 2 | `WHERE rating = 'G' AND rental_rate = 0.99 AND length < 60` | 12 filmes |
| 3 | `SELECT title, length, rental_rate` | 12 filmes, 3 colunas |
| 4 | `ORDER BY length, title` | 12 filmes ordenados (HARPER DYING e SIDE ARK empatam em 52; o título desempata) |
| 5 | `LIMIT 5` | 5 filmes |

Mais um, com `LIKE` em um texto longo: filmes PG cuja sinopse menciona um tubarão (*Shark*) nas Montanhas Rochosas canadenses.

**▶ Executar**

```sql
SELECT title
FROM film
WHERE description LIKE '%Shark%'
  AND description LIKE '%Canadian Rockies%'
  AND rating = 'PG'
ORDER BY title;
```

```text
+-----------------+
| title           |
+-----------------+
| CHILL LUCK      |
| LIAISONS SWEET  |
| PIRATES ROXANNE |
+-----------------+
```

---

## 11. Resumindo

| Cláusula | Para quê | Exemplo |
|----------|----------|---------|
| `SELECT col1, col2` | Escolher colunas | `SELECT title, length` |
| `expressão AS apelido` | Calcular e nomear | `length / 60 AS horas` |
| `DISTINCT` | Remover combinações repetidas | `SELECT DISTINCT rating` |
| `WHERE` | Filtrar linhas | `WHERE length > 180` |
| `AND` / `OR` / `NOT (...)` | Combinar condições (use parênteses) | `WHERE (rating = 'G' OR rating = 'PG') AND length > 180` |
| `IN (...)` / `NOT IN (...)` | Lista de valores (cuidado com `NULL`) | `WHERE last_name IN ('ZELLWEGER', 'MONROE')` |
| `BETWEEN a AND b` | Intervalo inclusivo | `WHERE length BETWEEN 46 AND 47` |
| `>= início AND < próximo` | Intervalo semiaberto (datas) | `WHERE rental_date >= '2005-05-24' AND rental_date < '2005-05-25'` |
| `LIKE` (`%`, `_`, `ESCAPE`) | Padrão de texto | `WHERE title LIKE '%LOVE%'` |
| `IS NULL` / `IS NOT NULL` | Testar ausência de valor | `WHERE return_date IS NULL` |
| `ORDER BY ... [ASC/DESC]` | Ordenar, com desempate | `ORDER BY first_name, actor_id` |
| `LIMIT n OFFSET k` | Limitar e paginar | `LIMIT 5 OFFSET 5` |

Até aqui, cada linha do resultado é uma linha da tabela. Mas e perguntas como "quanto a locadora recebeu?" ou "quantos filmes há em cada classificação?" A resposta não é uma linha da tabela, e sim um **resumo** de muitas linhas.

---

## 12. Exercícios

1. **Previsão.** Sem executar, diga quantas linhas tem `SELECT DISTINCT rental_duration FROM film WHERE film_id BETWEEN 1 AND 8;`. Os valores de `rental_duration` desses filmes, na ordem dos IDs, são: 6, 3, 7, 5, 6, 3, 6, 6.
2. **Escrita.** Liste, em ordem alfabética, os títulos que começam com `AIR`.
3. **Depuração.** Um colega quer contar os aluguéis em aberto e escreveu `SELECT COUNT(*) FROM rental WHERE return_date = NULL;`. O resultado foi 0. Explique e corrija.
4. **Datas.** Quantos aluguéis foram feitos no dia 25/05/2005? Use um intervalo semiaberto.
5. **Precedência.** Compare `WHERE rating = 'G' OR rating = 'PG' AND length > 180` com a versão com parênteses em volta do `OR`. Qual responde "filmes G ou PG com mais de 180 minutos"?
6. **Apelido.** Corrija: `SELECT title, rental_rate * 1.1 AS novo FROM film WHERE novo > 5;`. Quantos filmes aparecem?

<details>
<summary><strong>Gabarito comentado</strong></summary>

**1.** **4 linhas** (6, 3, 7, 5). Os valores repetidos (6 e 3) aparecem uma vez cada.

**2.**

```sql
SELECT title FROM film WHERE title LIKE 'AIR%' ORDER BY title;
```

AIRPLANE SIERRA e AIRPORT POLLOCK.

**3.** `return_date = NULL` é sempre desconhecido, e o `WHERE` descarta o desconhecido. Correção: `WHERE return_date IS NULL`, que devolve **183**.

**4.**

```sql
SELECT COUNT(*) FROM rental
WHERE rental_date >= '2005-05-25' AND rental_date < '2005-05-26';
```

**137** aluguéis. `rental_date = '2005-05-25'` estaria errado: compararia com `2005-05-25 00:00:00` exato.

**5.** Sem parênteses, o MySQL lê `rating = 'G' OR (rating = 'PG' AND length > 180)`: todos os filmes G, de qualquer duração, mais os PG longos (182 filmes). A versão `(rating = 'G' OR rating = 'PG') AND length > 180` responde à pergunta (13 filmes).

**6.** O apelido não existe no `WHERE`. Repita a expressão:

```sql
SELECT title, rental_rate * 1.1 AS novo
FROM film
WHERE rental_rate * 1.1 > 5;
```

São **336** filmes, exatamente os de `rental_rate = 4.99` (4.99 × 1.1 = 5.489).

</details>

> **Próximo passo:** o [nível 3](sql_comandos_3.md) ensina a **resumir** conjuntos de linhas com `COUNT`, `SUM`, `AVG`, `GROUP BY` e `HAVING`.
