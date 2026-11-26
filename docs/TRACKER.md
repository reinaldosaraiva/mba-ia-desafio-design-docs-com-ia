# Tracker de rastreabilidade

Este tracker é a referência cruzada do pacote: para cada item registrado nos documentos (requisito, decisão, restrição, alternativa, risco, contrato, erro, ponto de integração), aponta **de onde ele veio** — uma fala da transcrição ou um arquivo do código. Se um item não tem origem aqui, ele não é requisito nem decisão.

| Resumo | Valor |
|---|---|
| Itens rastreados (tabela principal) | 449 |
| Com `Fonte = TRANSCRICAO` | 392 (87.3%) |
| Com `Fonte = CODIGO` | 57 |
| Propostas sem origem direta (apêndice, fora da tabela) | 105 |
| Cobertura | 81.0% dos IDs definidos nos documentos têm linha na tabela principal |

## Como ler

- **ID** é o identificador do item no documento onde ele é definido: `PRD-TIPO-NN`, `RFC-TIPO-NN`, `FDD-TIPO-NN` ou `ADR-NNN` (e `ADR-NNN-ALT-k` para as alternativas registradas em cada ADR).
- **Documento** é o arquivo onde o item aparece.
- **Tipo** decorre do sufixo do ID (legenda abaixo).
- **Fonte** é `TRANSCRICAO` ou `CODIGO`. Para `TRANSCRICAO`, **Localização** traz o minuto e o falante (`[hh:mm] Nome`) da fala que originou o item; para decisão fechada por várias vozes, a fala que a fecha. Para `CODIGO`, **Localização** é o caminho do arquivo que sustenta a afirmação.
- As propostas que a documentação precisou fazer porque a reunião não decidiu um detalhe de implementação estão listadas no apêndice, fora da tabela: não são requisito nem decisão e, por isso, não têm Fonte.

## Legenda de tipos

| Sufixo do ID | Tipo |
|---|---|
| `META-` | Metadado |
| `CTX-` | Contexto |
| `PUB-` | Público-alvo |
| `CEN-` | Cenário de uso |
| `OBJ-` | Objetivo e métrica |
| `ESC-` | No escopo |
| `OOS-` | Fora de escopo |
| `FR-` | Requisito Funcional |
| `RNF-` | Requisito Não Funcional |
| `DEC-` | Decisão |
| `DEP-` | Dependência |
| `RISCO-` | Risco |
| `CA-` | Critério de aceitação |
| `TESTE-` | Teste e validação |
| `ALT-` | Alternativa considerada |
| `QA-` | Questão em aberto |
| `GAP-` | Lacuna não decidida na reunião |
| `IMP-` | Impacto |
| `PROX-` | Próximo passo |
| `COMP-` | Componente |
| `DADOS-` | Modelo de dados |
| `FLUXO-` | Passo de fluxo |
| `CONTRATO-` | Contrato |
| `HEADER-` | Header da entrega |
| `ERRO-` | Erro |
| `RES-` | Resiliência |
| `OBS-` | Observabilidade |
| `INT-` | Integração com o código existente |
| `PROP-` | Proposta a confirmar |
| `(ADR-NNN, sem sufixo)` | Decisão |

## Itens rastreáveis

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-CA-01 | docs/PRD.md | Critério de aceitação | Cadastro HTTPS devolve a chave gerada pela plataforma | TRANSCRICAO | [09:31] Marcos |
| PRD-CA-02 | docs/PRD.md | Critério de aceitação | Cadastro com endereço http recusado com erro de validação | TRANSCRICAO | [09:23] Sofia |
| PRD-CA-03 | docs/PRD.md | Critério de aceitação | Usuário autenticado edita, remove e lista webhooks de um cliente | TRANSCRICAO | [09:33] Bruno |
| PRD-CA-04 | docs/PRD.md | Critério de aceitação | Aviso de status assinado chega em menos de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-CA-05 | docs/PRD.md | Critério de aceitação | Status não assinado não gera aviso | TRANSCRICAO | [09:34] Bruno |
| PRD-CA-06 | docs/PRD.md | Critério de aceitação | Aviso assinado e chave distinta por cadastro | TRANSCRICAO | [09:22] Sofia |
| PRD-CA-07 | docs/PRD.md | Critério de aceitação | Chave antiga válida por 24 horas após a troca | TRANSCRICAO | [09:21] Sofia |
| PRD-CA-08 | docs/PRD.md | Critério de aceitação | Identificador único por evento, mantido nos reenvios | TRANSCRICAO | [09:25] Diego |
| PRD-CA-09 | docs/PRD.md | Critério de aceitação | Reenvio em 1m/5m/30m/2h/12h e fila de falhas após o teto de 5 tentativas | TRANSCRICAO | [09:17] Larissa |
| PRD-CA-10 | docs/PRD.md | Critério de aceitação | Sem resposta em 10 segundos conta como falha e vai para nova tentativa | TRANSCRICAO | [09:42] Diego |
| PRD-CA-11 | docs/PRD.md | Critério de aceitação | Reprocessamento só para administrador, com registro de quem o fez | TRANSCRICAO | [09:36] Sofia |
| PRD-CA-12 | docs/PRD.md | Critério de aceitação | Histórico com os últimos 100 webhooks enviados | TRANSCRICAO | [09:34] Marcos |
| PRD-CA-13 | docs/PRD.md | Critério de aceitação | Aviso acima de 64KB não é enviado e gera erro | TRANSCRICAO | [09:24] Larissa |
| PRD-CA-14 | docs/PRD.md | Critério de aceitação | Falha no registro do aviso mantém o status anterior | TRANSCRICAO | [09:40] Bruno |
| PRD-CEN-01 | docs/PRD.md | Cenário de uso | Cliente avisado quando o status do pedido muda | TRANSCRICAO | [09:00] Marcos |
| PRD-CEN-02 | docs/PRD.md | Cenário de uso | Cliente acompanha só SHIPPED e DELIVERED | TRANSCRICAO | [09:33] Marcos |
| PRD-CEN-03 | docs/PRD.md | Cenário de uso | Endpoint fora do ar por duas horas recebe o aviso ao voltar | TRANSCRICAO | [09:16] Diego |
| PRD-CEN-04 | docs/PRD.md | Cenário de uso | Cliente consulta o histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-CEN-05 | docs/PRD.md | Cenário de uso | Administrador reprocessa entrega com falha persistente | TRANSCRICAO | [09:18] Diego |
| PRD-CEN-06 | docs/PRD.md | Cenário de uso | Cliente troca a chave sem interromper o recebimento | TRANSCRICAO | [09:21] Sofia |
| PRD-CEN-07 | docs/PRD.md | Cenário de uso | Cliente descarta aviso repetido pelo identificador do evento | TRANSCRICAO | [09:25] Diego |
| PRD-CTX-01 | docs/PRD.md | Contexto | Pedido formal de Atlas Comercial, MaxDistribuição e Nova Cargo | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Contexto | Clientes consultam pedidos de tempos em tempos; integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Contexto | Atlas pode migrar para o concorrente se não houver entrega até o fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-04 | docs/PRD.md | Contexto | Atlas quer a funcionalidade para fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-CTX-05 | docs/PRD.md | Contexto | Abaixo de 10 segundos já é "tempo real" para os clientes | TRANSCRICAO | [09:02] Marcos |
| PRD-CTX-06 | docs/PRD.md | Contexto | Alcance só de saída: plataforma notifica, cliente só recebe | TRANSCRICAO | [09:02] Marcos |
| PRD-DEC-01 | docs/PRD.md | Decisão e trade-off | Aviso registrado no banco atual junto com a mudança de status e enviado fora dela | TRANSCRICAO | [09:08] Larissa |
| PRD-DEC-02 | docs/PRD.md | Decisão e trade-off | Processo de entrega único e separado, coleta a cada 2 segundos, ordenação só por pedido | TRANSCRICAO | [09:10] Larissa |
| PRD-DEC-03 | docs/PRD.md | Decisão e trade-off | Novas tentativas 1m/5m/30m/2h/12h, teto de 5 tentativas, fila de falhas e reprocessamento manual | TRANSCRICAO | [09:17] Larissa |
| PRD-DEC-04 | docs/PRD.md | Decisão e trade-off | Assinatura HMAC-SHA256, chave por endereço gerada pela plataforma, troca com 24 horas em paralelo | TRANSCRICAO | [09:22] Sofia |
| PRD-DEC-05 | docs/PRD.md | Decisão e trade-off | Entrega at-least-once com identificador único para o cliente descartar repetições | TRANSCRICAO | [09:26] Larissa |
| PRD-DEC-06 | docs/PRD.md | Decisão e trade-off | Reuso máximo dos padrões existentes do projeto | TRANSCRICAO | [09:30] Larissa |
| PRD-DEC-07 | docs/PRD.md | Decisão e trade-off | Conteúdo do aviso montado quando o status muda e enviado como foi montado | TRANSCRICAO | [09:52] Larissa |
| PRD-DEC-08 | docs/PRD.md | Decisão e trade-off | Lista de status por cadastro com filtro aplicado no registro do aviso | TRANSCRICAO | [09:34] Bruno |
| PRD-DEP-01 | docs/PRD.md | Dependência | Documentação destacada no portal do desenvolvedor, assumida por Marcos | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-02 | docs/PRD.md | Dependência | Revisão de segurança antes da produção, com pelo menos dois dias úteis | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-03 | docs/PRD.md | Dependência | Cliente precisa disponibilizar endereço HTTPS | TRANSCRICAO | [09:23] Sofia |
| PRD-DEP-04 | docs/PRD.md | Dependência | Cliente precisa tratar avisos repetidos | TRANSCRICAO | [09:24] Diego |
| PRD-DEP-05 | docs/PRD.md | Dependência | Cliente precisa verificar a assinatura do lado dele | TRANSCRICAO | [09:20] Sofia |
| PRD-DEP-06 | docs/PRD.md | Dependência | Confirmação do prazo com a Atlas Comercial por Marcos | TRANSCRICAO | [09:47] Marcos |
| PRD-DEP-07 | docs/PRD.md | Dependência | Plano de 3 sprints com a revisão de segurança incluída no fim | TRANSCRICAO | [09:47] Larissa |
| PRD-DEP-08 | docs/PRD.md | Dependência | Revisão do documento de design com Bruno e Diego antes da implementação | TRANSCRICAO | [09:50] Larissa |
| PRD-ESC-01 | docs/PRD.md | No escopo | Cadastro e gestão dos endereços pelo cliente | TRANSCRICAO | [09:31] Marcos |
| PRD-ESC-02 | docs/PRD.md | No escopo | Aviso de mudança de status | TRANSCRICAO | [09:00] Marcos |
| PRD-ESC-03 | docs/PRD.md | No escopo | Escolha de status por cadastro | TRANSCRICAO | [09:33] Marcos |
| PRD-ESC-04 | docs/PRD.md | No escopo | Novas tentativas com cliente indisponível | TRANSCRICAO | [09:17] Larissa |
| PRD-ESC-05 | docs/PRD.md | No escopo | Guarda de falhas persistentes e reprocessamento manual | TRANSCRICAO | [09:19] Larissa |
| PRD-ESC-06 | docs/PRD.md | No escopo | Assinatura dos avisos e troca da chave | TRANSCRICAO | [09:22] Sofia |
| PRD-ESC-07 | docs/PRD.md | No escopo | Histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-ESC-08 | docs/PRD.md | No escopo | Documentação no portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar endereço com URL e lista de status | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Chave gerada pela plataforma e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Cliente do cadastro informado explicitamente, não deduzido do usuário logado | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Editar cadastro | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Remover cadastro | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Listar cadastros de um cliente | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Escolha de status individual por cadastro | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Cadastro com estado ativo ou inativo | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Qualquer usuário autenticado gerencia os cadastros nesta fase | TRANSCRICAO | [09:37] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Aviso só quando o novo status está na lista do cadastro | TRANSCRICAO | [09:34] Bruno |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Conteúdo do aviso enxuto, sem itens do pedido | TRANSCRICAO | [09:43] Diego |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Aviso reflete o estado do pedido no instante da mudança | TRANSCRICAO | [09:52] Larissa |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Aviso identifica o cadastro que o originou | TRANSCRICAO | [09:44] Sofia |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | Aviso informa o momento do envio para detectar reenvio malicioso | TRANSCRICAO | [09:44] Diego |
| PRD-FR-15 | docs/PRD.md | Requisito Funcional | Identificador único do evento para descartar repetições | TRANSCRICAO | [09:25] Diego |
| PRD-FR-16 | docs/PRD.md | Requisito Funcional | Cliente consulta detalhes do pedido pela API | TRANSCRICAO | [09:43] Diego |
| PRD-FR-17 | docs/PRD.md | Requisito Funcional | Histórico com os últimos 100 envios, resultado, conteúdo, resposta e tempo | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-18 | docs/PRD.md | Requisito Funcional | Novas tentativas: 5 tentativas, 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-19 | docs/PRD.md | Requisito Funcional | Guarda da entrega com falha permanente, motivo e momento | TRANSCRICAO | [09:18] Diego |
| PRD-FR-20 | docs/PRD.md | Requisito Funcional | Reprocessamento manual só por administrador | TRANSCRICAO | [09:36] Larissa |
| PRD-FR-21 | docs/PRD.md | Requisito Funcional | Registro de quem fez o reprocessamento, para auditoria | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-22 | docs/PRD.md | Requisito Funcional | Cliente confirma origem e integridade do aviso | TRANSCRICAO | [09:19] Sofia |
| PRD-FR-23 | docs/PRD.md | Requisito Funcional | Nova chave pela API; antiga válida por 24 horas em paralelo | TRANSCRICAO | [09:21] Sofia |
| PRD-META-01 | docs/PRD.md | Metadado | Autor do PRD: Marcos, product manager que levantou o pedido dos clientes | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-01 | docs/PRD.md | Objetivo e métrica | Aviso abaixo de 10 segundos após a mudança de status | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo e métrica | Nenhuma mudança de status sem aviso quando há cadastro interessado | TRANSCRICAO | [09:40] Bruno |
| PRD-OBJ-03 | docs/PRD.md | Objetivo e métrica | Entregar o aviso quando o cliente volta em até 12 horas | TRANSCRICAO | [09:15] Diego |
| PRD-OBJ-04 | docs/PRD.md | Objetivo e métrica | Reduzir consultas periódicas; meta não definida na reunião | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-05 | docs/PRD.md | Objetivo e métrica | Manter a Atlas Comercial, que ameaçou migrar | TRANSCRICAO | [09:00] Marcos |
| PRD-OOS-01 | docs/PRD.md | Fora de escopo | E-mail de alerta adiado para a próxima fase | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-02 | docs/PRD.md | Fora de escopo | Painel visual para o cliente | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-03 | docs/PRD.md | Fora de escopo | Limite de envio por cliente, a observar | TRANSCRICAO | [09:39] Larissa |
| PRD-OOS-04 | docs/PRD.md | Fora de escopo | Arquivamento dos avisos entregues ("30 dias ou assim") | TRANSCRICAO | [09:08] Diego |
| PRD-OOS-05 | docs/PRD.md | Fora de escopo | Mais de um processo de entrega em paralelo | TRANSCRICAO | [09:13] Diego |
| PRD-OOS-06 | docs/PRD.md | Fora de escopo | Notificações do cliente para a plataforma | TRANSCRICAO | [09:02] Marcos |
| PRD-OOS-07 | docs/PRD.md | Fora de escopo | Itens do pedido no aviso | TRANSCRICAO | [09:43] Diego |
| PRD-OOS-08 | docs/PRD.md | Fora de escopo | Garantia de entrega exatamente uma vez | TRANSCRICAO | [09:25] Diego |
| PRD-OOS-09 | docs/PRD.md | Fora de escopo | Aviso enviado dentro da alteração do pedido | TRANSCRICAO | [09:06] Diego |
| PRD-OOS-10 | docs/PRD.md | Fora de escopo | Garantia de ordem global | TRANSCRICAO | [09:13] Larissa |
| PRD-OOS-11 | docs/PRD.md | Fora de escopo | Endurecimento futuro das permissões dos cadastros | TRANSCRICAO | [09:37] Sofia |
| PRD-PUB-01 | docs/PRD.md | Público-alvo | Clientes B2B que pediram e receberão os avisos | TRANSCRICAO | [09:00] Marcos |
| PRD-PUB-02 | docs/PRD.md | Público-alvo | Usuários que representam o cliente na API da plataforma | TRANSCRICAO | [09:32] Marcos |
| PRD-PUB-03 | docs/PRD.md | Público-alvo | Administradores, únicos que reprocessam entregas | TRANSCRICAO | [09:36] Sofia |
| PRD-PUB-04 | docs/PRD.md | Público-alvo | Time de Pedidos, dono da mudança de status alterada pela feature | TRANSCRICAO | [09:40] Bruno |
| PRD-PUB-05 | docs/PRD.md | Público-alvo | Time de Plataforma, que propôs o modelo de registro e entrega | TRANSCRICAO | [09:06] Diego |
| PRD-PUB-06 | docs/PRD.md | Público-alvo | Segurança, interessada na assinatura e na revisão antes da produção | TRANSCRICAO | [09:46] Sofia |
| PRD-RISCO-01 | docs/PRD.md | Risco | Cliente indisponível quando o aviso é enviado | TRANSCRICAO | [09:14] Larissa |
| PRD-RISCO-02 | docs/PRD.md | Risco | Vazamento da chave de assinatura, já ocorrido com um cliente | TRANSCRICAO | [09:22] Diego |
| PRD-RISCO-03 | docs/PRD.md | Risco | Cliente não descartar avisos repetidos | TRANSCRICAO | [09:24] Diego |
| PRD-RISCO-04 | docs/PRD.md | Risco | Falha no registro do aviso impedir a mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-RISCO-05 | docs/PRD.md | Risco | Qualquer usuário autenticado cadastrar endereço para qualquer cliente | TRANSCRICAO | [09:36] Marcos |
| PRD-RISCO-06 | docs/PRD.md | Risco | Perder a Atlas Comercial, que ameaçou migrar | TRANSCRICAO | [09:00] Marcos |
| PRD-RISCO-07 | docs/PRD.md | Risco | Entrega atrasar quando o cliente responde devagar | TRANSCRICAO | [09:04] Bruno |
| PRD-RISCO-08 | docs/PRD.md | Risco | Não cumprir o prazo de fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-RISCO-09 | docs/PRD.md | Risco | Endereço de notificação apontando para a infraestrutura interna; plataforma passa a enviar requisições para lá (inferência do documento) | TRANSCRICAO | [09:31] Marcos |
| PRD-RNF-01 | docs/PRD.md | Requisito Não Funcional | Latência abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-RNF-02 | docs/PRD.md | Requisito Não Funcional | Endereço obrigatoriamente HTTPS; HTTP recusado com erro de validação | TRANSCRICAO | [09:23] Sofia |
| PRD-RNF-03 | docs/PRD.md | Requisito Não Funcional | Aviso com no máximo 64KB; acima disso, erro | TRANSCRICAO | [09:24] Larissa |
| PRD-RNF-04 | docs/PRD.md | Requisito Não Funcional | Sem resposta em 10 segundos, a entrega é falha e vai para nova tentativa | TRANSCRICAO | [09:42] Diego |
| PRD-RNF-05 | docs/PRD.md | Requisito Não Funcional | Chave própria por cadastro, nunca única para todos | TRANSCRICAO | [09:21] Sofia |
| PRD-RNF-06 | docs/PRD.md | Requisito Não Funcional | Assinatura HMAC-SHA256 sobre o conteúdo enviado | TRANSCRICAO | [09:22] Sofia |
| PRD-RNF-07 | docs/PRD.md | Requisito Não Funcional | Entrega ao menos uma vez, com identificador para deduplicação | TRANSCRICAO | [09:26] Larissa |
| PRD-RNF-08 | docs/PRD.md | Requisito Não Funcional | Ordem garantida só por pedido e com processo de entrega único | TRANSCRICAO | [09:13] Larissa |
| PRD-RNF-09 | docs/PRD.md | Requisito Não Funcional | Revisão de segurança antes da produção, com pelo menos dois dias úteis | TRANSCRICAO | [09:46] Sofia |
| PRD-TESTE-01 | docs/PRD.md | Teste e validação | Testes de ponta a ponta da integração com a mudança de status | TRANSCRICAO | [09:46] Larissa |
| PRD-TESTE-02 | docs/PRD.md | Teste e validação | Revisão de segurança da assinatura e da geração da chave | TRANSCRICAO | [09:46] Sofia |
| PRD-TESTE-03 | docs/PRD.md | Teste e validação | Suíte de testes automatizados existente como base | CODIGO | tests/orders.test.ts |
| PRD-TESTE-04 | docs/PRD.md | Teste e validação | Mudança de status com e sem interessado e filtro por status | TRANSCRICAO | [09:33] Marcos |
| PRD-TESTE-05 | docs/PRD.md | Teste e validação | Repetição e descarte pelo identificador do evento | TRANSCRICAO | [09:25] Diego |
| PRD-TESTE-06 | docs/PRD.md | Teste e validação | Indisponibilidade e cliente lento: novas tentativas e fila de falhas | TRANSCRICAO | [09:17] Larissa |
| PRD-TESTE-07 | docs/PRD.md | Teste e validação | Reprocessamento restrito ao administrador | TRANSCRICAO | [09:36] Sofia |
| PRD-TESTE-08 | docs/PRD.md | Teste e validação | Troca de chave com 24 horas de validade da antiga | TRANSCRICAO | [09:21] Sofia |
| PRD-TESTE-09 | docs/PRD.md | Teste e validação | Histórico de entregas com os últimos 100 webhooks | TRANSCRICAO | [09:34] Marcos |
| PRD-TESTE-10 | docs/PRD.md | Teste e validação | Recusa de endereço não HTTPS e de aviso acima de 64KB | TRANSCRICAO | [09:23] Sofia |
| RFC-ALT-01 | docs/RFC.md | Alternativa considerada | Disparo síncrono no service de pedidos, descartado | TRANSCRICAO | [09:03] Larissa |
| RFC-ALT-02 | docs/RFC.md | Alternativa considerada | Redis Streams ou Redis Cluster como fila, descartado | TRANSCRICAO | [09:07] Larissa |
| RFC-ALT-03 | docs/RFC.md | Alternativa considerada | Trigger do banco para o worker reagir, descartada | TRANSCRICAO | [09:09] Bruno |
| RFC-ALT-04 | docs/RFC.md | Alternativa considerada | Novas tentativas sem limite, descartadas | TRANSCRICAO | [09:15] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa considerada | Limite de 3 tentativas, descartado | TRANSCRICAO | [09:16] Bruno |
| RFC-ALT-06 | docs/RFC.md | Alternativa considerada | Entrega exatamente uma vez, descartada | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-07 | docs/RFC.md | Alternativa considerada | Secret global da plataforma, descartada | TRANSCRICAO | [09:21] Sofia |
| RFC-ALT-08 | docs/RFC.md | Alternativa considerada | Id auto incremental na outbox, preterido pelo UUID, que é o padrão do projeto | TRANSCRICAO | [09:51] Diego |
| RFC-CTX-01 | docs/RFC.md | Contexto | Pedido formal de Atlas Comercial, MaxDistribuição e Nova Cargo; a consulta periódica deixa a integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| RFC-CTX-02 | docs/RFC.md | Contexto | Abaixo de 10 segundos já é "tempo real" para os clientes | TRANSCRICAO | [09:02] Marcos |
| RFC-CTX-03 | docs/RFC.md | Contexto | Alcance só de notificação de saída; os clientes recebem e não enviam | TRANSCRICAO | [09:02] Marcos |
| RFC-CTX-04 | docs/RFC.md | Contexto | Não há módulo de notificação, evento, fila ou webhook; os módulos montados são auth, users, customers, products e orders | CODIGO | src/app.ts |
| RFC-CTX-05 | docs/RFC.md | Contexto | A transação de mudança de status já é pesada (pedido, histórico e estoque) | TRANSCRICAO | [09:04] Bruno |
| RFC-CTX-06 | docs/RFC.md | Contexto | Risco de a Atlas migrar para o concorrente sem entrega até o fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| RFC-CTX-07 | docs/RFC.md | Contexto | A Atlas quer a entrega para o fim de novembro | TRANSCRICAO | [09:45] Marcos |
| RFC-DEC-01 | docs/RFC.md | Decisão | Outbox no MySQL existente, na mesma transação da mudança de status | TRANSCRICAO | [09:08] Larissa |
| RFC-DEC-02 | docs/RFC.md | Decisão | Worker em processo separado, polling de 2 segundos, worker único | TRANSCRICAO | [09:10] Larissa |
| RFC-DEC-03 | docs/RFC.md | Decisão | 5 tentativas, intervalos 1m/5m/30m/2h/12h, fila de mortos em tabela separada, replay manual com role ADMIN | TRANSCRICAO | [09:17] Larissa |
| RFC-DEC-04 | docs/RFC.md | Decisão | HMAC-SHA256 sobre o corpo do request, secret por endpoint, rotação com 24 horas de validade da secret antiga | TRANSCRICAO | [09:22] Sofia |
| RFC-DEC-05 | docs/RFC.md | Decisão | Entrega at-least-once com identificador de evento para deduplicação no cliente | TRANSCRICAO | [09:26] Larissa |
| RFC-DEC-06 | docs/RFC.md | Decisão | Reuso máximo dos padrões existentes; webhook como módulo igual aos demais | TRANSCRICAO | [09:30] Larissa |
| RFC-DEC-07 | docs/RFC.md | Decisão | Snapshot do payload no momento da inserção na outbox | TRANSCRICAO | [09:52] Larissa |
| RFC-DEC-08 | docs/RFC.md | Decisão | Filtro por lista de status aplicado na inserção; sem webhook interessado, nada é gravado | TRANSCRICAO | [09:34] Bruno |
| RFC-GAP-01 | docs/RFC.md | Lacuna não discutida na reunião | Evento pendente com cadastro removido ou desativado | TRANSCRICAO | [09:33] Bruno |
| RFC-GAP-02 | docs/RFC.md | Lacuna não discutida na reunião | Qual resposta do cliente conta como sucesso | TRANSCRICAO | [09:42] Diego |
| RFC-GAP-03 | docs/RFC.md | Lacuna não discutida na reunião | Eventos em processamento quando o worker cai | TRANSCRICAO | [09:08] Diego |
| RFC-GAP-04 | docs/RFC.md | Lacuna não discutida na reunião | Se a criação de pedido também gera evento | TRANSCRICAO | [09:40] Bruno |
| RFC-GAP-05 | docs/RFC.md | Lacuna não discutida na reunião | Formato da assinatura e armazenamento da secret | TRANSCRICAO | [09:21] Sofia |
| RFC-GAP-06 | docs/RFC.md | Lacuna não discutida na reunião | Código de erro para URL sem https | TRANSCRICAO | [09:23] Sofia |
| RFC-GAP-07 | docs/RFC.md | Lacuna não discutida na reunião | Validação do destino das entregas: URL informada pelo cliente e requisição feita pelo worker | TRANSCRICAO | [09:31] Marcos |
| RFC-IMP-01 | docs/RFC.md | Impacto | Tabelas novas no MySQL existente, sem infraestrutura nova | TRANSCRICAO | [09:06] Diego |
| RFC-IMP-02 | docs/RFC.md | Impacto | Worker como processo novo, separado da API | TRANSCRICAO | [09:11] Diego |
| RFC-IMP-03 | docs/RFC.md | Impacto | Service de pedidos registra o evento na transação, com rollback se falhar | TRANSCRICAO | [09:40] Bruno |
| RFC-IMP-04 | docs/RFC.md | Impacto | Módulo novo de webhooks no padrão dos existentes | TRANSCRICAO | [09:27] Bruno |
| RFC-IMP-05 | docs/RFC.md | Impacto | Revisão de segurança antes do deploy, com pelo menos dois dias úteis | TRANSCRICAO | [09:46] Sofia |
| RFC-IMP-06 | docs/RFC.md | Impacto | Documentação destacada no portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| RFC-IMP-07 | docs/RFC.md | Impacto | Esforço de três sprints com a revisão de Sofia incluída | TRANSCRICAO | [09:47] Larissa |
| RFC-META-01 | docs/RFC.md | Metadado | Autora do RFC: Larissa (Tech Lead), que conduziu a reunião e disse que abriria o documento de design da feature | TRANSCRICAO | [09:50] Larissa |
| RFC-PROX-01 | docs/RFC.md | Próximo passo | Sessão de revisão do documento de design com Bruno e Diego | TRANSCRICAO | [09:50] Larissa |
| RFC-PROX-02 | docs/RFC.md | Próximo passo | Agendar a revisão de segurança de Sofia antes do deploy | TRANSCRICAO | [09:49] Sofia |
| RFC-PROX-03 | docs/RFC.md | Próximo passo | Confirmação do prazo com a Atlas | TRANSCRICAO | [09:47] Marcos |
| RFC-QA-01 | docs/RFC.md | Questão em aberto | Rate limiting de saída por cliente: observar e decidir depois | TRANSCRICAO | [09:38] Diego |
| RFC-QA-02 | docs/RFC.md | Questão em aberto | Escala para múltiplos workers em paralelo, adiada | TRANSCRICAO | [09:12] Diego |
| RFC-QA-03 | docs/RFC.md | Questão em aberto | Arquivamento de linhas entregues, fora do escopo | TRANSCRICAO | [09:08] Diego |
| RFC-QA-04 | docs/RFC.md | Questão em aberto | E-mail ao cliente com webhook com problema, próxima fase | TRANSCRICAO | [09:37] Marcos |
| RFC-QA-05 | docs/RFC.md | Questão em aberto | Endurecimento da autorização do cadastro de webhooks | TRANSCRICAO | [09:37] Sofia |
| RFC-QA-06 | docs/RFC.md | Questão em aberto | Leitura de "5 tentativas" frente aos cinco intervalos | TRANSCRICAO | [09:17] Diego |
| RFC-QA-07 | docs/RFC.md | Questão em aberto | Qual secret assina durante as 24 horas da rotação | TRANSCRICAO | [09:21] Sofia |
| RFC-QA-08 | docs/RFC.md | Questão em aberto | Filtro comparado ao status de destino ou de origem | TRANSCRICAO | [09:34] Bruno |
| RFC-QA-09 | docs/RFC.md | Questão em aberto | Identificador do evento mantido ou novo no replay | TRANSCRICAO | [09:25] Diego |
| RFC-RISCO-01 | docs/RFC.md | Risco | Cliente indisponível no momento da entrega | TRANSCRICAO | [09:16] Diego |
| RFC-RISCO-02 | docs/RFC.md | Risco | Vazamento de secret | TRANSCRICAO | [09:22] Diego |
| RFC-RISCO-03 | docs/RFC.md | Risco | Cliente que não deduplica eventos repetidos | TRANSCRICAO | [09:24] Diego |
| RFC-RISCO-04 | docs/RFC.md | Risco | Falha no registro do evento impede a mudança de status | TRANSCRICAO | [09:40] Bruno |
| RFC-RISCO-05 | docs/RFC.md | Risco | Perda da Atlas por migração ao concorrente | TRANSCRICAO | [09:00] Marcos |
| RFC-RISCO-06 | docs/RFC.md | Risco | Estimativa de três sprints frente ao fim de novembro pedido | TRANSCRICAO | [09:45] Marcos |
| RFC-RISCO-07 | docs/RFC.md | Risco | Registro de usuário público e sem vínculo com cliente, combinado com o acesso amplo ao cadastro | CODIGO | src/modules/auth/auth.routes.ts |
| FDD-CA-01 | docs/FDD.md | Critério de aceite técnico | Status, histórico e linha da outbox gravados juntos; falha na outbox desfaz a mudança | TRANSCRICAO | [09:40] Bruno |
| FDD-CA-02 | docs/FDD.md | Critério de aceite técnico | Sem cadastro que assine o status, nenhuma linha na outbox | TRANSCRICAO | [09:34] Bruno |
| FDD-CA-03 | docs/FDD.md | Critério de aceite técnico | Filtro compara o status de destino da transição | TRANSCRICAO | [09:33] Marcos |
| FDD-CA-04 | docs/FDD.md | Critério de aceite técnico | Worker coleta a entrega em até 2 segundos após o commit | TRANSCRICAO | [09:09] Diego |
| FDD-CA-05 | docs/FDD.md | Critério de aceite técnico | Cinco headers e campos snake_case do corpo na entrega | TRANSCRICAO | [09:44] Diego |
| FDD-CA-06 | docs/FDD.md | Critério de aceite técnico | HMAC-SHA256 do corpo verificável pelo cliente com a secret | TRANSCRICAO | [09:20] Sofia |
| FDD-CA-07 | docs/FDD.md | Critério de aceite técnico | Sem resposta em 10 segundos é falha, com nova tentativa | TRANSCRICAO | [09:42] Diego |
| FDD-CA-08 | docs/FDD.md | Critério de aceite técnico | Intervalos 1m/5m/30m/2h/12h e teto de tentativas | TRANSCRICAO | [09:17] Larissa |
| FDD-CA-09 | docs/FDD.md | Critério de aceite técnico | DLQ recebe payload, motivo da falha e timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-CA-10 | docs/FDD.md | Critério de aceite técnico | Replay recusado a quem não é ADMIN e registrado em log com o usuário | TRANSCRICAO | [09:36] Sofia |
| FDD-CA-11 | docs/FDD.md | Critério de aceite técnico | Secret anterior vale por 24 horas após a rotação | TRANSCRICAO | [09:21] Sofia |
| FDD-CA-12 | docs/FDD.md | Critério de aceite técnico | Histórico com as últimas 100 entregas e os quatro dados | TRANSCRICAO | [09:34] Marcos |
| FDD-CA-13 | docs/FDD.md | Critério de aceite técnico | URL sem https recusada com erro de validação | TRANSCRICAO | [09:23] Sofia |
| FDD-CA-14 | docs/FDD.md | Critério de aceite técnico | Payload acima de 64KB não é enviado | TRANSCRICAO | [09:24] Larissa |
| FDD-CA-15 | docs/FDD.md | Critério de aceite técnico | Testes existentes de pedidos continuam passando sem alteração | CODIGO | tests/orders.test.ts |
| FDD-CA-16 | docs/FDD.md | Critério de aceite técnico | Corpo enviado igual ao snapshot mesmo após remoção do pedido | TRANSCRICAO | [09:52] Larissa |
| FDD-COMP-01 | docs/FDD.md | Componente | Módulo de webhooks da API em `src/modules/webhooks` | TRANSCRICAO | [09:27] Bruno |
| FDD-COMP-02 | docs/FDD.md | Componente | Ponto de publicação dentro de `OrderService.changeStatus` | TRANSCRICAO | [09:40] Bruno |
| FDD-COMP-03 | docs/FDD.md | Componente | Entry-point `src/worker.ts` (novo) e script `npm run worker` | TRANSCRICAO | [09:11] Larissa |
| FDD-COMP-04 | docs/FDD.md | Componente | Processador de entregas dentro do módulo | TRANSCRICAO | [09:28] Bruno |
| FDD-COMP-05 | docs/FDD.md | Componente | Tabela `webhook_outbox` com status e índices | TRANSCRICAO | [09:06] Diego |
| FDD-COMP-06 | docs/FDD.md | Componente | Tabela de cadastros com url, secret, customer_id e estado ativo | TRANSCRICAO | [09:21] Bruno |
| FDD-COMP-07 | docs/FDD.md | Componente | Tabela de histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-COMP-08 | docs/FDD.md | Componente | Tabela `webhook_dead_letter` com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | `POST /api/v1/webhooks`: cadastro com `url`, `customerId` e `statuses`; `201` com a secret gerada pela plataforma | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | `GET /api/v1/webhooks`: listagem paginada dos cadastros de um cliente, sem a secret | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | `PATCH /api/v1/webhooks/:id`: edição de `url`, `statuses` e `active`, sem devolver a secret | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | `DELETE /api/v1/webhooks/:id`: remoção do cadastro com `204` | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | `GET /api/v1/webhooks/:id/deliveries`: últimas 100 entregas com resultado, payload, resposta e tempo de resposta | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | Rotação de secret (rota proposta `POST /api/v1/webhooks/:id/rotate-secret`); a anterior vale por 24 horas | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | `POST /api/v1/admin/webhooks/dead-letter/:id/replay`: recoloca o evento na outbox como pendente; exige ADMIN e registra quem fez | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Entrega do evento: request do worker para a URL `https` cadastrada, com corpo JSON do evento | TRANSCRICAO | [09:43] Diego |
| FDD-CTX-01 | docs/FDD.md | Contexto técnico | Três clientes B2B pedem notificação em tempo real; hoje consultam GET /orders periodicamente | TRANSCRICAO | [09:00] Marcos |
| FDD-CTX-02 | docs/FDD.md | Contexto técnico | Abaixo de 10 segundos é "tempo real" para os clientes | TRANSCRICAO | [09:02] Marcos |
| FDD-CTX-03 | docs/FDD.md | Contexto técnico | Alcance apenas de notificação de saída | TRANSCRICAO | [09:02] Marcos |
| FDD-CTX-04 | docs/FDD.md | Contexto técnico | Não há notificação, evento, fila ou webhook; só auth, users, customers, products e orders são montados | CODIGO | src/routes/index.ts |
| FDD-CTX-05 | docs/FDD.md | Contexto técnico | Transação de mudança de status já é pesada; HTTP nela travaria outros pedidos | TRANSCRICAO | [09:04] Bruno |
| FDD-DADOS-01 | docs/FDD.md | Elemento de modelo de dados | Chave primária UUID `@db.Char(36)` e ids de referência em `Char(36)` | CODIGO | prisma/schema.prisma |
| FDD-DADOS-02 | docs/FDD.md | Elemento de modelo de dados | `createdAt`/`updatedAt` em registro alterável; só `createdAt` em registro imutável | CODIGO | prisma/schema.prisma |
| FDD-DADOS-03 | docs/FDD.md | Elemento de modelo de dados | Model PascalCase com `@@map` snake_case; campos camelCase | CODIGO | prisma/schema.prisma |
| FDD-DADOS-04 | docs/FDD.md | Elemento de modelo de dados | Índice simples por coluna com `@@index` | CODIGO | prisma/schema.prisma |
| FDD-DADOS-05 | docs/FDD.md | Elemento de modelo de dados | Enums PascalCase com valores em maiúsculas | CODIGO | prisma/schema.prisma |
| FDD-DADOS-06 | docs/FDD.md | Elemento de modelo de dados | `webhooks.id` UUID, usado no path e em `X-Webhook-Id` | TRANSCRICAO | [09:51] Larissa |
| FDD-DADOS-07 | docs/FDD.md | Elemento de modelo de dados | `webhooks.customerId`, cliente dono do cadastro | TRANSCRICAO | [09:21] Sofia |
| FDD-DADOS-08 | docs/FDD.md | Elemento de modelo de dados | `webhooks.url`, destino das entregas | TRANSCRICAO | [09:31] Marcos |
| FDD-DADOS-09 | docs/FDD.md | Elemento de modelo de dados | `webhooks.secret`, secret única por endpoint | TRANSCRICAO | [09:21] Sofia |
| FDD-DADOS-10 | docs/FDD.md | Elemento de modelo de dados | `webhooks.previousSecret`, secret antiga válida por 24 horas | TRANSCRICAO | [09:21] Sofia |
| FDD-DADOS-11 | docs/FDD.md | Elemento de modelo de dados | `webhooks.previousSecretExpiresAt`, fim das 24 horas | TRANSCRICAO | [09:21] Sofia |
| FDD-DADOS-12 | docs/FDD.md | Elemento de modelo de dados | `webhooks.statuses`, lista de status assinados | TRANSCRICAO | [09:33] Marcos |
| FDD-DADOS-13 | docs/FDD.md | Elemento de modelo de dados | `webhooks.active`, estado ativo do cadastro | TRANSCRICAO | [09:21] Sofia |
| FDD-DADOS-14 | docs/FDD.md | Elemento de modelo de dados | `webhooks.createdAt`/`updatedAt` | CODIGO | prisma/schema.prisma |
| FDD-DADOS-17 | docs/FDD.md | Elemento de modelo de dados | `webhook_outbox.id` UUID | TRANSCRICAO | [09:51] Larissa |
| FDD-DADOS-18 | docs/FDD.md | Elemento de modelo de dados | `webhook_outbox.eventId`, UUID gerado na entrada na outbox | TRANSCRICAO | [09:25] Diego |
| FDD-DADOS-22 | docs/FDD.md | Elemento de modelo de dados | `webhook_outbox.status`: pendente, processando, falhou, entregue | TRANSCRICAO | [09:08] Diego |
| FDD-DADOS-23 | docs/FDD.md | Elemento de modelo de dados | `webhook_outbox.payload` renderizado na inserção | TRANSCRICAO | [09:52] Larissa |
| FDD-DADOS-26 | docs/FDD.md | Elemento de modelo de dados | `webhook_outbox.createdAt`, ordem de processamento | TRANSCRICAO | [09:12] Diego |
| FDD-DADOS-27 | docs/FDD.md | Elemento de modelo de dados | `webhook_outbox.updatedAt`, última mudança de estado | CODIGO | prisma/schema.prisma |
| FDD-DADOS-28 | docs/FDD.md | Elemento de modelo de dados | Índices em `status` e `createdAt` da outbox | TRANSCRICAO | [09:08] Diego |
| FDD-DADOS-30 | docs/FDD.md | Elemento de modelo de dados | `webhook_deliveries.id` UUID | TRANSCRICAO | [09:51] Larissa |
| FDD-DADOS-31 | docs/FDD.md | Elemento de modelo de dados | `webhook_deliveries.webhookId`, filtro do histórico por webhook | TRANSCRICAO | [09:34] Marcos |
| FDD-DADOS-33 | docs/FDD.md | Elemento de modelo de dados | `webhook_deliveries.success`, sucesso/falha | TRANSCRICAO | [09:34] Marcos |
| FDD-DADOS-34 | docs/FDD.md | Elemento de modelo de dados | `webhook_deliveries.payload` | TRANSCRICAO | [09:34] Marcos |
| FDD-DADOS-35 | docs/FDD.md | Elemento de modelo de dados | `webhook_deliveries.responseStatus` | TRANSCRICAO | [09:34] Marcos |
| FDD-DADOS-36 | docs/FDD.md | Elemento de modelo de dados | `webhook_deliveries.responseBody` | TRANSCRICAO | [09:34] Marcos |
| FDD-DADOS-38 | docs/FDD.md | Elemento de modelo de dados | `webhook_deliveries.responseTimeMs`, tempo de resposta | TRANSCRICAO | [09:34] Marcos |
| FDD-DADOS-39 | docs/FDD.md | Elemento de modelo de dados | `webhook_deliveries.createdAt`, ordena os últimos envios | TRANSCRICAO | [09:34] Marcos |
| FDD-DADOS-41 | docs/FDD.md | Elemento de modelo de dados | `webhook_dead_letter.id` UUID, `:id` do replay | TRANSCRICAO | [09:51] Larissa |
| FDD-DADOS-47 | docs/FDD.md | Elemento de modelo de dados | `webhook_dead_letter.payload` | TRANSCRICAO | [09:18] Diego |
| FDD-DADOS-48 | docs/FDD.md | Elemento de modelo de dados | `webhook_dead_letter.reason`, motivo da falha | TRANSCRICAO | [09:18] Diego |
| FDD-DADOS-50 | docs/FDD.md | Elemento de modelo de dados | `webhook_dead_letter.createdAt`, timestamp da ida para a DLQ | TRANSCRICAO | [09:18] Diego |
| FDD-DADOS-56 | docs/FDD.md | Elemento de modelo de dados | Migration gerada por `npm run db:migrate` em `prisma/migrations/` | CODIGO | package.json |
| FDD-DADOS-57 | docs/FDD.md | Elemento de modelo de dados | Conteúdo e formato da migration das quatro tabelas | CODIGO | prisma/migrations/20260519182739_init/migration.sql |
| FDD-DADOS-58 | docs/FDD.md | Elemento de modelo de dados | Relação inversa `webhooks Webhook[]` em `Customer`, sem coluna nova | CODIGO | prisma/schema.prisma |
| FDD-DADOS-59 | docs/FDD.md | Elemento de modelo de dados | Sem migração de dados; a emissão nasce em `changeStatus` | TRANSCRICAO | [09:40] Bruno |
| FDD-DADOS-60 | docs/FDD.md | Elemento de modelo de dados | Limpeza de `tests/setup.ts` inclui as tabelas novas | CODIGO | tests/setup.ts |
| FDD-DEP-01 | docs/FDD.md | Dependência ou compatibilidade | Node.js `>=20` exigido em `engines` | CODIGO | package.json |
| FDD-DEP-02 | docs/FDD.md | Dependência ou compatibilidade | `express` 4.21.1 reusada no router do módulo | CODIGO | package.json |
| FDD-DEP-03 | docs/FDD.md | Dependência ou compatibilidade | `@prisma/client` e `prisma` 5.22.0 reusados; migration pelo script `db:migrate` | CODIGO | package.json |
| FDD-DEP-04 | docs/FDD.md | Dependência ou compatibilidade | `zod` 3.23.8 reusada nos schemas do módulo | CODIGO | package.json |
| FDD-DEP-05 | docs/FDD.md | Dependência ou compatibilidade | `pino` 9.5.0 reusada pela API e pelo worker | CODIGO | package.json |
| FDD-DEP-06 | docs/FDD.md | Dependência ou compatibilidade | `uuid` 11.0.3 já instalada; disponível para gerar o id do evento antes do insert | CODIGO | package.json |
| FDD-DEP-09 | docs/FDD.md | Dependência ou compatibilidade | MySQL 8.0 existente recebe as quatro tabelas novas | CODIGO | docker-compose.yml |
| FDD-DEP-11 | docs/FDD.md | Dependência ou compatibilidade | Sem variável nova; worker precisa do mesmo `.env` da API, inclusive `JWT_SECRET` | CODIGO | src/config/env.ts |
| FDD-DEP-12 | docs/FDD.md | Dependência ou compatibilidade | Rotas existentes sem mudança de contrato; módulo entra como novo sub-router | CODIGO | src/routes/index.ts |
| FDD-DEP-13 | docs/FDD.md | Dependência ou compatibilidade | `tests/orders.test.ts` continua passando; tabelas novas na limpeza de `tests/setup.ts` | CODIGO | tests/orders.test.ts |
| FDD-DEP-14 | docs/FDD.md | Dependência ou compatibilidade | `include` de `tsconfig.build.json` já cobre `src/worker.ts` (novo) | CODIGO | tsconfig.build.json |
| FDD-DEP-15 | docs/FDD.md | Dependência ou compatibilidade | Importação de tipos obrigatória pelo ESLint | CODIGO | .eslintrc.json |
| FDD-DEP-16 | docs/FDD.md | Dependência ou compatibilidade | Prettier com aspas simples e vírgula final | CODIGO | .prettierrc |
| FDD-DEP-17 | docs/FDD.md | Dependência ou compatibilidade | Projeto ESM com imports relativos terminados em `.js` | CODIGO | package.json |
| FDD-ERRO-01 | docs/FDD.md | Erro | `WEBHOOK_NOT_FOUND` (404) para cadastro inexistente nas rotas por id | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-03 | docs/FDD.md | Erro | Erro quando o payload ultrapassa 64KB; nome `WEBHOOK_PAYLOAD_TOO_LARGE` é proposta | TRANSCRICAO | [09:24] Larissa |
| FDD-ERRO-04 | docs/FDD.md | Erro | Timeout de 10 segundos tratado como falha de entrega; nome do código é proposta | TRANSCRICAO | [09:42] Diego |
| FDD-ESC-01 | docs/FDD.md | No escopo | Registro do evento na `webhook_outbox` dentro de `changeStatus` | TRANSCRICAO | [09:08] Larissa |
| FDD-ESC-02 | docs/FDD.md | No escopo | Worker em processo separado com polling de 2 segundos | TRANSCRICAO | [09:11] Diego |
| FDD-ESC-03 | docs/FDD.md | No escopo | Novas tentativas 1m/5m/30m/2h/12h e fila de mortos | TRANSCRICAO | [09:17] Larissa |
| FDD-ESC-04 | docs/FDD.md | No escopo | Assinatura HMAC-SHA256 com secret por endpoint, headers e limite de 64KB | TRANSCRICAO | [09:22] Sofia |
| FDD-ESC-05 | docs/FDD.md | No escopo | API de gerenciamento com URL https e filtro por status | TRANSCRICAO | [09:33] Marcos |
| FDD-ESC-06 | docs/FDD.md | No escopo | Histórico de entregas por cadastro | TRANSCRICAO | [09:34] Marcos |
| FDD-ESC-07 | docs/FDD.md | No escopo | Rotação de secret com 24 horas de validade paralela | TRANSCRICAO | [09:21] Sofia |
| FDD-ESC-08 | docs/FDD.md | No escopo | Replay manual da DLQ restrito a ADMIN, com registro de autoria | TRANSCRICAO | [09:36] Larissa |
| FDD-FLUXO-01 | docs/FDD.md | Passo de fluxo | `changeStatus` chama `publishWebhookEvent` com o `tx`, sem injetar repository | TRANSCRICAO | [09:41] Diego |
| FDD-FLUXO-02 | docs/FDD.md | Passo de fluxo | Consulta dos cadastros ativos do cliente que assinam o status de destino | TRANSCRICAO | [09:34] Bruno |
| FDD-FLUXO-03 | docs/FDD.md | Passo de fluxo | Sem cadastro selecionado, nada é inserido na outbox | TRANSCRICAO | [09:34] Bruno |
| FDD-FLUXO-04 | docs/FDD.md | Passo de fluxo | Geração do UUID do evento na entrada na outbox | TRANSCRICAO | [09:25] Diego |
| FDD-FLUXO-05 | docs/FDD.md | Passo de fluxo | Montagem do payload com os campos ditados, sem itens | TRANSCRICAO | [09:43] Diego |
| FDD-FLUXO-06 | docs/FDD.md | Passo de fluxo | Inserção da linha pendente em `webhook_outbox` pelo `tx` | TRANSCRICAO | [09:06] Diego |
| FDD-FLUXO-07 | docs/FDD.md | Passo de fluxo | Erro na inserção desfaz a transação e o status não muda | TRANSCRICAO | [09:40] Bruno |
| FDD-FLUXO-08 | docs/FDD.md | Passo de fluxo | `bootstrap` e `shutdown` do worker no modelo de `src/server.ts` | CODIGO | src/server.ts |
| FDD-FLUXO-09 | docs/FDD.md | Passo de fluxo | Leitura dos pendentes por `created_at`, em lote pequeno | TRANSCRICAO | [09:08] Diego |
| FDD-FLUXO-10 | docs/FDD.md | Passo de fluxo | Linhas lidas marcadas como em processamento | TRANSCRICAO | [09:08] Diego |
| FDD-FLUXO-14 | docs/FDD.md | Passo de fluxo | Envio do payload gravado, sem nova renderização | TRANSCRICAO | [09:52] Larissa |
| FDD-FLUXO-15 | docs/FDD.md | Passo de fluxo | Corpo acima de 64KB não é enviado | TRANSCRICAO | [09:24] Larissa |
| FDD-FLUXO-16 | docs/FDD.md | Passo de fluxo | HMAC-SHA256 do corpo com a secret do cadastro | TRANSCRICAO | [09:22] Sofia |
| FDD-FLUXO-17 | docs/FDD.md | Passo de fluxo | Montagem dos cinco headers, com `X-Timestamp` do envio e `X-Webhook-Id` do cadastro | TRANSCRICAO | [09:45] Diego |
| FDD-FLUXO-18 | docs/FDD.md | Passo de fluxo | Envio com timeout de 10 segundos; estouro conta como falha | TRANSCRICAO | [09:42] Diego |
| FDD-FLUXO-19 | docs/FDD.md | Passo de fluxo | Registro de resposta e tempo de resposta para o histórico | TRANSCRICAO | [09:34] Marcos |
| FDD-FLUXO-21 | docs/FDD.md | Passo de fluxo | Sucesso marca a linha como entregue e grava a entrega no histórico | TRANSCRICAO | [09:08] Diego |
| FDD-FLUXO-22 | docs/FDD.md | Passo de fluxo | Falha grava a tentativa com o motivo no histórico | TRANSCRICAO | [09:34] Marcos |
| FDD-FLUXO-23 | docs/FDD.md | Passo de fluxo | Reagendamento pelo intervalo seguinte de 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| FDD-FLUXO-24 | docs/FDD.md | Passo de fluxo | Falha da 5ª nova tentativa leva à fila de mortos | TRANSCRICAO | [09:17] Larissa |
| FDD-FLUXO-27 | docs/FDD.md | Passo de fluxo | Inserção em `webhook_dead_letter` com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-30 | docs/FDD.md | Passo de fluxo | Replay exige role ADMIN via `requireRole` | TRANSCRICAO | [09:36] Larissa |
| FDD-FLUXO-31 | docs/FDD.md | Passo de fluxo | Validação do `:id` com `validate` e resposta `400 VALIDATION_ERROR` | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-FLUXO-32 | docs/FDD.md | Passo de fluxo | Registro da fila de mortos inexistente gera erro de não encontrado | CODIGO | src/shared/errors/http-errors.ts |
| FDD-FLUXO-33 | docs/FDD.md | Passo de fluxo | Replay recoloca o evento na outbox como pendente | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-34 | docs/FDD.md | Passo de fluxo | Log de auditoria de quem fez o replay | TRANSCRICAO | [09:36] Sofia |
| FDD-FLUXO-36 | docs/FDD.md | Passo de fluxo | Rotação autenticada, com `:id` validado | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-FLUXO-38 | docs/FDD.md | Passo de fluxo | Secret anterior mantida válida por 24 horas | TRANSCRICAO | [09:21] Sofia |
| FDD-FLUXO-39 | docs/FDD.md | Passo de fluxo | Nova secret devolvida ao cliente pela API | TRANSCRICAO | [09:21] Sofia |
| FDD-FLUXO-41 | docs/FDD.md | Passo de fluxo | CRUD acessível a qualquer usuário autenticado nesta fase | TRANSCRICAO | [09:37] Sofia |
| FDD-FLUXO-42 | docs/FDD.md | Passo de fluxo | Cliente informado na requisição, nunca derivado do JWT | TRANSCRICAO | [09:32] Larissa |
| FDD-FLUXO-43 | docs/FDD.md | Passo de fluxo | URL `https` obrigatória, validada no schema Zod | TRANSCRICAO | [09:23] Sofia |
| FDD-FLUXO-44 | docs/FDD.md | Passo de fluxo | Secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| FDD-FLUXO-45 | docs/FDD.md | Passo de fluxo | Edição por PATCH responde 200, sem a secret | TRANSCRICAO | [09:33] Bruno |
| FDD-FLUXO-46 | docs/FDD.md | Passo de fluxo | Remoção responde 204 sem corpo | CODIGO | src/modules/customers/customer.controller.ts |
| FDD-FLUXO-47 | docs/FDD.md | Passo de fluxo | Listagem com envelope de `paginated()` | CODIGO | src/shared/http/response.ts |
| FDD-FLUXO-48 | docs/FDD.md | Passo de fluxo | Consulta do histórico autenticada, com `:id` validado | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-FLUXO-49 | docs/FDD.md | Passo de fluxo | Histórico das últimas 100 entregas com sucesso/falha, payload, resposta e tempo | TRANSCRICAO | [09:34] Marcos |
| FDD-FLUXO-50 | docs/FDD.md | Passo de fluxo | Histórico responde 200, como as listagens existentes | CODIGO | src/modules/customers/customer.controller.ts |
| FDD-GAP-01 | docs/FDD.md | Lacuna não decidida na reunião | Granularidade da linha da outbox e local do estado de retry | TRANSCRICAO | [09:06] Diego |
| FDD-GAP-02 | docs/FDD.md | Lacuna não decidida na reunião | Papel do status "falhou" e transições diante da DLQ separada | TRANSCRICAO | [09:18] Diego |
| FDD-GAP-03 | docs/FDD.md | Lacuna não decidida na reunião | Recuperação de linhas presas em "processando" | TRANSCRICAO | [09:11] Diego |
| FDD-GAP-04 | docs/FDD.md | Lacuna não decidida na reunião | Tamanho do lote e concorrência no worker | TRANSCRICAO | [09:08] Diego |
| FDD-GAP-05 | docs/FDD.md | Lacuna não decidida na reunião | Critério de entrega bem-sucedida | TRANSCRICAO | [09:34] Marcos |
| FDD-GAP-06 | docs/FDD.md | Lacuna não decidida na reunião | Codificação, prefixo e bytes do HMAC | TRANSCRICAO | [09:22] Sofia |
| FDD-GAP-07 | docs/FDD.md | Lacuna não decidida na reunião | Headers da entrega fora da assinatura | TRANSCRICAO | [09:44] Diego |
| FDD-GAP-08 | docs/FDD.md | Lacuna não decidida na reunião | Secret em repouso e redação no logger | TRANSCRICAO | [09:21] Sofia |
| FDD-GAP-09 | docs/FDD.md | Lacuna não decidida na reunião | Criação do pedido não emite evento | TRANSCRICAO | [09:40] Bruno |
| FDD-GAP-10 | docs/FDD.md | Lacuna não decidida na reunião | Código de erro para URL sem https | TRANSCRICAO | [09:28] Bruno |
| FDD-GAP-11 | docs/FDD.md | Lacuna não decidida na reunião | Leitura de "5 tentativas" | TRANSCRICAO | [09:17] Larissa |
| FDD-GAP-12 | docs/FDD.md | Lacuna não decidida na reunião | Nomes dos campos JSON da API e da lista de status | TRANSCRICAO | [09:31] Marcos |
| FDD-GAP-13 | docs/FDD.md | Lacuna não decidida na reunião | Rotas não ditadas e posição do customerId | TRANSCRICAO | [09:32] Larissa |
| FDD-GAP-14 | docs/FDD.md | Lacuna não decidida na reunião | Nomes de arquivos e tabelas não ditados | TRANSCRICAO | [09:28] Bruno |
| FDD-GAP-15 | docs/FDD.md | Lacuna não decidida na reunião | Pendentes de cadastro removido, desativado ou com URL nova | TRANSCRICAO | [09:33] Bruno |
| FDD-GAP-16 | docs/FDD.md | Lacuna não decidida na reunião | Tratamento de payload acima de 64KB | TRANSCRICAO | [09:24] Larissa |
| FDD-GAP-17 | docs/FDD.md | Lacuna não decidida na reunião | Cliente HTTP e biblioteca de HMAC | CODIGO | package.json |
| FDD-GAP-18 | docs/FDD.md | Lacuna não decidida na reunião | Configuração do worker em constantes ou ambiente | CODIGO | src/config/env.ts |
| FDD-GAP-19 | docs/FDD.md | Lacuna não decidida na reunião | Formato e entropia da secret gerada | TRANSCRICAO | [09:31] Marcos |
| FDD-GAP-20 | docs/FDD.md | Lacuna não decidida na reunião | Métricas e tracing não discutidos | TRANSCRICAO | [09:29] Bruno |
| FDD-GAP-21 | docs/FDD.md | Lacuna não decidida na reunião | Secret que assina durante as 24 horas de carência | TRANSCRICAO | [09:21] Sofia |
| FDD-GAP-22 | docs/FDD.md | Lacuna não decidida na reunião | Limpeza e arquivamento fora do escopo | TRANSCRICAO | [09:08] Diego |
| FDD-GAP-23 | docs/FDD.md | Lacuna não decidida na reunião | Chaves estrangeiras e exclusão das tabelas novas | CODIGO | src/modules/orders/order.service.ts |
| FDD-GAP-24 | docs/FDD.md | Lacuna não decidida na reunião | Replay: id de evento, payload e contagem de tentativas | TRANSCRICAO | [09:18] Diego |
| FDD-GAP-25 | docs/FDD.md | Lacuna não decidida na reunião | Ordem por pedido diante de nova tentativa | TRANSCRICAO | [09:12] Diego |
| FDD-GAP-26 | docs/FDD.md | Lacuna não decidida na reunião | Ponto de medição da meta de 10 segundos: até o início do envio ou até a resposta do cliente | TRANSCRICAO | [09:02] Marcos |
| FDD-GAP-27 | docs/FDD.md | Lacuna não decidida na reunião | "Últimos 100": limite fixo ou exemplo; formato da resposta do histórico | TRANSCRICAO | [09:34] Marcos |
| FDD-GAP-28 | docs/FDD.md | Lacuna não decidida na reunião | Filtro de inserção ignora cadastros inativos? | TRANSCRICAO | [09:21] Sofia |
| FDD-GAP-29 | docs/FDD.md | Lacuna não decidida na reunião | Regras do cadastro: mínimo e repetição de status, valor inicial do estado ativo, inativos na listagem, campos editáveis | TRANSCRICAO | [09:21] Sofia |
| FDD-GAP-30 | docs/FDD.md | Lacuna não decidida na reunião | Método HTTP da entrega e tratamento de redirecionamento | TRANSCRICAO | [09:06] Diego |
| FDD-GAP-31 | docs/FDD.md | Lacuna não decidida na reunião | Formato do `X-Timestamp` | TRANSCRICAO | [09:44] Diego |
| FDD-GAP-32 | docs/FDD.md | Lacuna não decidida na reunião | Replay: resposta, permanência do item na fila de mortos, replay repetido e replay de cadastro removido ou inativo | TRANSCRICAO | [09:18] Diego |
| FDD-GAP-33 | docs/FDD.md | Lacuna não decidida na reunião | Rotação repetida antes do fim das 24 horas de carência | TRANSCRICAO | [09:21] Sofia |
| FDD-GAP-34 | docs/FDD.md | Lacuna não decidida na reunião | Conteúdo e tamanho da resposta do cliente guardada no histórico | TRANSCRICAO | [09:34] Marcos |
| FDD-GAP-35 | docs/FDD.md | Lacuna não decidida na reunião | Chave estrangeira de `webhookId` nas tabelas de evento e destino do histórico quando o cadastro é removido | TRANSCRICAO | [09:33] Bruno |
| FDD-GAP-36 | docs/FDD.md | Lacuna não decidida na reunião | Tamanho das colunas de URL e de secret | TRANSCRICAO | [09:21] Sofia |
| FDD-GAP-37 | docs/FDD.md | Lacuna não decidida na reunião | Opções de `timeout` e `maxWait` da transação interativa de `changeStatus` | CODIGO | src/modules/orders/order.service.ts |
| FDD-GAP-38 | docs/FDD.md | Lacuna não decidida na reunião | Alerta interno ao time quando a fila de mortos cresce | TRANSCRICAO | [09:37] Larissa |
| FDD-GAP-40 | docs/FDD.md | Lacuna não decidida na reunião | Identificação do processo nos logs: `createLogger` fixa `service` para os dois processos | CODIGO | src/shared/logger/index.ts |
| FDD-GAP-41 | docs/FDD.md | Lacuna não decidida na reunião | O worker exige o mesmo `.env` da API, inclusive `JWT_SECRET` | CODIGO | src/config/env.ts |
| FDD-GAP-42 | docs/FDD.md | Lacuna não decidida na reunião | Comportamento do encerramento do worker com lote em andamento | TRANSCRICAO | [09:11] Larissa |
| FDD-GAP-43 | docs/FDD.md | Lacuna não decidida na reunião | Unidade de medida do limite de 64KB: bytes do corpo JSON serializado ou outra grandeza | TRANSCRICAO | [09:24] Diego |
| FDD-GAP-45 | docs/FDD.md | Lacuna não decidida na reunião | Validação do destino das entregas: a URL é informada pelo usuário e o worker faz a chamada (risco de requisições a endereços internos) | TRANSCRICAO | [09:31] Marcos |
| FDD-GAP-46 | docs/FDD.md | Lacuna não decidida na reunião | Autorização do cadastro diante do registro público: a premissa de que todo usuário autenticado representa um cliente não vale no código | TRANSCRICAO | [09:32] Marcos |
| FDD-GAP-47 | docs/FDD.md | Lacuna não decidida na reunião | Teto de cadastros por cliente (amplificação do trabalho do worker único) | TRANSCRICAO | [09:38] Diego |
| FDD-HEADER-01 | docs/FDD.md | Header da entrega | `Content-Type: application/json` | TRANSCRICAO | [09:44] Diego |
| FDD-HEADER-02 | docs/FDD.md | Header da entrega | `X-Event-Id` com o UUID gerado na entrada na outbox, para deduplicação | TRANSCRICAO | [09:25] Diego |
| FDD-HEADER-03 | docs/FDD.md | Header da entrega | `X-Signature` com HMAC-SHA256 sobre o corpo, secret por endpoint | TRANSCRICAO | [09:22] Sofia |
| FDD-HEADER-04 | docs/FDD.md | Header da entrega | `X-Timestamp` com o timestamp do envio, para detecção de replay | TRANSCRICAO | [09:44] Diego |
| FDD-HEADER-05 | docs/FDD.md | Header da entrega | `X-Webhook-Id` com o id do cadastro, para cliente com vários cadastros | TRANSCRICAO | [09:45] Diego |
| FDD-INT-01 | docs/FDD.md | Integração com o código existente | `publishWebhookEvent(tx, order, from, to)` chamada na transação de `changeStatus`, entre o histórico e a releitura; exceção desfaz tudo; construtor, `create` e `delete` não mudam | TRANSCRICAO | [09:41] Diego |
| FDD-INT-02 | docs/FDD.md | Integração com o código existente | Quatro models novos com convenções do schema e migration apenas aditiva | CODIGO | prisma/schema.prisma |
| FDD-INT-03 | docs/FDD.md | Integração com o código existente | Classes de erro `WEBHOOK_` em `http-errors.ts`; `WEBHOOK_NOT_FOUND` estende `AppError` porque `NotFoundError` fixa `NOT_FOUND` | TRANSCRICAO | [09:29] Larissa |
| FDD-INT-04 | docs/FDD.md | Integração com o código existente | `errorMiddleware` sem mudança: já responde qualquer `AppError` | TRANSCRICAO | [09:29] Bruno |
| FDD-INT-05 | docs/FDD.md | Integração com o código existente | `validate` sem mudança; exigência de `https` e demais regras no schema Zod do módulo | TRANSCRICAO | [09:23] Sofia |
| FDD-INT-06 | docs/FDD.md | Integração com o código existente | `authenticate` nas rotas do módulo e `requireRole('ADMIN')` no replay, sem mudar o middleware | TRANSCRICAO | [09:36] Larissa |
| FDD-INT-07 | docs/FDD.md | Integração com o código existente | `buildControllers` instancia repository, service e controller do módulo; `buildApp` não muda | CODIGO | src/app.ts |
| FDD-INT-08 | docs/FDD.md | Integração com o código existente | `Controllers` ganha `webhooks`; `buildApiRouter` monta `/webhooks` e `/admin/webhooks` | CODIGO | src/routes/index.ts |
| FDD-INT-09 | docs/FDD.md | Integração com o código existente | `src/server.ts` sem mudança, usado como modelo de bootstrap e shutdown de `src/worker.ts` (novo) | TRANSCRICAO | [09:11] Larissa |
| FDD-INT-10 | docs/FDD.md | Integração com o código existente | Worker obtém instância própria de `PrismaClient` via `createPrismaClient`, no mesmo banco | TRANSCRICAO | [09:30] Bruno |
| FDD-INT-11 | docs/FDD.md | Integração com o código existente | `env.ts` sem mudança; valores do worker como constantes; worker herda a validação de todas as variáveis | CODIGO | src/config/env.ts |
| FDD-INT-12 | docs/FDD.md | Integração com o código existente | Logger Pino reusado sem mudança; extensão da redação apenas como proposta | TRANSCRICAO | [09:29] Bruno |
| FDD-INT-13 | docs/FDD.md | Integração com o código existente | `paginated()` usado na listagem de cadastros, sem mudança no arquivo | CODIGO | src/shared/http/response.ts |
| FDD-INT-14 | docs/FDD.md | Integração com o código existente | Scripts `worker` e `start:worker` no molde de `dev` e `start`; nenhuma dependência nova | CODIGO | package.json |
| FDD-INT-15 | docs/FDD.md | Integração com o código existente | Limpeza das tabelas novas em `tests/setup.ts`, reuso das fábricas e testes de pedidos inalterados | CODIGO | tests/setup.ts |
| FDD-OBJ-01 | docs/FDD.md | Objetivo técnico | Registrar o evento na mesma transação SQL da mudança de status | TRANSCRICAO | [09:06] Diego |
| FDD-OBJ-02 | docs/FDD.md | Objetivo técnico | Envio inicial em menos de 10 segundos após o commit | TRANSCRICAO | [09:02] Marcos |
| FDD-OBJ-03 | docs/FDD.md | Objetivo técnico | Suportar indisponibilidade com novas tentativas, teto e DLQ | TRANSCRICAO | [09:15] Diego |
| FDD-OBJ-04 | docs/FDD.md | Objetivo técnico | Cliente valida origem e integridade pela assinatura | TRANSCRICAO | [09:19] Sofia |
| FDD-OBJ-05 | docs/FDD.md | Objetivo técnico | Reuso dos padrões existentes do projeto | TRANSCRICAO | [09:30] Larissa |
| FDD-OBJ-06 | docs/FDD.md | Objetivo técnico | Nenhuma chamada HTTP na transação; entrega em processo separado | TRANSCRICAO | [09:04] Bruno |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logs do módulo e do worker só pelo Pino existente, no formato de `http_request`, sem `console.log` | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Nenhum log carrega a secret; redação de `*.secret`; corpo do evento e resposta ficam fora dos logs | TRANSCRICAO | [09:22] Diego |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | `X-Request-Id` e `http_request` continuam nas rotas do módulo; eventos da API levam `requestId` | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-OBS-05 | docs/FDD.md | Observabilidade | Log de quem fez o replay da DLQ (`webhook_dead_letter_replayed`) | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-10 | docs/FDD.md | Observabilidade | Log `bootstrap_failed` do worker no padrão do entry-point da API | CODIGO | src/server.ts |
| FDD-OBS-16 | docs/FDD.md | Observabilidade | Projeto sem biblioteca de métricas; métricas por contagem de logs e consulta às tabelas | CODIGO | package.json |
| FDD-OBS-23 | docs/FDD.md | Observabilidade | Projeto sem tracing distribuído; esta entrega não o introduz | CODIGO | package.json |
| FDD-OOS-01 | docs/FDD.md | Fora de escopo (exclusão) | E-mail de alerta ao cliente | TRANSCRICAO | [09:37] Larissa |
| FDD-OOS-02 | docs/FDD.md | Fora de escopo (exclusão) | Painel visual para o cliente | TRANSCRICAO | [09:40] Larissa |
| FDD-OOS-03 | docs/FDD.md | Fora de escopo (exclusão) | Limite de envio por cliente, a observar | TRANSCRICAO | [09:39] Larissa |
| FDD-OOS-04 | docs/FDD.md | Fora de escopo (exclusão) | Arquivamento das linhas entregues depois de 30 dias | TRANSCRICAO | [09:08] Diego |
| FDD-OOS-05 | docs/FDD.md | Fora de escopo (exclusão) | Múltiplos workers em paralelo | TRANSCRICAO | [09:13] Diego |
| FDD-OOS-06 | docs/FDD.md | Fora de escopo (exclusão) | Webhooks de entrada | TRANSCRICAO | [09:02] Marcos |
| FDD-OOS-07 | docs/FDD.md | Fora de escopo (exclusão) | Itens do pedido no payload | TRANSCRICAO | [09:43] Diego |
| FDD-RES-01 | docs/FDD.md | Resiliência | Atomicidade entre mudança de status e inserção na outbox | TRANSCRICAO | [09:40] Bruno |
| FDD-RES-02 | docs/FDD.md | Resiliência | Polling da outbox a cada 2 segundos | TRANSCRICAO | [09:10] Larissa |
| FDD-RES-03 | docs/FDD.md | Resiliência | Timeout de 10 segundos no envio | TRANSCRICAO | [09:42] Diego |
| FDD-RES-04 | docs/FDD.md | Resiliência | Tabela de intervalos 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| FDD-RES-05 | docs/FDD.md | Resiliência | Teto de 5 tentativas | TRANSCRICAO | [09:17] Larissa |
| FDD-RES-06 | docs/FDD.md | Resiliência | Fila de mortos em `webhook_dead_letter` | TRANSCRICAO | [09:18] Diego |
| FDD-RES-07 | docs/FDD.md | Resiliência | Replay manual com role ADMIN e auditoria | TRANSCRICAO | [09:36] Larissa |
| FDD-RES-08 | docs/FDD.md | Resiliência | At-least-once com deduplicação pelo `X-Event-Id` | TRANSCRICAO | [09:26] Larissa |
| FDD-RES-09 | docs/FDD.md | Resiliência | Carência de 24 horas na rotação de secret | TRANSCRICAO | [09:22] Sofia |
| FDD-RES-10 | docs/FDD.md | Resiliência | Limite de 64KB no corpo enviado | TRANSCRICAO | [09:24] Larissa |
| FDD-RES-11 | docs/FDD.md | Resiliência | Ordem por pedido com worker único | TRANSCRICAO | [09:13] Larissa |
| FDD-RISCO-01 | docs/FDD.md | Risco | Endpoint lento faz eventos de outros clientes passarem da meta de 10 segundos | TRANSCRICAO | [09:42] Diego |
| FDD-RISCO-02 | docs/FDD.md | Risco | Falha no registro do evento impede a mudança de status | TRANSCRICAO | [09:40] Bruno |
| FDD-RISCO-03 | docs/FDD.md | Risco | Secret registrada em log do cliente ou da plataforma | TRANSCRICAO | [09:22] Diego |
| FDD-RISCO-04 | docs/FDD.md | Risco | Cliente processa evento repetido por causa do at-least-once | TRANSCRICAO | [09:24] Diego |
| FDD-RISCO-05 | docs/FDD.md | Risco | Qualquer autenticado cadastra webhook para qualquer cliente | TRANSCRICAO | [09:37] Sofia |
| FDD-RISCO-06 | docs/FDD.md | Risco | Role ADMIN obtida por autorregistro com `role` no corpo da rota pública | CODIGO | src/modules/auth/auth.schemas.ts |
| FDD-RISCO-07 | docs/FDD.md | Risco | Transação de `changeStatus` mais longa, aberta sem opções | CODIGO | src/modules/orders/order.service.ts |
| FDD-RISCO-08 | docs/FDD.md | Risco | Worker único parado e linhas presas em processamento | TRANSCRICAO | [09:11] Diego |
| FDD-RISCO-09 | docs/FDD.md | Risco | Preservação de ordem atrasa eventos do mesmo pedido por até quase 15 horas | TRANSCRICAO | [09:17] Diego |
| FDD-RISCO-10 | docs/FDD.md | Risco | Destino das entregas é URL informada pelo usuário e o worker faz a chamada: risco de requisições a endereços internos (inferência do documento) | TRANSCRICAO | [09:31] Marcos |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Evento de mudança de status registrado em outbox no MySQL, na mesma transação que atualiza o pedido e o histórico, com rollback se a inserção falhar; worker separado faz as chamadas HTTP | TRANSCRICAO | [09:08] Larissa |
| ADR-002 | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisão | Entrega por worker em polling de 2 segundos, em processo separado da API e único (ordenação implícita por order_id) | TRANSCRICAO | [09:10] Larissa |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | Até 5 novas tentativas com intervalos 1m/5m/30m/2h/12h; esgotado o teto, o evento vai para a tabela separada `webhook_dead_letter`; replay manual por endpoint admin recoloca o evento na outbox como pendente | TRANSCRICAO | [09:17] Larissa |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Assinatura HMAC-SHA256 sobre o corpo do request no header X-Signature; secret única por endpoint, gerada pela plataforma e devolvida na criação; rotação pela API com a antiga válida por 24 horas em paralelo | TRANSCRICAO | [09:22] Sofia |
| ADR-005 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | Entrega at-least-once; UUID por evento gerado na entrada da outbox e enviado no header `X-Event-Id`; cliente deduplica por esse id; comportamento documentado no portal do desenvolvedor | TRANSCRICAO | [09:26] Larissa |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Reuso máximo dos padrões existentes no módulo de webhooks: AppError, Pino, error middleware, padrão de módulos, schemas Zod e códigos de erro com prefixo WEBHOOK_ | TRANSCRICAO | [09:30] Larissa |
| ADR-007 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Decisão | A linha da outbox guarda o payload já renderizado na inserção (snapshot), não só o `order_id`; o evento reflete o estado do pedido de quando o status mudou | TRANSCRICAO | [09:52] Bruno |
| ADR-008 | docs/adrs/ADR-008-filtro-de-eventos-na-insercao.md | Decisão | Cada webhook tem uma lista de status, e o filtro é aplicado na inserção na outbox; sem webhook do cliente que assine o status de destino, nenhuma linha é inserida | TRANSCRICAO | [09:34] Bruno |
| ADR-001-ALT-1 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa considerada | Disparo síncrono da notificação no service de pedidos, rejeitado por travar a transação de mudança de status | TRANSCRICAO | [09:03] Larissa |
| ADR-002-ALT-1 | docs/adrs/ADR-002-worker-separado-com-polling.md | Alternativa considerada | Worker dentro da mesma instância da API; descartada porque o reinício da API derruba o worker | TRANSCRICAO | [09:11] Diego |
| ADR-003-ALT-1 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Alternativa considerada | Retry indefinido com backoff, descartado por deixar o evento pendurado para sempre se o cliente sumiu | TRANSCRICAO | [09:15] Diego |
| ADR-004-ALT-1 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Alternativa considerada | Secret global da plataforma, descartada porque, se vaza uma, vaza tudo | TRANSCRICAO | [09:21] Sofia |
| ADR-005-ALT-1 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Alternativa considerada | Garantia exactly-once, descartada por exigir coordenação dos dois lados e ser muito mais complexa | TRANSCRICAO | [09:25] Diego |
| ADR-006-ALT-1 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Alternativa considerada | Injetar o repository do webhook no OrderService em vez de uma função que recebe o tx | TRANSCRICAO | [09:41] Bruno |
| ADR-007-ALT-1 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Alternativa considerada | Guardar só o `order_id` e renderizar o payload na hora do envio; descartada em [09:52] Larissa pelo "caso esquisito" | TRANSCRICAO | [09:51] Bruno |
| ADR-008-ALT-1 | docs/adrs/ADR-008-filtro-de-eventos-na-insercao.md | Alternativa considerada | Filtrar no momento do envio, e não na inserção; descartada em favor da inserção | TRANSCRICAO | [09:34] Diego |
| ADR-001-ALT-2 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa considerada | Redis Streams / Redis Cluster como fila, descartado por exigir infraestrutura nova (overengineering para time pequeno) | TRANSCRICAO | [09:07] Larissa |
| ADR-002-ALT-2 | docs/adrs/ADR-002-worker-separado-com-polling.md | Alternativa considerada | Trigger do banco para o worker reagir imediatamente; descartada por falta de listener nativo no MySQL | TRANSCRICAO | [09:09] Bruno |
| ADR-003-ALT-2 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Alternativa considerada | 3 tentativas (mais agressivo), descartado por matar o evento em indisponibilidades como a de duas horas em manutenção planejada | TRANSCRICAO | [09:16] Bruno |
| ADR-006-ALT-2 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Alternativa considerada | Worker usar a mesma instância de PrismaClient da API | TRANSCRICAO | [09:29] Diego |
| ADR-002-ALT-3 | docs/adrs/ADR-002-worker-separado-com-polling.md | Alternativa considerada | Múltiplos workers em paralelo; não adotada nesta fase por perder a ordenação por order_id | TRANSCRICAO | [09:12] Diego |
| ADR-003-ALT-3 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Alternativa considerada | Marcar como "failed" na própria outbox, preterido pela tabela separada `webhook_dead_letter` | TRANSCRICAO | [09:17] Larissa |

## Apêndice — propostas sem origem direta, a confirmar na revisão

Estes itens foram introduzidos pela documentação porque a reunião não fechou o detalhe correspondente. Cada um está marcado como proposta no documento onde aparece e responde a uma lacuna (`-GAP-`) da seção 15 do FDD. Não são requisito nem decisão e, por não terem origem na transcrição nem no código, ficam fora da tabela de rastreabilidade: a lista abaixo só registra onde cada uma aparece.

- `FDD-DADOS-15` (docs/FDD.md, Elemento de modelo de dados): Índice proposto em `webhooks.customerId`
- `FDD-DADOS-16` (docs/FDD.md, Elemento de modelo de dados): Uso das colunas de secret na rotação e na assinatura
- `FDD-DADOS-19` (docs/FDD.md, Elemento de modelo de dados): `webhook_outbox.webhookId`, cadastro de destino
- `FDD-DADOS-20` (docs/FDD.md, Elemento de modelo de dados): `webhook_outbox.orderId` sem chave estrangeira
- `FDD-DADOS-21` (docs/FDD.md, Elemento de modelo de dados): `webhook_outbox.customerId` sem chave estrangeira
- `FDD-DADOS-24` (docs/FDD.md, Elemento de modelo de dados): `webhook_outbox.attempts`, contagem de envios
- `FDD-DADOS-25` (docs/FDD.md, Elemento de modelo de dados): `webhook_outbox.nextAttemptAt`, próxima tentativa
- `FDD-DADOS-29` (docs/FDD.md, Elemento de modelo de dados): Histórico com uma linha por request enviado
- `FDD-DADOS-32` (docs/FDD.md, Elemento de modelo de dados): `webhook_deliveries.eventId` para correlação
- `FDD-DADOS-37` (docs/FDD.md, Elemento de modelo de dados): `webhook_deliveries.errorMessage` para falha sem resposta
- `FDD-DADOS-40` (docs/FDD.md, Elemento de modelo de dados): Índices propostos em `webhookId` e `createdAt` do histórico
- `FDD-DADOS-42` (docs/FDD.md, Elemento de modelo de dados): `webhook_dead_letter.outboxId` para recolocar a mesma linha
- `FDD-DADOS-43` (docs/FDD.md, Elemento de modelo de dados): `webhook_dead_letter.eventId` preservado no replay
- `FDD-DADOS-44` (docs/FDD.md, Elemento de modelo de dados): `webhook_dead_letter.webhookId`, cadastro de destino
- `FDD-DADOS-45` (docs/FDD.md, Elemento de modelo de dados): `webhook_dead_letter.orderId` sem chave estrangeira
- `FDD-DADOS-46` (docs/FDD.md, Elemento de modelo de dados): `webhook_dead_letter.customerId` sem chave estrangeira
- `FDD-DADOS-49` (docs/FDD.md, Elemento de modelo de dados): `webhook_dead_letter.attempts`, envios até a DLQ
- `FDD-DADOS-51` (docs/FDD.md, Elemento de modelo de dados): DLQ sem índice além da chave primária
- `FDD-DADOS-52` (docs/FDD.md, Elemento de modelo de dados): Remoção de pedido não afeta as linhas de evento (sem chave estrangeira)
- `FDD-DADOS-53` (docs/FDD.md, Elemento de modelo de dados): Tabelas de evento sem chave estrangeira para `customers`
- `FDD-DADOS-54` (docs/FDD.md, Elemento de modelo de dados): Chave estrangeira `webhooks.customerId` com `ON DELETE RESTRICT`
- `FDD-DADOS-55` (docs/FDD.md, Elemento de modelo de dados): Entregas pendentes de cadastro removido ou inativo vão para a DLQ
- `FDD-DEP-07` (docs/FDD.md, Dependência ou compatibilidade): `node:crypto` para HMAC-SHA256 e geração da secret
- `FDD-DEP-08` (docs/FDD.md, Dependência ou compatibilidade): `fetch` global do Node 20 para a entrega, com timeout de 10 segundos
- `FDD-DEP-10` (docs/FDD.md, Dependência ou compatibilidade): Nenhuma dependência nova
- `FDD-ERRO-02` (docs/FDD.md, Erro): `WEBHOOK_DEAD_LETTER_NOT_FOUND` (404) para item inexistente no replay
- `FDD-ERRO-05` (docs/FDD.md, Erro): Erro de conexão como motivo de falha de entrega
- `FDD-ERRO-06` (docs/FDD.md, Erro): Resposta fora de 2xx como motivo de falha de entrega
- `FDD-ERRO-07` (docs/FDD.md, Erro): Cadastro desativado com entrega pendente vai para a DLQ
- `FDD-ERRO-08` (docs/FDD.md, Erro): Cadastro removido com entrega pendente vai para a DLQ
- `FDD-FLUXO-11` (docs/FDD.md, Passo de fluxo): Paralelismo dentro do lote e ciclos sem sobreposição
- `FDD-FLUXO-12` (docs/FDD.md, Passo de fluxo): Linhas presas em processamento após queda; retomada é proposta
- `FDD-FLUXO-13` (docs/FDD.md, Passo de fluxo): Cadastro inativo ou removido desvia a linha para a fila de mortos
- `FDD-FLUXO-20` (docs/FDD.md, Passo de fluxo): Classificação: 2xx é sucesso; demais respostas, erro de rede e timeout são falha
- `FDD-FLUXO-25` (docs/FDD.md, Passo de fluxo): Log estruturado por tentativa com `event_id`
- `FDD-FLUXO-26` (docs/FDD.md, Passo de fluxo): Linha marcada como falhou antes da ida à fila de mortos
- `FDD-FLUXO-28` (docs/FDD.md, Passo de fluxo): Repetição da movimentação para a fila de mortos no ciclo seguinte, em caso de falha
- `FDD-FLUXO-29` (docs/FDD.md, Passo de fluxo): Log estruturado da ida à fila de mortos
- `FDD-FLUXO-35` (docs/FDD.md, Passo de fluxo): Replay de evento com cadastro inativo ou removido volta à fila de mortos
- `FDD-FLUXO-37` (docs/FDD.md, Passo de fluxo): Geração da nova secret com gerador seguro
- `FDD-FLUXO-40` (docs/FDD.md, Passo de fluxo): Secret usada na assinatura durante a carência
- `FDD-GAP-39` (docs/FDD.md, Lacuna do desenho proposto): Log de publicação emitido dentro da transação, antes do commit
- `FDD-GAP-44` (docs/FDD.md, Lacuna do desenho proposto): Quantidade e granularidade das migrations da feature
- `FDD-OBS-04` (docs/FDD.md, Observabilidade): Logger filho no worker para distinguir o processo, já que `createLogger` fixa `base.service`
- `FDD-OBS-06` (docs/FDD.md, Observabilidade): Log `webhook_config_changed` nas operações de cadastro e rotação de secret
- `FDD-OBS-07` (docs/FDD.md, Observabilidade): Log `webhook_event_published` em `publishWebhookEvent`, com quantidade de linhas inseridas
- `FDD-OBS-08` (docs/FDD.md, Observabilidade): Log `webhook_worker_started` com intervalo de polling e tamanho do lote
- `FDD-OBS-09` (docs/FDD.md, Observabilidade): Log `webhook_worker_stopped` com o sinal recebido
- `FDD-OBS-11` (docs/FDD.md, Observabilidade): Log `webhook_delivery_succeeded` com tentativa, status HTTP, duração e latência
- `FDD-OBS-12` (docs/FDD.md, Observabilidade): Log `webhook_delivery_failed` com tipo de falha e próxima tentativa
- `FDD-OBS-13` (docs/FDD.md, Observabilidade): Log `webhook_dead_lettered` com total de tentativas e motivo
- `FDD-OBS-14` (docs/FDD.md, Observabilidade): Log `webhook_stuck_recovered` na retomada de linhas presas em processamento
- `FDD-OBS-15` (docs/FDD.md, Observabilidade): Log `webhook_poll_failed` quando o ciclo de polling falha
- `FDD-OBS-17` (docs/FDD.md, Observabilidade): Métrica de eventos inseridos na outbox
- `FDD-OBS-18` (docs/FDD.md, Observabilidade): Métrica de entregas por resultado
- `FDD-OBS-19` (docs/FDD.md, Observabilidade): Métrica de tentativas por evento
- `FDD-OBS-20` (docs/FDD.md, Observabilidade): Métrica de eventos na fila de mortos
- `FDD-OBS-21` (docs/FDD.md, Observabilidade): Métrica de tempo entre a inserção e a entrega, comparada ao limite de 10 segundos
- `FDD-OBS-22` (docs/FDD.md, Observabilidade): Métrica de eventos pendentes por consulta à outbox
- `FDD-OBS-24` (docs/FDD.md, Observabilidade): Correlação pelo identificador do evento entre outbox, logs, histórico e `X-Event-Id`
- `FDD-PROP-01` (docs/FDD.md, Proposta a confirmar): Uma linha por evento × destino
- `FDD-PROP-02` (docs/FDD.md, Proposta a confirmar): Pendente com próxima tentativa; "falhou" antes da DLQ; saída da outbox
- `FDD-PROP-03` (docs/FDD.md, Proposta a confirmar): Retomar linhas presas após limite a definir
- `FDD-PROP-04` (docs/FDD.md, Proposta a confirmar): Lote pequeno com envio paralelo; números a definir
- `FDD-PROP-05` (docs/FDD.md, Proposta a confirmar): 2xx é sucesso; o resto é falha
- `FDD-PROP-06` (docs/FDD.md, Proposta a confirmar): HMAC sobre os bytes enviados, hexadecimal, sem prefixo
- `FDD-PROP-07` (docs/FDD.md, Proposta a confirmar): Deduplicação e replay pelos campos assinados do corpo
- `FDD-PROP-08` (docs/FDD.md, Proposta a confirmar): Secret recuperável e redação estendida no logger
- `FDD-PROP-09` (docs/FDD.md, Proposta a confirmar): Emissão só em changeStatus
- `FDD-PROP-10` (docs/FDD.md, Proposta a confirmar): URL sem https rejeitada pelo Zod com VALIDATION_ERROR
- `FDD-PROP-11` (docs/FDD.md, Proposta a confirmar): 1 envio inicial + 5 novas tentativas
- `FDD-PROP-12` (docs/FDD.md, Proposta a confirmar): camelCase e campo statuses
- `FDD-PROP-13` (docs/FDD.md, Proposta a confirmar): customerId em body/query; rota de rotação; sem consulta por id
- `FDD-PROP-14` (docs/FDD.md, Proposta a confirmar): Nomes de arquivos e tabelas usados no documento
- `FDD-PROP-15` (docs/FDD.md, Proposta a confirmar): Pendentes de cadastro inativo ou removido vão para a DLQ
- `FDD-PROP-16` (docs/FDD.md, Proposta a confirmar): Payload acima de 64KB direto para a DLQ
- `FDD-PROP-17` (docs/FDD.md, Proposta a confirmar): fetch global e node:crypto, sem dependência nova
- `FDD-PROP-18` (docs/FDD.md, Proposta a confirmar): Valores decididos como constantes do módulo
- `FDD-PROP-19` (docs/FDD.md, Proposta a confirmar): Gerador seguro do Node; tamanho com Sofia
- `FDD-PROP-20` (docs/FDD.md, Proposta a confirmar): Métricas por logs e correlação por event_id
- `FDD-PROP-21` (docs/FDD.md, Proposta a confirmar): Assinar com a secret antiga durante as 24 horas
- `FDD-PROP-22` (docs/FDD.md, Proposta a confirmar): Tabelas de evento sem FK para pedido e cliente
- `FDD-PROP-23` (docs/FDD.md, Proposta a confirmar): Replay com mesmo id e payload, contagem zerada
- `FDD-PROP-24` (docs/FDD.md, Proposta a confirmar): Não entregar evento posterior com anterior pendente
- `FDD-PROP-25` (docs/FDD.md, Proposta a confirmar): A meta de 10 segundos é medida do commit da mudança de status até o início do envio inicial; as novas tentativas ficam fora da meta. A métrica de produto do PRD (PRD-OBJ-01) mede até a chegada ao cliente: as duas leituras devem ser reconciliadas na revisão
- `FDD-PROP-26` (docs/FDD.md, Proposta a confirmar): O histórico devolve as últimas 100 entregas em lista simples, na chave `data`, sem paginação
- `FDD-PROP-27` (docs/FDD.md, Proposta a confirmar): O filtro de inserção só considera cadastros ativos
- `FDD-PROP-28` (docs/FDD.md, Proposta a confirmar): `statuses` aceita ao menos um valor, sem repetição
- `FDD-PROP-29` (docs/FDD.md, Proposta a confirmar): O cadastro nasce com `active: true`
- `FDD-PROP-30` (docs/FDD.md, Proposta a confirmar): A listagem inclui os cadastros inativos, com `active: false`
- `FDD-PROP-31` (docs/FDD.md, Proposta a confirmar): A edição aceita `url`, `statuses` e `active`; não troca o cliente do cadastro
- `FDD-PROP-32` (docs/FDD.md, Proposta a confirmar): Rotação e replay respondem `200`; a rotação devolve a nova secret e o instante em que a anterior deixa de valer; o replay devolve os identificadores do evento e do cadastro
- `FDD-PROP-33` (docs/FDD.md, Proposta a confirmar): A entrega usa `POST` e não segue redirecionamento; resposta 3xx conta como falha
- `FDD-PROP-34` (docs/FDD.md, Proposta a confirmar): `X-Timestamp` vai em ISO 8601, em UTC
- `FDD-PROP-35` (docs/FDD.md, Proposta a confirmar): O item da fila de mortos permanece nela após o replay, com a data do último replay
- `FDD-PROP-36` (docs/FDD.md, Proposta a confirmar): Em rotação repetida, a secret que ainda estava em carência é descartada; só a última anterior é mantida
- `FDD-PROP-37` (docs/FDD.md, Proposta a confirmar): O histórico guarda o status HTTP e o corpo da resposta do cliente, com o corpo truncado num limite a definir
- `FDD-PROP-38` (docs/FDD.md, Proposta a confirmar): `webhookId` nas tabelas de evento não tem chave estrangeira; o histórico é mantido quando o cadastro é removido
- `FDD-PROP-39` (docs/FDD.md, Proposta a confirmar): Manter os padrões do Prisma na transação de `changeStatus` e medir a duração dela; definir `timeout` explícito só se a medição justificar
- `FDD-PROP-40` (docs/FDD.md, Proposta a confirmar): O encerramento do worker aguarda o término do lote corrente antes de desconectar o Prisma
- `FDD-PROP-41` (docs/FDD.md, Proposta a confirmar): O limite de 64KB aplica-se aos bytes do corpo JSON serializado em UTF-8
- `FDD-PROP-42` (docs/FDD.md, Proposta a confirmar): Uma única migration cria as quatro tabelas novas
- `FDD-PROP-43` (docs/FDD.md, Proposta a confirmar): O destino é validado no cadastro e a cada envio: o worker recusa os que resolvem para endereços privados, loopback ou link-local, conecta ao endereço validado (contra troca de DNS) e exige certificado TLS válido; entregas com destino recusado não têm resposta guardada
- `FDD-PROP-44` (docs/FDD.md, Proposta a confirmar): Até existir vínculo entre usuário e cliente, restringir os contratos de gerenciamento (FDD-CONTRATO-01 a FDD-CONTRATO-06) a `requireRole('ADMIN')` e checar a quem pertence o cadastro nas rotas por id. Reabre a decisão de `[09:37] Sofia`; não é requisito
- `FDD-PROP-45` (docs/FDD.md, Proposta a confirmar): Teto pequeno de cadastros por cliente, com número a fixar na revisão do FDD
