
‏from binance.client import Client
‏from telegram import Update
‏from telegram.ext import Updater, CommandHandler, CallbackContext
‏import pandas as pd
‏import talib
‏import numpy as np

# ========== 🔑 إعداداتك ========== #
‏BINANCE_API_KEY =
‏"d3RGrO6eht9RQmuRuVDKwCVfr8LZVzkJ1EUuD7spOa9BwtHb5adhMCCm1m4LIply"
‏BINANCE_SECRET_KEY =
‏k14PQQ6B7kgmkvdZpye6vxCeT1q3XDdyOngwjPmRrKrvrQGU5IviOfb8duSEHg50
‏TELEGRAM_TOKEN = 
‏7518600834:AAH8mBtt2hcYHkRRt7J3M-y36713lUbz7p8

‏CRYPTO_PAIRS = [
‏    "BTCUSDT", "ETHUSDT", "BNBUSDT", "SOLUSDT", "XRPUSDT",
‏    "ADAUSDT", "DOGEUSDT", "DOTUSDT", "MATICUSDT", "AVAXUSDT"
]
# ================================ #

‏client = Client(BINANCE_API_KEY, BINANCE_SECRET_KEY)

‏def get_enhanced_data(symbol, interval='4h', limit=200):
    """جلب بيانات معززة بإطارات زمنية متعددة"""
‏    frames = []
‏    for interval in ['15m', '1h', '4h']:
‏        klines = client.futures_klines(symbol=symbol, interval=interval, limit=limit)
‏        df = pd.DataFrame(klines, columns=['timestamp', 'open', 'high', 'low', 'close', 'volume', 
‏                                          'close_time', 'quote_asset_volume', 'trades', 
‏                                          'taker_buy_base', 'taker_buy_quote', 'ignore'])
‏        df = df[['timestamp', 'open', 'high', 'low', 'close', 'volume']].astype(float)
‏        df['interval'] = interval
‏        frames.append(df)
‏    return pd.concat(frames)

‏def advanced_analysis(symbol):
    """تحليل متقدم باستخدام مؤشرات متعددة"""
‏    try:
‏        df = get_enhanced_data(symbol)
        
        # تحليلات الإطار الزمني 4h (الأكثر أهمية)
‏        df_4h = df[df['interval'] == '4h']
        
        # حساب المؤشرات
‏        df_4h['RSI'] = talib.RSI(df_4h['close'], timeperiod=14)
‏        df_4h['MACD'], _, _ = talib.MACD(df_4h['close'], fastperiod=12, slowperiod=26, signalperiod=9)
‏        df_4h['EMA_50'] = talib.EMA(df_4h['close'], timeperiod=50)
‏        df_4h['EMA_200'] = talib.EMA(df_4h['close'], timeperiod=200)
‏        df_4h['ATR'] = talib.ATR(df_4h['high'], df_4h['low'], df_4h['close'], timeperiod=14)
        
‏        last = df_4h.iloc[-1]
‏        prev = df_4h.iloc[-2]
        
        # فلترة الإشارات
‏        buy_signal = False
‏        sell_signal = False
        
        # شروط الشراء (مثال)
‏        if (last['RSI'] < 35 and 
‏            last['MACD'] > 0 and 
‏            last['close'] > last['EMA_50'] and 
‏            last['EMA_50'] > last['EMA_200']):
‏            buy_signal = True
            
        # شروط البيع (مثال)
‏        if (last['RSI'] > 65 or 
‏            last['close'] < last['EMA_200']):
‏            sell_signal = True
            
        # إعداد التقرير
‏        report = f"\n🔍 {symbol} تحليل متقدم:\n"
‏        report += f"📊 السعر: {last['close']:.2f}\n"
‏        report += f"📈 RSI: {last['RSI']:.1f} ({'⬆️' if last['RSI'] > prev['RSI'] else '⬇️'})\n"
‏        report += f"📉 MACD: {last['MACD']:.4f}\n"
‏        report += f"📌 EMA 50/200: {last['EMA_50']:.2f}/{last['EMA_200']:.2f}\n"
‏        report += f"🎯 ATR (تقلبات): {last['ATR']:.2f}\n\n"
        
‏        if buy_signal:
‏            report += "✅ إشارة شراء قوية (تأكيد متعدد المؤشرات)"
‏        elif sell_signal:
‏            report += "❌ إشارة بيع تحذيرية"
‏        else:
‏            report += "🔄 لا توجد إشارات قوية حالياً"
            
‏        return report
        
‏    except Exception as e:
‏        return f"⚠️ خطأ في تحليل {symbol}: {str(e)}"

‏def analyze_all(update: Update, context: CallbackContext):
    """تحليل جميع العملات"""
‏    update.message.reply_text("🔄 جاري تحليل السوق... (قد يستغرق دقيقة)")
    
‏    results = []
‏    for symbol in CRYPTO_PAIRS:
‏        try:
‏            analysis = advanced_analysis(symbol)
‏            results.append(analysis)
‏        except Exception as e:
‏            results.append(f"❌ فشل تحليل {symbol}")
    
    # تقسيم النتائج لعدة رسائل لتجنب حد التلغرام
‏    for i in range(0, len(results), 3):
‏        batch = results[i:i+3]
‏        update.message.reply_text("\n\n".join(batch))

‏def main():
‏    updater = Updater(TELEGRAM_TOKEN)
‏    dp = updater.dispatcher
    
‏    dp.add_handler(CommandHandler("start", lambda u,c: u.message.reply_text(
        "🏦 بوت التحليل المتقدم للعملات الرقمية\n"
        "الأوامر:\n"
‏        "/analyze_all - تحليل جميع العملات\n"
‏        "/analyze BTCUSDT - تحليل عملة محددة"
    )))
    
‏    dp.add_handler(CommandHandler("analyze_all", analyze_all))
‏    dp.add_handler(CommandHandler("analyze", 
‏        lambda u,c: u.message.reply_text(advanced_analysis(u.args[0].upper() if u.args else "BTCUSDT"))))
    
‏    updater.start_polling()
‏    print("✅ البوت يعمل بنجاح!")
‏    updater.idle()

‏if __name__ == '__main__':
‏    main()
