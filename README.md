# Da Reunião ao Documento: Design Docs Gerados por IA

Entrega do desafio do módulo **Design Docs com IA** (MBA em Engenharia de Software com IA): a transcrição de uma reunião técnica virou um pacote de design docs da feature **Sistema de Webhooks de Notificação de Pedidos** — PRD, RFC, FDD, 8 ADRs e um tracker de rastreabilidade. O enunciado original está no repositório base ([enunciado no commit inicial](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia/blob/e7f6311b96ee3b21bf8cb7077f53a6e4045143eb/README.md)); este README documenta **como** o pacote foi produzido.

## Sobre o desafio

A empresa do cenário opera um Order Management System (Node.js, TypeScript, Express, Prisma, MySQL) e decidiu, numa reunião de ~55 minutos entre tech lead, PM, dois engenheiros e segurança, construir webhooks de notificação de mudança de status de pedido. Nada foi registrado além da transcrição. O trabalho era transformar essa transcrição, mais o código existente, em documentação técnica acionável para o time começar a implementar: PRD (por quê e o quê), RFC (a proposta e o que está em aberto), ADRs (cada decisão isolada), FDD (como construir) e um tracker ligando cada item à sua origem.

A regra central do desafio é a que mais moldou o processo: **toda informação registrada precisa ser rastreável à transcrição ou ao código**. Requisito, decisão ou restrição sem origem identificável não pode entrar. E o que a reunião descartou ou adiou tem de ficar de fora. Por isso o processo foi desenhado em torno de duas perguntas, feitas a cada linha produzida: "de onde isso veio?" e "isso foi decidido, ou foi só mencionado?".

## Ferramentas de IA utilizadas

| Camada | Ferramenta | Papel |
|---|---|---|
| Geração | Claude Opus 5.5, raciocínio `xhigh`, acionado por um harness interno de chamada única (`ask`), sem ferramentas e somente leitura | Toda a redação: inventário da transcrição, mapa do código, ADRs, RFC, FDD, PRD. Também fez a auditoria adversarial, em contexto novo, sobre os documentos já escritos |
| Coordenação | Agente `pi` (modelo `glm-5.4`, da Zhipu AI) | Escreveu os prompts, montou o material de origem de cada chamada, leu cada saída contra a transcrição e o código, aplicou correções, consolidou lacunas e montou o pacote |
| Revisão de segurança | Subagente `pi` isolado (modelo `glm-5.4`, somente leitura, sem acesso ao processo de redação) | Revisão adversarial do desenho em contexto novo: segurança, contradições entre documentos e itens descartados que não deveriam aparecer |
| Deliberação | Council consultivo de modelos de famílias diferentes: GLM-5.4 (Zhipu AI) e Qwen3.6-27B (Alibaba) como membros; Llama-3.3-70B (Meta) como chairman | Deliberou, em três estágios (resposta independente, avaliação por pares às cegas, síntese), sobre o bloqueio apontado pela revisão de segurança e as condições de publicação |
| Verificação | Script Python próprio, sem LLM (não faz parte da entrega, por ser documental) | Conferiu mecanicamente caminhos de arquivo, timestamps e falantes, IDs, cobertura do tracker, seções e contagens dos critérios de aceite; tem controles negativos |

O gerador não teve acesso livre ao repositório: só recebe o que o coordenador coloca no prompt. Cada chamada de redação leva a transcrição integral e os arquivos de código relevantes, e devolve só texto.

## Workflow adotado

A ordem seguiu a sugerida no enunciado, com duas etapas antes e duas depois:

1. **Inventário dirigido da transcrição.** Um prompt de extração classificou cada fala relevante em decisão, requisito, alternativa descartada, item adiado, questão em aberto, limitação, correção feita na própria reunião e ambiguidade. Cada item saiu com uma **citação literal**. Um script conferiu as 259 citações contra a linha do falante naquele minuto: 100% batem. O inventário foi o controle de completude — cada decisão e cada requisito extraído foi depois procurado nos documentos.
2. **Mapa do código.** Um segundo prompt respondeu a 15 perguntas específicas sobre o código (o que `changeStatus` faz na transação, quais classes de erro aceitam código próprio, o que o middleware responde, como são os testes). Ele revelou armadilhas que os documentos precisavam registrar: `NotFoundError` grava código fixo, um schema Zod rejeitado responde `VALIDATION_ERROR` e não um código do módulo, e a lista de redação do logger não cobre a secret.
3. **ADRs.** Oito chamadas, uma por decisão, cada uma com a transcrição, os arquivos de código pertinentes e um breve com as alternativas que foram de fato discutidas.
4. **RFC.** Duas chamadas (metadados, contexto e proposta; alternativas, questões em aberto, impacto e decisões relacionadas), sobre os ADRs já escritos, mais uma reformulação compacta da segunda metade (ver Ajuste 7).
5. **FDD.** Nove chamadas, uma por bloco de seções, com um bloco de convenções compartilhadas para as partes não divergirem: nomes de arquivos e tabelas, IDs fixos dos contratos e um **registro de lacunas** com numeração fixa.
6. **PRD.** Duas chamadas, sobre a transcrição e o resumo dos ADRs.
7. **Tracker.** Cada documento sai da chamada com uma tabela "fontes por ID", escrita no momento da redação. O tracker é a junção dessas tabelas.
8. **Revisão em três camadas:** leitura do coordenador contra as fontes; auditoria adversarial pelo gerador em contexto novo (11 chamadas: 8 de fidelidade à transcrição, 2 só de afirmações sobre código e 1 sobre o RFC final); revisão de segurança do desenho por um subagente isolado, com acesso somente leitura; deliberação por council multiprovider sobre o bloqueio encontrado; validador determinístico.
9. **README** por último.

Como a interação com a IA foi organizada:

- **Regras de integridade fixas** em todo prompt de redação: origem única (transcrição ou código), nada inventado para preencher lacuna, item descartado nunca vira requisito, afirmação retificada vale só na forma final, ambiguidade não se resolve em silêncio.
- **ID em tudo que é normativo.** Cada requisito, decisão, contrato, erro, fluxo e risco recebe um ID (`PRD-FR-01`, `RFC-ALT-02`, `FDD-CONTRATO-03`, `ADR-002`). Isso tornou a cobertura verificável por script.
- **Três marcações separando o que é fato do que é hipótese:** `(novo)` para arquivo que a implementação vai criar; `(proposta, ver FDD-PROP-NN)` para detalhe que a reunião não decidiu; `(consequência técnica inferida)` para dedução a partir do código.
- **Lacuna com dono.** Onde a reunião parou antes do detalhe de implementação, o FDD registra a lacuna, uma proposta de trabalho e quem confirma. A reunião fechou o desenho; não fechou, por exemplo, o que conta como entrega bem-sucedida, o método HTTP da entrega nem o que acontece com um evento quando o pedido anterior está em nova tentativa.
- **Tarefas pequenas por chamada.** O limite de saída do gerador (65.536 tokens) cobre raciocínio e texto, e há um corte de tempo no provedor. Tarefas grandes eram truncadas em silêncio. Por isso tudo foi particionado, e cada resposta passou a terminar com um marcador cuja ausência denuncia corte.

## Prompts customizados

Três dos prompts centrais, abreviados apenas onde entrava material de origem — a transcrição e os arquivos de código, sempre incluídos por inteiro na chamada.

**1. Inventário dirigido da transcrição** — o prompt que faz a filtragem do que entra e do que não entra:

```text
# Papel
Você é analista de requisitos sênior. Sua tarefa é transformar a transcrição de uma
reunião técnica em um inventário completo e fiel [...]

# Princípios (inegociáveis)
1. A reunião é a única fonte. Não complete lacunas com boas práticas de mercado.
2. Não resolva ambiguidades: registre-as, com as leituras possíveis e os timestamps.
3. Uma frase só vale na forma corrigida, quando foi retificada mais tarde.
   A forma original vai para o tipo CORRECAO.
4. "Mencionado" não é "decidido". Uma opção que alguém levanta e a equipe rejeita é
   ALTERNATIVA_DESCARTADA; um pedido que a tech lead recusa é FORA_DE_ESCOPO ou ADIADO;
   um ponto que fica "para decidir depois" é QUESTAO_ABERTA.
5. Um item por fato.
6. Percorra a transcrição linha a linha, do primeiro ao último timestamp.

# Tipos (campo `tipo`)
CONTEXTO | DECISAO | REQUISITO_FUNCIONAL | REQUISITO_NAO_FUNCIONAL | RESTRIÇÃO |
ALTERNATIVA_DESCARTADA | ADIADO | FORA_DE_ESCOPO | QUESTAO_ABERTA | LIMITACAO_CONHECIDA |
CORRECAO | GANCHO_CODIGO | DETALHE_TECNICO | PRAZO | AMBIGUIDADE

# Material de origem (única fonte permitida)
[TRANSCRICAO.md integral]

# Formato de saída — somente JSON:
{ "itens": [ { "id", "timestamp": "[09:08]", "falante", "tipo", "resumo",
  "citacao": "trecho literal e contínuo, copiado caractere a caractere da fala desse
              falante nesse timestamp",
  "fechado_por": "[09:08] Larissa", "motivo", "corrigido_por", "relacionados", "observacao" } ] }
```

**2. Redação de um ADR** — a estrutura se repetiu nos oito, mudando o breve:

```text
# Papel
Você é engenheiro sênior do time de Plataforma [...]. Sua tarefa é registrar, em um ADR,
UMA decisão arquitetural fechada numa reunião técnica.

# Tarefa
Registre a decisão ADR-003 — Retry com backoff e fila de mortos (DLQ)
(arquivo docs/adrs/ADR-003-retry-com-backoff-e-dlq.md).

- Decisão a registrar: novas tentativas com intervalos de 1 minuto, 5 minutos,
  30 minutos, 2 horas e 12 horas, até o teto de 5 tentativas; esgotado o teto, o evento
  vai para uma tabela separada webhook_dead_letter [...] Fechada em [09:17] Larissa.
- Alternativas que devem constar:
  ADR-003-ALT-1 — retry indefinido (descartado em [09:15] Diego)
  ADR-003-ALT-2 — 3 tentativas (proposto em [09:16] Bruno; descartado em [09:16] Diego)
  ADR-003-ALT-3 — marcar como "failed" na própria outbox ([09:17] Larissa; [09:18] Diego)
- Interpretação a confirmar (obrigatória): a transcrição diz "5 tentativas" e lista cinco
  intervalos, com "quase 15 horas entre primeira falha e última tentativa". Adote a leitura
  "1 envio inicial + 5 novas tentativas", justifique, confira a soma e registre a outra
  leitura como ponto a confirmar.
- Atenção: não defina o que a reunião não discutiu (formato da assinatura, por exemplo).

# Regras de integridade (valem para o documento inteiro)
1. Origem única: só vira requisito, decisão, restrição, número ou nome aquilo que está na
   transcrição anexa ou é fato verificável no código anexo.
2. Nada inventado para preencher lacuna [...]
11. Caminho de arquivo existente vai entre crases e é copiado exatamente; arquivo que a
    implementação vai criar vem seguido de (novo).

[Formato MADR em português: Status, Contexto, Decisão, Alternativas Consideradas,
 Consequências (positivas, negativas e trade-off explícito), Referências]

## Fontes por ID (obrigatório, no fim da resposta)
<!-- FONTES-POR-ID -->
| ID | Tipo | Conteúdo (resumo) | Fonte | Localização |
Fonte é TRANSCRICAO ou CODIGO. Para TRANSCRICAO, Localização é `[hh:mm] Nome`.

[TRANSCRICAO.md integral] [arquivos de código integrais]
```

**3. Auditoria adversarial** — contexto novo, sem acesso ao processo de redação:

```text
# Papel
Você é revisor adversarial de documentação técnica. Você não escreve documentos:
procura erros de fidelidade à fonte.

# O que procurar (nesta ordem de gravidade)
- P0 Contradição: o documento afirma algo que contraria a transcrição ou o código.
- P0 Invenção: requisito, decisão, restrição ou número apresentado como vindo da reunião
  ou do código, mas ausente das duas fontes, e sem marcação de proposta.
- P1 Atribuição errada: a citação [hh:mm] Falante não sustenta a afirmação ligada a ela.
  Atenção: a transcrição tem várias falas do mesmo falante no mesmo minuto.
- P1 Conflito entre partes: a mesma coisa dita de formas incompatíveis.
- P2 Imprecisão: paráfrase que perde ou acrescenta detalhe sem mudar o sentido.
Não reporte estilo. Só erros de fidelidade à fonte.

# Formato de saída
### A-01 — <P0|P1|P2> — <documento> — <ID ou seção>
- Trecho / Problema / Evidência ("até 20 palavras literais da fala") / Correção sugerida
## Conferido sem problema
- Citações conferidas: <n> · Afirmações sobre código conferidas: <n>

[trecho do documento] [TRANSCRICAO.md integral] [arquivos de código]
```

## Iterações e ajustes

Foram **6 ciclos principais** de geração, revisão e ajuste, além da preparação. Os momentos em que a IA errou ou ficou superficial e a correção aplicada:

**Ajuste 1 — inventário truncado em silêncio.** A primeira versão do inventário, numa chamada única, parou no meio do JSON. O gerador não devolve erro: o limite de saída cobre o raciocínio, e em `xhigh` o raciocínio consome a maior parte. Correção: quatro janelas de tempo, uma detecção de corte por marcador e, a partir daí, toda tarefa particionada. A mesma causa cortou uma parte do FDD, que foi refeita em duas chamadas, e a primeira tentativa de encurtar o RFC.

**Ajuste 2 — o gerador se recusou a inventar.** O primeiro prompt do inventário saiu, por erro de montagem do coordenador, sem a transcrição. Em vez de preencher com "boas práticas", o modelo devolveu uma lista vazia explicando que a fonte estava ausente. Foi o sinal de que as regras de integridade estavam funcionando; o prompt foi corrigido.

**Ajuste 3 — HTTPS e limite de 64KB não viraram ADR.** Na reunião, a própria tech lead classificou esses dois pontos como requisito não funcional, não como decisão arquitetural. Eles ficaram no PRD e no FDD. Em compensação, duas decisões complementares em que houve alternativa real discutida viraram ADR: o snapshot do payload e o filtro de eventos na inserção.

**Ajuste 4 — inconsistências entre partes geradas em paralelo.** O FDD foi escrito em nove chamadas. A revisão achou divergências: a edição aceitar `customerId` numa parte e não aceitar em outra; o histórico de entregas com `result` num trecho e `success` em outro; "só cadastros ativos" afirmado como fato em vez de proposta; um exemplo de `order_number` como `000123`, quando o código gera `ORD-000123`. No PRD, a parte de critérios de aceitação citou **IDs de requisito errados em 14 critérios e 8 testes** — os rótulos estavam certos, os números não. Tudo foi harmonizado, e o validador ganhou uma regra de referência a ID inexistente.

**Ajuste 5 — propostas sem marcação.** A auditoria adversarial produziu 79 achados. A classe mais comum foi detalhe de implementação apresentado como decidido: processar o lote em paralelo, a escolha do pacote `uuid`, o formato da resposta da rotação de secret, incluir `*.secret` na lista de redação do logger, o nome `webhook.processor.ts`. Cada um foi rebaixado a proposta, com ID. A auditoria também pegou atribuições de fala imprecisas (quem fecha o snapshot é Bruno, em [09:52], e não Larissa; "prazo confirmado com a Atlas" quando a confirmação ainda é pendente), riscos inferidos sem marcação, uma contradição interna no PRD sobre a ordem de entrega durante novas tentativas e nove imprecisões sobre o código, como dizer que `env.ts` lê a URL do banco quando ele só a valida.

**Ajuste 6 — propostas ganharam número e dono.** Várias partes sugeriram lacunas que o coordenador não tinha listado: onde termina a contagem dos 10 segundos, o método HTTP da entrega, redirecionamento, o que acontece no encerramento do worker com lote em andamento, e o fato de o worker passar a exigir `JWT_SECRET` só por importar a configuração de ambiente. Foram consolidadas na seção 15 do FDD, que fecha com 47 lacunas e 45 propostas. Uma lacuna foi removida por não ter origem nem na transcrição nem no código. Uma lacuna de segurança entrou por leitura crítica do desenho, e não por indicação da IA: a URL do webhook é informada pelo usuário e o worker faz a requisição — risco de a plataforma virar cliente de endereços internos, que a reunião não discutiu.

**Ajuste 7 — RFC longo demais.** A primeira versão tinha 2.803 palavras, acima do alvo de 2 a 4 páginas. As seções de alternativas, questões em aberto e impacto foram refeitas em formato de tabela, com a restrição de manter exatamente os mesmos IDs e citações. O resultado tem cerca de 2.500 palavras. Um script comparou IDs e citações antes e depois: nada se perdeu.

**Ajuste 8 — o harness censurava a resposta.** A camada de saneamento do harness troca por `[REDACTED]` qualquer valor que segue uma chave como `secret` ou `token`. Exemplos de contrato perdiam o campo. Correção: a IA passou a usar um placeholder explícito e cada saída foi revista quanto a marcadores de censura.

**Ajuste 9 — a revisão de segurança isolada mudou a severidade de um risco.** O risco de "qualquer usuário autenticado cadastrar um webhook para qualquer cliente" estava registrado como probabilidade média, herdando a leitura da reunião. O revisor isolado mostrou o encadeamento que faltava: no código o registro de usuário é público e não há vínculo entre usuário e cliente, então "usuário autenticado" significa qualquer pessoa. O risco passou a alto e bloqueante de produção, com uma proposta que reabre a decisão da reunião — e por isso está marcada como proposta, não como requisito. Entraram também a leitura de serviços internos pela resposta guardada no histórico, a amplificação do trabalho do worker pelo número de cadastros e o uso da secret antiga por 24 horas após a rotação. Um achado do revisor foi aceito com ressalva: inverter a proposta de rotação é razoável, mas o ganho de segurança alegado é menor do que o revisor afirma, porque a janela de falsificação depende de o cliente aceitar a secret antiga, e não de qual secret a plataforma usa para assinar. As duas leituras estão registradas com esse trade-off.

**Ajuste 10 — o bloqueio foi para o council, não para a decisão isolada do coordenador.** A revisão de segurança isolada deixou um veredito "bloqueado". Em vez de o coordenador decidir sozinho, o caso foi a um council: dois modelos de provedores diferentes do gerador responderam à mesma consulta, avaliaram às cegas a resposta um do outro, e um terceiro modelo sintetizou. O council foi unânime em três pontos: o bloqueio é do deploy, não da entrega documental; o tratamento dado respeita a fidelidade à reunião; e a publicação pode seguir com ajustes. Os ajustes pedidos e aplicados: separar no texto do risco o que é pré-existente do que a feature acrescenta; dizer que "bloqueante" significa bloqueio de deploy, ancorado na revisão de segurança que o próprio time agendou ([09:46] Sofia); e anotar no requisito derivado de [09:37] que a premissa de [09:32] não vale no código. A recomendação de engenharia do council, a levar à revisão de segurança: corrigir primeiro o registro público que aceita `role`, porque sem isso restringir a ADMIN não tem efeito; restringir o gerenciamento de webhooks a ADMIN como contenção; e, no médio prazo, vincular usuário a cliente e checar propriedade. Um pedido do council foi rebatido: criar um ADR para o conflito. O enunciado reserva os ADRs — no máximo oito — a decisões fechadas na reunião, e o conflito já está registrado como questão em aberto no RFC e como lacuna e proposta no FDD. Os membros também apontaram um erro "do coordenador" que era da consulta: a minha exposição omitira [09:21], que pede o endpoint de rotação de secret; corrigi a consulta antes da síntese. O gerador não participou do council: a rota dele recusou a consulta em um segundo, provavelmente por filtro de moderação do provedor sobre o conteúdo. Não havia como contornar, e ele era o interessado.

**Ajuste 11 — auditoria linha a linha por modelos de outras famílias.** As rotas de validador configuradas no ambiente continuavam fora. Para a conferência não ficar só a cargo de quem escreveu, rodei a mesma auditoria de fidelidade com modelos de três famílias diferentes da do gerador e da do coordenador: Llama-3.3-70B em 9 trechos, Qwen3.6-27B em 17 sub-trechos e Gemma-4-12B em 1 trecho. Foram 206 achados. Triagem um a um contra a transcrição e o código: nenhum apontou contradição ou invenção. A esmagadora maioria não se sustentava: acusava de "não está na transcrição" uma frase que está lá literalmente, ou cobrava da transcrição um fato do código anexo ou uma proposta declarada como proposta. Cinco renderaram ajustes de citação ou de redação. Limite honesto: os três modelos rodaram sem raciocínio. A verificação é independente, mas rasa.

A tentativa com raciocínio foi parcial. O Qwen com raciocínio só aceita pedaços pequenos: o servidor encerra requisições acima de cerca de dez minutos, e com pedaços maiores o modelo gastava todo o limite de saída antes de responder. Em 14 tentativas, 5 concluíram (RFC seções 5 a 8; ADRs 003 e 004; FDD 12.1 a 12.8; FDD contratos 05 a 07; PRD seções 10 a 12) e deram 6 achados: nenhuma contradição nem invenção, e três melhorias de precisão — dizer que as falas citadas numa lacuna são vizinhas do assunto e não decisão sobre ele; não atribuir a Bruno a ausência de uma rota, quando ele apenas não a enumerou; marcar a ordenação do histórico de entregas como proposta. Os outros nove pedaços não puderam ser validados por um modelo com raciocínio de outra família: nesses, a conferência com raciocínio que existe é a do próprio gerador.

## Como navegar a entrega

```text
.
├── README.md                  este documento
├── TRANSCRICAO.md             fonte primária (não alterada)
└── docs/
    ├── PRD.md                 por que e o quê: problema, público, escopo, requisitos
    ├── RFC.md                 a proposta técnica, alternativas descartadas, questões em aberto
    ├── FDD.md                 como construir: dados, fluxos, contratos, erros, integração com o código
    ├── TRACKER.md             cada ID dos documentos com sua origem na transcrição ou no código
    └── adrs/
        ├── README.md          índice dos ADRs
        └── ADR-001 … ADR-008  uma decisão por arquivo
```

Ordem sugerida de leitura:

1. **PRD** — o problema e o escopo. A seção "Fora de escopo" mostra o que a reunião adiou ou recusou.
2. **RFC** — a proposta em uma página e meia, as 8 alternativas descartadas e o que segue em aberto.
3. **ADRs**, na ordem — os seis primeiros são as decisões principais; 007 e 008 são complementares.
4. **FDD** — comece pela seção 12 (como o módulo se encaixa no código existente) e pela seção 15 (o que precisa de confirmação antes de codar).
5. **TRACKER** — use como dicionário: procure o ID de um item e veja de onde ele veio.

Ao ler, três marcações importam: `(novo)` depois de um caminho indica arquivo que ainda não existe; `(proposta, ver FDD-PROP-NN)` indica que a reunião não decidiu aquele detalhe; `(consequência técnica inferida)` indica dedução feita a partir do código.

## Verificação e limitações

O que foi verificado de forma mecânica no repositório final:

- 1331 citações `[hh:mm] Falante` no corpo dos documentos: todas existem, com aquele falante naquele minuto.
- 272 menções a caminhos de código existentes: todos os caminhos existem; os que ainda não existem estão marcados `(novo)`.
- 554 IDs definidos nos documentos. 449 têm origem direta e estão na tabela principal do tracker (87,3% transcrição, 57 código); os outros 105 são propostas sem origem direta e estão no apêndice, identificadas como tais. Pela leitura estrita do critério do enunciado, a cobertura da tabela principal é de 81,0%.
- Nenhum valor numérico fora do que foi decidido na reunião, exceto os que o próprio documento identifica como padrão do Prisma.
- Os controles negativos do validador (timestamp inventado, falante trocado, caminho inexistente, emoji, item descartado disfarçado de requisito) são todos reprovados por ele.

O que foi verificado por revisão, e suas limitações:

- **Revisão independente por outro provedor de LLM.** A auditoria adversarial foi feita pelo mesmo gerador, em contexto novo. - **Validação cruzada por modelo de família diferente da do gerador: feita, mas de forma consultiva e com modelos sem raciocínio.** As rotas de validador configuradas no ambiente estavam indisponíveis na data (uma com erro de provedor, a outra com cota esgotada). Duas verificações fizeram esse papel: o council do Ajuste 10, sobre o bloqueio de segurança, e a auditoria linha a linha do Ajuste 11, com Llama, Qwen e Gemma — sem raciocínio em todos os trechos e, no Qwen com raciocínio, só em 5 dos 14 pedaços tentados. Nenhuma delas tem a profundidade da auditoria feita pelo próprio gerador, e nenhuma substitui o validador configurado.
- **Revisão de segurança: veredito "bloqueado" quanto ao deploy da feature, não quanto à documentação.** 10 achados, todos aceitos e incorporados. O principal: a reunião aceitou que qualquer usuário autenticado gerencia os cadastros "por enquanto" ([09:37] Sofia) e o código tem registro de usuário público, sem vínculo entre usuário e cliente. Qualquer pessoa se cadastra e passa a receber os eventos de pedidos de qualquer cliente. Ler esses pedidos já era possível antes da feature; o que ela acrescenta é o envio automático, o poder de alterar cadastros alheios e a resposta do destino guardada no histórico. Está registrado como bloqueante de deploy, a resolver na revisão de segurança que o time agendou ([09:46] Sofia): FDD-RISCO-05, FDD-RISCO-06, PRD-RISCO-05 e RFC-RISCO-07. Corrigir exige mexer no código ou reabrir uma decisão da reunião, o que a entrega documental não pode fazer por conta própria: a proposta FDD-PROP-44 faz as duas coisas e está marcada como proposta. Um council de modelos de três famílias diferentes da do revisor confirmou essa leitura (Ajuste 10).
- **Nenhum teste da aplicação foi executado.** A entrega é documental e o código não foi alterado; a suíte existente depende de MySQL e não é necessária para estes artefatos.
- **O inventário da transcrição é uma ajuda, não uma prova.** Ele guiou a completude (cada decisão e requisito extraído foi localizado nos documentos), mas a conferência de cada item nos documentos foi feita contra a transcrição, não contra o inventário.

## Processo em números

| Documento | Tamanho (palavras, incluindo tabelas) | IDs definidos |
|---|---|---|
| `docs/PRD.md` | 6455 | 125 |
| `docs/RFC.md` | 2634 | 57 |
| `docs/FDD.md` | 23681 | 350 |
| `docs/TRACKER.md` | 9871 | — |
| `README.md` | 4120 | — |
| `docs/adrs/` (8 ADRs) | 11345 | 22 |
| `TRANSCRICAO.md` (fonte) | 3287 | — |

- 49 chamadas ao gerador — geração, inventário e auditoria —, incluindo sondas e as 3 que falharam; 11 delas de auditoria.
- 3.521.851 tokens contabilizados pelo harness nas chamadas ao gerador.
- 79 achados na auditoria do gerador e 10 na revisão de segurança isolada; todos triados, e os aceitos corrigidos com casamento exato de texto.
- 128 pares minuto/falante na transcrição; 1331 citações nos documentos.
- Validação cruzada independente: 32 respostas de Llama-3.3-70B, Qwen3.6-27B e Gemma-4-12B sobre 9 trechos e 17 sub-trechos; 206 achados, nenhum defeito real, 5 ajustes de citação ou redação. Com raciocínio, o Qwen concluiu 5 de 14 pedaços (6 achados, 3 de precisão aceitos).

A geração usou 49 chamadas ao gerador. O council do Ajuste 10 somou seis chamadas com êxito — duas respostas independentes, duas avaliações por pares e duas sínteses do chairman, sendo a segunda com as restrições da entrega explicitadas — e duas tentativas frustradas de incluir o gerador.

## Enunciado original

O texto completo do desafio está no repositório base: [README do commit inicial](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia/blob/e7f6311b96ee3b21bf8cb7077f53a6e4045143eb/README.md).
