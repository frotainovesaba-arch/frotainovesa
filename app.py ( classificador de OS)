import streamlit as st
import pandas as pd
import unicodedata
import re
from difflib import SequenceMatcher

# ------------- Utilidades -------------
def normalize_text(t):
    """Normaliza texto: minúsculas, sem acentos, sem pontuação extra."""
    if pd.isna(t):
        return ""
    t = str(t).lower()
    t = unicodedata.normalize("NFD", t)
    t = t.encode("ascii", "ignore").decode("utf-8")
    t = re.sub(r"[^\w\s]", " ", t)
    t = re.sub(r"\s+", " ", t).strip()
    return t

def score_match(query_norm, keywords_norm_list):
    """
    Pontua a correspondência entre a consulta e uma lista de palavras-chave.
    - +1 para cada keyword contida na consulta
    - bônus pequeno por similaridade parcial
    """
    score = 0.0
    for kw in keywords_norm_list:
        if not kw:
            continue
        if kw in query_norm:
            score += 1.0
        else:
            score += max(SequenceMatcher(None, query_norm, kw).ratio() - 0.5, 0) * 0.5
    return score

def safe_split_keywords(s):
    """Divide a coluna 'Palavras-chave' por ';' e normaliza cada termo."""
    if pd.isna(s):
        return []
    return [normalize_text(k) for k in str(s).split(";")]

# ------------- UI / Layout -------------
st.set_page_config(page_title="Classificador de Serviços", page_icon="🧰", layout="centered")
st.title("Classificador de Serviços de Manutenção 🧰")
st.caption("Digite a peça/serviço e eu sugiro o Grupo e o Item com base na sua planilha.")

with st.expander("Como preparar a planilha base (CSV/XLSX)", expanded=False):
    st.markdown(
        "- Colunas obrigatórias: `Grupo`, `Itens do Grupo`\n"
        "- Colunas opcionais: `Palavras-chave` (ex.: 'radiador; valvula termostatica; bomba dagua'), "
        "`Aplicavel` (ex.: 'TP; CP; CC; LEV')\n"
        "- Dica: para cada item, inclua palavras que o usuário provavelmente digitaria."
    )

uploaded = st.file_uploader("Carregue a planilha base (CSV ou Excel)", type=["csv", "xlsx", "xls"])

col1, col2 = st.columns([2,1])
with col1:
    termo = st.text_input("Digite a peça/serviço:", "", placeholder="Ex.: radiador vazando, junta do cabeçote, pastilha de freio...")
with col2:
    tipo = st.selectbox("Tipo de equipamento (opcional)", ["(Não especificado)", "TP", "CP", "CC", "LEV"])

top_k = st.slider("Quantidade de sugestões", min_value=1, max_value=5, value=3)

# ------------- Processamento -------------
df = None
if uploaded:
    try:
        if uploaded.name.endswith(".csv"):
            df = pd.read_csv(uploaded)
        else:
            df = pd.read_excel(uploaded)
    except Exception as e:
        st.error(f"Erro ao ler a planilha: {e}")
        st.stop()

    # Checagem de colunas
    obrigatorias = ["Grupo", "Itens do Grupo"]
    faltantes = [c for c in obrigatorias if c not in df.columns]
    if faltantes:
        st.error(f"Colunas obrigatórias ausentes: {', '.join(faltantes)}")
        st.stop()

    if "Palavras-chave" not in df.columns:
        df["Palavras-chave"] = ""

    # Normalizações auxiliares
    df["_grupo_norm"] = df["Grupo"].apply(normalize_text)
    df["_item_norm"] = df["Itens do Grupo"].apply(normalize_text)
    df["_keys_list"] = df["Palavras-chave"].apply(safe_split_keywords)

    # Botão de classificação
    do_classify = st.button("Classificar")

    if do_classify:
        if not termo.strip():
            st.warning("Digite um termo de consulta.")
            st.stop()

        query_norm = normalize_text(termo)

        # Filtrar por tipo (se houver coluna 'Aplicavel' e tipo selecionado)
        df_filtered = df.copy()
        tipo_sel = tipo.upper()
        if "Aplicavel" in df.columns and tipo_sel != "(NÃO ESPECIFICADO)":
            mask = df["Aplicavel"].fillna("").str.upper().str.contains(tipo_sel, na=False)
            if mask.any():
                df_filtered = df[mask].copy()
            else:
                # Fallback caso nenhum item marque explicitamente esse tipo
                df_filtered = df.copy()

        # Scoring geral: keywords + similaridade com nome do item e do grupo
        scores = []
        for _, row in df_filtered.iterrows():
            s = 0.0
            s += score_match(query_norm, row["_keys_list"])
            s += SequenceMatcher(None, query_norm, row["_item_norm"]).ratio() * 0.7
            s += SequenceMatcher(None, query_norm, row["_grupo_norm"]).ratio() * 0.3

            if "Aplicavel" in df.columns and tipo_sel != "(NÃO ESPECIFICADO)":
                aplicavel = str(row.get("Aplicavel", "")).upper()
                if tipo_sel in aplicavel:
                    s += 0.2  # pequeno bônus por alinhamento de tipo

            scores.append(s)

        df_out = df_filtered.copy()
        df_out["_score"] = scores
        df_out = df_out.sort_values("_score", ascending=False)

        if df_out.empty or df_out["_score"].max() <= 0:
            st.info("Nenhuma correspondência forte encontrada. Tente detalhar mais o termo ou revise as palavras‑chave na base.")
        else:
            st.subheader("Sugestões")
            st.dataframe(df_out.head(top_k)[["Grupo", "Itens do Grupo", "Palavras-chave"]].reset_index(drop=True), use_container_width=True)

            best = df_out.iloc[0]
            st.success(f"Classificação sugerida: {best['Grupo']} → {best['Itens do Grupo']}")

            with st.expander("Detalhes da pontuação (top 10)", expanded=False):
                st.dataframe(
                    df_out.head(10)[["Grupo", "Itens do Grupo", "Palavras-chave", "_score"]]
                    .reset_index(drop=True)
                )
else:
    st.info("Aguardando upload da planilha base.")

# ------------- Rodapé / Ajuda -------------
st.markdown("""
Formato recomendado da planilha:
- Grupo
- Itens do Grupo
- Palavras-chave (ex.: "radiador; valvula termostatica; bomba dagua; superaquecimento")
- Aplicavel (opcional): valores como "TP; CP; CC; LEV" para indicar tipos.

Dicas:
- Inclua variações sem acento e termos populares.
- Se o termo for algo como "TP radiador vazando", você pode selecionar TP no campo acima para priorizar serviços de tratores.
""")
