import streamlit as st
from openai import OpenAI
import matplotlib.pyplot as plt
import json
import re

# --- CONFIGURAÇÃO INICIAL ---
st.set_page_config(page_title="Hub de Agentes - Data Viz", page_icon="📊", layout="wide")

# Estilos CSS para dar um visual de Dashboard
st.markdown("""
<style>
    .stButton>button {width: 100%; border-radius: 8px; height: 3em; background-color: #2962FF; color: white; font-weight: bold;}
    h1, h2, h3 {font-family: 'Helvetica Neue', sans-serif;}
    .metric-card {background-color: #f0f2f6; padding: 20px; border-radius: 10px; border-left: 5px solid #2962FF;}
</style>
""", unsafe_allow_html=True)

# --- SIDEBAR & API ---
st.sidebar.title("💎 Hub Premium")
api_key = st.sidebar.text_input("Sua API Key (OpenAI)", type="password")
client = OpenAI(api_key=api_key) if api_key else None

# --- FUNÇÃO AUXILIAR: GERADOR DE GRÁFICOS ---
def gerar_grafico_mercado(dados_json):
    """
    Recebe um dicionário com dados e gera um gráfico de rosca (Donut Chart).
    """
    labels = list(dados_json.keys())
    sizes = list(dados_json.values())
    
    # Cores profissionais
    colors = ['#2962FF', '#448AFF', '#82B1FF', '#B388FF', '#EA80FC']
    
    fig, ax = plt.subplots(figsize=(6, 3))
    wedges, texts, autotexts = ax.pie(sizes, labels=labels, autopct='%1.1f%%', startangle=90, colors=colors[:len(labels)], textprops=dict(color="black"))
    
    # Transformar em gráfico de rosca
    centre_circle = plt.Circle((0,0),0.70,fc='white')
    fig.gca().add_artist(centre_circle)
    
    ax.axis('equal')  
    plt.tight_layout()
    return fig

# --- CÉREBRO DO FLUXO (PIPELINE) ---
def executar_fluxo_com_dados(tema_negocio):
    resultados = []
    
    # 1. ANÁLISE QUALITATIVA (TEXTO)
    with st.status("🤖 Passo 1: Agentes analisando o mercado...", expanded=True) as status:
        prompt_texto = f"Analise o nicho '{tema_negocio}'. Liste 3 grandes oportunidades e 3 riscos principais de forma direta."
        res_texto = client.chat.completions.create(model="gpt-4o", messages=[{"role": "user", "content": prompt_texto}]).choices[0].message.content
        resultados.append({"tipo": "texto", "titulo": "📝 Análise Estratégica", "conteudo": res_texto})
        status.update(label="Análise Textual Completa!", state="running")
        
        # 2. ANÁLISE QUANTITATIVA (DADOS PARA O GRÁFICO)
        # Aqui está o segredo: Pedimos JSON estrito para o Python ler.
        prompt_dados = f"""
        Com base no nicho '{tema_negocio}', estime uma divisão percentual fictícia mas realista dos 4 principais tipos de concorrentes ou canais de venda.
        Responda APENAS um JSON válido no formato: {{"Tipo A": 30, "Tipo B": 20, "Tipo C": 40, "Outros": 10}}. Não escreva nada além do JSON.
        """
        res_dados_raw = client.chat.completions.create(model="gpt-4o", messages=[{"role": "user", "content": prompt_dados}]).choices[0].message.content
        
        # Limpeza simples para garantir que o JSON funcione
        try:
            # Tenta encontrar o JSON dentro da resposta (caso venha texto extra)
            match = re.search(r"\{.*\}", res_dados_raw, re.DOTALL)
            json_str = match.group(0) if match else res_dados_raw
            dados_dict = json.loads(json_str)
            
            # Gera o gráfico
            fig_grafico = gerar_grafico_mercado(dados_dict)
            resultados.append({"tipo": "grafico", "titulo": "📊 Market Share Estimado (Canais/Concorrência)", "objeto": fig_grafico})
            
        except Exception as e:
            resultados.append({"tipo": "erro", "titulo": "Erro no Gráfico", "conteudo": f"Não foi possível gerar os dados visuais: {e}"})

        status.update(label="Gráficos Gerados!", state="running")
        
        # 3. PLANO DE AÇÃO (CONCLUSÃO)
        prompt_acao = f"Com base na análise anterior ({res_texto}), crie um plano de ação de 3 passos para lançar em 30 dias."
        res_acao = client.chat.completions.create(model="gpt-4o", messages=[{"role": "user", "content": prompt_acao}]).choices[0].message.content
        resultados.append({"tipo": "texto", "titulo": "🚀 Plano de Ação 30 Dias", "conteudo": res_acao})
        
        status.update(label="Processo Finalizado com Sucesso!", state="complete", expanded=False)
        
    return resultados

# --- INTERFACE DO USUÁRIO ---
st.header("📊 Hub de Agentes: Intelligence Dashboard")
st.markdown("Gere análises de mercado completas com **relatórios visuais automáticos**.")

entrada = st.text_input("Qual mercado você quer dominar hoje?", placeholder="Ex: Roupas fitness sustentáveis")

if st.button("Gerar Intelligence Report"):
    if not api_key:
        st.error("Insira a API Key na barra lateral.")
    elif not entrada:
        st.warning("Digite um mercado para começar.")
    else:
        # Execução
        steps = executar_fluxo_com_dados(entrada)
        
        st.markdown("---")
        st.subheader(f"Relatório de Inteligência: {entrada.title()}")
        
        # Renderização Dinâmica
        col1, col2 = st.columns([3, 2]) # Coluna texto maior, gráfico menor
        
        for step in steps:
            if step["tipo"] == "texto":
                with st.expander(step["titulo"], expanded=True):
                    st.markdown(step["conteudo"])
            elif step["tipo"] == "grafico":
                # Mostra o gráfico em um container visualmente agradável
                with col2:
                    st.markdown(f"**{step['titulo']}**")
                    st.pyplot(step["objeto"])
                    st.caption("*Dados estimados via IA baseados em tendências de mercado.")

# --- DICA PRO ---
st.sidebar.markdown("---")
st.sidebar.info("💡 **Dica de Senior:** Este gráfico é gerado em tempo real pelo Python (Matplotlib) lendo dados estruturados extraídos pelo GPT-4.")
