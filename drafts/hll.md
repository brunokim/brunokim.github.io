lang: pt-BR
---

# O problema de "count distinct"

Você é a pessoa de dados de uma empresa com
um site bem popular. A cada vez que alguém
acessa uma página do site, um registro é
escrito em uma base de dados com informações
sobre o evento, a página, e a pessoa que
visualizou:

```yaml
---
hora_de_acesso: !timestamp '2025-11-12 09:33:18'
ip_de_origem: 103.64.97.230
página: /home.html
publicação: !timestamp '2025-06-02'
cliente: 1276901
data_de_registro: 2022-08-17
faixa_etária: '25-34 anos'
```

## MAU

Sendo a pessoa de dados, uma das primeiras
tarefas que te pedem é:

> **Quantos clientes
acessaram o site nos últimos 30 dias?**

Esta é uma das métricas mais básicas de
negócios online, chamada MAU (_monthly active
users_, clientes ativos mensais).

Isto é simples de se fazer se a base suporta
fazer consultas com SQL:

```sql
SELECT COUNT(DISTINCT cliente) AS mau
FROM site_logs
WHERE date_diff('day',
    hora_de_acesso,
    CURRENT_TIMESTAMP) <= 30;
```

Mas como essa consulta funciona por baixo?
Vamos tentar implementar o mesmo em Python
para ter uma ideia. Para contar os elementos
sem repetiçáo, usamos um `set` para conter todos os elementos únicos.

```python
from datetime import datetime, timedelta

def monthly_active_users(logs):
  unique = set()
  for log in logs:
    if (datetime.now() - log.hora_de_acesso <
        timedelta(days=30)):
      unique.add(log.cliente)
  return len(unique)
```

Quanto de memória essa consulta usa?
Uma estrutura de dados do tipo
`set` precisa armazenar todos os elementos
observados; algumas implementações mais
rápidas podem precisar de ainda mais
memória por elemento.
Se cada número de cliente tem 8 bytes,
a consulta precisaria de 8 KB se a
empresa tivesse 1000 clientes ativos
mensais,
e 8 MB se tivesse 1 milhão de clientes
ativos mensais.
Esses valores são bem pequenos sabendo
que smartphones baratos tem 32 GB de
memória, quase 4000x mais.

## DAU

Logo que você entrega o resultado da
consulta, uma nova requisição chega:

> **Quantos clientes ativos diários (DAU, _daily active users_) nós tivemos em cada um dos últimos 30 dias?**

Esta é uma consulta que podemos fazer usando GROUP BY em SQL:

```sql
SELECT
  date(hora_de_acesso) AS data,
  COUNT(DISTINCT cliente) AS dau
FROM site_logs
WHERE date_diff('day',
    hora_de_acesso,
    CURRENT_TIMESTAMP) <= 30;
GROUP BY date(hora_de_acesso)
```

Quanta memória essa consulta usa? Para
cada dia, é necessário criar um conjunto
distinto para armazenar os clientes,
então temos na média 30·DAU elementos
para armazenar.

Pela definição dessas métricas, DAU 
sempre será menor ou igual a MAU:
um cliente ativo em um dia sempre será
ativo no mês, mas não necessariamente
um cliente ativo em um mês será ativo
em um dia específico dele.
O sonho de todo produto digital é ser
tão útil e necessário que DAU ≈ MAU,
como o WhatsApp, que é usado diariamente
por bilhões de pessoas.
Na prática, cada produto tem uma certa
frequência de uso pelos seus clientes,
então vamos supor que DAU = 20% do MAU
na nossa empresa.
Isso quer dizer que, na média,
um cliente entra a cada 5 dias no site.

Para as nossas consultas, isso quer dizer
que calcular o DAU para um mês vai consumir 30·20% = 6x mais de memória
do que calcular o MAU. A nossa margem
de comparação com um smartphone diminuiu
de ~4000x para ~600x.

## Dimensões

Os nossos logs são razoavelmente ricos,
e o time de negócios gostaria de ainda
mais informações comparativas a partir
deles:

> **O MAU é diferente entre faixas etárias?**

> **O DAU muda entre os dias da semana para as páginas "/home.html" e "/app.html"?**

> **Como é a distribuição de MAU pelo estado do Brasil de onde a pessoa está acessando?**

Cada uma dessas perguntas vira uma
consulta diferente:

```sql
SELECT
  b.estado,
  COUNT(DISTINCT a.cliente) AS mau_por_estado
FROM site_logs AS a
LEFT JOIN geoloc_ip AS b
  ON a.ip_de_origem = b.ip
     AND date(a.hora_de_acesso) = b.data
WHERE date_diff('day',
    hora_de_acesso,
    CURRENT_TIMESTAMP) <= 30;
```
