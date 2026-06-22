# Gerador de Relatórios de Requilometragem

Prompt de especificação para gerar uma ferramenta de recálculo de distâncias/rotas entre seções tarifárias de um arquivo de itinerário de transporte.

Crie uma página HTML única (HTML+CSS+JS embutidos, sem build step, Leaflet via CDN para mapas, OSRM público para roteamento) que funcione como ferramenta de **recálculo de distâncias/rotas entre seções tarifárias de um arquivo de itinerário de transporte**. Siga rigorosamente o modelo de dados e o fluxo abaixo.

## 1. Formato do arquivo de entrada (.txt contendo JSON)

```json
{
  "Ida": {
    "Partidas": [
      { "Cidade": "...", "Parada": "...", "Latitude": "-23.638998", "Longitude": "-46.537731",
        "Horarios": ["06:00", "07:00", ...], "Secao": true/false, "TipoSecao": "..." }
    ],
    "Frequencias": [ /* códigos de frequência, paralelo às colunas de Horarios, NÃO às Partidas */ ],
    "TiposOnibus": [ /* idem, paralelo às colunas de Horarios */ ],
    "Distancias": [ /* array linear triangular, valores string com VÍRGULA decimal, ex: "117,00" */ ],
    "Tarifas": [ /* mesmo índice de Distancias, string com vírgula decimal, pode vir vazia "" quando ainda não há tarifa cadastrada para o par — isso NÃO impede o cálculo de rota/distância, é só um dado de referência exibido na tabela */ ],
    "QtdSecoes": N,
    "QtdParadasObrigatorias": 0,
    "Itinerario": "texto descritivo do trajeto",
    "UsaMediaPonderada": false,
    "ValorMediaPonderada": ""
  },
  "Volta": { /* mesma estrutura, sentido oposto (UsaMediaPonderada/ValorMediaPonderada são opcionais, podem não existir) */ },
  "Info": { /* metadados da linha: Autos, Empresa, LetraItinerario, CidadeOrigem, CidadeDestino, DataPublicacao, Observacao, etc. */ },
  "Conexoes": [ /* lista de conexões, pode ser vazia */ ],
  "Frequencias": [ /* tabela de definição de cada código de frequência usado em Ida/Volta: {Nome, Dias, Meses, TipoFeriado, TodosFeriados} */ ]
}
```

- **Latitude/Longitude usam PONTO decimal** (`"-23.638998"`), formato já pronto para `parseFloat` direto — não confundir com `Distancias`/`Tarifas`, que usam VÍRGULA decimal (`"117,00"`, exige `replace(',', '.')` antes do parse). Ao exportar, gravar lat/lon de volta no mesmo formato com ponto, como string.
- `Secao=true` define um **ponto de seção** (usado para definir tarifa entre seções). `Secao=false` é uma **parada comum** (ônibus, não-seção). Ambos já existem no arquivo — nenhum dos dois pode ser criado pela ferramenta, apenas reposicionado (lat/lon). A proporção entre os dois tipos varia bastante por **categoria de linha**, e a ferramenta precisa funcionar bem nos dois extremos:
  - **Linhas rodoviárias/intermunicipais longas** (ex.: `607A txt proposto.txt`): poucas paradas no total e praticamente todas são `Secao=true` (no exemplo, 5 de 5 paradas em cada sentido são seção, zero paradas comuns).
  - **Linhas semiurbanas/metropolitanas** (ex.: `OK-0001A-2025-12-11-DOE.txt`): muitas paradas comuns intercaladas com poucos pontos de seção — no exemplo real, a Ida tem 72 paradas no total e só 2 são `Secao=true` (as outras 70 são `Secao=false`); a Volta tem 64 paradas no total, também com só 2 de seção. Ou seja, é esperado ter **dezenas de paradas comuns** na tabela de edição e no mapa de um único sentido — a interface (tabela de pontos, marcadores no mapa, lista "pontos do trecho") deve permanecer usável e performática nesses casos (rolagem na tabela, sem travar o mapa com dezenas de marcadores).
  - **Ida e Volta podem ter quantidades totais de paradas diferentes entre si** (no exemplo OK-0001A, 72 vs. 64) — isso é normal e não é erro; o que precisa ser simétrico/espelhável é apenas a contagem de pontos `Secao=true` (`QtdSecoes`) entre os dois sentidos, não o total de `Partidas`.
- `QtdSecoes` conta apenas as paradas com `Secao=true`; é tipicamente muito menor que `Partidas.length` em linhas semiurbanas, e pode ser igual a `Partidas.length` em linhas rodoviárias sem paradas comuns.
- Indexação triangular de `Distancias`/`Tarifas`: `idx = (k-1)*(k-2)/2 + (i-1)` (1-based, i<k), contando só as seções (não todas as paradas).
- Pareamento Ida↔Volta por posição espelhada: a i-ésima seção de Ida equivale à (N-1-i)-ésima de Volta.
- **Campos a preservar intactos na exportação, sem qualquer alteração**: `Info`, `Conexoes`, `Frequencias` de nível raiz, e dentro de cada sentido `Frequencias`, `TiposOnibus`, `QtdParadasObrigatorias`, `Itinerario`, `UsaMediaPonderada`, `ValorMediaPonderada`, além de `Distancias`/`Tarifas` (nunca recalculados no arquivo, só na tela). A ferramenta deve clonar o JSON original por completo e tocar apenas em `Latitude`/`Longitude`/`Secao`/`TipoSecao`/ordem de `Partidas` dentro de Ida/Volta.

## 2. Carregamento — tudo já calculado de cara

Ao carregar o arquivo (etapa 1), a ferramenta deve, automaticamente e sem botões intermediários:

1. Fazer o parse do JSON e montar o estado por sentido (Ida/Volta).
2. Identificar as seções e montar **todas** as combinações possíveis de trechos — todo par (i, k) com i < k entre as N seções, ou seja, C(N,2) trechos, **independentemente de o par ter tarifa cadastrada ou não**. A existência (ou não) de venda de passagem nunca é critério para calcular ou exibir um trecho — todo trecho é sempre calculado.
3. Disparar as chamadas OSRM para calcular a rota real (Ida e Volta) de cada um desses trechos.
4. Renderizar diretamente os mapas e tabelas de todas as etapas — não existe mais um botão "Montar trechos" ou "Calcular este trecho" para o carregamento inicial (apenas um indicador de progresso enquanto as chamadas OSRM rodam, já que podem ser várias em sequência). Um botão de recálculo manual por trecho pode permanecer disponível para depois de uma edição, mas o carregamento inicial já entrega tudo calculado.

## 3. Etapa "Editar pontos" — layout invertido, sem criação de pontos de seção/não-seção

- **Mapa à esquerda, tabela de pontos à direita**
- Mostra todos os pontos do sentido atual (seção e não-seção), cada um arrastável no mapa — arrastar apenas atualiza lat/lon daquele ponto (sem recalcular rota automaticamente nesta etapa geral; isso acontece nos mapas de trecho).
- A tabela ao lado lista cidade, parada, tipo (Seção/Parada), lat, lon (editável manualmente) — **sem botão de adicionar ponto, sem formulário de criação de parada/seção**. Essa funcionalidade é removida por completo desta etapa.
- Não existe seletor "Tipo de ponto" (parada/ponto de rota), nem campo de cidade/nome para novo ponto, nem checkbox de "espelhar no sentido oposto" — tudo isso pertence apenas ao novo fluxo de pontos de rota dentro do mapa de cada trecho (etapa 4).
- Em linhas semiurbanas a lista de pontos de um único sentido pode ter dezenas de itens (ex.: 70+ paradas comuns, como no `OK-0001A-2025-12-11-DOE.txt`). A tabela deve ter rolagem própria (não estourar a página) e o mapa deve continuar responsivo com esse volume de marcadores — evitar redesenhar todos os marcadores do mapa a cada tecla digitada num input de lat/lon, só ao confirmar a alteração (`onchange`, não `oninput`).

## 4. Mapas de trecho — todas as combinações de seções, único lugar onde se criam pontos (somente "ponto de rota")

### Sem filtro por tarifa — todo trecho é sempre calculado

A ferramenta **não filtra trechos pela existência de tarifa cadastrada**. Para N seções, são sempre montados e calculados **todos os C(N,2) pares (i, k) com i < k**, um card de mapa por par, independentemente de `Tarifas[idx]` estar preenchida, vazia, ou ausente. A coluna de tarifa (ver tabela-resumo, seção 5) é apenas informativa/de conferência — um par sem tarifa cadastrada ainda assim tem sua rota real calculada via OSRM e sua distância exibida normalmente, igual a qualquer outro par. Isso é importante porque o cálculo de distância serve também para **definir** tarifas que ainda não existem no arquivo, não só para conferir as que já existem.

### Layout de cada card de trecho

- **Mapa do trecho à esquerda.**
- **Tabela de pontos daquele trecho à direita** — lista os pontos (seção, paradas comuns e pontos de rota) que compõem o trajeto Ida/Volta daquele trecho, com lat/lon e tipo. Lat/lon continuam editáveis apenas via mapa (drag), nunca direto na tabela desta etapa.

Cada mapa de trecho é renderizado já com a rota Ida (linha sólida) e Volta (linha tracejada, resolvida pela seção espelhada correspondente) calculadas via OSRM no carregamento.

### Trechos sobrepostos e exclusão de pontos **só daquele cálculo** (não exclusão global)

Os trechos frequentemente se sobrepõem: por exemplo, o trecho A×D passa fisicamente pelos mesmos pontos que os trechos A×B, B×C e C×D (a sequência completa é A→B→C→D). Isso é esperado e não deve ser "desduplicado" — cada combinação é um card independente, mesmo repetindo pontos de outros cards.

O motivo prático: o ônibus às vezes faz o trajeto A→D sem realmente passar fisicamente por todas as paradas intermediárias de B e C (ex.: variações de itinerário, ou paradas que só fazem sentido operacional para quem embarca/desembarca em B ou C, não para quem viaja direto de A a D). Quem decide se um ponto intermediário deve ou não entrar no cálculo de distância de **cada combinação específica** é o usuário, caso a caso.

Por isso, a tabela de pontos de cada card de trecho deve ter:

- Um **checkbox por ponto intermediário** (seção, parada comum, ou ponto de rota) permitindo incluir/excluir aquele ponto **apenas do cálculo daquele card específico** — a mesma parada física pode estar marcada como incluída no trecho A×B e excluída no trecho A×D, simultaneamente, sem conflito.
- Os pontos de **origem e destino do próprio trecho** (os dois pontos de seção que definem o card) são sempre obrigatórios — checkbox desabilitado/sempre marcado, não pode ser excluído.
- Ao (des)marcar o checkbox, a rota daquele card é recalculada via OSRM imediatamente, passando a incluir/ignorar aquele ponto como waypoint forçado, e a distância/badge daquele card é atualizada.
- Essa seleção deve ser mantida em um estado **por trecho** (chave = par de seções do card, ex. `i-k`), nunca em um estado global por ponto — exatamente para suportar o cenário acima, em que o mesmo ponto pode ter status diferente em cards diferentes.

### Criação de ponto de rota (único tipo de ponto criável, e só dentro do mapa do trecho)

Não há formulário, seletor de tipo, nem botão "definir local no mapa". O processo é 100% feito clicando diretamente sobre a linha do itinerário já desenhada:

- Ao clicar em qualquer ponto sobre a polyline da rota (Ida ou Volta, conforme a linha clicada), um **ponto de rota** é criado exatamente naquela coordenada e inserido na sequência de pontos daquele sentido, na posição correta (entre os dois pontos cujo segmento foi clicado).
- A rota é imediatamente recalculada via OSRM passando a forçar a passagem por esse novo ponto, e a polyline é redesenhada.
- **Arrastar** um ponto de rota (recém-criado ou já existente) para uma nova posição deve, ao soltar, disparar novo cálculo OSRM e redesenhar a rota passando pela nova posição.
- **Clique com o botão direito** em um ponto de rota abre uma opção de exclusão; confirmando, o ponto é removido da sequência e a rota é recalculada/redesenhada sem ele.
- Pontos de seção e não-seção continuam podendo ser arrastados dentro do mapa de trecho (corrige geolocalização), também disparando recálculo da rota — mas eles nunca podem ser excluídos nem criados, só reposicionados.
- Cliques fora da linha da rota (área vazia do mapa) não criam nada — criação só ocorre clicando sobre a própria rota desenhada.

## 5. Tabela-resumo e exportação

- Tabela-resumo: origem, destino, pontos intermediários considerados, distância Ida/Volta/média, tarifa e `Distancias[]` original do arquivo, recalculada automaticamente após qualquer alteração nos trechos. Lista **todos** os trechos calculados (todos os C(N,2) pares de seções), inclusive os que ainda não têm tarifa cadastrada no arquivo — para esses, a coluna de tarifa aparece vazia, mas a distância calculada é exibida normalmente.
- A exportação gera **dois arquivos**, baixados juntos, para o usuário guardar na mesma pasta:

### 5.1. Arquivo 1 — 607A alterado (mesmo nome de entrada + sufixo)

- Clonar o JSON original **por completo** (`Info`, `Conexoes`, `Frequencias` raiz, e dentro de Ida/Volta: `Frequencias`, `TiposOnibus`, `QtdParadasObrigatorias`, `Itinerario`, `UsaMediaPonderada`, `ValorMediaPonderada`, `Distancias`, `Tarifas`, `QtdSecoes`) sem qualquer alteração.
- Dentro de `Ida.Partidas` e `Volta.Partidas`, atualizar apenas `Latitude`/`Longitude` (string com ponto decimal, mesmo formato do arquivo original) dos pontos de seção e não-seção conforme reposicionados na ferramenta.
- **Remover da lista de Partidas exportada qualquer ponto de rota** — eles nunca entram nesse arquivo, pois não fazem parte do esquema oficial do 607A.
- `Distancias`/`Tarifas` permanecem exatamente como vieram do arquivo original — nunca sobrescritos pelos valores calculados em tela.
- Nome de saída: `<nome-base-do-arquivo-carregado>_recalculado.txt`.

### 5.2. Arquivo 2 — pontos de rota e seleção de pontos por trecho, para complementação

Este arquivo guarda **tudo que não cabe no esquema oficial do 607A**, mas que é necessário para reproduzir exatamente o relatório que o usuário montou na tela: (a) os pontos de rota criados, e (b) quais pontos cada trecho considerou ou ignorou no cálculo (a seleção descrita na subseção anterior). Sem isso, recarregar só o Arquivo 1 não seria suficiente para reobter o mesmo relatório.

- Como pontos de rota não têm índice oficial no esquema 607A, cada um deve ser serializado referenciando seus **vizinhos na sequência** (cidade+parada do ponto anterior e do ponto seguinte no mesmo sentido), além de sentido (Ida/Volta), lat e lon — isso permite reinserir o ponto na posição correta mesmo após reabrir o arquivo original em outra sessão.
- Cada **trecho** (combinação i-k) deve gravar a lista de pontos que o usuário **excluiu** daquele cálculo específico (a exclusão é a exceção; por padrão todo ponto entra). Identificar cada ponto excluído por: sentido, cidade+parada (para pontos originais) ou referência de vizinhos (para pontos de rota, igual ao item anterior), evitando depender de índices que mudam quando o arquivo é editado.
- Formato sugerido:
  ```json
  {
    "arquivoOrigem": "607A txt proposto.txt",
    "pontosDeRota": [
      {
        "sentido": "ida",
        "lat": -23.55,
        "lon": -46.6,
        "anterior": { "cidade": "São Paulo", "parada": "Rodov. Barra Funda" },
        "seguinte": { "cidade": "Sorocaba", "parada": "Term. São Paulo" }
      }
    ],
    "selecaoTrechos": [
      {
        "trecho": {
          "origem": { "cidade": "Santo André", "parada": "Rodov." },
          "destino": { "cidade": "Sorocaba", "parada": "Term. Rodov." }
        },
        "pontosExcluidosIda": [
          { "cidade": "São Caetano do Sul", "parada": "Rodov." }
        ],
        "pontosExcluidosVolta": []
      }
    ]
  }
  ```
- Nome de saída: `<nome-base-do-arquivo-carregado> Pontos de rota para complementação 607A.txt`, baixado junto ao Arquivo 1, para o usuário manter os dois na mesma pasta.
- Ao recarregar o arquivo .txt original **junto com este arquivo de complementação**, a ferramenta deve restaurar exatamente o mesmo relatório: os mesmos pontos de rota nas mesmas posições, e cada card de trecho com os mesmos pontos marcados/desmarcados — sem o usuário precisar refazer nenhuma escolha manualmente.

### 5.3. Tabela final em formato matriz (substitui/complementa a tabela-resumo linear)

Além da tabela-resumo linear (uma linha por trecho, já descrita acima), a ferramenta deve apresentar **ao final** uma segunda visualização em formato de **matriz triangular**, usando como rótulos de linha/coluna as seções na mesma ordem em que aparecem no sentido Ida (a primeira seção = `a`, a segunda = `b`, etc. — usar o nome real `Cidade - Parada` da seção como rótulo, a letra é só ilustrativa). Formato:

```
        a       b       c       d
  a     X
  b   118,3     X
  c   145,0   95,2      X
  d   210,7  180,4    70,1      X
```

Onde `a`, `b`, `c`, `d` = seções na ordem da Ida (ex.: `a` = "Santo André - Rodov."). Regras de montagem:

- **Matriz triangular inferior apenas**: a diagonal principal é sempre `X` (distância de uma seção para ela mesma não existe); a parte abaixo da diagonal é preenchida com **todos** os pares (i, k), sem excluir nenhum por falta de tarifa; a parte acima não é exibida (a matriz de distância é simétrica por natureza — Ida e Volta já foram combinadas na média `distMedia` de cada trecho, então não há "ida" e "volta" separadas nessa matriz, só o valor final consolidado).
- Cada célula abaixo da diagonal mostra a **distância média (`distMedia`)** calculada para aquele par de seções, formatada com vírgula decimal (mesmo padrão do arquivo original, ex. `118,30`) — não o valor antigo de `Distancias[]` do arquivo (esse já aparece para conferência na tabela-resumo linear). Isso vale mesmo para pares sem tarifa cadastrada no arquivo: a distância é calculada e mostrada igual.
- Pares **ainda não calculados** (ex.: falha pontual de OSRM) devem mostrar um indicador de pendência (ex.: `?`), nunca `0` (zero é um valor de distância real e não pode ser confundido com "não calculado").
- Essa matriz é só para conferência visual rápida — não é exportada como parte do Arquivo 1 (que mantém os campos `Distancias`/`Tarifas` originais intocados) nem precisa virar um terceiro arquivo; ela vive apenas na tela.

## 6. Regras de negócio a preservar/ajustar

- Resolver posição de pontos sempre por identidade do objeto (`indexOf`) no momento do uso, nunca por índice cacheado.
- Detectar em qual segmento da polyline o clique ocorreu (ponto mais próximo do segmento) para inserir o novo ponto de rota na posição correta da sequência daquele sentido.
- Toda alteração (drag, criação ou exclusão de ponto de rota) num mapa de trecho deve disparar recálculo OSRM apenas daquele trecho/sentido afetado, atualizando a tabela-resumo em seguida.
- Validar igualdade do número de seções Ida/Volta, mas seguir mesmo com aviso.
- Trechos sobrepostos (ex.: A×D contém fisicamente os mesmos pontos de A×B, B×C, C×D) são todos exibidos como cards independentes — nunca "deduplicar" ou ocultar um trecho por já estar contido em outro maior. A inclusão/exclusão de um ponto intermediário do cálculo é sempre local a um card específico, nunca propagada para outros cards que compartilhem o mesmo ponto físico.

## 7. Regras de organização, legibilidade e boas práticas de código

O código continua sendo um único arquivo HTML autocontido (sem build step), mas isso não é desculpa para desorganização. Seguir:

### 7.1. Estrutura geral do arquivo

- Ordem fixa de blocos: `<style>` no `<head>`, markup no `<body>`, e todo o JavaScript num único `<script>` ao final do `<body>` — nunca misturar `<script>` espalhados em vários pontos do HTML.
- Dividir o JS em **seções nomeadas com comentário de cabeçalho em caixa alta**, na ordem em que o fluxo de uso acontece (ex.: `ESTADO GLOBAL`, `CARREGAR ARQUIVO`, `MAPA DE EDIÇÃO`, `SEÇÕES E TRECHOS`, `OSRM`, `RESUMO E Distancias[] ORIGINAL`, `EXPORTAÇÃO`). Cada bloco deve conter só funções daquele domínio — não espalhar lógica de exportação no meio do bloco de mapa, por exemplo.
- Dentro de cada bloco, ordenar funções da mais "alto nível" (entrada do fluxo) para a mais utilitária (detalhe de implementação), de forma que o arquivo possa ser lido de cima para baixo como uma narrativa.

### 7.2. Estado e nomenclatura

- Manter um número pequeno e explícito de variáveis de estado global (equivalente a `RAW`, `DATA`, `CURRENT_DIR`, `TRECHOS`), cada uma com nome e comentário de uma linha explicando o que guarda. Não introduzir estado global novo sem necessidade — preferir derivar valores a partir do estado existente.
- Escolher **um idioma para identificadores de código (português) e manter 100% consistente** — nomes de função, variável e propriedade de objeto sempre em português, sem misturar com inglês (ex.: não usar `loadDirection` e `calcularTrecho` no mesmo arquivo; escolher um padrão e segui-lo). Strings de UI e mensagens ao usuário também em português.
- Nomes de função devem descrever a ação (`calcularTrecho`, `redrawEditMap`, `exportarPontosRota`), nunca abreviações obscuras. Nomes de variável devem indicar o conteúdo, não o tipo (`pontos`, não `arr` ou `lst`).
- IDs de elementos DOM gerados dinamicamente devem seguir um padrão único e previsível (`map_${ti}`, `info_${ti}`, `badge_${ti}`) — todo elemento de um mesmo card de trecho usa o mesmo índice `ti` como sufixo, nunca índices recalculados de formas diferentes em pontos diferentes do código.

### 7.3. Separação de responsabilidades

- Separar claramente três tipos de função: (1) **cálculo/transformação de dados** (não tocam DOM, retornam valores — ex.: `getPointsBetween`, `idxDistanciasOriginal`), (2) **acesso a rede** (chamadas OSRM, sempre `async` com tratamento de erro), e (3) **renderização** (leem o estado e escrevem no DOM — `render*`, `redraw*`). Uma função de renderização nunca deve conter lógica de negócio "escondida"; ela só lê o estado já calculado e o exibe.
- Toda lógica repetida em mais de um lugar (ex.: inserir ponto na sequência, montar ícone de marcador, popular um `<select>` com pontos) deve existir em **uma única função compartilhada**, reutilizada pelo mapa principal e pelos mapas de trecho — nunca duplicar a mesma lógica em dois lugares "porque é mais rápido copiar e colar".
- Funções de renderização devem ser **idempotentes**: chamar duas vezes a função que desenha um mapa ou tabela deve produzir o mesmo resultado final, sem acumular marcadores/linhas duplicadas (sempre limpar o que existia antes de redesenhar).

### 7.4. Tratamento de erros e robustez

- Toda chamada `fetch` ao OSRM deve estar em `try/catch`, nunca deixar uma promise rejeitada sem tratamento. Em caso de erro, mostrar mensagem clara no card do trecho afetado (ex.: "Erro ao calcular: <motivo>") sem travar o restante da página.
- Parse de JSON do arquivo carregado deve estar em `try/catch` com mensagem de erro amigável ao usuário (`alert` ou área de erro na tela), nunca deixar uma exceção não tratada quebrar o carregamento silenciosamente.
- Qualquer dado vindo do arquivo do usuário que seja inserido no DOM via `innerHTML` deve passar por uma função de escape de HTML (`escapeHtml`) — nunca interpolar `Cidade`/`Parada`/`Itinerario` direto em template strings que vão para `innerHTML`, para evitar XSS caso o arquivo contenha conteúdo malicioso.
- Validar previamente que os campos esperados existem antes de usá-los (`src.Partidas || []`, `RAW.Ida?.Tarifas`), em vez de assumir que o arquivo sempre tem o formato perfeito.

### 7.5. Desempenho e uso responsável da API pública do OSRM

- Como o OSRM público tem limite de uso, as chamadas de cálculo de todos os trechos (todos os C(N,2) pares) no carregamento inicial devem ser feitas **sequencialmente** (uma de cada vez, aguardando a resposta anterior), nunca em paralelo descontrolado (`Promise.all` disparando dezenas de fetches simultâneos) — mostrar um indicador de progresso ("Calculando trecho 3 de 7...") durante esse processo.
- Após qualquer edição (drag, criação ou exclusão de ponto de rota), recalcular **apenas o trecho afetado**, nunca todos os trechos novamente — local de impacto da mudança é o card do trecho onde a edição ocorreu.
- Evitar recomputar uma rota que não mudou: se o usuário clicar em "recalcular" sem ter alterado nenhum ponto daquele trecho desde o último cálculo, a ferramenta pode pular a chamada e reaproveitar o resultado já em memória (registrar um flag simples de "sujo"/"limpo" por trecho).

### 7.6. CSS e marcação

- Reaproveitar classes CSS para padrões visuais repetidos (cor por tipo de ponto, badge de status, layout mapa/tabela) — evitar estilos inline espalhados pelo JS, exceto quando o valor é genuinamente dinâmico (ex.: cor calculada em tempo de execução para o ícone do marcador).
- Manter o HTML estático (estrutura dos cards fixos: carregar arquivo, editar pontos, trechos, resumo, exportar) separado do HTML gerado dinamicamente via template string no JS — o primeiro fica no `<body>`, o segundo é sempre montado por função de renderização dedicada.

### 7.7. Comentários

- Comentar **só o que não é óbvio pelo nome da função/variável**: fórmulas (índice triangular, espelhamento Ida↔Volta), decisões não triviais (por que a seleção de pontos excluídos é mantida por trecho e não globalmente), e armadilhas conhecidas (por que resolver posição por identidade de objeto e não por índice cacheado).
- Não comentar o óbvio (`// adiciona o ponto à lista` acima de `lista.push(ponto)`) e não deixar comentários sobre o processo de implementação ("// TODO: melhorar depois", código morto comentado) — remover, não comentar, código que não é mais usado.
