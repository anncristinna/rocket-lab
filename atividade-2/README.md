# CineData Analytics: Arquitetura Medalhão no Databricks

Projeto da Atividade 2 do Rocket Lab 2026 (Visagio). A ideia é pegar um catálogo de filmes (base combinada TMDB/IMDb), entregue de forma suja e fragmentada em CSVs, e organizar tudo em um Data Lakehouse seguindo a Arquitetura Medalhão (Bronze, Silver e Gold), até chegar em um Star Schema pronto para BI.

## Como o projeto está organizado

Tudo foi feito no Databricks (Free Edition), catálogo `workspace`, com os schemas `landing`, `bronze`, `silver` e `gold`. Os CSVs originais ficam no volume `workspace.landing.inputs`.

```
notebooks/
  01_bronze.ipynb
  02_silver.ipynb
  03_gold.ipynb
  04_analytics.ipynb
README.md
```

Para reproduzir, é só criar os schemas e o volume, subir os cinco CSVs no volume e rodar os notebooks na ordem 01, 02, 03 e 04.

## Camada Bronze

O notebook 01 lê cada CSV sem alterar conteúdo (tudo entra como texto), adiciona a coluna `ingestion_datetime` e grava em Delta com modo append. Além dos cinco CSVs, a cotação do dólar vem da API PTAX do Banco Central e é gravada em `bronze.tb_cotacao_dolar`. As datas da consulta são widgets do notebook, no formato MM-DD-AAAA, e o padrão é olhar os últimos 7 dias corridos.

## Camada Silver

O notebook 02 limpa, tipa e traduz as colunas para português. Algumas decisões que valem registrar:

**Duplicatas.** As tabelas de origem tinham muitas linhas repetidas (por exemplo, 213 mil linhas de filmes para 97 mil ids distintos). A deduplicação mantém o registro mais recente pela data de ingestão. Como todas as cargas aconteceram no mesmo instante, usei um hash da linha como desempate, para o resultado ser sempre o mesmo.

**Status do filme.** O texto é normalizado (minúsculas, sem hífen, espaço ou ruído) antes de ser traduzido. Valores que não correspondem a nenhum status, como datas que caíram na coluna por causa do deslocamento de colunas, viram "Não Informado".

**Datas de lançamento.** A origem mistura formatos. Pelo que apareceu nos dados, datas com barra seguem dia/mês/ano, e datas com hífen no formato não ISO seguem mês-dia-ano (um exemplo é 01-17-2018, em que 17 não poderia ser mês). A ordem dos formatos testados respeita isso, e só vira NULL o que nenhum formato consegue converter.

**Financeiro.** Os valores de orçamento e receita passam por uma limpeza de símbolos de moeda e separadores de milhar. Zero, negativo e textos como "Unknown" viram NULL. A conversão para reais usa a cotação mais recente da série contínua (na execução feita, 4,9692). Lucro e margem ficam NULL quando falta orçamento ou receita, e a margem é protegida contra divisão por zero.

**Métricas de engajamento.** A conversão de tipo é segura, então texto fora de contexto vira NULL sem quebrar o pipeline. Notas fora da escala de 0 a 10 e contagens ou popularidades negativas também viram NULL.

**Avaliações de usuários.** Notas fora de 0 a 10 viram NULL e comentários vazios ou só com espaços recebem "Sem comentário". Conferi que a base não tem avaliações integralmente duplicadas (32.412 linhas e 32.412 distintas), então a remoção de duplicatas não elimina nada, mas a regra continua implementada.

**Gêneros.** A coluna de gêneros aceita três separadores (vírgula, ponto e vírgula e barra vertical). Por causa do deslocamento de colunas, apareceram nomes de produtoras, países e trechos de texto como se fossem gêneros (mais de 680 valores distintos). Por isso a tabela filtra pela lista fechada de 19 gêneros do TMDB.

**Pessoas e empresas.** Atores, diretores, roteiristas e produtoras ficam em uma tabela única com a coluna `tipo_entidade`. A capitalização é padronizada, valores vazios, "N/A", números soltos e sobras de aspas são removidos e as duplicatas eliminadas.

**Cotação do dólar.** A API não devolve fins de semana e feriados, então a série é completada dia a dia e os dias sem cotação recebem o valor do último dia útil (forward fill).

## Camada Gold

O notebook 03 monta o Star Schema com chaves substitutas geradas por `row_number()`, que são determinísticas e geram as mesmas chaves a cada reprocessamento.

A tabela fato `fact_movies_performance` tem um registro por filme e considera apenas filmes com status Lançado, como pede o enunciado. As dimensões são `dim_movies`, `dim_genres`, `dim_people`, `dim_companies` e `dim_reviews`. As relações de muitos para muitos são resolvidas pelas tabelas-ponte `bridge_movie_genre`, `bridge_movie_person` e `bridge_movie_company`, o que evita duplicar linhas na fato.

Na `dim_reviews`, só entram avaliações de filmes que existem em `dim_movies`, para manter a integridade da chave estrangeira. Por isso ela tem 27.226 filmes, um pouco menos que os ids distintos de avaliações na Bronze. A `dim_people` considera o par nome e tipo, então uma pessoa que atua e dirige aparece uma vez em cada papel.

No final do notebook há testes de validação: nenhum `sk_movie_id` duplicado na fato e nenhuma chave estrangeira órfã.

## Perguntas de negócio

O notebook 04 responde às seis perguntas usando apenas as tabelas da Gold. Para os recortes de 2 e 5 anos, a data limite é o lançamento mais recente da base, ignorando filmes não lançados e datas futuras. Na execução feita, essa data foi 19/02/2026.

Algumas observações sobre as respostas. O ator com mais participações nos últimos 2 anos foi Kevin Hart (63 filmes), contando qualquer participação no elenco, sem separar por tipo de produção. A produtora com maior lucro nos últimos 5 anos foi a Universal Pictures. Como um filme pode ter várias produtoras, o lucro dele é somado inteiro para cada uma delas, e filmes sem orçamento ou receita ficam de fora porque o lucro deles é NULL. O ranking de produtoras usa o lucro em dólar.

## Limitações conhecidas

A maioria dos filmes não tem orçamento nem receita informados, então os números financeiros representam só uma parte da base. Os nomes de pessoas e produtoras não têm uma lista fechada para validar, então ainda pode sobrar algum resíduo isolado vindo do deslocamento de colunas. A cotação usada é única para todos os filmes, e não a cotação da data de lançamento de cada um.