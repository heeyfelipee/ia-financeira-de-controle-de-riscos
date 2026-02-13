# Código da Aplicação

Esta pasta contém o código do seu agente financeiro.

## Estrutura Sugerida

```
src/
├── app.py              # Aplicação principal (Streamlit/Gradio)
├── agente.py           # Lógica do agente
├── config.py           # Configurações (API keys, etc.)
└── requirements.txt    # Dependências
```

## Exemplo de requirements.txt

```
streamlit
openai
python-dotenv
```

## Como Rodar

```bash
# Instalar dependências
pip install -r requirements.txt

# Rodar a aplicação
streamlit run app.py
```

## Código que utilizei:

```
# app.py — CoopGuard (production-ready)
import os
import json
import time
import traceback
import logging
from logging.handlers import RotatingFileHandler
from datetime import datetime
import concurrent.futures

import pandas as pd
import requests
import streamlit as st

# ------------ CONFIGURAÇÃO via ENV (fallbacks) ------------
BASE_DIR = os.path.dirname(os.path.abspath(__file__))

OLLAMA_URL = os.environ.get("OLLAMA_URL", "http://localhost:11434/api/generate")
MODELO = os.environ.get("MODELO", "gpt-oss:120b-cloud")
REQUEST_TIMEOUT = int(os.environ.get("REQUEST_TIMEOUT", "120"))
MAX_RETRIES = int(os.environ.get("MAX_RETRIES", "3"))
EXECUTOR_MAX_WORKERS = int(os.environ.get("EXECUTOR_MAX_WORKERS", "2"))
LOG_FILE = os.environ.get("COOPGUARD_LOG", os.path.join(BASE_DIR, "coopguard.log"))
COOPGUARD_DEBUG = os.environ.get("COOPGUARD_DEBUG", "0") == "1"

# ------------ LOGGING (rotativo) ------------
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

# ------------ DATA DIR DETECTION ------------
candidate_paths = [
    os.path.join(BASE_DIR, "data"),
    os.path.abspath(os.path.join(BASE_DIR, "..", "data")),
    os.path.abspath(os.path.join(BASE_DIR, "..", "..", "data")),
]
DATA_DIR = next((p for p in candidate_paths if os.path.isdir(p)), candidate_paths[0])

def data_path(*parts):
    return os.path.join(DATA_DIR, *parts)

# ------------ I/O UTILITIES ------------
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

# ------------ SESSION STATE INIT ------------
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

# ------------ LOAD DATA (robust) ------------
perfil = safe_load_json(data_path('perfil_investidor.json'), {
    "nome": "N/D", "idade": 0, "perfil_investidor": "N/D",
    "objetivo_principal": "N/D", "patrimonio_total": 0, "reserva_emergencia_atual": 0
})
transacoes = safe_read_csv_or_empty(data_path('transacoes.csv'))
historico = safe_read_csv_or_empty(data_path('historico_atendimento.csv'))
produtos = safe_load_json(data_path('produtos_financeiros.json'), [])

# ------------ CONTEXT BUILD (safe/truncated) ------------
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

# ------------ SYSTEM PROMPT (kept concise) ------------
SYSTEM_PROMPT = (
    "Você é um agente financeiro inteligente especializado em prevenção de fraudes, conformidade e gestão de riscos "
    "em cooperativas financeiras. Priorize segurança, auditabilidade e baseie-se em registros primários."
)

# ------------ MODEL RESPONSE PARSER ------------
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

# ------------ CALL OLLAMA (robust with retries) ------------
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

# ------------ BACKGROUND EXECUTION (submit/poll) ------------
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

# ensure executor shutdown on exit
import atexit
def _shutdown_executor():
    try:
        ex = st.session_state.get("executor")
        if ex:
            ex.shutdown(wait=False)
            log("Executor shutdown invoked.")
    except Exception:
        pass
atexit.register(_shutdown_executor)

# ------------ STREAMLIT UI ------------
st.set_page_config(page_title="CoopGuard", layout="wide")
st.title("CoopGuard — O Guardião da sua Cooperativa")

st.markdown(f"**Contexto carregado de:** `{DATA_DIR}`")
with st.expander("Ver contexto (apenas para revisão interna)"):
    st.code(contexto[:1000] + ("\n\n... (contexto truncado)" if len(contexto) > 1000 else ""), language="text")

# Sidebar health
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

# input (chat_input if available)
def get_user_input():
    try:
        return st.chat_input("Sua dúvida sobre fraudes...")
    except Exception:
        return st.text_input("Sua dúvida sobre fraudes...")

pergunta_text = get_user_input()

# optional debug button (only when COOPGUARD_DEBUG=1)
if COOPGUARD_DEBUG:
    if st.button("Forçar checagem de resultado (debug)"):
        poll_future()
        st.session_state["_coopguard_force_rerun"] = not st.session_state.get("_coopguard_force_rerun", False)
        st.info("Checagem executada; o app será re-renderizado automaticamente.")

# immediate poll to reduce wait window
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

# show result
if st.session_state.get("last_result"):
    st.markdown("**Resposta do modelo:**")
    st.write(st.session_state.get("last_result"))
    rodape = f"\n\n---\n**Audit:** origem=modelo {MODELO}; timestamp={now_ts()}"
    st.markdown(rodape)

# debug expander (minimal)
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
