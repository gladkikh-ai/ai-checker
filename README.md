# AI Visibility Checker — Signum.AI

Бесплатный лид-магнит: проверка AI-готовности сайта. Деплоится на Vercel за 2 минуты.

## Что проверяет

| Метрика | Баллы | Как проверяет |
|---------|-------|---------------|
| Agent-readable content | /20 | Считает слова в сыром HTML без JS |
| Server-side rendering | /10 | Есть ли контент без исполнения JS |
| AI agent access | /15 | Парсит robots.txt для каждого бота |
| llms.txt | /15 | Запрос к /llms.txt |
| Markdown availability | /15 | Tied to llms.txt (можно расширить) |
| Token economics | /15 | Размер HTML / 4 ≈ токены |
| Performance | /10 | Реальный response time |

## Деплой на Vercel (5 минут)

### 1. Загрузи на GitHub

```bash
git init
git add .
git commit -m "AI checker"
gh repo create ai-checker --public --push
```

### 2. Задеплой на Vercel

1. Заходишь на vercel.com
2. "Import Project" → выбираешь репо
3. Framework Preset: **Other**
4. Root Directory: оставить пустым
5. Deploy → готово

Vercel автоматически поднимет `/api/check.js` как serverless Edge Function.

### 3. Вставь на WordPress (опционально)

Если хочешь на поддомен WP (`tools.signum.ai`):
- Добавь CNAME `tools.signum.ai → cname.vercel-dns.com` в DNS
- В Vercel добавь кастомный домен

Либо вставь как отдельный iframe на страницу WP:
```html
<iframe src="https://ai-checker.vercel.app" width="100%" height="900px" frameborder="0"></iframe>
```

## Что сделать после деплоя

- [ ] Поменяй ссылки `https://my.signum.ai` и `https://signum.ai#book-demo` в `public/index.html` под реальные UTM-параметры
- [ ] Добавь аналитику (GA4 или Plausible) в `<head>`
- [ ] Опционально: добавь захват email через Zapier webhook перед показом результатов

## Расширение: реальные боты

Сейчас бот-доступ проверяется через robots.txt логику.
Для реальной симуляции (как Siteline) нужен доп. Edge Function,
который шлёт запрос с нужным User-Agent:

```js
// api/bot-check.js
const res = await fetch(url, {
  headers: { 'User-Agent': 'GPTBot/1.0' }
});
```

Это работает на Vercel Edge, но некоторые сайты детектируют дата-центр Vercel.
