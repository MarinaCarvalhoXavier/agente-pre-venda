# Agente de triagem e precificação

Um agente que leva um lead de Discovery Enterprise da transcrição da reunião até a proposta pronta (Word e PDF), dentro do Notion da CITi. A IA lê e pergunta, uma regra fixa decide o preço, e duas pessoas aprovam no meio: o gerente de contas envia as perguntas ao cliente e o líder de Produto aprova o preço.

Este repositório tem o workflow do n8n exportado e limpo (sem chaves, sem ids internos e sem dados de cliente) e a documentação de como ele funciona.

## O problema

A precificação dependia de alguém do Produto ter tempo. Cada etapa (perguntas ao cliente, leitura das respostas, preço, proposta) esperava essa pessoa. Antes das skills, cada lead custava cerca de 5 h do Produto.

## Como funciona

Cada lead é uma página no database "Discovery Enterprise" do Notion. Uma pessoa muda o Status; o workflow confere a cada 2 minutos e responde escrevendo uma seção nova na página.

| Quem | Muda o Status para | O agente faz | Status depois |
| --- | --- | --- | --- |
| Gerente de contas | Novo (com a transcrição) | Triagem: riscos, perguntas ao cliente, mensagem de WhatsApp pronta | Perguntas prontas |
| Gerente de contas | Mais perguntas (com as respostas coladas) | 2ª rodada: só o que ficou em aberto, no máximo 4 perguntas | Perguntas prontas |
| Gerente de contas | Pronto para precificar | Revisor de coerência + cálculo pela regra, menção ao líder | Aguardando aprovação |
| Líder de Produto | Aprovado ou Ajustar (com preço final e motivo) | Registra a decisão e avisa o gerente | Gerando proposta |
| (automático) | — | Proposta com a skill de proposta (Word e PDF) e lista do que conferir | Proposta |
| (qualquer falha) | — | Escreve o erro na página | Erro no agente |

Regras de operação:

- Nada vai ao cliente sem uma pessoa: o gerente revisa e envia a mensagem, o líder aprova o preço.
- Cada seção nova recolhe as anteriores em toggles; a página mostra só o passo atual aberto.
- Toda seção termina com o custo da rodada e o total do lead até ali.

## Arquitetura

Um workflow no n8n Cloud (48 nós) dispara a cada 2 minutos e faz uma busca no Notion por fase:

| Ramo | Busca (Status) | Caminho principal |
| --- | --- | --- |
| Triagem | vazio, Novo ou Mais perguntas | Extrair texto dos arquivos → Preparar triagem → Triagem (Claude) → Calcular preço → Mensagem de WhatsApp (Claude) → Montar página |
| Precificação | Pronto para precificar | Triagem → Revisar coerência (Claude) → Calcular preço → Montar página |
| Decisão | Aprovado ou Ajustar | Montar decisão (preço final, diferença, menções) |
| Proposta | Gerando proposta (um por vez) | Ler arquivos → Escrever proposta (Claude + skills, loop de pause_turn) → Baixar arquivos → Subir no Notion |

Todos os ramos terminam em: atualizar propriedades → recolher histórico → escrever na página. Os nós de Claude, de código e de HTTP têm saída de erro ligada a "Montar erro".

Peças importantes:

- **A IA responde critérios; o código decide.** A triagem (Claude Sonnet 4.6, temperatura 0) devolve JSON com critérios justificados pelo trecho da transcrição. O nó "Calcular preço" aplica a regra. Isso impede que a IA, ou quem vende, negocie o preço.
- **Revisor de coerência:** uma segunda chamada impede o agente de perguntar de novo o que o cliente já respondeu.
- **Respostas do cliente valem mais que a transcrição:** a triagem confere cada resposta antes de perguntar.
- **Proposta:** Messages API com `code_execution` e skills (uma skill própria de proposta e a skill de docx da Anthropic), com loop de `pause_turn` e arquivos baixados pela Files API.

## Regras de precificação

1. **Semanas:** 3 ou mais frentes → 6. Uma frente, até 3 usuários-chave confirmados pelo cliente e fluxo atual descrito por ele por escrito → exceção de 4. Qualquer outro caso → 5. Decisão colegiada durante o Discovery → +1 (máximo 6).
2. **Time:** 1 gerente, 1 especialista e 2 analistas.
3. **Preço:** unidades × semanas × 5 dias × valor da diária, arredondado para a centena.
4. **Fora do padrão** (o preço continua o de tabela e o líder decide o ajuste): criticidade alta, escopo amplo demais, trabalho de campo distante ou sensibilidade a preço.

## Custo de IA por lead

| Etapa | Custo por rodada (US$) |
| --- | --- |
| Triagem | 0,15 |
| 2ª rodada (se houver) | ~0,15 |
| Precificação | 0,21 |
| Proposta | 0,39 (era 6,70) |
| **Lead completo** | **~0,75 a 0,90** |

O custo da proposta caiu de US$ 6,70 para US$ 0,39 sem trocar de modelo: o PDF passou a ir como texto extraído (em vez de uma imagem por página), o pedido usa prompt caching no loop de `pause_turn`, e o Claude não reimprime arquivos inteiros.

## Para importar

1. No n8n: **Import from file** e escolha `workflow.json`.
2. Crie as credenciais e ligue em cada nó marcado com `CREDENCIAL_REMOVIDA`: Notion (integração com acesso ao database) e Anthropic (API key).
3. Troque `<ID_NOTION>` pelo id do seu database e `skill_<SEU_ID>` pelo id da sua skill de proposta.
4. O database precisa das propriedades: Status, Semáforo, Criticidade, Gerente de contas, Transcrição, Respostas do cliente, Preço de tabela, Semanas, Fora do padrão, Preço final, Motivo do ajuste, Proposta, Custo do agente (US$).

## Limites

- Testado em leads reais do CITi, anonimizados. É um piloto, não uma amostra.
- Na 2ª rodada, às vezes pede de novo um documento já prometido ou junta duas perguntas num item. O gerente revisa antes de enviar.

## Feito por

Marina Xavier, Líder de Produto na CITi. Portfólio: https://marinaxavier.vercel.app
