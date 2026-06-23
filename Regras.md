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
