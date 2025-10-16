const http = require('http')
const https = require('https')

const PORT = 3000
const TARGET = 'https://time.com'

function getHtml(u) {
  return new Promise((ok, no) => {
    https.get(u, r => {
      let d = ''
      r.on('data', c => d += c)
      r.on('end', () => ok(d))
    }).on('error', no)
  })
}

function clean(t) {
  return t.replace(/<[^>]*>/g, '').replace(/\s+/g, ' ').trim()
}

function getStories(h) {
  const out = []
  const done = new Set()
  const reg = /<a\b([^>]*?)href=(["'])(.*?)\2([^>]*)>([\s\S]*?)<\/a>/gi
  let m
  while ((m = reg.exec(h)) !== null) {
    let href = m[3]
    let text = m[5] || ''
    if (href.startsWith('//')) href = 'https:' + href
    else if (href.startsWith('/')) href = 'https://time.com' + href
    if (!href.startsWith('https://time.com')) continue
    if (!(/\/\d{4,}\/|\/articles\/|\/\d{5,}\//.test(href))) continue
    const title = clean(text)
    if (!title) continue
    const key = href + title
    if (done.has(key)) continue
    done.add(key)
    out.push({ title, link: href })
    if (out.length >= 6) break
  }
  return out
}

const server = http.createServer(async (req, res) => {
  if (req.method === 'GET' && req.url === '/getTimeStories') {
    try {
      const html = await getHtml(TARGET)
      const data = getStories(html)
      res.writeHead(200, { 'Content-Type': 'application/json' })
      res.end(JSON.stringify(data, null, 2))
    } catch {
      res.writeHead(500, { 'Content-Type': 'application/json' })
      res.end(JSON.stringify({ error: 'failed' }))
    }
  } else {
    res.writeHead(404)
    res.end('use /getTimeStories')
  }
})

server.listen(PORT, () => console.log(`http://localhost:${PORT}/getTimeStories`))
