## Projeto
### WebScrapping no MercadoLivre
##
Este projeto tem como premissa, suprir a necessidade de procurar produtos no MercadoLivre, de forma automatica, com a junção de técnicas 
de pesquisa e manipulação de texto, foi possivel criar uma aplicação capaz de me enviar por SMS os principais produtos do MercadoLivre.
##
### Índice
1. Introdução
2. Resumo do script
3. Uso
4. Código:
    * Como funciona
    * Funções Utilizadas
    * SMS com Twilio
5. Atualizações
##
### 1. Introdução
Com a premissa de facilitar a automatizar a busca de produtos, este projeto usa ocmo base inicial, a estrutura do MercadoLivre, 
sendo capaz de buscar uma quantidade de itens, podendo enviar ao usuario, o nome o valor e o link de compra do produto. 
Pelo fato de o limite de caracter em SMS ser de 1600 caracteres (no aparelho testado), foi necessário implementar ao código uma API para encurtar os links.
##
### 2. Resumo do Script
O script tem como estrutura 4 agrupamentos, sendo eles:
1. Importação de bibliotecas
```
from bs4 import BeautifulSoup
import requests
import re
from dados import sid, token, remetente, destinatario
```

Foi utilizado as seguintes bibliotecas 
  * BeatifulSoup, para manipular a estrutura do HTML.
  * Requests, para receber os dados das API's
  * re (regex), sendo utilizado para organizar o texto com o valor do produto, inicialmente recebemos uma string "R$ (valor do produto) em reais", porém, com a raspagem do regex, é possivel extrair apenas os números
  * dados, sendo um script onde contem os dados para utilização do twilio 

2. Funções utilizadas
```
def shorten_url(long_url):
    api_url = "http://tinyurl.com/api-create.php"
    params = {'url': long_url}
    response = requests.get(api_url, params=params)
    
    if response.status_code == 200:
        return response.text
    else:
        return None

def corrigir_valor(valor):
    pattern = r'\d+(?:\.\d{2})?'

    if 'aria-label' in valor.attrs and "reais" in valor['aria-label']:
        preco = re.findall(pattern, span['aria-label'])
        
        return float(preco[0])
```
  * shorten_url: Utilizado para reduzir o link do anúncio
  * corrigir_valor: Utilizado para corrigir o texto contendo o preço do produto

3. Estrutura do loop
```
# Lista de exemplo com itens aleatórios
produtos = ['Morango','Chocolate','Bicicleta']
    
    # Criando variavel vazia, para acrescentar os itens da compra
mensagem = ""

# Loop para verificar os itens da lista no mercado livre
for produto in produtos:
    url = f"https://lista.mercadolivre.com.br/{produto}"
    response = requests.get(url)
    html_content = response.content

    # Analisando o HTML do site
    site = BeautifulSoup(html_content, 'html.parser')

    # Navegando nas tags para encontrar o <spans>
        # Tag com links
    h2 = site.find_all("h2", class_='ui-search-item__title') 
        # Tag com informações dos itens
    spans = site.find_all("span", class_="andes-money-amount ui-search-price__part ui-search-price__part--medium andes-money-amount--cents-superscript")
        # Tag com link do produto
    links = site.find_all('a', class_='ui-search-item__group__element ui-search-link__title-card ui-search-link')

    # Criando lista com os produtos encontrados
    lista_produtos = [produto.get_text() for produto in h2]
        # Pegando os Links dos produtos encontrados
    links = [tag['href'] for tag in links]

    # Percorrendo lista de valores e corrigindo os textos dos valores
    for i, span in enumerate(spans[:3]):
        if 'aria-label' in span.attrs and "reais" in span['aria-label']:

            long_url = links[i]
            short_url = shorten_url(long_url)
            preco_123 = corrigir_valor(span)
        
        # Salvando resultados na variavel
        mensagem += (f" \nProduto: {lista_produtos[i]}, R$ {span['aria-label']}\n{short_url}\n")
```
A estrutura principal do código, responsável pelo acesso ao HTML e a raspagem dos produtos no site. 
Após a raspagem, os produtos encontrados são armazenandos em uma variável, o código atual pega apenas 3 itens de cada produto da lista, tendo como limite 1600 caracteres, para assim ser enviado via SMS com Twilio.

4. Envio por SMS
```
# Com twilio, pego a pesquisa gerada e envio um SMS para o número cadastrado
from twilio.rest import Client

account_sid = sid
token = token

client = Client(account_sid,token)

message = client.messages.create(
    body= mensagem,
    from_= remetente,
    to= destinatario
)
```
Nessa etapa, após o agrupamento dos produtos, preços e links de anúncio estarem conclúidos, a variável criada, sera enviada pelo Twilio, que permite ser enviada por SMS para um telefone já cadastrado no acesso.
Para sua utilização é necessário o cadastro no site, gerando assim um número para remetente (número para envio do SMS), sendo possível salvar um destinatário (número que receberá o SMS).
