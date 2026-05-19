# Corrigir o disconnect do WordPress MCP

Guia para estabilizar o conector MCP self-hosted apontado para
`https://musicaemercado.org/wp-json/mcp/wp-mcp-ultimate` que aparece como
"Server disconnected. For troubleshooting guidance, please visit our
debugging documentation".

> Importante: a configuração do conector MCP **não vive neste
> repositório**. Ela é gerenciada no painel de Connectors do Claude Code
> em https://code.claude.com. Este guia descreve mudanças que precisam
> ser feitas no **WordPress** e, se necessário, no painel de Connectors.

## 1. Identificar qual conector está caindo

No diagnóstico desta sessão existem dois conectores WordPress:

| Conector | Endpoint | Status |
|---|---|---|
| WordPress.com oficial (`ad42fa47…`) | `public-api.wordpress.com/wpcom/v2/mcp/v1` | OK |
| Self-hosted (`9bfa2fdd…`) | `musicaemercado.org/wp-json/mcp/wp-mcp-ultimate` | Disconnect |

O problema está apenas no **self-hosted**. Se você só precisa publicar em
musicaemercado.org via WordPress.com, considere remover o conector
self-hosted no painel Connectors e usar só o oficial.

## 2. Testar o endpoint a partir da sua máquina

```bash
# 2.1 - O caminho /wp-json responde?
curl -sS -i https://musicaemercado.org/wp-json/ | head -20

# 2.2 - A rota MCP existe?
curl -sS -i https://musicaemercado.org/wp-json/mcp/wp-mcp-ultimate | head -20

# 2.3 - Handshake MCP (initialize) - deve retornar 200 com JSON em < 10s
curl -sS -i --max-time 30 -X POST \
  https://musicaemercado.org/wp-json/mcp/wp-mcp-ultimate \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"diag","version":"1.0"}}}'
```

Interprete a saída de 2.3:

- **200 + JSON** com `result.serverInfo` → o servidor está saudável; o
  problema é intermitência (vá para a seção 5).
- **401 / 403** → autenticação. Vá para a seção 3.
- **404** → o plugin WP MCP Ultimate não está ativo ou a rota mudou. Vá
  para a seção 4.
- **5xx** ou **timeout** → PHP travando. Vá para a seção 4.
- **Resposta HTML do Cloudflare/Wordfence** → firewall bloqueando. Vá
  para a seção 6.

## 3. Autenticação

Se o conector usa Application Password do WordPress:

1. Em `wp-admin → Usuários → Seu perfil → Application Passwords`, revogue
   o token antigo e crie um novo.
2. No painel Connectors do Claude Code, edite o conector self-hosted e
   cole o novo token.
3. Reabra a sessão.

Se usa OAuth/JWT do plugin, gere uma nova credencial pelo painel do
próprio plugin.

## 4. Plugin WP MCP Ultimate

1. Atualize para a versão mais recente: `wp-admin → Plugins`.
2. Desative e reative para forçar re-registro das rotas REST.
3. Salve permalinks: `wp-admin → Configurações → Links permanentes →
   Salvar`. Isso reescreve as regras do `.htaccess`/Nginx e corrige
   rotas REST que somem após updates.
4. Confirme que a rota aparece:
   ```bash
   curl -sS https://musicaemercado.org/wp-json/ | python3 -m json.tool | grep -A1 mcp
   ```
5. Habilite log do plugin (geralmente em `wp-content/debug.log`):
   ```php
   // wp-config.php
   define( 'WP_DEBUG', true );
   define( 'WP_DEBUG_LOG', true );
   define( 'WP_DEBUG_DISPLAY', false );
   ```
   Reproduza o disconnect e cheque `wp-content/debug.log` por
   `Fatal error`, `Maximum execution time` ou `Allowed memory size`.

## 5. Timeouts e estabilidade do PHP

O handshake MCP às vezes mantém conexão aberta (SSE). Hosts com
`max_execution_time` baixo cortam a conexão e o cliente reporta
"disconnected".

No `php.ini` (ou no painel da hospedagem) suba para valores como:

```ini
max_execution_time = 120
max_input_time = 120
memory_limit = 256M
default_socket_timeout = 120
```

No Nginx/Apache, garanta:

- `proxy_read_timeout 120s;` (Nginx)
- `Timeout 120` (Apache)
- Keep-alive habilitado.
- Se houver `fastcgi_buffering on` e o plugin usar SSE, troque para
  `fastcgi_buffering off;` ou adicione cabeçalho `X-Accel-Buffering: no`
  nas respostas do plugin.

## 6. Firewall / CDN

Wordfence, Cloudflare ou plugin de segurança costumam bloquear:

- POSTs repetidos do mesmo IP (rate limit).
- `Accept: text/event-stream` (consideram scraping).
- Range de IPs de cloud (AWS/GCP) — a Anthropic faz proxy a partir desses.

Ações:

1. **Cloudflare**: crie uma regra `Page Rule` ou `Configuration Rule`
   para `musicaemercado.org/wp-json/mcp/*` com:
   - Security Level: Essentially Off
   - Cache: Bypass
   - Browser Integrity Check: Off
2. **Wordfence**: em `Firewall → Allowlisted URLs`, adicione
   `/wp-json/mcp/wp-mcp-ultimate`. Em `Rate Limiting`, suba o limite
   para essa URL ou desative para rotas REST autenticadas.
3. Confirme que não há plugin tipo "Limit Login Attempts" tratando os
   POSTs de MCP como login.

## 7. HTTPS / certificado

```bash
echo | openssl s_client -servername musicaemercado.org \
  -connect musicaemercado.org:443 2>/dev/null \
  | openssl x509 -noout -dates -issuer
```

Cadeia incompleta ou cert expirado também causam disconnect silencioso.
Renove via Let's Encrypt / painel da hospedagem.

## 8. Verificação final

Após aplicar os ajustes:

1. Rode novamente o `curl` do passo 2.3 — espere 200 + JSON consistente
   em < 5s, repetido 5 vezes seguidas.
2. Abra uma nova sessão do Claude Code on the web.
3. Confirme que as ferramentas `mcp__9bfa2fdd…__*` aparecem no início
   da sessão sem a mensagem "Server disconnected".

## 9. Se mesmo assim cair: troque pelo WordPress.com oficial

Você já tem o conector `wpcom-*` funcionando. Para a maioria dos casos
(criar/editar post, página, mídia, taxonomia, pattern) ele substitui o
self-hosted. Remova o conector self-hosted no painel Connectors enquanto
debuga o plugin, para não receber mais a mensagem de erro a cada
sessão.
