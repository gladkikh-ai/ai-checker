export const config = { runtime: 'edge' };

const AI_BOTS = [
  { name: 'GPTBot', ua: 'GPTBot' },
  { name: 'ChatGPT-User', ua: 'ChatGPT-User' },
  { name: 'Claude-SearchBot', ua: 'Claude-SearchBot' },
  { name: 'ClaudeBot', ua: 'ClaudeBot' },
  { name: 'PerplexityBot', ua: 'PerplexityBot' },
  { name: 'Googlebot', ua: 'Googlebot' },
  { name: 'Meta-ExternalAgent', ua: 'Meta-ExternalAgent' },
  { name: 'Gemini', ua: 'Google-Extended' },
];

function normalizeDomain(input) {
  let d = input.trim().toLowerCase();
  d = d.replace(/^https?:\/\//, '');
  d = d.replace(/^www\./, '');
  d = d.split('/')[0];
  return d;
}

async function fetchWithTimeout(url, options = {}, timeout = 8000) {
  const controller = new AbortController();
  const id = setTimeout(() => controller.abort(), timeout);
  try {
    const res = await fetch(url, { ...options, signal: controller.signal });
    clearTimeout(id);
    return res;
  } catch (e) {
    clearTimeout(id);
    throw e;
  }
}

function parseRobotsTxt(text) {
  const lines = text.split('\n').map(l => l.trim());
  const rules = {};
  let currentAgent = null;

  for (const line of lines) {
    if (line.startsWith('#') || !line) continue;
    const [key, ...rest] = line.split(':');
    const val = rest.join(':').trim();
    if (key.toLowerCase() === 'user-agent') {
      currentAgent = val.toLowerCase();
      if (!rules[currentAgent]) rules[currentAgent] = [];
    } else if (key.toLowerCase() === 'disallow' && currentAgent) {
      rules[currentAgent].push(val);
    }
  }
  return rules;
}

function isBotBlocked(rules, botUa) {
  const ua = botUa.toLowerCase();
  const star = rules['*'] || [];
  const specific = rules[ua] || [];
  const allRules = [...specific, ...star];
  // blocked if disallow: / or disallow: (non-empty path covering root)
  return allRules.some(r => r === '/' || r === '');
}

function countWords(html) {
  const text = html.replace(/<[^>]+>/g, ' ').replace(/\s+/g, ' ').trim();
  return text.split(' ').filter(w => w.length > 1).length;
}

function estimateTokens(html) {
  // rough: 1 token ≈ 4 chars
  return Math.round(html.length / 4);
}

export default async function handler(req) {
  const url = new URL(req.url);
  const domain = url.searchParams.get('domain');

  if (!domain) {
    return new Response(JSON.stringify({ error: 'No domain provided' }), {
      status: 400,
      headers: { 'Content-Type': 'application/json', 'Access-Control-Allow-Origin': '*' },
    });
  }

  const d = normalizeDomain(domain);
  const baseUrl = `https://${d}`;
  const results = { domain: d };

  // 1. Fetch main page
  let mainHtml = '';
  let responseTime = null;
  let isSSR = false;
  let statusOk = false;

  try {
    const t0 = Date.now();
    const res = await fetchWithTimeout(baseUrl, {
      headers: { 'User-Agent': 'Mozilla/5.0 (compatible; SitecheckBot/1.0)' }
    });
    responseTime = Date.now() - t0;
    statusOk = res.ok;
    mainHtml = await res.text();
    // SSR: substantial content in raw HTML (not just script tags)
    const textContent = mainHtml.replace(/<script[^>]*>[\s\S]*?<\/script>/gi, '')
      .replace(/<style[^>]*>[\s\S]*?<\/style>/gi, '');
    const wordCount = countWords(textContent);
    isSSR = wordCount > 200;
    results.wordCount = wordCount;
    results.tokenCount = estimateTokens(mainHtml);
  } catch (e) {
    results.error = 'Could not reach domain';
  }

  results.responseTime = responseTime;
  results.isSSR = isSSR;
  results.statusOk = statusOk;

  // 2. robots.txt
  let robotsRules = {};
  let robotsExists = false;
  try {
    const res = await fetchWithTimeout(`${baseUrl}/robots.txt`);
    if (res.ok) {
      const text = await res.text();
      robotsExists = text.length > 10;
      robotsRules = parseRobotsTxt(text);
    }
  } catch (e) {}
  results.robotsExists = robotsExists;

  // 3. llms.txt
  let llmsExists = false;
  try {
    const res = await fetchWithTimeout(`${baseUrl}/llms.txt`);
    llmsExists = res.ok && (await res.text()).length > 10;
  } catch (e) {}
  results.llmsExists = llmsExists;

  // 4. sitemap
  let sitemapExists = false;
  try {
    const res = await fetchWithTimeout(`${baseUrl}/sitemap.xml`);
    if (res.ok) {
      const text = await res.text();
      sitemapExists = text.includes('<urlset') || text.includes('<sitemapindex');
    }
  } catch (e) {}
  // also check robots.txt for sitemap
  if (!sitemapExists && robotsExists) {
    // check via robots
    const res2 = await fetchWithTimeout(`${baseUrl}/sitemap_index.xml`).catch(() => null);
    if (res2?.ok) sitemapExists = true;
  }
  results.sitemapExists = sitemapExists;

  // 5. Markdown / llms.txt content
  results.markdownAvailable = llmsExists;

  // 6. Bot access simulation based on robots.txt
  results.bots = AI_BOTS.map(bot => {
    const blocked = isBotBlocked(robotsRules, bot.ua);
    return {
      name: bot.name,
      robotsAllowed: robotsExists ? !blocked : true,
      // simulated access: if robots allows but we couldn't verify actual connection
      simulatedAccess: statusOk && (robotsExists ? !blocked : true),
    };
  });

  // 7. Score calculation
  let score = 0;
  // Agent-readable content (20)
  if (results.wordCount > 500) score += 20;
  else if (results.wordCount > 200) score += 10;
  // SSR (10)
  if (isSSR) score += 10;
  // AI agent access — bots not blocked (15)
  const allowedBots = results.bots.filter(b => b.robotsAllowed).length;
  score += Math.round((allowedBots / AI_BOTS.length) * 15);
  // llms.txt (15)
  if (llmsExists) score += 15;
  // Markdown (15)
  if (results.markdownAvailable) score += 15;
  // Performance (10)
  if (responseTime && responseTime < 600) score += 10;
  else if (responseTime && responseTime < 1200) score += 5;
  // Token economics (15)
  if (results.tokenCount && results.tokenCount < 8000 * 4) score += 15;
  else if (results.tokenCount && results.tokenCount < 50000 * 4) score += 7;

  results.score = score;

  return new Response(JSON.stringify(results), {
    status: 200,
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*',
    },
  });
}
