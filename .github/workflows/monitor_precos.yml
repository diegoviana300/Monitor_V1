import requests
from bs4 import BeautifulSoup
from telegram import Bot
import json
import os
import asyncio

# --- CONFIGURAÇÃO INICIAL ---
# Não precisa de dotenv, pois as variáveis vêm dos secrets do GitHub
TOKEN = os.getenv("TOKEN")
CHAT_ID = os.getenv("CHAT_ID")

if not TOKEN or not CHAT_ID:
    # Mensagem de erro específica para o ambiente do GitHub Actions
    raise ValueError("ERRO: Secrets TOKEN e CHAT_ID não foram encontrados. Verifique as configurações do repositório.")

bot = Bot(token=TOKEN)

# --- FUNÇÕES DO BOT ---

def carregar_produtos_do_json():
    """Lê o arquivo produtos.json do repositório."""
    try:
        with open("produtos.json", "r", encoding="utf-8") as f:
            return json.load(f)
    except FileNotFoundError:
        print("ERRO: O arquivo 'produtos.json' não foi encontrado no repositório.")
        return []
    except json.JSONDecodeError:
        print("ERRO: O arquivo 'produtos.json' contém um erro de formatação.")
        return []

def pegar_preco_exato(url):
    """Extrai o preço do Mercado Livre com múltiplas estratégias."""
    if not url or not url.strip(): return None
    print(f"  🔍 Buscando preço em: {url[:80]}...")
    headers = {"User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36"}
    try:
        response = requests.get(url, headers=headers, timeout=15)
        response.raise_for_status()
        soup = BeautifulSoup(response.text, "html.parser")
        
        # Estratégia 1: A meta tag (a melhor)
        meta_tag = soup.select_one('meta[itemprop="price"]')
        if meta_tag and meta_tag.get('content'):
            print(f'  ✓ Preço encontrado com Estratégia 1 (meta tag)')
            return float(meta_tag['content'])

        # Estratégia 2: O container principal
        price_container = soup.select_one(".ui-pdp-price__main-container")
        if price_container:
            fraction = price_container.select_one(".andes-money-amount__fraction")
            cents = price_container.select_one(".andes-money-amount__cents")
            if fraction:
                price_str = fraction.text.replace('.', '')
                if cents and cents.text: price_str += f".{cents.text}"
                print(f'  ✓ Preço encontrado com Estratégia 2 (container)')
                return float(price_str)

        # Estratégia 3: Seletor genérico
        fraction = soup.select_one(".price-tag-fraction, .andes-money-amount__fraction")
        if fraction:
             print(f'  ✓ Preço encontrado com Estratégia 3 (genérico)')
             return float(fraction.text.replace('.', '').replace(',', '.'))

        print(f"  ❌ Preço não encontrado para {url[:50]}...")
        return None
    except Exception as e:
        print(f"  ❌ Erro inesperado ao buscar preço: {type(e).__name__}: {e}")
        return None

async def enviar_alerta(nome, url, preco, preco_desejado):
    """Envia alerta via Telegram."""
    mensagem = (
        f"🎉 *PREÇO BAIXOU!*\n\n"
        f"**Produto:** {nome}\n"
        f"**💰 Preço atual:** R$ {preco:,.2f}\n"
        f"**🎯 Preço desejado:** R$ {preco_desejado:,.2f}\n\n"
        f"🔗 [Ver produto]({url})"
    )
    await bot.send_message(chat_id=CHAT_ID, text=mensagem, parse_mode="Markdown")
    print(f"  ✓ Alerta enviado para o Telegram!")

async def fazer_verificacao_unica():
    """Executa uma verificação completa de preços e encerra."""
    print("\n" + "="*50)
    print("🤖 MONITOR DE PREÇOS - INICIANDO VERIFICAÇÃO (AGENDADA)")
    print("="*50)
    
    produtos = carregar_produtos_do_json()
    
    if not produtos:
        print("\n❌ Nenhum produto para monitorar. Verificação encerrada.")
        return
    
    print(f"\n🔍 Iniciando monitoramento de {len(produtos)} produtos...")
    
    for i, produto in enumerate(produtos, 1):
        nome = produto.get('nome', 'Produto sem nome')
        url = produto.get('url', '')
        preco_desejado = produto.get('preco_desejado', 0)
        
        print(f"\n[{i}/{len(produtos)}] 🛍️ {nome}")
        
        if not url:
            print("  ❌ URL inválida, pulando...")
            continue
        
        preco_atual = pegar_preco_exato(url)
        
        if preco_atual is not None:
            print(f"  -> Preço atual: R$ {preco_atual:.2f} | Desejado: R$ {preco_desejado:,.2f}")
            if preco_atual <= preco_desejado:
                await enviar_alerta(nome, url, preco_atual, preco_desejado)
        else:
            print(f"  ⚠️  Não foi possível obter o preço")
        
        if i < len(produtos): await asyncio.sleep(3)
    
    print(f"\n" + "="*50)
    print(f"✅ VERIFICAÇÃO CONCLUÍDA!")
    print("="*50)

if __name__ == "__main__":
    asyncio.run(fazer_verificacao_unica())
