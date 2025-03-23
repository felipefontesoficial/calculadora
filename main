import streamlit as st
import pandas as pd
import numpy as np
import datetime
import matplotlib.pyplot as plt
import base64
from io import BytesIO

# Configuração da página
st.set_page_config(
    page_title="Calculadora Previdenciária",
    page_icon="📊",
    layout="wide",
    initial_sidebar_state="expanded"
)

# Estilos CSS
st.markdown("""
<style>
    .main-header {
        font-size: 2.5rem;
        color: #1E6091;
        text-align: center;
        margin-bottom: 1rem;
    }
    .sub-header {
        font-size: 1.5rem;
        color: #1E6091;
        margin-bottom: 1rem;
    }
    .info-box {
        background-color: #e6f2ff;
        padding: 10px;
        border-radius: 5px;
        margin-bottom: 10px;
    }
    .warning-box {
        background-color: #fff3cd;
        padding: 10px;
        border-radius: 5px;
        margin-bottom: 10px;
    }
    .success-box {
        background-color: #d4edda;
        padding: 10px;
        border-radius: 5px;
        margin-bottom: 10px;
    }
</style>
""", unsafe_allow_html=True)

# Título
st.markdown("<h1 class='main-header'>Calculadora Previdenciária</h1>", unsafe_allow_html=True)

# Sidebar para navegação
st.sidebar.title("Navegação")
page = st.sidebar.radio("Ir para:", 
    ["Início", "Dados do Segurado", "Contribuições", "Tempo Especial", 
     "Regras de Aposentadoria", "Resultados", "Atrasados", "Exportar Relatório"])

# Dados (normalmente viriam de um banco de dados)
@st.cache_data
def load_inss_data():
    # Teto do INSS por ano (valores simplificados para exemplo)
    tetos = {
        2019: 5839.45,
        2020: 6101.06,
        2021: 6433.57,
        2022: 7087.22,
        2023: 7507.49,
        2024: 7786.02,
        2025: 8000.00,  # Valor fictício para exemplo
    }
    
    # Salário mínimo por ano
    salarios_minimos = {
        2019: 998.00,
        2020: 1045.00,
        2021: 1100.00,
        2022: 1212.00,
        2023: 1320.00,
        2024: 1412.00,
        2025: 1500.00,  # Valor fictício para exemplo
    }
    
    # Índices INPC simplificados (acumulado anual fictício)
    inpc = {
        2019: 1.0400,
        2020: 1.0558,
        2021: 1.1006,
        2022: 1.0583,
        2023: 1.0462,
        2024: 1.0400,
        2025: 1.0200,  # Valor fictício para exemplo
    }
    
    return tetos, salarios_minimos, inpc

# Função para calcular a média salarial
def calcular_media_salarial(contribuicoes, regra='atual'):
    if not contribuicoes:
        return 0
    
    # Ordenar contribuições por valor (do maior para o menor)
    contribuicoes_ordenadas = sorted(contribuicoes, key=lambda x: x['valor_corrigido'], reverse=True)
    
    if regra == 'atual':
        # Regra atual: média de todas as contribuições
        total = sum(contrib['valor_corrigido'] for contrib in contribuicoes)
        return total / len(contribuicoes)
    else:
        # Regra antiga: média dos 80% maiores salários
        n_contrib = int(len(contribuicoes) * 0.8)
        total = sum(contrib['valor_corrigido'] for contrib in contribuicoes_ordenadas[:n_contrib])
        return total / n_contrib if n_contrib > 0 else 0

# Função para calcular tempo de contribuição
def calcular_tempo_contribuicao(contribuicoes, tempo_especial=0):
    # Contar meses de contribuição (simplificado)
    meses = len(contribuicoes)
    # Adicionar tempo especial convertido (em meses)
    meses += tempo_especial
    
    anos = meses // 12
    meses_restantes = meses % 12
    
    return anos, meses_restantes

# Função para calcular fator previdenciário (simplificado)
def calcular_fator_previdenciario(idade, tempo_contribuicao, sexo):
    # Fórmula simplificada para fins didáticos
    expectativa_sobrevida = 85 - idade if idade < 85 else 5  # Simplificação
    
    # Tc = tempo de contribuição, Id = idade, Es = expectativa de sobrevida
    # Fator = (Tc * 0.31) / Es * [1 + (Id + Tc * 0.31) / 100]
    tc_anos = tempo_contribuicao[0] + (tempo_contribuicao[1] / 12)
    
    fator = (tc_anos * 0.31) / expectativa_sobrevida * (1 + (idade + tc_anos * 0.31) / 100)
    
    return max(fator, 1)  # O fator não pode ser inferior a 1 em algumas regras

# Funções para as regras de aposentadoria
def verificar_aposentadoria_idade(idade, tempo_contribuicao, sexo):
    # Regra atual pós reforma
    idade_minima = 62 if sexo == "F" else 65
    carencia_minima = 15 if sexo == "F" else 20  # Anos
    
    if idade >= idade_minima and tempo_contribuicao[0] >= carencia_minima:
        return True, "Elegível para Aposentadoria por Idade"
    else:
        return False, f"Não elegível. Idade mínima: {idade_minima}, Carência mínima: {carencia_minima} anos."

def verificar_aposentadoria_tempo_contribuicao(idade, tempo_contribuicao, sexo):
    # Regra de transição por pontos (simplificada)
    pontos = idade + tempo_contribuicao[0] + (tempo_contribuicao[1] / 12)
    pontos_necessarios = 89 if sexo == "F" else 99  # Para 2023
    
    tempo_minimo = 30 if sexo == "F" else 35  # Anos
    
    if tempo_contribuicao[0] >= tempo_minimo and pontos >= pontos_necessarios:
        return True, f"Elegível pela regra de pontos. Pontos atuais: {pontos:.1f}"
    else:
        return False, f"Não elegível. Pontos necessários: {pontos_necessarios}, Pontos atuais: {pontos:.1f}. Tempo mínimo: {tempo_minimo} anos."

def verificar_aposentadoria_idade_progressiva(idade, tempo_contribuicao, sexo):
    # Regra de transição por idade progressiva
    idade_minima = 58 if sexo == "F" else 63  # Para 2023
    tempo_minimo = 30 if sexo == "F" else 35  # Anos
    
    if idade >= idade_minima and tempo_contribuicao[0] >= tempo_minimo:
        return True, f"Elegível pela regra de idade progressiva. Idade mínima: {idade_minima}"
    else:
        return False, f"Não elegível. Idade mínima: {idade_minima}, Idade atual: {idade}. Tempo mínimo: {tempo_minimo} anos."

def calcular_rmi(media_salarial, tempo_contribuicao, regra, sexo):
    # Cálculo da RMI pela regra atual
    if regra == 'atual':
        tempo_minimo = 15 if sexo == "F" else 20
        anos_excedentes = max(0, tempo_contribuicao[0] - tempo_minimo)
        percentual = 60 + (anos_excedentes * 2)  # 60% + 2% por ano excedente
        percentual = min(percentual, 100)  # Máximo de 100%
        
        rmi = media_salarial * (percentual / 100)
    else:
        # Regra antiga com fator previdenciário
        idade = st.session_state.idade if 'idade' in st.session_state else 55
        fator = calcular_fator_previdenciario(idade, tempo_contribuicao, sexo)
        rmi = media_salarial * fator
    
    # Verificar limites (teto e piso)
    tetos, salarios_minimos, _ = load_inss_data()
    ano_atual = datetime.datetime.now().year
    
    teto = tetos.get(ano_atual, 7786.02)  # Valor padrão caso não encontre
    piso = salarios_minimos.get(ano_atual, 1412.00)  # Valor padrão caso não encontre
    
    rmi = max(min(rmi, teto), piso)
    
    return rmi

# Função para exportar relatório como Excel
def export_to_excel():
    if 'contribuicoes' not in st.session_state or len(st.session_state.contribuicoes) == 0:
        return None
    
    # Criar um DataFrame com as contribuições
    df = pd.DataFrame(st.session_state.contribuicoes)
    
    # Criar um buffer para salvar o Excel
    output = BytesIO()
    
    # Criar um writer do Excel
    with pd.ExcelWriter(output, engine='xlsxwriter') as writer:
        # Dados do segurado
        info_dados = {
            'Campo': ['Nome', 'CPF', 'Data de Nascimento', 'Sexo', 'DER'],
            'Valor': [
                st.session_state.get('nome', ''),
                st.session_state.get('cpf', ''),
                st.session_state.get('data_nascimento', ''),
                st.session_state.get('sexo', ''),
                st.session_state.get('der', '')
            ]
        }
        df_dados = pd.DataFrame(info_dados)
        df_dados.to_excel(writer, sheet_name='Dados_Segurado', index=False)
        
        # Contribuições
        df.to_excel(writer, sheet_name='Contribuicoes', index=False)
        
        # Resultados
        if 'rmi_atual' in st.session_state and 'rmi_antiga' in st.session_state:
            info_resultados = {
                'Métrica': [
                    'RMI (Regra Atual)', 
                    'RMI (Regra Antiga)',
                    'Tempo de Contribuição (Anos)',
                    'Tempo de Contribuição (Meses)',
                    'Média Salarial (Regra Atual)',
                    'Média Salarial (Regra Antiga)'
                ],
                'Valor': [
                    st.session_state.get('rmi_atual', 0),
                    st.session_state.get('rmi_antiga', 0),
                    st.session_state.get('tempo_contribuicao', (0, 0))[0],
                    st.session_state.get('tempo_contribuicao', (0, 0))[1],
                    st.session_state.get('media_atual', 0),
                    st.session_state.get('media_antiga', 0)
                ]
            }
            df_resultados = pd.DataFrame(info_resultados)
            df_resultados.to_excel(writer, sheet_name='Resultados', index=False)
    
    return output.getvalue()

# Inicialização da sessão
if 'contribuicoes' not in st.session_state:
    st.session_state.contribuicoes = []

if 'tempo_especial_meses' not in st.session_state:
    st.session_state.tempo_especial_meses = 0

# Páginas
if page == "Início":
    st.markdown("<div class='info-box'>", unsafe_allow_html=True)
    st.markdown("""
    ### Bem-vindo à Calculadora Previdenciária
    
    Esta ferramenta permite realizar cálculos previdenciários para aposentadorias no Brasil, 
    considerando a legislação atual e as regras de transição.
    
    **Funcionalidades:**
    - Cadastro de dados do segurado
    - Registro de contribuições
    - Cadastro de tempo especial
    - Simulação de regras de aposentadoria
    - Cálculo de benefícios
    - Geração de relatórios
    
    **Como utilizar:**
    1. Navegue pelas seções usando o menu à esquerda
    2. Preencha os dados solicitados
    3. Verifique os resultados
    4. Exporte o relatório
    
    > Nota: Esta é uma versão simplificada para fins educacionais
    """)
    st.markdown("</div>", unsafe_allow_html=True)
    
    st.image("https://via.placeholder.com/800x300?text=Calculadora+Previdenci%C3%A1ria", use_column_width=True)

elif page == "Dados do Segurado":
    st.markdown("<h2 class='sub-header'>Dados do Segurado</h2>", unsafe_allow_html=True)
    
    col1, col2 = st.columns(2)
    
    with col1:
        nome = st.text_input("Nome completo", key="nome_input")
        if nome:
            st.session_state.nome = nome
            
        cpf = st.text_input("CPF", key="cpf_input")
        if cpf:
            st.session_state.cpf = cpf
        
        data_nascimento = st.date_input("Data de Nascimento", 
                                        min_value=datetime.date(1940, 1, 1),
                                        max_value=datetime.date.today(),
                                        value=datetime.date(1970, 1, 1),
                                        key="data_nascimento_input")
        if data_nascimento:
            st.session_state.data_nascimento = data_nascimento
            # Calcular idade
            hoje = datetime.date.today()
            idade = hoje.year - data_nascimento.year - ((hoje.month, hoje.day) < (data_nascimento.month, data_nascimento.day))
            st.session_state.idade = idade
    
    with col2:
        sexo = st.selectbox("Sexo", ["M", "F"], key="sexo_input")
        if sexo:
            st.session_state.sexo = sexo
            
        der = st.date_input("Data de Entrada do Requerimento (DER)",
                           min_value=datetime.date(2019, 11, 13),  # Data da reforma previdenciária
                           max_value=datetime.date.today() + datetime.timedelta(days=365),
                           value=datetime.date.today(),
                           key="der_input")
        if der:
            st.session_state.der = der
            
        categoria = st.selectbox("Categoria de Segurado", 
                               ["Empregado", "Contribuinte Individual", "Segurado Especial", 
                                "Facultativo", "Doméstico", "Avulso"],
                               key="categoria_input")
        if categoria:
            st.session_state.categoria = categoria
    
    if 'idade' in st.session_state:
        st.markdown(f"<div class='info-box'>Idade atual: {st.session_state.idade} anos</div>", unsafe_allow_html=True)
    
    if st.button("Salvar Dados do Segurado"):
        st.markdown("<div class='success-box'>Dados salvos com sucesso!</div>", unsafe_allow_html=True)

elif page == "Contribuições":
    st.markdown("<h2 class='sub-header'>Cadastro de Contribuições</h2>", unsafe_allow_html=True)
    
    with st.expander("Adicionar Nova Contribuição", expanded=True):
        col1, col2 = st.columns(2)
        
        with col1:
            competencia = st.date_input("Competência (mês/ano)",
                                      min_value=datetime.date(1994, 7, 1),
                                      max_value=datetime.date.today(),
                                      value=datetime.date.today().replace(day=1) - datetime.timedelta(days=1),
                                      key="competencia_input")
            
            valor = st.number_input("Valor da Contribuição (R$)", 
                                  min_value=0.0, 
                                  max_value=10000.0, 
                                  value=1500.0,
                                  key="valor_input")
            
        with col2:
            tipo_contribuicao = st.selectbox("Tipo de Contribuição",
                                           ["Normal", "Facultativa", "Complementar"],
                                           key="tipo_contribuicao_input")
            
            comprovante = st.selectbox("Tipo de Comprovante",
                                     ["CNIS", "GPS", "Carnê", "Folha de Pagamento", "Outro"],
                                     key="comprovante_input")
        
        # Carregar dados do INSS para correção monetária
        tetos, salarios_minimos, inpc = load_inss_data()
        
        # Correção monetária simplificada
        ano_competencia = competencia.year
        ano_atual = datetime.datetime.now().year
        
        fator_correcao = 1.0
        for ano in range(ano_competencia, ano_atual + 1):
            if ano in inpc:
                fator_correcao *= inpc[ano]
        
        valor_corrigido = valor * fator_correcao
        
        # Exibir valor corrigido
        st.markdown(f"<div class='info-box'>Valor corrigido: R$ {valor_corrigido:.2f}</div>", unsafe_allow_html=True)
        
        if st.button("Adicionar Contribuição"):
            # Verificar limites
            teto_ano = tetos.get(ano_competencia, tetos[min(tetos.keys(), key=lambda k: abs(k-ano_competencia))])
            salario_minimo_ano = salarios_minimos.get(ano_competencia, salarios_minimos[min(salarios_minimos.keys(), key=lambda k: abs(k-ano_competencia))])
            
            # Ajustar para o teto, se necessário
            valor_ajustado = min(valor, teto_ano)
            
            if valor_ajustado < salario_minimo_ano and tipo_contribuicao != "Complementar":
                st.warning(f"Valor abaixo do salário mínimo ({salario_minimo_ano:.2f})")
            
            nova_contribuicao = {
                "competencia": competencia.strftime("%m/%Y"),
                "valor": valor_ajustado,
                "tipo": tipo_contribuicao,
                "comprovante": comprovante,
                "valor_corrigido": valor_corrigido
            }
            
            st.session_state.contribuicoes.append(nova_contribuicao)
            st.markdown("<div class='success-box'>Contribuição adicionada com sucesso!</div>", unsafe_allow_html=True)
    
    # Exibir contribuições cadastradas
    if st.session_state.contribuicoes:
        st.markdown("<h3>Contribuições Cadastradas</h3>", unsafe_allow_html=True)
        
        # Converter para DataFrame para melhor exibição
        df_contribuicoes = pd.DataFrame(st.session_state.contribuicoes)
        st.dataframe(df_contribuicoes, use_container_width=True)
        
        # Opção para remover contribuições
        if st.button("Limpar Todas as Contribuições"):
            st.session_state.contribuicoes = []
            st.experimental_rerun()
        
        # Calcular e exibir estatísticas
        total_contribuicoes = len(st.session_state.contribuicoes)
        soma_valores = sum(contrib["valor"] for contrib in st.session_state.contribuicoes)
        soma_valores_corrigidos = sum(contrib["valor_corrigido"] for contrib in st.session_state.contribuicoes)
        
        st.markdown(f"""
        <div class='info-box'>
        <strong>Resumo:</strong><br>
        Total de contribuições: {total_contribuicoes}<br>
        Soma dos valores originais: R$ {soma_valores:.2f}<br>
        Soma dos valores corrigidos: R$ {soma_valores_corrigidos:.2f}
        </div>
        """, unsafe_allow_html=True)
        
        # Gráfico de contribuições ao longo do tempo
        if st.checkbox("Exibir Gráfico de Contribuições"):
            df_contribuicoes['competencia'] = pd.to_datetime(df_contribuicoes['competencia'], format='%m/%Y')
            df_contribuicoes = df_contribuicoes.sort_values('competencia')
            
            fig, ax = plt.subplots(figsize=(10, 6))
            ax.plot(df_contribuicoes['competencia'], df_contribuicoes['valor'], label='Valor Original')
            ax.plot(df_contribuicoes['competencia'], df_contribuicoes['valor_corrigido'], label='Valor Corrigido')
            ax.set_title('Evolução das Contribuições ao Longo do Tempo')
            ax.set_xlabel('Competência')
            ax.set_ylabel('Valor (R$)')
            ax.legend()
            ax.grid(True)
            
            st.pyplot(fig)
    else:
        st.markdown("<div class='warning-box'>Nenhuma contribuição cadastrada.</div>", unsafe_allow_html=True)

elif page == "Tempo Especial":
    st.markdown("<h2 class='sub-header'>Tempo Especial</h2>", unsafe_allow_html=True)
    
    with st.expander("Adicionar Tempo Especial", expanded=True):
        col1, col2 = st.columns(2)
        
        with col1:
            data_inicio = st.date_input("Data de Início",
                                       min_value=datetime.date(1970, 1, 1),
                                       max_value=datetime.date.today(),
                                       value=datetime.date(2000, 1, 1),
                                       key="data_inicio_especial")
            
            fator_conversao = st.selectbox("Fator de Conversão",
                                         [1.4, 1.75, 2.0],
                                         key="fator_conversao")
        
        with col2:
            data_fim = st.date_input("Data de Fim",
                                    min_value=data_inicio,
                                    max_value=datetime.date.today(),
                                    value=datetime.date(2005, 1, 1),
                                    key="data_fim_especial")
            
            agente_nocivo = st.selectbox("Agente Nocivo",
                                       ["Ruído", "Calor", "Químicos", "Biológicos", "Radiação", "Outro"],
                                       key="agente_nocivo")
        
        tem_ppp = st.checkbox("Possui PPP/LTCAT", value=True, key="tem_ppp")
        
        # Cálculo do tempo em dias e meses
        if data_inicio and data_fim:
            delta = data_fim - data_inicio
            dias_trabalhados = delta.days + 1  # Inclusivo
            meses_trabalhados = dias_trabalhados // 30
            
            # Aplicar fator de conversão
            meses_convertidos = int(meses_trabalhados * fator_conversao)
            
            st.markdown(f"""
            <div class='info-box'>
            Tempo trabalhado: {meses_trabalhados} meses ({dias_trabalhados} dias)<br>
            Tempo convertido: {meses_convertidos} meses com fator {fator_conversao}
            </div>
            """, unsafe_allow_html=True)
        
        if st.button("Adicionar Tempo Especial"):
            if not tem_ppp:
                st.warning("Recomenda-se ter PPP/LTCAT para comprovar o tempo especial")
            
            if 'tempo_especial' not in st.session_state:
                st.session_state.tempo_especial = []
            
            novo_tempo_especial = {
                "inicio": data_inicio.strftime("%d/%m/%Y"),
                "fim": data_fim.strftime("%d/%m/%Y"),
                "fator": fator_conversao,
                "agente": agente_nocivo,
                "tem_ppp": tem_ppp,
                "meses_originais": meses_trabalhados,
                "meses_convertidos": meses_convertidos
            }
            
            st.session_state.tempo_especial.append(novo_tempo_especial)
            st.session_state.tempo_especial_meses += meses_convertidos - meses_trabalhados  # Adicionar apenas o acréscimo
            st.markdown("<div class='success-box'>Tempo especial adicionado com sucesso!</div>", unsafe_allow_html=True)
    
    # Exibir tempos especiais cadastrados
    if 'tempo_especial' in st.session_state and st.session_state.tempo_especial:
        st.markdown("<h3>Tempos Especiais Cadastrados</h3>", unsafe_allow_html=True)
        
        # Converter para DataFrame para melhor exibição
        df_tempo_especial = pd.DataFrame(st.session_state.tempo_especial)
        st.dataframe(df_tempo_especial, use_container_width=True)
        
        # Opção para remover tempos especiais
        if st.button("Limpar Todos os Tempos Especiais"):
            st.session_state.tempo_especial = []
            st.session_state.tempo_especial_meses = 0
            st.experimental_rerun()
        
        # Calcular e exibir estatísticas
        total_meses_originais = sum(tempo["meses_originais"] for tempo in st.session_state.tempo_especial)
        total_meses_convertidos = sum(tempo["meses_convertidos"] for tempo in st.session_state.tempo_especial)
        acrescimo_meses = total_meses_convertidos - total_meses_originais
        
        st.markdown(f"""
        <div class='info-box'>
        <strong>Resumo:</strong><br>
        Total de períodos especiais: {len(st.session_state.tempo_especial)}<br>
        Total em meses (original): {total_meses_originais} meses ({total_meses_originais/12:.1f} anos)<br>
        Total em meses (convertido): {total_meses_convertidos} meses ({total_meses_convertidos/12:.1f} anos)<br>
        Acréscimo: {acrescimo_meses} meses ({acrescimo_meses/12:.1f} anos)
        </div>
        """, unsafe_allow_html=True)
    else:
        st.markdown("<div class='warning-box'>Nenhum tempo especial cadastrado.</div>", unsafe_allow_html=True)

elif page == "Regras de Aposentadoria":
    st.markdown("<h2 class='sub-header'>Regras de Aposentadoria</h2>", unsafe_allow_html=True)
    
    # Verificar se há dados do segurado
    if 'idade' not in st.session_state or 'sexo' not in st.session_state:
        st.markdown("<div class='warning-box'>Preencha primeiro os dados do segurado na seção 'Dados do Segurado'.</div>", unsafe_allow_html=True)
    elif len(st.session_state.contribuicoes) == 0:
        st.markdown("<div class='warning-box'>Cadastre pelo menos uma contribuição na seção 'Contribuições'.</div>", unsafe_allow_html=True)
    else:
        # Recuperar dados do segurado
        idade = st.session_state.idade
        sexo = st.session_state.sexo
        
        # Calcular tempo de contribuição
        tempo_contribuicao = calcular_tempo_contribuicao(
            st.session_state.contribuicoes, 
            st.session_state.tempo_especial_meses
        )
        
        # Guardar na sessão
        st.session_state.tempo_contribuicao = tempo_contribuicao
        
        # Exibir dados básicos
        st.markdown(f"""
        <div class='info-box'>
        <strong>Dados para Análise:</strong><br>
        Idade atual: {idade} anos<br>
        Sexo: {"Feminino" if sexo == "F" else "Masculino"}<br>
        Tempo de contribuição: {tempo_contribuicao[0]} anos e {tempo_contribuicao[1]} meses
        </div>
        """, unsafe_allow_html=True)
        
        # Verificar regras
        st.markdown("<h3>Análise de Regras</h3>", unsafe_allow_html=True)
        
        # Aposentadoria por idade
        elegivel_idade, msg_idade = verificar_aposentadoria_idade(idade, tempo_contribuicao, sexo)
        st.markdown(f"""
        <div class='{"success-box" if elegivel_idade else "warning-box"}'>
        <strong>Aposentadoria por Idade:</strong><br>
        {msg_idade}
        </div>
        """, unsafe_allow_html=True)
        
        # Aposentadoria por tempo de contribuição (regra de pontos)
        elegivel_pontos, msg_pontos = verificar_aposentadoria_tempo_contribuicao(idade, tempo_contribuicao, sexo)
        st.markdown(f"""
        <div class='{"success-box" if elegivel_pontos else "warning-box"}'>
        <strong>Aposentadoria por Tempo de Contribuição (Regra de Pontos):</strong><br>
        {msg_pontos}
        </div>
        """, unsafe_allow_html=True)
        
        # Aposentadoria por idade progressiva
        elegivel_progressiva, msg_progressiva = verificar_aposentadoria_idade_progressiva(idade, tempo_contribuicao, sexo)
        st.markdown(f"""
        <div class='{"success-box" if elegivel_progressiva else "warning-box"}'>
        <strong>Aposentadoria por Idade Progressiva:</strong><br>
        {msg_progressiva}
        </div>
        """, unsafe_allow_html=True)
        
        # Calcular médias salariais
        media_atual = calcular_media_salarial(st.session_state.contribuicoes, 'atual')
        media_antiga = calcular_media_salarial(st.session_state.contribuicoes, 'antiga')
        st.session_state.media_atual = media_atual
        st.session_state.media_antiga = media_antiga
        
        # Calcular RMI para ambas as regras
        rmi_atual = calcular_rmi(media_atual, tempo_contribuicao, 'atual', sexo)
        rmi_antiga = calcular_rmi(media_antiga, tempo_contribuicao, 'antiga', sexo)
        st.session_state.rmi_atual = rmi_atual
        st.session_state.rmi_antiga = rmi_antiga
        
        st.markdown(f"""
        <div class='info-box'>
        <strong>Resultados dos Cálculos:</strong><br>
        Média Salarial (Regra Atual): R$ {media_atual:.2f}<br>
        Média Salarial (Regra Antiga): R$ {media_antiga:.2f}<br>
        RMI (Regra Atual): R$ {rmi_atual:.2f}<br>
        RMI (Regra Antiga): R$ {rmi_antiga:.2f}
        </div>
        """, unsafe_allow_html=True)

elif page == "Resultados":
    st.markdown("<h2 class='sub-header'>Resultados</h2>", unsafe_allow_html=True)
    if 'rmi_atual' not in st.session_state or 'rmi_antiga' not in st.session_state:
        st.markdown("<div class='warning-box'>Realize os cálculos na seção 'Regras de Aposentadoria' para ver os resultados.</div>", unsafe_allow_html=True)
    else:
        st.markdown(f"""
        <div class='info-box'>
        <strong>Resultados dos Cálculos:</strong><br>
        Média Salarial (Regra Atual): R$ {st.session_state.media_atual:.2f}<br>
        Média Salarial (Regra Antiga): R$ {st.session_state.media_antiga:.2f}<br>
        Tempo de Contribuição: {st.session_state.tempo_contribuicao[0]} anos e {st.session_state.tempo_contribuicao[1]} meses<br>
        RMI (Regra Atual): R$ {st.session_state.rmi_atual:.2f}<br>
        RMI (Regra Antiga): R$ {st.session_state.rmi_antiga:.2f}
        </div>
        """, unsafe_allow_html=True)

        # Gráfico comparativo de RMI
        fig, ax = plt.subplots(figsize=(8, 4))
        ax.bar(["RMI Atual", "RMI Antiga"], [st.session_state.rmi_atual, st.session_state.rmi_antiga])
        ax.set_ylabel("Valor (R$)")
        ax.set_title("Comparativo RMI")
        st.pyplot(fig)

elif page == "Atrasados":
    st.markdown("<h2 class='sub-header'>Cálculo de Atrasados</h2>", unsafe_allow_html=True)
    if 'rmi_atual' not in st.session_state:
        st.markdown("<div class='warning-box'>Realize os cálculos na seção 'Regras de Aposentadoria' para calcular os benefícios.</div>", unsafe_allow_html=True)
    else:
        st.markdown("<div class='info-box'>Calcule os atrasados com base no RMI calculado.</div>", unsafe_allow_html=True)
        meses_atraso = st.number_input("Digite o número de meses de atraso", min_value=1, value=6, step=1)
        rmi_atrasados = st.session_state.rmi_atual * meses_atraso
        st.markdown(f"<div class='info-box'>Total de atrasados: R$ {rmi_atrasados:.2f} ({meses_atraso} meses)</div>", unsafe_allow_html=True)

elif page == "Exportar Relatório":
    st.markdown("<h2 class='sub-header'>Exportar Relatório</h2>", unsafe_allow_html=True)
    relatorio = export_to_excel()
    if relatorio is None:
        st.markdown("<div class='warning-box'>Não há dados suficientes para gerar o relatório. Certifique-se de preencher os dados do segurado e cadastrar contribuições.</div>", unsafe_allow_html=True)
    else:
        b64 = base64.b64encode(relatorio).decode()
        href = f'<a href="data:application/octet-stream;base64,{b64}" download="relatorio_previdenciario.xlsx">Clique aqui para baixar o relatório</a>'
        st.markdown(href, unsafe_allow_html=True)
