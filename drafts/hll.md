lang: pt-BR
---

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
Sendo a pessoa de dados, uma das primeiras
tarefas que te pedem é: **quantos clientes
acessaram o site nos últimos 30 dias?**
Esta é uma das métricas mais básicas de
negócios online, chamada MAU (monthly active
users).

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
sem repetiçáo, precisamos usar um conjunto.

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

