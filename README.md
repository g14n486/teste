from datetime import datetime, timedelta
import pandas as pd
import requests
import os
from io import StringIO

def update_anbima_data_filtered(file_path="renda_fixa.csv"):

    # Define URL com a data de ONTEM no formato YYMMDD
    yesterday = datetime.now() - timedelta(days=1) # Subtrai um dia para obter a data de ontem
    date_str = yesterday.strftime("%y%m%d") # Formato YYMMDD
    url = f"https://www.anbima.com.br/informacoes/merc-sec/arqs/ms{date_str}.txt"

    print(f"Tentando baixar dados de: {url}")

    # 2. Baixa os dados da URL
    try:
        response = requests.get(url)
        response.raise_for_status()  # Levanta uma exceção para erros HTTP (4xx ou 5xx)
        data = response.text

        print("Dados baixados com sucesso.")

    except requests.exceptions.RequestException as e:
        print(f"Erro ao baixar os dados da ANBIMA para {yesterday.strftime('%Y-%m-%d')}: {e}")
        return

    # 2. Formata dados em uma planilha pandas
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
                ]
        )

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

# Exemplo de uso:
if __name__ == "__main__":
    update_anbima_data_filtered()
