Sim, é totalmente possível e, na verdade, é o método mais recomendado e confiável fazer a busca diretamente no site da CVM (Comissão de Valores Mobiliários).

Utilizar a CVM como fonte primária oferece vantagens significativas sobre o scraping de sites de Relações com Investidores (RI) individuais:

Centralização: Todos os documentos oficiais de todas as companhias abertas estão em um único lugar.

Padronização: O formato de entrega e a nomenclatura dos documentos (ITR, DFP, etc.) são padronizados pela CVM.

Confiabilidade: É a fonte oficial. A informação é disponibilizada na CVM no momento em que se torna pública para todo o mercado.

Disponibilidade: O site da CVM é historicamente mais estável e menos propenso a mudanças de layout que quebrariam um script de scraping.

O Desafio: Interagindo com o Portal Empresas.NET

O sistema da CVM para consulta de documentos é o Empresas.NET. Diferente de um site estático, a consulta de documentos nele é feita através do preenchimento de um formulário e envio dos dados para o servidor (uma requisição do tipo POST).

Para automatizar esse processo, nosso script precisa simular o preenchimento e envio deste formulário. O resultado da busca nos dará um link para o download do documento, que geralmente vem em um arquivo .zip.

Código Python para Baixar Releases da CVM

O script a seguir implementa a lógica para buscar e baixar o último release trimestral (ITR - Informações Trimestrais) de uma empresa diretamente do portal Empresas.NET.

Generated python
import requests
from bs4 import BeautifulSoup
import os
from datetime import date
import re
from zipfile import ZipFile
from io import BytesIO

def baixar_ultimo_release_cvm(nome_empresa):
    """
    Busca e baixa o último documento ITR (release trimestral) de uma empresa
    diretamente do sistema Empresas.NET da CVM.
    """
    print(f"Iniciando busca na CVM para: '{nome_empresa}'")

    # URL do formulário de consulta de documentos periódicos da CVM
    url_consulta = "http://sistemas.cvm.gov.br/consultadocumentos/FormularioConsultaDocumentos.aspx"
    
    # URL base para visualizar o documento encontrado
    url_download_base = "http://sistemas.cvm.gov.br/consultadocumentos/"

    # Usamos uma sessão para manter o estado da navegação (importante para sites com formulários)
    session = requests.Session()
    
    # Primeiro, acessamos a página para obter os dados necessários para o formulário (viewstate, etc.)
    try:
        response_form = session.get(url_consulta)
        response_form.raise_for_status()
        soup_form = BeautifulSoup(response_form.text, 'html.parser')

        # Extraímos os campos ocultos necessários para a requisição POST
        viewstate = soup_form.find('input', {'name': '__VIEWSTATE'})['value']
        viewstategenerator = soup_form.find('input', {'name': '__VIEWSTATEGENERATOR'})['value']
        eventvalidation = soup_form.find('input', {'name': '__EVENTVALIDATION'})['value']

    except requests.exceptions.RequestException as e:
        print(f"Erro ao acessar o formulário da CVM: {e}")
        return
    except (TypeError, KeyError):
        print("Não foi possível encontrar os campos de formulário necessários. O site da CVM pode ter mudado.")
        return

    # Dados do formulário (payload) que vamos enviar via POST
    # Simulamos o preenchimento do nome da empresa e a busca por ITR
    payload = {
        '__EVENTTARGET': '',
        '__EVENTARGUMENT': '',
        '__LASTFOCUS': '',
        '__VIEWSTATE': viewstate,
        '__VIEWSTATEGENERATOR': viewstategenerator,
        '__EVENTVALIDATION': eventvalidation,
        'txtEmpresa': nome_empresa,
        'chkTrabalho': 'on', # Importante para buscar os documentos
        'radDocumento': '1', # 1 = Todos, 2 = Último dia, 3 = Últimos 7 dias
        'cmbCategoria': '1', # 1 = ITR
        'cmbAno': '-1', # -1 = Todos os anos
        'btnConsultar': 'Consultar'
    }

    print("Enviando consulta de documentos para a CVM...")
    try:
        response_result = session.post(url_consulta, data=payload)
        response_result.raise_for_status()
        soup_result = BeautifulSoup(response_result.text, 'html.parser')

        # Procuramos a tabela de resultados
        tabela_resultados = soup_result.find('table', {'class': 'Resultado'})

        if not tabela_resultados:
            print(f"Nenhum documento ITR encontrado para '{nome_empresa}' com os critérios fornecidos.")
            return

        # Pegamos a primeira linha da tabela (o resultado mais recente)
        primeiro_resultado = tabela_resultados.find('tr', {'class': 'GridRow_Alternating'})
        if not primeiro_resultado:
             primeiro_resultado = tabela_resultados.find('tr', {'class': 'GridRow'})
        
        if not primeiro_resultado:
             print("Não foi possível encontrar a linha de resultado na tabela.")
             return

        # O link está dentro de uma função JavaScript no atributo 'onclick'
        onclick_attr = primeiro_resultado.find('a')['onclick']
        
        # Usamos regex para extrair o número do documento (ID) da função JS
        match = re.search(r"VisualizarDocumento\('(\d+)'\)", onclick_attr)
        if not match:
            print("Não foi possível extrair o ID do documento do link.")
            return

        id_documento = match.group(1)
        print(f"Documento mais recente encontrado. ID: {id_documento}")

        # Construímos a URL final para download
        url_download_doc = f"{url_download_base}VisualizarDocumento.aspx?id={id_documento}"

        print(f"Baixando arquivo de: {url_download_doc}")
        response_doc = session.get(url_download_doc)
        response_doc.raise_for_status()
        
        # O arquivo da CVM é um ZIP. Precisamos extrair o PDF de dentro dele.
        with ZipFile(BytesIO(response_doc.content)) as z:
            # Procura pelo arquivo PDF dentro do ZIP
            pdf_file_name = next((name for name in z.namelist() if name.lower().endswith('.pdf')), None)
            
            if pdf_file_name:
                z.extract(pdf_file_name, path=".") # Extrai o PDF para a pasta atual
                print(f"\nSucesso! Release '{pdf_file_name}' baixado e extraído.")
            else:
                print("Arquivo ZIP baixado, mas nenhum PDF foi encontrado dentro dele.")

    except requests.exceptions.RequestException as e:
        print(f"Erro durante a consulta ou download do documento: {e}")
    except Exception as e:
        print(f"Ocorreu um erro inesperado: {e}")

if __name__ == '__main__':
    # IMPORTANTE: Use o nome da empresa como cadastrado na CVM.
    # Evite usar o ticker (ex: WEGE3). Use o nome completo ou uma parte clara dele.
    nome_da_empresa = "WEG S.A."
    # nome_da_empresa = "PETROLEO BRASILEIRO S.A. PETROBRAS"
    # nome_da_empresa = "VALE S.A."
    
    baixar_ultimo_release_cvm(nome_da_empresa)

Como Usar e Principais Pontos de Atenção

Instale as Bibliotecas: Se ainda não as tiver, instale as dependências.

Generated bash
pip install requests beautifulsoup4
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

Nome da Empresa: O campo de busca da CVM funciona melhor com o nome oficial da empresa (razão social) e não com o código de negociação (ticker). Para usar o script, altere a variável nome_da_empresa para o nome da companhia desejada.

Tipo de Documento: Este código está configurado para buscar "ITR - Informações Trimestrais" ('cmbCategoria': '1'). Se você quiser outros documentos, como as "DFP - Demonstrações Financeiras Padronizadas" (anuais), você precisaria inspecionar o código-fonte da página da CVM para encontrar o valor correto para a categoria e ajustar o payload.

Arquivo .zip: Este é um ponto crucial. O sistema da CVM agrupa todos os arquivos de um comunicado (o release em PDF, as demonstrações financeiras em planilhas, etc.) em um único arquivo .zip. O código já está preparado para baixar este zip, encontrar o arquivo PDF dentro dele e extraí-lo para a mesma pasta onde o script está sendo executado.

Este método é, sem dúvida, a abordagem mais profissional e robusta para a automação da coleta de dados financeiros no mercado brasileiro.
