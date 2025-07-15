pdf-sentiment-model/
├── 📁 data/
│   ├── 📁 raw/                 # PDFs originais
│   ├── 📁 extracted_text/      # Texto bruto extraído dos PDFs
│   ├── 📁 tokens/              # Dados tokenizados
│   ├── 📁 sentiment/           # Resultados da análise de sentimentos
│   ├── 📁 financials/          # Dados financeiros paralelos (CSV, JSON)
│   └── 📁 processed/           # Dados finais para modelagem
│
├── 📁 src/                     # Código-fonte
│   ├── 📁 extract/
│   │   └── pdf_to_text.py      # Extrator de texto dos PDFs
│   ├── 📁 tokenize/
│   │   └── tokenizer.py        # Tokenização e limpeza de texto
│   ├── 📁 sentiment/
│   │   └── sentiment_model.py  # Análise de sentimento (LLM ou modelo local)
│   ├── 📁 financials/
│   │   └── load_financials.py  # Parser de dados financeiros
│   ├── 📁 integrate/
│   │   └── merge_data.py       # Junta sentimentos + dados financeiros
│   └── 📁 model/
│       └── predictive_model.py # Modelo estatístico preditivo
│
├── 📁 notebooks/               # Jupyter notebooks exploratórios
│   └── analysis.ipynb
│
├── 📁 models/                  # Modelos treinados e pesos
│   └── final_model.pkl
│
├── 📁 config/                  # Configurações, paths, parâmetros
│   └── settings.yaml
│
├── 📁 utils/                   # Funções auxiliares reutilizáveis
│   └── helpers.py
│
├── 📁 tests/                   # Testes unitários
│   └── test_tokenizer.py
│
├── main.py                    # Pipeline orquestrador
├── requirements.txt           # Dependências
└── README.md                  # Documentação do projeto


