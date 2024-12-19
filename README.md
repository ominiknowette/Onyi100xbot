# Onyi100xbot
import requests
import time
from solana.rpc.api import Client
from textblob import TextBlob
import tweepy
from telegram import Bot

# Solana RPC Client
solana_client = Client("https://api.mainnet-beta.solana.com")

# APIs and Telegram Configuration
COINGECKO_API_URL = "https://api.coingecko.com/api/v3/simple/price"
SOLSCAN_NEW_TOKENS_API = "https://api.solscan.io/token/list"
TELEGRAM_BOT_TOKEN = "your_telegram_bot_token"
TELEGRAM_CHAT_ID = "your_telegram_chat_id"

# Twitter API credentials
TWITTER_API_KEY = "your_api_key"
TWITTER_API_SECRET = "your_api_secret"
TWITTER_ACCESS_TOKEN = "your_access_token"
TWITTER_ACCESS_SECRET = "your_access_secret"

# Thresholds and Constants
LIQUIDITY_THRESHOLD = 10000  # Minimum liquidity in USD
DEV_WALLET_PERCENTAGE = 20  # Max percentage of total supply held by one wallet
SENTIMENT_THRESHOLD = 0.1  # Positive sentiment score threshold
SYMBOLS_TO_TRACK = ["samoyedcoin", "bonk"]  # Initial symbols to track
TRACKED_MEMECOINS = set(SYMBOLS_TO_TRACK)
HISTORICAL_MEMECOINS = [
    {"symbol": "samoyedcoin", "start_price": 0.001, "peak_price": 0.1},
    {"symbol": "bonk", "start_price": 0.0000001, "peak_price": 0.00001},
]

# Set up Twitter API client
auth = tweepy.OAuthHandler(TWITTER_API_KEY, TWITTER_API_SECRET)
auth.set_access_token(TWITTER_ACCESS_TOKEN, TWITTER_ACCESS_SECRET)
twitter_api = tweepy.API(auth)

# Set up Telegram Bot
telegram_bot = Bot(token=TELEGRAM_BOT_TOKEN)

def send_telegram_message(message):
    """Send a message to the configured Telegram chat."""
    telegram_bot.send_message(chat_id=TELEGRAM_CHAT_ID, text=message)

def discover_new_tokens():
    """Discover new tokens on the Solana blockchain."""
    try:
        response = requests.get(SOLSCAN_NEW_TOKENS_API)
        if response.status_code == 200:
            tokens = response.json()["data"]
            for token in tokens:
                symbol = token.get("symbol", "").lower()
                if (
                    symbol not in TRACKED_MEMECOINS
                    and any(keyword in symbol for keyword in ["doge", "inu", "bonk", "shiba"])
                ):
                    TRACKED_MEMECOINS.add(symbol)
                    SYMBOLS_TO_TRACK.append(symbol)
                    send_telegram_message(f"🎉 New memecoin discovered: {symbol.upper()}")
    except Exception as e:
        send_telegram_message(f"Error discovering new tokens: {e}")

def get_token_prices(symbols):
    """Fetch token prices from CoinGecko."""
    response = requests.get(COINGECKO_API_URL, params={
        "ids": ",".join(symbols),
        "vs_currencies": "usd"
    })
    if response.status_code == 200:
        return response.json()
    return {}

def fetch_tweets(keyword):
    """Fetch recent tweets about a specific keyword."""
    try:
        tweets = twitter_api.search_tweets(q=keyword, count=100, lang="en")
        return [tweet.text for tweet in tweets]
    except Exception as e:
        send_telegram_message(f"Error fetching tweets for {keyword}: {e}")
        return []

def analyze_sentiment(tweets):
    """Analyze the sentiment of a list of tweets."""
    sentiment_scores = [TextBlob(tweet).sentiment.polarity for tweet in tweets]
    if sentiment_scores:
        avg_sentiment = sum(sentiment_scores) / len(sentiment_scores)
        return avg_sentiment
    return 0

def check_community_sentiment(symbol):
    """Check social sentiment for a token."""
    tweets = fetch_tweets(f"#{symbol}")
    sentiment_score = analyze_sentiment(tweets)
    if sentiment_score > SENTIMENT_THRESHOLD:
        return f"🚀 Strong positive sentiment detected! (Score: {sentiment_score:.2f})"
    elif sentiment_score < -SENTIMENT_THRESHOLD:
        return f"⚠️ Negative sentiment detected. (Score: {sentiment_score:.2f})"
    return f"Sentiment is neutral. (Score: {sentiment_score:.2f})"

def check_liquidity(token_address):
    """Check the token's liquidity pool on Solana."""
    # Placeholder: Replace with actual API call to Raydium or Orca
    try:
        response = solana_client.get_token_largest_accounts(token_address)
        if response['result']:
            total_liquidity = sum(account['amount'] for account in response['result']['value'])
            return total_liquidity > LIQUIDITY_THRESHOLD
    except Exception as e:
        send_telegram_message(f"Error checking liquidity: {e}")
    return False

def check_rug_signals(token_address):
    """Check for potential rug signals."""
    liquidity_ok = check_liquidity(token_address)
    if not liquidity_ok:
        return "⚠️ Low liquidity detected. Possible rug pull."
    return None

def analyze_historical_100x_patterns():
    """Analyze historical patterns of successful memecoins."""
    successful_patterns = []
    for memecoin in HISTORICAL_MEMECOINS:
        growth_factor = memecoin["peak_price"] / memecoin["start_price"]
        if growth_factor >= 100:
            successful_patterns.append(memecoin["symbol"])
    return successful_patterns

def analyze_market(symbols):
    """Analyze token prices, sentiment, and generate signals."""
    prices = get_token_prices(symbols)
    signals = []
    for symbol, data in prices.items():
        price = data["usd"]

        # Community sentiment check
        sentiment = check_community_sentiment(symbol)
        
        # Rug pull detection (dummy token address used here)
        rug_signal = check_rug_signals("DummyTokenAddress")
        
        # Combine signals
        if rug_signal:
            signals.append(f"{symbol.upper()} - {rug_signal}")
        else:
            signals.append(f"{symbol.upper()} - {sentiment} Price: ${price}")
    return signals

def main():
    send_telegram_message("🚀 Bot started. Monitoring memecoins...")
    
    # Log successful patterns
    successful_patterns = analyze_historical_100x_patterns()
    send_telegram_message(f"Identified successful patterns: {', '.join(successful_patterns)}")

    while True:
        try:
            # Discover new tokens
            discover_new_tokens()
            
            # Analyze market for tracked tokens
            signals = analyze_market(SYMBOLS_TO_TRACK)

            for signal in signals:
                send_telegram_message(signal)  # Send signals to Telegram
            
            time.sleep(60)  # Wait before next cycle
        except Exception as e:
            error_message = f"Error: {e}"
            send_telegram_message(error_message)
            time.sleep(10)

if __name__ == "__main__":
    main()
