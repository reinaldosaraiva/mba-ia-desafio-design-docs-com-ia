# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| Título | Sistema de Webhooks de Notificação de Pedidos |
| Autor (PRD-META-01) | Marcos (Product Manager) |
| Data | 2026-11-25 |
| Status | Registrado a partir da reunião técnica transcrita em `TRANSCRICAO.md` |
| Documentos relacionados | [RFC](RFC.md) · [FDD](FDD.md) · [Tracker de rastreabilidade](TRACKER.md) · ADR-001 a ADR-008 em `adrs/` |

Marcos é o product manager que levantou o pedido dos clientes e listou os requisitos funcionais (`[09:00] Marcos`, `[09:31] Marcos`).

## 1. Resumo e contexto

Esta feature faz a plataforma avisar automaticamente os clientes B2B quando o status de um pedido deles muda. Com isso, eles deixam de consultar a plataforma de tempos em tempos. O cliente cadastra os endereços que recebem os avisos, escolhe em cada um os status que quer acompanhar e consulta o histórico do que foi enviado. A plataforma assina cada aviso e tenta de novo quando o endereço do cliente está indisponível. O que falhar de forma persistente fica guardado para reprocessamento por um administrador.

A demanda veio de um pedido formal de três clientes, com prazo comercial e risco de perda de cliente. Este PRD trata do porquê e do quê. O como está no RFC, nos ADR-001 a ADR-008 e no FDD. O vínculo de cada item com a sua origem está no Tracker de rastreabilidade.

**Fatos do contexto**

| ID | Fato | Origem |
|---|---|---|
| PRD-CTX-01 | Três clientes B2B fizeram um pedido formal da funcionalidade na semana anterior à reunião: Atlas Comercial, MaxDistribuição e Nova Cargo. | `[09:00] Marcos` |
| PRD-CTX-02 | Hoje esses clientes consultam a lista de pedidos da plataforma de tempos em tempos para ver se algo mudou. Isso deixa a integração lenta e cara para eles. | `[09:00] Marcos` |
| PRD-CTX-03 | A Atlas Comercial sugeriu que pode migrar para o concorrente se a funcionalidade não for entregue até o fim do trimestre. | `[09:00] Marcos` |
| PRD-CTX-04 | A Atlas Comercial quer a funcionalidade para fim de novembro. Marcos ficou de confirmar o prazo com o cliente. | `[09:45] Marcos`, `[09:47] Marcos` |
| PRD-CTX-05 | Para os clientes, qualquer coisa abaixo de 10 segundos já é "tempo real". O importante é não precisarem ficar atualizando manualmente. | `[09:02] Marcos` |
| PRD-CTX-06 | O alcance é só de saída: a plataforma notifica e o cliente apenas recebe. O cliente não envia notificações para a plataforma. | `[09:02] Marcos`, `[09:03] Sofia` |

**Interpretação a confirmar (PRD-CTX-03 e PRD-CTX-04).** A transcrição traz dois marcos da Atlas Comercial: fim do trimestre, associado ao risco de migração (`[09:00] Marcos`), e fim de novembro, data pedida (`[09:45] Marcos`). Leitura adotada: fim de novembro é a data de entrega pedida, e fim do trimestre é o limite a partir do qual o cliente cogita migrar. Outra leitura: fim de novembro seria uma antecipação que substitui o prazo de fim do trimestre. A reunião não relaciona os dois marcos, e Marcos ficou de confirmar o prazo (`[09:47] Marcos`).

## 2. Problema e motivação

Os clientes B2B não ficam sabendo, por iniciativa da plataforma, que o status de um pedido deles mudou. Para descobrir, a integração de cada um precisa consultar a lista de pedidos de tempos em tempos e verificar se algo mudou (PRD-CTX-02). Esse modelo traz dois custos para o cliente. O primeiro é o atraso: a mudança só aparece na consulta seguinte, e o cliente acaba tendo de ficar atualizando manualmente para que o acompanhamento do pedido não fique pendurado (`[09:02] Marcos`). O segundo é operacional: a integração fica lenta e cara para quem a mantém (`[09:00] Marcos`).

Para o negócio, a falta dessa capacidade já tem efeito comercial. Três clientes fizeram pedido formal pela mesma funcionalidade (PRD-CTX-01). A Atlas Comercial associou a entrega a um prazo e sugeriu que pode migrar para o concorrente se não for atendida até o fim do trimestre (PRD-CTX-03, PRD-CTX-04). Enquanto a plataforma não avisar os clientes, eles continuam arcando com o custo das consultas periódicas. A plataforma, por sua vez, continua exposta à perda de um cliente que já declarou essa possibilidade.

## 3. Público-alvo e cenários de uso

### 3.1 Público-alvo

- **PRD-PUB-01.** Clientes B2B que pediram a funcionalidade (Atlas Comercial, MaxDistribuição e Nova Cargo) e que receberão os avisos de mudança de status dos pedidos deles (`[09:00] Marcos`).
- **PRD-PUB-02.** Usuários da plataforma que representam o cliente e operam a API em nome dele. São eles que cadastram e gerenciam os endereços de notificação, autenticados na própria plataforma (`[09:32] Marcos`, `[09:32] Bruno`).
- **PRD-PUB-03.** Administradores da plataforma (perfil ADMIN), os únicos que podem reprocessar entregas que falharam de forma persistente (`[09:36] Sofia`).
- **PRD-PUB-04.** Time de Pedidos, interessado porque o aviso nasce da mudança de status do pedido, que a feature altera (`[09:40] Bruno`).
- **PRD-PUB-05.** Time de Plataforma, interessado no modelo de registro e entrega dos avisos que propôs na reunião (`[09:06] Diego`).
- **PRD-PUB-06.** Segurança, interessada na assinatura dos avisos, na chave e na revisão de segurança antes da produção (`[09:19] Sofia`, `[09:46] Sofia`).

### 3.2 Cenários de uso

- **PRD-CEN-01.** O status de um pedido de um cliente muda na plataforma. Abaixo de 10 segundos depois, o sistema do cliente recebe o aviso, sem ter consultado a lista de pedidos (`[09:00] Marcos`, `[09:02] Marcos`).
- **PRD-CEN-02.** O cliente só quer saber quando o pedido vira SHIPPED e DELIVERED. Ele cadastra o endereço com essa lista e não recebe aviso das demais mudanças (`[09:33] Marcos`).
- **PRD-CEN-03.** O endereço do cliente fica fora do ar por duas horas numa manutenção planejada, como já ocorreu com um cliente da plataforma. A plataforma tenta de novo em intervalos crescentes e entrega o aviso quando o endereço volta (`[09:16] Diego`).
- **PRD-CEN-04.** O cliente quer conferir o que a plataforma mandou para ele. Ele consulta o histórico de entregas do cadastro e vê, para cada uma, o resultado, o conteúdo, a resposta e o tempo de resposta (`[09:34] Marcos`).
- **PRD-CEN-05.** Uma entrega esgota as novas tentativas e fica guardada como falha persistente. Um administrador a reprocessa e ela volta para envio. A plataforma registra quem fez o reprocessamento (`[09:18] Diego`, `[09:36] Sofia`).
- **PRD-CEN-06.** O cliente desconfia que a chave vazou, como já aconteceu com um cliente que a expôs em log da aplicação dele (`[09:22] Diego`). Ele pede uma nova chave, e a antiga continua válida por 24 horas em paralelo. Nesse prazo, ele migra os sistemas dele sem interromper o recebimento (`[09:21] Sofia`).
- **PRD-CEN-07.** O sistema do cliente recebe o mesmo aviso duas vezes. Pelo identificador único do evento, ele reconhece a repetição e descarta a segunda cópia (`[09:24] Diego`, `[09:25] Diego`).

## 4. Objetivos e métricas de sucesso

| ID | Objetivo | Métrica | Meta | Origem |
|---|---|---|---|---|
| PRD-OBJ-01 | Avisar o cliente rápido o bastante para ele considerar "tempo real" | Tempo entre a mudança de status do pedido e a chegada do aviso ao endereço do cliente | Abaixo de 10 segundos | `[09:02] Marcos`, `[09:10] Marcos` |
| PRD-OBJ-02 | Não perder aviso de mudança de status | Mudanças de status em que algum cadastro do cliente quer o novo status e que ficaram sem aviso registrado | Nenhuma ocorrência | `[09:40] Bruno` |
| PRD-OBJ-03 | Entregar o aviso ao cliente que volta depois de ficar indisponível | Avisos entregues a endereços que voltaram a responder depois de ficar fora do ar | Aviso entregue quando o endereço volta em até 12 horas | `[09:15] Diego`, `[09:17] Marcos` |
| PRD-OBJ-04 | Reduzir a consulta periódica que os clientes fazem hoje à lista de pedidos | Quantidade de consultas periódicas dos clientes à lista de pedidos | não definida na reunião | `[09:00] Marcos` |
| PRD-OBJ-05 | Manter a Atlas Comercial, que sugeriu migrar para o concorrente | Permanência da Atlas Comercial como cliente após o fim do trimestre | A Atlas Comercial continua cliente, e a funcionalidade é entregue até fim de novembro, data pedida por ela (PRD-CTX-04) | `[09:00] Marcos`, `[09:45] Marcos` |

**Interpretação a confirmar**

- **PRD-OBJ-01.** A reunião não fixou onde a medição começa e termina, nem se a meta vale quando o endereço do cliente não responde. Leitura adotada: a meta vale para a primeira entrega a um endereço que responde, já que as novas tentativas seguem intervalos de minutos a horas (PRD-FR-18). Outra leitura: medir até a resposta do cliente. Como essa resposta pode levar até o limite de 10 segundos de espera (PRD-RNF-04), a meta poderia ser ultrapassada mesmo sem falha. O FDD adota, para teste e acompanhamento, a medição até o início do envio (FDD-PROP-25); as duas leituras precisam ser reconciliadas na revisão do documento de design.
- **PRD-OBJ-02.** A leitura literal de "não pode ter caso de status mudar e evento não sair" (`[09:40] Bruno`) exigiria aviso para toda mudança de status, o que contradiz o filtro de `[09:34] Bruno`. Leitura adotada: nenhuma perda quando há cadastro interessado no novo status (PRD-FR-10). Ver ADR-001.
- **PRD-OBJ-03.** Diego estimou que 5 tentativas cobrem "uma janela de até 12 ou 24 horas" (`[09:15] Diego`). Depois, detalhou "quase 15 horas entre primeira falha e última tentativa" (`[09:17] Diego`), e Marcos aceitou (`[09:17] Marcos`). A meta de 12 horas depende da contagem adotada em PRD-FR-18. Na outra leitura dessa contagem, o intervalo de 12 horas fica sem uso e a janela não chega a 12 horas.

## 5. Escopo

### 5.1 No escopo

- **PRD-ESC-01.** Cadastro e gestão dos endereços de notificação (criar, editar, remover e listar) pelo próprio cliente, por meio dos usuários que o representam na API (`[09:31] Marcos`, `[09:32] Marcos`, `[09:33] Bruno`).
- **PRD-ESC-02.** Aviso automático ao cliente quando o status de um pedido dele muda (`[09:00] Marcos`).
- **PRD-ESC-03.** Escolha, por cadastro, de quais status o endereço recebe (`[09:33] Marcos`).
- **PRD-ESC-04.** Novas tentativas automáticas quando o endereço do cliente está indisponível (`[09:15] Diego`, `[09:17] Larissa`).
- **PRD-ESC-05.** Guarda das entregas que falharam de forma persistente e reprocessamento manual por administrador (`[09:18] Diego`, `[09:19] Larissa`).
- **PRD-ESC-06.** Assinatura dos avisos e troca da chave de assinatura a pedido do cliente (`[09:21] Sofia`, `[09:22] Sofia`).
- **PRD-ESC-07.** Histórico de entregas consultável pelo cliente (`[09:34] Marcos`).
- **PRD-ESC-08.** Documentação no portal do desenvolvedor sobre como integrar via API, com destaque para a possibilidade de o mesmo aviso chegar mais de uma vez (`[09:26] Marcos`, `[09:40] Marcos`).

### 5.2 Fora de escopo

- **PRD-OOS-01.** E-mail de alerta ao cliente quando o endereço dele está com problema (exemplo citado: após 3 falhas seguidas). Fica adiado para uma fase seguinte, depois de medir o impacto (`[09:37] Marcos`, `[09:37] Larissa`, `[09:38] Marcos`).
- **PRD-OOS-02.** Painel visual para o cliente ver os cadastros dele. Nesta fase há só a API, e o painel é projeto separado do time de frontend (`[09:39] Marcos`, `[09:40] Larissa`).
- **PRD-OOS-03.** Limite de envio por cliente, como no exemplo de 50 pedidos mudando de status em um minuto. Fica como ponto a observar e decidir depois (`[09:38] Diego`, `[09:39] Larissa`).
- **PRD-OOS-04.** Arquivamento dos avisos já entregues, citado como "depois de 30 dias ou assim" (`[09:08] Diego`).
- **PRD-OOS-05.** Mais de um processo de entrega em paralelo, tratado como "problema do futuro, não agora" (`[09:12] Diego`, `[09:13] Diego`).
- **PRD-OOS-06.** Notificações enviadas pelo cliente para a plataforma (`[09:02] Marcos`).
- **PRD-OOS-07.** Itens do pedido dentro do aviso, deixados de fora para não inflar o conteúdo (`[09:43] Diego`).
- **PRD-OOS-08.** Garantia de entrega exatamente uma vez, descartada por exigir coordenação dos dois lados (`[09:25] Diego`).
- **PRD-OOS-09.** Envio do aviso dentro da própria alteração do pedido, descartado para que um cliente lento ou fora do ar não trave nem desfaça a mudança de status (`[09:04] Bruno`, `[09:06] Diego`).
- **PRD-OOS-10.** Garantia de ordem global entre pedidos diferentes, que os clientes nunca pediram (`[09:13] Larissa`, `[09:14] Marcos`).
- **PRD-OOS-11.** Restrição da gestão dos cadastros a perfis específicos, citada como possível endurecimento futuro (`[09:37] Sofia`).

## 6. Requisitos funcionais

| ID | Requisito | Origem |
|---|---|---|
| PRD-FR-01 | O cliente consegue cadastrar um endereço de notificação informando a URL e a lista de status que quer receber. | `[09:31] Marcos` |
| PRD-FR-02 | Na criação do cadastro, a plataforma gera a chave de assinatura e a devolve ao cliente. O cliente não escolhe a chave. | `[09:31] Marcos` |
| PRD-FR-03 | O cadastro informa explicitamente a que cliente pertence, sem deduzi-lo do usuário logado: esse usuário é um operador que representa o cliente, e não o próprio cliente. Vale a forma de `[09:32] Larissa`, que corrige o "implícito" de `[09:31] Marcos`. O modo de informar fica para o FDD. | `[09:32] Bruno`, `[09:32] Larissa` |
| PRD-FR-04 | O cliente consegue editar um cadastro existente. | `[09:33] Bruno` |
| PRD-FR-05 | O cliente consegue remover um cadastro. | `[09:33] Bruno` |
| PRD-FR-06 | O cliente consegue listar os cadastros de um cliente. | `[09:33] Bruno` |
| PRD-FR-07 | A escolha de status é individual por cadastro: endereços diferentes do mesmo cliente podem acompanhar status diferentes. | `[09:33] Bruno`, `[09:33] Marcos` |
| PRD-FR-08 | Cada cadastro tem um estado ativo ou inativo. | `[09:21] Bruno`, `[09:21] Sofia` |
| PRD-FR-09 | Nesta fase, qualquer usuário autenticado na plataforma, de qualquer perfil, consegue gerenciar os cadastros. Só o reprocessamento (PRD-FR-20) exige administrador. | `[09:36] Marcos`, `[09:37] Sofia` |
| PRD-FR-10 | A plataforma gera aviso para um cadastro só quando o novo status do pedido está na lista daquele cadastro. Se nenhum cadastro do cliente quer aquele status, nenhum aviso é gerado. | `[09:33] Marcos`, `[09:34] Bruno` |
| PRD-FR-11 | O aviso identifica o evento e o seu tipo, o pedido (identificador e número), o cliente, o status anterior e o novo, data e hora e dados básicos do pedido, como o valor total. Não traz os itens. O formato fica no FDD. | `[09:43] Diego` |
| PRD-FR-12 | O aviso reflete o estado do pedido no instante em que o status mudou, mesmo que o pedido seja alterado antes do envio. | `[09:52] Larissa` |
| PRD-FR-13 | Cada aviso identifica o cadastro que o originou, para que o cliente com vários cadastros saiba qual deles recebeu o envio. | `[09:44] Sofia` |
| PRD-FR-14 | Cada aviso informa o momento do envio, para que o cliente consiga, se quiser, detectar o reenvio malicioso de um aviso antigo. | `[09:44] Diego` |
| PRD-FR-15 | Cada aviso traz um identificador único do evento, com o qual o cliente reconhece e descarta avisos repetidos. | `[09:25] Diego` |
| PRD-FR-16 | Quando quiser mais informação, como os itens, o cliente consulta os detalhes do pedido pela API da plataforma. | `[09:43] Diego` |
| PRD-FR-17 | O cliente consegue consultar o histórico de entregas de um cadastro. O histórico mostra os "últimos 100 webhooks" enviados, com resultado (sucesso ou falha), conteúdo enviado, resposta recebida e tempo de resposta. | `[09:34] Marcos` |
| PRD-FR-18 | Quando a entrega falha, a plataforma tenta de novo automaticamente, com teto de 5 tentativas e intervalos crescentes de 1m/5m/30m/2h/12h. | `[09:15] Diego`, `[09:17] Larissa` |
| PRD-FR-19 | Esgotadas as tentativas, a plataforma considera a falha permanente e guarda a entrega com o conteúdo, o motivo da falha e o momento, para análise e reprocessamento. | `[09:15] Diego`, `[09:18] Diego` |
| PRD-FR-20 | Um administrador consegue reprocessar uma entrega guardada, que volta a aguardar envio como pendente. O reprocessamento é só manual e exige o perfil de administrador. | `[09:18] Diego`, `[09:36] Larissa` |
| PRD-FR-21 | A plataforma registra quem fez cada reprocessamento, para auditoria. | `[09:36] Sofia` |
| PRD-FR-22 | O cliente consegue confirmar que cada aviso partiu da plataforma e que o conteúdo não foi alterado no caminho, verificando a assinatura com a chave do cadastro. | `[09:19] Sofia`, `[09:20] Sofia` |
| PRD-FR-23 | O cliente consegue pedir uma nova chave de assinatura pela API. A antiga continua válida por 24 horas em paralelo com a nova e depois deixa de valer. | `[09:21] Sofia` |

**Interpretação a confirmar**

- **PRD-FR-08.** Bruno listou o "estado ativo" no cadastro e Sofia confirmou, mas a reunião não disse quem altera esse estado nem qual é o efeito dele (`[09:21] Bruno`, `[09:21] Sofia`). Leitura adotada: o cliente altera o estado pela edição (PRD-FR-04), e um cadastro inativo não recebe avisos. Outra leitura: o estado é controlado só pela plataforma. Detalhamento no FDD.
- **PRD-FR-10 (status comparado).** Leitura adotada: compara-se o novo status, como no exemplo "só quero saber quando vira SHIPPED e DELIVERED" (`[09:33] Marcos`). Outra leitura: "aquele status" (`[09:34] Bruno`) não diz se é o novo status ou o anterior. Ver ADR-008.
- **PRD-FR-10 (criação do pedido).** A reunião fala em aviso quando o status "muda" (`[09:00] Marcos`) e cita como ponto de origem apenas a mudança de status (`[09:40] Bruno`). Leitura adotada: a criação do pedido, que já nasce em PENDING, não gera aviso. Outra leitura: a criação também contaria como mudança de status. Ver ADR-001.
- **PRD-FR-15.** Leitura adotada: o identificador é o mesmo em todas as entregas do evento, inclusive após o reprocessamento (PRD-FR-20), por ser "único por evento" (`[09:25] Diego`). Outra leitura: o identificador nasce quando o evento é registrado para envio (`[09:25] Diego`), e o reprocessamento recoloca o evento para envio (`[09:18] Diego`). Nesse caso, o evento reprocessado ganharia um identificador novo, e o cliente não o reconheceria como repetido. Ver ADR-005.
- **PRD-FR-17.** Marcos fala nos últimos 100 enviados "pra mim", isto é, por cliente, mas associa a consulta a um cadastro específico (`[09:34] Marcos`). Leitura adotada: as últimas 100 entregas de cada cadastro. Outra leitura: as últimas 100 entregas do cliente, somando todos os cadastros.
- **PRD-FR-18.** O teto de "5 tentativas" convive com cinco intervalos. Leitura adotada (ADR-003): envio inicial mais 5 novas tentativas, com "quase 15 horas entre primeira falha e última tentativa" (`[09:17] Diego`). Outra leitura: "total 5 tentativas" (`[09:48] Larissa`), ou seja, 5 envios, com o intervalo de 12 horas sem uso. A confirmação cabe à revisão do FDD (`[09:50] Larissa`).
- **PRD-FR-14.** A reunião não disse como o cliente detecta o reenvio (`[09:44] Diego`). O momento do envio não é assinado, e o instante gravado no aviso é o da mudança de status, não o do envio. Por isso, no desenho, a defesa contra reenvio é o descarte pelo identificador do evento (PRD-FR-15). Ver FDD-GAP-07.
- **PRD-FR-23.** A reunião não disse com qual chave a plataforma assina durante as 24 horas (`[09:21] Sofia`). As opções são assinar com a antiga até o fim do prazo ou com a nova desde a troca, com o cliente aceitando as duas. A escolha fica para o FDD, sujeita à revisão de segurança (PRD-RNF-09). Ver ADR-004.

## 7. Requisitos não funcionais

| ID | Requisito | Origem |
|---|---|---|
| PRD-RNF-01 | O aviso chega ao cliente abaixo de 10 segundos depois da mudança de status. A meta e a medição estão em PRD-OBJ-01. | `[09:02] Marcos`, `[09:09] Diego` |
| PRD-RNF-02 | O endereço de notificação é obrigatoriamente HTTPS. Um cadastro com endereço HTTP é recusado com erro de validação. | `[09:23] Sofia` |
| PRD-RNF-03 | Cada aviso tem no máximo 64KB. Acima disso, a plataforma não envia o aviso e trata o caso como erro, sem truncar o conteúdo. O destino do aviso recusado não foi discutido e fica para o FDD. | `[09:23] Sofia`, `[09:24] Diego`, `[09:24] Larissa` |
| PRD-RNF-04 | A plataforma espera no máximo 10 segundos pela resposta do endereço do cliente. Sem resposta nesse prazo, a entrega conta como falha e segue para nova tentativa (PRD-FR-18). | `[09:42] Diego` |
| PRD-RNF-05 | Cada cadastro tem chave de assinatura própria. Nunca há uma chave única para todos, para que o vazamento de uma não comprometa as demais. | `[09:21] Sofia` |
| PRD-RNF-06 | A assinatura usa HMAC-SHA256 sobre o conteúdo enviado, padrão de mercado para o qual os clientes têm suporte disponível. | `[09:20] Sofia`, `[09:22] Sofia` |
| PRD-RNF-07 | A entrega acontece ao menos uma vez. O cliente pode receber o mesmo aviso mais de uma vez e deve estar preparado para isso, usando o identificador do evento para descartar repetições (PRD-FR-15). | `[09:24] Diego`, `[09:26] Larissa` |
| PRD-RNF-08 | A ordem dos avisos é garantida apenas por pedido, e só enquanto houver um único processo de entrega. Não há garantia de ordem global. | `[09:12] Diego`, `[09:13] Larissa` |
| PRD-RNF-09 | Antes de entrar em produção, a feature passa pela revisão de segurança de Sofia, com pelo menos dois dias úteis reservados e atenção à assinatura e à geração da chave. A revisão está incluída no fim das três sprints estimadas. | `[09:46] Sofia`, `[09:47] Larissa` |

**Interpretação a confirmar**

- **PRD-RNF-08.** A reunião não tratou da ordem no caso em que um aviso de um pedido aguarda nova tentativa e um aviso posterior do mesmo pedido fica pronto para envio (`[09:12] Diego`, `[09:17] Larissa`). A garantia acima vale para o fluxo sem novas tentativas. Para esse caso, o desenho ainda precisa decidir entre segurar os avisos seguintes do pedido até o anterior ser entregue ou aceitar a inversão. Detalhamento no FDD (lacuna FDD-GAP-25).

## 8. Decisões e trade-offs principais

Cada linha resume, em linguagem de produto, uma decisão fechada na reunião. O mecanismo e as alternativas descartadas estão no ADR indicado em "Registro". As colunas de trade-off trazem só o que a equipe disse e aceitou.

| ID | Decisão | O que ganhamos | O que aceitamos em troca | Registro |
|---|---|---|---|---|
| PRD-DEC-01 | Cada aviso é registrado no banco de dados já usado pela plataforma, na mesma operação que muda o status do pedido. O envio acontece depois, fora dessa operação. | Se a mudança de status foi gravada, o aviso também foi; se foi desfeita, o aviso some junto (`[09:06] Diego`). Um cliente lento ou fora do ar não trava a mudança de status de outros pedidos (`[09:04] Bruno`). Nenhuma infraestrutura nova para um time pequeno operar (`[09:07] Diego`). | Se o registro do aviso falhar, a mudança de status é desfeita (`[09:40] Bruno`). | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| PRD-DEC-02 | Um único processo de entrega, separado da API, busca a cada 2 segundos os avisos pendentes mais antigos, em lote pequeno, e os envia na ordem de registro. | Atende o requisito de "abaixo de 10 segundos" (`[09:09] Diego`, `[09:10] Marcos`). A entrega não cai junto quando a API reinicia (`[09:11] Diego`). Os avisos de um mesmo pedido chegam na ordem em que foram registrados (`[09:12] Diego`). | Latência de 2 segundos no pior caso (`[09:10] Larissa`). A ordenação vale só por pedido e enquanto houver um único processo de entrega, sem garantia global (`[09:13] Larissa`); os clientes não pediram ordenação global (`[09:14] Marcos`). Escalar com vários processos em paralelo fica para o futuro (`[09:13] Diego`). | [ADR-002](adrs/ADR-002-worker-separado-com-polling.md) |
| PRD-DEC-03 | Em caso de falha, o aviso é reenviado em intervalos crescentes de 1m/5m/30m/2h/12h, com teto de 5 tentativas. Esgotado o teto, vai para uma fila de falhas permanentes com o conteúdo, o motivo da falha e o registro de data e hora. Só volta a ser enviado por reprocessamento manual de um administrador. | Cobre indisponibilidades como a de duas horas em manutenção planejada, que um cliente já teve (`[09:16] Diego`). O aviso não fica pendurado para sempre se o cliente sumiu (`[09:15] Diego`). A fila separada deixa limpa a leitura dos avisos ativos e serve de evidência para investigação e reprocessamento (`[09:18] Diego`). | O reenvio automático termina depois de quase 15 horas entre a primeira falha e a última tentativa (`[09:17] Diego`). Marcos considerou isso aceitável para um cliente que fique fora por 15 horas (`[09:17] Marcos`). Depois disso, o reprocessamento é só manual (`[09:18] Diego`). | [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md) |
| PRD-DEC-04 | Cada aviso leva uma assinatura HMAC-SHA256 sobre o corpo do envio. A chave é única por endereço cadastrado, gerada pela plataforma e devolvida na criação do cadastro. O cliente pode pedir uma nova chave; a antiga continua válida por 24 horas em paralelo e depois deixa de valer. | O cliente confirma que o aviso veio da plataforma e não foi adulterado (`[09:19] Sofia`). O vazamento de uma chave não compromete os demais endereços (`[09:21] Sofia`). Na troca, o cliente tem tempo para migrar os sistemas dele (`[09:21] Sofia`). | O cliente precisa verificar a assinatura do lado dele (`[09:20] Sofia`). Durante 24 horas após a troca, a chave antiga continua válida (`[09:21] Sofia`). | [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) |
| PRD-DEC-05 | A entrega é feita pelo menos uma vez (at-least-once). Cada aviso leva um identificador único do evento, que o cliente usa para descartar repetições. | Evita a coordenação dos dois lados que a entrega exatamente uma vez exigiria, solução que "fica muito mais complexo" (`[09:25] Diego`). Segue o padrão de mercado citado na reunião (`[09:25] Diego`). | O cliente pode receber o mesmo aviso mais de uma vez e assume a tarefa de descartar repetições (`[09:24] Diego`, `[09:25] Sofia`). Em contrapartida, o portal traz documentação destacada sobre isso (`[09:26] Marcos`). | [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) |
| PRD-DEC-06 | A funcionalidade reaproveita ao máximo os padrões do projeto: módulo igual aos demais, tratamento e códigos de erro, registro de logs, validação de entrada e controle de acesso existentes. | O tratamento centralizado de erros atende a funcionalidade "sem precisar mudar nada", e o registro de logs não ganha componente novo (`[09:29] Bruno`). | Por enquanto, qualquer usuário autenticado pode gerenciar os cadastros de webhook; endurecer esse acesso fica "mais pra frente" (`[09:37] Sofia`). O processo de entrega usa conexão própria com o mesmo banco, por ser outro processo (`[09:30] Bruno`). | [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) |
| PRD-DEC-07 | O conteúdo do aviso é montado no momento em que o status muda e enviado como foi montado, sem nova montagem na hora do envio. | O aviso reflete o estado do pedido de quando o status mudou, mesmo que o pedido mude depois, e evita o "caso esquisito" citado (`[09:52] Larissa`). | A reunião não registrou custo para esta decisão. | [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md) |
| PRD-DEC-08 | Cada cadastro guarda a lista de status que quer receber, e o filtro é aplicado quando o aviso é registrado. Se nenhum cadastro do cliente do pedido quer aquele status, nenhum aviso é registrado. | Não se acumulam registros de avisos que nenhum cadastro quer ("Economiza linha na tabela", `[09:34] Bruno`). | A reunião não registrou custo para esta decisão. | [ADR-008](adrs/ADR-008-filtro-de-eventos-na-insercao.md) |

**Interpretações a confirmar**

1. **PRD-DEC-01 (rollback e filtro).** A frase "não pode ter caso de status mudar e evento não sair" (`[09:40] Bruno`) foi lida como válida para falha no registro do aviso. Ela não se aplica à ausência de aviso causada pelo filtro de PRD-DEC-08 (`[09:34] Bruno`). A leitura literal exigiria um aviso em toda mudança de status.
2. **PRD-DEC-01 (criação do pedido).** A reunião não tratou se a criação do pedido, que já registra o status PENDING no histórico, gera aviso. A leitura adotada é que não gera, porque só a mudança de status foi citada (`[09:40] Bruno`).
3. **PRD-DEC-02 (latência).** A fala de `[09:10] Larissa` combina "latência mínima" com "pior caso". Outra leitura parte da coleta a cada 2 segundos (`[09:09] Diego`): a espera até a coleta varia de 0 a 2 segundos, e 2 segundos é o limite superior, não um piso.
4. **PRD-DEC-03 (teto de tentativas).** O teto de 5 tentativas foi lido como cinco novas tentativas após o envio inicial, porque há cinco intervalos em `[09:17] Diego`. A outra leitura, favorecida por "total 5 tentativas" (`[09:48] Larissa`), conta o envio inicial e deixa sem uso o intervalo de 12 horas. A confirmação fica na revisão de PRD-DEP-08.
5. **PRD-DEC-04 (troca de chave).** A reunião não disse qual chave assina os avisos nas 24 horas após a troca. Há duas leituras: a antiga até o fim do prazo, ou a nova desde a troca, com o cliente aceitando as duas. A decisão fica no FDD, para a revisão de segurança.
6. **PRD-DEC-05 (identificador no reprocessamento).** A leitura adotada é que o identificador se mantém em todos os reenvios do mesmo evento, inclusive após reprocessamento manual (`[09:25] Diego`). Na outra leitura, como o reprocessamento recoloca o aviso como pendente (`[09:18] Diego`), o aviso receberia um identificador novo.
7. **PRD-DEC-08 (status comparado).** A lista é comparada com o status de destino, pelo exemplo "quando vira SHIPPED e DELIVERED" (`[09:33] Marcos`). A outra leitura compara com o status de origem, porque "aquele status" (`[09:34] Bruno`) não especifica qual dos dois.

## 9. Dependências

Itens dos quais a entrega ou o resultado da funcionalidade dependem e que estão fora do desenvolvimento propriamente dito.

- **PRD-DEP-01.** Documentação para os clientes no portal do desenvolvedor, assumida por Marcos. Ela explica como integrar via API (`[09:40] Marcos`) e informa com destaque que o mesmo aviso pode chegar mais de uma vez (`[09:26] Marcos`).
- **PRD-DEP-02.** Revisão de segurança de Sofia antes da entrada em produção, com pelo menos dois dias úteis reservados e foco na assinatura e na geração da chave (`[09:46] Sofia`). Sofia pediu para ser agendada antes da subida (`[09:49] Sofia`).
- **PRD-DEP-03.** Cada cliente precisa disponibilizar um endereço HTTPS; um cadastro com endereço http é recusado (`[09:23] Sofia`).
- **PRD-DEP-04.** Cada cliente precisa estar preparado para receber o mesmo aviso mais de uma vez e descartar as repetições pelo identificador do evento (`[09:24] Diego`, `[09:25] Diego`).
- **PRD-DEP-05.** Cada cliente precisa verificar a assinatura do lado dele (`[09:20] Sofia`).
- **PRD-DEP-06.** Marcos confirma o prazo com a Atlas Comercial, o cliente solicitante (`[09:47] Marcos`), e atualiza os clientes na tarde da reunião (`[09:49] Marcos`).
- **PRD-DEP-07.** Plano de entrega em 3 sprints, com a revisão de segurança incluída no fim (`[09:47] Larissa`). A estimativa de Larissa ("eu chuto") soma (`[09:46] Larissa`):
  - registro de avisos e fila de falhas: uma sprint;
  - processo de entrega e novas tentativas: uma sprint;
  - gestão dos cadastros e histórico de entregas: meia sprint;
  - integração com a mudança de status e testes de ponta a ponta: mais meia sprint;
  - assinatura e validações: "mais um pouco".
- **PRD-DEP-08.** Sessão de revisão do documento de design com Bruno e Diego antes do início da implementação (`[09:50] Larissa`).

## 10. Riscos e mitigação

Probabilidade e impacto (baixa, média, alta) são avaliação deste documento, porque a reunião não os classificou; essas classificações não recebem ID.

| ID | Risco | Probabilidade | Impacto | Mitigação | Origem |
|---|---|---|---|---|---|
| PRD-RISCO-01 | O cliente estar indisponível quando o aviso é enviado, como na manutenção planejada de duas horas que um cliente já teve. | Média | Média | Novas tentativas em 1m/5m/30m/2h/12h, fila de falhas e reprocessamento manual (PRD-DEC-03). | `[09:14] Larissa`, `[09:16] Diego` |
| PRD-RISCO-02 | Vazamento da chave de assinatura, como já aconteceu com um cliente que expôs a chave em log da aplicação dele. | Média | Alta | Chave única por endereço, que limita o alcance do vazamento, e troca de chave com 24 horas em paralelo (PRD-DEC-04). Revisão de segurança antes da produção (PRD-DEP-02). | `[09:22] Diego` |
| PRD-RISCO-03 | O cliente não descartar avisos repetidos e processar o mesmo evento mais de uma vez. | Média | Média | Identificador único por evento (PRD-DEC-05) e documentação destacada no portal (PRD-DEP-01). | `[09:24] Diego`, `[09:25] Sofia` |
| PRD-RISCO-04 | Uma falha no registro do aviso impedir a própria mudança de status do pedido. | Baixa | Alta | Comportamento aceito pela equipe para que nenhum status mude sem aviso registrado (PRD-DEC-01). Cobertura nos testes de ponta a ponta (PRD-TESTE-01). | `[09:40] Bruno` |
| PRD-RISCO-05 | (inferência deste documento a partir da decisão de acesso e do código) Qualquer pessoa pode passar a receber os avisos de pedidos de qualquer cliente. O acesso amplo foi aceito "por enquanto" (`[09:37] Sofia`) sobre a premissa de que os usuários representam os clientes (`[09:32] Marcos`); no código, porém, qualquer pessoa se cadastra na plataforma, e não há vínculo entre usuário e cliente. | Alta | Alta | Bloqueante de produção: reexaminar o acesso amplo na revisão de segurança antes da produção (PRD-DEP-02). A reunião não definiu outra mitigação; o detalhamento e a proposta de correção estão no FDD (FDD-RISCO-05, FDD-GAP-46). | `[09:32] Marcos`, `[09:37] Sofia` |
| PRD-RISCO-06 | Perder a Atlas Comercial, que sugeriu migrar para o concorrente se a funcionalidade não for entregue no prazo. | Média | Alta | Plano de 3 sprints (PRD-DEP-07) e confirmação do prazo com o cliente (PRD-DEP-06). | `[09:00] Marcos` |
| PRD-RISCO-07 | A entrega atrasar quando o cliente responde devagar. | Média | Média | Tempo limite de 10 segundos por envio; ao estourar, o envio é tratado como falha e vai para nova tentativa (`[09:42] Diego`). O envio fica fora da mudança de status e não trava outros pedidos (PRD-DEC-01). | `[09:04] Bruno`, `[09:42] Diego` |
| PRD-RISCO-08 | (inferência deste documento) Não cumprir o prazo pedido pela Atlas Comercial (fim de novembro) com o plano de 3 sprints. | Média | Alta | Plano de 3 sprints com a revisão de segurança incluída (PRD-DEP-07), dois dias úteis reservados para a revisão (PRD-DEP-02) e confirmação do prazo (PRD-DEP-06). | `[09:45] Marcos`, `[09:47] Larissa` |
| PRD-RISCO-09 | (inferência deste documento) Um usuário autenticado cadastra um endereço de notificação apontando para a infraestrutura da própria plataforma, e a plataforma passa a enviar requisições para lá. | Média | Alto | Não discutido na reunião. Validar o destino das entregas antes do envio (lacuna FDD-GAP-45 do FDD) e levar o ponto à revisão de segurança (PRD-DEP-02). | `[09:31] Marcos`, `[09:06] Diego` |

**Prazo da Atlas Comercial.** Em `[09:00] Marcos`, "fim do trimestre" é o limite a partir do qual a Atlas cogita migrar; em `[09:45] Marcos`, "fim de novembro" é a data que ela pede. As duas datas coexistem, como registrado na seção 1. A confirmação fica com PRD-DEP-06.

## 11. Critérios de aceitação

Cada critério é verificável pelo comportamento observado pelo cliente ou pelo administrador.

| ID | Critério | Requisito relacionado | Origem |
|---|---|---|---|
| PRD-CA-01 | Ao cadastrar um webhook com endereço HTTPS e lista de status, o usuário recebe na resposta a chave de assinatura gerada pela plataforma. | PRD-FR-01, PRD-FR-02 | `[09:31] Marcos` |
| PRD-CA-02 | Um cadastro com endereço http é recusado com erro de validação. | PRD-RNF-02 | `[09:23] Sofia` |
| PRD-CA-03 | Um usuário autenticado consegue editar, remover e listar os webhooks de um cliente. | PRD-FR-04, PRD-FR-05, PRD-FR-06 | `[09:33] Bruno` |
| PRD-CA-04 | Quando o status de um pedido muda para um status assinado por um webhook do cliente, o endereço cadastrado, se disponível, recebe o aviso em menos de 10 segundos. | PRD-FR-10, PRD-RNF-01 | `[09:02] Marcos` |
| PRD-CA-05 | Quando o status muda para um status que nenhum webhook do cliente assina, nenhum aviso é registrado nem enviado. | PRD-FR-10 | `[09:34] Bruno` |
| PRD-CA-06 | Todo aviso chega assinado, e o cliente valida a assinatura com a chave do seu cadastro. Dois cadastros nunca compartilham a mesma chave. | PRD-FR-22, PRD-RNF-05, PRD-RNF-06 | `[09:22] Sofia` |
| PRD-CA-07 | Após a troca de chave pedida pelo cliente, a chave antiga permanece válida por 24 horas e deixa de valer em seguida. | PRD-FR-23 | `[09:21] Sofia` |
| PRD-CA-08 | Todo aviso traz um identificador único do evento, e os reenvios do mesmo evento trazem o mesmo identificador. | PRD-FR-15, PRD-RNF-07 | `[09:25] Diego` |
| PRD-CA-09 | Com o endereço do cliente indisponível, o aviso é reenviado nos intervalos 1m/5m/30m/2h/12h. Esgotado o teto de 5 tentativas, vai para a fila de falhas com o conteúdo, o motivo da falha e o registro de data e hora. | PRD-FR-18, PRD-FR-19 | `[09:15] Diego`, `[09:17] Larissa`, `[09:18] Diego` |
| PRD-CA-10 | Um envio sem resposta em 10 segundos é tratado como falha e vai para nova tentativa. | PRD-RNF-04 | `[09:42] Diego` |
| PRD-CA-11 | O reprocessamento de um aviso da fila de falhas é aceito só para usuário com papel de administrador e recusado para os demais. Quando aceito, devolve o aviso ao envio como pendente e registra quem o fez. | PRD-FR-20, PRD-FR-21 | `[09:18] Diego`, `[09:36] Sofia` |
| PRD-CA-12 | O histórico de entregas de um webhook mostra os últimos 100 webhooks enviados, com sucesso ou falha, conteúdo, resposta e tempo de resposta. | PRD-FR-17 | `[09:34] Marcos` |
| PRD-CA-13 | Um aviso com conteúdo acima de 64KB não é enviado e resulta em erro. | PRD-RNF-03 | `[09:24] Larissa` |
| PRD-CA-14 | Quando o registro do aviso falha, o status do pedido permanece o anterior. | PRD-OBJ-02, PRD-DEC-01 | `[09:40] Bruno` |

PRD-CA-07, PRD-CA-08 e PRD-CA-09 dependem das interpretações a confirmar da seção 8 (PRD-DEC-04, PRD-DEC-05 e PRD-DEC-03), cuja confirmação está ligada à revisão de PRD-DEP-08.

## 12. Estratégia de testes e validação

A estratégia combina o que a reunião definiu, a base de testes que o projeto já tem e validações derivadas dos requisitos. O detalhamento dos casos fica no FDD.

### Definido na reunião

- **PRD-TESTE-01.** Testes de ponta a ponta da integração com a mudança de status do pedido, previstos no plano de entrega (`[09:46] Larissa`). Cobrem PRD-FR-10 e PRD-OBJ-02.
- **PRD-TESTE-02.** Revisão de segurança da assinatura e da geração da chave, feita por Sofia antes da entrada em produção, com pelo menos dois dias úteis (`[09:46] Sofia`, PRD-DEP-02). Cobre PRD-RNF-05, PRD-RNF-06 e PRD-FR-23.

### Base existente no projeto

- **PRD-TESTE-03.** Os testes da funcionalidade partem da suíte de testes automatizados já existente no projeto. Essa suíte já verifica a criação de pedidos e as mudanças de status, válidas e inválidas.

### Validações derivadas dos requisitos

- **PRD-TESTE-04.** Mudança de status com e sem cliente interessado e filtro por status. Um webhook que assina só alguns status recebe apenas esses, em menos de 10 segundos. Uma mudança sem cadastro interessado não gera aviso. Cobre PRD-FR-07, PRD-FR-10 e PRD-RNF-01.
- **PRD-TESTE-05.** Repetição e descarte: os reenvios do mesmo evento chegam com o mesmo identificador, o que permite ao cliente descartá-los. Cobre PRD-FR-15 e PRD-RNF-07.
- **PRD-TESTE-06.** Indisponibilidade do cliente e simulação com um cliente que responde devagar. Simula um endereço fora do ar e outro que não responde em 10 segundos. Verifica que ambos contam como falha, seguem os intervalos 1m/5m/30m/2h/12h, respeitam o teto de 5 tentativas e terminam na fila de falhas. Cobre PRD-FR-18, PRD-FR-19 e PRD-RNF-04.
- **PRD-TESTE-07.** Reprocessamento restrito ao administrador: aceito para o papel de administrador e recusado para os demais, com registro de quem o fez. Cobre PRD-FR-20 e PRD-FR-21.
- **PRD-TESTE-08.** Troca de chave: a chave antiga vale por 24 horas após a troca e deixa de valer depois. Cobre PRD-FR-23.
- **PRD-TESTE-09.** Histórico de entregas: os últimos 100 webhooks enviados aparecem com sucesso ou falha, conteúdo, resposta e tempo de resposta. Cobre PRD-FR-17.
- **PRD-TESTE-10.** Recusa de endereço não HTTPS no cadastro e de aviso acima de 64KB. Cobre PRD-RNF-02 e PRD-RNF-03.
