# Sistema de Webhooks de Notificação de Pedidos: Design Docs

Este repositório contém o pacote de design docs produzido para a feature de Webhooks de Notificação de Pedidos, a partir da transcrição de uma reunião técnica e do código existente do OMS.

## Sobre o desafio

O desafio propõe transformar a transcrição de uma reunião técnica em um pacote completo de documentação de engenharia, usando IA como ferramenta principal de produção. A reunião reúne tech lead, PM, engenheiros e segurança discutindo a construção de um sistema de webhooks para notificar sistemas externos sobre mudanças de status de pedidos em um Order Management System já em produção.

O trabalho consistiu em ler a transcrição e o código com atenção, separar o que foi decidido do que foi descartado ou adiado, e produzir PRD, RFC, FDD, ADRs e um tracker de rastreabilidade que amarra cada afirmação dos documentos à sua origem real, seja um trecho da transcrição, seja um arquivo do código.

## Ferramentas de IA utilizadas

- **Claude Code (modelo Sonnet 5)**: ferramenta principal de produção. Usada para ler a transcrição completa e o código-fonte do OMS, revisar o material de apoio do capítulo "Design Docs com IA" e gerar cada um dos documentos do pacote a partir de prompts dirigidos. O modelo Sonnet 5, recém lançado na época da execução, se mostrou suficiente para o desafio: todas as decisões técnicas já estavam fechadas na transcrição, então não havia necessidade de pesquisa externa ou de um modelo com capacidades adicionais.

## Workflow adotado

Antes de escrever qualquer prompt, li a transcrição inteira, explorei o codebase e revisei os documentos de apoio do capítulo "Design Docs com IA" disponibilizados no curso. Esse material serviu de base tanto para entender o problema quanto para estruturar os prompts usados na geração de cada documento.

A produção seguiu a ordem PRD, FDD, RFC e ADRs, atualizando o `docs/TRACKER.md` a cada documento gerado, para manter a rastreabilidade sempre em dia em vez de deixar esse trabalho para o final.

Depois de gerado o pacote inicial, não foram necessárias grandes rodadas de retrabalho. As interações seguintes foram, principalmente, prompts de validação e revisão dos documentos já gerados, conferindo cada um novamente contra a transcrição original para garantir que nada tinha sido incluído sem origem identificável ou, ao contrário, que nada relevante tinha ficado de fora.

## Prompts customizados

Abaixo estão todos os prompts usados na produção do pacote, na ordem em que foram aplicados.

Em todos os prompts, o papel atribuído à IA foi o de "programador sênior especialista em design docs". Na transcrição, os participantes da reunião não eram exatamente programadores (havia tech lead, PM, engenheiros e segurança), mas esse foi o papel escolhido porque é o que melhor descreve a postura necessária para produzir os documentos: alguém que conhece o codebase, tem capacidade técnica alta e entende o negócio o suficiente para traduzir a conversa em documentação de engenharia. Também não era papel da IA tomar decisões técnicas ou de produto, já que essas decisões já estavam fechadas na transcrição; o trabalho era registrar e estruturar o que já tinha sido decidido, não decidir por conta própria.

**Prompt usado para gerar o PRD:**

```
Você é um programador sênior especialista em design docs de produto e engenharia de software. Leia o README.md e a TRANSCRICAO.md deste repositório e extraia a feature que a equipe propôs na reunião, incluindo problema relatado, público afetado, prazos comerciais mencionados e decisões que já ficaram fechadas na conversa.

Com base nisso, escreva o PRD em docs/PRD.md. Use como referência estrutural o esqueleto deste exemplo: https://devfullcycle.notion.site/Exemplo-de-PRD-para-Feature-Cat-logo-de-eCommerce-2971423c038880ecaa19e70634fc6dc0

O documento deve conter, no mínimo: resumo e contexto, problema e motivação, público-alvo e cenários de uso, objetivos e métricas de sucesso, escopo (incluso e fora de escopo), requisitos funcionais, requisitos não funcionais, decisões e trade-offs principais, dependências, riscos e mitigação, critérios de aceitação e estratégia de testes e validação.

Cite os trechos relevantes da transcrição que embasam cada decisão ou requisito, indicando o horário e quem falou (ex.: `[09:04] Bruno`), para que qualquer revisor consiga rastrear a origem de cada afirmação até a reunião original.

Escreva em português, em tom direto e humano, como um documento que um engenheiro sênior realmente escreveria para o time. Não use jargões de inteligência artificial, não use travessões e não dramatize o texto com adjetivos vazios.

Ao final, atualize o TRACKER.md registrando o PRD como documento entregue e o que ele cobre.
```

**Prompt usado para gerar o FDD:**

```
Você é um programador sênior especialista em design docs de produto e engenharia de software. Leia o README.md, a TRANSCRICAO.md e o PRD.md já escrito neste repositório. O PRD define o que será construído e por quê; agora o objetivo é descer ao nível de desenho técnico de como será construído.

Escreva o Feature Design Document em docs/FDD.md, detalhando a solução técnica da feature descrita no PRD. Use como referência estrutural o esqueleto deste exemplo: https://devfullcycle.notion.site/Feature-Design-Doc-Rate-Limiter-29e1423c038880b69b25e34d0ce85b2f

O documento deve conter, no mínimo: contexto e motivação técnica, objetivos técnicos, escopo e exclusões, modelagem de dados, fluxos detalhados (incluindo os principais caminhos de erro), contratos públicos (assinaturas de API, headers, exemplos de payload), configuração e opções, erros e estratégia de fallback, métricas/logs/tracing, dependências e compatibilidade, integração com o sistema existente, critérios de aceite técnicos e riscos e mitigação.

Baseie as decisões técnicas nos trechos da transcrição, citando horário e autor (ex.: `[09:09] Diego`), e mantenha consistência com o que já foi definido no PRD, sem contradizer escopo ou objetivos já fechados ali.

Escreva em português, em tom direto e humano, como um documento que um engenheiro sênior realmente escreveria para o time. Não use jargões de inteligência artificial, não use travessões e não dramatize o texto com adjetivos vazios.

Ao final, atualize o TRACKER.md registrando o FDD como documento entregue e o que ele cobre.
```

**Prompt usado para auditar consistência entre transcrição e documentos:**

```
Você é um programador sênior especialista em design docs de produto e engenharia de software. Leia o README.md, a TRANSCRICAO.md, o PRD.md, o FDD.md e o TRACKER.md deste repositório.

Faça uma auditoria de consistência entre a transcrição da reunião e os documentos já escritos: identifique qualquer item que foi discutido e explicitamente descartado, adiado ou deixado como questão em aberto na reunião, mas que apareceu no PRD ou no FDD como algo que será feito nesta fase.

Para cada inconsistência encontrada, aponte o trecho da transcrição que mostra o descarte (horário e autor), o trecho do documento onde o item aparece incorretamente como incluído, e proponha a correção necessária, seja movendo o item para "fora de escopo" ou para "questões em aberto", seja removendo a afirmação indevida.

Aplique as correções diretamente nos documentos e registre no TRACKER.md o que foi revisado e ajustado nesta rodada.

Escreva em português, em tom direto e humano. Não use jargões de inteligência artificial, não use travessões e não dramatize o texto com adjetivos vazios.
```

**Prompt usado para gerar o RFC e os ADRs:**

```
Você é um programador sênior especialista em design docs de produto e engenharia de software. Leia o README.md, a TRANSCRICAO.md, o PRD.md e o FDD.md já escritos neste repositório.

Escreva o RFC da feature em docs/RFC.md. O RFC é o documento voltado à decisão e ao alinhamento entre stakeholders: menos detalhe de implementação que o FDD, mais foco no porquê da abordagem escolhida, nas alternativas descartadas e nos riscos assumidos. Baseie-se no esqueleto comumente usado em RFCs de engenharia, mas incorpore elementos de contexto de negócio equivalentes aos do PRD (prazo comercial, público afetado, impacto de não fazer).

O documento deve conter, no mínimo: cabeçalho com autor, status, data, revisores e documentos relacionados; TL;DR; contexto e problema; proposta técnica; alternativas consideradas e por que foram descartadas; questões em aberto; impacto e riscos (no código existente, operacional, de segurança e de confiabilidade) e estimativa de esforço.

Além do RFC, gere um Architecture Decision Record em docs/adrs/ para cada decisão arquitetural relevante que tenha sido fechada na reunião (por exemplo, escolha de padrão de persistência, separação de processos, estratégia de retry, mecanismo de autenticação, garantias de entrega). Nomeie os arquivos como ADR-NNN-titulo-curto.md e estruture cada um com Status, Contexto, Decisão, Alternativas Consideradas e Consequências (positivas e negativas). Referencie os ADRs a partir da seção final do RFC.

Cite os trechos da transcrição que embasam cada decisão e alternativa descartada, com horário e autor (ex.: `[09:07] Diego/Larissa`), e mantenha coerência com o que já está definido no PRD e no FDD, sem duplicar o nível de detalhe do FDD.

Escreva em português, em tom direto e humano, como um documento que um engenheiro sênior realmente escreveria para o time. Não use jargões de inteligência artificial, não use travessões e não dramatize o texto com adjetivos vazios.

Ao final, atualize o TRACKER.md registrando o RFC e os ADRs como documentos entregues e o que cada um cobre.
```

## Iterações e ajustes

O fluxo de produção foi direto, sem grandes retrabalhos. Depois de gerado cada documento com o prompt específico, a principal iteração foi o prompt de auditoria acima: pedi para a IA reler a transcrição e os documentos já escritos e apontar qualquer item que tivesse sido discutido e explicitamente descartado ou adiado na reunião, mas que tivesse entrado no PRD ou no FDD como algo incluído nesta fase. Esse prompt foi o principal mecanismo usado para corrigir inconsistências antes de fechar o pacote.

Fora essa auditoria, os ajustes foram pontuais: validar se as citações da transcrição batiam com o horário e o falante corretos, e conferir se os caminhos de arquivo citados no FDD e nos ADRs realmente existiam no código.

## Como navegar a entrega

Ordem sugerida de leitura:

1. [docs/PRD.md](docs/PRD.md): o que é a feature, para quem e por quê.
2. [docs/RFC.md](docs/RFC.md): a proposta técnica submetida à revisão, com alternativas e questões em aberto.
3. [docs/adrs/](docs/adrs/): cada decisão arquitetural fechada, isoladamente.
4. [docs/FDD.md](docs/FDD.md): o detalhamento de implementação, fluxos e contratos.
5. [docs/TRACKER.md](docs/TRACKER.md): rastreabilidade de cada item até a transcrição ou o código.
