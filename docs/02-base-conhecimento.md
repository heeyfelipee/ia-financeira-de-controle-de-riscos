# Base de Conhecimento

## Dados Utilizados

| Arquivo | Formato | Para que serve no CoopGuard? |
|---------|---------|---------------------|
| `historico_atendimento.csv` | CSV | Contextualizar interações anteriores e identificar padrões de fraudes ou irregularidades. |
| `perfil_investidor.json` | JSON | Personalizar recomendações financeiras de acordo com o perfil do associado, evitando descumprimentos de normas. |
| `produtos_financeiros.json` | JSON | Sugerir produtos adequados ao perfil do cliente, considerando risco de mercado e conformidade regulatória. |
| `transacoes.csv` | CSV | Analisar padrões de gastos e movimentações financeiras para detectar risco operacional, liquidez e possíveis irregularidades. |

---

## Adaptações nos Dados

> Você modificou ou expandiu os dados mockados? Descreva aqui.

Sim. Os conjuntos de dados originais foram ampliados e enriquecidos para refletir cenários reais que ocorrem em cooperativas financeiras, com foco em fraudes, irregularidades, risco operacional, risco de liquidez, risco de mercado e risco de conformidade, além de incluir mecanismos e metadados que permitem ao agente IA prevenir e antecipar descumprimentos de normas. As modificações incluem: inclusão de eventos de alerta (AML/fraude), estados de investigação, entradas que disparam controles automáticos, campos operacionais (logs, IDs de alerta), cenários simulados, controles recomendados e notas de conformidade. Abaixo descrevo, arquivo a arquivo, o que foi alterado e expandido.

### historico_atendimento.csv — O que foi modificado e por quê.
- [x] Novas linhas de eventos reais: incluímos casos como suspeita de fraude, irregularidade em empréstimo, alerta AML e falha operacional, além das interações rotineiras (CDB, Tesouro, atualização cadastral).
- [x] Campo resolvido com status realistas: valores sim e pendente para indicar casos que exigem investigação contínua.
- [x] Resumo detalhado: cada registro ganhou descrições que apontam ações tomadas (bloqueio de transação, investigação, KYC atualizado, reunião agendada).
- [x] Datas e canais variados: entradas distribuídas por chat, telefone e email para simular diferentes vetores de entrada de risco.
- [x] Foco em gatilhos de controle: registros que explicitam quando um evento acionou investigação AML, bloqueio de cartão, ou encaminhamento para compliance.

### Por que foi feito?
- [x] Permitir ao agente IA identificar padrões de fraude e irregularidade a partir do histórico de atendimento.
- [x] Fornecer sinais para checagens automatizadas (ex.: quando marcar pendente e acionar workflows de investigação).
- [x] Simular a interação entre operações, TI e compliance para treinar respostas e alertas.

---

### perfil_investidor.json — O que foi modificado e por quê.
- [x] Campos operacionais e de segurança: contato_consentimento, perfil_operacional (limite_transferencia_diaria, preferencia_canal, 2FA, última atualização cadastral).
- [x] Histórico de alertas: array historico_alertas com IDs, datas, tipos (fraude, irregularidade documental, transação atípica), descrições, status e ações tomadas.
- [x] Indicadores de risco: indicadores_risco com níveis qualitativos (inadimplencia_potencial, exposicao_mercado, risco_liquidez, risco_operacional, conformidade).
- [x] Checagens automatizadas: lista checagens_automatizadas (KYC, detecção de transações atípicas, checagem anti-alucinação) com frequência e estado.
- [x] Medidas preventivas e recomendações: ações operacionais, de conformidade, educacionais e financeiras vinculadas ao perfil.
- [x] Cenários simulados: casos práticos (tentativa de fraude, pedido de empréstimo com documentos falsos, queda de liquidez) com resultados esperados.
- [x] Notas de compliance e ultima_atualizacao para rastreabilidade.

### Por que foi feito?
- [x] Transformar o perfil em um ativo operacional que o agente pode usar para decisões de suitability, bloqueios automáticos e priorização de investigações.
- [x] Permitir simulações e testes de workflows (ex.: resposta a alerta AML, acionamento de 2FA, suspensão de análise de crédito).
- [x] Facilitar auditoria e rastreabilidade das ações tomadas sobre o associado.

---

### produtos_financeiros.json — O que foi modificado e por quê.
- [x] Descrição ampliada de cada produto (objetivo, características, limitações).
- [x] Riscos associados explicitados por produto (risco de mercado, crédito, liquidez, operacional, fraude).
- [x] Controles recomendados para cada produto (limites de resgate, 2FA, stress tests, checagem de suitability).
- [x] Requisitos de conformidade (KYC, documentação de origem de recursos, prospecto atualizado).
- [x] Notas operacionais com recomendações práticas (bloqueios para perfis inadequados, monitoramento de saques em massa).
- [x] Inclusão de um produto adicional (Conta Remunerada Cooperativa) com riscos e controles específicos.

### Por que foi feito?
- [x] Permitir ao agente avaliar adequação de produtos ao perfil do associado e evitar recomendações inadequadas.
- [x] Fornecer gatilhos para controles automáticos (ex.: bloquear oferta para perfis conservadores; exigir documentação para aportes atípicos).
- [x] Ajudar a equipe de compliance a auditar ofertas e comunicações comerciais.

---

### transacoes.csv — O que foi modificado e por quê.
- [x] Linhas de transações atípicas e eventos de risco: transferências internacionais suspeitas, estornos por transação não reconhecida, resgates em massa de CDB (alerta de liquidez), saques concentrados, transferências entre contas vinculadas atípicas, alerta AML com valores elevados.
- [x] Categorias de risco: além de categorias pessoais (alimentacao, moradia), foram incluídas categorias como fraude, liquidez, irregularidade, risco_operacional, conformidade, mercado.
- [x] Eventos de correção e reversão: estornos, depósitos retidos, reversões por erro operacional, retornos de transferências para contas bloqueadas.
- [x] Entradas que disparam ações: por exemplo, resgate em massa com valor e tag liquidez para acionar stress tests e comunicação com gestores.
- [x] Datas e sequência temporal para simular evolução de um caso (fraude detectada → estorno → investigação → reaparecimento de padrão).

### Por que foi feito?
- [x] Fornecer ao agente IA dados transacionais ricos para detecção de padrões anômalos, regras AML e gatilhos de liquidez.
- [x] Simular cenários que exigem coordenação entre controles internos, compliance e TI (ex.: saques em massa, transferências atípicas).
- [x] Permitir testes de modelos de detecção de fraude e de regras de negócio (limites, bloqueios, solicitações de documentação).

---

## Estratégia de Integração

### Como os dados são carregados?
> Descreva como seu agente acessa a base de conhecimento.

Os arquivos JSON e CSV são ingeridos por um pipeline controlado que transforma os dados brutos em contexto utilizável pelo agente IA. O objetivo é garantir que o LLM receba apenas informações validadas, auditáveis e compatíveis com controles de segurança e conformidade.

### 1. Ingestão inicial e escopo da sessão.
- Carregamento no início da sessão: ao iniciar uma sessão de análise, os arquivos relevantes (historico_atendimento.csv, perfil_investidor.json, produtos_financeiros.json, transacoes.csv) são carregados no ambiente de processamento.
- Inclusão no contexto do prompt: trechos selecionados e sumarizados são incorporados ao contexto enviado ao LLM (RAG — Retrieval Augmented Generation), respeitando limites de tokens.
- Refresh on change: se um arquivo for atualizado, o pipeline detecta a alteração e recarrega apenas os blocos afetados.

---

### 2. Pré‑processamento e validação.
- Validação de esquema: cada arquivo passa por checagem de schema (tipos, campos obrigatórios, formatos de data e valores numéricos). Registros inválidos são isolados e logados.
- Normalização: padronização de nomes de campos, categorias e formatos (ex.: datas ISO, valores numéricos com ponto decimal).
- Enriquecimento: adição de metadados úteis ao agente, como id_alerta, status_investigacao, canal_origem, e tags de risco (fraude, liquidez, conformidade).
- Anonimização e consentimento: remoção ou mascaramento de PII quando necessário; verificação de consentimento antes de usar dados sensíveis.

---

### 3. Indexação e recuperação.
- Chunking: arquivos grandes são divididos em blocos semânticos (ex.: conversas por atendimento, transações por período) para facilitar recuperação eficiente.
- Vetorização: blocos relevantes são convertidos em embeddings para busca semântica; índices vetoriais permitem recuperar rapidamente contextos similares.
- Regras de priorização: alertas com status pendente ou investigacao têm prioridade na recuperação; eventos de alto valor (ex.: alerta AML) são destacados.

---

### 4. Uso em tempo real com Whisper Groq Cloud e gTTS.
- Pipeline de áudio: áudio do usuário → OpenAI Whisper (STT) → texto pré‑processado → enriquecimento com dados do perfil e histórico → envio ao Groq Cloud (Llama 3.3 70B) para geração de resposta.
- Validação de saída: antes de apresentar a resposta, o texto gerado é checado contra a base de conhecimento e regras de conformidade; se aprovado, é convertido em voz via gTTS quando solicitado.
- Contexto dinâmico: transcrições recentes e eventos críticos podem ser injetados no contexto em tempo real para decisões imediatas (bloqueio, solicitação de documentação).

---

### 5. Controles de segurança e anti‑alucinação.
- Cross‑check automático: cada afirmação do LLM é confrontada com registros primários (CSV/JSON) e com regras de negócio; respostas sem respaldo são marcadas como “não confirmado” ou redirecionadas.
- Gatilhos automáticos: padrões detectados (saques em massa, transferências atípicas, documentos inconsistentes) disparam workflows: bloqueio temporário, abertura de ticket, notificação a compliance.
- Logs e auditoria: todas as consultas, trechos de contexto usados e decisões automatizadas são registrados para auditoria e revisão humana.

---

### 6. Operação, monitoramento e manutenção.
- Cache e performance: resultados de consultas frequentes são cacheados com TTL configurável; índices vetoriais são reindexados periodicamente.
- Métricas: monitoramento de latência, taxa de falsos positivos/negativos em alertas e cobertura de dados; relatórios periódicos para ajustes de regras e modelos.
- Fallbacks: em caso de inconsistência ou falta de dados, o agente admite limitação, solicita documentação adicional e encaminha para análise humana.

---

### Em dados práticos, cada uma destas 6 (seis) operações são:
Os arquivos são carregados no início da sessão, validados, normalizados, indexados e transformados em blocos recuperáveis. O LLM recebe apenas contexto selecionado e verificado; saídas são validadas contra a base antes de serem apresentadas ou sintetizadas em voz. Tudo isso com logs, controles de acesso e gatilhos automáticos para prevenir fraudes, irregularidades e riscos operacionais, de liquidez, mercado e conformidade.

---

### Como os dados são usados no prompt?
> Os dados vão no system prompt? São consultados dinamicamente?

Os dados não são colocados integralmente no system prompt. O system prompt contém apenas instruções de comportamento, segurança e limites do agente. Os arquivos JSON/CSV são consultados dinamicamente por um pipeline de recuperação (RAG) e trechos verificados são injetados no prompt de forma controlada quando necessário.

---

### Onde os dados residem e como são acessados.
- Armazenamento: arquivos originais ficam em repositório seguro ou storage interno; índices vetoriais e caches são mantidos para busca semântica.
- Acesso: o agente recupera blocos relevantes por consulta semântica ou por regras de priorização (ex.: alertas pendente, transações atípicas).
- Escopo: apenas trechos sumarizados e comprováveis são enviados ao LLM para respeitar limites de tokens e privacidade.

---

### Composição do prompt e divisão de responsabilidades.
- System prompt: define tom, regras de segurança, políticas anti‑alucinação, e instruções para checagem de fontes. Não contém dados sensíveis.
- User prompt: contém a pergunta do usuário e, quando aplicável, trechos de contexto injetados (ex.: resumo do perfil_investidor, alertas recentes, transação específica).
- Assistant prompt (interna): inclui instruções de pós‑processamento, validação e ações a executar (ex.: abrir ticket, solicitar documentação).

---

### Processo de recuperação e injeção dinâmica.
- Busca semântica: converte consulta em embedding e recupera blocos relevantes (chunking).
- Filtragem e validação: valida esquema, checa consentimento e remove PII quando necessário.
- Sumarização: transforma blocos longos em resumos factuais curtos antes de injetar no prompt.
- Injeção controlada: insere apenas os trechos necessários com etiquetas claras (ex.: CONTEXTO_PERFIL, ALERTA_TRANSACAO) e limites de tokens.
- Prioridade: eventos com status = pendente ou tags AML/fraude têm prioridade na injeção.

---

### Anti‑alucinação, rastreabilidade e fallback.
- Cross‑check: cada afirmação do LLM é comparada automaticamente com registros primários; respostas sem respaldo são marcadas como não confirmadas.
- Rastreabilidade: o prompt inclui metadados sobre quais arquivos e blocos foram usados; tudo é logado para auditoria.
- Fallback: se não houver evidência suficiente, o agente admite limitação, solicita documentação e encaminha para revisão humana.

---

### Exemplo prático de trecho injetado no prompt
- [x] CONTEXTO_PERFIL: perfil_investidor.json resumo — nome João Silva; perfil moderado; alertas recentes: ALRT-2025-09-18-001 (fraude, investigação).
- [x] CONTEXTO_TRANSACAO: transacoes.csv resumo — 2025-10-22 transferência internacional suspeita R$3200; estorno 2025-10-23.
- [x] INSTRUÇÃO: "Use apenas informações acima para responder; valide qualquer afirmação com os registros; se incerto, peça documentação."

---

### Dados finais:
Os dados são consultados dinamicamente, validados e resumidos antes de serem injetados no prompt. O system prompt define regras e limites; o contexto factual vem de recuperação controlada, com checagens automáticas para prevenir alucinações e garantir auditabilidade.

---

## Exemplo de Contexto Montado

> Mostre um exemplo de como os dados são formatados para o agente.

```
DADOS_DO_CLIENTE
- Nome: Murilo Duarte
- Idade: 25
- Perfil: Moderado
- Renda mensal: R$ 5.000,00
- Saldo disponível: R$ 5.000,00
- Última atualização cadastral: 2025-09-30
- Autenticação 2FA: ativa

ULTIMAS_TRANSACOES (últimos 30 dias)
- 2025-10-01: Salário - Entrada - R$ 5.000,00
- 2025-10-22: Transferência internacional suspeita - Saída - R$ 3.200,00
- 2025-10-23: Estorno por transação não reconhecida - Entrada - R$ 3.200,00
- 2025-10-25: Resgate CDB em massa (alerta liquidez) - Saída - R$ 15.000,00
- 2025-10-28: Saque em espécie concentrado (3 saques em 1h) - Saída - R$ 6.000,00

ALERTAS_RELEVANTES
- ALRT-2025-09-18-001: Débito não reconhecido; cartão bloqueado; investigação em andamento.
- ALRT-2025-10-05-002: Pedido de empréstimo com documentos inconsistentes; caso encaminhado para compliance.
- ALRT-2025-11-28-003: Transação atípica entre contas vinculadas; documentação solicitada; monitoramento ativo.

INDICADORES_DE_RISCO
- Inadimplência potencial: baixa
- Exposição a risco de mercado: moderada
- Risco de liquidez: baixo a moderado (resgates recentes sinalizam atenção)
- Risco operacional: moderado (falhas de conciliação registradas)
- Conformidade: em monitoramento (casos AML pendentes)

PRODUTOS_SUGERIDOS (adequação)
- Tesouro Selic — indicado para reserva de emergência; exigir KYC atualizado.
- CDB Liquidez Diária — indicado com limite de resgate diário e 2FA para resgates acima de R$ 5.000.
- Bloquear ofertas de fundos de ações até regularização de alertas AML.

ACOES_RECOMENDADAS (prioridade)
- Prioridade alta: confirmar origem de recursos do resgate CDB e do aporte extraordinário; manter bloqueios temporários se documentação ausente.
- Prioridade média: reforçar 2FA e limites de transferência; enviar material educativo sobre prevenção a fraudes.
- Prioridade baixa: revisar suitability antes de oferecer produtos com maior risco de mercado.

METADADOS_USADOS
- Fontes: historico_atendimento.csv; perfil_investidor.json; transacoes.csv; produtos_financeiros.json
- Contexto injetado: resumo do perfil + últimas 5 transações + 3 alertas prioritários
- Nota: usar apenas informações acima para responder; validar qualquer afirmação com registros primários; se incerto, solicitar documentação.

```

---

### Observações práticas:
- Mantenha o bloco abaixo de 300–600 tokens para garantir eficiência no LLM.
- Priorize sempre alertas com status = pendente ao montar o contexto dinâmico.
- Inclua metadados de origem para rastreabilidade e auditoria.
