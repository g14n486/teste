import re
from patterns import patterns

# Build combined regex using patterns from the dictionary
token_pattern = re.compile(
    fr"({patterns['currency']}\s*{patterns['number']})"       # Currency + number
    r"|([+-]?\d+(?:[\.,]\d+)?%)"                             # Signed percentages
    r"|(" + patterns['word'] + r")"                           # Words
    r"|(" + patterns['number'] + r")"                         # Numbers alone
    r"|(" + patterns['percent'] + r")",                       # Percent words or symbols
    re.IGNORECASE,
)

def tokenize_financial_text(text):
    tokens = []
    for match in token_pattern.finditer(text):
        token = next(t for t in match.groups() if t)
        tokens.append(token)
    return tokens

# Example usage:
text = "O lucro foi de R$ 1.234,56 no último trimestre, um aumento de +12.5% em relação ao USD 2,000."
tokens = tokenize_financial_text(text)
print(tokens)
