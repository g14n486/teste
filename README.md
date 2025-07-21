# ==============================================================================
# 1. IMPORTAÇÕES E CONFIGURAÇÕES INICIAIS
# ==============================================================================
# Comentário: Importamos as bibliotecas necessárias.
# tensorflow e keras para construir a rede neural.
# pandas para manipulação de dados e numpy para operações numéricas.
# sklearn para pré-processamento e métricas.
# matplotlib para visualização e os para manipulação de arquivos.
import tensorflow as tf
import pandas as pd
import numpy as np
from sklearn.preprocessing import MinMaxScaler
from sklearn.metrics import mean_squared_error

from tensorflow.keras.models import Sequential, load_model
from tensorflow.keras.layers import LSTM, Dense, Dropout, InputLayer
from tensorflow.keras.callbacks import ModelCheckpoint, EarlyStopping
from tensorflow.keras.optimizers import Adam
from tensorflow.keras.regularizers import l2

import matplotlib.pyplot as plt
import os
import warnings

# Suprime avisos de formatação do TensorFlow para um output mais limpo
warnings.filterwarnings("ignore", category=UserWarning)
tf.get_logger().setLevel('ERROR')


# ==============================================================================
# 2. FUNÇÕES AUXILIARES (DADOS, PREPARAÇÃO, MODELO)
# ==============================================================================

def gerar_dados_empresa(ticker, n_releases=80):
    """
    Comentário: Gera um DataFrame de dados simulados para uma empresa. No mundo real,
    você substituiria esta função por uma que carrega seus dados processados.
    """
    np.random.seed(hash(ticker) % (2**32 - 1))
    dates = pd.to_datetime(pd.date_range(start='2005-01-01', periods=n_releases, freq='Q'))
    sentimento = np.random.uniform(-1, 1, n_releases)
    pib_variacao = np.random.uniform(-0.01, 0.03, n_releases) + np.sin(np.arange(n_releases) / 10) * 0.01
    taxa_juros = np.random.uniform(0.02, 0.14, n_releases)
    receita_esperada = np.linspace(1000, 5000, n_releases) * (1 + np.random.uniform(-0.05, 0.05, n_releases))
    receita_realizada = receita_esperada * (1 + np.random.normal(0, 0.1, n_releases))
    choque_receita = (receita_realizada - receita_esperada) / receita_esperada
    retorno_acao = (sentimento * 0.2 + choque_receita * 0.5 - taxa_juros * 0.1 + np.random.normal(0, 0.05, n_releases))
    df = pd.DataFrame({
        'Data': dates, 'Sentimento_Release': sentimento, 'PIB_Variacao': pib_variacao,
        'Taxa_Juros': taxa_juros, 'Choque_Receita': choque_receita,
        'Retorno_Acao_Dia_Seguinte': retorno_acao
    })
    df.set_index('Data', inplace=True)
    return df

def df_to_X_y(df, window_size=4):
    """
    Comentário: Converte um DataFrame em janelas de sequências (X) e o valor alvo (y).
    Esta é a estrutura de dados que a LSTM espera.
    """
    df_as_np = df.to_numpy()
    X, y = [], []
    for i in range(len(df_as_np) - window_size):
        X.append(df_as_np[i:i + window_size])
        y.append(df_as_np[i + window_size][-1]) # O alvo é a última coluna do passo seguinte
    return np.array(X), np.array(y)

def construir_modelo_lstm(window_size, n_features):
    """
    Comentário: Constrói a arquitetura do modelo LSTM.
    Inclui Dropout e Regularização L2 para combater o overfitting.
    """
    model = Sequential([
        InputLayer((window_size, n_features)),
        LSTM(64, kernel_regularizer=l2(0.01)),
        Dropout(0.2),
        Dense(8, activation='relu'),
        Dense(1, activation='linear')
    ])
    return model

# ==============================================================================
# 3. FUNÇÃO PRINCIPAL DE ANÁLISE
# ==============================================================================
def analisar_empresa(ticker):
    """
    Função principal que executa todo o pipeline:
    1. Carrega e prepara os dados.
    2. Busca pela melhor janela temporal.
    3. Treina o modelo final com a melhor janela.
    4. Gera e exibe as previsões como um range.
    """
    print(f"===========================================================")
    print(f"=== INICIANDO ANÁLISE COMPLETA PARA A EMPRESA: {ticker} ===")
    print(f"===========================================================")

    # ETAPA 1: CARREGAR E PREPARAR OS DADOS
    # --------------------------------------------------------------------------
    print("\n--- ETAPA 1: Carregando e Normalizando os Dados ---")
    df_bruto = gerar_dados_empresa(ticker)
    features = ['Sentimento_Release', 'PIB_Variacao', 'Taxa_Juros', 'Choque_Receita', 'Retorno_Acao_Dia_Seguinte']
    df_proc = df_bruto[features]

    # Normalizar os dados para o intervalo [0, 1]
    scaler = MinMaxScaler()
    df_norm = pd.DataFrame(scaler.fit_transform(df_proc), columns=features, index=df_proc.index)
    
    # Guardar o scaler da variável alvo para reverter a transformação depois
    target_col_index = features.index('Retorno_Acao_Dia_Seguinte')
    scaler_y = MinMaxScaler()
    scaler_y.min_, scaler_y.scale_ = scaler.min_[target_col_index], scaler.scale_[target_col_index]
    print("Dados normalizados com sucesso.")

    # ETAPA 2: BUSCA PELA JANELA TEMPORAL IDEAL
    # --------------------------------------------------------------------------
    print("\n--- ETAPA 2: Buscando a Janela Temporal Ideal ---")
    window_candidates = [2, 3, 4, 6, 8, 12]
    results = {}

    for w in window_candidates:
        print(f"Testando janela de tamanho: {w}...", end="")
        X, y = df_to_X_y(df_norm, window_size=w)
        
        if len(X) < 30:
            print(" Dataset muito pequeno, pulando.")
            continue
            
        n = len(X)
        X_train, y_train = X[:int(n*0.7)], y[:int(n*0.7)]
        X_val, y_val = X[int(n*0.7):], y[int(n*0.7):] # Usaremos o resto para validação nesta busca

        model_temp = construir_modelo_lstm(w, len(features))
        model_temp.compile(loss='mse', optimizer=Adam(learning_rate=0.001))
        
        # Treina silenciosamente com early stopping para ser rápido
        model_temp.fit(X_train, y_train, validation_data=(X_val, y_val),
                       epochs=50, verbose=0, callbacks=[EarlyStopping(monitor='val_loss', patience=5)])
        
        val_loss = model_temp.evaluate(X_val, y_val, verbose=0)
        results[w] = val_loss
        print(f" Erro de Validação (MSE): {val_loss:.5f}")

    best_window = min(results, key=results.get)
    print(f"\n>>> Janela ideal encontrada: {best_window} trimestres (menor erro de validação).")

    # ETAPA 3: TREINAMENTO DO MODELO FINAL
    # --------------------------------------------------------------------------
    print("\n--- ETAPA 3: Treinando o Modelo Final com a Melhor Janela ---")
    X, y = df_to_X_y(df_norm, window_size=best_window)
    n = len(X)
    X_train, y_train = X[:int(n*0.7)], y[:int(n*0.7)]
    X_val, y_val = X[int(n*0.7):int(n*0.85)], y[int(n*0.7):int(n*0.85)]
    X_test, y_test = X[int(n*0.85):], y[int(n*0.85):]
    
    model_final = construir_modelo_lstm(best_window, len(features))
    model_final.compile(loss='mse', optimizer=Adam(learning_rate=0.001), metrics=['mean_absolute_error'])

    # Callbacks para o treino final
    checkpoint_path = 'best_model.keras'
    cp = ModelCheckpoint(checkpoint_path, save_best_only=True, monitor='val_loss', mode='min', verbose=0)
    es = EarlyStopping(monitor='val_loss', patience=15, restore_best_weights=True)

    model_final.fit(X_train, y_train, validation_data=(X_val, y_val), epochs=100, callbacks=[cp, es], verbose=1)

    # Carrega o melhor modelo salvo pelo checkpoint
    best_model = load_model(checkpoint_path)
    print("Modelo final treinado e melhor versão carregada.")

    # ETAPA 4: CÁLCULO DA INCERTEZA DO MODELO
    # --------------------------------------------------------------------------
    print("\n--- ETAPA 4: Calculando a Incerteza do Modelo (Volatilidade dos Erros) ---")
    val_preds_scaled = best_model.predict(X_val)
    val_preds = scaler_y.inverse_transform(val_preds_scaled)
    y_val_original = scaler_y.inverse_transform(y_val.reshape(-1, 1))

    erros_de_validacao = y_val_original - val_preds
    volatilidade_dos_erros = np.std(erros_de_validacao)
    print(f"Desvio padrão dos erros de validação: {volatilidade_dos_erros:.4f}")

    # ETAPA 5: GERAÇÃO DO RANGE DE PREVISÃO
    # --------------------------------------------------------------------------
    print("\n--- ETAPA 5: Gerando Previsões com Range para o Conjunto de Teste ---")
    test_preds_scaled = best_model.predict(X_test)
    
    # Desnormaliza a previsão central
    predicoes_centrais = scaler_y.inverse_transform(test_preds_scaled)
    y_test_original = scaler_y.inverse_transform(y_test.reshape(-1, 1))

    # Calcula o intervalo de confiança (95% -> fator 1.96)
    fator_confianca = 1.96
    margem_de_erro = fator_confianca * volatilidade_dos_erros
    limite_inferior = predicoes_centrais - margem_de_erro
    limite_superior = predicoes_centrais + margem_de_erro

    # ETAPA 6: OUTPUT E VISUALIZAÇÃO
    # --------------------------------------------------------------------------
    print("\n--- ETAPA 6: Resultados Finais ---")
    
    # Visualização
    plt.figure(figsize=(15, 7))
    plt.title(f'Previsão com Range de Confiança (95%) para {ticker}')
    plt.plot(y_test_original, label='Valores Reais', color='blue', marker='o', linestyle='None', zorder=5)
    plt.plot(predicoes_centrais, label='Previsão Central', color='red', linestyle='--', zorder=4)
    plt.fill_between(range(len(predicoes_centrais)),
                     limite_inferior.flatten(),
                     limite_superior.flatten(),
                     color='red', alpha=0.2, label='Range de Previsão')
    plt.xlabel('Release (no conjunto de teste)')
    plt.ylabel('Retorno da Ação')
    plt.legend()
    plt.grid(True)
    plt.show()

    # Output da previsão para o próximo período (usando o primeiro ponto do teste como exemplo)
    print("\n>>> PREVISÃO PARA O PRÓXIMO RELEASE <<<")
    print(f"Com base nos dados mais recentes, o modelo prevê que o retorno da ação terá:")
    print(f"  - Previsão Central (Mais Provável): {predicoes_centrais[0][0]:.4f}")
    print(f"  - Range Esperado (com 95% de confiança): de {limite_inferior[0][0]:.4f} a {limite_superior[0][0]:.4f}")

    # Limpa o arquivo do modelo salvo
    if os.path.exists(checkpoint_path):
        os.remove(checkpoint_path)
    
    print(f"\nAnálise para {ticker} concluída.")


# ==============================================================================
# --- PONTO DE ENTRADA DO SCRIPT ---
# ==============================================================================
if __name__ == '__main__':
    # Para analisar uma empresa, basta chamar a função com o ticker desejado.
    analisar_empresa(ticker='PETROBRAS_PN')
    
    # Você pode descomentar a linha abaixo para rodar a análise para outra empresa
    # analisar_empresa(ticker='VALE_ON')
