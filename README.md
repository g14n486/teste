import re

# Pattern components:
word_pattern = r"[A-Za-zÀ-ÿ]+"  # English + accented Portuguese letters
number_pattern = r"(?:\d{1,3}(?:[\.,]\d{3})*|\d+)(?:[\.,]\d+)?"  # Numbers with thousand separators
currency_pattern = r"(?:R\$|USD|\$|€)?"  # Common currency symbols
percent_pattern = r"%|\bpor cento\b"

# Combine patterns
token_pattern = re.compile(
    fr"({currency_pattern}\s*{number_pattern})"  # Currency + number
    r"|([+-]?\d+(?:[\.,]\d+)?%)"                 # Percentage with sign
    r"|(" + word_pattern + r")"                   # Words
    r"|(" + number_pattern + r")"                 # Numbers alone
    r"|(" + percent_pattern + r")",               # Percent words or symbols
    re.IGNORECASE,
)

def tokenize_financial_text(text):
    tokens = []
    for match in token_pattern.finditer(text):
        # Each group corresponds to one type of token
        token = next(t for t in match.groups() if t)
        tokens.append(token)
    return tokens

# Example usage:
text = "O lucro foi de R$ 1.234,56 no último trimestre, um aumento de +12.5% em relação ao USD 2,000."
tokens = tokenize_financial_text(text)
print(tokens)
