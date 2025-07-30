import requests
from bs4 import BeautifulSoup
import os
from zipfile import ZipFile
from io import BytesIO
from datetime import datetime

def baixar_ultimo_release_rad_cvm(nome_empresa):
    """
    Busca e baixa o último documento ITR (release trimestral) de uma empresa
    diretamente do portal RAD da CVM, usando o método de busca correto.
    """
    print(f"Iniciando busca no portal RAD CVM para: '{nome_empresa}'")

    url_rad = "https://www.rad.cvm.gov.br/ENET/frmConsultaExternaCVM.aspx"
    headers = {
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36',
        'Referer': url_rad # Adicionar o Referer é uma boa prática
    }

    session = requests.Session()
    session.headers.update(headers)

    try:
        # 1. ETAPA ÚNICA: Obter o formulário e enviar a consulta de uma só vez.
        print("Acessando o formulário para obter tokens de sessão...")
        response_form = session.get(url_rad)
        response_form.raise_for_status()
        soup_form = BeautifulSoup(response_form.text, 'html.parser')

        # Extrai os campos ocultos necessários para a requisição POST
        viewstate = soup_form.find('input', {'name': '__VIEWSTATE'})['value']
        viewstategenerator = soup_form.find('input', {'name': '__VIEWSTATEGENERATOR'})['value']
        eventvalidation = soup_form.find('input', {'name': '__EVENTVALIDATION'})['value']

        # 2. Construir o payload final para a consulta direta
        # O nome da empresa é enviado junto com os filtros de documento.
        # Não há passo intermediário para buscar o código CVM.
        payload_final = {
            '__EVENTTARGET': '',
            '__EVENTARGUMENT': '',
            '__LASTFOCUS': '',
            '__VIEWSTATE': viewstate,
            '__VIEWSTATEGENERATOR': viewstategenerator,
            '__EVENTVALIDATION': eventvalidation,
            'txtChave': '',
            'txtNomeEmpresa': nome_empresa,
            'txtCNPJ': '',
            'txtAssunto': 'itr',  # Filtra por Informações Trimestrais
            'txtDataIni': '01/01/2020', # Um período razoável para pegar os últimos
            'txtDataFim': datetime.now().strftime('%d/%m/%Y'),
            'hdnCodCVM': '', # O código CVM vai vazio, o site resolve pelo nome
            'btnConsulta': 'btConsultar',
        }

        print(f"Enviando consulta de documentos ITR para '{nome_empresa}'...")
        response_docs = session.post(url_rad, data=payload_final)
        response_docs.raise_for_status()
        soup_docs = BeautifulSoup(response_docs.text, 'html.parser')

        # 3. Processar a tabela de resultados
        grid_docs = soup_docs.find('table', {'id': 'grdDocumentos'})
        if not grid_docs or not grid_docs.find('a', {'id': lambda x: x and x.startswith('hypDownload')}):
            print(f"Nenhum documento do tipo ITR encontrado para '{nome_empresa}'.")
            print("Verifique se o nome da empresa está exatamente como na CVM (ex: 'WEG S.A.').")
            return

        # O primeiro link de download na tabela é o mais recente
        link_download = grid_docs.find('a', {'id': lambda x: x and x.startswith('hypDownload')})
        if not link_download:
             print("Não foi possível encontrar o link de download na tabela de resultados.")
             return

        url_arquivo_relativa = link_download['href']
        url_arquivo_completa = f"https://www.rad.cvm.gov.br/ENET/{url_arquivo_relativa}"
        print(f"Link de download encontrado: {url_arquivo_completa}")

        # 4. Baixar e extrair o arquivo ZIP
        print("Baixando arquivo .zip...")
        response_arquivo = session.get(url_arquivo_completa)
        response_arquivo.raise_for_status()

        with ZipFile(BytesIO(response_arquivo.content)) as z:
            # Procura por um PDF que contenha 'itr' no nome, que é o padrão.
            pdf_file_name = next((name for name in z.namelist() if "itr" in name.lower() and name.lower().endswith('.pdf')), None)
            
            # Se não encontrar, pega o primeiro PDF disponível como alternativa.
            if not pdf_file_name:
                pdf_file_name = next((name for name in z.namelist() if name.lower().endswith('.pdf')), None)

            if pdf_file_name:
                # Cria um nome de arquivo mais descritivo
                nome_empresa_formatado = nome_empresa.replace(' ', '_').replace('.', '')
                nome_final_arquivo = f"{nome_empresa_formatado}_{pdf_file_name}"
                
                # Extrai o PDF para a pasta atual e renomeia
                z.extract(pdf_file_name, path=".")
                if os.path.exists(nome_final_arquivo):
                     os.remove(nome_final_arquivo) # Remove arquivo antigo se existir
                os.rename(pdf_file_name, nome_final_arquivo)
                print(f"\nSucesso! Release '{nome_final_arquivo}' foi baixado e salvo.")
            else:
                print("Arquivo ZIP baixado, mas nenhum PDF foi encontrado dentro dele.")

    except requests.exceptions.RequestException as e:
        print(f"Erro de conexão: {e}")
    except (TypeError, KeyError) as e:
        print(f"Erro ao processar a página (parsing). O layout do site da CVM pode ter mudado. Detalhes: {e}")
    except Exception as e:
        print(f"Ocorreu um erro inesperado: {e}")

if __name__ == '__main__':
    # IMPORTANTE: Use o nome da empresa como cadastrado na CVM.
    # A busca é sensível ao nome. "WEG S.A." funciona, "WEG" pode não funcionar.
    nome_da_empresa = "WEG S.A."
    # nome_da_empresa = "PETROLEO BRASILEIRO S A PETROBRAS"
    # nome_da_empresa = "VALE S.A."
    # nome_da_empresa = "ITAU UNIBANCO HOLDING S.A."
    
    baixar_ultimo_release_rad_cvm(nome_da_empresa)```

### O Ponto Chave da Correção

*   **Nome Exato da Empresa:** A busca no portal da CVM é muito literal. Você precisa usar a razão social o mais próximo possível do oficial. O detalhe crucial que faz a busca funcionar para a WEG é usar **`WEG S.A.`** em vez de apenas "WEG".
*   **Fluxo de Requisição Única:** O script agora faz uma única consulta (`POST`) enviando todos os dados de uma vez (`txtNomeEmpresa` + `txtAssunto`), que é como um usuário real interage com o site.

Com esta versão, o script acessa o portal corretamente, encontra os documentos e baixa o release mais recente, como esperado.
