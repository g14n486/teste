import requests
from bs4 import BeautifulSoup
import os
import re
from zipfile import ZipFile
from io import BytesIO
from datetime import datetime

def baixar_ultimo_release_rad_cvm(nome_empresa):
    """
    Busca e baixa o último documento ITR (release trimestral) de uma empresa
    diretamente do novo portal RAD da CVM.
    """
    print(f"Iniciando busca no portal RAD da CVM para: '{nome_empresa}'")

    url_rad = "https://www.rad.cvm.gov.br/ENET/frmConsultaExternaCVM.aspx"
    headers = {
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36'
    }

    session = requests.Session()
    session.headers.update(headers)

    try:
        # 1. Acessar a página para obter os dados do formulário
        print("Acessando formulário de consulta...")
        response_form = session.get(url_rad)
        response_form.raise_for_status()
        soup_form = BeautifulSoup(response_form.text, 'html.parser')

        viewstate = soup_form.find('input', {'name': '__VIEWSTATE'})['value']
        viewstategenerator = soup_form.find('input', {'name': '__VIEWSTATEGENERATOR'})['value']
        eventvalidation = soup_form.find('input', {'name': '__EVENTVALIDATION'})['value']

        # 2. Enviar a requisição POST para buscar os documentos
        # Primeiro, buscamos pelo nome da empresa para obter seu Código CVM
        payload_busca_empresa = {
            '__EVENTTARGET': 'txtNomeEmpresa',
            '__EVENTARGUMENT': '',
            '__LASTFOCUS': '',
            '__VIEWSTATE': viewstate,
            '__VIEWSTATEGENERATOR': viewstategenerator,
            '__EVENTVALIDATION': eventvalidation,
            'txtChave': '',
            'txtNomeEmpresa': nome_empresa,
            'txtCNPJ': '',
            'txtAssunto': '',
            'txtDataIni': '',
            'txtDataFim': '',
            'hdnCodCVM': '',
        }

        print(f"Buscando código CVM para '{nome_empresa}'...")
        response_busca = session.post(url_rad, data=payload_busca_empresa)
        response_busca.raise_for_status()
        soup_busca = BeautifulSoup(response_busca.text, 'html.parser')

        cod_cvm_input = soup_busca.find('input', {'id': 'hdnCodCVM'})
        if not cod_cvm_input or not cod_cvm_input.get('value'):
            print(f"Não foi possível encontrar o Código CVM para '{nome_empresa}'. Tente o nome exato.")
            return

        cod_cvm = cod_cvm_input['value']
        print(f"Código CVM encontrado: {cod_cvm}")

        # 3. Agora, fazemos a busca final pelos documentos ITR
        # Atualizamos o payload com o código CVM e os filtros corretos
        viewstate = soup_busca.find('input', {'name': '__VIEWSTATE'})['value']
        eventvalidation = soup_busca.find('input', {'name': '__EVENTVALIDATION'})['value']
        
        payload_consulta_docs = {
            '__EVENTTARGET': '',
            '__EVENTARGUMENT': '',
            '__LASTFOCUS': '',
            '__VIEWSTATE': viewstate,
            '__VIEWSTATEGENERATOR': viewstategenerator,
            '__EVENTVALIDATION': eventvalidation,
            'txtChave': '',
            'txtNomeEmpresa': nome_empresa,
            'txtCNPJ': '',
            'txtAssunto': 'itr',  # Filtra por ITR
            'txtDataIni': '01/01/2020', # Data inicial para garantir que pegamos os mais recentes
            'txtDataFim': datetime.now().strftime('%d/%m/%Y'),
            'hdnCodCVM': cod_cvm,
            'btnConsulta': 'btConsultar',
        }

        print("Consultando documentos ITR...")
        response_docs = session.post(url_rad, data=payload_consulta_docs)
        response_docs.raise_for_status()
        soup_docs = BeautifulSoup(response_docs.text, 'html.parser')

        # 4. Processar a tabela de resultados
        grid_docs = soup_docs.find('table', {'id': 'grdDocumentos'})
        if not grid_docs or not grid_docs.find('a', {'id': re.compile(r'hypDownload')}):
            print(f"Nenhum documento do tipo ITR encontrado para '{nome_empresa}' no portal RAD.")
            return

        # O primeiro link de download na tabela é o mais recente
        link_download = grid_docs.find('a', {'id': re.compile(r'hypDownload')})
        if not link_download:
             print("Não foi possível encontrar o link para download na tabela de resultados.")
             return

        # O link está no atributo 'href'
        url_arquivo = f"https://www.rad.cvm.gov.br/ENET/{link_download['href']}"
        print(f"Link de download encontrado: {url_arquivo}")

        # 5. Baixar e extrair o arquivo
        response_arquivo = session.get(url_arquivo)
        response_arquivo.raise_for_status()

        # O arquivo da CVM é um ZIP
        with ZipFile(BytesIO(response_arquivo.content)) as z:
            pdf_file_name = next((name for name in z.namelist() if "itr" in name.lower() and name.lower().endswith('.pdf')), None)
            
            if not pdf_file_name: # Plano B: pega qualquer PDF se o específico não for encontrado
                pdf_file_name = next((name for name in z.namelist() if name.lower().endswith('.pdf')), None)

            if pdf_file_name:
                nome_final_arquivo = f"{nome_empresa.replace(' ', '_')}_{pdf_file_name}"
                z.extract(pdf_file_name, path=".")
                # Renomeia o arquivo para incluir o nome da empresa
                os.rename(pdf_file_name, nome_final_arquivo)
                print(f"\nSucesso! Release '{nome_final_arquivo}' baixado e extraído.")
            else:
                print("Arquivo ZIP baixado, mas nenhum PDF foi encontrado dentro dele.")

    except requests.exceptions.RequestException as e:
        print(f"Erro de conexão: {e}")
    except (TypeError, KeyError) as e:
        print(f"Erro ao processar a página (parsing). O layout do site da CVM pode ter mudado. Detalhes: {e}")
    except Exception as e:
        print(f"Ocorreu um erro inesperado: {e}")

if __name__ == '__main__':
    # Use o nome da empresa como cadastrado na CVM.
    # Ex: "WEG", "PETROBRAS", "VALE", "ITAÚ UNIBANCO"
    nome_da_empresa = "WEG"  # Tente usar um nome mais simples primeiro
    # nome_da_empresa = "PETROLEO BRASILEIRO S A PETROBRAS"
    # nome_da_empresa = "VALE"

    baixar_ultimo_release_rad_cvm(nome_da_empresa)
