# Cinéfilo — SBC de regras para recomendação de filmes

Sistema Baseado em Conhecimento que recomenda filmes a partir das características de uma sessão de
cinema. O usuário descreve com quem vai assistir, se há crianças, quanto tempo tem, o clima que
procura e a preferência de época; um motor de inferência aplica doze regras IF–THEN encadeadas em
três níveis, explica cada decisão tomada e converte as conclusões em uma consulta à API da TMDB, que
devolve filmes reais.

Mini-Projeto 1 — Sistemas Baseados em Conhecimento (UFPB).

## Domínio

Escolher um filme para assistir é uma decisão com restrições que interagem entre si, e é justamente
essa interação que justifica um sistema de regras em vez de uma sequência de `if`s.

A presença de crianças, por exemplo, não é apenas mais um filtro: ela restringe a classificação
indicativa, e essa restrição por sua vez invalida gêneros que o humor escolhido teria selecionado
(quem pediu "medo" com crianças na sala não pode receber terror) e ainda reduz a duração máxima
aceitável. O mesmo acontece com a época: pedir clássicos não muda apenas a faixa de anos, muda também
o que conta como um filme bem avaliado, porque filmes antigos acumulam menos votos na TMDB. Uma
decisão altera o contexto em que as outras são tomadas, e o encadeamento entre regras resolve isso
naturalmente.

As variáveis de entrada são cinco:

| Variável | Valores |
|---|---|
| Companhia | sozinho, casal, amigos, família |
| Crianças | sim, não |
| Tempo disponível | 60 a 300 minutos |
| Humor | rir, emocionar, adrenalina, medo, pensar |
| Época | recente, clássico, tanto faz |

A saída é uma consulta ao endpoint `/discover/movie` da TMDB, acompanhada da justificativa de cada
parâmetro que a compõe.

## Modelagem

Cada fato representa um único conceito, sem duplicação de dados entre eles.

| Fato | Papel | Campos |
|---|---|---|
| `Sessao` | entrada do usuário | `companhia`, `criancas`, `tempo`, `humor`, `epoca` |
| `Restricao` | classificação indicativa máxima | `cert` (`L` ou `18`) |
| `Genero` | gêneros a buscar, combinados com OU | `ids` (identificadores da TMDB) |
| `Periodo` | faixa de lançamento | `inicio`, `fim` |
| `Criterio` | ordenação dos resultados | `ordenar`, `votos_min` |
| `Duracao` | faixa de duração aceitável | `min`, `max` |
| `Consulta` | saída do motor | `params` |

Todo fato derivado carrega também o campo `regras`, com a cadeia de regras que o produziu. É esse
campo que permite ao sistema responder *por que* chegou a cada conclusão, e não apenas *qual* foi a
conclusão.

## As doze regras

**R1.** Se há crianças na sessão, então a classificação máxima é Livre. *(nível 1, salience 10)*

**R2.** Se nenhuma restrição infantil foi estabelecida, então a sessão é adulta e não recebe filtro
de classificação. *(nível 1, salience −10 — regra padrão)*

**R3.** Se o humor desejado é H, então os gêneros base são os associados a H: rir leva a comédia;
emocionar, a drama; adrenalina, a ação e suspense; medo, a terror; pensar, a ficção científica e
mistério. *(nível 1, salience 0)*

**R4.** Se a época preferida é E, então a faixa de lançamento é a correspondente: recente restringe
aos últimos cinco anos, clássico restringe a lançamentos até 1999, e tanto faz não impõe limite.
*(nível 1, salience 0)*

**R5.** Se a sessão é em grupo (amigos ou família), então os resultados são ordenados por
popularidade. *(nível 1, salience 10)*

**R6.** Se nenhum critério de ordenação foi estabelecido, então os resultados são ordenados pelos
filmes mais bem avaliados. *(nível 1, salience −10 — regra padrão)*

**R7.** Se a classificação é Livre e os gêneros selecionados incluem algum impróprio para crianças
(terror, suspense, crime ou guerra), então esses gêneros são substituídos por animação, família e
aventura. *(nível 2, salience 5)*

**R8.** Se o tempo disponível é T e a classificação é C, então a duração aceitável vai de 60 minutos
até T, com T limitado a 100 minutos quando C é Livre. *(nível 2, salience 0)*

**R9.** Se a sessão é de um casal sem restrição infantil, o humor é rir ou emocionar, e romance ainda
não está entre os gêneros, então romance é acrescentado. *(nível 2, salience 5)*

**R11.** Se a busca é por clássicos e o mínimo de votos exigido está acima de 200, então o mínimo cai
para 200. *(nível 2, salience 5)*

**R12.** Se a sessão é longa (duração máxima de 150 minutos ou mais) e o piso de duração ainda é o
padrão, então o piso sobe para 90 minutos. *(nível 2, salience 5)*

**R10.** Se classificação, gêneros, duração, período e critério já estão definidos e nenhuma consulta
foi montada, então monta-se a consulta à TMDB com todos esses parâmetros. *(nível 3, salience −50)*

As duas últimas regras do nível 2 existem porque um critério estabelecido no nível 1 pode ficar
inadequado depois que outra decisão é tomada. A R11 corrige o mínimo de votos: exigir mil avaliações
faz sentido para lançamentos recentes, mas descarta clássicos legítimos, que circularam antes da
TMDB existir. A R12 corrige o piso de duração: quem separou três horas para uma sessão provavelmente
não quer um filme de setenta minutos.

### Encadeamento em três níveis

O encadeamento não é decorativo: nenhuma execução chega ao resultado sem atravessar os três níveis.

- **Nível 1 (R1 a R6)** lê apenas o fato `Sessao` e produz `Restricao`, `Genero`, `Periodo` e
  `Criterio`.
- **Nível 2 (R7, R8, R9, R11, R12)** lê fatos produzidos no nível 1. A R8 depende de `Restricao` para
  criar `Duracao`; R7 e R9 corrigem `Genero` à luz da classificação já decidida; R11 corrige
  `Criterio` à luz de `Periodo`; R12 corrige a `Duracao` que a própria R8 acabou de criar.
- **Nível 3 (R10)** depende de `Duracao`, que só passa a existir depois do nível 2.

Vale notar que "nível" descreve dependência entre regras, não ordem de disparo. No trace é comum ver
uma regra de nível 2 disparar antes de uma de nível 1 que ainda estava pendente na agenda — o que
importa é que a R10 jamais dispara antes que seus cinco fatos de entrada existam.

### Resolução de conflito

O sistema usa duas estratégias combinadas, `salience` e `NOT`.

**Regras padrão canceladas por regras específicas.** Os pares R1/R2 e R5/R6 disputam a mesma decisão.
As regras padrão (R2 e R6) têm salience −10 e condição negativa: `NOT(Restricao())` e
`NOT(Criterio())`. Quando a regra específica correspondente dispara primeiro — R1 e R5 têm salience
10 — o fato passa a existir e a ativação da regra padrão é removida da agenda antes de chegar a vez
dela. O resultado é que nunca coexistem duas classificações nem dois critérios de ordenação.

**Conclusão adiada até o fim.** A R10 tem salience −50, abaixo de todas as outras. Isso garante que a
consulta só seja montada depois que R7, R9, R11 e R12 já tenham feito seus ajustes, e não com valores
intermediários.

O notebook torna isso observável: a função `mostrar_agenda` imprime o conjunto de conflito antes da
execução, e a comparação com o trace mostra explicitamente quais ativações foram canceladas.

### Ausência de laços

As regras que modificam fatos existentes poderiam, em princípio, reativar a si mesmas. Todas são
bloqueadas pela própria condição: a regra só dispara quando o fato está fora do estado desejado, e
seu efeito é justamente colocá-lo nesse estado.

| Regra | Condição de disparo | Efeito |
|---|---|---|
| R7 | há gênero impróprio na lista | produz lista sem nenhum |
| R9 | romance ausente | produz lista com romance |
| R11 | mínimo de votos acima de 200 | grava exatamente 200 |
| R12 | piso de duração abaixo de 90 | grava exatamente 90 |
| R10 | `NOT(Consulta())` | cria o fato `Consulta` |

## Casos de teste

Os três casos estão nas células 16 a 18 do notebook, cada um com a saída esperada comentada e
verificada por asserções sobre o conjunto de regras disparadas e sobre a consulta gerada. Ambos são
determinísticos; apenas os filmes devolvidos pela TMDB variam ao longo do tempo.

### Caso 1 — família com crianças

Entrada: família, com crianças, 120 minutos, humor "medo", época indiferente.

Exercita o caminho infantil completo. R1 impõe classificação Livre e cancela R2; R5 impõe ordenação
por popularidade e cancela R6. R3 seleciona terror a partir do humor, e R7 o substitui por animação,
família e aventura. R8 reduz a duração de 120 para 100 minutos apesar do tempo disponível. R11 e R12
não se aplicam: a época é indiferente e a sessão não é longa.

Regras disparadas: R1, R3, R4, R5, R7, R8, R10.

```
with_genres=16|10751|12         certification.lte=L (BR)
with_runtime.gte=60             with_runtime.lte=100
sort_by=popularity.desc         vote_count.gte=500
```

### Caso 2 — casal sem crianças

Entrada: casal, sem crianças, 150 minutos, humor "emocionar", filmes recentes.

Exercita as duas regras padrão, a regra de romance e a de sessão longa. Sem crianças, R1 não dispara
e R2 assume a sessão adulta; sem sessão em grupo, R5 não dispara e R6 assume a ordenação por
avaliação. R3 seleciona drama e R9 acrescenta romance. R12 eleva o piso de duração para 90 minutos.

Regras disparadas: R2, R3, R4, R6, R8, R9, R12, R10.

```
with_genres=18|10749            sem filtro de classificação
with_runtime.gte=90             with_runtime.lte=150
sort_by=vote_average.desc       vote_count.gte=1000
primary_release_date.gte=<ano atual − 5>-01-01
```

### Caso 3 — sozinho, clássicos

Entrada: sozinho, sem crianças, 110 minutos, humor "adrenalina", clássicos.

Exercita a regra de clássicos e serve de controle para as regras de ajuste que não devem disparar.
R11 detecta que o critério herdado de R6 exige mil votos e o reduz para 200. R7 não se aplica porque
não há restrição infantil, R9 porque não é casal, e R12 porque a sessão não é longa. Os gêneros
selecionados por R3 chegam intactos à consulta.

Regras disparadas: R2, R3, R4, R6, R8, R11, R10.

```
with_genres=28|53               with_runtime.gte=60
with_runtime.lte=110            sort_by=vote_average.desc
vote_count.gte=200              primary_release_date.lte=1999-12-31
```

### Verificações globais

A célula 19 acrescenta três asserções sobre o conjunto dos casos: nenhuma regra dispara mais de uma
vez em uma mesma execução, as doze regras são exercitadas por ao menos um dos casos, e todas as
execuções percorrem os três níveis.

## Trace e explicação

Cada execução registra a ordem de disparo e justifica as decisões individualmente. Saída real do
Caso 1:

```
TRACE (ordem de disparo)
 1. [R5 | nível 1] Sessão em grupo → ordenar por popularidade.
 2. [R1 | nível 1] Há crianças → classificação máxima L.
 3. [R8 | nível 2] 120 min disponíveis, mas com classificação L o limite é 100 min.
 4. [R4 | nível 1] Época 'tanto_faz' → filmes qualquer ano.
 5. [R3 | nível 1] Humor 'medo' → gêneros base: Terror.
 6. [R7 | nível 2] Classificação L proíbe Terror → gêneros passam a ser Animação, Família, Aventura.
 7. [R10| nível 3] Todos os critérios definidos → consulta TMDB montada.

Regras que NÃO dispararam: R2, R6, R9, R11, R12

EXPLICAÇÃO DAS DECISÕES
• Classificação máxima L, porque R1 disparou.
• Gêneros: Animação, Família, Aventura, porque R1, R3 e R7 dispararam.
• Duração entre 60 e 100 min, porque R1 e R8 dispararam.
• Lançamento: qualquer ano, porque R4 disparou.
• Ordenação: mais populares, porque R5 disparou.
• Consulta final à TMDB, porque R1, R3, R4, R5, R7, R8 e R10 dispararam.
```

O trace mostra a ordem em que o motor agiu; a explicação parte de cada decisão e recupera a cadeia
que a sustenta. A segunda linha da explicação ilustra bem o encadeamento: a escolha dos gêneros
depende de R3, mas também de R1, que foi quem tornou a R7 aplicável. O mesmo vale no Caso 3, onde a
ordenação é justificada por R4, R6 e R11 — a época escolhida acabou influenciando o critério de
qualidade.

## Como executar

O notebook roda de ponta a ponta no Google Colab.

1. Abra `Cinefilo_SBC.ipynb` no Colab.
2. Opcionalmente, crie uma conta gratuita na [TMDB](https://www.themoviedb.org/) e gere uma chave em
   *Configurações → API*. Servem tanto a API Key v3 quanto o token de leitura v4.
3. No Colab, abra o painel de *Secrets* na barra lateral, crie o segredo `TMDB_API_KEY` com a chave e
   autorize o acesso do notebook. Se preferir, o notebook pede a chave durante a execução.
4. Execute as células em ordem, ou use *Ambiente de execução → Executar tudo*.

A chave é opcional. Sem ela, o motor de inferência, o trace, as explicações e os três casos de teste
funcionam normalmente; apenas a busca por filmes reais é ignorada.

O notebook instala as dependências na primeira célula e inclui uma correção de compatibilidade
necessária porque a `experta` depende de `frozendict` 1.2, que referencia `collections.Mapping`,
removido no Python 3.10.

## Tecnologias

- experta: motor de inferência com algoritmo RETE
- catálogo de filmes via API do TMDB
- Python 3 / Google Colab

## Autores

- Nicolas Kleiton da Silva Melo - 20240009044
- Rodrigo Monteiro Fortes de Oliveira — 20240097664