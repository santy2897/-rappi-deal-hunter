const express = require('express');
const axios = require('axios');
const cors = require('cors');
const app = express();
app.use(express.json());
app.use(cors());

const VERIFY_TOKEN = 'rappi_deal_hunter_2024';
const WHATSAPP_TOKEN = process.env.WHATSAPP_TOKEN;
const PHONE_NUMBER_ID = process.env.PHONE_NUMBER_ID;
const GROQ_API_KEY = process.env.GROQ_API_KEY;
const SANTIAGO_PHONE = process.env.SANTIAGO_PHONE;

const conversations = {};

// ── WEBHOOK VERIFICATION ─────────────────────
app.get('/webhook', (req, res) => {
  const mode = req.query['hub.mode'];
  const token = req.query['hub.verify_token'];
  const challenge = req.query['hub.challenge'];
  if (mode === 'subscribe' && token === VERIFY_TOKEN) {
    console.log('Webhook verificado OK');
    res.status(200).send(challenge);
  } else {
    res.sendStatus(403);
  }
});

// ── RECIBIR MENSAJES ─────────────────────────
app.post('/webhook', async (req, res) => {
  res.sendStatus(200);
  try {
    const body = req.body;
    if (body.object !== 'whatsapp_business_account') return;
    const entry = body.entry?.[0];
    const value = entry?.changes?.[0]?.value;
    const messages = value?.messages;
    if (!messages?.length) return;
    const msg = messages[0];
    const from = msg.from;
    const text = msg.type === 'text' ? msg.text?.body : '[mensaje no textual]';
    console.log(`[${new Date().toLocaleTimeString()}] Mensaje de ${from}: ${text}`);
    if (!conversations[from]) {
      conversations[from] = {
        msgs: [], stage: 'new', stageLabel: 'Nuevo',
        comision: '—', progress: 5,
        docs: { dni:false, cbu:false, menu:false, logo:false, horarios:false },
        lastActivity: Date.now(), flagged: false
      };
    }
    const conv = conversations[from];
    conv.lastActivity = Date.now();
    conv.msgs.push({ role: 'user', content: text });
    const reply = await callAgent(conv, from);
    if (reply.needsIntervention) {
      const alertMsg = `INTERVENCION REQUERIDA\n\nProspecto: ${from}\n\n${reply.summary}\n\nRespondele directo desde tu WhatsApp.`;
      await sendWA(SANTIAGO_PHONE, alertMsg);
      conv.flagged = true;
    } else {
      await sendWA(from, reply.message);
      conv.msgs.push({ role: 'assistant', content: reply.message });
      if (reply.stage) { conv.stage = reply.stage; conv.stageLabel = reply.stageLabel; }
      if (reply.comision && reply.comision !== '—') conv.comision = reply.comision;
      if (reply.progress) conv.progress = reply.progress;
      if (reply.docs) Object.assign(conv.docs, reply.docs);
    }
  } catch(err) {
    console.error('Error webhook:', err.message);
  }
});

// ── SYNC API ──────────────────────────────────
app.post('/api/sync-prospects', (req, res) => {
  const { prospects } = req.body || {};
  if (!prospects) return res.json({ ok: false });
  res.json({ ok: true, count: prospects.length });
});

app.get('/api/get-updates', (req, res) => {
  const updates = Object.entries(conversations).map(([phone, conv]) => ({
    phone, stage: conv.stage, stageLabel: conv.stageLabel,
    comision: conv.comision, progress: conv.progress,
    docs: conv.docs, flagged: conv.flagged
  }));
  res.json({ updates });
});

// ── ENVIAR MENSAJE WA ─────────────────────────
async function sendWA(to, text) {
  await axios.post(
    `https://graph.facebook.com/v18.0/${PHONE_NUMBER_ID}/messages`,
    { messaging_product: 'whatsapp', to, type: 'text', text: { body: text } },
    { headers: { Authorization: `Bearer ${WHATSAPP_TOKEN}`, 'Content-Type': 'application/json' } }
  );
}

// ── GROQ AGENT ───────────────────────────────
async function callAgent(conv, phone) {
  const systemMsg = `Sos Santiago Rojas, Deal Hunter del equipo de Alianzas Estratégicas de Rappi Argentina. Tu objetivo es convencer a dueños de restaurantes para que se sumen a Rappi.

PROPUESTA:
- Radio de cobertura: 6km (supera a PedidosYa)
- Pagos semanales: lunes a domingo, acreditado el miércoles siguiente
- Comisión genérica: 30%, negociable
- Oferta: 23% final + 3 meses publicidad gratis + primeros 2 meses al 0%

TONO: corto, directo, amistoso, sin emojis, serio, confiable.

FLUJO:
1. Saludo breve y presentación
2. Preguntar comisión actual en PedidosYa
3. Ofrecer 23% + 3 meses publicidad + 2 meses bonificados
4. Manejar objeciones
5. Pedir docs de a uno: DNI frente y dorso, CBU/alias, menú, logo, horarios

OBJECIONES:
- Ya tengo PedidosYa: Rappi es canal adicional, radio 6km, 2 meses gratis sin riesgo
- Comisión alta: 23% con 2 meses al 0% está por debajo del mercado
- No tengo tiempo: Solo necesito 5 datos y lo activo yo
- No me interesa: Solo una pregunta, cuanto pagan hoy de comision?

PEDIR INTERVENCION (needsIntervention true) cuando:
- Objecion muy agresiva o definitiva
- Pide hablar en persona o por llamada
- Toda la documentacion lista, cierre formal
- Condicion fuera de la propuesta estandar

Cuando pidas intervencion, summary debe tener: quien es, etapa, problema exacto, sugerencia para Santiago.

Estado actual: Etapa=${conv.stage} | Comision PYA=${conv.comision} | Progreso=${conv.progress}%
Docs: DNI=${conv.docs.dni}, CBU=${conv.docs.cbu}, Menu=${conv.docs.menu}, Logo=${conv.docs.logo}, Horarios=${conv.docs.horarios}

RESPONDE SOLO con este JSON sin markdown ni texto extra:
{"message":"respuesta","needsIntervention":false,"summary":"","stage":"new","stageLabel":"Nuevo","comision":"","progress":5,"docs":{"dni":false,"cbu":false,"menu":false,"logo":false,"horarios":false}}`;

  const messages = [
    { role: 'system', content: systemMsg },
    ...conv.msgs
  ];

  const res = await axios.post(
    'https://api.groq.com/openai/v1/chat/completions',
    {
      model: 'llama-3.3-70b-versatile',
      max_tokens: 800,
      temperature: 0.3,
      messages
    },
    {
      headers: {
        'Authorization': `Bearer ${GROQ_API_KEY}`,
        'Content-Type': 'application/json'
      }
    }
  );

  const raw = res.data.choices[0].message.content.trim();
  try {
    return JSON.parse(raw.replace(/```json|```/g, '').trim());
  } catch {
    return {
      message: raw,
      needsIntervention: false,
      summary: '',
      stage: conv.stage,
      stageLabel: conv.stageLabel,
      comision: conv.comision,
      progress: conv.progress,
      docs: conv.docs
    };
  }
}

// ── STATUS ────────────────────────────────────
app.get('/', (req, res) => {
  res.json({
    status: 'ok',
    agent: 'Rappi Deal Hunter',
    model: 'llama-3.3-70b (Groq)',
    conversations: Object.keys(conversations).length,
    uptime: process.uptime()
  });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`Rappi Deal Hunter corriendo en puerto ${PORT}`));
