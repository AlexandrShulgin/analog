Вот полный перевод вашего проекта на русский язык, готовый для сохранения в Word документ:

# Готовая к Production Платформа для Показа Видеорекламы в Telegram Mini Apps

## Архитектура системы

```
1. SDK (ads-sdk.js) - Клиентская библиотека для Mini Apps
2. Бэкенд (FastAPI) - Показ рекламы и верификация
3. React Frontend - Демо-интерфейс
4. Ad Iframe - Изолированный контейнер для рекламного контента
```

## 1. Бэкенд (FastAPI)

### `main.py`

```python
import os
from datetime import datetime
from fastapi import FastAPI, Request, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import hashlib
import hmac
import secrets
from typing import Dict

app = FastAPI()

# Настройки CORS для разработки
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Временное хранилище для демо (в продакшене заменить на БД)
AD_BLOCKS = {
    "demo_block": {
        "id": "demo_block",
        "postback_url": "https://your-webhook-url.com/postback",
        "ads": [
            {
                "id": "ad1",
                "type": "rewarded",
                "video_url": "https://example.com/ads/video1.mp4",
                "duration": 30,
                "reward_amount": 10
            }
        ]
    }
}

# Секретный ключ для подписи запросов (в продакшене использовать переменную окружения)
SECRET_KEY = os.getenv("SECRET_KEY", "demo-secret-key-change-me")

class AdRequest(BaseModel):
    block_id: str
    user_id: str
    chat_instance: str
    is_premium: bool
    language: str
    platform: str

class AdEvent(BaseModel):
    session_id: str
    event_type: str  # "start", "complete", "close"
    signature: str
    timestamp: int

@app.post("/api/ads/request")
async def request_ad(ad_request: AdRequest, request: Request):
    """
    Эндпоинт для запроса рекламы
    - Проверяет существование block_id
    - Создает сессию для отслеживания
    - Возвращает данные рекламы с подписанной сессией
    """
    if ad_request.block_id not in AD_BLOCKS:
        raise HTTPException(status_code=404, detail="Рекламный блок не найден")
    
    # Генерация уникального ID сессии
    session_id = secrets.token_hex(16)
    timestamp = int(datetime.now().timestamp())
    
    # Создание подписи
    payload = f"{ad_request.user_id}:{ad_request.block_id}:{session_id}:{timestamp}"
    signature = hmac.new(SECRET_KEY.encode(), payload.encode(), hashlib.sha256).hexdigest()
    
    # Выбор случайной рекламы из блока
    ad = secrets.choice(AD_BLOCKS[ad_request.block_id]["ads"])
    
    return {
        "ad": ad,
        "session": {
            "id": session_id,
            "signature": signature,
            "timestamp": timestamp
        }
    }

@app.post("/api/ads/event")
async def track_ad_event(event: AdEvent):
    """
    Эндпоинт для отслеживания просмотра рекламы
    - Проверяет подпись сессии
    - Обрабатывает разные типы событий
    - Отправляет постбэк для завершенных просмотров
    """
    # Проверка подписи
    expected_signature = hmac.new(
        SECRET_KEY.encode(),
        f"{event.session_id}:{event.timestamp}".encode(),
        hashlib.sha256
    ).hexdigest()
    
    if not hmac.compare_digest(expected_signature, event.signature):
        raise HTTPException(status_code=401, detail="Неверная подпись")
    
    # В продакшене: Запись события в БД
    print(f"Событие рекламы: {event.event_type} для сессии {event.session_id}")
    
    # Если реклама просмотрена до конца, отправить постбэк
    if event.event_type == "complete":
        # В продакшене получали бы из БД
        block_id = "demo_block"  # В реальной системе сопоставляли бы по session_id
        postback_url = AD_BLOCKS[block_id]["postback_url"]
        
        # В продакшене: Использовать асинхронный HTTP клиент
        print(f"Отправка постбэка на: {postback_url}")
        # requests.post(postback_url, json={"status": "completed", "session_id": event.session_id})
    
    return {"status": "ok"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## 2. SDK (ads-sdk.js)

```javascript
/**
 * SDK рекламной платформы для Telegram Mini Apps
 * Готово для продакшена с обработкой ошибок и телеметрией
 */
(function(window) {
    'use strict';
    
    const VERSION = '1.0.0';
    const API_BASE_URL = 'http://localhost:8000/api'; // Заменить на продакшен URL
    
    class AdError extends Error {
        constructor(message, code) {
            super(message);
            this.name = 'AdError';
            this.code = code || 'AD_ERROR';
        }
    }
    
    class AdController {
        constructor(blockId, options = {}) {
            if (!blockId) throw new AdError('Требуется blockId', 'INVALID_CONFIG');
            
            this.blockId = blockId;
            this.options = {
                debug: false,
                ...options
            };
            this.currentSession = null;
            this.iframe = null;
        }
        
        async show() {
            try {
                // Получение данных пользователя Telegram WebApp
                const tgData = this._getTelegramData();
                
                // Запрос рекламы с сервера
                const { ad, session } = await this._requestAd(tgData);
                
                // Сохранение текущей сессии
                this.currentSession = session;
                
                // Показать рекламу в iframe
                return new Promise((resolve, reject) => {
                    this._showAdInIframe(ad, resolve, reject);
                });
            } catch (error) {
                this._logError(error);
                throw error;
            }
        }
        
        async _requestAd(tgData) {
            const response = await fetch(`${API_BASE_URL}/ads/request`, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify({
                    block_id: this.blockId,
                    user_id: tgData.user?.id || 'unknown',
                    chat_instance: tgData.chat_instance || 'unknown',
                    is_premium: tgData.user?.is_premium || false,
                    language: tgData.user?.language_code || 'en',
                    platform: tgData.platform || 'unknown'
                })
            });
            
            if (!response.ok) {
                const error = await response.json().catch(() => ({}));
                throw new AdError(
                    error.detail || 'Ошибка запроса рекламы', 
                    error.code || 'AD_REQUEST_FAILED'
                );
            }
            
            return await response.json();
        }
        
        _showAdInIframe(ad, onComplete, onError) {
            // Создание контейнера для iframe
            const container = document.createElement('div');
            container.style.position = 'fixed';
            container.style.top = '0';
            container.style.left = '0';
            container.style.width = '100%';
            container.style.height = '100%';
            container.style.backgroundColor = 'rgba(0,0,0,0.9)';
            container.style.zIndex = '9999';
            
            // Создание iframe
            this.iframe = document.createElement('iframe');
            this.iframe.src = this._getAdIframeUrl(ad);
            this.iframe.style.border = 'none';
            this.iframe.style.width = '100%';
            this.iframe.style.height = '100%';
            
            // Добавление обработчика сообщений
            window.addEventListener('message', this._handleIframeMessage.bind(this, onComplete, onError));
            
            // Добавление в DOM
            container.appendChild(this.iframe);
            document.body.appendChild(container);
        }
        
        _getAdIframeUrl(ad) {
            // В продакшене указывает на сервер с рекламным контентом
            return `http://localhost:3000/ad-iframe.html?adId=${ad.id}&type=${ad.type}`;
        }
        
        _handleIframeMessage(onComplete, onError, event) {
            if (!this.currentSession) return;
            
            try {
                const { type, data } = JSON.parse(event.data);
                
                switch (type) {
                    case 'ad_started':
                        this._trackEvent('start');
                        break;
                        
                    case 'ad_completed':
                        this._trackEvent('complete');
                        this._closeAd();
                        onComplete();
                        break;
                        
                    case 'ad_closed':
                        this._trackEvent('close');
                        this._closeAd();
                        onComplete(); // Для interstitial рекламы
                        break;
                        
                    case 'ad_error':
                        this._trackEvent('error', { error: data.error });
                        this._closeAd();
                        onError(new AdError(data.error.message, data.error.code));
                        break;
                }
            } catch (error) {
                this._logError(error);
            }
        }
        
        async _trackEvent(eventType, extraData = {}) {
            if (!this.currentSession) return;
            
            try {
                await fetch(`${API_BASE_URL}/ads/event`, {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json',
                    },
                    body: JSON.stringify({
                        session_id: this.currentSession.id,
                        event_type: eventType,
                        signature: this.currentSession.signature,
                        timestamp: this.currentSession.timestamp,
                        ...extraData
                    })
                });
            } catch (error) {
                this._logError(error);
            }
        }
        
        _closeAd() {
            if (this.iframe && this.iframe.parentNode) {
                this.iframe.parentNode.remove();
                this.iframe = null;
            }
            window.removeEventListener('message', this._handleIframeMessage);
        }
        
        _getTelegramData() {
            // Получение данных из Telegram WebApp
            if (window.Telegram && window.Telegram.WebApp) {
                return {
                    user: window.Telegram.WebApp.initDataUnsafe?.user,
                    chat_instance: window.Telegram.WebApp.initDataUnsafe?.chat_instance,
                    platform: window.Telegram.WebApp.platform
                };
            }
            return {};
        }
        
        _logError(error) {
            if (this.options.debug) {
                console.error('[AdSDK]', error);
            }
            // В продакшене: Отправка ошибки в телеметрию
        }
    }
    
    // Экспорт в window
    window.AdSDK = {
        init: (blockId, options) => {
            return new AdController(blockId, options);
        },
        version: VERSION
    };
    
})(window);
```

## 3. React Frontend (Демо)

### `App.js`

```jsx
import React, { useState } from 'react';
import './App.css';

function App() {
  const [status, setStatus] = useState('Готово');
  const [reward, setReward] = useState(0);
  
  const showAd = async () => {
    try {
      setStatus('Загрузка рекламы...');
      
      // Инициализация SDK
      const adController = window.AdSDK.init('demo_block', {
        debug: true
      });
      
      // Показ рекламы
      await adController.show();
      
      // Если дошли сюда, реклама просмотрена
      setStatus('Реклама просмотрена!');
      setReward(prev => prev + 10); // Начисление награды
    } catch (error) {
      setStatus(`Ошибка: ${error.message}`);
    }
  };
  
  return (
    <div className="app">
      <h1>Демо рекламной платформы</h1>
      <div className="status">Статус: {status}</div>
      <div className="reward">Награда: {reward} очков</div>
      <button onClick={showAd} disabled={status === 'Загрузка рекламы...'}>
        Показать рекламу за награду
      </button>
      
      <div className="info">
        <h3>Как это работает:</h3>
        <ol>
          <li>Нажмите "Показать рекламу за награду"</li>
          <li>Смотрите рекламу во фрейме</li>
          <li>Досмотрите до конца для получения награды</li>
          <li>Постбэк отправляется напрямую между серверами</li>
        </ol>
      </div>
    </div>
  );
}

export default App;
```

## 4. Ad Iframe (ad-iframe.html)

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Рекламный контент</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            background: #000;
            color: #fff;
            font-family: Arial, sans-serif;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            height: 100vh;
        }
        
        .ad-container {
            text-align: center;
            max-width: 90%;
        }
        
        video {
            max-width: 100%;
            max-height: 60vh;
        }
        
        .ad-controls {
            margin-top: 20px;
        }
        
        button {
            padding: 10px 20px;
            margin: 0 10px;
            background: #6200ee;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
        }
    </style>
</head>
<body>
    <div class="ad-container">
        <h1 id="ad-title">Демо реклама</h1>
        <video id="ad-video" controls>
            <source src="" type="video/mp4">
        </video>
        <div class="ad-controls">
            <button id="complete-btn">Завершить просмотр</button>
            <button id="close-btn">Закрыть рекламу</button>
        </div>
    </div>

    <script>
        // Парсинг параметров URL
        const urlParams = new URLSearchParams(window.location.search);
        const adId = urlParams.get('adId');
        const adType = urlParams.get('type');
        
        // Уведомление родительского фрейма о готовности
        window.parent.postMessage(JSON.stringify({
            type: 'ad_ready',
            data: { adId, adType }
        }), '*');
        
        // Настройка разных типов рекламы
        if (adType === 'rewarded') {
            document.getElementById('close-btn').style.display = 'none';
        }
        
        // Настройка видео
        const video = document.getElementById('ad-video');
        video.src = 'https://samplelib.com/lib/preview/mp4/sample-5s.mp4'; // Демо видео
        
        // Уведомление о начале воспроизведения
        video.addEventListener('play', () => {
            window.parent.postMessage(JSON.stringify({
                type: 'ad_started',
                data: { adId }
            }), '*');
        });
        
        // Обработчик кнопки завершения
        document.getElementById('complete-btn').addEventListener('click', () => {
            window.parent.postMessage(JSON.stringify({
                type: 'ad_completed',
                data: { adId }
            }), '*');
        });
        
        // Обработчик кнопки закрытия (для interstitial)
        document.getElementById('close-btn').addEventListener('click', () => {
            window.parent.postMessage(JSON.stringify({
                type: 'ad_closed',
                data: { adId }
            }), '*');
        });
        
        // Обработка ошибок
        video.addEventListener('error', () => {
            window.parent.postMessage(JSON.stringify({
                type: 'ad_error',
                data: { 
                    error: {
                        message: 'Ошибка загрузки видео',
                        code: 'VIDEO_LOAD_ERROR'
                    }
                }
            }), '*');
        });
    </script>
</body>
</html>
```

## Ключевые особенности безопасности

1. **Подписанные сессии**: Каждая сессия криптографически подписана для защиты от подделки
2. **Защита постбэков**: URL постбэков никогда не раскрываются клиенту
3. **Валидация данных Telegram**: Корректная проверка данных Telegram WebApp
4. **Изоляция iframe**: Реклама запускается в изолированных iframe для безопасности
5. **Обработка ошибок**: Полноценная обработка ошибок на всех уровнях

## Инструкции по развертыванию

1. **Бэкенд**:
   ```bash
   pip install fastapi uvicorn
   uvicorn main:app --reload
   ```

2. **Фронтенд**:
   ```bash
   npx create-react-app ad-platform-demo
   # Заменить src/App.js на нашу версию
   npm start
   ```

3. **SDK**:
   - Разместить `ads-sdk.js` на CDN или статическом сервере
   - Подключить в Mini App через `<script src="путь/к/ads-sdk.js"></script>`

4. **Ad Iframe**:
   - Разместить `ad-iframe.html` на публичном сервере
   - Обновить URL в методе `_getAdIframeUrl` SDK

Этот код представляет собой готовую основу для платформы показа рекламы, которую можно расширять дополнительными функциями.
