// Cloudflare Worker for Writer98 share server

const MAX_AUDIT_ENTRIES = 20;
const corsHeaders = {
  'Access-Control-Allow-Origin': 'http://localhost:8004',
  'Access-Control-Allow-Methods': 'POST, OPTIONS',
  'Access-Control-Allow-Headers': 'Content-Type, Cache-Control',
  'Access-Control-Max-Age': '86400'
};

const jsonResponse = (payload, status = 200) =>
  new Response(JSON.stringify(payload), {
    status,
    headers: { ...corsHeaders, 'content-type': 'application/json' }
  });

const normalizeSlug = value => (typeof value === 'string' ? value.trim() : '');
const normalizeCollab = value => normalizeSlug(value).replace(/\s+/g, '-');
const buildDocKey = (slug, collab) => `doc:${slug}:${collab}`;
const buildAuditKey = slug => `audit:${slug}`;
const buildPublishKey = key => `publish:${key}`;

const appendAuditEntry = async (slug, entry) => {
  if (!slug) return;
  const key = buildAuditKey(slug);
  const current = (await AUDIT_KV.get(key, 'json')) || [];
  current.push(entry);
  if (current.length > MAX_AUDIT_ENTRIES) {
    current.splice(0, current.length - MAX_AUDIT_ENTRIES);
  }
  await AUDIT_KV.put(key, JSON.stringify(current));
};

const generatePublishKey = () =>
  (typeof crypto?.randomUUID === 'function' ? crypto.randomUUID() : `${Date.now()}-${Math.random()}`);

const extractIp = request => request.headers.get('CF-Connecting-IP') || 'unknown';

// Map allowed bucket names (as provided by clients) to actual Worker R2 binding names.
// Configure these mappings to match your Cloudflare Worker bindings. For example,
// { 'my-bucket': 'MY_R2_BUCKET' } means the client may send bucket='my-bucket'
// and the Worker will use the R2 binding named MY_R2_BUCKET.
const BUCKET_BINDINGS = {
  // Example mapping - update to match your environment
  'my-bucket': 'MY_R2_BUCKET'
};

const handleSync = async (request, body) => {
  const slug = normalizeSlug(body.slug);
  const collabCode = normalizeCollab(body.collabCode);
  const content = typeof body.content === 'string' ? body.content : '';
  const updatedAt = Number(body.updatedAt) || Date.now();
  if (!slug || !collabCode || !content) {
    return jsonResponse({ status: 'error', message: 'Missing slug/collabCode/content' }, 400);
  }
  const key = buildDocKey(slug, collabCode);
  const stored = (await DOC_KV.get(key, 'json')) || null;
  let authoritative = stored;
  if (!stored || updatedAt > (stored.updatedAt || 0)) {
    authoritative = { content, updatedAt, collabCode, lastEditorIp: extractIp(request) };
    await DOC_KV.put(key, JSON.stringify(authoritative));
  }
  await appendAuditEntry(slug, {
    collabCode,
    timestamp: Date.now(),
    ip: extractIp(request)
  });
  return jsonResponse({
    status: 'ok',
    message: 'Sync complete',
    payload: {
      content: authoritative.content,
      updatedAt: authoritative.updatedAt || updatedAt
    }
  });
};

const handlePublish = async body => {
  const slug = normalizeSlug(body.slug);
  const collabCode = normalizeCollab(body.collabCode) || 'default';
  const content = typeof body.content === 'string' ? body.content : '';
  const updatedAt = Number(body.updatedAt) || Date.now();
  if (!slug || !content) {
    return jsonResponse({ status: 'error', message: 'Missing slug or content' }, 400);
  }
  const publishKey = generatePublishKey();
  await PUBLISH_KV.put(
    buildPublishKey(publishKey),
    JSON.stringify({ slug, content, updatedAt, collabCode, createdAt: Date.now() })
  );
  return jsonResponse({
    status: 'ok',
    message: 'Published',
    payload: { publishKey }
  });
};

const handleUnpublish = async body => {
  const publishKey = normalizeSlug(body.publishKey);
  if (!publishKey) {
    return jsonResponse({ status: 'error', message: 'Missing publish key' }, 400);
  }
  const key = buildPublishKey(publishKey);
  const existing = await PUBLISH_KV.get(key);
  if (!existing) {
    return jsonResponse({ status: 'error', message: 'Publish key invalid' }, 404);
  }
  await PUBLISH_KV.delete(key);
  return jsonResponse({ status: 'ok', message: 'Unpublished' });
};

const handleFetchPublish = async body => {
  const publishKey = normalizeSlug(body.publishKey);
  if (!publishKey) {
    return jsonResponse({ status: 'error', message: 'Missing publish key' }, 400);
  }
  const raw = await PUBLISH_KV.get(buildPublishKey(publishKey), 'json');
  if (!raw) {
    return jsonResponse({ status: 'error', message: 'Publish link expired or invalid' }, 404);
  }
  return jsonResponse({
    status: 'ok',
    message: 'Publish entry fetched',
    payload: {
      content: raw.content,
      updatedAt: raw.updatedAt,
      slug: raw.slug,
      readOnly: true
    }
  });
};

const handleR2Push = async body => {
  const bucket = (typeof body.bucket === 'string' ? body.bucket.trim() : '');
  const key = (typeof body.key === 'string' ? body.key.trim() : '');
  const strategy = body.strategy === 'timestamp' ? 'timestamp' : 'overwrite';
  const payload = body.payload;
  if (!bucket || !key || payload === undefined) {
    return jsonResponse({ ok: false, error: 'Missing bucket/key/payload' }, 400);
  }
  const bindingName = BUCKET_BINDINGS[bucket];
  if (!bindingName) {
    return jsonResponse({ ok: false, error: 'Unknown bucket' }, 400);
  }
  const r2 = globalThis[bindingName];
  if (!r2 || typeof r2.put !== 'function') {
    return jsonResponse({ ok: false, error: `R2 binding not configured: ${bindingName}` }, 500);
  }

  try {
    const updatedAt = Date.now();
    let targetKey = key;
    if (strategy === 'timestamp') {
      // Save backups with a timestamp suffix so original key is preserved
      const ts = updatedAt;
      const ext = key.includes('.') ? '' : '.json';
      targetKey = `${key}.backup.${ts}${ext}`;
    }
    const value = typeof payload === 'string' ? payload : JSON.stringify(payload);
    await r2.put(targetKey, value, { httpMetadata: { contentType: 'application/json' }, metadata: { updatedAt: String(updatedAt) } });
    return jsonResponse({ ok: true, updatedAt, key: targetKey }, 200);
  } catch (err) {
    return jsonResponse({ ok: false, error: `R2 write failed: ${err.message}` }, 500);
  }
};

const handleR2Pull = async body => {
  const bucket = (typeof body.bucket === 'string' ? body.bucket.trim() : '');
  const key = (typeof body.key === 'string' ? body.key.trim() : '');
  if (!bucket || !key) {
    return jsonResponse({ ok: false, error: 'Missing bucket/key' }, 400);
  }
  const bindingName = BUCKET_BINDINGS[bucket];
  if (!bindingName) {
    return jsonResponse({ ok: false, error: 'Unknown bucket' }, 400);
  }
  const r2 = globalThis[bindingName];
  if (!r2 || typeof r2.get !== 'function') {
    return jsonResponse({ ok: false, error: `R2 binding not configured: ${bindingName}` }, 500);
  }
  try {
    const obj = await r2.get(key);
    if (!obj) return jsonResponse({ ok: false, error: 'Object not found' }, 404);
    const text = await obj.text();
    let parsed = null;
    try {
      parsed = JSON.parse(text);
    } catch (e) {
      // If payload isn't JSON, return raw text
      parsed = text;
    }
    const updatedAt = obj?.metadata?.updatedAt ? Number(obj.metadata.updatedAt) : null;
    return jsonResponse({ ok: true, payload: parsed, updatedAt }, 200);
  } catch (err) {
    return jsonResponse({ ok: false, error: `R2 read failed: ${err.message}` }, 500);
  }
};

const parseJson = async request => {
  try {
    return await request.json();
  } catch (error) {
    return null;
  }
};

const handleRequest = async request => {
  if (request.method === 'OPTIONS') {
    return new Response(null, { status: 204, headers: corsHeaders });
  }
  if (request.method !== 'POST') {
    return jsonResponse({ status: 'error', message: 'POST required' }, 405);
  }
  const body = await parseJson(request);
  if (!body || typeof body.action !== 'string') {
    return jsonResponse({ status: 'error', message: 'Invalid payload' }, 400);
  }
  switch (body.action) {
    case 'sync':
      return handleSync(request, body);
    case 'publish':
      return handlePublish(body);
    case 'unpublish':
      return handleUnpublish(body);
    case 'fetch-publish':
      return handleFetchPublish(body);
    default:
      return jsonResponse({ status: 'error', message: 'Unknown action' }, 400);
  }
};

addEventListener('fetch', event => {
  event.respondWith(
    handleRequest(event.request).catch(error =>
      jsonResponse({ status: 'error', message: `Worker failure: ${error.message}` }, 500)
    )
  );
});
