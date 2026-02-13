# Avaliação e Métricas

## Como Avaliar seu Agente

A avaliação pode ser feita de duas formas complementares:

1. **Testes estruturados:** Você define perguntas e respostas esperadas;
2. **Feedback real:** Pessoas testam o agente e dão notas.

---

## Métricas de Qualidade

| Métrica | O que avalia | Exemplo de teste |
|---------|--------------|------------------|
| **Assertividade** | O agente respondeu o que foi perguntado? | Perguntar o saldo e receber o valor correto |
| **Segurança** | O agente evitou inventar informações? | Perguntar algo fora do contexto e ele admitir que não sabe |
| **Coerência** | A resposta faz sentido para o perfil do cliente? | Sugerir investimento conservador para cliente conservador |

> [!TIP]
> Peça para 3-5 pessoas (amigos, família, colegas) testarem seu agente e avaliarem cada métrica com notas de 1 a 5. Isso torna suas métricas mais confiáveis! Caso use os arquivos da pasta `data`, lembre-se de contextualizar os participantes sobre o **cliente fictício** representado nesses dados.

---

## Exemplos de Cenários de Teste

Crie testes simples para validar seu agente:

### Teste 1: Consulta de gastos
- **Pergunta:** "Quanto gastei com alimentação?"
- **Resposta esperada:** Valor baseado no `transacoes.csv`
- **Resultado:** [x] Correto  [ ] Incorreto

### Teste 2: Recomendação de produto
- **Pergunta:** "Qual investimento você recomenda para mim?"
- **Resposta esperada:** Produto compatível com o perfil do cliente
- **Resultado:** [x] Correto  [ ] Incorreto

### Teste 3: Pergunta fora do escopo
- **Pergunta:** "Qual a previsão do tempo?"
- **Resposta esperada:** Agente informa que só trata de finanças
- **Resultado:** [x] Correto  [ ] Incorreto

### Teste 4: Informação inexistente
- **Pergunta:** "Quanto rende o produto XYZ?"
- **Resposta esperada:** Agente admite não ter essa informação
- **Resultado:** [x] Correto  [ ] Incorreto

---

## Resultados

Após os testes, registre suas conclusões:

**O que funcionou bem:**
- Execução em background: uso de ThreadPoolExecutor em st.session_state evita bloqueio da UI e permite envio assíncrono de prompts.
- Integração com Ollama: chamadas diretas retornaram status 200 e a chave response foi corretamente parseada.
- Polling e consumo do resultado: poll_future() captura o future e preenche last_result quando concluído.
- Observabilidade: logs no terminal e arquivo (coopguard.log) com RotatingFileHandler permitem auditoria e investigação.
- Robustez do prompt: truncamento do contexto e limites em previews reduzem tamanho do prompt e latência.
- Fallbacks e compatibilidade: suporte a st.chat_input/st.text_input e toggle para forçar rerun funcionam em diferentes versões do Streamlit.

**O que pode melhorar:**
- Timeouts e retries: aumentar REQUEST_TIMEOUT e MAX_RETRIES para cenários de alta latência; atualmente pode falhar em picos.
- Sanitização de dados sensíveis: remover/mascarar CPF, senhas e dados pessoais antes de enviar ao modelo.
- Endpoint programático: criar uma rota API (FastAPI/Flask) para testes automatizados (run_tests.py) em vez de depender só da UI.
- Gestão de concorrência: monitorar e ajustar EXECUTOR_MAX_WORKERS conforme CPU/RAM; evitar threads órfãs.
- UI/UX: melhorar feedback ao usuário enquanto o future está running (barra de progresso, estimativa de latência).
- Testes automatizados e CI: agendar run_tests.py, armazenar resultados históricos e alertar quando score cair abaixo do threshold.
- Segurança e exposição: proteger o app com autenticação se exposto em rede; não expor OLLAMA_URL publicamente.
- Remoção de debug em produção: garantir COOPGUARD_DEBUG=0 e remover prints sensíveis do log.

**Prioridade imediata**
- Aumentar tolerância: definir REQUEST_TIMEOUT = 180 e MAX_RETRIES = 4.
- Ativar logs em tempo real: rodar Get-Content .\coopguard.log -Wait -Tail 80 enquanto testa para capturar erros.
- Sanitizar prompt: aplicar função que remove/mascara campos sensíveis antes de montar prompt.

**Melhorias de médio prazo (planejar 1–2 semanas)**
- Criar endpoint API /api/query para integração com run_tests.py e runners CI.
- Automatizar testes: agendar run_tests.py diário e armazenar CSVs em results/ com histórico.
- Monitoramento e alertas: integrar healthchecks e alertas (Slack/e‑mail) quando Score < threshold.
- Hardening: adicionar autenticação, TLS e revisão de exposição do OLLAMA_URL.
- Dashboard de métricas: gerar relatório periódico (latência, assertividade, taxa de erro).
  
---

## Métricas Avançadas (Opcional)

### Estrutura do repositório:
```
coopguard/
├─ src/
│  ├─ app.py
│  ├─ api_query.py
│  ├─ debug_ollama.py
│  ├─ run_tests.py
│  ├─ evaluate_results.py
│  ├─ report.py
│  └─ tail_log.ps1
├─ data/
│  ├─ perfil_investidor.json
│  ├─ transacoes.csv
│  └─ historico_atendimento.csv
├─ results/                # gerado pelos testes
├─ .github/
│  └─ workflows/
│     └─ ci.yml
├─ .gitignore
├─ requirements.txt
├─ Dockerfile
└─ README.md
```
---

**src/app.py (production-ready)**
```
# src/app.py — CoopGuard (production-ready)
import os
import json
import time
import traceback
import logging
from logging.handlers import RotatingFileHandler
from datetime import datetime
import concurrent.futures
import atexit

import pandas as pd
import requests
import streamlit as st

# CONFIG via ENV
BASE_DIR = os.path.dirname(os.path.abspath(__file__))
OLLAMA_URL = os.environ.get("OLLAMA_URL", "http://localhost:11434/api/generate")
MODELO = os.environ.get("MODELO", "gpt-oss:120b-cloud")
REQUEST_TIMEOUT = int(os.environ.get("REQUEST_TIMEOUT", "120"))
MAX_RETRIES = int(os.environ.get("MAX_RETRIES", "3"))
EXECUTOR_MAX_WORKERS = int(os.environ.get("EXECUTOR_MAX_WORKERS", "2"))
LOG_FILE = os.environ.get("COOPGUARD_LOG", os.path.join(BASE_DIR, "..", "coopguard.log"))
COOPGUARD_DEBUG = os.environ.get("COOPGUARD_DEBUG", "0") == "1"

# Logging rotativo
logger = logging.getLogger("coopguard")
if not logger.handlers:
    logger.setLevel(logging.INFO)
    handler = RotatingFileHandler(LOG_FILE, maxBytes=5_000_000, backupCount=5, encoding="utf-8")
    fmt = logging.Formatter("%(asctime)s %(levelname)s %(message)s")
    handler.setFormatter(fmt)
    logger.addHandler(handler)
    stream_handler = logging.StreamHandler()
    stream_handler.setFormatter(fmt)
    logger.addHandler(stream_handler)

def log(msg, level="info"):
    if level == "debug":
        logger.debug(msg)
    elif level == "warning":
        logger.warning(msg)
    elif level == "error":
        logger.error(msg)
    else:
        logger.info(msg)

def now_ts():
    return datetime.utcnow().isoformat() + "Z"

# DATA DIR detection
candidate_paths = [
    os.path.join(BASE_DIR, "data"),
    os.path.abspath(os.path.join(BASE_DIR, "..", "data")),
    os.path.abspath(os.path.join(BASE_DIR, "..", "..", "data")),
]
DATA_DIR = next((p for p in candidate_paths if os.path.isdir(p)), candidate_paths[0])

def data_path(*parts):
    return os.path.join(DATA_DIR, *parts)

# I/O helpers
def load_json(path):
    if not os.path.exists(path):
        raise FileNotFoundError(f"Arquivo não encontrado: {path}")
    with open(path, "r", encoding="utf-8") as f:
        return json.load(f)

def safe_load_json(path, default):
    try:
        return load_json(path)
    except Exception:
        return default

def safe_read_csv_or_empty(path):
    try:
        return pd.read_csv(path)
    except Exception:
        return pd.DataFrame()

# Session state init
if "last_future" not in st.session_state:
    st.session_state["last_future"] = None
if "last_result" not in st.session_state:
    st.session_state["last_result"] = None
if "last_request_ts" not in st.session_state:
    st.session_state["last_request_ts"] = None
if "last_sent_text" not in st.session_state:
    st.session_state["last_sent_text"] = None
if "_coopguard_force_rerun" not in st.session_state:
    st.session_state["_coopguard_force_rerun"] = False
if "executor" not in st.session_state:
    st.session_state["executor"] = concurrent.futures.ThreadPoolExecutor(max_workers=EXECUTOR_MAX_WORKERS)

# Load data
perfil = safe_load_json(data_path('perfil_investidor.json'), {
    "nome": "N/D", "idade": 0, "perfil_investidor": "N/D",
    "objetivo_principal": "N/D", "patrimonio_total": 0, "reserva_emergencia_atual": 0
})
transacoes = safe_read_csv_or_empty(data_path('transacoes.csv'))
historico = safe_read_csv_or_empty(data_path('historico_atendimento.csv'))
produtos = safe_load_json(data_path('produtos_financeiros.json'), [])

# Context build (truncated)
def truncate_text(s, max_chars=1000):
    if not s:
        return ""
    s = str(s)
    return s if len(s) <= max_chars else s[:max_chars] + "\n... (truncado)"

def df_preview(df, n=20, max_chars=1200):
    if df is None or df.empty:
        return "(sem registros)"
    preview = df.tail(n).to_string(index=False)
    return truncate_text(preview, max_chars=max_chars)

contexto = f"""
CLIENTE: {truncate_text(perfil.get('nome'),200)}, {perfil.get('idade')} anos, perfil {truncate_text(perfil.get('perfil_investidor'),50)}
OBJETIVO: {truncate_text(perfil.get('objetivo_principal'),200)}
PATRIMÔNIO: R$ {perfil.get('patrimonio_total')} | RESERVA: R$ {perfil.get('reserva_emergencia_atual')}

TRANSAÇÕES RECENTES (ult. 20):
{df_preview(transacoes, 20)}

ATENDIMENTOS ANTERIORES (ult. 20):
{df_preview(historico, 20)}

PRODUTOS DISPONÍVEIS (amostra):
{truncate_text(json.dumps(produtos if isinstance(produtos, list) else [], ensure_ascii=False), 1200)}
"""

SYSTEM_PROMPT = (
    "Você é um agente financeiro inteligente especializado em prevenção de fraudes, conformidade e gestão de riscos "
    "em cooperativas financeiras. Priorize segurança, auditabilidade e baseie-se em registros primários."
)

# Parse model response
def parse_model_response(data):
    if not isinstance(data, dict):
        return str(data)
    for key in ("response", "text", "result", "output"):
        if key in data:
            return data[key]
    if "choices" in data and isinstance(data["choices"], list) and data["choices"]:
        first = data["choices"][0]
        if isinstance(first, dict):
            return first.get("text") or first.get("message") or str(first)
        return str(first)
    if "results" in data and isinstance(data["results"], list) and data["results"]:
        return str(data["results"][0])
    return json.dumps(data, ensure_ascii=False)

# Call Ollama with retries
def call_ollama_sync(prompt):
    last_exception = None
    for attempt in range(1, MAX_RETRIES + 1):
        try:
            r = requests.post(
                OLLAMA_URL,
                json={"model": MODELO, "prompt": prompt, "stream": False},
                timeout=REQUEST_TIMEOUT
            )
            r.raise_for_status()
            break
        except requests.Timeout as e:
            last_exception = e
            log(f"Timeout na tentativa {attempt}: {e}", level="warning")
        except requests.RequestException as e:
            last_exception = e
            log(f"RequestException na tentativa {attempt}: {e}", level="warning")
        if attempt < MAX_RETRIES:
            backoff = 2 * attempt
            log(f"Aguardando {backoff}s antes da próxima tentativa", level="debug")
            time.sleep(backoff)
    else:
        return f"Erro na requisição ao servidor de modelo após {MAX_RETRIES} tentativas: {last_exception}"

    try:
        raw = r.text
        log(f"Ollama status: {r.status_code}; text_preview: {raw[:2000]}", level="debug")
    except Exception as e:
        log(f"Erro ao logar resposta: {e}", level="error")

    try:
        data = r.json()
    except ValueError:
        log("Resposta inválida do servidor (não JSON)", level="error")
        return f"Resposta inválida do servidor (não JSON): {r.text}"
    return parse_model_response(data)

# Background submit/poll
def perguntar_async(msg):
    prompt = f"{SYSTEM_PROMPT}\n\nCONTEXTO DO CLIENTE:\n{contexto}\n\nPergunta: {msg}"
    st.session_state["last_request_ts"] = now_ts()
    log(f"Submetendo future (prompt {len(prompt)} chars)")
    future = st.session_state["executor"].submit(call_ollama_sync, prompt)
    st.session_state["last_future"] = future

def poll_future():
    fut = st.session_state.get("last_future")
    if not fut:
        return
    try:
        if fut.done():
            try:
                result = fut.result()
                st.session_state["last_result"] = result
                log("Future concluído; resultado armazenado.")
            except Exception as e:
                st.session_state["last_result"] = f"Erro interno no processamento do modelo: {e}"
                log(f"Exceção ao obter result do future: {e}", level="error")
                log(traceback.format_exc(), level="debug")
            finally:
                st.session_state["last_future"] = None
    except Exception as e:
        st.session_state["last_result"] = f"Erro ao checar future: {e}\nVeja coopguard.log para detalhes."
        log(f"Erro ao checar future: {e}", level="error")
        log(traceback.format_exc(), level="debug")
        st.session_state["last_future"] = None

# Shutdown executor on exit
def _shutdown_executor():
    try:
        ex = st.session_state.get("executor")
        if ex:
            ex.shutdown(wait=False)
            log("Executor shutdown invoked.")
    except Exception:
        pass
atexit.register(_shutdown_executor)

# Streamlit UI
st.set_page_config(page_title="CoopGuard", layout="wide")
st.title("CoopGuard — O Guardião da sua Cooperativa")

st.markdown(f"**Contexto carregado de:** `{DATA_DIR}`")
with st.expander("Ver contexto (apenas para revisão interna)"):
    st.code(contexto[:1000] + ("\n\n... (contexto truncado)" if len(contexto) > 1000 else ""), language="text")

def ollama_health():
    try:
        r = requests.get(OLLAMA_URL.replace("/api/generate", "/"), timeout=3)
        return f"OK ({r.status_code})"
    except Exception as e:
        return f"Indisponível: {e}"

st.sidebar.markdown("**Status**")
st.sidebar.write("Ollama:", ollama_health())
st.sidebar.write("Modelo:", MODELO)
st.sidebar.write("Última requisição:", st.session_state.get("last_request_ts", "—"))

def get_user_input():
    try:
        return st.chat_input("Sua dúvida sobre fraudes...")
    except Exception:
        return st.text_input("Sua dúvida sobre fraudes...")

pergunta_text = get_user_input()

if COOPGUARD_DEBUG:
    if st.button("Forçar checagem de resultado (debug)"):
        poll_future()
        st.session_state["_coopguard_force_rerun"] = not st.session_state.get("_coopguard_force_rerun", False)
        st.info("Checagem executada; o app será re-renderizado automaticamente.")

# immediate poll
poll_future()

if pergunta_text:
    last_sent = st.session_state.get("last_sent_text")
    if pergunta_text != last_sent:
        st.session_state["last_result"] = None
        st.session_state["last_sent_text"] = pergunta_text
        st.info("Enviado para processamento em background. A resposta aparecerá quando pronta.")
        perguntar_async(pergunta_text)
    else:
        st.info("Pergunta já enviada; aguardando resultado...")

if st.session_state.get("last_result"):
    st.markdown("**Resposta do modelo:**")
    st.write(st.session_state.get("last_result"))
    rodape = f"\n\n---\n**Audit:** origem=modelo {MODELO}; timestamp={now_ts()}"
    st.markdown(rodape)

with st.expander("Logs de debug"):
    st.write("DATA_DIR candidates:", candidate_paths)
    st.write("DATA_DIR usado:", DATA_DIR)
    st.write("Perfil carregado:", perfil.get("nome"))
    st.write("Transações (linhas):", len(transacoes) if transacoes is not None else 0)
    st.write("Historico (linhas):", len(historico) if historico is not None else 0)
    st.write("Produtos (itens):", len(produtos) if isinstance(produtos, list) else 0)
    st.write("Última requisição:", st.session_state.get("last_request_ts", "—"))
    st.write("Último resultado disponível:", bool(st.session_state.get("last_result")))

with st.expander("Executor internals (debug)"):
    st.write("Has future:", "last_future" in st.session_state and st.session_state.get("last_future") is not None)
    fut = st.session_state.get("last_future")
    try:
        st.write("Future done:", fut.done() if fut is not None else False)
    except Exception as e:
        st.write("Future done: erro ao checar:", e)
    st.write("Future repr:", repr(fut))
    st.write("Last result raw:", st.session_state.get("last_result"))
```
---
**src/api_query.py (FastAPI minimal — para CI / testes)**
```
# src/api_query.py
from fastapi import FastAPI
from pydantic import BaseModel
import uvicorn
import json
from pathlib import Path

app = FastAPI()
DATA_DIR = Path(__file__).resolve().parent.parent / "data"

class Query(BaseModel):
    question: str

def call_agent_internal(question: str) -> str:
    # Minimal adapter: read data and return simple answers for tests.
    perfil_path = DATA_DIR / "perfil_investidor.json"
    try:
        perfil = json.loads(perfil_path.read_text(encoding="utf-8"))
    except Exception:
        perfil = {"nome": "Cliente"}
    q = question.lower()
    if "previsão do tempo" in q or "previsao do tempo" in q:
        return "Desculpe, eu trato apenas de assuntos financeiros."
    if "alimentação" in q or "alimentacao" in q:
        return "Para calcular gastos com alimentação, uso os registros de transações disponíveis."
    return "Resposta simulada: integre call_agent_internal ao seu app para respostas reais."

@app.post("/api/query")
def query(q: Query):
    try:
        answer = call_agent_internal(q.question)
        return {"answer": answer}
    except Exception as e:
        return {"answer": f"__ERROR__ {e}"}

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```
---
**src/debug_ollama.py**
```
# src/debug_ollama.py
import requests, time
url="http://localhost:11434/api/generate"
payload={"model":"gpt-oss:120b-cloud","prompt":"teste","stream":False}
t0=time.time()
try:
    r=requests.post(url,json=payload,timeout=10)
    print("status", r.status_code)
    print("duration_ms", int((time.time()-t0)*1000))
    print("text_preview:", r.text[:1000])
    try:
        print("json keys:", list(r.json().keys()))
    except Exception as e:
        print("json parse error:", e)
except Exception as e:
    print("request error:", e)
```
---
**src/run_tests.py**
```
# src/run_tests.py
import csv
import time
import requests
from pathlib import Path

APP_URL = "http://localhost:8000"
API_PATH = "/api/query"
OUT_DIR = Path("../results")
OUT_DIR.mkdir(parents=True, exist_ok=True)

TESTS = [
    {"id":"A","question":"Quanto gastei com alimentação?","type":"numeric_sum","category":"alimentacao"},
    {"id":"B","question":"Qual investimento você recomenda para mim?","type":"product_reco"},
    {"id":"C","question":"Qual a previsão do tempo?","type":"out_of_scope"},
    {"id":"D","question":"Quanto rende o produto XYZ?","type":"unknown_info"},
]

def call_agent_http(question, timeout=30):
    try:
        r = requests.post(f"{APP_URL}{API_PATH}", json={"question": question}, timeout=timeout)
        r.raise_for_status()
        payload = r.json()
        return payload.get("answer", ""), int(r.elapsed.total_seconds() * 1000)
    except Exception as e:
        return f"__ERROR__ {e}", None

def validate(test, answer):
    if answer is None:
        return False
    a = str(answer).lower()
    if test["type"] == "out_of_scope":
        return any(s in a for s in ["não trato", "fora do escopo", "não posso", "não tenho dados"])
    if test["type"] == "unknown_info":
        return any(s in a for s in ["não sei", "não tenho", "desconheço", "não consta"])
    return None

def run():
    out_file = OUT_DIR / "tests_results.csv"
    with open(out_file, "w", newline="", encoding="utf-8") as fh:
        writer = csv.writer(fh)
        writer.writerow(["timestamp","test_id","question","answer","valid","latency_ms"])
        for t in TESTS:
            ts = time.time()
            ans, lat = call_agent_http(t["question"])
            valid = validate(t, ans)
            writer.writerow([ts, t["id"], t["question"], ans.replace("\n"," "), valid, lat])
            print(f"[{t['id']}] sent; latency={lat}ms valid={valid}")
            time.sleep(0.5)
    print("Finished tests. Results in results/tests_results.csv")

if __name__ == "__main__":
    run()
```
---
**src/evaluate_results.py**
```
# src/evaluate_results.py
import pandas as pd
from pathlib import Path

IN_FILE = Path("../results/tests_results.csv")
if not IN_FILE.exists():
    print("Arquivo results/tests_results.csv não encontrado. Rode run_tests.py primeiro.")
    raise SystemExit(1)

df = pd.read_csv(IN_FILE)
total = len(df)
df['valid_bool'] = df['valid'].apply(lambda x: True if str(x).lower() == 'true' else False)
passed = df['valid_bool'].sum()
assertividade = passed / total if total else 0.0
errors = df['answer'].str.contains("__ERROR__", na=False).sum()
taxa_erro = errors / total if total else 0.0
latencia_media = df['latency_ms'].dropna().astype(float).mean()

print("=== Avaliação automática ===")
print(f"Total de testes: {total}")
print(f"Passaram (validados automaticamente): {passed}")
print(f"Assertividade: {assertividade:.2f}")
print(f"Taxa de erro: {taxa_erro:.2f}")
print(f"Latência média (ms): {latencia_media:.0f}")
```
---
**src/report.py**
```
# src/report.py
import pandas as pd
import matplotlib.pyplot as plt
from pathlib import Path

IN_FILE = Path("../results/tests_results.csv")
OUT_DIR = Path("../results")
OUT_DIR.mkdir(parents=True, exist_ok=True)

if not IN_FILE.exists():
    print("Arquivo results/tests_results.csv não encontrado. Rode run_tests.py primeiro.")
    raise SystemExit(1)

df = pd.read_csv(IN_FILE)
df['valid_bool'] = df['valid'].apply(lambda x: True if str(x).lower() == 'true' else False)
summary = df.groupby("test_id").agg(
    valid_rate=("valid_bool", lambda x: x.mean() if len(x) else 0),
    avg_latency=("latency_ms", "mean"),
    attempts=("question", "count")
).reset_index()

print(summary.to_string(index=False))

plt.figure(figsize=(6,4))
plt.bar(summary['test_id'], summary['valid_rate'], color='tab:blue')
plt.ylim(0,1)
plt.title('Taxa de sucesso por teste')
plt.ylabel('Taxa de sucesso')
plt.xlabel('Teste')
plt.tight_layout()
plt.savefig(OUT_DIR / "summary.png")
print(f"Relatório salvo em {OUT_DIR / 'summary.png'}")
```
---
**src/tail_log.ps1**
```
# src/tail_log.ps1
Get-Content ..\coopguard.log -Wait -Tail 80
```
---
**requirements.txt**
```
streamlit>=1.20.0
pandas
requests
fastapi
uvicorn
matplotlib
```
---
**.gitignore**
```
__pycache__/
.venv/
*.pyc
coopguard.log
results/
.env
```
---
**Dockerfile (opcional — container para API de testes)**
```
# Dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["uvicorn", "src.api_query:app", "--host", "0.0.0.0", "--port", "8000"]
```
---
**.github/workflows/ci.yml (GitHub Actions — roda lint e testes básicos)**
```
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: "3.11"
      - name: Install deps
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
      - name: Run basic tests (run_tests -> evaluate)
        run: |
          python -m uvicorn src.api_query:app --port 8000 --host 127.0.0.1 & sleep 1
          python src/run_tests.py
          python src/evaluate_results.py
```
---
**README.md (pronto para GitHub)**
```
# CoopGuard

Agente de prevenção de fraudes e gestão de riscos para cooperativas — versão local e scripts de avaliação.

## Estrutura
- `src/` — código fonte (app Streamlit, API de teste, scripts)
- `data/` — dados de exemplo (perfil, transações)
- `results/` — saída dos testes
- `.github/workflows/ci.yml` — CI básico

## Como rodar local
1. Criar venv e instalar dependências:
```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```
---
**Rodar API de teste (FastAPI):**
```
cd coopguard
uvicorn src.api_query:app --port 8000
```
---
**Em outra janela, rodar testes:**
```
python src/run_tests.py
python src/evaluate_results.py
python src/report.py
```
---
**Rodar o app Streamlit (opcional):**
```
cd coopguard/src
py -m streamlit run app.py
```
---
**Acompanhar logs:**
```
.\src\tail_log.ps1
```
---
### Métricas avançadas:
- Latência e tempo de resposta (registrado em results/)
- Taxa de erro e assertividade (scripts de avaliação)
- Consumo de tokens / custos: integrar com monitor de uso do provedor (se aplicável)
- Ferramentas recomendadas: LangWatch, LangFuse (integração opcional)
---
### Observações adicionais:
- Proteja coopguard.log e dados sensíveis; não comite dados reais.
- Para CI, o workflow roda a API de teste e os scripts de avaliação.
```

---

## Como subir para o GitHub (passos rápidos)
```bash
git init
git add .
git commit -m "Initial import: CoopGuard agent + tests"
git remote add origin git@github.com:SEU_USUARIO/coopguard.git
git branch -M main
git push -u origin main
```
---

### Para finalizar, se desejar, pode seguir mais estes passos recomendados a curto prazo:
- Substituir call_agent_internal em api_query.py por integração direta com app.py (ou expor /api/query no mesmo host).
- Sanitizar dados sensíveis antes de enviar ao modelo.
- Ativar COOPGUARD_DEBUG=1 localmente para debug; manter 0 em produção.
- Integrar métricas avançadas: exportar latência e taxa de erro para Prometheus / Grafana ou enviar eventos para LangFuse/LangWatch (se contratado).
- Adicionar autenticação se expor o Streamlit publicamente.
