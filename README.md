# ==============================================================================
# 1. IMPORTAÇÕES E CONFIGURAÇÕES INICIAIS
# ==============================================================================
# Comentário: Importamos as bibliotecas necessárias.
# tensorflow e keras para construir a rede neural.
# pandas para manipulação de dados.
# numpy para operações numéricas.
# sklearn para pré-processamento (normalização) e métricas.
import tensorflow as tf
import pandas as pd
import numpy as np
from sklearn.preprocessing import MinMaxScaler
from sklearn.metrics import mean_squared_error

from tensorflow.keras.models import Sequential
from tensorflow.keras.layers import LSTM, Dense, Dropout, InputLayer
from tensorflow.keras.callbacks import ModelCheckpoint, EarlyStopping
from tensorflow.keras.optimizers import Adam
from tensorflow.keras.regularizers import l2

import matplotlib.pyplot as plt
import os

# ==============================================================================
# 2. SIMULAÇÃO DE DADOS (PARA FINS DE DEMONSTRAÇÃO)
# ==============================================================================
# Comentário: Esta função gera um DataFrame com dados fictícios para uma empresa.
# No mundo real, você substituiria esta função por uma que carrega seus dados
# reais de um arquivo CSV, banco de dados ou API.
def gerar_dados_empresa(ticker, n_releases=80):
    """
    Gera um DataFrame de dados simulados para uma empresa específica.

    Args:
        ticker (str): O identificador da empresa (ex: 'EMPRESA_A').
        n_releases (int): O número de releases trimestrais a serem gerados.

    Returns:
        pd.DataFrame: Um DataFrame com os dados simulados.
    """
    # Comentário: Cria um range de datas trimestrais.
    dates = pd.to_datetime(pd.date_range(start='2005-01-01', periods=n_releases, freq='Q'))
    
    # Comentário: Gera dados aleatórios, mas com alguma lógica.
    np.random.seed(hash(ticker) % (2**32 - 1)) # Garante que a mesma empresa sempre gere os mesmos dados
    
    # Sentimento do release: de -1 a 1
    sentimento = np.random.uniform(-1, 1, n_releases)
    
    # Variáveis macroeconômicas
    pib_variacao = np.random.uniform(-0.01, 0.03, n_releases) + np.sin(np.arange(n_releases) / 10) * 0.01
    taxa_juros = np.random.uniform(0.02, 0.14, n_releases)
    
    # Dados financeiros (esperado vs. realizado)
    receita_esperada = np.linspace(1000, 5000, n_releases) * (1 + np.random.uniform(-0.05, 0.05, n_releases))
    # A receita realizada é baseada na esperada, com um "choque" aleatório
    receita_realizada = receita_esperada * (1 + np.random.normal(0, 0.1, n_releases))
    
    # Variável Alvo: Retorno da ação no dia seguinte
    # Comentário: O retorno é influenciado pelo sentimento, choque de receita e juros.
    choque_receita = (receita_realizada - receita_esperada) / receita_esperada
    retorno_acao = (sentimento * 0.2 + choque_receita * 0.5 - taxa_juros * 0.1 + np.random.normal(0, 0.05, n_releases))
    
    df = pd.DataFrame({
        'Data': dates,
        'Sentimento_Release': sentimento,
        'PIB_Variacao': pib_variacao,
        'Taxa_Juros': taxa_juros,
        'Choque_Receita': choque_receita,
        'Retorno_Acao_Dia_Seguinte': retorno_acao
    })
    
    df.set_index('Data', inplace=True)
    return df

# ==============================================================================
# 3. PREPARAÇÃO E PRÉ-PROCESSAMENTO DOS DADOS
# ==============================================================================
# Comentário: Esta função transforma o DataFrame em sequências (janelas) que
# o modelo LSTM pode usar para treinamento.
def df_to_X_y(df, window_size=4):
    """
    Converte um DataFrame de série temporal em amostras de entrada (X) e saída (y).

    Args:
        df (pd.DataFrame): DataFrame com as features e o target.
        window_size (int): O número de passos de tempo passados a serem usados para prever o próximo passo.

    Returns:
        tuple: Um tuple contendo os arrays numpy para X e y.
    """
    df_as_np = df.to_numpy()
    X = []
    y = []
    # Comentário: Itera sobre os dados para criar as janelas.
    for i in range(len(df_as_np) - window_size):
        # A janela de features
        row = df_as_np[i:i + window_size]
        X.append(row)
        # O label é o valor da variável alvo no final da janela
        label = df_as_np[i + window_size][-1] # O target é a última coluna
        y.append(label)
    return np.array(X), np.array(y)

# ==============================================================================
# 4. CONSTRUÇÃO E TREINAMENTO DO MODELO
# ==============================================================================
def construir_e_treinar_modelo(X_train, y_train, X_val, y_val, window_size, n_features):
    """
    Constrói, compila e treina o modelo LSTM.

    Args:
        X_train, y_train: Dados de treinamento.
        X_val, y_val: Dados de validação.
        window_size (int): Tamanho da janela de tempo.
        n_features (int): Número de variáveis de entrada.

    Returns:
        tensorflow.keras.Model: O modelo treinado.
    """
    # --- Construção do Modelo ---
    # Comentário: Usamos a API Sequencial do Keras para empilhar as camadas.
    model = Sequential()
    
    # Camada de Entrada: Define o formato dos dados de entrada (janelas x features)
    model.add(InputLayer((window_size, n_features)))
    
    # Camada LSTM: A camada principal que aprende os padrões sequenciais.
    # 64 é o número de unidades (neurônios) na camada.
    # return_sequences=False porque esta é a última camada recorrente.
    # kernel_regularizer=l2(0.01) é uma técnica para prevenir overfitting, penalizando pesos grandes.
    model.add(LSTM(64, kernel_regularizer=l2(0.01)))

    # Camada Dropout: Outra técnica contra overfitting.
    # "Desliga" aleatoriamente 20% dos neurônios durante o treinamento para
    # evitar que o modelo se torne muito dependente de alguns neurônios específicos.
    model.add(Dropout(0.2))
    
    # Camada Densa: Uma camada de conexão total para processamento adicional.
    model.add(Dense(8, activation='relu'))
    
    # Camada de Saída: Produz a previsão final.
    # 1 neurônio porque estamos prevendo um único valor (Retorno_Acao_Dia_Seguinte).
    # 'linear' é a ativação padrão para problemas de regressão.
    model.add(Dense(1, activation='linear'))
    
    model.summary()
    
    # --- Compilação do Modelo ---
    # Comentário: Preparamos o modelo para o treinamento.
    # loss: Função de erro a ser minimizada (MeanSquaredError para regressão).
    # optimizer: Algoritmo para atualizar os pesos do modelo (Adam é uma escolha robusta).
    # metrics: Métricas para monitorar durante o treinamento.
    model.compile(loss='mse', optimizer=Adam(learning_rate=0.001), metrics=['mean_absolute_error'])
    
    # --- Callbacks para o Treinamento ---
    # ModelCheckpoint: Salva o melhor modelo encontrado durante o treinamento (baseado na perda de validação).
    cp = ModelCheckpoint('best_model.keras', save_best_only=True, monitor='val_loss', mode='min')
    
    # EarlyStopping: Para o treinamento se a perda de validação não melhorar por 'patience' épocas.
    # Isso evita o overfitting e economiza tempo de treinamento.
    es = EarlyStopping(monitor='val_loss', patience=15, restore_best_weights=True)
    
    # --- Treinamento ---
    # Comentário: Ajustamos o modelo aos dados de treinamento.
    # epochs: Número máximo de vezes que o modelo verá todo o conjunto de dados de treinamento.
    # batch_size: Número de amostras a serem processadas antes de atualizar os pesos.
    model.fit(X_train, y_train, validation_data=(X_val, y_val), epochs=100, batch_size=16, callbacks=[cp, es])
    
    return model

# ==============================================================================
# 5. FUNÇÃO PRINCIPAL PARA ANÁLISE DA EMPRESA
# ==============================================================================
def analisar_empresa(ticker, window_size=4):
    """
    Função principal que executa todo o pipeline para uma empresa específica.
    """
    print(f"--- Iniciando análise para a empresa: {ticker} ---")
    
    # 1. Obter Dados
    # Comentário: Aqui chamamos a função para gerar/carregar os dados.
    df = gerar_dados_empresa(ticker)
    print("\nDados brutos gerados (primeiras 5 linhas):")
    print(df.head())
    
    # 2. Definir Features e Target
    # Comentário: Separamos as variáveis de entrada (features) da variável de saída (target).
    features = ['Sentimento_Release', 'PIB_Variacao', 'Taxa_Juros', 'Choque_Receita']
    target = 'Retorno_Acao_Dia_Seguinte'
    
    # 3. Normalização dos Dados
    # Comentário: Normalizamos os dados para a escala [0, 1]. Isso é crucial para
    # o bom desempenho de redes neurais.
    scaler_x = MinMaxScaler()
    scaler_y = MinMaxScaler()
    
    df[features] = scaler_x.fit_transform(df[features])
    df[[target]] = scaler_y.fit_transform(df[[target]])
    
    # 4. Criação das Janelas Temporais
    X, y = df_to_X_y(df[features + [target]], window_size=window_size)
    print(f"\nFormato dos dados em janela: X.shape={X.shape}, y.shape={y.shape}")
    
    # 5. Divisão em Treino, Validação e Teste
    # Comentário: Dividimos os dados cronologicamente para evitar vazamento de dados do futuro.
    # 70% para treino, 15% para validação, 15% para teste.
    n = len(X)
    X_train, y_train = X[:int(n*0.7)], y[:int(n*0.7)]
    X_val, y_val = X[int(n*0.7):int(n*0.85)], y[int(n*0.7):int(n*0.85)]
    X_test, y_test = X[int(n*0.85):], y[int(n*0.85):]
    
    print(f"Tamanhos dos conjuntos: Treino={len(X_train)}, Validação={len(X_val)}, Teste={len(X_test)}")
    
    # 6. Construção e Treinamento do Modelo
    model = construir_e_treinar_modelo(X_train, y_train, X_val, y_val, window_size, len(features) + 1)
    
    # 7. Avaliação no Conjunto de Teste
    # Comentário: Carregamos o melhor modelo salvo pelo ModelCheckpoint para garantir
    # que estamos usando a melhor versão, não necessariamente a última.
    best_model = tf.keras.models.load_model('best_model.keras')
    
    test_predictions_scaled = best_model.predict(X_test)
    
    # 8. Pós-processamento: Inverter a normalização para interpretar os resultados.
    # Comentário: As previsões estão na escala [0, 1], precisamos revertê-las para a
    # escala original de retorno de ação.
    test_predictions = scaler_y.inverse_transform(test_predictions_scaled)
    y_test_original = scaler_y.inverse_transform(y_test.reshape(-1, 1))
    
    # 9. Visualização dos Resultados
    plt.figure(figsize=(14, 6))
    plt.title(f'Previsão vs. Real - Retorno da Ação para {ticker}')
    plt.plot(y_test_original, label='Valores Reais', color='blue', marker='o', linestyle='None')
    plt.plot(test_predictions, label='Previsões do Modelo', color='red', marker='x', linestyle='None')
    plt.xlabel('Release (no conjunto de teste)')
    plt.ylabel('Retorno da Ação')
    plt.legend()
    plt.grid(True)
    plt.show()
    
    # 10. Métrica de Erro
    rmse = np.sqrt(mean_squared_error(y_test_original, test_predictions))
    print(f"\nAnálise para {ticker} concluída.")
    print(f"Raiz do Erro Quadrático Médio (RMSE) no conjunto de teste: {rmse:.4f}")
    
    # Limpa o arquivo do modelo para a próxima execução
    if os.path.exists('best_model.keras'):
        os.remove('best_model.keras')

# ==============================================================================
# --- PONTO DE ENTRADA DO SCRIPT ---
# ==============================================================================
if __name__ == '__main__':
    # Comentário: Para analisar uma empresa, basta chamar a função com o ticker desejado.
    # Você pode adicionar mais tickers e analisar um por um.
    
    # Analisando a primeira empresa
    analisar_empresa(ticker='EMPRESA_A')
    
    print("\n" + "="*80 + "\n")
    
    # Analisando uma segunda empresa (os dados serão diferentes devido à seed)
    analisar_empresa(ticker='EMPRESA_B', window_size=5)
