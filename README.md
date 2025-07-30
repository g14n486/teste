Com certeza! Vamos criar um código que busca o Fato Relevante mais recente da WEGE3 na CVM e baixa o PDF correspondente.
Para isso, utilizaremos a biblioteca brfinance, que se mostra uma das mais adequadas para interagir com os dados da CVM no Python.
Passo a passo:
 * Instalar as bibliotecas: Se você ainda não as tem, instale brfinance e requests.
   pip install brfinance requests pandas

 * Obter o código CVM da WEGE3: Precisamos do código CVM da WEGE3 para fazer a busca. Embora possamos buscar por "WEGE3", é mais robusto usar o código CVM diretamente. A brfinance oferece um método para mapear o código CVM a partir do nome da empresa.
 * Definir a categoria "Fato Relevante": Identificar a categoria correta para "Fato Relevante" nos dados da CVM.
 * Buscar o Fato Relevante mais recente: Fazer uma busca reversa por data para obter o mais recente.
 * Extrair o link do PDF: Dos resultados da busca, extrair o URL do documento.
 * Baixar o PDF: Usar a biblioteca requests para fazer o download do PDF.
<!-- end list -->
import requests
import pandas as pd
from brfinance import CVMAsyncBackend
from datetime import date, timedelta
import os
import asyncio

# Função assíncrona para buscar e baixar o PDF
async def download_latest_wege3_press_release():
    print("Iniciando a busca pelo Fato Relevante mais recente da WEGE3...")
    
    cvm_httpclient = CVMAsyncBackend()

    try:
        # 1. Obter o código CVM da WEGE3
        # Os códigos CVM são um dicionário onde a chave é o código CVM e o valor é o nome da empresa
        cvm_codes = await cvm_httpclient.get_cvm_codes()
        
        wege3_cvm_code = None
        for code, company_name in cvm_codes.items():
            if "WEG S.A." in company_name.upper(): # O nome oficial pode ser "WEG S.A."
                wege3_cvm_code = code
                break
        
        if not wege3_cvm_code:
            print("Erro: Não foi possível encontrar o código CVM para WEG S.A. (WEGE3).")
            print("Verifique se o nome da empresa está correto ou ajuste a busca.")
            return

        print(f"Código CVM da WEGE3 (WEG S.A.) encontrado: {wege3_cvm_code}")

        # 2. Obter as categorias de documentos e encontrar "Fato Relevante"
        categories = await cvm_httpclient.get_consulta_externa_cvm_categories()
        fato_relevante_category_code = None
        for code, name in categories.items():
            if "fato relevante" in name.lower():
                fato_relevante_category_code = code
                break

        if not fato_relevante_category_code:
            print("Erro: Não foi possível encontrar a categoria 'Fato Relevante'.")
            print("Verifique as categorias disponíveis na documentação da CVM ou da brfinance.")
            return

        print(f"Código da categoria 'Fato Relevante' encontrado: {fato_relevante_category_code}")

        # 3. Definir o período de busca (últimos 365 dias, por exemplo, para garantir o mais recente)
        # É bom ter uma janela ampla e depois ordenar para pegar o mais recente.
        end_date = date.today()
        start_date = end_date - timedelta(days=365 * 2) # Buscar nos últimos 2 anos para ter certeza de pegar o mais recente
        
        print(f"Buscando Fatos Relevantes para WEGE3 entre {start_date} e {end_date}...")

        search_results = await cvm_httpclient.get_consulta_externa_cvm_results(
            start_date=start_date,
            end_date=end_date,
            cvm_codes=[wege3_cvm_code],
            category=[fato_relevante_category_code]
        )

        if search_results.empty:
            print("Nenhum Fato Relevante encontrado para WEGE3 no período especificado.")
            return

        # 4. Encontrar o Fato Relevante mais recente
        # A coluna 'data_referencia' geralmente contém a data do documento.
        # Ordenar por data em ordem decrescente para pegar o mais recente.
        latest_press_release = search_results.sort_values(by='data_referencia', ascending=False).iloc[0]

        link_documento = latest_press_release['link_documento']
        assunto = latest_press_release['assunto']
        data_referencia = latest_press_release['data_referencia'].strftime('%Y-%m-%d')
        
        print(f"\nFato Relevante mais recente encontrado:")
        print(f"  Data: {data_referencia}")
        print(f"  Assunto: {assunto}")
        print(f"  Link: {link_documento}")

        # 5. Baixar o PDF
        if link_documento and link_documento.endswith('.pdf'):
            try:
                # Cria um nome de arquivo seguro e descritivo
                # Remove caracteres inválidos para nome de arquivo
                safe_assunto = "".join([c for c in assunto if c.isalnum() or c in (' ', '.', '_')]).strip()
                safe_assunto = safe_assunto.replace(' ', '_')
                filename = f"WEGE3_FatoRelevante_{data_referencia}_{safe_assunto}.pdf"
                
                print(f"Tentando baixar o PDF para: {filename}")

                response = requests.get(link_documento, stream=True)
                response.raise_for_status() # Lança um erro para status de erro HTTP

                with open(filename, 'wb') as pdf_file:
                    for chunk in response.iter_content(chunk_size=8192):
                        pdf_file.write(chunk)
                
                print(f"PDF baixado com sucesso: {filename}")
            except requests.exceptions.RequestException as e:
                print(f"Erro ao baixar o PDF do link {link_documento}: {e}")
            except Exception as e:
                print(f"Erro ao salvar o PDF: {e}")
        else:
            print(f"O documento não é um PDF ou o link não foi encontrado/termina em .pdf: {link_documento}")

    except Exception as e:
        print(f"Ocorreu um erro: {e}")
        print("Certifique-se de que o acesso à internet está funcionando e que as bibliotecas estão atualizadas.")

# Para executar a função assíncrona
if __name__ == "__main__":
    asyncio.run(download_latest_wege3_press_release())

Como usar o código:
 * Salve o código: Salve o código acima em um arquivo Python (por exemplo, download_wege3.py).
 * Execute no terminal: Abra seu terminal ou prompt de comando e navegue até o diretório onde você salvou o arquivo. Em seguida, execute:
   python download_wege3.py

O que o código faz:
 * Importações: Importa as bibliotecas necessárias (requests para download, pandas para manipulação de dados, brfinance para acesso à CVM, datetime para datas, os para caminhos de arquivos e asyncio para funções assíncronas).
 * download_latest_wege3_press_release() (função assíncrona):
   * Instancia CVMAsyncBackend(): Cria um cliente para interagir com a API da CVM.
   * Busca o código CVM da WEG S.A.: Itera sobre os códigos CVM para encontrar o da WEG S.A. (que corresponde à WEGE3).
   * Identifica a categoria "Fato Relevante": Procura a categoria de documento que corresponde a "Fato Relevante".
   * Define o período de busca: Busca documentos dos últimos 2 anos para garantir que o mais recente seja encontrado.
   * Consulta à CVM: Usa get_consulta_externa_cvm_results para buscar os Fatos Relevantes da WEGE3 no período.
   * Encontra o mais recente: Ordena os resultados pela coluna data_referencia em ordem decrescente e pega o primeiro item, que será o mais recente.
   * Extrai informações: Obtém o link_documento, assunto e data_referencia do Fato Relevante mais recente.
   * Baixa o PDF: Se o link_documento existir e terminar em .pdf, ele usa requests para baixar o arquivo e salvá-lo com um nome descritivo (WEGE3_FatoRelevante_DATA_ASSUNTO.pdf) no mesmo diretório do script.
 * Execução Assíncrona: O bloco if __name__ == "__main__": garante que a função assíncrona seja executada corretamente.
Este script é uma solução robusta para o seu objetivo!
