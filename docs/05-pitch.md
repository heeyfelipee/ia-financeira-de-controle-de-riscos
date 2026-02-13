# Pitch (3 minutos)

> [!TIP]
> Você pode usar alguns slides pra apoiar no seu Pitch e mostrar sua solução na prática.
 
## Roteiro Sugerido

### 1. O Problema (30 seg)
> Qual dor do cliente você resolve?

Cooperativas perdem tempo e dinheiro porque fraudes e irregularidades são detectadas tardiamente, exigem investigação manual e sobrecarregam equipes de compliance.

“Cooperativas enfrentam um problema simples e caro: fraudes e irregularidades são detectadas tarde demais e exigem investigações manuais que consomem tempo e recursos. Isso gera perdas financeiras, risco regulatório e desgaste da confiança dos associados. Soluções corporativas existentes são caras e pouco adaptadas ao dia a dia das cooperativas, deixando equipes sobrecarregadas e processos lentos.”

### 2. A Solução (1 min)
> Como seu agente resolve esse problema?

CoopGuard é um agente conversacional e motor de análise que transforma dados operacionais (transações, histórico de atendimento, catálogo de produtos) em detecções acionáveis, priorização de casos e recomendações auditáveis — tudo em tempo real e integrado ao fluxo de trabalho da cooperativa.

“CoopGuard integra seus dados operacionais, aplica detecção automatizada e usa um assistente conversacional para transformar alertas em ações. Ele sanitiza os dados, gera alertas classificados por risco, prioriza os casos que realmente importam e entrega recomendações práticas com evidências e trilha de auditoria. Tudo isso roda em background para não travar a operação e grava logs para compliance. Em resumo: menos tempo perdido em investigações, menos perdas por fraude e decisões mais rápidas e rastreáveis para sua cooperativa.”

### 3. Demonstração (1 min)
> Mostre o agente funcionando (pode ser gravação de tela)

**Estrutura (60s total)**
- 0–05s — Abertura rápida
```
Visual: tela do app (cabeçalho “CoopGuard — O Guardião da sua Cooperativa”).
Narração (2–3s): “Vou mostrar o CoopGuard em ação com um caso realista.”
```
- 05–20s — Contexto do cliente
```
Visual: expander “Ver contexto” aberto; destaque nome, perfil, patrimônio e resumo de transações.
Narração (8–10s): “Aqui temos o cliente Murilo, perfil moderado, patrimônio R$15.000 e histórico de transações — o contexto que o agente usa para analisar riscos.”
On‑screen callout: marque com retângulo o bloco “TRANSAÇÕES RECENTES”.
```
- 20–35s — Envio da pergunta
```
Visual: cursor no campo de chat; digitar (ou colar) a pergunta:
```
> Quais alertas ativos para Murilo? Resuma por tipo, data e status.
- Narração (3–4s): “Agora pergunto: ‘Quais alertas ativos para Murilo?’”
- Ação: pressione Enter; mostre indicador de envio (mensagem “Enviado para processamento em background”).

- 35–50s — Resposta do agente
```
Visual: resposta do modelo aparece; destaque trechos: tipo do alerta, data, valor e status; role a resposta se for longa.
Narração (8–10s): “O agente retorna um resumo com tipos de alerta (fraude, AML, risco operacional), datas, valores e status — e sugere próximos passos operacionais.”
On‑screen callout: destaque a linha “Audit: origem=modelo …; timestamp=…”
```
- 50–60s — Log de auditoria e encerramento
```
Visual: abrir coopguard.log (tail) ou expander “Executor internals”; mostre a entrada com timestamp e preview do prompt/resposta.
Narração (6–8s): “Cada resposta tem trilha de auditoria: prompt, timestamp e origem do modelo — essencial para compliance. Vamos conversar sobre piloto?”
Final: tela com contato rápido (nome e e‑mail) por 1–2s.
```
---
### Texto para narração:
> “Vou mostrar o CoopGuard em ação com um caso realista.
Aqui temos o cliente Murilo, perfil moderado, patrimônio quinze mil reais e um histórico de transações — esse é o contexto que o agente usa.
Pergunto: ‘Quais alertas ativos para Murilo? Resuma por tipo, data e status.’
O agente retorna um resumo com tipos de alerta, datas, valores, status e próximos passos operacionais.
Cada resposta inclui uma trilha de auditoria com prompt, timestamp e origem do modelo — essencial para compliance. Obrigado, vamos conversar sobre um piloto?”

### 4. Diferencial e Impacto (30 seg)
> Por que essa solução é inovadora e qual é o impacto dela na sociedade?

**Diferencial:**
- CoopGuard combina detecção automatizada com um assistente conversacional auditável, aplicando sanitização de dados, trilha de auditoria por resposta e prioritização inteligente para reduzir falsos positivos e focar analistas nos casos que realmente importam. A execução assíncrona mantém a operação fluida e a integração via API/CSV torna a solução acessível para cooperativas com recursos limitados.

**Impacto social:**
- Ao acelerar a detecção e investigação de fraudes, CoopGuard reduz perdas financeiras, diminui o risco regulatório e protege a confiança dos associados. Isso libera tempo dos profissionais para tarefas de maior valor, melhora a resiliência das instituições locais e amplia o acesso a ferramentas de compliance antes restritas a grandes players.

> “O diferencial do CoopGuard é entregar respostas auditáveis, seguras e priorizadas: sanitizamos dados sensíveis, registramos prompt, resposta e timestamp, e priorizamos casos por risco real. Isso reduz perdas financeiras, acelera investigações e permite que equipes foquem no que importa. Em suma: mais eficiência operacional, menos risco regulatório e maior proteção para os associados.”

---

## Checklist do Pitch

- [x] Duração máxima de 3 minutos
- [x] Problema claramente definido
- [x] Solução demonstrada na prática
- [x] Diferencial explicado
- [x] Áudio e vídeo com boa qualidade

---
