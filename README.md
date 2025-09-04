from datetime import datetime, timedelta
import pandas as pd
import requests
import os
from io import StringIO

# Atualiza base de dados com 
def ambima_rf(file_path="renda_fixa.csv"):

    # Define URL com a data de ONTEM no formato YYMMDD
    yesterday = datetime.now() - timedelta(days=1) # Subtrai um dia para obter a data de ontem
    web_date = yesterday.strftime("%y%m%d") # Formato YYMMDD
    check_date = yesterday.strftime("%Y-%m-%d") # Formato YYMMDD

    url = f"https://www.anbima.com.br/informacoes/merc-sec/arqs/ms{web_date}.txt"

    # Verifica se o csv está atualizado
    try:
        # Tenta ler o arquivo CSV para verificação
        existing_df = pd.read_csv(file_path, encoding='utf-8')

        # Verifica se o df está atualizado com os dados mais recentes
        if check_date in existing_df['Data Referencia'].values:
            print(f"Os dados para {yesterday.strftime('%Y-%m-%d')} já existem em '{file_path}'. Não será necessário baixar novamente.")
            return

        else:
            print(f"Dados para {yesterday.strftime('%Y-%m-%d')} não encontrados no arquivo. Prosseguindo com o download.")

    except Exception as e:
        # Caso o arquivo exista, mas ocorra um erro ao ler
        print(f"Erro ao ler o arquivo '{file_path}' para verificação: {e}")
        print("Prosseguindo com o download para evitar duplicação.")

    print(f"Tentando baixar dados de: {url}")

    # Define base de dados com valores obtidos
    try:
        response = requests.get(url)
        response.raise_for_status()  # Levanta uma exceção para erros HTTP (4xx ou 5xx)
        data = response.text

        print("Dados baixados com sucesso.")

    except requests.exceptions.RequestException as e:
        print(f"Erro ao baixar os dados da ANBIMA para {yesterday.strftime('%Y%m%d')}: {e}")
        return

    # Formata dados em um df pandas
    try:
        df = pd.read_csv(StringIO(data),
            sep='@',
            encoding='latin1',
            skiprows=1,
            usecols=['Titulo',
            'Data Referencia',
            'Codigo SELIC',
            'Data Base/Emissao',
            'Data Vencimento',
            'Tx. Compra',
            'Tx. Venda',
            'Tx. Indicativas',
            'PU',
            'Desvio padrao',
            'Interv. Ind. Inf. (D0)',
            'Interv. Ind. Sup. (D0)',
            'Interv. Ind. Inf. (D+1)',
            'Interv. Ind. Sup. (D+1)'
            ],
            parse_dates=['Data Referencia','Data Base/Emissao','Data Vencimento'],
        )

        # Display de dados
        print("---------------------------------------------------------")
        print("Dados parseados com sucesso para um DataFrame.")
        print(df.head())
        print("---------------------------------------------------------")

    except Exception as e:
        print(f"Erro ao parsear os dados para o DataFrame pandas: {e}")
        print("Amostra dos dados brutos (primeiros 500 caracteres):")
        print(data[:500])
        print("Por favor, verifique o formato do arquivo da ANBIMA e ajuste a lógica de parsing se necessário.")
        return


    if not os.path.exists(file_path):
        # Primeira vez, inclui cabeçalhos
        df.to_csv(file_path, index=False, header=True, encoding='utf-8')
        print(f"Novo arquivo CSV '{file_path}' criado com cabeçalhos.")
    else:
        # Próximas vezes, anexa sem cabeçalhos
        df.to_csv(file_path, index=False, header=False, mode='a', encoding='utf-8')
        print(f"Dados anexados a '{file_path}' sem cabeçalhos.")

if __name__ == "__main__":
    ambima_rf()
