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
    CURRENT_TIMESTAMP) <= 30
GROUP BY date(hora_de_acesso);
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
  faixa_etária,
  COUNT(DISTINCT a.cliente) AS mau_por_faixa_etária
FROM site_logs
WHERE date_diff('day',
    hora_de_acesso,
    CURRENT_TIMESTAMP) <= 30;

SELECT
  date(hora_de_acesso) AS data,
  day_of_week(hora_de_acesso) AS dia_da_semana,
  página,
  COUNT(DISTINCT cliente) AS dau_por_dia_e_página
FROM site_logs
WHERE página IN ('app.html', 'home.html')
  AND date_diff('day',
    hora_de_acesso,
    CURRENT_TIMESTAMP) <= 30
GROUP BY
  date(hora_de_acesso),
  day_of_week(hora_de_acesso),
  página;

SELECT
  b.estado,
  COUNT(DISTINCT a.cliente) AS mau_por_estado
FROM site_logs AS a
LEFT JOIN geoloc_ip AS b
  ON a.ip_de_origem = b.ip
     AND date(a.hora_de_acesso) = b.data
WHERE date_diff('day',
    hora_de_acesso,
    CURRENT_TIMESTAMP) <= 30
GROUP BY b.estado;
```

Para cada arquivo de dados requisitado,
os analistas carregam no Excel, fazem
gráficos, e voltam com novas perguntas.
Isso nunca vai acabar, você vai ficar
o tempo inteiro só fazendo consultas!

Felizmente, a indústria já está muito
madura e sabe como entregar as chaves
para os dados na mão dos analistas:
ferramentas de dashboards.
As pessoas de dados precisam apenas
preparar o dataset com **métricas e
dimensões**, e os analistas selecionam
aquelas que os interessam a cada nova
análise.

Perceba que todas as consultas seguem um
padrão:

```sql
SELECT
  dimensão_1,
  dimensão_2,
  COUNT(DISTINCT cliente) AS métrica
FROM site_logs
WHERE filtro
GROUP BY
  dimensão_1,
  dimensão_2;
```

**Dimensão** é uma característica que
pode ser uma variável
explicativa de um fenômeno.
Se o fenômeno é "frequência de acesso"
e analisamos pela variável "faixa etária",
podemos descobrir uma relação
entre idade e interesse pelo que temos
a ofertar.
Esse tipo de relação não quer dizer que
uma coisa _causa_ a outra, mas só a
existência de uma _correlação_ já pode
direcionar o time de produto a realizar
propagandas focando em uma ou outra
faixa etária.

As **métricas** são as variáveis de
efeito de um fenômeno.
São elas que desejamos controlar
indiretamente ao alterar as variáveis
explicativas.
Continuando o exemplo anterior, se
comprarmos propagandas direcionadas
para um público que acreditamos ser mais
ativo, esperamos mensurar um aumento
na métrica como consequência dessa ação.

A tarefa de criar dimensões para
análise é contínua, mas após você
configurar algumas no dashboard, os
analistas estão muito mais felizes e
autônomos, e param de te pedir coisas,
certo? Certo?

## Pré-agregação

Não demora muito para o encanto inicial
com o dashboard dar lugar a novas
demandas. A mais presente é simples e
lacônica:

> **O dashboard é muito lento**

Você testa o dashboard e ele não te
parece mais lento do que fazer a
consulta, mas acompanhando um analista
você percebe que o problema está nas
expectativas de cada um.
No Excel, eles podem configurar e
reconfigurar um gráfico várias vezes por
minuto até chegar no formato que desejam,
mas no seu dashboard, cada interação
exige uma nova consulta de vários segundos
sobre milhões de logs.

Você até aconselha os analistas para 
analisarem menos dados por vez,
limitando a janela de tempo de interesse,
mas você sente que precisa existir um
jeito melhor.

Nas suas pesquisas, você encontra
relatos sobre montar um **cubo de
dados**, onde cada métrica é pré-calculada
para todas as combinações de dimensões, 
e análises exigem apenas combinar as
métricas das dimensões de interesse no
momento.

Por exemplo, para as dimensões da seção
anterior, poderíamos montar o seguinte
cubo para a métrica "contagem de registros":

```sql
CREATE TABLE logs_cubo AS
SELECT
  date(a.hora_de_acesso) AS data,
  day_of_week(a.hora_de_acesso) AS dia_da_semana,
  a.faixa_etária,
  a.página,
  b.estado
  COUNT(*) AS num_registros
FROM site_logs AS a
LEFT JOIN geoloc_ip AS b
  ON a.ip_de_origem = b.ip
     AND date(a.hora_de_acesso) = b.data
GROUP BY 1, 2, 3, 4, 5
```

Esta pré-agregação já simplifica e 
reduz o número de linhas a serem
consultadas pelo dashboard.
Agora, para obter aquelas mesmas
métricas podemos apenas ajustar os
filtros e combinar a métrica com SUM:

```sql
SELECT
  faixa_etária,
  SUM(num_registros) AS num_registros
FROM logs_cubo
WHERE data > CURRENT_DATE - INTERVAL '30' DAY
GROUP BY faixa_etária;
```

Outro benefício dessa pré-agregação é
que ela é **incremental**. Uma vez que
calculamos a agregação para uma data,
não precisamos calcular de novo.
Podemos, então, a cada dia, calcular e
inserir em `logs_cubo` apenas as
agregações deste período.
Só seria necessário recalcular todo o
histórico ao adicionar ou modificar
uma dimensão ou métrica.

Mas resta um grande problema: como
podemos fazer o mesmo para `COUNT(DISTINCT cliente)`?
O mesmo cliente pode estar presente em
mais de um conjunto de dimensões (e.g.,
dias diferentes, estados diferentes...),
então ao somar as contagens únicas
isoladamente, podemos estar contando
um elemento mais de uma vez.

## Conjuntos exatos

Vamos voltar ao Python...

```py
def métrica_zero():
  return {
    "num_registros": 0,
    "clientes_únicos": set(),
  }

cubo = {}

for log in logs:
  # Calcula as dimensões associadas a
  # esse registro de log.
  dims = (
    calc_data(log.hora_de_acesso),
    calc_dia_da_semana(log.hora_de_acesso),
    log.faixa_etária,
    log.página,
    geoloc_ip(log.ip_de_origem).estado,
  )

  # Inicializa as métricas da dimensão,
  # se não existirem ainda.
  if dims not in cubo:
    cubo[dims] = métrica_zero()

  # Incrementa as métricas associadas a
  # essa combinação de dimensões
  cubo[dims]["num_registros"] += 1
  cubo[dims]["clientes_únicos"].add(log.cliente)
```
