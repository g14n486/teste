def parece_portugues(token):
    token = token.strip().lower()
    
    # If it contains any digit → not a Portuguese word
    if any(char.isdigit() for char in token):
        return False

    # If it contains accented characters → likely Portuguese
    if re.search(r'[áéíóúãõâêîôûçà]', token):
        return True

    # If it ends with typical Portuguese suffixes → likely Portuguese
    if re.search(r'(ção|dade|mente|eiro|ista|ável|ível|amento|ento|al)$', token):
        return True

    return False

