// ==UserScript==
// @name         SIS Bot
// @namespace    https://sis-agora.com
// @version      1.5.9
// @description  Bot do Sis-Ágora para o SAGE Labs: motor do SLY Assistant 0.7.35, configurado pelo dashboard do Sis-Ágora com aceite local.
// @author       Sis-Ágora (motor: SLY Assistant — SLY w/ Contributions by niofox, SkyLove512, anthonyra, [AEP] Valkynen, Risingson, Swift42)
// @match        https://*.based.staratlas.com/sis-bot
// @require      https://unpkg.com/@solana/web3.js@1.95.8/lib/index.iife.min.js#sha256=a759deca1b65df140e8dda5ad8645c19579536bf822e5c0c7e4adb7793a5bd08
// @require      https://raw.githubusercontent.com/sisagora/Sis-Bot/main/lib/anchor-browserified.js#sha256=f29ef75915bcf59221279f809eefc55074dbebf94cf16c968e783558e7ae3f0a
// @require      https://raw.githubusercontent.com/sisagora/Sis-Bot/main/lib/buffer-browserified.js#sha256=4fa88e735f9f1fdbff85f4f92520e8874f2fec4e882b15633fad28a200693392
// @require      https://raw.githubusercontent.com/sisagora/Sis-Bot/main/lib/bs58-browserified.js#sha256=87095371ec192e5a0e50c6576f327eb02532a7c29f1ed86700a2f8fb5018d947
// @icon         https://www.google.com/s2/favicons?sz=64&domain=staratlas.com
// @grant        GM_setValue
// @grant        GM_getValue
// @grant        GM_deleteValue
// @grant        GM_listValues
// @grant        GM_xmlhttpRequest
// @connect      clashofiris.com
// @noframes
// ==/UserScript==

(async function() {
    'use strict';

    // ==================================================================================================
    // [Sis-Ágora 1.3.0] Pool de RPCs com failover
    // Até a 1.2.7 o Sis-Bot usava um RPC só (Ironforge com a chave pública do SLYA). Quando essa chave
    // estourou a cota grátis ("Your free tier request limit has been reached", HTTP 429), toda leitura
    // e todo envio de transação falhava e o doProxyStuff antigo ficava preso para sempre tentando o
    // mesmo RPC: as frotas paravam e o painel continuava "rodando".
    // Agora:
    //  - vários RPCs, em ordem de prioridade: os seus (deste navegador) → os do dashboard (Administração › RPCs)
    //    → a lista padrão abaixo (só quando o dashboard não mandou nenhuma);
    //  - limite de requisições por segundo por RPC (evita o 429 antes de acontecer);
    //  - 429/5xx/timeout/cota esgotada => o RPC fica de castigo (com recuo exponencial ou o Retry-After)
    //    e a mesma requisição vai na hora para o próximo;
    //  - método que um RPC não aceita (ex.: getProgramAccounts sem token) é aprendido e não volta para ele;
    //  - sendTransaction vai para todos os RPCs disponíveis ao mesmo tempo (broadcast);
    //  - nada de URL com chave no console, no log de erros ou no dashboard (só o host).
    // GM 'sisRpcs' = seus RPCs (painel Sis-Bot); GM 'sisRpcsDash' = última lista recebida do dashboard.
    // ==================================================================================================
    const SIS_RPC_KEY = 'sisRpcs';
    const SIS_RPC_DASH_KEY = 'sisRpcsDash';
    // [1.5.1] lista padrão = só o RPC oficial do jogo. É o único usado quando o dashboard não mandou lista (ou mandou vazia).
    // Nenhum outro RPC é usado se o dashboard não o enviou habilitado agora.
    const SIS_RPC_PADRAO = [
        { url: 'https://mainnet-rpc.staratlas.com/mainnet?apiKey=01JEEEQP3FTZJFCP5RCCKB2NSQ', nome: 'Star Atlas (oficial)', rps: 10 },
    ];
    const SIS_RPC_TENTATIVAS = 8;      // tentativas por requisição antes de devolver erro ao web3.js
    const SIS_RPC_TIMEOUT = 20000;     // ms (getProgramAccounts/getMultipleAccounts: 45 s)
    const SIS_RPC_COTA_MS = 2 * 60000; // castigo quando a cota do plano grátis acabou (1.3.4: 2 min; a sonda traz de volta antes se responder)
    const SIS_RPC_DASH_VALIDADE = 10 * 60000; // [1.5.1] lista guardada do dashboard só vale na partida se foi confirmada há menos de 10 min
    // [1.5.1] item marcado como desabilitado pelo dashboard nunca entra
    const sisRpcHabilitado = (x) => !!x && x.ativo !== false && x.habilitado !== false && x.enabled !== false && x.ativo !== 0 && x.ativo !== '0' && x.habilitado !== 0 && x.habilitado !== '0';
    const sisRpcUrlOk = (u) => typeof u === 'string' && u.length <= 400 && /^https:\/\/[^\s"'<>\\]+$/.test(u);

    let sisRpcProprios = [];
    try {
        const v = JSON.parse(await GM.getValue(SIS_RPC_KEY, '[]'));
        if (Array.isArray(v)) sisRpcProprios = v.map(String).map(u => u.trim()).filter(sisRpcUrlOk).slice(0, 8);
    } catch (e) { sisRpcProprios = []; }
    // lista do dashboard (só usada se "Usar os RPCs do dashboard" estiver ligado no painel do Sis-Bot)
    let sisRpcDash = { versao: '', lista: [] };
    try {
        const d = JSON.parse(await GM.getValue(SIS_RPC_DASH_KEY, '{}'));
        const loc = JSON.parse(await GM.getValue('sisLocal', '{}'));
        if (d && Array.isArray(d.lista) && loc.aceitarRpcs !== false && (Date.now() - (+d.em || 0)) < SIS_RPC_DASH_VALIDADE) sisRpcDash = { versao: String(d.versao || ''), origem: ['usuario', 'instancia'].includes(d.origem) ? d.origem : 'global', lista: d.lista.filter(x => sisRpcHabilitado(x) && sisRpcUrlOk(x.url)).slice(0, 12) };
    } catch (e) { }

    function sisRpcHost(u) { try { return new URL(u).host; } catch (e) { return '?'; } }
    // tira chaves de qualquer texto (mensagens de erro do web3.js trazem a URL inteira)
    function sisRpcMascarar(t) {
        return String(t)
            .replace(/([?&#](?:x-api-key|api[-_]?key|apikey|key|token|auth|authorization)=)[^&\s"']+/gi, '$1***')
            .replace(/(https:\/\/[^\/\s"']+\/)[A-Za-z0-9_-]{20,}/g, '$1***');
    }
    // cabeçalhos no fragmento da URL: https://host/caminho#x-api-key=CHAVE&outro-cabecalho=valor
    // (o fragmento nunca vai para o servidor; o Sis-Bot tira dele os cabeçalhos HTTP — ex.: Tatum usa x-api-key)
    function sisRpcSeparar(u) {
        const i = u.indexOf('#'); if (i < 0) return { url: u, cab: null };
        const cab = {};
        for (const par of u.slice(i + 1).split('&')) {
            const j = par.indexOf('='); if (j <= 0) continue;
            const k = decodeURIComponent(par.slice(0, j)).trim(); const v = decodeURIComponent(par.slice(j + 1)).trim();
            if (/^[A-Za-z0-9-]{1,40}$/.test(k) && !/^(content-type|host|origin|cookie)$/i.test(k) && v) cab[k] = v;
        }
        return { url: u.slice(0, i), cab: Object.keys(cab).length ? cab : null };
    }
    function sisRpcNovo(def, nivel, origem) {
        const rps = +def.rps > 0 ? Math.min(+def.rps, 1000) : 40;
        return {
            url: def.url, ...(() => { const x = sisRpcSeparar(def.url); return { urlReal: x.url, cab: x.cab }; })(), host: sisRpcHost(def.url), nome: String(def.nome || '').slice(0, 60) || sisRpcHost(def.url), nivel, origem, proprio: origem === 'navegador',
            rps, rpsMetodo: +def.rpsMetodo > 0 ? +def.rpsMetodo : rps, sem: new Set(def.sem || []), ordem: 0,
            tat: 0, metodos: new Map(),
            ate: 0, falhas: 0, ok: 0, erros: 0, ms: 0, emUso: 0, ult: '', ultEm: 0, motivo: '',
        };
    }
    const sisRpcPool = [];
    // (re)monta o pool sem perder o estado (castigo, contadores) dos RPCs que continuam
    function sisRpcMontar() {
        const antes = new Map(sisRpcPool.map(e => [e.nivel + '|' + e.url, e]));
        const base = sisRpcDash.lista.length ? sisRpcDash.lista.map(d => [d, sisRpcDash.origem === 'usuario' ? 'seus (dashboard)' : sisRpcDash.origem === 'instancia' ? 'deste navegador (dashboard)' : 'dashboard']) : SIS_RPC_PADRAO.map(d => [d, 'padrão']);
        const novos = [].concat(
            sisRpcProprios.map(u => [{ url: u, nome: 'seu RPC', rps: 40 }, 'navegador', 0]),
            base.map(([d, o]) => [d, o, 1]),
        ).map(([d, o, nv]) => {
            const velho = antes.get(nv + '|' + d.url);
            if (velho) { velho.nome = String(d.nome || velho.nome).slice(0, 60); velho.origem = o; if (+d.rps > 0) velho.rps = velho.rpsMetodo = Math.min(+d.rps, 1000); return velho; }
            return sisRpcNovo(d, nv, o);
        });
        const vistos = new Set();
        sisRpcPool.splice(0, sisRpcPool.length, ...novos.filter(e => !vistos.has(e.url) && vistos.add(e.url)));
        sisRpcPool.forEach((e, i) => { e.ordem = i; });
        readRPCs = writeRPCs = sisRpcPool.map(e => e.url);
    }
    // nova lista vinda do dashboard (sync). Devolve true se mudou.
    async function sisRpcReceberDash(r) {
        if (!r || typeof r !== 'object' || !Array.isArray(r.lista)) return false;
        let aceitar = true; try { aceitar = sisLocal.aceitarRpcs !== false; } catch (e) { }
        const versao = String(r.versao || '');
        if (!aceitar) { if (sisRpcDash.lista.length) { sisRpcDash = { versao: '', lista: [] }; sisRpcMontar(); } return false; }
        if (versao === sisRpcDash.versao) {
            // [1.5.1] mesma versão: renova a data de confirmação (no máximo 1×/min) para a lista guardada continuar valendo na partida
            if (Date.now() - (sisRpcDash.em || 0) > 60000) { sisRpcDash.em = Date.now(); try { await GM.setValue(SIS_RPC_DASH_KEY, JSON.stringify(sisRpcDash)); } catch (e) { } }
            return false;
        }
        const lista = r.lista.filter(x => sisRpcHabilitado(x) && sisRpcUrlOk(x.url)).slice(0, 12).map(x => ({ url: x.url, nome: String(x.nome || '').slice(0, 60), rps: +x.rps || 0 }));
        sisRpcDash = { versao, origem: ['usuario', 'instancia'].includes(r.origem) ? r.origem : 'global', lista, em: Date.now() };
        try { await GM.setValue(SIS_RPC_DASH_KEY, JSON.stringify(sisRpcDash)); } catch (e) { }
        sisRpcMontar();
        sisRpcLog(1, `[RPC] lista do dashboard aplicada (v${versao.slice(0, 8)}): ${sisRpcPool.map(e => e.host).join(', ')}`);
        try { sisRpcAtualizarChip(); } catch (e) { }
        return true;
    }
    // mantém os nomes antigos do SLYA (o resto do código e os logs ainda citam)
    let customReadRPCs = sisRpcProprios.slice();
    let customWriteRPCs = sisRpcProprios.slice();
    let saRPCs = SIS_RPC_PADRAO.map(e => e.url);
    let readRPCs = [], writeRPCs = [];
    sisRpcMontar();

    function sisRpcLog(nivel, ...a) { try { cLog(nivel, ...a.map(sisRpcMascarar)); } catch (e) { } }

    // agenda tipo GCRA (balde furado): cada RPC tem um "próximo horário livre" no total e por método.
    // A requisição reserva o primeiro horário livre e dorme até ele: fila justa, sem 429 por excesso nosso.
    const SIS_RPC_RAJADA_MS = 250; // rajada curta: a Helius conta por segundo
    function sisRpcHorario(ep, metodo, agora) {
        let m = ep.metodos.get(metodo); if (!m) { m = { tat: 0 }; ep.metodos.set(metodo, m); }
        return Math.max(agora, ep.tat - SIS_RPC_RAJADA_MS, m.tat - SIS_RPC_RAJADA_MS);
    }
    // [1.4.0] navegador de reserva usa só uma fração do req/s de cada RPC (o dashboard manda rpc_fator); mínimo 0,5 req/s
    let sisRpcFatorAtual = 1;
    function sisRpcFator(f) { f = +f; sisRpcFatorAtual = f > 0 && f <= 1 ? f : 1; }
    function sisRpcReservar(ep, metodo, t) {
        const m = ep.metodos.get(metodo) || { tat: 0 }; ep.metodos.set(metodo, m);
        const rps = Math.max(0.5, ep.rps * sisRpcFatorAtual), rpsM = Math.max(0.5, ep.rpsMetodo * sisRpcFatorAtual);
        ep.tat = Math.max(ep.tat, t) + 1000 / rps;
        m.tat = Math.max(m.tat, t) + 1000 / rpsM;
    }

    // escolhe o RPC e dorme até o horário reservado nele
    async function sisRpcEscolher(metodo, tentados) {
        const agora = Date.now();
        let cand = sisRpcPool.filter(e => !tentados.has(e) && e.ate <= agora && !e.sem.has(metodo));
        // 1.3.4: todos de castigo → tenta assim mesmo o que volta primeiro (nunca fica sem RPC)
        if (!cand.length) { const r = sisRpcPool.filter(e => !tentados.has(e) && !e.sem.has(metodo)).sort((a, b) => a.ate - b.ate); if (!r.length) return null; cand = [r[0]]; }
        // prioridade = ordem da lista; passa para o próximo quando o horário livre do anterior está > 250 ms à frente
        let melhor = null, tMelhor = Infinity;
        for (const e of cand) {
            const t = sisRpcHorario(e, metodo, agora) + e.emUso * 3 + e.ordem * 250;
            if (t < tMelhor) { tMelhor = t; melhor = e; }
        }
        const t = sisRpcHorario(melhor, metodo, agora);
        sisRpcReservar(melhor, metodo, t);
        if (t > agora) await new Promise(r => setTimeout(r, t - agora));
        // pode ter entrado de castigo enquanto esperava (outra requisição tomou 429): escolhe de novo
        if (melhor.ate > Date.now() && sisRpcPool.some(e => e !== melhor && !tentados.has(e) && e.ate <= Date.now() && !e.sem.has(metodo))) return await sisRpcEscolher(metodo, tentados);
        return melhor;
    }

    function sisRpcCastigo(ep, ms, motivo) {
        ep.falhas++; ep.erros++; ep.ult = sisRpcMascarar(motivo).slice(0, 160); ep.ultEm = Date.now(); ep.motivo = motivo.length > 60 ? motivo.slice(0, 60) : motivo;
        const t = ms || Math.min(30000, 1000 * Math.pow(2, Math.min(ep.falhas - 1, 5))); // 1.3.4: no máximo 30 s fora
        ep.ate = Date.now() + t;
        sisRpcLog(2, `[RPC] ${ep.host} fora por ${Math.round(t / 1000)} s: ${motivo}`);
        try { sisRpcAtualizarChip(); } catch (e) { }
    }
    function sisRpcSucesso(ep, ms) {
        ep.falhas = 0; ep.ok++; ep.ms = ep.ms ? Math.round(ep.ms * 0.8 + ms * 0.2) : ms; ep.motivo = '';
    }
    // [1.5.7] RPC atrasado: as leituras de conta trazem context.slot. Resposta muito atrás do maior slot já visto é um retrato
    // antigo da blockchain e não vale (caso [EC] Kram Log 18/11: um nó devolveu o tanque de ~2 h antes, o reabastecer achou que
    // tinha combustível e a frota parou sem combustível no meio do caminho). Castigo de 60 s nesse RPC e a requisição vai para outro.
    const SIS_RPC_ATRASO_MAX = 75; // slots (~30 s)
    let sisRpcSlotMax = 0;
    function sisRpcSlotResposta(j) {
        const um = (x) => (x && x.result && x.result.context && Number.isFinite(+x.result.context.slot) ? +x.result.context.slot : null);
        if (Array.isArray(j)) { const v = j.map(um).filter((s) => s !== null); return v.length ? Math.min(...v) : null; }
        return um(j);
    }

    // uma requisição num RPC. {ok:true, resp} = devolve ao web3.js; {ok:false, resp?} = tenta o próximo
    async function sisRpcUma(ep, init, metodo) {
        const ctl = new AbortController();
        const lento = /^(getProgramAccounts|getMultipleAccounts|getSignaturesForAddress|getParsedProgramAccounts)$/.test(metodo);
        const tmr = setTimeout(() => ctl.abort(), lento ? 45000 : SIS_RPC_TIMEOUT);
        const t0 = Date.now(); ep.emUso++;
        try {
            const opc = Object.assign({}, init, { signal: ctl.signal });
            if (ep.cab) opc.headers = Object.assign({}, init.headers || {}, ep.cab);
            const r = await fetch(ep.urlReal || ep.url, opc);
            const texto = await r.text();
            const ms = Date.now() - t0;
            const resp = () => new Response(texto, { status: r.status, statusText: r.statusText, headers: { 'content-type': 'application/json' } });
            const baixo = texto.slice(0, 400).toLowerCase();
            if (r.status === 429) {
                const ra = parseInt(r.headers.get('retry-after')); // segundos
                // limite por segundo (ex.: Tatum grátis = 3/s, "exceeded your limit of 3 requests per second") é castigo curto; cota do plano esgotada, 10 min
                const porSeg = /per second|\/second|requests per|request limit reached|rate_limit|too many requests|-32007/.test(baixo);
                const cota = !porSeg && /free tier|limit has been reached|upgrade|quota|exceeded your|credits/.test(baixo);
                sisRpcCastigo(ep, cota ? SIS_RPC_COTA_MS : (ra > 0 ? ra * 1000 : porSeg ? 1000 : 0), cota ? 'cota do plano esgotada (429)' : 'muitas requisições (429)');
                return { ok: false, resp: resp() };
            }
            // 413 / -32615 (ex.: QuickNode grátis "getMultipleAccounts is limited to a 5 range"): este RPC não serve para esse pedido; vai para o próximo sem castigo
            if (r.status === 413 || /-32615|limited to an? \d+ range/.test(baixo)) { ep.sem.add(metodo); ep.ult = `limite de tamanho em ${metodo}`; ep.ultEm = Date.now(); return { ok: false, resp: resp() }; }
            if (r.status === 401 || r.status === 403 || r.status === 400 || r.status === 404 || r.status === 405) {
                // método bloqueado neste RPC (ex.: "Indexed requests require a personal token") => não manda mais esse método para ele
                if (/method|indexed|not available|not allowed|not supported|require|personal token|free plan/.test(baixo) && metodo !== '?') {
                    ep.sem.add(metodo); ep.ult = `não aceita ${metodo}`; ep.ultEm = Date.now();
                    sisRpcLog(2, `[RPC] ${ep.host} não aceita ${metodo}; vai para outro RPC`);
                    return { ok: false, resp: resp() };
                }
                sisRpcCastigo(ep, SIS_RPC_COTA_MS, `recusado (${r.status})`);
                return { ok: false, resp: resp() };
            }
            if (r.status >= 500) { sisRpcCastigo(ep, 0, `erro do servidor (${r.status})`); return { ok: false, resp: resp() }; }
            if (r.ok) {
                // erros JSON-RPC que significam "este RPC não serve agora", não "a requisição está errada"
                let j = null; try { j = JSON.parse(texto); } catch (e) { sisRpcCastigo(ep, 0, 'resposta inválida'); return { ok: false }; }
                const err = Array.isArray(j) ? (j.find(x => x && x.error) || {}).error : j && j.error;
                if (err) {
                    const msg = String(err.message || '').toLowerCase(); const code = +err.code;
                    if (code === -32601 || /method .*not (available|found|supported)|is available (on|for)|paid plan|not allowed/.test(msg)) {
                        ep.sem.add(metodo); ep.ult = `não aceita ${metodo}`; ep.ultEm = Date.now();
                        return { ok: false, resp: resp() };
                    }
                    // [1.4.1] Helius: "account index service overloaded, please try again" (getTokenAccountsByOwner) — antes voltava como erro ao
                    // web3.js e o SLYA repetia no mesmo RPC para sempre (a ordem da frota travava e o Sis-Bot ficava em "Wait...")
                    if (code === -32005 || code === -32004 || code === -32014 || code === -32016 || /node is behind|is unhealthy|too many requests|rate limit|overloaded|please try again|temporarily unavailable/.test(msg)) {
                        sisRpcCastigo(ep, /too many|rate limit/.test(msg) ? 0 : 5000, `RPC indisponível (${code}: ${msg.slice(0, 60)})`);
                        return { ok: false, resp: resp() };
                    }
                }
                const slot = sisRpcSlotResposta(j);
                if (slot !== null) {
                    if (sisRpcSlotMax - slot > SIS_RPC_ATRASO_MAX) {
                        ep.atrasos = (ep.atrasos || 0) + 1; ep.atrasadoAte = Date.now() + 60000;
                        sisRpcCastigo(ep, 60000, `RPC atrasado (${sisRpcSlotMax - slot} slots atrás)`);
                        return { ok: false, resp: resp() };
                    }
                    if (slot > sisRpcSlotMax) sisRpcSlotMax = slot;
                }
                sisRpcSucesso(ep, ms);
                return { ok: true, resp: resp() };
            }
            sisRpcCastigo(ep, 0, `HTTP ${r.status}`);
            return { ok: false, resp: resp() };
        } catch (e) {
            sisRpcCastigo(ep, 0, ctl.signal.aborted ? 'sem resposta (timeout)' : `falha de rede (${e && e.message || e})`);
            return { ok: false };
        } finally { clearTimeout(tmr); ep.emUso--; }
    }

    function sisRpcErroFinal(corpo, msg) {
        let id = 1; try { const j = JSON.parse(corpo); id = Array.isArray(j) ? (j[0] && j[0].id) : j.id; } catch (e) { }
        return new Response(JSON.stringify({ jsonrpc: '2.0', id, error: { code: -32000, message: msg } }), { status: 503, statusText: 'Service Unavailable', headers: { 'content-type': 'application/json' } });
    }

    // sendTransaction: manda para todos os RPCs disponíveis ao mesmo tempo e devolve a primeira resposta boa
    async function sisRpcBroadcast(init, corpo) {
        const agora = Date.now();
        // 1.3.4: a transação vai para TODOS os RPCs (inclusive os de castigo): mandar não custa nada e aumenta a chance de entrar
        let alvos = sisRpcPool.filter(e => !e.sem.has('sendTransaction'));
        alvos.forEach(e => sisRpcReservar(e, 'sendTransaction', Date.now()));
        return await new Promise(resolve => {
            let pendentes = alvos.length, primeiroErro = null, resolvido = false;
            if (!pendentes) { resolve(sisRpcErroFinal(corpo, 'Sis-Bot: nenhum RPC aceita sendTransaction')); return; }
            for (const e of alvos) {
                sisRpcUma(e, init, 'sendTransaction').then(r => {
                    if (r.ok && !resolvido) {
                        // resposta com "error" de verdade (ex.: tx inválida) também é resposta final
                        resolvido = true; resolve(r.resp); return;
                    }
                    if (!primeiroErro && r.resp) primeiroErro = r.resp;
                    if (--pendentes === 0 && !resolvido) { resolvido = true; resolve(primeiroErro || sisRpcErroFinal(corpo, 'Sis-Bot: nenhum RPC respondeu ao envio')); }
                });
            }
        });
    }

    // fetch que o web3.js usa (Connection({fetch})). Ignora a URL pedida e usa o pool.
    async function sisRpcFetch(_url, init) {
        init = init || {};
        const corpo = typeof init.body === 'string' ? init.body : '';
        let metodo = '?';
        try { const j = JSON.parse(corpo); metodo = String(Array.isArray(j) ? (j[0] && j[0].method) : j.method); } catch (e) { }
        if (metodo === 'sendTransaction') return await sisRpcBroadcast(init, corpo);
        const tentados = new Set(); let ultimo = null; let espera = 400;
        for (let i = 0; i < SIS_RPC_TENTATIVAS; i++) {
            const ep = await sisRpcEscolher(metodo, tentados);
            if (!ep) {
                // todos já tentados ou de castigo: espera o primeiro voltar (até 15 s) e recomeça
                const prox = Math.min(...sisRpcPool.filter(e => !e.sem.has(metodo)).map(e => e.ate), Date.now() + 15000);
                await new Promise(r => setTimeout(r, Math.max(espera, Math.min(prox - Date.now(), 15000))));
                espera = Math.min(espera * 2, 15000); tentados.clear(); continue;
            }
            tentados.add(ep);
            const r = await sisRpcUma(ep, init, metodo);
            if (r.ok) return r.resp;
            if (r.resp) ultimo = r.resp;
        }
        try { sisRpcAtualizarChip(); } catch (e) { }
        return sisRpcErroFinal(corpo, `Sis-Bot: nenhum RPC respondeu a ${metodo}` + (ultimo ? ` (último HTTP ${ultimo.status})` : ''));
    }

    // [1.3.4] sonda: a cada 10 s, os RPCs de castigo recebem um getSlot leve; respondeu = volta na hora para a fila
    // (antes ficavam fora até o castigo acabar, e com vários fora ao mesmo tempo o bot ficava sem RPC).
    let sisRpcSondando = false;
    async function sisRpcSondar() {
        if (sisRpcSondando) return; sisRpcSondando = true;
        try {
            const agora = Date.now();
            await Promise.all(sisRpcPool.filter(e => e.ate > agora && !(e.atrasadoAte > agora)).map(async (e) => {
                const ctl = new AbortController(); const tmr = setTimeout(() => ctl.abort(), 6000);
                try {
                    const opc = { method: 'POST', headers: Object.assign({ 'Content-Type': 'application/json' }, e.cab || {}), body: '{"jsonrpc":"2.0","id":1,"method":"getSlot"}', signal: ctl.signal };
                    const r = await fetch(e.urlReal || e.url, opc); const j = await r.json().catch(() => null);
                    if (r.ok && j && typeof j.result === 'number') { e.ate = 0; e.falhas = 0; e.motivo = ''; sisRpcLog(2, `[RPC] ${e.host} voltou (sonda ok)`); }
                } catch (x) { } finally { clearTimeout(tmr); }
            }));
            try { sisRpcAtualizarChip(); } catch (x) { }
        } finally { sisRpcSondando = false; }
    }
    setInterval(() => { sisRpcSondar(); }, 10000);

    // [1.4.0] sondas de latência (todos os navegadores, primário e reservas): getSlot em cada RPC a cada 30 s; guarda 10 min.
    // É a mesma medida em todos os navegadores do perfil: o dashboard compara e escolhe o primário (melhor par navegador × RPC)
    // e a ordem da lista geral. Custa 1 requisição leve por RPC a cada 30 s.
    const sisRpcSondas = new Map(); // host -> [{t, ms|null}]
    let sisRpcMedindo = false;
    async function sisRpcMedir() {
        if (sisRpcMedindo) return; sisRpcMedindo = true;
        try {
            await Promise.all(sisRpcPool.map(async (e) => {
                if (e.ate > Date.now() || e.sem.has('getSlot')) return; // fora: a sonda de volta cuida; a nota já penaliza
                const t = sisRpcHorario(e, 'getSlot', Date.now()); sisRpcReservar(e, 'getSlot', t);
                if (t > Date.now()) await new Promise(r => setTimeout(r, t - Date.now()));
                const ctl = new AbortController(); const tmr = setTimeout(() => ctl.abort(), 8000); const t0 = performance.now(); let ms = null;
                try {
                    const r = await fetch(e.urlReal || e.url, { method: 'POST', headers: Object.assign({ 'Content-Type': 'application/json' }, e.cab || {}), body: '{"jsonrpc":"2.0","id":1,"method":"getSlot"}', signal: ctl.signal });
                    const j = await r.json().catch(() => null);
                    if (r.ok && j && typeof j.result === 'number') ms = Math.round(performance.now() - t0);
                    // [1.5.7] a sonda de todos os RPCs também diz qual é o slot atual (é o que pega um RPC atrasado que está na frente da fila)
                    if (r.ok && j && typeof j.result === 'number' && j.result > sisRpcSlotMax) sisRpcSlotMax = j.result;
                } catch (x) { } finally { clearTimeout(tmr); }
                const l = sisRpcSondas.get(e.host) || []; l.push({ t: Date.now(), ms });
                while (l.length && l[0].t < Date.now() - 600000) l.shift();
                sisRpcSondas.set(e.host, l);
            }));
        } finally { sisRpcMedindo = false; }
    }
    setTimeout(() => { sisRpcMedir(); }, 4000);
    setInterval(() => { sisRpcMedir(); }, 30000);
    // p50/p90 das sondas (10 min) e falhas, por RPC do pool — vai para o dashboard em cada sync (só host, nunca a URL)
    function sisRpcMetricas() {
        const agora = Date.now();
        return sisRpcPool.map(e => {
            const l = (sisRpcSondas.get(e.host) || []).filter(x => x.t > agora - 600000);
            const ok = l.filter(x => x.ms !== null).map(x => x.ms).sort((a, b) => a - b);
            const q = (p) => ok.length ? ok[Math.min(ok.length - 1, Math.floor(ok.length * p))] : null;
            return { host: e.host, nome: e.nome, p50: q(0.5), p90: q(0.9), n: l.length, falhas: l.length - ok.length, ms: e.ms || null, estado: e.ate > agora ? 'fora' : 'ok', atrasos: e.atrasos || 0 };
        });
    }

    function sisRpcNovaConexao() {
        // a URL é só rótulo (o fetch acima escolhe o RPC); disableRetryOnRateLimit: o pool já troca de RPC no 429
        return new solanaWeb3.Connection(sisRpcPool[0].url, { commitment: 'confirmed', fetch: sisRpcFetch, disableRetryOnRateLimit: true });
    }

    // resumo (painel, status para o dashboard): só host, nunca a URL
    function sisRpcResumo() {
        const agora = Date.now();
        return sisRpcPool.map(e => ({
            host: e.host, nome: e.nome, proprio: e.proprio, origem: e.origem,
            estado: e.ate > agora ? 'fora' : 'ok', volta_em_s: e.ate > agora ? Math.round((e.ate - agora) / 1000) : 0,
            ok: e.ok, erros: e.erros, ms: e.ms, sem: [...e.sem], ult: e.ult, ult_ha_s: e.ultEm ? Math.round((agora - e.ultEm) / 1000) : null,
        }));
    }
    function sisRpcAtualizarChip() {
        const el = document.getElementById('sisRpcChip'); if (!el) return;
        const agora = Date.now();
        const ok = sisRpcPool.filter(e => e.ate <= agora);
        const usados = ok.slice().sort((a, b) => a.ordem - b.ordem);
        const txt = ok.length === 0 ? 'RPC: todos fora' : `RPC: ${ok.length}/${sisRpcPool.length}` + (usados[0] ? ` · ${usados[0].host.replace(/^(api\.|rpc\.|public\.rpc\.)/, '')}` : '');
        el.textContent = txt;
        el.style.color = ok.length === 0 ? 'tomato' : (ok.length < sisRpcPool.length ? 'orange' : 'inherit');
        el.title = sisRpcPool.map(e => `${e.nome} (${e.host}, ${e.origem}): ${e.ate > agora ? 'fora por ' + Math.round((e.ate - agora) / 1000) + ' s — ' + e.motivo : 'ok'}${e.ms ? ' · ' + e.ms + ' ms' : ''} · ${e.ok} ok / ${e.erros} erros${e.sem.size ? ' · não aceita ' + [...e.sem].join(', ') : ''}`).join('\n');
    }
    setInterval(() => { try { sisRpcAtualizarChip(); } catch (e) { } }, 5000);

    // taxa de prioridade da rede para as contas graváveis da tx (microLamports/CU, percentil 75 dos últimos ~150 slots)
    const sisTaxaRedeCache = new Map();
    async function sisTaxaRede(conn, chaves) {
        try {
            const k = chaves.slice(0, 6).join(',');
            const c = sisTaxaRedeCache.get(k); if (c && Date.now() - c.em < 20000) return c.v;
            const r = await conn.getRecentPrioritizationFees({ lockedWritableAccounts: chaves.slice(0, 64).map(x => new solanaWeb3.PublicKey(x)) });
            const v = (r || []).map(x => x.prioritizationFee).filter(x => x > 0).sort((a, b) => a - b);
            const p75 = v.length ? v[Math.min(v.length - 1, Math.floor(v.length * 0.75))] : 0;
            sisTaxaRedeCache.set(k, { v: p75, em: Date.now() });
            if (sisTaxaRedeCache.size > 200) sisTaxaRedeCache.clear();
            return p75;
        } catch (e) { return 0; }
    }
    // simula a tx (sem assinatura) e devolve o limite de CU a pedir (consumo × 1,15 + folga) ou 0 se não deu
    async function sisLimiteCU(conn, instructions, blockhash) {
        try {
            const msg = new solanaWeb3.TransactionMessage({ payerKey: userPublicKey, recentBlockhash: blockhash, instructions }).compileToV0Message(addressLookupTables);
            const sim = await conn.simulateTransaction(new solanaWeb3.VersionedTransaction(msg), { sigVerify: false, replaceRecentBlockhash: true, commitment: 'confirmed' });
            const v = sim && sim.value; if (!v) return 0;
            if (v.err) { sisRpcLog(3, '[CU] simulação com erro (envia sem limite de CU):', JSON.stringify(v.err)); return 0; }
            const u = +v.unitsConsumed; if (!(u > 0)) return 0;
            // [1.3.7] scan de SDU: o achado é sorteado na hora, então a simulação pode dar "sem achado" (~136 mil CU) e o scan real
            // achar (~160 mil: ConsumeCargo + transferência do SDU + LegitimizeCargo). Com × 1,15 + 5000 a folga ficava ~0 e o achado
            // falhava por CU esgotado (08/11 06:41, Kram Scan 2: 792 SDU perdidos). No scan pede sempre o consumo + 40 mil.
            const scan = instructions.some((i) => i && i.programId && i.programId.toString() === 'SAGE2HAwep459SNq61LHvjxPk4pLPEJLoMETef7f7EE'
                && i.data && i.data.length >= 8 && Array.from(i.data.slice(0, 8), (b) => (b & 255).toString(16).padStart(2, '0')).join('') === '5466ea017e88ba93');
            const lim = Math.ceil(u * 1.15) + 5000;
            // [1.5.9] piso no scan: em 25/11 17:46 (Kram Scan 1, logo depois de trocar o navegador primário e recarregar a página) a simulação
            // devolveu só 29.659 CU (o normal é ~136 mil; com achado ~160 mil) → limite de 69.659 → "CU esgotado". Um scan nunca gasta menos
            // que isso: simulação abaixo de 100 mil CU é tratada como leitura incompleta e vale o piso de 200 mil (o maior consumo visto: ~160 mil).
            if (scan && u < 100000) sisRpcLog(2, '[CU] scan: a simulação deu só ' + Math.round(u) + ' CU (esperado ~136 mil); usando o piso de 200 mil');
            return Math.min(1400000, scan ? Math.max(lim, Math.ceil(u) + 40000, 200000) : lim);
        } catch (e) { return 0; }
    }
    // [1.3.2] limites de taxa do usuário (Sis-Bot › Conexão e alertas / Administração › RPCs › Por usuário):
    // lamports de PRIORIDADE por transação (além dos 5000 de base). null = sem limite. O teto local (tetoTaxa) ainda vale por cima.
    let sisTaxaLimites = { min: null, max: null };
    try { const t = JSON.parse(await GM.getValue('sisTaxaLimites', '{}')); sisTaxaLimites = { min: Number.isFinite(t.min) ? t.min : null, max: Number.isFinite(t.max) ? t.max : null }; } catch (e) { }
    async function sisReceberTaxaLimites(t) {
        if (!t || typeof t !== 'object') return;
        const num = (v) => (v === null || v === undefined || v === '' || !Number.isFinite(+v) ? null : Math.max(0, Math.min(10000000, Math.round(+v))));
        const novo = { min: num(t.min), max: num(t.max) };
        if (novo.min !== null && novo.max !== null && novo.min > novo.max) novo.min = novo.max;
        if (novo.min === sisTaxaLimites.min && novo.max === sisTaxaLimites.max) return;
        sisTaxaLimites = novo;
        try { await GM.setValue('sisTaxaLimites', JSON.stringify(novo)); } catch (e) { }
        sisRpcLog(1, `[Taxa] limites do dashboard: mínimo ${novo.min ?? '—'} / máximo ${novo.max ?? '—'} lamports de prioridade por transação`);
    }
    // µlamports/CU → aplica mínimo/máximo em lamports por tx (com o CU pedido) e depois o teto local
    // [1.3.4] reenvio (a tx expirou sem entrar): sobe a taxa de prioridade a cada tentativa — 2.000, 4.000, 8.000… lamports
    // (até 64.000), sempre dentro do máximo do usuário e do teto local
    const SIS_REFORCO_BASE = 2000;
    function sisAplicarLimitesTaxa(micro, cu, reenvios = 0) {
        const c = cu > 0 ? cu : 200000;
        if (reenvios > 0) micro = Math.max(micro, Math.ceil(SIS_REFORCO_BASE * Math.pow(2, Math.min(reenvios, 6) - 1) * 1e6 / c));
        if (sisTaxaLimites.min !== null && micro * c / 1e6 < sisTaxaLimites.min) micro = Math.ceil(sisTaxaLimites.min * 1e6 / c);
        if (sisTaxaLimites.max !== null && micro * c / 1e6 > sisTaxaLimites.max) micro = Math.floor(sisTaxaLimites.max * 1e6 / c);
        // teto local do navegador ("Taxa de prioridade máxima (lamports)") = lamports de prioridade por tx; vale acima do dashboard
        try { const teto = +sisLocal.tetoTaxa; if (teto > 0 && micro * c / 1e6 > teto) micro = Math.floor(teto * 1e6 / c); } catch (e) { }
        return Math.max(0, micro);
    }

    // [1.3.1] chip "Fee": a taxa do SLYA (lamports por 200 mil CU, sobe sozinha só quando as tx demoram) + o que a última tx pagou de fato
    let sisTaxaUltima = null;
    function sisAtualizarChipTaxa() {
        const el = document.getElementById('assist-modal-fee'); if (!el) return;
        let fee = '?'; try { fee = currentFee; } catch (e) { }
        const u = sisTaxaUltima;
        if (!u) { el.textContent = 'Fee:' + fee; el.title = ''; return; }
        const cu = u.cuEfetivo || (u.cu > 0 ? u.cu : 200000);
        const lamp = Math.ceil(u.usada * cu / 1e6);
        el.textContent = `Fee:${fee} · +${lamp} lamp`;
        let auto = false, tmin = 10; try { auto = !!globalSettings.automaticFee; tmin = globalSettings.automaticFeeTimeMin; } catch (e) { }
        el.title = `Taxa do SLYA: ${fee} (lamports por 200 mil CU; ${auto ? `automática — só sobe quando as transações levam mais de ${tmin} s para confirmar` : 'fixa'}).\n`
            + `Última transação: ${u.usada} µlamports/CU × ${u.cu > 0 ? u.cu + ' CU (simulado)' : '200 mil CU (sem simulação)'} = ${lamp} lamports de prioridade + 5000 de base por assinatura.\n`
            + (sisTaxaLimites.min !== null || sisTaxaLimites.max !== null ? `Seus limites (dashboard): mínimo ${sisTaxaLimites.min ?? '—'} / máximo ${sisTaxaLimites.max ?? '—'} lamports por tx.\n` : '')
            + (u.rede ? `Taxa da rede nas mesmas contas (p75): ${u.rede} µlamports/CU${u.usada > u.base ? ' — usada como piso' : ''}.` : 'Taxa da rede nas mesmas contas: 0 (rede sem disputa).');
    }
    // ===================================================================================== fim do pool de RPCs

    let customKeypair = null;

    //Program public keys
    const sageProgramPK = new solanaWeb3.PublicKey('SAGE2HAwep459SNq61LHvjxPk4pLPEJLoMETef7f7EE');
    const profileProgramPK = new solanaWeb3.PublicKey('pprofELXjL5Kck7Jn5hCpwAL82DpTkSYBENzahVtbc9');
    const cargoProgramPK = new solanaWeb3.PublicKey('Cargo2VNTPPTi9c1vq1Jw5d3BWUNr18MjRtSupAghKEk');
    const profileFactionProgramPK = new solanaWeb3.PublicKey('pFACSRuobDmvfMKq1bAzwj27t6d2GJhSCHb1VcfnRmq');
    const craftingProgramPK = new solanaWeb3.PublicKey('CRAFT2RPXPJWCEix4WpJST3E7NLf79GTqZUL75wngXo5');
    const pointsProgramId = new solanaWeb3.PublicKey('Point2iBvz7j5TMVef8nEgpmz4pDr7tU7v3RjAfkQbM');
    const pointsStoreProgramId = new solanaWeb3.PublicKey('PsToRxhEPScGt1Bxpm7zNDRzaMk31t8Aox7fyewoVse');
    const dataRunningXpCategory = new solanaWeb3.PublicKey('DataJpxFgHhzwu4zYJeHCnAv21YqWtanEBphNxXBHdEY');
    const councilRankXpCategory = new solanaWeb3.PublicKey('XPneyd1Wvoay3aAa24QiKyPjs8SUbZnGg5xvpKvTgN9'); //??
    const pilotingXpCategory = new solanaWeb3.PublicKey('PiLotBQoUBUvKxMrrQbuR3qDhqgwLJctWsXj3uR7fGs');
    const miningXpCategory = new solanaWeb3.PublicKey('MineMBxARiRdMh7s1wdStSK4Ns3YfnLjBfvF5ZCnzuw');
    const craftingXpCategory = new solanaWeb3.PublicKey('CraftndAV62acibnaW7TiwEYwu8MmJZBdyrfyN54nre7');
    const LPCategory = new solanaWeb3.PublicKey('LPkmmDQG8iBDAfKkWN6QadeoiLSvD1p3fGgq8m8QdMu');
    const addressLookupTableAddresses = [new solanaWeb3.PublicKey('AyC4m8fYEgR9mYcf6zzajevPjF8QhptY9Nae5LX6xgiu'), new solanaWeb3.PublicKey('F4jZvnU9fdi2mGp13TyewdAj96cKGUcwBBSMTsL1nRoC')];

	//Token addresses
	const programAddy = 'ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL';
	const tokenProgAddy = 'TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA';

	//Commonly used public keys
	const programPK = new solanaWeb3.PublicKey(programAddy);
	const tokenProgramPK = new solanaWeb3.PublicKey(tokenProgAddy);

	let enableAssistant = false;
	let initComplete = false;
	let solanaErrorCount = 0;

	let globalSettings;
	const settingsGmKey = 'globalSettings';
	// [Sis-Ágora] declarado antes de loadGlobalSettings() (const não é içado)
	const CAMPOS_SECRETOS = ['mySecretKey'];

	// =====================================================================================================
	// [Sis-Ágora] Sis-Bot — comunicação com a VPS (dashboard do Sis-Ágora)
	// A VPS manda só DADOS: uma configuração validada (aplicada apenas depois do aceite local) e comandos de uma
	// lista fixa. Nada vindo da VPS é executado como código, vira HTML ou assina transação diretamente.
	// O navegador envia: estado das frotas/crafts, eventos, produção, estatísticas e saldo da hot wallet.
	// =====================================================================================================
	const SIS_URL = 'https://clashofiris.com/~sisagora/dashboard/bot.php';
	const SIS_LOCAL_KEY = 'sisLocal';
	const SIS_PADRAO_LOCAL = { token: '', tetoTaxa: 5000, tetoMultCraft: 300, tetoTxHora: 900, comandosRemotos: true, modoLocal: false, cfgAceita: null, recusada: 0, querRodando: false, retomarAuto: true, aceitarRpcs: true, espelharCfg: true };
	const SIS_MINERAVEIS = ['Arco', 'Biomass', 'Carbon', 'Copper Ore', 'Diamond', 'Hydrogen', 'Iron Ore', 'Lumanite', 'Nitrogen', 'Rochinol', 'Silica', 'Titanium Ore'];
	const SIS_TAREFAS = ['', 'Mine', 'Transport', 'Scan', 'Supply', 'Garimpo', 'Prospectar'];
	// tarefas decididas pelo Sis-Bot: o motor local do SLYA não opera essas frotas. 1.2.0: Prospectar (SDU); "Scan" segue no motor do SLYA
	const SIS_CEREBRO = ['Supply', 'Garimpo', 'Prospectar'];
	const sisEhCerebro = (x) => !!x && SIS_CEREBRO.includes(x.assignment);
	const SIS_CRAFT_MODOS = ['fixo', 'regras'];
	const SIS_ESPECIAIS = ['', 'f2', 'f3', 't2', 't3'];
	// mesmos campos e limites de inc/bot.php (o navegador valida de novo antes de mostrar para aceite)
	const SIS_GLOBAL = {
		priorityFee: ['int', 1, 0, 50000], minPriorityFeeForMultiIx: ['int', 0, 0, 50000], automaticFee: ['bool', false], automaticFeeStep: ['int', 80, 1, 1000],
		automaticFeeMin: ['int', 1, 0, 50000], automaticFeeMax: ['int', 10000, 0, 50000], automaticFeeTimeMin: ['int', 10, 1, 300], automaticFeeTimeMax: ['int', 70, 1, 600],
		craftingTxMultiplier: ['int', 200, 1, 500], craftingTxAffectsAutoFee: ['bool', true], craftingJobs: ['int', 4, 0, 20],
		transportKeep1: ['bool', false], transportLoadUnloadSingleTx: ['bool', false], transportUnloadsUnknownRSS: ['bool', false], transportUseAmmoBank: ['bool', true],
		transportStopOnError: ['bool', true], transportFuel100: ['bool', true], minerUnloadsAll: ['bool', false], minerSupplySingleTx: ['bool', false], minerKeep1: ['bool', false],
		starbaseKeep1: ['bool', false], fleetForceConsumableAmount: ['bool', true], subwarpShortDist: ['bool', true], queueExitWarpSubwarp: ['bool', false],
		confirmationCheckingDelay: ['int', 2000, 2000, 10000], autoStartScript: ['bool', false], reloadPageOnFailedFleets: ['int', 0, 0, 999], excludeFleets: ['str', '', 0, 4000],
		scanBlockPattern: ['enum', 'square'], scanBlockLength: ['int', 5, 1, 50], scanBlockResetAfterResupply: ['bool', false], scanResupplyOnLowFuel: ['bool', false],
		scanSectorRegenTime: ['int', 90, 0, 3600], scanPauseTime: ['int', 600, 0, 7200], scanStrikeCount: ['int', 3, 1, 50],
		supplyPctMax: ['int', 40, 1, 100], supplyPctMin: ['int', 100, 1, 1000], craftModo: ['enum', 'fixo'],
		multiIxFeeAuto: ['bool', false], multiIxFeeAutoValor: ['int', 300, 1, 5000], multiIxFeeAutoSeg: ['int', 30, 5, 600],
		sduMapa: ['bool', false], prospectarAtivo: ['bool', true], craftColetar: ['bool', true],
		// [1.5.2] chaves "Aceite no navegador": desligada NA CONFIGURAÇÃO JÁ ACEITA = mudança só nas frotas / só no crafting é aplicada sozinha
		aceiteFrotas: ['bool', true], aceiteCrafting: ['bool', true],
	};
	const SIS_FROTA = {
		assignment: ['enum', ''], reserva: ['enum', ''], mineResource: ['mint', ''], dest: ['coord', ''], starbase: ['coord', ''], subwarpPref: ['int', 0, 0, 3], supplyReabastecer: ['enum', 'origem'], prioridade: ['bool', false],
		transportResource1: ['mint', ''], transportResource1Perc: ['int', 0, 0, 1e9], transportResource1Crew: ['int', 0, 0, 1e6],
		transportResource2: ['mint', ''], transportResource2Perc: ['int', 0, 0, 1e9], transportResource3: ['mint', ''], transportResource3Perc: ['int', 0, 0, 1e9],
		transportResource4: ['mint', ''], transportResource4Perc: ['int', 0, 0, 1e9],
		transportSBResource1: ['mint', ''], transportSBResource1Perc: ['int', 0, 0, 1e9], transportSBResource1Crew: ['int', 0, 0, 1e6],
		transportSBResource2: ['mint', ''], transportSBResource2Perc: ['int', 0, 0, 1e9], transportSBResource3: ['mint', ''], transportSBResource3Perc: ['int', 0, 0, 1e9],
		transportSBResource4: ['mint', ''], transportSBResource4Perc: ['int', 0, 0, 1e9],
		scanMin: ['int', 15, 0, 100], scanMin2: ['int', 0, 0, 100], scanMin3: ['int', 0, 0, 100], scanMove: ['bool', false], scanPattern: ['enum', ''], scanPatternLength: ['int', 0, 0, 100],
		scanSearchDist: ['int', 15, 1, 100], scanClusterFactor: ['int', 50, 0, 100], scanNeighborhoodMinGood: ['int', 3, 0, 25], scanCheckWhileCooldownLeft: ['int', 0, 0, 100],
		scanBypassPercent: ['int', 4, 0, 100], scanHomeAtPercent: ['int', 0, 0, 1000],
		prospRaio: ['int', 8, 2, 30], prospMinAuto: ['bool', true], prospMin: ['int', 15, 0, 100],
	};
	const SIS_SUPPLY = { mint: ['mint', ''], origem: ['coord', ''], origemMin: ['int', 0, 0, 1e9], destino: ['coord', ''], min: ['int', 0, 0, 1e9], max: ['int', 0, 0, 1e9], reabastecer: ['enum', 'origem'],
		ativo: ['bool', true], movimento: ['int', -1, -1, 3], critico: ['int', 0, 0, 1e9], origemCritico: ['int', 0, 0, 1e9],
		// [1.5.3] campos que só o cérebro usa: guardados para as diferenças e para o "aplica sozinha" verem a mudança
		viagemMin: ['int', 0, 0, 1e9], somaMint: ['mint', ''], somaFator: ['num', 1, 0.0001, 1e6] };
	const SIS_GARIMPO = { mint: ['mint', ''], setor: ['coord', ''], destino: ['coord', ''], min: ['int', 0, 0, 1e9], max: ['int', 0, 0, 1e9], critico: ['int', 0, 0, 1e9], ativo: ['bool', true], movimento: ['int', -1, -1, 3],
		// [1.5.3] só o cérebro usa (gatilho, em conjunto); a condição (cond) é validada à parte
		somaMint: ['mint', ''], somaFator: ['num', 1, 0.0001, 1e6], gatilhoMint: ['mint', ''], gatilhoSb: ['coord', ''], gatilhoAbaixo: ['int', 0, 0, 1e9] };
	const SIS_REGRA = { ativo: ['bool', true], receita: ['str', '', 0, 100], starbase: ['coord', ''], lote: ['int', 1, 1, 1e9], tripulacao: ['int', 1, 1, 1e6], produtoMin: ['int', 0, 0, 1e9],
		produtoMax: ['int', 0, 0, 1e9], lucroPct: ['num', -1, -1, 100000], lucroUsd: ['num', -1, -1, 1e9], encadear: ['bool', true] };
	const SIS_CRAFT = { coordinates: ['coord', ''], crew: ['int', 0, 0, 1e6], item: ['str', '', 0, 100], amount: ['int', 0, 0, 1e9], belowAmount: ['int', 0, 0, 1e9], special: ['enum', ''] };
	const SIS_ROTULOS = {
		reserva: 'tarefa reserva', aceiteFrotas: 'mudanças nas frotas precisam de aceite', aceiteCrafting: 'mudanças no crafting precisam de aceite',
		assignment: 'tarefa', prioridade: 'prioridade pela ordem da lista', mineResource: 'recurso', dest: 'destino', starbase: 'starbase', subwarpPref: 'movimento', supplyReabastecer: 'reabastecer', scanMin: 'scan mínimo', scanMin2: 'desistir abaixo', scanMin3: 'desistir após sucesso',
		scanMove: 'mover escaneando', scanPattern: 'padrão', scanPatternLength: 'comprimento', scanSearchDist: 'distância de busca', scanClusterFactor: 'agrupamento', scanNeighborhoodMinGood: 'vizinhos bons',
		scanCheckWhileCooldownLeft: 'conferir na recarga', scanBypassPercent: 'desvio', scanHomeAtPercent: 'voltar com combustível', coordinates: 'starbase', crew: 'tripulação', item: 'receita', amount: 'quantidade',
		belowAmount: 'só abaixo de', special: 'especial', priorityFee: 'taxa de prioridade', automaticFee: 'taxa automática', craftingJobs: 'trabalhos de crafting', excludeFleets: 'frotas ignoradas',
		sduMapa: 'auto(…) com o mapa de SDU do Sis-Ágora', prospectarAtivo: 'Prospectar SDU ligado (chave geral)', prospRaio: 'raio da área de prospecção', prospMinAuto: 'chance mínima pelo custo', prospMin: 'chance mínima fixa',
	};

	let sisLocal = Object.assign({}, SIS_PADRAO_LOCAL);
	async function sisCarregarLocal() {
		try { sisLocal = Object.assign({}, SIS_PADRAO_LOCAL, JSON.parse(await GM.getValue(SIS_LOCAL_KEY, '{}'))); } catch (e) { sisLocal = Object.assign({}, SIS_PADRAO_LOCAL); }
	}
	async function sisSalvarLocal() { await GM.setValue(SIS_LOCAL_KEY, JSON.stringify(sisLocal)); }
	await sisCarregarLocal();

	const sisFila = { eventos: [], producao: [], comandos: [] };
	const sisStatus = new Map();
	const sisTxHora = [];
	// sessão desta página: se a página for recarregada no meio de uma ordem, o dashboard marca a ordem como interrompida
	const sisSessao = Math.random().toString(36).slice(2, 10) + Date.now().toString(36);
	const sisSessaoInicio = Date.now();

	// =====================================================================================================
	// [1.4.0] Vários navegadores no mesmo perfil.
	// Cada navegador tem um id fixo (GM sisInstancia). O dashboard decide o papel: PRIMÁRIO (executa tudo), RESERVA
	// (aquecida: sincroniza, mede os RPCs, espelha a configuração aceita e assume se o primário cair ou ficar pior) ou
	// BLOQUEADO (sem liberação para vários navegadores, desligado no dashboard).
	// Sem conflito: só assina transação o primário com o prazo (lease) válido. Cada contato com o dashboard renova o prazo;
	// sem contato além da janela de autonomia, o primário para sozinho (o dashboard só promove outro depois disso).
	// Época: sobe a cada troca; papel ou época diferente => para o assistente e recusa transações (txSignAndSend).
	// Duas abas do mesmo navegador: a mais nova fica parada (trava GM sisAba).
	// =====================================================================================================
	const SIS_INST_KEY = 'sisInstancia';
	function sisNomePadrao() {
		const u = navigator.userAgent || '';
		const so = /Windows/.test(u) ? 'Windows' : /Android/.test(u) ? 'Android' : /Mac OS/.test(u) ? 'Mac' : /Linux/.test(u) ? 'Linux' : 'PC';
		const nav = /Edg\//.test(u) ? 'Edge' : /OPR\//.test(u) ? 'Opera' : /Firefox\//.test(u) ? 'Firefox' : /Chrome\//.test(u) ? 'Chrome' : 'navegador';
		return `${so} · ${nav}`;
	}
	let sisInstLocal = {};
	try { sisInstLocal = JSON.parse(await GM.getValue(SIS_INST_KEY, '{}')) || {}; } catch (e) { sisInstLocal = {}; }
	if (!/^[a-z0-9]{12,32}$/.test(String(sisInstLocal.id || ''))) {
		sisInstLocal = { id: Array.from(crypto.getRandomValues(new Uint8Array(12)), (b) => (b % 36).toString(36)).join('') + Date.now().toString(36).slice(-4), nome: sisNomePadrao() };
		try { await GM.setValue(SIS_INST_KEY, JSON.stringify(sisInstLocal)); } catch (e) { }
	}
	// papel: '' = ainda não sabe; 'unico' = sem dashboard (sem token) ou dashboard antigo; 'sem_contato' = primário que parou sozinho
	const sisInst = { papel: '', epoca: 0, leaseAte: 0, autonomiaMs: 30000, contatoEm: 0, motivo: '', rodar: false, reservas: 0, max: 1, primaria: null, troca: null,
		ultimaTroca: '', renomear: '', inicioPendente: false, parouSozinho: false, paradaSistema: false, avisoEm: 0 };
	// prazo do último contato (sobrevive a recarregar a página dentro da janela de autonomia)
	try { const u = JSON.parse(await GM.getValue('sisInstUltimo', '{}')); if (u && (u.papel === 'primario') && +u.leaseAte > Date.now()) { sisInst.papel = 'primario'; sisInst.epoca = +u.epoca || 0; sisInst.leaseAte = +u.leaseAte; } } catch (e) { }
	let sisTxEmCurso = 0, sisTxOkEm = 0;
	const sisPingRtt = [];
	function sisPrimarioAgora() { return (sisInst.papel === 'primario' || sisInst.papel === 'saindo') && Date.now() < sisInst.leaseAte; }
	// pode assinar transação?
	function sisPodeAgir() {
		if (sisAbaDuplicada) return false;
		if (!sisLocal.token || sisInst.papel === 'unico') return true;
		return sisPrimarioAgora();
	}
	// pode ligar o assistente? (o navegador que está passando o posto não liga de novo)
	function sisPodeIniciar() {
		if (sisAbaDuplicada) return false;
		if (!sisLocal.token || sisInst.papel === 'unico') return true;
		return sisInst.papel === 'primario' && Date.now() < sisInst.leaseAte;
	}
	const SIS_PAPEL_TXT = { primario: 'PRIMÁRIO', saindo: 'passando o posto', reserva: 'reserva', bloqueada: 'bloqueado', sem_contato: 'sem contato (parado)', unico: 'único', '': 'conectando…' };
	function sisInstTexto() {
		if (sisAbaDuplicada) return 'outra aba deste navegador já está com o Sis-Bot aberto (esta fica parada)';
		const p = SIS_PAPEL_TXT[sisInst.papel] ?? sisInst.papel;
		return `${sisInstLocal.nome || 'este navegador'}: ${p}` + (sisInst.epoca ? ` · época ${sisInst.epoca}` : '')
			+ (sisInst.papel !== 'primario' && sisInst.primaria ? ` · primário: ${sisInst.primaria}` : '')
			+ (sisInst.troca ? ` · troca para ${sisInst.troca.para}` : '') + (sisInst.motivo && sisInst.papel !== 'primario' ? ` (${sisInst.motivo})` : '');
	}
	// o que o dashboard precisa saber deste navegador
	function sisInstCorpo(sync) {
		const o = { id: sisInstLocal.id, nome: sisInstLocal.nome || sisNomePadrao(), epoca: sisInst.epoca, papel: sisInst.papel,
			pronto_sair: sisInst.papel === 'saindo' && !enableAssistant && !globalWaitForStopSequence && sisTxEmCurso === 0 };
		if (sisInst.renomear) o.renomear = sisInst.renomear;
		if (sync) {
			const r = sisPingRtt.slice().sort((a, b) => a - b);
			o.metr = { ping_ms: r.length ? Math.round(r[Math.floor(r.length / 2)]) : null, rpcs: sisRpcMetricas(), ua: sisNomePadrao(),
				fuso: (() => { try { return Intl.DateTimeFormat().resolvedOptions().timeZone; } catch (e) { return ''; } })(), modo_rpc: sisRpcDash.origem || '',
				// [1.4.2] hot wallet no script ou extensão (que espera aprovação e trava o bot): navegador com extensão não assume sozinho
				carteira: (() => { try { return customKeypair ? 'hot' : 'extensao'; } catch (e) { return ''; } })(), tx_ok_em: sisTxOkEm || null };
		}
		return o;
	}
	// resposta do dashboard (sync e ping): papel, época, prazo
	async function sisInstReceber(x) {
		if (!x || typeof x !== 'object') { if (sisInst.papel === '') sisInst.papel = 'unico'; return; }
		const antes = sisInst.papel, epAntes = sisInst.epoca;
		sisInst.papel = ['primario', 'saindo', 'reserva', 'bloqueada'].includes(x.papel) ? x.papel : 'reserva';
		sisInst.epoca = +x.epoca || 0; sisInst.motivo = String(x.motivo || '').slice(0, 200); sisInst.rodar = !!x.rodar;
		sisInst.reservas = +x.reservas || 0; sisInst.max = +x.max || 1; sisInst.primaria = x.primaria ? String(x.primaria).slice(0, 40) : null;
		sisInst.troca = x.troca && typeof x.troca === 'object' ? { para: String(x.troca.para || '').slice(0, 40), motivo: String(x.troca.motivo || '') } : null;
		sisInst.ultimaTroca = String(x.ultima_troca || ''); sisInst.contatoEm = Date.now();
		sisInst.autonomiaMs = Math.max(10, Math.min(900, +x.autonomia_s || 30)) * 1000;
		if (sisInst.renomear && x.nome === sisInst.renomear) sisInst.renomear = '';
		if (x.nome && x.nome !== sisInstLocal.nome && !sisInst.renomear) { sisInstLocal.nome = String(x.nome).slice(0, 40); try { await GM.setValue(SIS_INST_KEY, JSON.stringify(sisInstLocal)); } catch (e) { } }
		try { sisRpcFator(+x.rpc_fator || 1); } catch (e) { }
		const prim = sisInst.papel === 'primario' || sisInst.papel === 'saindo';
		sisInst.leaseAte = prim ? Date.now() + sisInst.autonomiaMs : 0;
		try { await GM.setValue('sisInstUltimo', JSON.stringify({ papel: sisInst.papel, epoca: sisInst.epoca, leaseAte: sisInst.leaseAte })); } catch (e) { }
		if (antes !== sisInst.papel || epAntes !== sisInst.epoca) await sisInstMudou(antes, epAntes);
		// passagem planejada: para com calma (termina dock/carga; ainda pode assinar) e avisa pronto_sair no próximo contato
		if (sisInst.papel === 'saindo' && enableAssistant && !globalWaitForStopSequence) {
			sisEvento('info', '', `Passando o posto de primário para ${sisInst.troca ? sisInst.troca.para : 'outro navegador'}: parando o Sis-Bot com calma neste navegador.`);
			sisInst.paradaSistema = true; toggleAssistant();
		}
		sisAtualizarBotaoInicio();
	}
	async function sisInstMudou(antes, epAntes) {
		const p = sisInst.papel; const eraPrim = antes === 'primario' || antes === 'saindo';
		if (p === 'primario' && antes !== 'primario') {
			// liga se o primário anterior estava rodando; no primeiro navegador do perfil, se este queria rodar; ou se ele mesmo parou por falta de contato
			const ligar = sisInst.rodar || sisInst.parouSozinho || (sisInst.inicioPendente && (sisInst.ultimaTroca === 'inicio' || antes === 'unico' || antes === ''));
			if (antes !== 'sem_contato' || epAntes !== sisInst.epoca) sisEvento('info', '', `Este navegador (${sisInstLocal.nome}) é o primário do perfil agora (época ${sisInst.epoca})${ligar ? ': ligando o Sis-Bot' : ''}.`);
			else sisEvento('info', '', `Contato com o dashboard de volta: ${sisInstLocal.nome} continua primário${ligar ? ' e religa o Sis-Bot' : ''}.`);
			sisInst.inicioPendente = false; sisInst.parouSozinho = false;
			if (ligar && !enableAssistant) sisReligar();
			return;
		}
		if (eraPrim && p !== 'primario' && p !== 'saindo') {
			sisEvento('info', '', `Este navegador deixou de ser o primário${sisInst.primaria ? ' (primário agora: ' + sisInst.primaria + ')' : ''}${sisInst.motivo ? ' — ' + sisInst.motivo : ''}. Fica de reserva.`);
			if (enableAssistant) { sisInst.paradaSistema = true; toggleAssistant(); }
		}
		if (p === 'bloqueada' && antes !== 'bloqueada') sisEvento('alerta', '', `Este navegador está bloqueado no dashboard: ${sisInst.motivo || 'sem liberação'}.`);
	}
	// sem contato além da janela de autonomia: o primário para sozinho (o dashboard pode promover uma reserva)
	setInterval(() => {
		try {
			if (!sisLocal.token || (sisInst.papel !== 'primario' && sisInst.papel !== 'saindo')) return;
			if (Date.now() < sisInst.leaseAte) { if (sisInst.leaseAte - Date.now() < 15000) sisPingar(true); return; }
			const s = Math.round((Date.now() - (sisInst.contatoEm || sisSessaoInicio)) / 1000);
			sisInst.papel = 'sem_contato';
			sisEvento('alerta', '', `Sem contato com o dashboard há ${s} s: este navegador parou o Sis-Bot para não operar junto com outro navegador do perfil. Volta sozinho quando o contato voltar (se continuar primário).`);
			if (enableAssistant) { sisInst.parouSozinho = true; sisInst.paradaSistema = true; toggleAssistant(); }
			sisAtualizarBotaoInicio();
		} catch (e) { }
	}, 2000);
	// toggleAssistant (patch do transformar): tentar ligar sem poder
	function sisInicioNegado() {
		sisInst.inicioPendente = true;
		if (Date.now() - sisInst.avisoEm > 60000) { sisInst.avisoEm = Date.now(); sisEvento('info', '', `Sis-Bot não ligado neste navegador — ${sisInstTexto()}. Ele liga sozinho se virar o primário.`); }
		sisAtualizarBotaoInicio();
	}
	function sisAtualizarBotaoInicio() {
		try {
			const s = document.querySelector('#autoScanBtn > span'); if (!s || enableAssistant || globalWaitForStopSequence || !initComplete) return;
			const txt = sisAbaDuplicada ? 'Outra aba' : sisPodeIniciar() ? 'Start' : sisInst.papel === 'bloqueada' ? 'Bloqueado' : sisInst.papel === '' ? 'Conectando' : 'Reserva';
			if (s.textContent !== 'Reload' && s.textContent !== txt) setTxt(s, txt);
			s.parentElement && (s.parentElement.title = sisInstTexto());
		} catch (e) { }
	}
	// trava de transação (fencing): o txSignAndSend do SLYA virou sisTxOriginal (transformar). Fora do primário, devolve um
	// resultado de falha no mesmo formato (o SLYA nunca rejeita essa promessa; quem chama confere meta.err).
	function txSignAndSend(ix, fleet, opName, priorityFeeMultiplier, extraSigner = false) {
		if (!sisPodeAgir()) {
			const m = `Sis-Bot: transação ${opName || ''} não enviada — ${sisInstTexto()}`;
			cLog(1, `${fleet && fleet.label ? FleetTimeStamp(fleet.label) + ' ' : ''}${m}`);
			try { if (fleet && fleet.publicKey) updateFleetState(fleet, 'Sis-Bot: navegador de reserva'); else if (fleet) fleet.state = 'Sis-Bot: navegador de reserva'; } catch (e) { }
			return Promise.resolve({ slot: 0, blockTime: null, sisRecusada: true,
				meta: { err: { SisBot: 'navegador não é o primário' }, logMessages: [m], fee: 0, preTokenBalances: [], postTokenBalances: [], innerInstructions: [] },
				transaction: { signatures: [], message: { staticAccountKeys: [], accountKeys: [] } } });
		}
		sisTxEmCurso++;
		return sisTxOriginal(ix, fleet, opName, priorityFeeMultiplier, extraSigner).finally(() => { sisTxEmCurso = Math.max(0, sisTxEmCurso - 1); });
	}
	// duas abas do mesmo navegador (mesmo id): a aberta antes fica; a outra para
	let sisAbaDuplicada = false;
	async function sisAbaVerificar() {
		try {
			const a = JSON.parse(await GM.getValue('sisAba', '{}')) || {};
			const outra = a.sessao && a.sessao !== sisSessao && Date.now() - (+a.t || 0) < 5000;
			const maisVelha = outra && ((+a.inicio || 0) < sisSessaoInicio || ((+a.inicio || 0) === sisSessaoInicio && String(a.sessao) < sisSessao));
			if (maisVelha) {
				if (!sisAbaDuplicada) { sisAbaDuplicada = true; sisEvento('alerta', '', 'Outra aba deste navegador já está com o Sis-Bot aberto: esta aba fica parada (não sincroniza nem assina). Feche uma das duas.'); if (enableAssistant) { sisInst.paradaSistema = true; toggleAssistant(); } sisAtualizarBotaoInicio(); }
				return;
			}
			if (sisAbaDuplicada) { sisAbaDuplicada = false; sisEvento('info', '', 'A outra aba foi fechada: esta aba volta a sincronizar.'); }
			await GM.setValue('sisAba', JSON.stringify({ sessao: sisSessao, inicio: sisSessaoInicio, t: Date.now() }));
		} catch (e) { }
	}
	setInterval(sisAbaVerificar, 1500);
	try { window.addEventListener('pagehide', () => { try { if (!sisAbaDuplicada) GM.setValue('sisAba', '{}'); } catch (e) { } }); } catch (e) { }
	await sisAbaVerificar();
	// [1.4.1] religar depois de uma parada do sistema: espera a parada terminar (o toggleAssistant ignorava o pedido durante o "Wait...")
	let sisReligando = false;
	async function sisReligar() {
		if (sisReligando) return; sisReligando = true;
		try {
			for (let i = 0; i < 400 && (globalWaitForStopSequence || !initComplete); i++) await wait(3000);
			if (!enableAssistant && sisPodeIniciar()) { try { sisPainelMostrar(); } catch (e) { } toggleAssistant(); }
		} finally { sisReligando = false; }
	}
	// laço de parada passou de 3 min (frota presa em Docking/Loading por ordem sem resposta): segue e avisa
	function sisParadaLonga() {
		const presas = userFleets.filter((f) => ['Mine Starting', 'Mining Stop', 'Unloading', 'Loading', 'Docking', 'Undocking', 'Refueling'].some((v) => String(f.state || '').startsWith(v))).map((f) => f.label);
		sisEvento('alerta', '', `Parada do Sis-Bot concluída depois de 3 min esperando ${presas.length ? presas.join(', ') : 'uma etapa'} (etapa sem resposta). O Sis-Bot recalcula essas frotas pela blockchain.`);
		presas.forEach((pk) => { try { const f = userFleets.find((x) => x.label === pk); if (f) updateFleetState(f, 'Sis-Bot: etapa sem resposta (parado)', true); } catch (e) { } });
	}
	// tempo de confirmação das transações (para o alerta e a taxa mínima automática de várias instruções)
	const sisTxTempos = [];
	const sisFeeAuto = { ligada: false, desde: 0, manual: null, aplicado: null, ultimaLenta: 0 };
	// 1.2.2: taxa paga (lamports, da própria transação confirmada) e número de transações por frota — vai em cada ordem do Sis-Bot
	const sisTaxaFrota = new Map();
	function sisMedirTx(nIx, ms, fleet, res) {
		try {
			const k = fleet && fleet.publicKey ? fleet.publicKey.toString() : (fleet && fleet.label) || '';
			if (k) { const a = sisTaxaFrota.get(k) || { lamports: 0, n: 0 }; a.lamports += (res && res.meta && +res.meta.fee) || 0; a.n++; sisTaxaFrota.set(k, a); }
		} catch (e) { }
		sisTxTempos.push({ t: Date.now(), multi: nIx > 1, s: ms / 1000 });
		if (res && res.meta && !res.meta.err) sisTxOkEm = Date.now(); // [1.4.2] última transação confirmada (saúde do navegador)
		while (sisTxTempos.length > 300 || (sisTxTempos.length && sisTxTempos[0].t < Date.now() - 3600000)) sisTxTempos.shift();
	}
	function sisResumoTx(multi, min = 30) {
		const l = sisTxTempos.filter((x) => x.multi === multi && x.t > Date.now() - min * 60000).map((x) => x.s).sort((a, b) => a - b);
		if (!l.length) return { n: 0, mediana: null, p90: null };
		return { n: l.length, mediana: +l[Math.floor(l.length / 2)].toFixed(1), p90: +l[Math.min(l.length - 1, Math.floor(l.length * 0.9))].toFixed(1) };
	}
	// liga a taxa mínima de várias instruções quando elas demoram (mediana dos últimos 30 min acima do limite, com pelo menos 5 transações)
	// e volta ao valor configurado quando ficam rápidas (mediana abaixo de metade do limite) por 30 min
	function sisAjustarFeeMulti() {
		const g = globalSettings;
		if (!sisFeeAuto.ligada || g.minPriorityFeeForMultiIx !== sisFeeAuto.aplicado) sisFeeAuto.manual = g.minPriorityFeeForMultiIx; // valor da configuração aceita
		if (!g.multiIxFeeAuto) { if (sisFeeAuto.ligada) { g.minPriorityFeeForMultiIx = sisFeeAuto.manual; sisFeeAuto.ligada = false; } return; }
		const r = sisResumoTx(true); const lim = +g.multiIxFeeAutoSeg || 30;
		const teto = Math.max(0, Math.min(50000, parseInt(sisLocal.tetoTaxa) || 0));
		const valor = Math.min(+g.multiIxFeeAutoValor || 300, teto);
		if (r.n >= 5 && r.mediana > lim) sisFeeAuto.ultimaLenta = Date.now();
		if (!sisFeeAuto.ligada && r.n >= 5 && r.mediana > lim && valor > (sisFeeAuto.manual || 0)) {
			sisFeeAuto.ligada = true; sisFeeAuto.desde = Date.now(); g.minPriorityFeeForMultiIx = sisFeeAuto.aplicado = valor;
			sisEvento('alerta', '', `Taxa mínima p/ várias instruções ligada automaticamente (${valor}): elas estão levando ${r.mediana} s (mediana de ${r.n} nos últimos 30 min).`);
		} else if (sisFeeAuto.ligada) {
			g.minPriorityFeeForMultiIx = sisFeeAuto.aplicado = Math.max(valor, sisFeeAuto.manual || 0);
			if (Date.now() - sisFeeAuto.ultimaLenta > 30 * 60000 && (r.n === 0 || r.mediana < lim / 2)) {
				sisFeeAuto.ligada = false; g.minPriorityFeeForMultiIx = sisFeeAuto.manual;
				sisEvento('info', '', `Taxa mínima p/ várias instruções voltou ao valor configurado (${sisFeeAuto.manual}): transações rápidas de novo.`);
			}
		}
	}
	let sisSaldo = null, sisSaldoEm = 0, sisPendente = null, sisCatalogoEnviar = true, sisIntervalo = 30, sisTimer = null, sisSincronizando = false, sisLimiteParou = false, sisAplicando = false;
	let sisUltimo = { ok: null, msg: 'ainda não sincronizou', em: 0, perfil: '' };

	// 1.2.2: logs de programa (ix error) viram uma linha com as instruções e o código do erro (antes o corte em 500 caracteres comia o código)
	function sisResumirIx(m) {
		m = String(m ?? ''); if (!m.includes('Program log:')) return m;
		const ins = [...new Set([...m.matchAll(/Instruction: (\w+)/g)].map((x) => x[1]))];
		const e = m.match(/Error Code: (\w+)(?:\. Error Number: (\d+))?(?:\. Error Message: ([^,.]+))?/);
		// [1.3.7] erros do runtime (não são códigos do programa): CU esgotado e afins
		const rt = e ? '' : /Computational budget exceeded|exceeded CUs meter/i.test(m) ? 'CU esgotado (limite de CU da transação)'
			: (m.match(/failed: ([^\n,]+)/) || [])[1] || '';
		return `${m.split(':')[0]} — ${ins.join(' › ')} — erro ${e ? e[1] + (e[2] ? ` (${e[2]})` : '') + (e[3] ? ': ' + e[3] : '') : rt || '?'}`;
	}
	function sisEvento(tipo, frota, msg) {
		msg = sisResumirIx(msg);
		sisFila.eventos.push({ t: Date.now(), tipo, frota: frota ? String(frota).slice(0, 64) : '', msg: String(msg ?? '').slice(0, 500) });
		if (sisFila.eventos.length > 300) sisFila.eventos.splice(0, sisFila.eventos.length - 300);
	}
	function sisProducao(tipo, frota, item, qtd) {
		qtd = Number(qtd);
		if (!item || !isFinite(qtd) || qtd <= 0) return;
		const mint = item === 'ATLAS' ? 'ATLASXmbPQxBUYbxPsV97usA3fPQYEqzQBUHgiFCUsXx' : (cargoItems.find(r => r.name == item) || {}).token || null;
		sisFila.producao.push({ t: Date.now(), tipo, frota: String(frota || '').slice(0, 64), item: String(item).slice(0, 64), mint, qtd });
		if (sisFila.producao.length > 800) sisFila.producao.splice(0, sisFila.producao.length - 800);
	}
	function sisRegistrarStatus(fleet) {
		if (!fleet || fleet.sisColeta) return;
		if (fleet.publicKey) sisStatus.set(fleet.publicKey.toString(), { pk: fleet.publicKey.toString(), nome: fleet.label, estado: fleet.state, comida: fleet.foodCnt, sdu: fleet.sduCnt, combustivel: fleet.fuelCnt, cor: fleet.fontColor });
		else if (fleet.label) sisStatus.set(fleet.label, { label: fleet.label, estado: fleet.state, item: fleet.item || '', coords: fleet.coordinates || '', craftingId: fleet.craftingId || 0, unica: !!fleet.sisUnica });
	}
	function sisContarTx(frota) {
		const agora = Date.now();
		sisTxHora.push(agora);
		while (sisTxHora.length && sisTxHora[0] < agora - 3600000) sisTxHora.shift();
		const teto = parseInt(sisLocal.tetoTxHora) || 0;
		if (teto > 0 && sisTxHora.length > teto && enableAssistant && !sisLimiteParou) {
			sisLimiteParou = true;
			sisEvento('alerta', frota, `Teto local de ${teto} transações por hora atingido: o Sis-Bot foi parado.`);
			toggleAssistant();
			setTimeout(() => { sisLimiteParou = false; }, 600000);
		}
	}

	// ---------- produção: as linhas que o SLYA mandava para o Influx viram registros de produção para a VPS
	function sisDividir(s, sep) {
		const out = []; let cur = '', q = false;
		for (let i = 0; i < s.length; i++) {
			const ch = s[i];
			if (ch === '\\' && i + 1 < s.length) { cur += ch + s[i + 1]; i++; continue; }
			if (ch === '"') { q = !q; cur += ch; continue; }
			if (ch === sep && !q) { out.push(cur); cur = ''; continue; }
			cur += ch;
		}
		out.push(cur);
		return out;
	}
	const sisDesesc = (s) => String(s ?? '').replace(/\\(.)/g, '$1').replace(/^"|"$/g, '');
	function sisLerInflux(linha) {
		const [cab, campos] = sisDividir(linha.trim(), ' ');
		const [medida, ...tags] = sisDividir(cab, ',');
		const T = {}, C = {};
		tags.forEach(t => { const [k, ...v] = sisDividir(t, '='); T[sisDesesc(k)] = sisDesesc(v.join('=')); });
		sisDividir(campos || '', ',').forEach(t => { const [k, ...v] = sisDividir(t, '='); const val = sisDesesc(v.join('=')); C[sisDesesc(k)] = val !== '' && !isNaN(+val) ? +val : val; });
		return { medida: sisDesesc(medida), T, C };
	}
	function sisInfluxParaProducao(msg) {
		for (const linha of String(msg).split('\n')) {
			if (!linha.trim()) continue;
			let r; try { r = sisLerInflux(linha); } catch (e) { continue; }
			const { medida, T, C } = r; const frota = T.fleet || '';
			if (medida === 'mining') {
				sisProducao('minerado', frota, T.rss, C.amount);
				sisProducao('consumo', frota, fuelItem && fuelItem.name, C.burnedFuel); sisProducao('consumo', frota, foodItem && foodItem.name, C.burnedFood); sisProducao('consumo', frota, ammoItem && ammoItem.name, C.burnedAmmo);
			} else if (medida === 'sdu') {
				sisProducao('sdu', frota, sduItem && sduItem.name, C.amount); sisProducao('consumo', frota, foodItem && foodItem.name, C.burnedFood);
			} else if (medida === 'fleetrss') {
				sisProducao(String(T.type).endsWith('_out') ? 'descarregado' : 'carregado', frota, T.rss, C.amount);
			} else if (medida === 'crafting') {
				const onde = 'Crafting ' + (T.starbase || '');
				if (T.type === 'Output') { sisProducao('craft_saida', onde, T.output, C.amount); sisProducao('taxa_craft', onde, 'ATLAS', C.fee); }
				else sisProducao('craft_entrada', onde, T.input, C.amount);
			} else if (medida === 'movement') {
				sisProducao('consumo', frota, fuelItem && fuelItem.name, C.burnedFuel); sisProducao('movimento', frota, C.type || 'warp', C.moveDist);
			}
		}
	}

	// ---------- conexão
	function sisPost(rota, corpo) {
		return new Promise(resolve => {
			GM_xmlhttpRequest({
				method: 'POST', url: SIS_URL + '?r=' + rota, timeout: 25000, data: JSON.stringify(corpo),
				headers: { 'Content-Type': 'application/json', 'X-Sis-Token': sisLocal.token },
				onload: (r) => { let j = null; try { j = JSON.parse(r.responseText); } catch (e) {} resolve({ status: r.status, j }); },
				onerror: () => resolve({ status: 0, j: null }),
				ontimeout: () => resolve({ status: 0, j: { erro: 'tempo esgotado' } }),
			});
		});
	}
	// Fase A: o mapa de SDU dos padrões auto(…) vem do Sis-Ágora (mesmo formato do sdu.json do SLYA); sem ele, o do SLYA
	let sisMapaAviso = 0;
	async function sisFetchMapaSdu(url) {
		if (globalSettings.sduMapa && sisLocal.token && userProfileAcct) {
			const r = await sisPost('sdu', { perfil: userProfileAcct.toString() });
			if (r.status === 200 && r.j && r.j.ok && Array.isArray(r.j.scans)) return { ok: true, status: 200, statusText: 'ok', json: async () => ({ timestamp: r.j.timestamp, scans: r.j.scans }) };
			if (Date.now() - sisMapaAviso > 3600000) { sisMapaAviso = Date.now(); sisEvento('info', '', `Mapa de SDU do Sis-Ágora indisponível (${(r.j && r.j.erro) || r.status}): os padrões auto usam o do SLYA.`); }
		}
		return fetch(url);
	}
	async function sisSha256(texto) {
		const b = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(texto));
		return Array.from(new Uint8Array(b)).map(x => x.toString(16).padStart(2, '0')).join('');
	}
	function sisCatalogo() {
		return {
			starbases: validTargets.map(t => ({ nome: t.name, x: t.x, y: t.y })),
			itens: cargoItems.map(i => ({ nome: i.name, mint: i.token })),
			receitas: craftRecipes.filter(r => r.name.indexOf('SDU') === -1).map(r => r.name),
			upgrades: upgradeRecipes.filter(r => r.name.indexOf('SB Tier') === -1).map(r => r.name),
			mineraveis: SIS_MINERAVEIS,
			frotas: userFleets.map(f => ({ pk: f.publicKey.toString(), nome: f.label, carga: f.cargoCapacity, combustivel: f.fuelCapacity, municao: f.ammoCapacity, emprestada: !!f.emprestada })),
		};
	}
	async function sisAtualizarSaldo() {
		if (!userPublicKey || (enableAssistant && sisSaldo) || Date.now() - sisSaldoEm < 300000) return;
		sisSaldoEm = Date.now();
		try {
			const sol = await solanaReadConnection.getBalance(userPublicKey);
			const atlas = await solanaReadConnection.getParsedTokenAccountsByOwner(userPublicKey, { mint: new solanaWeb3.PublicKey('ATLASXmbPQxBUYbxPsV97usA3fPQYEqzQBUHgiFCUsXx') });
			sisSaldo = { sol: sol / 1000000000, atlas: atlas.value[0] ? atlas.value[0].account.data.parsed.info.tokenAmount.uiAmount : 0 };
		} catch (e) { cLog(2, 'SIS: saldo', e); }
	}
	// [1.3.5] tempos (lentidão da rede): desde quando cada frota está no estado atual; ciclos vindos do dashboard
	const sisEstDesde = new Map(); let sisCiclos = {}; let sisCiclosEm = 0;
	function sisDesdeEstado(pk, estado) {
		const k = String(estado || '').replace(/\s*\[[^\]]*\]|\d+(?:[.,]\d+)?\s*(?:min|h|s)\b|\d+%/g, '').trim(); // ignora contagens que mudam sozinhas
		const a = sisEstDesde.get(pk); if (a && a.k === k) return a.t;
		const t = Date.now(); sisEstDesde.set(pk, { k, t }); return t;
	}
	function sisTempo(ms) { const t = Math.max(0, Math.round(ms / 1000)); const h = Math.floor(t / 3600), m = Math.floor(t % 3600 / 60), s2 = t % 60;
		return t < 60 ? `${t} s` : h ? `${h}:${String(m).padStart(2, '0')}:${String(s2).padStart(2, '0')}` : `${m}:${String(s2).padStart(2, '0')}`; }
	setInterval(() => { try { for (const r of sisPainel.linhas.values()) if (r.tempo) sisPintarTempos(r); } catch (e) { } }, 1000);
	async function sisFrotasStatus() {
		const out = [];
		for (const f of userFleets) {
			const pk = f.publicKey.toString(); const s = sisStatus.get(pk) || {};
			let tarefa = '';
			try { const g = JSON.parse(await GM.getValue(pk, '{}')); tarefa = g.assignment || ''; sisPainel.tarefas.set(pk, { tarefa, sis: g.sisCerebro ? (g.sisTarefa || 'Supply') : '' }); } catch (e) {}
			const est0 = s.estado || f.state || '';
			out.push({ pk, nome: f.label, estado: est0, desde: sisDesdeEstado(pk, est0), comida: f.foodCnt, sdu: f.sduCnt, combustivel: f.fuelCnt, cor: f.fontColor || 'white', tarefa });
		}
		return out;
	}
	function sisAgendar(seg) { clearTimeout(sisTimer); sisTimer = setTimeout(sisSync, Math.max(5, seg) * 1000); }
	// 1.2.2: consulta leve a cada 3 s; havendo comando ou ordem esperando no dashboard, sincroniza na hora (mín. 5 s entre
	// sincronizações). Assim "parar/iniciar a tarefa" e as ordens do Sis-Bot chegam em segundos, sem esperar o intervalo do sync.
	let sisUltimoSyncEm = 0, sisPingOcupado = false, sisPingEm = 0;
	// [1.4.1] o ping também renova o prazo do primário: roda durante um sync demorado e antes de a página terminar de carregar
	// (antes, um sync preso de 25 s + recarga da página faziam o primário perder o prazo e parar à toa)
	setInterval(() => { sisPingar(false); }, 3000);
	async function sisPingar(urgente) {
		try { if (sisPingOcupado || !sisLocal.token || !userProfileAcct || sisAbaDuplicada) return; } catch (e) { return; }
		if (!urgente && Date.now() - sisPingEm < 2500) return;
		sisPingOcupado = true; sisPingEm = Date.now();
		try {
			const t0 = performance.now();
			const r = await sisPost('ping', { perfil: userProfileAcct.toString(), v: GM_info.script.version, inst: sisInstCorpo(false) });
			if (r.status === 200 && r.j) {
				// [1.4.0] ida e volta até o dashboard (sem o tempo do próprio servidor): entra na nota deste navegador
				const ms = performance.now() - t0 - (+r.j.srv_ms || 0); if (ms > 0 && ms < 30000) { sisPingRtt.push(ms); if (sisPingRtt.length > 40) sisPingRtt.shift(); }
				if (r.j.inst) await sisInstReceber(r.j.inst);
				if (+r.j.n > 0 && initComplete && !sisSincronizando && Date.now() - sisUltimoSyncEm >= 5200) sisSyncAgora();
			}
		}
		catch (e) { } finally { sisPingOcupado = false; }
	}
	async function sisSync() {
		if (sisSincronizando) return;
		if (!sisLocal.token || !initComplete || !userProfileAcct) { sisAtualizarUI(); sisAgendar(10); return; }
		if (sisAbaDuplicada) { sisUltimo = { ok: false, msg: 'outra aba deste navegador está com o Sis-Bot aberto', em: Date.now(), perfil: '' }; sisAtualizarUI(); sisAgendar(10); return; }
		sisSincronizando = true; sisUltimoSyncEm = Date.now();
		try { sisAjustarFeeMulti(); } catch (e) { cLog(2, 'Sis-Bot: taxa automática', e); }
		const lote = { eventos: sisFila.eventos.splice(0), producao: sisFila.producao.splice(0), comandos: sisFila.comandos.splice(0) };
		let r = { status: 0, j: null };
		try {
			await sisAtualizarSaldo();
			const botao = document.querySelector('#autoScanBtn > span');
			const corpo = {
				v: GM_info.script.version, perfil: userProfileAcct.toString(), chave: userPublicKey ? userPublicKey.toString() : '', rodando: !!enableAssistant, quer_rodar: !!sisLocal.querRodando,
				botao: botao ? botao.textContent : '', modo_local: !!sisLocal.modoLocal, cfg_aceita: sisLocal.cfgAceita ? { versao: sisLocal.cfgAceita.versao } : null,
				frotas: await sisFrotasStatus(), crafts: [...sisStatus.values()].filter(x => x.label), saldo: sisSaldo, taxa: currentFee,
				stats: { start: transactionStats.start, groups: transactionStats.groups, leituras: solanaReadCount, escritas: solanaWriteCount, erros: solanaErrorCount, rpc: sisRpcResumo() },
				eventos: lote.eventos, producao: lote.producao, comandos: lote.comandos, sessao: sisSessao, inst: sisInstCorpo(true),
				txs: { multi: sisResumoTx(true), simples: sisResumoTx(false), auto: sisFeeAuto.ligada, minMulti: globalSettings.minPriorityFeeForMultiIx },
			};
			if (sisCatalogoEnviar) corpo.catalogo = sisCatalogo();
			r = await sisPost('sync', corpo);
			if (r.status === 200 && r.j && r.j.ok) {
				sisUltimo = { ok: true, msg: 'conectado', em: Date.now(), perfil: r.j.perfil || '' };
				sisCatalogoEnviar = !corpo.catalogo && !!r.j.quer_catalogo;
				sisIntervalo = Math.max(10, Math.min(300, parseInt(r.j.intervalo) || 30));
				// [1.4.0] papel deste navegador (antes de configuração e ordens)
				await sisInstReceber(r.j.inst || null);
				// [1.3.0] RPCs definidos pelo dono em Administração › RPCs (só se "Usar os RPCs do dashboard" estiver ligado)
				if (r.j.rpcs) { try { await sisRpcReceberDash(r.j.rpcs); } catch (e) { cLog(2, 'Sis-Bot: RPCs do dashboard', e); } }
				if (r.j.ciclos && typeof r.j.ciclos === 'object') { sisCiclos = r.j.ciclos; sisCiclosEm = Date.now(); }
				if (r.j.taxa_limites) { try { await sisReceberTaxaLimites(r.j.taxa_limites); } catch (e) { cLog(2, 'Sis-Bot: limites de taxa', e); } }
				if (r.j.config && !sisLocal.modoLocal) await sisReceberConfig(r.j.config);
				for (const cmd of (r.j.comandos || [])) {
					// [1.4.0] o dashboard só manda ordens ao primário; se chegar aqui sem ser, recusa (não executa)
					if (sisLocal.token && sisInst.papel !== 'unico' && !sisPrimarioAgora()) { sisFila.comandos.push({ id: cmd.id, ok: false, msg: 'este navegador não é o primário agora' }); continue; }
					await sisExecutarComando(cmd);
				}
			} else {
				sisUltimo = { ok: false, msg: (r.j && r.j.erro) || (r.status ? 'erro ' + r.status : 'sem conexão com o dashboard'), em: Date.now(), perfil: '' };
				if (r.status === 0 || r.status >= 500 || r.status === 429) { // devolve à fila para a próxima tentativa
					sisFila.eventos.unshift(...lote.eventos); sisFila.producao.unshift(...lote.producao); sisFila.comandos.unshift(...lote.comandos);
				}
			}
		} catch (e) {
			sisUltimo = { ok: false, msg: 'falha: ' + (e && e.message ? e.message : e), em: Date.now(), perfil: '' };
			sisFila.eventos.unshift(...lote.eventos); sisFila.producao.unshift(...lote.producao); sisFila.comandos.unshift(...lote.comandos);
		}
		sisSincronizando = false;
		sisAtualizarUI();
		sisAgendar(r.status === 401 || r.status === 403 ? 120 : sisIntervalo);
		sisAtualizarBotaoInicio();
	}

	// ---------- comandos remotos (lista fixa; o membro pode bloquear)
	async function sisExecutarComando(cmd) {
		const responder = (ok, msg) => { sisFila.comandos.push({ id: cmd.id, ok, msg }); sisEvento('comando', '', `${cmd.acao}: ${msg}`); };
		if (!sisLocal.comandosRemotos) return responder(false, 'comandos remotos bloqueados neste navegador');
		if (sisAplicando) return responder(false, 'aplicando uma configuração');
		// 1.2.6: criar / adicionar naves / desfazer frota — fila própria (pode levar minutos) e resposta no fim
		if (SIS_GERIR.includes(cmd.acao)) return sisGerirEnfileirar(cmd);
		// 1.2.7: coletar claim stakes da hot wallet — fila própria, resposta no fim
		if (cmd.acao === 'claim_coletar') return sisClaimEnfileirar(cmd);
		const frota = cmd.alvo ? userFleets.find(f => f.publicKey.toString() === cmd.alvo) : null;
		switch (cmd.acao) {
			case 'iniciar':
				if (!initComplete) return responder(false, 'script ainda iniciando');
				if (enableAssistant) return responder(true, 'já estava rodando');
				toggleAssistant(); return responder(true, 'iniciado');
			case 'parar':
				if (!enableAssistant) return responder(true, 'já estava parado');
				toggleAssistant(); return responder(true, 'parando (termina as operações em andamento)');
			case 'parar_frota':
				if (!frota) return responder(false, 'frota não encontrada');
				if (await sisFrotaDoCerebro(cmd.alvo)) { sisCanceladas.add(cmd.alvo); updateFleetState(frota, 'Sis-Bot: tarefa parada pelo dashboard'); return responder(true, 'tarefa do Sis-Bot parada: ' + frota.label); }
				if (frota.stopping || String(frota.state).includes('STOPPED')) return responder(true, 'já parada');
				frota.stopping = true; updateFleetState(frota, 'Stopping ...'); return responder(true, 'parando ' + frota.label);
			case 'reiniciar_frota':
				if (!frota) return responder(false, 'frota não encontrada');
				if (sisCanceladas.has(cmd.alvo) || await sisFrotaDoCerebro(cmd.alvo)) {
					const estava = sisCanceladas.delete(cmd.alvo); if (estava) updateFleetState(frota, 'Sis-Bot: aguardando o Sis-Bot', true);
					return responder(true, estava ? 'tarefa do Sis-Bot iniciada de novo: ' + frota.label : 'a tarefa do Sis-Bot já estava rodando');
				}
				if (!((String(frota.state).includes('ERROR') && !String(frota.state).includes('⌛')) || String(frota.state).includes('STOPPED'))) return responder(false, 'a frota não está parada nem com erro');
				await resetFleetState(frota); return responder(true, 'reiniciando ' + frota.label);
			case 'desacoplar_todas':
				if (enableAssistant) return responder(false, 'pare o Sis-Bot antes de desacoplar');
				handleUndockAll(); return responder(true, 'desacoplando as frotas na starbase');
			case 'recarregar':
				responder(true, 'recarregando a página');
				await sisSyncAgora(); setTimeout(() => location.reload(), 500); return;
		}
		if (/^(frota_|craft_definir$|craft_coletar$)/.test(cmd.acao)) return sisExecutarOrdem(cmd);
		responder(false, 'comando desconhecido');
	}
	// ---------- executor das ordens do cérebro da VPS (o navegador só executa; não decide)
	const sisFilasFrota = new Map();
	// 1.2.2: frotas com a tarefa parada pelo dashboard: a ordem em espera é interrompida e as próximas são recusadas como
	// "interrompida" (o Sis-Bot guarda a rota e recalcula quando a tarefa for iniciada de novo)
	const sisCanceladas = new Set();
	const SIS_PARADA = 'interrompida: tarefa parada pelo dashboard';
	// [1.5.0] missões (botão Missão do dashboard): pk → id da missão das últimas ordens recebidas para a frota
	const sisEmMissao = new Map();
	// [1.5.5] missão Crafting: tripulação tirada da frota para o craft da missão (coords → {qtd, ate}); os trabalhos craftN
	// descontam esse número da tripulação livre da starbase até o craft da missão começar
	const sisCrewMissao = new Map();
	async function sisFrotaDoCerebro(pk) { if (sisEmMissao.has(pk)) return true; try { return !!JSON.parse(await GM.getValue(pk, '{}')).sisCerebro; } catch (e) { return false; } }
	// porão, tanque e compartimento de munição (antes e depois de cada ordem: o que entrou, saiu e foi gasto)
	async function sisFoto(pk) {
		const fl = userFleets.find((f) => f.publicKey.toString() === pk); if (!fl) return null;
		const ler = async (pod) => { const o = {}; (await solanaReadConnection.getParsedTokenAccountsByOwner(pod, { programId: tokenProgramPK })).value.forEach((t) => { const i = t.account.data.parsed.info; o[i.mint] = (o[i.mint] || 0) + (i.tokenAmount.uiAmount || 0); }); return o; };
		try {
			const [c, t, b] = await Promise.all([ler(fl.cargoHold), ler(fl.fuelTank), ler(fl.ammoBank)]);
			return { c, comb: t[sageGameAcct.account.mints.fuel.toString()] || 0, mun: b[sageGameAcct.account.mints.ammo.toString()] || 0 };
		} catch (e) { return null; }
	}
	// [1.5.7] combustível no tanque (uma leitura; null se o RPC não respondeu)
	async function sisLerTanque(fl) {
		const r = await solanaReadConnection.getParsedTokenAccountsByOwner(fl.fuelTank, { programId: tokenProgramPK });
		if (!r || !r.value) return null;
		const fm = sageGameAcct.account.mints.fuel.toString(); let s = 0;
		r.value.forEach((t) => { const i = t.account.data.parsed.info; if (i.mint === fm) s += i.tokenAmount.uiAmount || 0; });
		return s;
	}
	// [1.5.7] custo mais barato (warp ou subwarp) de "de" até "para" para esta frota
	function sisCustoChegar(fl, de, para) {
		const d = calculateMovementDistance(de, para); if (!(d > 0)) return 0;
		const sub = Math.ceil(calculateSubwarpFuelBurn(fl, d));
		const wp = fl.maxWarpDistance > 0 ? calcWarpFuelReq(fl, de, para, false) : 0;
		return wp > 0 ? Math.min(sub, wp) : sub;
	}
	function sisDiferenca(a, b) {
		if (!a || !b) return null;
		const c = {}; for (const m of new Set([...Object.keys(a.c), ...Object.keys(b.c)])) { const d = (b.c[m] || 0) - (a.c[m] || 0); if (Math.abs(d) >= 1e-6) c[m] = +d.toFixed(6); }
		return { carga: c, comb: +(b.comb - a.comb).toFixed(4), mun: +(b.mun - a.mun).toFixed(4) };
	}
	const SIS_MOVIMENTO = (m) => (+m === 1 ? 'subwarp' : (+m === 2 ? 'warpsubwarp' : (+m === 3 ? 'warp-subwarp-warp' : 'warp')));
	function sisNoPlano(cmd) {
		const pl = sisLocal.plano; const p = cmd.params || {};
		if (!pl) return 'nenhuma configuração aceita';
		// 1.2.3: coletar um craft concluído (de qualquer origem) numa starbase da sua facção; só move o produto para a sua conta na starbase
		if (cmd.acao === 'craft_coletar') {
			if (globalSettings.craftColetar === false) return 'coleta de crafts concluídos desligada na configuração aceita';
			if (!validTargets.some((t) => (t.x + ',' + t.y) === p.coords)) return 'starbase fora da sua facção';
			return /^[1-9A-HJ-NP-Za-km-z]{32,44}$/.test(String(p.processo || '')) ? '' : 'processo inválido';
		}
		if (cmd.acao === 'craft_definir') {
			if (!pl.craft.ativo) return 'crafting por regras não está ligado na configuração aceita';
			const n = parseInt(String(p.job || '').replace('craft', '')); if (!(n >= 1 && n <= pl.craft.jobs)) return 'trabalho de crafting fora da configuração';
			if (p.item && !pl.craft.receitas.includes(p.item)) return 'receita fora das regras aceitas';
			if (p.item && !pl.craft.coords.includes(p.coords)) return 'starbase fora das regras aceitas';
			if (p.item && (+p.crew > pl.craft.crewMax || !(+p.qtd > 0) || +p.qtd > 1e9)) return 'quantidade ou tripulação fora do limite';
			return '';
		}
		// [1.5.0] ordem de uma missão lançada no dashboard: vale sem aceite (como criar/desfazer frota; "bloquear comandos remotos"
		// também bloqueia). A frota precisa ser deste navegador; carga, tripulação, suprimentos e combustível só em starbases da sua
		// facção; mover, escanear e minerar em qualquer setor do mapa. Nada sai da conta do jogador.
		if (p.missao && /^frota_/.test(cmd.acao)) {
			if (!userFleets.some((x) => x.publicKey.toString() === cmd.alvo)) return 'frota não encontrada neste navegador';
			const sbOk = (c) => validTargets.some((t) => (t.x + ',' + t.y) === c);
			const setorOk = (c) => /^-?\d{1,2},-?\d{1,2}$/.test(String(c || '')) && String(c).split(',').every((v) => Math.abs(+v) <= 50);
			if (cmd.acao === 'frota_missao') return ['inicio', 'fim'].includes(p.fase) ? '' : 'fase da missão inválida';
			if (cmd.acao === 'frota_concluir') return '';
			if (['frota_mover', 'frota_scan', 'frota_minerar'].includes(cmd.acao)) return setorOk(p.coords) ? '' : 'setor fora do mapa';
			if (cmd.acao === 'frota_suprir' && !setorOk(p.mineCoords)) return 'setor fora do mapa';
			if (cmd.acao === 'frota_tripulantes' && !['carregar', 'descarregar'].includes(p.modo)) return 'ordem de tripulantes inválida';
			if (cmd.acao === 'frota_craftar' && p.iniciar && (typeof p.iniciar.item !== 'string' || !(+p.iniciar.qtd > 0) || +p.iniciar.qtd > 1e9 || !(+p.iniciar.crew > 0) || +p.iniciar.crew > 1e6)) return 'ordem de crafting inválida';
			if (!sbOk(p.coords)) return 'missão: starbase fora da sua facção';
			if (p.destino && !sbOk(p.destino)) return 'missão: destino fora da sua facção';
			return '';
		}
		const f0 = pl.frotas[cmd.alvo]; if (!f0) return 'esta frota não tem tarefa do Sis-Bot na configuração aceita';
		// [1.5.2] vale pela tarefa principal ou pela tarefa reserva aceita
		const r1 = sisChecarFrota(cmd, f0); if (!r1 || !f0.reserva) return r1;
		return sisChecarFrota(cmd, f0.reserva) ? r1 : '';
	}
	function sisChecarFrota(cmd, f) {
		const p = cmd.params || {};
		if (cmd.acao === 'frota_concluir') return ''; // só termina um movimento que já está em andamento
		// scan decidido pelo Sis-Bot: escanear e andar só dentro da área aceita (raio em volta do destino); base só em starbase da facção
		const naArea = (c) => { if (f.tarefa !== 'Prospectar' || !f.centro || !c) return false; const [ax, ay] = String(c).split(',').map(Number); const [cx, cy] = String(f.centro).split(',').map(Number); return Math.abs(ax - cx) <= f.raio && Math.abs(ay - cy) <= f.raio && Math.abs(ax) <= 50 && Math.abs(ay) <= 50; };
		if (cmd.acao === 'frota_scan') return naArea(p.coords) ? '' : 'scan fora da área aceita';
		if (cmd.acao === 'frota_mover' && naArea(p.coords)) return '';
		// garimpo: minerar e pôr suprimentos só nos pares (setor, recurso) das linhas aceitas; suprimentos só na starbase de entrega
		if (cmd.acao === 'frota_minerar') return (f.minerar || []).includes(`${p.coords}|${p.mint}`) ? '' : 'mineração fora das linhas aceitas';
		if (cmd.acao === 'frota_suprir') {
			if (!(f.minerar || []).includes(`${p.mineCoords}|${p.mint}`)) return 'mineração fora das linhas aceitas';
			return (f.entregas || []).includes(p.coords) ? '' : 'suprimentos só nas starbases de entrega aceitas';
		}
		// carregar: só nas starbases das rotas aceitas e só os itens delas. Mover, atracar, desatracar, reabastecer e
		// descarregar: também em qualquer starbase da sua facção (desvio para reabastecer ou retomar a rota de onde a frota parou)
		const daFaccao = (c) => f.coords.includes(c) || validTargets.some((t) => (t.x + ',' + t.y) === c);
		if (cmd.acao === 'frota_base_scan') return f.tarefa === 'Prospectar' && daFaccao(p.coords) ? '' : 'base do scan só numa starbase da sua facção';
		if (cmd.acao === 'frota_carregar') {
			if ((f.tarefa && f.tarefa !== 'Supply') || !f.coords.includes(p.coords)) return 'carregar só nas starbases das rotas aceitas';
			if ((p.itens || []).some((i) => !f.mints.includes(i.mint) || !(+i.qtd > 0 || i.tanque))) return 'item fora das rotas aceitas';
			return '';
		}
		if (p.coords && !daFaccao(p.coords)) return 'starbase fora da sua facção';
		if (p.destino && !daFaccao(p.destino)) return 'destino fora da sua facção';
		return '';
	}
	function sisExecutarOrdem(cmd) {
		const responder = (ok, msg, dados) => { sisFila.comandos.push({ id: cmd.id, ok, msg: String(msg).slice(0, 250), dados: dados || null }); sisSyncAgora(); };
		const fora = sisNoPlano(cmd);
		if (fora) { sisEvento('alerta', '', `Ordem recusada (${cmd.acao}): ${fora}`); return responder(false, 'recusada: ' + fora); }
		if (cmd.params && cmd.params.missao && cmd.alvo && /^frota_/.test(cmd.acao)) sisEmMissao.set(cmd.alvo, +cmd.params.missao);
		if (sisCanceladas.has(cmd.alvo)) return responder(false, SIS_PARADA);
		if (!enableAssistant) return responder(false, 'o Sis-Bot está parado neste navegador');
		const chave = /^craft_/.test(cmd.acao) ? 'craft' : cmd.alvo;
		const anterior = sisFilasFrota.get(chave) || Promise.resolve();
		const proxima = anterior.then(async () => {
			if (sisCanceladas.has(cmd.alvo)) return responder(false, SIS_PARADA);
			sisPainel.ordens.set(cmd.alvo, { acao: cmd.acao, rotulo: (cmd.params && cmd.params.rotulo) ? String(cmd.params.rotulo).slice(0, 120) : '', desde: Date.now() });
			try {
				// 1.2.2: mede o que a ordem mudou no porão/tanque/munição e a taxa paga (estatística da frota no dashboard)
				const frota = !/^craft_/.test(cmd.acao);
				// [1.4.1] foto com prazo (leitura presa não segura a ordem) e ordem com prazo (antes uma leitura presa travava a fila da frota para sempre)
				const foto = (pk) => sisComPrazo(sisFoto(pk), 45000, 'ler o porão').catch(() => null);
				const foto0 = frota ? await foto(cmd.alvo) : null; const tx0 = Object.assign({ lamports: 0, n: 0 }, sisTaxaFrota.get(cmd.alvo)); const t0 = Date.now();
				// [1.4.2] concluir a chegada espera a frota chegar (subwarp longo): mesmo prazo do mover
				const prazo = (cmd.acao === 'frota_minerar' || cmd.acao === 'frota_missao') ? 150 : (cmd.acao === 'frota_mover' || cmd.acao === 'frota_concluir') ? 120 : 25;
				const r = await sisComPrazo(sisExecutar(cmd), prazo * 60000, `executar ${cmd.acao}`).catch((e) => {
					try { const fl = frota ? userFleets.find((f) => f.publicKey.toString() === cmd.alvo) : null;
						if (fl && ['Mine Starting', 'Mining Stop', 'Unloading', 'Loading', 'Docking', 'Undocking', 'Refueling'].some((v) => String(fl.state || '').startsWith(v))) updateFleetState(fl, 'Sis-Bot: etapa sem resposta', true); } catch (x) { }
					return { ok: false, msg: 'falha: ' + e.message + ' (o Sis-Bot recalcula pela blockchain)' };
				});
				if (frota) {
					const tx1 = sisTaxaFrota.get(cmd.alvo) || { lamports: 0, n: 0 };
					r.dados = Object.assign({}, r.dados || {}, { delta: sisDiferenca(foto0, await foto(cmd.alvo)), taxa: { lamports: tx1.lamports - tx0.lamports, n: tx1.n - tx0.n }, ms: Date.now() - t0 });
				}
				responder(r.ok, r.msg, r.dados);
			}
			catch (e) { responder(false, 'falha: ' + (e && e.message ? e.message : e)); }
			finally { sisPainel.ordens.delete(cmd.alvo); }
		});
		sisFilasFrota.set(chave, proxima);
	}
	async function sisEstadoChain(fl) { const info = await getAccountInfo(fl.label, 'SIS ordem', fl.publicKey); return getFleetState(info, fl); }
	async function sisAtracadaEm(fl, coords) {
		const [st, ex] = await sisEstadoChain(fl);
		if (st !== 'StarbaseLoadingBay') return { st, ex, aqui: false };
		const [x, y] = ConvertCoords(coords); const sb = await getStarbaseFromCoords(x, y);
		return { st, ex, aqui: sb && ex && ex.starbase && ex.starbase.toString() === sb.publicKey.toString() };
	}
	// funções do SLYA que rodam dentro de "new Promise(async …)" nunca terminam se derem erro lá dentro: prazo para a ordem falhar em vez de travar a fila da frota
	function sisComPrazo(prom, ms, oque) {
		let t; return Promise.race([prom, new Promise((_, rej) => { t = setTimeout(() => rej(new Error(`sem resposta ao ${oque} em ${Math.round(ms / 60000)} min`)), ms); })]).finally(() => clearTimeout(t));
	}
	// recurso minerável no setor: o mineItem, o recurso do planeta (riqueza) e o planeta
	async function sisLocalMineracao(mint, coords) {
		const mineItem = mineItems.find((m) => m.account.mint.toString() === mint);
		const nome = (cargoItems.find((c) => c.token === mint) || {}).name || mint;
		if (!mineItem) return { erro: `${nome} não é minerável` };
		const [mx, my] = ConvertCoords(coords);
		for (const planet of await getPlanetsFromCoords(mx, my)) {
			const r = await getMineableResourceFromPlanet(planet.publicKey.toString(), mineItem.publicKey.toString());
			if (r && r.publicKey && r.account) return { mineItem, sageResource: r, planet, dureza: mineItem.account.resourceHardness, riqueza: r.account.systemRichness };
		}
		return { erro: `${nome} não existe para minerar em [${coords}]` };
	}
	async function sisLerCarga(fl) {
		const c = await solanaReadConnection.getParsedTokenAccountsByOwner(fl.cargoHold, { programId: tokenProgramPK });
		const out = {}; c.value.forEach((t) => { const i = t.account.data.parsed.info; if (i.tokenAmount.uiAmount) out[i.mint] = i.tokenAmount.uiAmount; });
		return out;
	}
	async function sisExecutar(cmd) {
		const p = cmd.params || {};
		if (cmd.acao === 'craft_definir') {
			const job = String(p.job); const atual = JSON.parse(await GM.getValue(job, '{}'));
			if (p.item) {
				if (atual.craftingId || (atual.state && !String(atual.state).startsWith('Idle') && atual.item)) return { ok: false, msg: `${job} ocupado (${atual.state || '?'})` };
				Object.assign(atual, { label: job, state: atual.state || 'Idle', item: p.item, coordinates: p.coords, amount: +p.qtd, crew: +p.crew, special: '', belowAmount: '', sisUnica: !!p.unica });
			} else Object.assign(atual, { label: job, item: '', amount: 0, sisUnica: false });
			await GM.setValue(job, JSON.stringify(atual));
			return { ok: true, msg: p.item ? `${job}: ${p.qtd} × ${p.item}` : `${job} liberado` };
		}
		if (cmd.acao === 'craft_coletar') return sisColetarCraft(p);
		const i = userFleets.findIndex((f) => f.publicKey.toString() === cmd.alvo);
		if (i < 0) return { ok: false, msg: 'frota não encontrada neste navegador' };
		const fl = userFleets[i];
		if (String(fl.state).includes('ERROR')) updateFleetState(fl, 'Idle', true);
		const erro = () => String(fl.state).includes('ERROR');
		const falhou = (msg) => { const m = String(fl.state).includes('ERROR') ? fl.state : msg; updateFleetState(fl, 'Sis-Bot: aguardando o Sis-Bot', true); return { ok: false, msg: m }; };
		const coords = p.coords || ''; const [x, y] = coords ? ConvertCoords(coords) : [null, null];
		if (p.movimento !== undefined) fl.moveType = SIS_MOVIMENTO(p.movimento);
		switch (cmd.acao) {
			case 'frota_mover': {
				// [1.5.7] não sai sem combustível para chegar ao destino desta ordem: com menos que o custo o jogo deixa sair e a frota
				// para no meio do caminho com o tanque vazio (caso [EC] Kram Log 18/11 em [1,-23]). Custo = o mais barato entre warp e
				// subwarp; só bloqueia se nenhuma de até 3 leituras (3 s entre elas) der o custo.
				{
					const [st0, ex0] = await sisEstadoChain(fl);
					if (st0 === 'Idle' && ex0 && !(+ex0[0] === x && +ex0[1] === y)) {
						const custo = sisCustoChegar(fl, [+ex0[0], +ex0[1]], [x, y]);
						let maior = null;
						for (let k = 0; k < 3; k++) {
							const v = await sisLerTanque(fl); if (v !== null) maior = maior === null ? v : Math.max(maior, v);
							if (maior === null || maior >= custo) break;
							await wait(3000);
						}
						if (maior !== null && maior < custo) {
							sisEvento('alerta', fl.label, `Sem combustível para chegar em [${coords}]: tem ${Math.floor(maior)}, precisa de ${custo}. A frota não saiu de [${ex0[0]},${ex0[1]}].`);
							return falhou(`combustível insuficiente: tem ${Math.floor(maior)}, precisa de ${custo} para chegar em [${coords}]; a frota não saiu de [${ex0[0]},${ex0[1]}]`);
						}
					}
				}
				for (let salto = 0; salto < 40; salto++) {
					const [st, ex] = await sisEstadoChain(fl);
					if (st === 'Idle' && ex && +ex[0] === x && +ex[1] === y) { updateFleetState(fl, 'Sis-Bot: chegou em [' + coords + ']'); return { ok: true, msg: 'chegou', dados: { coords, chegou: true } }; }
					if (st === 'StarbaseLoadingBay') return falhou('a frota está atracada; desatraque antes');
					if (st === 'MineAsteroid') return falhou('a frota está minerando');
					fl.moveTarget = coords;
					const d = st === 'Idle' && ex ? calculateMovementDistance(ex, [x, y]) : 1;
					await handleMovement(i, d, x, y, false);
					if (fl.exitWarpSubwarpPending) { if (fl.exitWarpSubwarpPending == 1) await execExitWarp(fl); else await execExitSubwarp(fl); fl.exitWarpSubwarpPending = 0; fl.exitSubwarpWillBurnFuel = 0; }
					if (erro()) return falhou('erro no movimento');
					if (!enableAssistant) return falhou('o Sis-Bot foi parado durante a viagem');
					if (sisCanceladas.has(cmd.alvo)) { updateFleetState(fl, 'Sis-Bot: tarefa parada pelo dashboard'); return { ok: false, msg: SIS_PARADA }; }
					if (p.umSalto) {
						// um salto por ordem: o Sis-Bot reavalia a rota em cada ponto do caminho antes de mandar o próximo
						const [st2, ex2] = await sisEstadoChain(fl);
						if (st2 === 'Idle' && ex2) {
							const chegou = +ex2[0] === x && +ex2[1] === y;
							updateFleetState(fl, chegou ? 'Sis-Bot: chegou em [' + coords + ']' : `Sis-Bot: salto até [${ex2[0]},${ex2[1]}]`);
							return { ok: true, msg: chegou ? 'chegou' : `salto concluído em [${ex2[0]},${ex2[1]}]`, dados: { coords: `${ex2[0]},${ex2[1]}`, chegou } };
						}
					}
				}
				return falhou('não chegou depois de 40 saltos');
			}
			case 'frota_concluir': {
				// a frota chegou (warp/subwarp terminado) mas o movimento não foi encerrado: encerra onde ela está
				if (fl.exitWarpSubwarpPending) { if (fl.exitWarpSubwarpPending == 1) await execExitWarp(fl); else await execExitSubwarp(fl); fl.exitWarpSubwarpPending = 0; fl.exitSubwarpWillBurnFuel = 0; }
				let [st, ex] = await sisEstadoChain(fl);
				if (st === 'MoveWarp' || st === 'MoveSubwarp' || st === 'ReadyToExitWarp') {
					const fim = st === 'MoveWarp' ? ex.warpFinish.toNumber() * 1000 : st === 'MoveSubwarp' ? ex.arrivalTime.toNumber() * 1000 : 0;
					if (fim > Date.now() + 5 * 60000) return falhou('a frota ainda está em viagem');
					while (fim > Date.now()) { if (sisCanceladas.has(cmd.alvo)) return { ok: false, msg: SIS_PARADA }; updateFleetState(fl, 'Move [' + TimeToStr(new Date(fim)) + ']'); await wait(Math.max(1000, Math.min(5000, fim - Date.now()))); }
					if (st === 'MoveSubwarp') await execExitSubwarp(fl); else await execExitWarp(fl);
					if (erro()) return falhou('erro ao concluir a chegada');
					await wait(2000);
					[st, ex] = await sisEstadoChain(fl);
				}
				if (st === 'Idle' && ex) { updateFleetState(fl, `Sis-Bot: parada em [${ex[0]},${ex[1]}]`); return { ok: true, msg: `parada em [${ex[0]},${ex[1]}]`, dados: { coords: `${ex[0]},${ex[1]}` } }; }
				if (st === 'StarbaseLoadingBay') return { ok: true, msg: 'atracada' };
				return falhou(`estado inesperado: ${st}`);
			}
			case 'frota_atracar': {
				const a = await sisAtracadaEm(fl, coords);
				if (a.aqui) return { ok: true, msg: 'já estava atracada' };
				if (a.st !== 'Idle' || !a.ex || +a.ex[0] !== x || +a.ex[1] !== y) return falhou(`a frota não está parada em [${coords}] (${a.st})`);
				await execDock(fl, coords);
				return erro() ? falhou('erro ao atracar') : { ok: true, msg: 'atracada' };
			}
			case 'frota_desatracar': {
				const a = await sisAtracadaEm(fl, coords);
				if (a.st === 'Idle') return { ok: true, msg: 'já estava fora da starbase' };
				if (!a.aqui) return falhou(`a frota não está atracada em [${coords}]`);
				await execUndock(fl, coords);
				return erro() ? falhou('erro ao desatracar') : { ok: true, msg: 'desatracada' };
			}
			case 'frota_carregar': {
				if (!(await sisAtracadaEm(fl, coords)).aqui) return falhou(`a frota não está atracada em [${coords}]`);
				if (p.explicito) return sisCarregarExplicito(fl, coords, p.itens || [], falhou, erro);
				const r = await handleTransportLoading(i, coords, (p.itens || []).map((it) => ({ res: it.mint, amt: +it.qtd })), false);
				const carga = await sisLerCarga(fl);
				if (!r.success || erro()) return falhou('não carregou');
				updateFleetState(fl, 'Sis-Bot: carregada');
				return { ok: true, msg: 'carregada', dados: { carga } };
			}
			case 'frota_descarregar': {
				if (!(await sisAtracadaEm(fl, coords)).aqui) return falhou(`a frota não está atracada em [${coords}]`);
				if (p.explicito) return sisDescarregarExplicito(fl, coords, p.itens || [], falhou, erro);
				const antes = await sisLerCarga(fl);
				await handleTransportUnloading(fl, coords, (p.itens || []).map((it) => ({ res: it.mint, amt: +it.qtd > 0 ? +it.qtd : 1e15, extra: +it.qtd > 0 })), false);
				const depois = await sisLerCarga(fl);
				if (erro()) return falhou('erro ao descarregar');
				const entregue = {}; for (const m in antes) { const d = antes[m] - (depois[m] || 0); if (d > 0) entregue[m] = d; }
				updateFleetState(fl, 'Sis-Bot: descarregada');
				return { ok: true, msg: 'descarregada', dados: { entregue } };
			}
			case 'frota_suprir': {
				// comida e munição para encher o porão minerando p.mint em p.mineCoords; combustível completo se faltar para ir e voltar
				if (!(await sisAtracadaEm(fl, coords)).aqui) return falhou(`a frota não está atracada em [${coords}]`);
				const loc = await sisLocalMineracao(p.mint, p.mineCoords); if (loc.erro) return falhou(loc.erro);
				const fuelMint = sageGameAcct.account.mints.fuel.toString(), foodMint = sageGameAcct.account.mints.food.toString(), ammoMint = sageGameAcct.account.mints.ammo.toString();
				const lerPod = async (pod) => (await solanaReadConnection.getParsedTokenAccountsByOwner(pod, { programId: tokenProgramPK })).value;
				const qtdDe = (l, m) => { const t = l.find((z) => z.account.data.parsed.info.mint === m); return t ? t.account.data.parsed.info.tokenAmount.uiAmount : 0; };
				const carga = await lerPod(fl.cargoHold); const tanque = await lerPod(fl.fuelTank); const banco = await lerPod(fl.ammoBank);
				const ocupado = carga.reduce((n, { account }) => n + account.data.parsed.info.tokenAmount.uiAmount, 0);
				const comida = qtdDe(carga, foodMint), comb = qtdDe(tanque, fuelMint), mun = qtdDe(banco, ammoMint);
				const dur = calculateMiningDuration(fl.cargoCapacity - ocupado + comida, fl.miningRate, loc.dureza, loc.riqueza);
				if (dur < 60) return falhou('o porão está cheio: descarregue antes de minerar');
				const precisaComida = Math.ceil(dur * (fl.foodConsumptionRate / 10000)) + (globalSettings.minerKeep1 ? 1 : 0);
				const precisaMun = Math.min(fl.ammoCapacity, Math.ceil(dur * (fl.ammoConsumptionRate / 10000)));
				const base = ConvertCoords(coords), alvo = ConvertCoords(p.mineCoords);
				let precisaComb = fl.planetExitFuelAmount || 0;
				if (!CoordsEqual(base, alvo)) {
					const sub = calculateSubwarpFuelBurn(fl, calculateMovementDistance(base, alvo)) * 2;
					const ws = fl.moveType == 'warpsubwarp';
					const warp = calcWarpFuelReq(fl, base, alvo, ws) + calcWarpFuelReq(fl, alvo, base, ws);
					precisaComb += fl.moveType == 'subwarp' || warp > fl.fuelCapacity ? sub : warp;
				}
				const faltou = []; const ct = (m) => cargoTypes.find((c) => c.account.mint.toString() == m);
				updateFleetState(fl, 'Loading');
				if (comb < precisaComb) {
					const t = tanque.find((z) => z.account.data.parsed.info.mint === fuelMint);
					const r = await execCargoFromStarbaseToFleet(fl, fl.fuelTank, t ? t.pubkey : fl.fuelToken, fuelMint, ct(fuelMint), coords, fl.fuelCapacity - comb, globalSettings.fleetForceConsumableAmount, false);
					if (r && r.name == 'NotEnoughResource') faltou.push('combustível');
				}
				if (mun < precisaMun) {
					const t = banco.find((z) => z.account.data.parsed.info.mint === ammoMint);
					const r = await execCargoFromStarbaseToFleet(fl, fl.ammoBank, t ? t.pubkey : await getFleetAmmoToken(fl), ammoMint, ct(ammoMint), coords, fl.ammoCapacity - mun, globalSettings.fleetForceConsumableAmount, false);
					if (r && r.name == 'NotEnoughResource') faltou.push('munição');
				}
				if (comida < precisaComida) {
					const t = carga.find((z) => z.account.data.parsed.info.mint === foodMint);
					const r = await execCargoFromStarbaseToFleet(fl, fl.cargoHold, t ? t.pubkey : fl.foodToken, foodMint, ct(foodMint), coords, precisaComida - comida, globalSettings.fleetForceConsumableAmount, false);
					if (r && r.name == 'NotEnoughResource') faltou.push('comida');
				}
				if (erro()) return falhou('erro ao pôr os suprimentos');
				if (faltou.length) return falhou(`falta ${faltou.join(', ')} na starbase [${coords}]`);
				updateFleetState(fl, 'Sis-Bot: suprimentos prontos');
				return { ok: true, msg: `suprimentos para ~${Math.round(dur / 60)} min de mineração`, dados: { duracao: dur, comida: precisaComida, municao: precisaMun, combustivel: precisaComb } };
			}
			case 'frota_minerar': {
				// começa a minerar (se ainda não estiver), espera encher o porão (ou acabar comida/munição) e para
				const loc = await sisLocalMineracao(p.mint, coords); if (loc.erro) return falhou(loc.erro);
				// execStartMining/execStopMining leem fl.destCoord (starbase do setor). Ele é refeito logo antes de cada uma:
				// aceitar uma configuração durante a mineração zera o destCoord das frotas do Sis-Bot (1.1.6 parava com erro "trim").
				const fixar = () => { fl.destCoord = coords; fl.mineResource = p.mint; };
				fixar();
				let [st, ex] = await sisEstadoChain(fl);
				if (st !== 'MineAsteroid') {
					if (st !== 'Idle' || !ex || +ex[0] !== x || +ex[1] !== y) return falhou(`a frota não está parada em [${coords}] (${st})`);
					fixar(); await sisComPrazo(execStartMining(fl, loc.mineItem, loc.sageResource, loc.planet), 180000, 'começar a minerar');
					if (erro()) return falhou('erro ao começar a minerar');
					for (let k = 0; k < 6 && st !== 'MineAsteroid'; k++) { await wait(10000); [st, ex] = await sisEstadoChain(fl); }
					if (st !== 'MineAsteroid') return falhou('a mineração não começou');
				}
				const foodMint = sageGameAcct.account.mints.food.toString(), ammoMint = sageGameAcct.account.mints.ammo.toString();
				const carga = (await solanaReadConnection.getParsedTokenAccountsByOwner(fl.cargoHold, { programId: tokenProgramPK })).value;
				const banco = (await solanaReadConnection.getParsedTokenAccountsByOwner(fl.ammoBank, { programId: tokenProgramPK })).value;
				const qtdDe = (l, m) => { const t = l.find((z) => z.account.data.parsed.info.mint === m); return t ? t.account.data.parsed.info.tokenAmount.uiAmount : 0; };
				const ocupado = carga.reduce((n, { account }) => n + account.data.parsed.info.tokenAmount.uiAmount, 0);
				const comida = qtdDe(carga, foodMint), mun = qtdDe(banco, ammoMint);
				// mesma conta do SLYA (handleMining, "Already mining"): os saldos só mudam na chain quando a mineração para
				const maxDur = calculateMiningDuration(fl.cargoCapacity, fl.miningRate, loc.dureza, loc.riqueza);
				const comidaMax = Math.ceil(maxDur * (fl.foodConsumptionRate / 10000));
				const decorrido = Date.now() / 1000 - ex.start.toNumber();
				const minerado = Math.ceil(decorrido * ((fl.miningRate / 10000) * (loc.riqueza / 100)) / (loc.dureza / 100));
				const tComida = fl.foodConsumptionRate > 0 ? Math.ceil((comida - Math.ceil(decorrido * (fl.foodConsumptionRate / 10000))) / (fl.foodConsumptionRate / 10000)) : maxDur;
				const tMun = fl.ammoConsumptionRate > 0 ? Math.ceil((mun - Math.ceil(decorrido * (fl.ammoConsumptionRate / 10000))) / (fl.ammoConsumptionRate / 10000)) : maxDur;
				const tCarga = calculateMiningDuration(fl.cargoCapacity - ocupado - minerado + comidaMax, fl.miningRate, loc.dureza, loc.riqueza);
				let fim = Date.now() + Math.max(0, Math.min(tComida, tMun, tCarga)) * 1000;
				// [1.5.0] missão por tempo ou por quantidade: para antes de encher o porão (maxSeg = segundos minerando; maxQtd = unidades)
				const inicioMin = ex.start.toNumber(); const taxaMin = ((fl.miningRate / 10000) * (loc.riqueza / 100)) / (loc.dureza / 100);
				if (+p.maxSeg > 0) fim = Math.min(fim, (inicioMin + +p.maxSeg) * 1000);
				if (+p.maxQtd > 0 && taxaMin > 0) fim = Math.min(fim, (inicioMin + Math.ceil(+p.maxQtd / taxaMin)) * 1000);
				fl.mineEnd = fim;
				while (Date.now() < fim) {
					if (!enableAssistant) { fl.mineEnd = 0; return falhou('o Sis-Bot foi parado durante a mineração (a frota continua minerando)'); }
					if (sisCanceladas.has(cmd.alvo)) { fl.mineEnd = 0; updateFleetState(fl, 'Sis-Bot: tarefa parada (a frota continua minerando)'); return { ok: false, msg: SIS_PARADA + ' (a frota continua minerando)' }; }
					updateFleetState(fl, 'Mine [' + TimeToStr(new Date(fim)) + ']');
					await wait(Math.max(1000, Math.min(30000, fim - Date.now())));
				}
				const resAcct = await sageProgram.account.resource.fetch(ex.resource);
				const mi = await sageProgram.account.mineItem.fetch(resAcct.mineItem);
				fixar(); await sisComPrazo(execStopMining(fl, ex.resource, resAcct, resAcct.mineItem, mi.mint), 180000, 'parar a mineração');
				const segMin = Math.max(0, Math.round(Date.now() / 1000 - inicioMin));
				fl.mineEnd = 0;
				if (erro()) return falhou('erro ao parar a mineração');
				await wait(3000);
				[st, ex] = await sisEstadoChain(fl);
				if (st === 'MineAsteroid') return falhou('a mineração não parou');
				const depois = await sisLerCarga(fl);
				updateFleetState(fl, 'Sis-Bot: mineração concluída');
				return { ok: true, msg: `minerado: ${Math.floor(depois[p.mint] || 0)} no porão`, dados: { coords, minerado: depois[p.mint] || 0, seg: segMin } };
			}
			case 'frota_scan': {
				// um scan no setor (espera a recarga pela chain) e devolve chance, SDU achados, porão, comida e combustível
				let [st, ex] = await sisEstadoChain(fl);
				if (st !== 'Idle' || !ex || +ex[0] !== x || +ex[1] !== y) return falhou(`a frota não está parada em [${coords}] (${st})`);
				await createScannerPDAs(fl);
				const conta = await sageProgram.account.fleet.fetch(fl.publicKey);
				const ate = (conta.scanCooldownExpiresAt ? Number(conta.scanCooldownExpiresAt.toString()) : 0) * 1000;
				// 1.2.1: com p.conferir {base, regen, min, antes}, confere a chance antes de escanear e não escaneia se ficou abaixo
				// do mínimo — o Sis-Bot escolhe outro setor. 1.3.6 (conferir.jogo): no fim da recarga simula a transação de scan
				// (sem assinar, nada vai para a rede) e lê a chance EXATA do jogo no log "SDU probability"; se a simulação não der,
				// usa a estimativa (base do mapa corrigida pelo último achado da conta Sector), como antes.
				const cf = p.conferir && +p.conferir.base > 0 ? p.conferir : null;
				const ateEspera = cf ? (cf.jogo ? ate + 300 : ate - Math.max(1, +cf.antes || 2) * 1000) : ate;
				while (ateEspera > Date.now()) {
					if (!enableAssistant) return falhou('o Sis-Bot foi parado esperando a recarga do scan');
					if (sisCanceladas.has(cmd.alvo)) return { ok: false, msg: SIS_PARADA };
					updateFleetState(fl, 'Waiting for scan cooldown [' + TimeToStr(new Date(ate)) + ']');
					await wait(Math.max(200, Math.min(15000, ateEspera - Date.now() + (cf ? 0 : 500))));
				}
				let conferido = null;
				if (cf && cf.jogo) {
					try {
						const lido = await sisChanceJogo(fl, x, y);
						if (lido && lido.chance !== null) conferido = { ts: Date.now(), chance: lido.chance, min: +cf.min || 0, fonte: 'jogo', tentativas: lido.tentativas };
						else cLog(2, 'Sis-Bot: simulação do scan não trouxe a chance (usa a estimativa)', lido && lido.erro);
					} catch (e) { cLog(2, 'Sis-Bot: simulação do scan falhou (usa a estimativa)', e); }
					if (conferido && conferido.chance < conferido.min) {
						updateFleetState(fl, `Sis-Bot: [${coords}] está em ${Math.round(conferido.chance * 100)}% no jogo — não escaneou`);
						const cg = await sisLerCarga(fl);
						return { ok: true, msg: `não escaneou: chance do jogo ${(conferido.chance * 100).toFixed(1)}% abaixo do mínimo ${(conferido.min * 100).toFixed(1)}%`, dados: { coords, ignorado: true, conferido, sduPorao: cg[sduItem.token] || 0, recargaAte: ate } };
					}
				}
				if (cf && !conferido) {
					try {
						const setor = await getSectorFromCoords(x, y);
						const sc = await sageProgram.account.sector.fetch(setor);
						const n = (v) => (v === undefined || v === null ? 0 : Number(v.toString()));
						const tAch = n(sc.lastScanTime); const cAch = n(sc.lastScanChance) / 10000; const regen = +cf.regen || 3600; const base = +cf.base;
						const dt = Date.now() / 1000 - tAch; const c0 = Math.min(cAch, base);
						const chance = tAch && dt < regen ? c0 + (base - c0) * Math.max(0, dt) / regen : base;
						conferido = { ts: Date.now(), chance, min: +cf.min || 0, fonte: 'estimativa', ultimoAchadoTs: tAch || null, ultimoAchadoChance: tAch ? cAch : null };
						if (chance < conferido.min) {
							updateFleetState(fl, `Sis-Bot: [${coords}] caiu para ${Math.round(chance * 100)}% — não escaneou`);
							const cg = await sisLerCarga(fl);
							return { ok: true, msg: `não escaneou: chance ${(chance * 100).toFixed(1)}% abaixo do mínimo ${(conferido.min * 100).toFixed(1)}%`, dados: { coords, ignorado: true, conferido, sduPorao: cg[sduItem.token] || 0, recargaAte: ate } };
						}
					} catch (e) { cLog(2, 'Sis-Bot: conferência do setor antes do scan falhou', e); }
					while (ate > Date.now()) await wait(Math.max(100, ate - Date.now() + 300));
				}
				const sduMint = sduItem.token, foodMint = sageGameAcct.account.mints.food.toString(), fuelMint = sageGameAcct.account.mints.fuel.toString();
				const antes = await sisLerCarga(fl);
				if (fl.scanCost > 0 && (antes[foodMint] || 0) < fl.scanCost) return falhou('sem comida para escanear (volte à base)');
				const res = await sisComPrazo(execScan(fl, [x, y]), 180000, 'escanear');
				if (!res || !res.meta || erro()) return falhou('erro no scan');
				const log = (res.meta.logMessages || []).find((l) => l.startsWith('Program log: SDU probability:'));
				const chance = log ? Number(log.split(' ').pop()) : null;
				const dSdu = getBalanceChange(res, fl.sduToken.toString()); const achou = Math.max(0, dSdu.postBalance - dSdu.preBalance);
				const dFood = fl.scanCost > 0 ? getBalanceChange(res, fl.foodToken.toString()) : { preBalance: 0, postBalance: 0 }; const gasto = Math.max(0, dFood.preBalance - dFood.postBalance);
				fl.sduCnt = dSdu.postBalance; fl.scanEnd = Date.now() + fl.scanCooldown * 1000;
				sisProducao('sdu', fl.label, sduItem.name, achou); if (gasto) sisProducao('consumo', fl.label, foodItem && foodItem.name, gasto);
				try { await alterStats('SDUs found', undefined, achou, 'SDU', 0); } catch (e) {}
				const carga = await sisLerCarga(fl);
				const ocupado = Object.entries(carga).reduce((n, [m, q]) => n + q * ((cargoItems.find((c) => c.token === m) || {}).size || 1), 0);
				const tanque = (await solanaReadConnection.getParsedTokenAccountsByOwner(fl.fuelTank, { programId: tokenProgramPK })).value.find((t) => t.account.data.parsed.info.mint === fuelMint);
				const comb = tanque ? tanque.account.data.parsed.info.tokenAmount.uiAmount : 0; fl.fuelCnt = comb;
				updateFleetState(fl, `Scanned [${chance !== null ? Math.round(chance * 100) : '?'}%]${achou ? ` +${achou}` : ''}`);
				return { ok: true, msg: `chance ${chance !== null ? (chance * 100).toFixed(1) + '%' : '?'}${achou ? `, achou ${achou} SDU` : ''}`,
					dados: { coords, chance, sdu: achou, conferido, comida: carga[foodMint] || 0, sduPorao: carga[sduMint] || 0, livre: Math.max(0, fl.cargoCapacity - ocupado), combustivel: comb, recargaAte: fl.scanEnd, sduPorScan: fl.sduPerScan, custo: fl.scanCost, recarga: fl.scanCooldown } };
			}
			case 'frota_base_scan': {
				// atracada: descarrega o SDU (deixa 1 para a conta do token), põe comida no porão (frota que gasta comida) e enche o tanque
				if (!(await sisAtracadaEm(fl, coords)).aqui) return falhou(`a frota não está atracada em [${coords}]`);
				const sduMint = sduItem.token, foodMint = sageGameAcct.account.mints.food.toString(), fuelMint = sageGameAcct.account.mints.fuel.toString();
				const ct = (m) => cargoTypes.find((c) => c.account.mint.toString() == m);
				const carga = await sisLerCarga(fl); const nSdu = Math.floor(carga[sduMint] || 0); let entregue = 0;
				if (nSdu > 1) { updateFleetState(fl, 'Unloading'); await execCargoFromFleetToStarbase(fl, fl.cargoHold, sduMint, coords, nSdu - 1); entregue = nSdu - 1; fl.sduCnt = 1; }
				if (erro()) return falhou('erro ao descarregar o SDU');
				if (entregue) sisProducao('descarregado', fl.label, sduItem.name, entregue);
				updateFleetState(fl, 'Loading'); const faltou = [];
				if (fl.scanCost > 0) {
					const c2 = await sisLerCarga(fl);
					const ocupado = Object.entries(c2).reduce((n, [m, q]) => n + q * ((cargoItems.find((c) => c.token === m) || {}).size || 1), 0);
					const quer = Math.floor(fl.cargoCapacity - ocupado);
					if (quer > 0) { const r = await execCargoFromStarbaseToFleet(fl, fl.cargoHold, fl.foodToken, foodMint, ct(foodMint), coords, quer, false, false); if (r && r.name == 'NotEnoughResource') faltou.push('comida'); }
				}
				const tq = (await solanaReadConnection.getParsedTokenAccountsByOwner(fl.fuelTank, { programId: tokenProgramPK })).value.find((t) => t.account.data.parsed.info.mint === fuelMint);
				const comb = tq ? tq.account.data.parsed.info.tokenAmount.uiAmount : 0;
				if (comb < fl.fuelCapacity) { const r = await execCargoFromStarbaseToFleet(fl, fl.fuelTank, tq ? tq.pubkey : fl.fuelToken, fuelMint, ct(fuelMint), coords, fl.fuelCapacity - comb, false, false); if (r && r.name == 'NotEnoughResource') faltou.push('combustível'); }
				if (erro()) return falhou('erro ao pôr comida/combustível');
				await createScannerPDAs(fl);
				const depois = await sisLerCarga(fl);
				if (fl.scanCost > 0 && (depois[foodMint] || 0) < fl.scanCost) return falhou(`falta comida na starbase [${coords}]`);
				updateFleetState(fl, 'Sis-Bot: pronta para escanear');
				return { ok: true, msg: `SDU entregue: ${entregue}${faltou.length ? ` · faltou ${faltou.join(', ')} (carregou o que tinha)` : ''}`, dados: { entregue, comida: depois[foodMint] || 0 } };
			}
			case 'frota_tripulantes': {
				// [1.5.0] missão Transporte de Tripulantes: embarca (carregar) ou desembarca (descarregar) passageiros na starbase
				if (!(await sisAtracadaEm(fl, coords)).aqui) return falhou(`a frota não está atracada em [${coords}]`);
				const lerCrew = async () => { updateFleetMiscStats(fl, await getAccountInfo(fl.label, 'SIS ordem', fl.publicKey)); return { n: +fl.crewCount || 0, req: +fl.requiredCrew || 0, cap: +fl.passengerCapacity || 0 }; };
				const c0 = await lerCrew(); const aBordo0 = Math.max(0, c0.n - c0.req); const quer = +p.qtd > 0 ? +p.qtd : Infinity;
				if (p.modo === 'descarregar') {
					const n = Math.min(quer, aBordo0);
					if (!(n > 0)) return { ok: true, msg: 'nenhum passageiro a bordo', dados: { qtd: 0, aBordo: 0 } };
					await handleCrewUnloading(fl, coords, n, false);
				} else {
					const n = Math.min(quer, Math.max(0, c0.req + c0.cap - c0.n));
					if (!(n > 0)) return { ok: true, msg: 'sem lugar para passageiros', dados: { qtd: 0, aBordo: aBordo0, cheia: true } };
					const r = await handleCrewLoading(fl, coords, n, false);
					if (r && r.name === 'NotEnoughCrew') { updateFleetState(fl, 'Sis-Bot: sem tripulantes livres na starbase'); return { ok: true, msg: 'nenhum tripulante livre na starbase', dados: { qtd: 0, aBordo: aBordo0 } }; }
				}
				if (erro()) return falhou(p.modo === 'descarregar' ? 'erro ao desembarcar os tripulantes' : 'erro ao embarcar os tripulantes');
				let c1 = c0; for (let k = 0; k < 5; k++) { await wait(2500); c1 = await lerCrew(); if (c1.n !== c0.n) break; }
				const qtd = Math.abs(c1.n - c0.n);
				if (!qtd) return falhou('a tripulação da frota não mudou (a transação não entrou?)');
				updateFleetState(fl, p.modo === 'descarregar' ? 'Sis-Bot: tripulantes desembarcados' : 'Sis-Bot: tripulantes a bordo');
				return { ok: true, msg: `${qtd} tripulantes ${p.modo === 'descarregar' ? 'desembarcados' : 'embarcados'}`, dados: { qtd, aBordo: Math.max(0, c1.n - c1.req) } };
			}
			case 'frota_craftar': {
				// [1.5.5] missão Crafting, na starbase onde a frota está atracada, nesta ordem (cada parte é opcional):
				//   coletar   = conclui o craft anterior da missão (processo); se ainda não terminou, responde pendente
				//   desembarcar = tira a tripulação da frota (toda: pilotos/tripulação mínima das naves + passageiros)
				//   iniciar   = começa {item, qtd, crew} com a tripulação livre (no máximo crew), só com a receita pedida (sem encadear)
				//   embarcar  = devolve N tripulantes à frota (fim da fila ou troca de starbase)
				// Entre desembarcar e o craft começar, os trabalhos craftN não usam essa tripulação (sisCrewMissao).
				if (!(await sisAtracadaEm(fl, coords)).aqui) return falhou(`a frota não está atracada em [${coords}]`);
				const out = { coletado: null, pendente: false, desembarcados: 0, iniciado: null, faltaMaterial: false, semCrew: false, semAtlas: false, embarcados: 0 };
				const lerCrew = async () => { updateFleetMiscStats(fl, await getAccountInfo(fl.label, 'SIS ordem', fl.publicKey)); return { n: +fl.crewCount || 0, req: +fl.requiredCrew || 0, cap: +fl.passengerCapacity || 0 }; };
				const starbase = await getStarbaseFromCoords(x, y, true); if (!starbase) return falhou(`starbase não encontrada em [${coords}]`);
				const sbp0 = await getStarbasePlayer(userProfileAcct, starbase.publicKey);
				const sbpPk = sbp0 ? sbp0.publicKey : await execRegisterStarbasePlayer(fl, coords);
				const crewLivre = async () => { const i = await sageProgram.account.starbasePlayer.fetch(sbpPk); return i.totalCrew - i.busyCrew.toNumber(); };
				const guarda = { qtd: 0, ate: Date.now() + 10 * 60000 };
				const soltar = () => { if (sisCrewMissao.get(coords) === guarda) sisCrewMissao.delete(coords); };
				try {
					// [1.5.6] a guarda vale desde antes da coleta: ao concluir o craft a tripulação fica livre e um trabalho craftN em
					// "Waiting for crew" a pegava antes de o próximo craft da missão começar (17/11: Fuel da regra 3 levou 90 de 145)
					guarda.qtd = Math.max(+((p.iniciar && p.iniciar.crew) || 0), +p.embarcar || 0);
					if (guarda.qtd > 0) sisCrewMissao.set(coords, guarda);
					if (p.coletar) {
						updateFleetState(fl, 'Sis-Bot: coletando o craft da missão');
						const r = await sisColetarCraft({ processo: String(p.coletar), coords });
						if (!r.ok) { if (/ainda não/.test(r.msg || '')) { out.pendente = true; updateFleetState(fl, 'Sis-Bot: esperando o craft da missão'); return { ok: true, msg: 'o craft da missão ainda não terminou', dados: out }; } return falhou('coletar o craft: ' + r.msg); }
						out.coletado = r.dados || {};
					}
					// [1.5.8] receita (craft ou upgrade) e ingredientes conferidos ANTES de tirar a tripulação: em 21/11 a receita de upgrade
					// "Electronics Upgrade" não era achada (só craftRecipes) e a frota desembarcava/embarcava 145 tripulantes a cada 4 min
					const rcIni = p.iniciar && p.iniciar.item ? (craftRecipes.find((r) => r.name === p.iniciar.item) || upgradeRecipes.find((r) => r.name === p.iniciar.item)) : null;
					const ehUp = !!rcIni && !craftRecipes.some((r) => r.name === rcIni.name);
					const daMat = (holds) => { let da = Infinity; for (const inp of rcIni.input) { const h = getStarbasePlayerCargoMaxItem(holds, inp.mint.toString());
						const tk = h && h.cargoHoldTokens ? h.cargoHoldTokens.find((t) => t.mint === inp.mint.toString()) : null; da = Math.min(da, Math.floor(((tk && tk.amount) || 0) / inp.amount)); } return da; };
					let vaiIniciar = !!(p.iniciar && p.iniciar.item);
					if (vaiIniciar) {
						if (!rcIni) return falhou(`receita "${p.iniciar.item}" não encontrada`);
						if (ehUp && starbase.account.level >= 5) { out.nivelMax = true; vaiIniciar = false; }
						else if (!(daMat(await getStarbasePlayerCargoHolds(sbpPk)) >= 1)) { out.faltaMaterial = true; vaiIniciar = false; }
					}
					// tripulação desembarcada nesta ordem volta para a frota se o craft não começar
					const devolver = async () => { if (!(out.desembarcados > 0)) return; try { updateFleetState(fl, 'Sis-Bot: devolvendo a tripulação', true); await handleCrewLoading(fl, coords, out.desembarcados, false); out.devolvidos = out.desembarcados; } catch (e) { /* o cérebro completa a tripulação */ } };
					if (p.desembarcar && vaiIniciar) {
						const c0 = await lerCrew();
						if (c0.n > 0) {
							updateFleetState(fl, 'Sis-Bot: desembarcando a tripulação');
							await handleCrewUnloading(fl, coords, c0.n, false);
							if (erro()) {
								// o jogo não deixou tirar a tripulação mínima das naves: tira só os passageiros
								updateFleetState(fl, 'Sis-Bot: desembarcando os passageiros', true);
								const pass = Math.max(0, c0.n - c0.req);
								if (pass > 0) await handleCrewUnloading(fl, coords, pass, false);
								if (erro()) return falhou('erro ao desembarcar a tripulação');
							}
							let c1 = c0; for (let k = 0; k < 6; k++) { await wait(2500); c1 = await lerCrew(); if (c1.n !== c0.n) break; }
							out.desembarcados = Math.max(0, c0.n - c1.n);
							out.ficaram = c1.n;
						}
					}
					if (vaiIniciar) {
						const rc = rcIni;
						const holds = await getStarbasePlayerCargoHolds(sbpPk);
						const da = daMat(holds);
						const qtd = Math.floor(Math.min(+p.iniciar.qtd, da));
						const livre = await crewLivre(); const crew = Math.floor(Math.min(+p.iniciar.crew, livre, p.desembarcar ? out.desembarcados : Infinity)); // só a tripulação da frota
						out.crewLivre = livre; out.daMaterial = Number.isFinite(da) ? da : 0;
						if (!(qtd >= 1)) out.faltaMaterial = true;
						else if (!(crew >= 1)) out.semCrew = true;
						else {
							const atlasPk = new solanaWeb3.PublicKey('ATLASXmbPQxBUYbxPsV97usA3fPQYEqzQBUHgiFCUsXx');
							const ab = await solanaReadConnection.getParsedTokenAccountsByOwner(userPublicKey, { mint: atlasPk });
							const temAtlas = ab.value[0] ? ab.value[0].account.data.parsed.info.tokenAmount.uiAmount : 0;
							if (!ehUp && temAtlas < Number((qtd * (rc.feeAmount || 0)).toFixed(10))) out.semAtlas = true; // upgrade não cobra ATLAS
							else {
								const uc = { label: 'Sis-Bot: missão crafting', sisColeta: true, state: 'Idle', item: rc.name, coordinates: coords, craftingCoords: coords, amount: qtd, crew, special: '', belowAmount: '' };
								updateFleetState(fl, `Sis-Bot: começando ${qtd} × ${rc.name}`);
								const res = await sisComPrazo(execStartCrafting(starbase, sbpPk, holds, rc, qtd, uc), 300000, 'começar o craft');
								if (String(uc.state).includes('ERROR') || !res || !res.craftingId) { const m = 'começar o craft: ' + (uc.state || (res && res.name) || 'falhou'); updateFleetState(fl, 'Sis-Bot: missão crafting', true); await devolver(); return falhou(m); }
								// processo criado (pelo craftingId) e a hora em que termina no relógio da starbase
								let proc = null;
								for (let k = 0; k < 6 && !proc; k++) {
									if (k) await wait(3000);
									const insts = await sageProgram.account.craftingInstance.all([{ memcmp: { offset: 11, bytes: sbpPk.toBase58() } }]);
									for (const ci of insts) {
										const ps = await craftingProgram.account.craftingProcess.all([{ memcmp: { offset: 17, bytes: ci.publicKey.toBase58() } }]);
										const pr = ps.find((z) => z.account.craftingId.toNumber() == res.craftingId); if (pr) { proc = pr; break; }
									}
								}
								const VEL = [0, .2, .275, .35, .425, .5, .5];
								let fim = Date.now() + (rc.duration * qtd / crew) * 1000;
								if (proc) { const t = await getStarbaseTime(starbase, ehUp ? 'Upgrade' : 'Craft'); const rest = Math.max(0, proc.account.endTime.toNumber() - t.starbaseTime); fim = Date.now() + (t.resRemaining > 0 ? rest : rest / (VEL[starbase.account.level] || 1)) * 1000; }
								out.iniciado = { receita: rc.name, qtd, crew, craftingId: res.craftingId, processo: proc ? proc.publicKey.toString() : '', fim: Math.round(fim), atlas: res.feeAtlas || 0, upgrade: ehUp };
								updateFleetState(fl, `Sis-Bot: craftando ${qtd} × ${rc.name} [${TimeToStr(new Date(fim))}]`);
							}
						}
						if (!out.iniciado && !(+p.embarcar > 0)) await devolver();
					}
				} finally { if (!(+p.embarcar > 0)) soltar(); }
				if (+p.embarcar > 0) try {
					const c0 = await lerCrew(); const n = Math.floor(Math.min(+p.embarcar, Math.max(0, c0.req + c0.cap - c0.n)));
					if (n > 0) {
						updateFleetState(fl, 'Sis-Bot: embarcando a tripulação');
						const r = await handleCrewLoading(fl, coords, n, false);
						if (!(r && r.name === 'NotEnoughCrew')) {
							if (erro()) return falhou('erro ao embarcar a tripulação');
							let c1 = c0; for (let k = 0; k < 6; k++) { await wait(2500); c1 = await lerCrew(); if (c1.n !== c0.n) break; }
							out.embarcados = Math.max(0, c1.n - c0.n);
						}
					}
					const c2 = await lerCrew(); out.aBordo = c2.n; out.minimo = c2.req;
				} finally { soltar(); }
				const partes = [out.coletado ? `coletado ${out.coletado.quantidade || '?'} × ${out.coletado.receita || ''}` : '', out.desembarcados ? `${out.desembarcados} tripulantes desembarcados` : '',
					out.iniciado ? `começou ${out.iniciado.qtd} × ${out.iniciado.receita} com ${out.iniciado.crew} tripulantes` : '', out.faltaMaterial ? 'faltam ingredientes na starbase' : '', out.semCrew ? 'sem tripulação livre na starbase' : '',
					out.semAtlas ? 'sem ATLAS para a taxa do craft' : '', out.nivelMax ? 'starbase no nível máximo (sem upgrade)' : '', out.devolvidos ? `${out.devolvidos} tripulantes devolvidos à frota` : '', out.embarcados ? `${out.embarcados} tripulantes embarcados` : ''].filter(Boolean);
				if (!out.iniciado) updateFleetState(fl, 'Sis-Bot: missão crafting');
				return { ok: true, msg: partes.join(' · ') || 'nada a fazer', dados: out };
			}
			case 'frota_missao': {
				// [1.5.0] início/fim de uma missão. Frota com tarefa do motor local (Minerar, Transportar, Scan): no início a tarefa é
				// parada (como o ⏸ do dashboard) e guardada; no fim ela volta. Frotas do Sis-Bot ou sem tarefa: só marca.
				const g = JSON.parse(await GM.getValue(cmd.alvo, '{}')) || {};
				if (p.fase === 'inicio') {
					if (g.assignment && !g.sisCerebro) {
						const parou = () => { const st = String(fl.state || ''); return st.includes('STOPPED') || (st.includes('ERROR') && !st.includes('⌛')); };
						if (!parou()) { fl.stopping = true; updateFleetState(fl, 'Stopping ...'); }
						const t0 = Date.now();
						while (!parou()) {
							if (!enableAssistant) return falhou('o Sis-Bot foi parado antes de a tarefa da frota terminar');
							if (sisCanceladas.has(cmd.alvo)) return { ok: false, msg: SIS_PARADA };
							if (Date.now() - t0 > 140 * 60000) return falhou('a tarefa atual da frota não parou em 140 min');
							await wait(3000);
						}
						g.sisMissaoDe = { assignment: g.assignment }; g.assignment = '';
						await GM.setValue(cmd.alvo, JSON.stringify(g));
						sisPainel.tarefas.delete(cmd.alvo);
					}
					updateFleetState(fl, 'Sis-Bot: em missão', true);
					return { ok: true, msg: 'missão iniciada', dados: { suspensa: (g.sisMissaoDe && g.sisMissaoDe.assignment) || '' } };
				}
				sisEmMissao.delete(cmd.alvo);
				let voltou = '';
				if (g.sisMissaoDe) {
					voltou = g.sisMissaoDe.assignment || '';
					if (voltou && !g.assignment && !g.sisCerebro) g.assignment = voltou;
					delete g.sisMissaoDe;
					await GM.setValue(cmd.alvo, JSON.stringify(g));
					sisPainel.tarefas.delete(cmd.alvo);
				}
				// de volta à tarefa do motor local: recomeça como o ▶ do dashboard; tarefa do Sis-Bot ou sem tarefa: fica esperando
				if (g.assignment && !g.sisCerebro) { updateFleetState(fl, 'STOPPED', true); await resetFleetState(fl); }
				else updateFleetState(fl, g.sisCerebro ? 'Sis-Bot: aguardando o Sis-Bot' : 'Idle', true);
				return { ok: true, msg: 'missão terminada', dados: { voltou } };
			}
			case 'frota_reabastecer': {
				if (!(await sisAtracadaEm(fl, coords)).aqui) return falhou(`a frota não está atracada em [${coords}]`);
				const de = ConvertCoords(coords), para = ConvertCoords(p.destino), idaVolta = p.idaVolta !== false;
				// [1.5.7] leitura conferida: o Sis-Bot manda o combustível que espera no tanque (combEsperado). Leitura acima disso = lê de
				// novo até 3 vezes (3 s entre elas) e fica com a menor; o reabastecer do SLYA não passa dela (sisTetoComb).
				const esp = p.combEsperado !== undefined && p.combEsperado !== null && Number.isFinite(+p.combEsperado) ? +p.combEsperado : null;
				const tol = Math.max(10, (+fl.fuelCapacity || 0) * 0.005);
				const leituras = []; const v0 = await sisLerTanque(fl); if (v0 !== null) leituras.push(v0);
				for (let k = 0; k < 3 && esp !== null && (!leituras.length || Math.min(...leituras) > esp + tol); k++) {
					await wait(3000); const v = await sisLerTanque(fl); if (v !== null) leituras.push(v);
				}
				const menor = leituras.length ? Math.min(...leituras) : null;
				const reab = async (teto) => {
					if (teto !== null) fl.sisTetoComb = teto;
					try { return await handleTransportRefueling(fl, coords, de, para, idaVolta, 0, [], false); } finally { delete fl.sisTetoComb; }
				};
				let r = await reab(menor);
				// leituras diferentes e a transação falhou (ex.: a menor era antiga e pediu mais do que cabe): tenta uma vez sem o teto
				if (((r && String(r.detail || '').includes('ERROR')) || erro()) && leituras.length > 1 && Math.max(...leituras) - menor > tol) {
					updateFleetState(fl, 'Sis-Bot: reabastecendo de novo', true); r = await reab(null);
				}
				if ((r && String(r.detail || '').includes('ERROR')) || erro()) return falhou(r && r.detail ? r.detail : 'erro ao reabastecer');
				// confere depois: o tanque tem que dar a viagem pedida (ida e volta ou só a ida). A transação pode demorar a aparecer
				// na leitura: até 4 leituras, 2,5 s entre elas.
				const fd = await getFleetFuelData(fl, de, para, idaVolta); const precisa = Math.min(+fd.fuelNeeded || 0, +fd.capacity || Infinity);
				let tanque = null;
				for (let k = 0; k < 4; k++) {
					if (k) await wait(2500);
					const v = await sisLerTanque(fl); if (v !== null) tanque = v;
					if (tanque !== null && tanque >= precisa) break;
				}
				const dados = { combustivel: r ? r.amount : null, tanque, precisa, lido: v0, menor, leituras: leituras.length, esperado: esp };
				if (tanque !== null && tanque < precisa) {
					sisEvento('alerta', fl.label, `Depois de reabastecer o tanque tem ${Math.floor(tanque)} e a viagem precisa de ${Math.ceil(precisa)}; a frota fica atracada.`);
					const f2 = falhou(`depois de reabastecer o tanque tem ${Math.floor(tanque)} e ${idaVolta ? 'a ida e volta' : 'a ida'} precisa de ${Math.ceil(precisa)}`);
					f2.dados = dados; return f2;
				}
				if (esp !== null && v0 !== null && v0 > esp + tol) sisEvento('info', fl.label, `Reabastecer: a 1ª leitura do tanque (${Math.floor(v0)}) não batia com o esperado pelo Sis-Bot (${Math.floor(esp)}); usou ${Math.floor(menor)} depois de ${leituras.length} leituras.`);
				return { ok: true, msg: 'reabastecida', dados };
			}
		}
		return { ok: false, msg: 'ação desconhecida' };
	}
	// [1.3.6] chance exata do jogo no setor (x, y) para esta frota: simula a transação de scan (mesmas contas do execScan,
	// sem assinatura e sem enviar) e lê "Program log: SDU probability: X" do resultado. Durante a recarga (ou se o relógio da
	// rede ainda não passou do fim dela) o jogo dá erro ou chance 0: tenta de novo a cada 1,5 s, até 6 vezes.
	// Devolve { chance (0–1) | null, tentativas, erro }.
	async function sisChanceJogo(fl, x, y) {
		const setor = await getSectorFromCoords(x, y);
		const sduCT = cargoTypes.find((c) => c.account.mint.toString() == sduItem.token);
		const foodCT = cargoTypes.find((c) => c.account.mint.toString() == sageGameAcct.account.mints.food);
		if (!sduCT || !foodCT || !fl.sduToken || !fl.foodToken) return { chance: null, tentativas: 0, erro: 'contas do scan incompletas' };
		const [sduTokenFrom] = BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync([sageSDUTrackerAcct.account.signer.toBuffer(), tokenProgramPK.toBuffer(), new solanaWeb3.PublicKey(sduItem.token).toBuffer()], programPK);
		const ix = await sageProgram.methods.scanForSurveyDataUnits({ keyIndex: new BrowserAnchor.anchor.BN(userProfileKeyIdx) }).accountsStrict({
			gameAccountsFleetAndOwner: { gameFleetAndOwner: { fleetAndOwner: { fleet: fl.publicKey, owningProfile: userProfileAcct, owningProfileFaction: userProfileFactionAcct.publicKey, key: userPublicKey }, gameId: sageGameAcct.publicKey }, gameState: sageGameAcct.account.gameState },
			sector: setor, surveyDataUnitTracker: sageSDUTrackerAcct.publicKey, surveyDataUnitTrackerSigner: sageSDUTrackerAcct.account.signer,
			cargoHold: fl.cargoHold, sduCargoType: sduCT.publicKey, resourceCargoType: foodCT.publicKey, cargoStatsDefinition: sageGameAcct.account.cargo.statsDefinition,
			sduTokenFrom, sduTokenTo: fl.sduToken, resourceTokenFrom: fl.foodToken, resourceMint: sageGameAcct.account.mints.food,
			dataRunningXpAccounts: userXpAccounts.userDataRunningXpAccounts, councilRankXpAccounts: userXpAccounts.userCouncilRankXpAccounts,
			progressionConfig: progressionConfigAcct, pointsProgram: pointsProgramId, cargoProgram: cargoProgramPK, tokenProgram: tokenProgramPK,
			recentSlothashes: new solanaWeb3.PublicKey('SysvarS1otHashes111111111111111111111111111'), instructionsSysvar: new solanaWeb3.PublicKey('Sysvar1nstructions1111111111111111111111111'),
		}).instruction();
		const instructions = [solanaWeb3.ComputeBudgetProgram.setComputeUnitLimit({ units: 1400000 }), ix];
		let ultimoErro = null;
		for (let k = 1; k <= 6; k++) {
			try {
				const msg = new solanaWeb3.TransactionMessage({ payerKey: userPublicKey, recentBlockhash: '11111111111111111111111111111111', instructions }).compileToV0Message(addressLookupTables);
				const sim = await solanaReadConnection.simulateTransaction(new solanaWeb3.VersionedTransaction(msg), { sigVerify: false, replaceRecentBlockhash: true, commitment: 'processed' });
				const v = sim && sim.value;
				const l = v && (v.logs || []).find((t) => t.startsWith('Program log: SDU probability:'));
				const c = l ? Number(l.split(' ').pop()) : NaN;
				// chance só vale de simulação sem erro; 0 nas 3 primeiras tentativas = provavelmente ainda na recarga pelo relógio da rede
				if (v && !v.err && Number.isFinite(c) && (c > 0 || k > 3)) return { chance: c, tentativas: k };
				ultimoErro = v && v.err ? JSON.stringify(v.err).slice(0, 160) : (Number.isFinite(c) ? `chance ${c}` : 'sem log de chance');
			} catch (e) { ultimoErro = String(e && e.message || e).slice(0, 160); }
			if (k < 6) await wait(1500);
		}
		return { chance: null, tentativas: 6, erro: ultimoErro };
	}
	// 1.2.2: carga/descarga com o compartimento escolhido pelo Sis-Bot (opção "Usar o tanque e o compartimento de munição"
	// da frota). itens: { mint, qtd (porão), tanque } — combustível: tanque 'encher' completa o tanque de combustível;
	// munição: tanque = quantidade para o compartimento de munição. Sem "tanque", só o porão (o SLYA usava o compartimento de
	// munição sozinho, pela opção geral transportUseAmmoBank).
	const sisMintFuel = () => sageGameAcct.account.mints.fuel.toString();
	const sisMintAmmo = () => sageGameAcct.account.mints.ammo.toString();
	async function sisContaNoPod(pod, mint) {
		const l = (await solanaReadConnection.getParsedTokenAccountsByOwner(pod, { programId: tokenProgramPK })).value;
		const t = l.find((z) => z.account.data.parsed.info.mint === mint);
		if (t) return { conta: t.pubkey, qtd: t.account.data.parsed.info.tokenAmount.uiAmount || 0 };
		const [pda] = BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync([pod.toBuffer(), tokenProgramPK.toBuffer(), new solanaWeb3.PublicKey(mint).toBuffer()], programPK);
		return { conta: pda, qtd: 0 };
	}
	async function sisCarregarExplicito(fl, coords, itens, falhou, erro) {
		updateFleetState(fl, 'Loading');
		const ct = (m) => cargoTypes.find((c) => c.account.mint.toString() == m);
		const tam = (m) => (cargoItems.find((r) => r.token == m) || {}).size || 1;
		const faltou = []; const noTanque = {};
		for (const it of itens) {
			const m = it.mint;
			if (it.tanque && (m === sisMintFuel() || m === sisMintAmmo())) {
				const comb = m === sisMintFuel(); const pod = comb ? fl.fuelTank : fl.ammoBank; const cap = comb ? fl.fuelCapacity : fl.ammoCapacity;
				const c = await sisContaNoPod(pod, m);
				const quer = Math.floor(Math.max(0, comb && it.tanque === 'encher' ? cap - c.qtd : Math.min(+it.tanque || 0, cap - c.qtd)));
				if (quer > 0) {
					const r = await execCargoFromStarbaseToFleet(fl, pod, c.conta, m, ct(m), coords, quer, false, false);
					if (r && r.name == 'NotEnoughResource') faltou.push((cargoItems.find((x) => x.token == m) || {}).name || m); else noTanque[m] = quer;
				}
				if (erro()) return falhou('erro ao carregar o ' + (comb ? 'tanque de combustível' : 'compartimento de munição'));
			}
		}
		const carga0 = await sisLerCarga(fl);
		let livre = fl.cargoCapacity - Object.entries(carga0).reduce((n, [m, q]) => n + q * tam(m), 0);
		for (const it of itens) {
			const q = Math.floor(Math.min(+it.qtd || 0, livre / tam(it.mint)));
			if (q <= 0) continue;
			const c = await sisContaNoPod(fl.cargoHold, it.mint);
			const r = await execCargoFromStarbaseToFleet(fl, fl.cargoHold, c.conta, it.mint, ct(it.mint), coords, q, false, false);
			if (r && r.name == 'NotEnoughResource') faltou.push((cargoItems.find((x) => x.token == it.mint) || {}).name || it.mint);
			else livre -= q * tam(it.mint);
			if (erro()) return falhou('não carregou');
		}
		const carga = await sisLerCarga(fl);
		const algo = Object.keys(noTanque).length || itens.some((it) => (carga[it.mint] || 0) > (carga0[it.mint] || 0));
		if (!algo) return falhou(faltou.length ? `falta ${faltou.join(', ')} na starbase [${coords}]` : 'não carregou');
		updateFleetState(fl, 'Sis-Bot: carregada');
		return { ok: true, msg: `carregada${Object.keys(noTanque).length ? ' (com tanque/munição)' : ''}${faltou.length ? ` · faltou ${faltou.join(', ')}` : ''}`, dados: { carga, tanque: noTanque } };
	}
	async function sisDescarregarExplicito(fl, coords, itens, falhou, erro) {
		updateFleetState(fl, 'Unloading');
		const keep = globalSettings.transportKeep1 ? 1 : 0;
		const entregue = {}; const doTanque = {};
		const soma = (m, q) => { entregue[m] = (entregue[m] || 0) + q; };
		for (const it of itens) {
			const c = await sisContaNoPod(fl.cargoHold, it.mint);
			let q = Math.floor(+it.qtd > 0 ? Math.min(c.qtd, +it.qtd) : c.qtd - keep);
			if (q > 0) { await execCargoFromFleetToStarbase(fl, fl.cargoHold, it.mint, coords, q, false); if (erro()) return falhou('erro ao descarregar'); soma(it.mint, q); }
			if (it.tanque && (it.mint === sisMintFuel() || it.mint === sisMintAmmo())) {
				const comb = it.mint === sisMintFuel(); const pod = comb ? fl.fuelTank : fl.ammoBank;
				const t = await sisContaNoPod(pod, it.mint);
				q = Math.floor(t.qtd - Math.max(comb ? 0 : keep, +(it.tanque.manter || 0)));
				if (q > 0) { await execCargoFromFleetToStarbase(fl, pod, it.mint, coords, q, false); if (erro()) return falhou('erro ao descarregar o ' + (comb ? 'tanque de combustível' : 'compartimento de munição')); soma(it.mint, q); doTanque[it.mint] = q; }
			}
		}
		updateFleetState(fl, 'Sis-Bot: descarregada');
		return { ok: true, msg: 'descarregada', dados: { entregue, tanque: doTanque } };
	}
	// 1.2.3: conclui (coleta) um processo de crafting/upgrade terminado, iniciado por qualquer fonte (jogo, outra ferramenta, trabalho
	// liberado). Os do próprio Sis-Bot (craftingId de um trabalho craftN) ficam com o trabalho, que já coleta sozinho.
	async function sisColetarCraft(p) {
		const [sx, sy] = ConvertCoords(p.coords);
		const starbase = await getStarbaseFromCoords(sx, sy, true); if (!starbase) return { ok: false, msg: `starbase não encontrada em [${p.coords}]` };
		const sbp = await getStarbasePlayer(userProfileAcct, starbase.publicKey); if (!sbp) return { ok: false, msg: 'sem conta nesta starbase' };
		const starbasePlayer = sbp.publicKey;
		const insts = await sageProgram.account.craftingInstance.all([{ memcmp: { offset: 11, bytes: starbasePlayer.toBase58() } }]);
		let achado = null;
		for (const ci of insts) {
			const procs = await craftingProgram.account.craftingProcess.all([{ memcmp: { offset: 17, bytes: ci.publicKey.toBase58() } }]);
			const pr = procs.find((x) => x.publicKey.toString() === p.processo); if (pr) { achado = { ci, pr }; break; }
		}
		if (!achado) return { ok: true, msg: 'já tinha sido coletado', dados: { coletado: false } };
		const a = achado.pr.account; const id = a.craftingId.toNumber();
		if (![2, 3].includes(a.status)) return { ok: false, msg: `ainda não concluído (status ${a.status})` };
		const rec = craftRecipes.find((r) => r.publicKey.toString() === a.recipe.toString()); const up = !rec;
		const tempo = await getStarbaseTime(starbase, up ? 'Upgrade' : 'Craft');
		if (a.endTime.toNumber() >= tempo.starbaseTime) return { ok: false, msg: 'ainda não terminou no relógio da starbase' };
		for (let n = 1; n <= 20; n++) { try { const j = JSON.parse(await GM.getValue('craft' + n, '{}')); if (j.craftingId && +j.craftingId === id) return { ok: true, msg: `é do trabalho craft${n}, que coleta sozinho`, dados: { coletado: false, job: 'craft' + n } }; } catch (e) { } }
		const nome = rec ? rec.name : ((upgradeRecipes.find((r) => r.publicKey.toString() === a.recipe.toString()) || {}).name || 'upgrade');
		const holds = await getStarbasePlayerCargoHolds(starbasePlayer);
		const obj = { craftingProcess: achado.pr.publicKey, craftingInstance: achado.ci.publicKey, recipe: a.recipe, status: a.status, inputsChecksum: a.inputsChecksum, outputsChecksum: a.outputsChecksum, craftingId: id, quantity: a.quantity.toNumber() };
		const uc = { label: 'Sis-Bot: coleta', state: 'Idle', sisColeta: true, craftingCoords: p.coords, coordinates: p.coords, item: nome };
		await sisComPrazo(up ? execCompleteUpgrade(starbase, starbasePlayer, holds, obj, uc) : execCompleteCrafting(starbase, starbasePlayer, holds, obj, uc), 180000, 'coletar o craft');
		if (String(uc.state).includes('ERROR')) return { ok: false, msg: uc.state };
		return { ok: true, msg: `coletado: ${a.quantity.toNumber()} × ${nome}`, dados: { coletado: true, receita: nome, quantidade: a.quantity.toNumber() } };
	}
	// =====================================================================================================
	// [Sis-Ágora] 1.2.7: coletar claim stakes (harvest) pedido pelo dashboard (botão Coletar Agora ou coleta automática a X%).
	// Programa Claim Stake STAKEr4Bh8… (@staratlas/claim-stake 0.12.0), instrução processHarvest: conferida contra o SDK
	// (mesmos dados e contas, na mesma ordem) e simulada na chain em 01/11/2026. Quem assina (playerAccount) é a dona do
	// claim stake, então só dá para coletar os claim stakes da hot wallet deste script. Tudo conferido de novo na chain:
	// a conta pedida tem de ser o PDA CLAIM_STAKING_INFO(esta chave, mint) e ter claim stakes em stake.
	// Os recursos (Fuel, Food, Ammo, Toolkit) vão para as contas de token da hot wallet (criadas se faltarem).
	// =====================================================================================================
	const SIS_CS_PROG = new solanaWeb3.PublicKey('STAKEr4Bh8sbBMoAVmTDBRqouPzgdocVrvtjmhJhd65');
	const SIS_CS_DISC = [0xbf, 0x46, 0x66, 0x29, 0xe2, 0x24, 0x7f, 0xa0]; // sha256('global:process_harvest')[0..8]
	const SIS_CS_R4 = [['fuel', 'fueL3hBZjLLLJHiFH9cqZoozTG3XQZ53diwFPwbzNim'], ['arms', 'ammoK8AkX2wnebQb35cDAZtTkvsXQbi82cGeTnUvvfK'],
		['food', 'foodQJAztMzX1DKpLaiounNe2BDMds5RNuPC6jsNrDG'], ['toolkit', 'tooLsNYLiVqzg8o4m3L2Uetbn62mvMWRqkog6PQeYKL']];
	const SIS_CS_POR_TX = 3;
	const sisCsPda = (seeds) => solanaWeb3.PublicKey.findProgramAddressSync(seeds.map((x) => (typeof x === 'string' ? BrowserBuffer.Buffer.Buffer.from(x) : x.toBuffer())), SIS_CS_PROG)[0];
	const sisAtaDe = (dono, mint) => solanaWeb3.PublicKey.findProgramAddressSync([dono.toBuffer(), tokenProgramPK.toBuffer(), mint.toBuffer()], programPK)[0];
	const sisU64 = (b, o) => new BrowserAnchor.anchor.BN(b.subarray(o, o + 8), 'le');
	let sisFilaClaim = Promise.resolve();

	function sisClaimEnfileirar(cmd) {
		const auto = !!(cmd.params && cmd.params.auto);
		sisEvento('comando', '', `Coletar claim stakes${auto ? ' (automático)' : ''}: recebido do dashboard`);
		sisFilaClaim = sisFilaClaim.then(async () => {
			let r;
			try { r = await sisComPrazo(sisClaimColetar(cmd), 6 * 60000, 'coletar claim stakes'); }
			catch (e) { r = { ok: false, msg: 'falha: ' + (e && e.message ? e.message : e) }; }
			sisFila.comandos.push({ id: cmd.id, ok: !!r.ok, msg: String(r.msg).slice(0, 250), dados: r.dados || null });
			sisEvento(r.ok ? 'comando' : 'alerta', '', `Coletar claim stakes${auto ? ' (automático)' : ''}: ${r.msg}`);
			sisSyncAgora();
		});
	}
	async function sisClaimLer(pk) {
		const info = await solanaReadConnection.getAccountInfo(pk, 'confirmed');
		if (!info || !info.owner.equals(SIS_CS_PROG) || info.data.length < 90) return null;
		const b = info.data;
		return { dono: new solanaWeb3.PublicKey(b.subarray(10, 42)), mint: new solanaWeb3.PublicKey(b.subarray(42, 74)), ultima: sisU64(b, 74).toNumber(), qtd: sisU64(b, 82).toNumber() };
	}
	async function sisSaldosR4(atas) {
		const l = await solanaReadConnection.getMultipleAccountsInfo(atas, 'confirmed');
		return l.map((i) => (i && i.data && i.data.length >= 72 ? sisU64(i.data, 64) : null));
	}
	async function sisClaimColetar(cmd) {
		const p = cmd.params || {};
		if (!userPublicKey) return { ok: false, msg: 'hot wallet não carregada' };
		const dono = userPublicKey;
		const pedidas = Array.isArray(p.contas) ? p.contas.slice(0, 40) : [];
		if (!pedidas.length) return { ok: false, msg: 'nenhum claim stake pedido' };
		// 1) conferir cada conta na chain
		const validas = [], recusadas = [];
		for (const x of pedidas) {
			const conta = String((x && x.conta) || ''), mintTxt = String((x && x.mint) || '');
			if (!SIS_PK_RE.test(conta) || !SIS_PK_RE.test(mintTxt)) { recusadas.push({ conta, msg: 'endereço inválido' }); continue; }
			const mint = new solanaWeb3.PublicKey(mintTxt);
			const pda = sisCsPda(['CLAIM_STAKING_INFO', dono, mint]);
			if (pda.toString() !== conta) { recusadas.push({ conta, msg: 'não é um claim stake desta hot wallet' }); continue; }
			const a = await sisClaimLer(pda);
			if (!a || !a.dono.equals(dono) || !a.mint.equals(mint)) { recusadas.push({ conta, msg: 'conta não encontrada na blockchain' }); continue; }
			if (a.qtd <= 0) { recusadas.push({ conta, msg: 'sem claim stakes em stake' }); continue; }
			if (validas.some((v) => v.conta === conta)) continue;
			validas.push({ conta, pda, mint, antes: a.ultima, qtd: a.qtd });
		}
		if (!validas.length) return { ok: false, msg: recusadas.map((r) => r.msg).filter((v, i, l) => l.indexOf(v) === i).join('; ') || 'nada para coletar', dados: { contas: [], recusadas } };
		// 2) contas de token da hot wallet para os 4 recursos (cria as que faltam, na 1ª transação)
		const mintsR4 = SIS_CS_R4.map(([n, m]) => ({ n, mint: new solanaWeb3.PublicKey(m) }));
		const atas = mintsR4.map((r) => sisAtaDe(dono, r.mint));
		const antes = await sisSaldosR4(atas);
		const criar = [];
		atas.forEach((ata, i) => {
			if (antes[i] !== null) return;
			criar.push({ instruction: new solanaWeb3.TransactionInstruction({ programId: programPK, data: BrowserBuffer.Buffer.Buffer.from([1]), keys: [
				{ pubkey: dono, isSigner: true, isWritable: true }, { pubkey: ata, isSigner: false, isWritable: true }, { pubkey: dono, isSigner: false, isWritable: false },
				{ pubkey: mintsR4[i].mint, isSigner: false, isWritable: false }, { pubkey: solanaWeb3.SystemProgram.programId, isSigner: false, isWritable: false },
				{ pubkey: tokenProgramPK, isSigner: false, isWritable: false }] }) });
		});
		const tes = { fuel: sisCsPda(['FUEL_TREASURY']), arms: sisCsPda(['ARMS_TREASURY']), food: sisCsPda(['FOOD_TREASURY']), toolkit: sisCsPda(['TOOLKIT_TREASURY_ACCOUNT']) };
		const auth = sisCsPda(['TREASURY_AUTHORITY']);
		const ixColetar = (v) => ({ instruction: new solanaWeb3.TransactionInstruction({ programId: SIS_CS_PROG, data: BrowserBuffer.Buffer.Buffer.from(SIS_CS_DISC), keys: [
			{ pubkey: dono, isSigner: true, isWritable: false }, { pubkey: v.pda, isSigner: false, isWritable: true },
			{ pubkey: sisCsPda(['CLAIM_STAKE_VARS', v.mint]), isSigner: false, isWritable: false },
			{ pubkey: tes.fuel, isSigner: false, isWritable: true }, { pubkey: tes.arms, isSigner: false, isWritable: true },
			{ pubkey: tes.food, isSigner: false, isWritable: true }, { pubkey: tes.toolkit, isSigner: false, isWritable: true },
			{ pubkey: atas[0], isSigner: false, isWritable: true }, { pubkey: atas[1], isSigner: false, isWritable: true },
			{ pubkey: atas[2], isSigner: false, isWritable: true }, { pubkey: atas[3], isSigner: false, isWritable: true },
			{ pubkey: auth, isSigner: false, isWritable: false }, { pubkey: v.mint, isSigner: false, isWritable: false },
			{ pubkey: tokenProgramPK, isSigner: false, isWritable: false }] }) });
		// 3) transações de até SIS_CS_POR_TX coletas
		const falhas = [];
		// (com contas de token para criar, a 1ª transação leva só 1 coleta, para caber no limite de tamanho)
		for (let i = 0, n = criar.length ? 1 : SIS_CS_POR_TX; i < validas.length; i += n, n = SIS_CS_POR_TX) {
			const lote = validas.slice(i, i + n);
			const ixs = (i === 0 ? criar : []).concat(lote.map(ixColetar));
			const erro = await sisTxFrota(ixs.length === 1 ? ixs[0] : ixs, 'CLAIM STAKE', 'Claim Stakes');
			if (erro) lote.forEach((v) => { v.erro = erro; falhas.push(v); });
		}
		// 4) conferir na chain: a última coleta mudou e quanto entrou na hot wallet
		const contas = [];
		for (const v of validas) {
			let depois = null;
			if (!v.erro) depois = await sisEsperarChain(async () => { const a = await sisClaimLer(v.pda); return a && a.ultima > v.antes ? a : null; }, 20000);
			contas.push({ conta: v.conta, ok: !!depois, ultima: depois ? depois.ultima : v.antes, msg: v.erro || (depois ? '' : 'a coleta não apareceu na blockchain') });
		}
		const dep = await sisSaldosR4(atas);
		const coletado = {};
		mintsR4.forEach((r, i) => { coletado[r.n] = dep[i] ? dep[i].sub(antes[i] || new BrowserAnchor.anchor.BN(0)).toNumber() : 0; });
		const okN = contas.filter((c) => c.ok).length;
		const txt = `Fuel ${coletado.fuel}, Food ${coletado.food}, Ammo ${coletado.arms}, Toolkit ${coletado.toolkit}`;
		const dados = { contas, recusadas, coletado, auto: !!p.auto };
		if (!okN) return { ok: false, msg: (falhas[0] && falhas[0].erro) || contas[0].msg || 'a coleta falhou', dados };
		return { ok: true, msg: `${okN} de ${pedidas.length} coletado(s): ${txt}` + (okN < pedidas.length ? ` (${pedidas.length - okN} não: ${(recusadas[0] || falhas[0] || contas.find((c) => !c.ok) || {}).msg || '?'})` : ''), dados };
	}
	// =====================================================================================================
	// 1.2.6: frotas pelo dashboard — criar, adicionar naves e desfazer (desbandar)
	// O dashboard manda só dados (starbase, nome, naves e quantidades). Aqui tudo é conferido de novo na blockchain:
	// starbase da sua facção, naves guardadas nessa starbase, nome livre (até 32 bytes), frota sua (não alugada), atracada e
	// sem tarefa rodando. Roda numa fila própria (não trava a sincronização) e responde quando termina.
	// As 5 instruções seguem o IDL do SAGE atual (@staratlas/sage 1.8.10, conferido byte a byte com o SDK em 31/10/2026):
	// createFleet/addShipToFleet têm crewAmount (nulo = a tripulação necessária) e addShipToFleet/disbandFleet levam a
	// starbase como conta extra. O IDL embutido do SLYA é antigo para essas instruções.
	// =====================================================================================================
	const SIS_GERIR = ['frotas_criar', 'frotas_adicionar', 'frotas_desfazer'];
	const SIS_GERIR_NOME = { frotas_criar: 'Criar frota', frotas_adicionar: 'Adicionar naves', frotas_desfazer: 'Desfazer frota' };
	const SIS_IX_FROTAS = (() => {
		const c = (name, isMut, isSigner) => ({ name, isMut, isSigner });
		const gp = { name: 'gameAccountsAndProfile', accounts: [{ name: 'gameAndProfileAndFaction', accounts: [c('key', false, true), c('profile', false, false), c('profileFaction', false, false), c('gameId', false, false)] }, c('gameState', false, false)] };
		const sb = { name: 'starbaseAndStarbasePlayer', accounts: [c('starbase', false, false), c('starbasePlayer', true, false)] };
		const inp = (t) => [{ name: 'input', type: { defined: t } }];
		const st = (name, fields) => ({ name, type: { kind: 'struct', fields: fields.map(([n, t]) => ({ name: n, type: t })) } });
		const a32 = { array: ['u8', 32] };
		return {
			version: '0.1.2', name: 'sage', accounts: [], errors: [],
			instructions: [
				{ name: 'createFleet', accounts: [gp, c('funder', true, true), c('fleet', true, false), c('fleetShips', true, false), c('cargoHold', true, false), c('fuelTank', true, false), c('ammoBank', true, false), c('ship', false, false), sb, c('cargoStatsDefinition', false, false), c('cargoProgram', false, false), c('systemProgram', false, false)], args: inp('CreateFleetInput') },
				{ name: 'addShipToFleet', accounts: [{ name: 'gameAccountsFleetAndOwner', accounts: [{ name: 'gameFleetAndOwner', accounts: [{ name: 'fleetAndOwner', accounts: [c('key', false, true), c('owningProfile', false, false), c('owningProfileFaction', false, false), c('fleet', true, false)] }, c('gameId', false, false)] }, c('gameState', false, false)] }, c('funder', true, true), c('fleetShips', true, false), c('ship', false, false), sb, c('systemProgram', false, false)], args: inp('AddShipToFleetInput') },
				{ name: 'disbandFleet', accounts: [gp, c('funder', true, true), c('disbandedFleet', true, false), c('fleet', true, false), c('fleetShips', true, false), c('cargoHold', true, false), c('fuelTank', true, false), c('ammoBank', true, false), sb, c('cargoProgram', false, false), c('systemProgram', false, false)], args: inp('DisbandFleetInput') },
				{ name: 'disbandedFleetToEscrow', accounts: [gp, c('funder', true, true), c('disbandedFleet', true, false), c('fleetShips', true, false), sb, c('ship', false, false), c('systemProgram', false, false)], args: inp('DisbandedFleetToEscrowInput') },
				{ name: 'closeDisbandedFleet', accounts: [c('key', false, true), c('playerProfile', false, false), c('fundsTo', true, false), c('disbandedFleet', true, false), c('fleetShips', true, false)], args: inp('CloseDisbandedFleetInput') },
			],
			types: [
				st('CreateFleetInput', [['shipAmount', 'u8'], ['fleetLabel', a32], ['shipEscrowIndex', 'u32'], ['cargoHoldSeeds', a32], ['fuelTankSeeds', a32], ['ammoBankSeeds', a32], ['keyIndex', 'u16'], ['crewAmount', { option: 'u16' }]]),
				st('AddShipToFleetInput', [['shipAmount', 'u8'], ['shipEscrowIndex', 'u32'], ['fleetShipInfoIndex', { option: 'u32' }], ['keyIndex', 'u16'], ['crewAmount', { option: 'u16' }]]),
				st('DisbandFleetInput', [['keyIndex', 'u16']]),
				st('DisbandedFleetToEscrowInput', [['shipAmount', 'u16'], ['shipEscrowIndex', { option: 'u32' }], ['fleetShipInfoIndex', 'u32'], ['keyIndex', 'u16']]),
				st('CloseDisbandedFleetInput', [['keyIndex', 'u16']]),
			],
		};
	})();
	let sisProgFrotasObj = null;
	const sisProgFrotas = () => (sisProgFrotasObj || (sisProgFrotasObj = new BrowserAnchor.anchor.Program(SIS_IX_FROTAS, sageProgramPK, anchorProvider)));
	const SIS_PK_RE = /^[1-9A-HJ-NP-Za-km-z]{32,44}$/;
	let sisFilaGerir = Promise.resolve(), sisGerindo = 0, sisRecarregarPedido = false;

	function sisGerirEnfileirar(cmd) {
		const nomeAcao = SIS_GERIR_NOME[cmd.acao];
		sisGerindo++;
		sisEvento('comando', String((cmd.params && cmd.params.nome) || ''), `${nomeAcao}: recebido do dashboard`);
		sisFilaGerir = sisFilaGerir.then(async () => {
			let r;
			try { r = await sisComPrazo(sisGerir(cmd), 20 * 60000, nomeAcao.toLowerCase()); }
			catch (e) { r = { ok: false, msg: 'falha: ' + (e && e.message ? e.message : e) }; }
			sisGerindo--;
			sisFila.comandos.push({ id: cmd.id, ok: !!r.ok, msg: String(r.msg).slice(0, 250), dados: r.dados || null });
			sisEvento(r.ok ? 'comando' : 'alerta', String((r.dados && r.dados.nome) || (cmd.params && cmd.params.nome) || ''), `${nomeAcao}: ${r.msg}`);
			sisSyncAgora();
			if (r.mudou && cmd.params && cmd.params.recarregar) sisRecarregarQuandoLivre();
		});
	}
	// a lista de frotas do SLYA é montada ao abrir a página: frota nova, desfeita ou com outras naves (stats) só aparece certa depois de recarregar
	async function sisRecarregarQuandoLivre() {
		if (sisRecarregarPedido) return; sisRecarregarPedido = true;
		sisEvento('info', '', 'Frotas mudaram: a página do jogo vai recarregar quando as ordens em andamento terminarem, para o Sis-Bot ver as frotas como estão agora.');
		const t0 = Date.now();
		while (Date.now() - t0 < 5 * 60000 && (sisGerindo > 0 || sisPainel.ordens.size > 0)) await wait(3000);
		try { await GM.setValue('sisRetomar', enableAssistant ? '1' : ''); } catch (e) { }
		await sisSyncAgora(); setTimeout(() => location.reload(), 800);
	}
	// ---- leitura das contas (dados brutos: o IDL embutido não conhece o layout atual dessas listas)
	const sisU32 = (b, o) => b[o] + b[o + 1] * 256 + b[o + 2] * 65536 + b[o + 3] * 16777216;
	function sisListaNaves(b, oCont, oIni) {
		const n = sisU32(b, oCont); const out = [];
		for (let i = 0; i < n && oIni + (i + 1) * 48 <= b.length; i++) {
			const o = oIni + i * 48;
			out.push({ i, ship: new solanaWeb3.PublicKey(b.subarray(o, o + 32)).toString(), qtd: new BrowserAnchor.anchor.BN(b.subarray(o + 32, o + 40), 'le').toNumber() });
		}
		return out;
	}
	// StarbasePlayer: 8 disc + 1 version + 4×32 + 1 bump = 138 → shipEscrowCount (u32); crew 142/146/150; lista de WrappedShipEscrow (48 bytes) a partir de 170
	async function sisHangar(starbasePk) {
		const sbp = await getStarbasePlayer(userProfileAcct, starbasePk);
		if (!sbp) return { sbp: null, naves: [] };
		const info = await solanaReadConnection.getAccountInfo(sbp.publicKey, 'confirmed');
		return { sbp: sbp.publicKey, naves: info ? sisListaNaves(info.data, 138, 170) : [] };
	}
	// FleetShips: 8 disc + 1 version + 32 fleet = 41 → fleetShipsInfoCount (u32); 45 bump; lista de FleetShipsInfo (48 bytes) a partir de 46
	const sisPdaFleetShips = (fleetPk) => BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync([BrowserBuffer.Buffer.Buffer.from('FleetShips'), fleetPk.toBuffer()], sageProgramPK)[0];
	async function sisNavesDaFrota(fleetShipsPk) {
		const info = await solanaReadConnection.getAccountInfo(fleetShipsPk, 'confirmed');
		return info ? sisListaNaves(info.data, 41, 46) : null;
	}
	const sisPdaLabel = (seed, label) => BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync([BrowserBuffer.Buffer.Buffer.from(seed), sageGameAcct.publicKey.toBuffer(), userProfileAcct.toBuffer(), BrowserBuffer.Buffer.Buffer.from(label)], sageProgramPK)[0];
	function sisLabel(nome) {
		const t = String(nome || '').trim();
		if (!t) return { erro: 'nome vazio' };
		if (/[\u0000-\u001f\u007f]/.test(t)) return { erro: 'nome com caractere inválido' };
		const b = new TextEncoder().encode(t);
		if (b.length > 32) return { erro: `nome longo demais (${b.length} bytes; o jogo aceita até 32)` };
		const a = new Array(32).fill(0); b.forEach((x, i) => { a[i] = x; });
		return { nome: t, label: a };
	}
	// a mesma nave pode ter várias entradas (versões do Ship / updateId): soma e maior entrada
	const sisTotalNave = (l, ship) => l.filter((e) => e.ship === ship).reduce((t, e) => t + e.qtd, 0);
	const sisMaiorEntrada = (l, ship) => l.filter((e) => e.ship === ship && e.qtd > 0).sort((a, b) => b.qtd - a.qtd)[0] || null;
	const sisNomeLabel = (arr) => new TextDecoder('utf-8').decode(new Uint8Array(arr)).replace(/\0/g, '');
	async function sisEsperarChain(fn, ms = 25000) {
		const t0 = Date.now();
		while (Date.now() - t0 < ms) { try { const v = await fn(); if (v) return v; } catch (e) { } await wait(1500); }
		return null;
	}
	function sisErroTx(res) {
		const logs = (res && res.meta && res.meta.logMessages) || [];
		const l = logs.find((x) => /Error Message:/.test(x)) || logs.find((x) => /Error Code:|custom program error|failed/.test(x));
		return l ? String(l).replace(/^Program log: /, '').slice(0, 200) : '';
	}
	async function sisTxFrota(ix, rotulo, nome) {
		const pseudo = { label: nome || 'Frotas', state: 'Idle', sisColeta: true, exitWarpSubwarpPending: 0 };
		const res = await sisComPrazo(txSignAndSend(ix, pseudo, rotulo, 100), 300000, rotulo.toLowerCase());
		if (!res || (res.meta && res.meta.err) || String(pseudo.state).includes('ERROR')) return sisErroTx(res) || 'a transação falhou';
		return '';
	}
	async function sisStarbasePorPk(pk) {
		const sb = await sageProgram.account.starbase.fetch(pk);
		const x = sb.sector[0].toNumber(), y = sb.sector[1].toNumber();
		const t = validTargets.find((v) => v.x === x && v.y === y);
		return { publicKey: pk, x, y, coords: `${x},${y}`, nome: t ? t.name : `[${x},${y}]`, daFaccao: !!t };
	}
	// naves pedidas pelo dashboard: [{ship, qtd}] somando repetidas
	function sisNavesPedidas(l) {
		if (!Array.isArray(l) || !l.length || l.length > 30) return { erro: 'lista de naves vazia ou grande demais' };
		const m = new Map();
		for (const x of l) {
			const ship = String((x && x.ship) || ''); const q = Math.floor(+(x && x.qtd));
			if (!SIS_PK_RE.test(ship)) return { erro: 'nave inválida' };
			if (!(q >= 1 && q <= 100000)) return { erro: 'quantidade de naves inválida' };
			m.set(ship, (m.get(ship) || 0) + q);
		}
		return { naves: [...m].map(([ship, qtd]) => ({ ship, qtd })) };
	}
	// frota sem tarefa rodando e sem ordem do Sis-Bot em andamento (espera até "ms" — o dashboard pode ter mandado parar a tarefa antes)
	async function sisFrotaLivre(pk, ms) {
		const t0 = Date.now();
		for (;;) {
			const fl = userFleets.find((f) => f.publicKey.toString() === pk);
			const ordem = sisPainel.ordens.has(pk);
			let tarefa = false;
			if (fl && enableAssistant) {
				let cfg = {}; try { cfg = JSON.parse(await GM.getValue(pk, '{}')) || {}; } catch (e) { }
				const st = String(fl.state || '');
				if (cfg.sisCerebro || sisEmMissao.has(pk)) tarefa = !sisCanceladas.has(pk);
				else if (cfg.assignment) tarefa = !st.includes('STOPPED') && !(st.includes('ERROR') && !st.includes('⌛'));
			}
			if (!ordem && !tarefa) return '';
			if (Date.now() - t0 > ms) return ordem ? 'a frota está executando uma ordem do Sis-Bot' : 'a frota está com a tarefa rodando — pare a tarefa dela (⏸ no dashboard) ou pare o Sis-Bot antes';
			await wait(3000);
		}
	}
	// frota sua, não alugada, atracada; se "atracar" e ela estiver parada no setor de uma starbase da facção, atraca antes
	async function sisFrotaAtracada(pk, atracar) {
		let acc; try { acc = await sageProgram.account.fleet.fetch(new solanaWeb3.PublicKey(pk)); } catch (e) { return { erro: 'frota não encontrada na blockchain' }; }
		const nome = sisNomeLabel(acc.fleetLabel);
		if (acc.ownerProfile.toString() !== userProfileAcct.toString()) return { erro: `${nome}: só o dono da frota pode mudar as naves dela`, nome };
		const sub = acc.subProfile && acc.subProfile.key ? acc.subProfile.key.toString() : '';
		if (sub && sub !== '11111111111111111111111111111111') return { erro: `${nome}: a frota está alugada para outro perfil`, nome };
		const ler = async () => getFleetState(await solanaReadConnection.getAccountInfo(new solanaWeb3.PublicKey(pk), 'confirmed'));
		let [st, ex] = await ler();
		if (st === 'Idle' && atracar && ex) {
			const t = validTargets.find((v) => v.x === +ex[0] && v.y === +ex[1]);
			const fl = userFleets.find((f) => f.publicKey.toString() === pk);
			if (t && fl) {
				sisEvento('info', nome, `atracando em ${t.name} antes de mudar a frota`);
				await sisComPrazo(execDock(fl, `${t.x},${t.y}`), 180000, 'atracar');
				const r = await sisEsperarChain(async () => { const v = await ler(); return v[0] === 'StarbaseLoadingBay' ? v : null; }, 30000);
				if (r) [st, ex] = r;
			}
		}
		if (st !== 'StarbaseLoadingBay') return { erro: `${nome}: a frota precisa estar atracada numa starbase (agora: ${st}${st === 'Idle' && ex ? ' em [' + ex.join(',') + ']' : ''})`, nome };
		return { acc, nome, starbase: await sisStarbasePorPk(ex.starbase) };
	}
	// põe as naves pedidas na frota (já criada) com addShipToFleet, no máx. 255 por instrução; relê starbase e frota a cada passo.
	// fleetShipInfoIndex nulo = entrada nova na lista da frota (como o jogo faz; a mesma nave pode ter várias entradas)
	async function sisPorNaves(fleetPk, nome, starbase, fila, feitas) {
		const fleetShips = sisPdaFleetShips(fleetPk); const k = Number(userProfileKeyIdx) || 0;
		for (const p of fila) {
			let resta = p.qtd;
			while (resta > 0) {
				const hg = await sisHangar(starbase.publicKey); const esc = sisMaiorEntrada(hg.naves, p.ship);
				if (!esc) return `não há mais ${p.ship.slice(0, 6)}… guardada em ${starbase.nome}`;
				const antes = sisTotalNave((await sisNavesDaFrota(fleetShips)) || [], p.ship);
				const q = Math.min(resta, esc.qtd, 255);
				const ix = { instruction: await sisProgFrotas().methods.addShipToFleet({ shipAmount: q, shipEscrowIndex: esc.i, fleetShipInfoIndex: null, keyIndex: k, crewAmount: null }).accountsStrict({
					gameAccountsFleetAndOwner: { gameFleetAndOwner: { fleetAndOwner: { key: userPublicKey, owningProfile: userProfileAcct, owningProfileFaction: userProfileFactionAcct.publicKey, fleet: fleetPk }, gameId: sageGameAcct.publicKey }, gameState: sageGameAcct.account.gameState },
					funder: userPublicKey, fleetShips, ship: new solanaWeb3.PublicKey(p.ship),
					starbaseAndStarbasePlayer: { starbase: starbase.publicKey, starbasePlayer: hg.sbp }, systemProgram: solanaWeb3.SystemProgram.programId,
				}).remainingAccounts([{ pubkey: starbase.publicKey, isSigner: false, isWritable: false }]).instruction() };
				const e = await sisTxFrota(ix, 'ADD SHIPS', nome);
				if (e) return e;
				await sisEsperarChain(async () => sisTotalNave((await sisNavesDaFrota(fleetShips)) || [], p.ship) >= antes + q);
				resta -= q; feitas.push({ ship: p.ship, qtd: q });
			}
		}
		return '';
	}
	let sisCarteiraAvisoEm = 0;
	function sisCarteiraOk() {
		const valida = (k) => k && typeof k.toBase58 === 'function';
		if (valida(userPublicKey)) return true;
		let pk = null;
		try {
			pk = customKeypair ? customKeypair.publicKey
				: (typeof solflare !== 'undefined' && solflare.publicKey) || (typeof phantom !== 'undefined' && phantom && phantom.solana && phantom.solana.publicKey) || (typeof solana !== 'undefined' && solana && solana.publicKey) || null;
		} catch (e) { pk = null; }
		if (valida(pk)) { userPublicKey = pk; return true; }
		return false;
	}
	function sisCarteiraAusente() {
		if (Date.now() - sisCarteiraAvisoEm < 300000) return;
		sisCarteiraAvisoEm = Date.now();
		logError('Carteira sem chave pública (extensão bloqueada/desconectada ou hot wallet não configurada): operações pausadas até a carteira voltar.');
	}
	async function sisGerir(cmd) {
		const p = cmd.params || {};
		if (!initComplete || !userProfileAcct || !sisCarteiraOk()) return { ok: false, msg: 'script ainda iniciando ou carteira sem chave pública' };
		if (!customKeypair && typeof solflare === 'undefined' && typeof phantom === 'undefined' && typeof solana === 'undefined') return { ok: false, msg: 'sem carteira para assinar' };
		const somar = (l) => l.reduce((t, x) => t + x.qtd, 0);
		const k = Number(userProfileKeyIdx) || 0;

		if (cmd.acao === 'frotas_criar') {
			const lb = sisLabel(p.nome); if (lb.erro) return { ok: false, msg: lb.erro };
			const np = sisNavesPedidas(p.naves); if (np.erro) return { ok: false, msg: np.erro };
			const t = validTargets.find((v) => `${v.x},${v.y}` === String(p.coords || '').replace(/\s/g, ''));
			if (!t) return { ok: false, msg: 'starbase fora da sua facção' };
			if (userFleets.some((f) => f.label === lb.nome)) return { ok: false, msg: `já existe a frota "${lb.nome}"` };
			const fleetPk = sisPdaLabel('Fleet', lb.label);
			if (await solanaReadConnection.getAccountInfo(fleetPk, 'confirmed')) return { ok: false, msg: `já existe uma frota com o nome "${lb.nome}"` };
			const sbAcc = await getStarbaseFromCoords(t.x, t.y, true); if (!sbAcc) return { ok: false, msg: `starbase não encontrada em [${t.x},${t.y}]` };
			const starbase = { publicKey: sbAcc.publicKey, x: t.x, y: t.y, coords: `${t.x},${t.y}`, nome: t.name };
			const hg = await sisHangar(starbase.publicKey);
			if (!hg.sbp) return { ok: false, msg: `você não tem nada guardado em ${t.name}` };
			for (const x of np.naves) {
				const tem = sisTotalNave(hg.naves, x.ship);
				if (tem < x.qtd) return { ok: false, msg: `em ${t.name} há ${tem} dessa nave (${x.ship.slice(0, 6)}…), pedido ${x.qtd}` };
			}
			// 1ª instrução: cria a frota com a 1ª nave (até 255); o resto entra com addShipToFleet
			const p0 = np.naves[0]; const e0 = sisMaiorEntrada(hg.naves, p0.ship); const q0 = Math.min(p0.qtd, e0.qtd, 255);
			const seeds = [0, 1, 2].map(() => solanaWeb3.Keypair.generate().publicKey.toBuffer());
			const pod = (s) => BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync([BrowserBuffer.Buffer.Buffer.from('cargo_pod'), s], cargoProgramPK)[0];
			const fleetShips = sisPdaFleetShips(fleetPk);
			sisEvento('info', lb.nome, `criando a frota em ${t.name} com ${somar(np.naves)} naves`);
			const ix = { instruction: await sisProgFrotas().methods.createFleet({ shipAmount: q0, fleetLabel: lb.label, shipEscrowIndex: e0.i, cargoHoldSeeds: Array.from(seeds[0]), fuelTankSeeds: Array.from(seeds[1]), ammoBankSeeds: Array.from(seeds[2]), keyIndex: k, crewAmount: null }).accountsStrict({
				gameAccountsAndProfile: { gameAndProfileAndFaction: { key: userPublicKey, profile: userProfileAcct, profileFaction: userProfileFactionAcct.publicKey, gameId: sageGameAcct.publicKey }, gameState: sageGameAcct.account.gameState },
				funder: userPublicKey, fleet: fleetPk, fleetShips, cargoHold: pod(seeds[0]), fuelTank: pod(seeds[1]), ammoBank: pod(seeds[2]), ship: new solanaWeb3.PublicKey(p0.ship),
				starbaseAndStarbasePlayer: { starbase: starbase.publicKey, starbasePlayer: hg.sbp }, cargoStatsDefinition: sageGameAcct.account.cargo.statsDefinition, cargoProgram: cargoProgramPK, systemProgram: solanaWeb3.SystemProgram.programId,
			}).instruction() };
			const e = await sisTxFrota(ix, 'CREATE FLEET', lb.nome);
			if (e) return { ok: false, msg: `a frota não foi criada: ${e}`, dados: { nome: lb.nome } };
			if (!(await sisEsperarChain(() => solanaReadConnection.getAccountInfo(fleetShips, 'confirmed')))) return { ok: false, msg: 'a transação foi confirmada, mas a frota ainda não aparece na blockchain; confira no jogo', dados: { nome: lb.nome, frota: fleetPk.toString() }, mudou: true };
			const feitas = [{ ship: p0.ship, qtd: q0 }];
			const fila = [{ ship: p0.ship, qtd: p0.qtd - q0 }, ...np.naves.slice(1)].filter((x) => x.qtd > 0);
			const falta = await sisPorNaves(fleetPk, lb.nome, starbase, fila, feitas);
			const total = somar(feitas);
			const dados = { frota: fleetPk.toString(), nome: lb.nome, coords: starbase.coords, starbase: starbase.nome, naves: feitas, pedidas: somar(np.naves) };
			return { ok: true, mudou: true, dados, msg: falta ? `frota "${lb.nome}" criada em ${t.name} com ${total} de ${somar(np.naves)} naves; o resto não entrou: ${falta}` : `frota "${lb.nome}" criada em ${t.name} com ${total} naves` };
		}

		const pk = String(cmd.alvo || p.frota || '');
		if (!SIS_PK_RE.test(pk)) return { ok: false, msg: 'frota inválida' };

		if (cmd.acao === 'frotas_adicionar') {
			const np = sisNavesPedidas(p.naves); if (np.erro) return { ok: false, msg: np.erro };
			const livre = await sisFrotaLivre(pk, 180000); if (livre) return { ok: false, msg: livre };
			const a = await sisFrotaAtracada(pk, p.atracar !== false); if (a.erro) return { ok: false, msg: a.erro, dados: { nome: a.nome || '' } };
			if (!a.starbase.daFaccao) return { ok: false, msg: `${a.nome}: a starbase não é da sua facção` };
			const hg = await sisHangar(a.starbase.publicKey);
			for (const x of np.naves) {
				const tem = sisTotalNave(hg.naves, x.ship);
				if (tem < x.qtd) return { ok: false, msg: `em ${a.starbase.nome} há ${tem} dessa nave (${x.ship.slice(0, 6)}…), pedido ${x.qtd}`, dados: { nome: a.nome } };
			}
			sisEvento('info', a.nome, `adicionando ${somar(np.naves)} naves em ${a.starbase.nome}`);
			const feitas = [];
			const falta = await sisPorNaves(new solanaWeb3.PublicKey(pk), a.nome, a.starbase, np.naves, feitas);
			const total = somar(feitas);
			const dados = { frota: pk, nome: a.nome, coords: a.starbase.coords, starbase: a.starbase.nome, naves: feitas, pedidas: somar(np.naves) };
			if (!total) return { ok: false, msg: `nenhuma nave entrou: ${falta}`, dados };
			return { ok: true, mudou: true, dados, msg: falta ? `${total} de ${somar(np.naves)} naves adicionadas a ${a.nome}; o resto não entrou: ${falta}` : `${total} naves adicionadas a ${a.nome} em ${a.starbase.nome}` };
		}

		if (cmd.acao === 'frotas_desfazer') {
			const fleetPk = new solanaWeb3.PublicKey(pk); const fleetShips = sisPdaFleetShips(fleetPk);
			let nome = '', label = null, starbase = null, devolvidas = [];
			const existe = await solanaReadConnection.getAccountInfo(fleetPk, 'confirmed');
			if (existe) {
				const livre = await sisFrotaLivre(pk, 180000); if (livre) return { ok: false, msg: livre };
				const a = await sisFrotaAtracada(pk, p.atracar !== false); if (a.erro) return { ok: false, msg: a.erro, dados: { nome: a.nome || '' } };
				nome = a.nome; label = Array.from(a.acc.fleetLabel); starbase = a.starbase;
				const sbp = await getStarbasePlayer(userProfileAcct, starbase.publicKey); if (!sbp) return { ok: false, msg: `${nome}: sem conta na starbase` };
				sisEvento('info', nome, `desfazendo a frota em ${starbase.nome}`);
				const ix = { instruction: await sisProgFrotas().methods.disbandFleet({ keyIndex: k }).accountsStrict({
					gameAccountsAndProfile: { gameAndProfileAndFaction: { key: userPublicKey, profile: userProfileAcct, profileFaction: userProfileFactionAcct.publicKey, gameId: sageGameAcct.publicKey }, gameState: sageGameAcct.account.gameState },
					funder: userPublicKey, disbandedFleet: sisPdaLabel('DisbandedFleet', label), fleet: fleetPk, fleetShips, cargoHold: a.acc.cargoHold, fuelTank: a.acc.fuelTank, ammoBank: a.acc.ammoBank,
					starbaseAndStarbasePlayer: { starbase: starbase.publicKey, starbasePlayer: sbp.publicKey }, cargoProgram: cargoProgramPK, systemProgram: solanaWeb3.SystemProgram.programId,
				}).remainingAccounts([{ pubkey: starbase.publicKey, isSigner: false, isWritable: false }]).instruction() };
				const e = await sisTxFrota(ix, 'DISBAND FLEET', nome);
				if (e) return { ok: false, msg: `${nome}: a frota não foi desfeita: ${e}`, dados: { nome } };
				await sisEsperarChain(async () => !(await solanaReadConnection.getAccountInfo(fleetPk, 'confirmed')));
			} else {
				// retomada: a frota já foi desfeita antes, mas sobraram naves na frota desfeita (DisbandedFleet) — precisa do nome
				const lb = sisLabel(p.nome); if (lb.erro) return { ok: false, msg: 'frota não encontrada na blockchain' };
				if (!sisPdaLabel('Fleet', lb.label).equals(fleetPk)) return { ok: false, msg: 'frota não encontrada na blockchain' };
				const di = await solanaReadConnection.getAccountInfo(sisPdaLabel('DisbandedFleet', lb.label), 'confirmed');
				if (!di) return { ok: true, msg: `${lb.nome} já tinha sido desfeita`, dados: { frota: pk, nome: lb.nome, naves: [] } };
				nome = lb.nome; label = lb.label; starbase = await sisStarbasePorPk(new solanaWeb3.PublicKey(di.data.subarray(73, 105)));
				sisEvento('info', nome, `continuando: devolvendo à starbase as naves que ficaram na frota desfeita`);
			}
			// naves de volta para a starbase (sempre a última da lista: as anteriores não mudam de posição)
			const disbanded = sisPdaLabel('DisbandedFleet', label);
			for (let passo = 0; passo < 60; passo++) {
				const l = await sisNavesDaFrota(fleetShips);
				if (!l || !l.length) break;
				const ult = l[l.length - 1];
				const hg = await sisHangar(starbase.publicKey);
				const q = Math.min(ult.qtd, 65535);
				// shipEscrowIndex nulo: o programa acha (ou cria) a entrada da nave na starbase — igual aos testes do SDK
				const ix = { instruction: await sisProgFrotas().methods.disbandedFleetToEscrow({ shipAmount: q, shipEscrowIndex: null, fleetShipInfoIndex: ult.i, keyIndex: k }).accountsStrict({
					gameAccountsAndProfile: { gameAndProfileAndFaction: { key: userPublicKey, profile: userProfileAcct, profileFaction: userProfileFactionAcct.publicKey, gameId: sageGameAcct.publicKey }, gameState: sageGameAcct.account.gameState },
					funder: userPublicKey, disbandedFleet: disbanded, fleetShips, starbaseAndStarbasePlayer: { starbase: starbase.publicKey, starbasePlayer: hg.sbp }, ship: new solanaWeb3.PublicKey(ult.ship), systemProgram: solanaWeb3.SystemProgram.programId,
				}).instruction() };
				const e = await sisTxFrota(ix, 'SHIPS TO STARBASE', nome);
				if (e) return { ok: false, mudou: true, msg: `${nome}: frota desfeita, mas ${l.reduce((t, x) => t + x.qtd, 0)} naves ainda não voltaram para ${starbase.nome} (${e}). Mande desfazer de novo para continuar.`, dados: { frota: pk, nome, naves: devolvidas } };
				devolvidas.push({ ship: ult.ship, qtd: q });
				await sisEsperarChain(async () => { const n = (await sisNavesDaFrota(fleetShips)) || []; return n.length < l.length || (n[ult.i] && n[ult.i].qtd < ult.qtd); });
			}
			const ix = { instruction: await sisProgFrotas().methods.closeDisbandedFleet({ keyIndex: k }).accountsStrict({ key: userPublicKey, playerProfile: userProfileAcct, fundsTo: userPublicKey, disbandedFleet: disbanded, fleetShips }).instruction() };
			const e = await sisTxFrota(ix, 'CLOSE DISBANDED', nome);
			const total = somar(devolvidas);
			const dados = { frota: pk, nome, coords: starbase.coords, starbase: starbase.nome, naves: devolvidas };
			return { ok: true, mudou: true, dados, msg: `${nome} desfeita em ${starbase.nome}: ${total} naves de volta à starbase` + (e ? ` (a conta da frota desfeita não fechou: ${e}; o aluguel fica preso até fechar)` : '') };
		}
		return { ok: false, msg: 'comando desconhecido' };
	}
	async function sisSyncAgora() { clearTimeout(sisTimer); for (let i = 0; i < 20 && sisSincronizando; i++) await wait(250); await sisSync(); }

	// ---------- configuração: validação local, diferenças, aceite e aplicação
	function sisCampo(nome, v, s, onde) {
		const [tipo, padrao, min, max] = s;
		if (v === undefined || v === null) return padrao;
		if (tipo === 'bool') return v === true || v === 1 || v === '1' || v === 'true';
		if (tipo === 'int') { const n = Math.round(Number(v)); if (!isFinite(n) || n < min || n > max) throw new Error(`${onde}: ${nome} fora do limite`); return n; }
		if (tipo === 'str') { const t = String(v); if (t.length > max) throw new Error(`${onde}: ${nome} muito longo`); return t; }
		if (tipo === 'coord') { const t = String(v); if (t === '') return ''; if (!/^-?\d{1,3},-?\d{1,3}$/.test(t)) throw new Error(`${onde}: coordenada inválida em ${nome}`); return t; }
		if (tipo === 'mint') { const t = String(v); if (t === '') return ''; if (!cargoItems.some(i => i.token === t)) throw new Error(`${onde}: item desconhecido em ${nome}`); return t; }
		if (tipo === 'num') { const n = Number(v); if (!isFinite(n) || n < min || n > max) throw new Error(`${onde}: ${nome} fora do limite`); return n; }
		if (tipo === 'enum') {
			const ops = nome === 'reserva' ? ['', 'Supply', 'Garimpo'] : nome === 'assignment' ? SIS_TAREFAS : nome === 'scanPattern' ? [''].concat(scanningPatterns) : nome === 'scanBlockPattern' ? scanningPatterns : nome === 'special' ? SIS_ESPECIAIS : nome === 'craftModo' ? SIS_CRAFT_MODOS : (nome === 'supplyReabastecer' || nome === 'reabastecer') ? ['origem', 'destino'] : [];
			if (!ops.includes(String(v))) throw new Error(`${onde}: valor inválido em ${nome}`); return String(v);
		}
		throw new Error('campo desconhecido');
	}
	// [1.5.4] condição de uma linha (Abastecer/Garimpar) ou regra de crafting: {modo: 'e'|'ou', itens: [{mint, sb, op, qtd}]} — só o cérebro usa
	function sisCond(c, onde) {
		if (!c || !Array.isArray(c.itens)) return null;
		const itens = c.itens.slice(0, 6).map((x, j) => ({ mint: sisCampo('mint', x && x.mint, ['mint', ''], `${onde}, condição ${j + 1}`), sb: sisCampo('sb', x && x.sb, ['coord', ''], `${onde}, condição ${j + 1}`),
			op: x && x.op === '>' ? '>' : '<', qtd: sisCampo('qtd', x && x.qtd, ['int', 0, 0, 1e11], `${onde}, condição ${j + 1}`) })).filter((x) => x.mint && x.qtd > 0);
		return itens.length ? { modo: c.modo === 'ou' ? 'ou' : 'e', itens } : null;
	}
	function sisValidarConfig(d) {
		if (!d || typeof d !== 'object') throw new Error('configuração vazia');
		const o = { global: {}, frotas: {}, crafts: [] };
		for (const k in SIS_GLOBAL) o.global[k] = sisCampo(k, d.global ? d.global[k] : undefined, SIS_GLOBAL[k], 'Ajustes');
		const alvos = validTargets.map(t => t.x + ',' + t.y);
		for (const pk in (d.frotas || {})) {
			if (!/^[1-9A-HJ-NP-Za-km-z]{32,44}$/.test(pk)) throw new Error('frota inválida');
			const f = d.frotas[pk]; const nome = String(f.nome || pk).slice(0, 64); const x = { nome };
			for (const k in SIS_FROTA) x[k] = sisCampo(k, f[k], SIS_FROTA[k], nome);
			x.supply = (Array.isArray(f.supply) ? f.supply : []).slice(0, 20).map((l, i) => {
				const y = {}; for (const k in SIS_SUPPLY) y[k] = sisCampo(k, l ? (k === 'reabastecer' && !l[k] ? x.supplyReabastecer : l[k]) : undefined, SIS_SUPPLY[k], `${nome}, abastecimento ${i + 1}`);
				y.cond = sisCond(l && l.cond, `${nome}, abastecimento ${i + 1}`); // [1.5.4] condição da linha
				if (!alvos.includes(y.origem) || !alvos.includes(y.destino)) throw new Error(`${nome}: abastecimento ${i + 1} com starbase fora da sua facção`);
				return y;
			}).filter((y) => y.mint);
			x.garimpo = (Array.isArray(f.garimpo) ? f.garimpo : []).slice(0, 20).map((l, i) => {
				const y = {}; for (const k in SIS_GARIMPO) y[k] = sisCampo(k, l ? l[k] : undefined, SIS_GARIMPO[k], `${nome}, garimpo ${i + 1}`);
				y.cond = sisCond(l && l.cond, `${nome}, garimpo ${i + 1}`); // [1.5.3/1.5.4] condição da linha
				if (!y.mint) return y;
				const it = cargoItems.find((c) => c.token === y.mint);
				if (!it || !SIS_MINERAVEIS.includes(it.name)) throw new Error(`${nome}: garimpo ${i + 1} com item que não é minerável`);
				if (!alvos.includes(y.setor) || !alvos.includes(y.destino)) throw new Error(`${nome}: garimpo ${i + 1} com starbase fora da sua facção`);
				return y;
			}).filter((y) => y.mint);
			// [1.5.2] tarefa reserva: só com principal Abastecer/Garimpar/Prospectar e com linhas da reserva
			if (!['Supply', 'Garimpo', 'Prospectar'].includes(x.assignment) || x.reserva === x.assignment) x.reserva = '';
			if ((x.reserva === 'Supply' && !x.supply.length) || (x.reserva === 'Garimpo' && !x.garimpo.length)) throw new Error(`${nome}: tarefa reserva sem linhas`);
			if (x.assignment === 'Supply') { if (!x.supply.length) throw new Error(`${nome}: abastecimento sem linhas`); }
			else if (x.assignment === 'Garimpo') { if (!x.garimpo.length) throw new Error(`${nome}: garimpo sem recursos`); }
			else if (x.assignment) {
				if (!alvos.includes(x.starbase)) throw new Error(`${nome}: a starbase não é da sua facção`);
				if (x.assignment !== 'Scan' && x.assignment !== 'Prospectar' && !alvos.includes(x.dest)) throw new Error(`${nome}: destino sem starbase`);
				if ((x.assignment === 'Scan' || x.assignment === 'Prospectar') && !x.dest) throw new Error(`${nome}: destino vazio`);
			}
			o.frotas[pk] = x;
		}
		const receitas = craftRecipes.map(r => r.name).concat(upgradeRecipes.map(r => r.name));
		(d.crafts || []).slice(0, o.global.craftingJobs).forEach((c, i) => {
			const x = {}; for (const k in SIS_CRAFT) x[k] = sisCampo(k, c ? c[k] : undefined, SIS_CRAFT[k], 'craft' + (i + 1));
			if (x.item && !receitas.includes(x.item)) throw new Error(`craft${i + 1}: receita desconhecida`);
			if (x.item && x.coordinates && !alvos.includes(x.coordinates)) throw new Error(`craft${i + 1}: starbase não é da sua facção`);
			o.crafts.push(x);
		});
		o.craftRegras = (Array.isArray(d.craftRegras) ? d.craftRegras : []).slice(0, 30).map((r, i) => {
			const x = {}; for (const k in SIS_REGRA) x[k] = sisCampo(k, r ? r[k] : undefined, SIS_REGRA[k], `Regra de craft ${i + 1}`);
			if (!receitas.includes(x.receita)) throw new Error(`Regra de craft ${i + 1}: receita desconhecida`);
			if (!alvos.includes(x.starbase)) throw new Error(`Regra de craft ${i + 1}: starbase não é da sua facção`);
			x.reservas = (Array.isArray(r.reservas) ? r.reservas : []).slice(0, 10).map((y) => ({ mint: sisCampo('mint', y.mint, ['mint', ''], 'reserva'), qtd: sisCampo('qtd', y.qtd, ['int', 0, 0, 1e9], 'reserva') }));
			x.cond = sisCond(r.cond, `Regra de craft ${i + 1}`); // [1.5.4] condição da regra
			return x;
		});
		return o;
	}
	// plano aceito: o que o cérebro da VPS pode pedir (o script recusa ordens fora disto)
	function sisMontarPlano(n) {
		const plano = { frotas: {}, craft: { receitas: [], coords: [], crewMax: 0, jobs: n.global.craftingJobs, ativo: n.global.craftModo === 'regras' } };
		const planoDe = (f, t) => {
			if (t === 'Supply') return { tarefa: 'Supply', coords: [...new Set((f.supply || []).flatMap((l) => [l.origem, l.destino]))], mints: [...new Set((f.supply || []).map((l) => l.mint))], movimento: f.subwarpPref };
			if (t === 'Garimpo') return { tarefa: 'Garimpo', coords: [...new Set((f.garimpo || []).flatMap((l) => [l.setor, l.destino]))], mints: [...new Set((f.garimpo || []).map((l) => l.mint))],
				minerar: [...new Set((f.garimpo || []).map((l) => `${l.setor}|${l.mint}`))], entregas: [...new Set((f.garimpo || []).map((l) => l.destino))], movimento: f.subwarpPref };
			if (t === 'Prospectar') return { tarefa: 'Prospectar', coords: [f.starbase], centro: f.dest, raio: f.prospRaio, movimento: f.subwarpPref };
			return null;
		};
		for (const pk in n.frotas) {
			const f = n.frotas[pk]; const pl = planoDe(f, f.assignment); if (!pl) continue;
			// [1.5.2] tarefa reserva: as ordens também valem pelas linhas da reserva (o cérebro usa quando a principal não tem nada a fazer)
			if (f.reserva && f.reserva !== f.assignment) pl.reserva = planoDe(f, f.reserva);
			plano.frotas[pk] = pl;
		}
		const nomes = new Set(n.craftRegras.map((r) => r.receita));
		const entradas = new Set();
		// encadeamento: receitas dos ingredientes, até 4 níveis (1.2.4; ex.: Electronics ← Polymer ← Hydrocarbon)
		for (let nivel = 0; nivel < 4; nivel++) {
			for (const r of craftRecipes.concat(upgradeRecipes)) if (nomes.has(r.name)) (r.input || []).forEach((e) => entradas.add(String(e.mint)));
			for (const r of craftRecipes) if (r.output && entradas.has(String(r.output.mint))) nomes.add(r.name);
		}
		plano.craft.receitas = [...nomes];
		plano.craft.coords = [...new Set(n.craftRegras.map((r) => r.starbase))];
		plano.craft.crewMax = Math.max(0, ...n.craftRegras.map((r) => r.tripulacao));
		return plano;
	}
	function sisValorLegivel(k, v) {
		if (v === '' || v === undefined || v === null) return '—';
		if (typeof v === 'boolean') return v ? 'sim' : 'não';
		if (/Resource\d?$|^mineResource$/.test(k)) return (cargoItems.find(i => i.token === v) || {}).name || v;
		if (k === 'starbase' || k === 'coordinates' || (k === 'dest')) { const t = validTargets.find(t => (t.x + ',' + t.y) === v); return t ? `${t.name} [${v}]` : `[${v}]`; }
		if (k === 'subwarpPref') return ['Warp', 'Subwarp', 'Warp(SB)/Subwarp', 'Warp-Subwarp-Warp'][v] || v;
		if (k === 'assignment' || k === 'reserva') return { '': 'nenhuma', Mine: 'Minerar', Transport: 'Transportar', Scan: 'Scan (motor do SLYA)', Supply: 'Abastecer', Garimpo: 'Garimpar', Prospectar: 'Prospectar SDU' }[v] || v;
		if (k === 'supplyReabastecer') return { origem: 'na origem do recurso', destino: 'na base destino' }[v] || v;
		return String(v).replace(/\n/g, ', ');
	}
	async function sisDiferencas(n) {
		const linhas = [];
		const g = JSON.parse(await GM.getValue(settingsGmKey, '{}'));
		for (const k in SIS_GLOBAL) { const a = g[k] === undefined ? SIS_GLOBAL[k][1] : g[k]; if (String(a) !== String(n.global[k])) linhas.push(`Ajustes · ${SIS_ROTULOS[k] || k}: ${sisValorLegivel(k, a)} → ${sisValorLegivel(k, n.global[k])}`); }
		for (const f of userFleets) {
			const pk = f.publicKey.toString(); const a = JSON.parse(await GM.getValue(pk, '{}')); const x = n.frotas[pk] || { assignment: '' };
			for (const k in SIS_FROTA) {
				let va = a[k] === undefined ? SIS_FROTA[k][1] : a[k]; const vn = x[k] === undefined ? SIS_FROTA[k][1] : x[k];
				if (a.sisCerebro && k === 'assignment') va = a.sisTarefa || 'Supply';
				if (SIS_CEREBRO.includes(x.assignment) && ((x.assignment !== 'Prospectar' && ['starbase', 'dest'].includes(k)) || k === 'mineResource' || /^(transport|scan)/.test(k))) continue;
				if (x.assignment !== 'Prospectar' && /^prosp/.test(k)) continue;
				if (a.sisCerebro && a.sisDe && ['starbase', 'dest'].includes(k)) va = a.sisDe[k] ?? va;
				if (String(va) !== String(vn) && (x.assignment || k === 'assignment')) linhas.push(`${f.label} · ${SIS_ROTULOS[k] || k}: ${sisValorLegivel(k, va)} → ${sisValorLegivel(k, vn)}`);
			}
		}
		const nomeIt = (m) => (cargoItems.find(i => i.token === m) || {}).name || m;
		const extras = (l) => `${l.critico ? ' crítico ' + l.critico : ''}${l.origemCritico ? ' crítico na origem ' + l.origemCritico : ''}${l.movimento >= 0 ? ' · ' + sisValorLegivel('subwarpPref', l.movimento) : ''}${l.ativo === false ? ' (desligada)' : ''}`;
		const descSupply = (l) => `${nomeIt(l.mint)}: ${sisValorLegivel('starbase', l.origem)} (mín. ${l.origemMin}) → ${sisValorLegivel('starbase', l.destino)} mín. ${l.min}${l.max ? ' máx. ' + l.max : ''}${extras(l)}${l.cond ? ` · só se ${l.cond.itens.map((c) => `${nomeIt(c.mint)} em ${sisValorLegivel('starbase', c.sb || l.destino)} ${c.op === '>' ? 'acima' : 'abaixo'} de ${c.qtd}`).join(l.cond.modo === 'ou' ? ' OU ' : ' E ')}` : ''}`;
		const descGarimpo = (l) => `${nomeIt(l.mint)}: minerar em ${sisValorLegivel('starbase', l.setor)} → entregar em ${sisValorLegivel('starbase', l.destino)} mín. ${l.min}${l.max ? ' máx. ' + l.max : ''}${extras(l)}${l.cond ? ` · só se ${l.cond.itens.map((c) => `${nomeIt(c.mint)} em ${sisValorLegivel('starbase', c.sb || l.destino)} ${c.op === '>' ? 'acima' : 'abaixo'} de ${c.qtd}`).join(l.cond.modo === 'ou' ? ' OU ' : ' E ')}` : ''}`;
		for (const f of userFleets) {
			const pk = f.publicKey.toString(); const x = n.frotas[pk]; if (!x || !SIS_CEREBRO.includes(x.assignment)) continue;
			const antF = (sisLocal.ultimaCfg && sisLocal.ultimaCfg.frotas && sisLocal.ultimaCfg.frotas[pk]) || {};
			if ((x.assignment === 'Supply' || x.reserva === 'Supply') && JSON.stringify(antF.supply || []) !== JSON.stringify(x.supply)) x.supply.forEach((l) => linhas.push(`${f.label} · abastecer ${descSupply(l)}`));
			if ((x.assignment === 'Garimpo' || x.reserva === 'Garimpo') && JSON.stringify(antF.garimpo || []) !== JSON.stringify(x.garimpo)) x.garimpo.forEach((l) => linhas.push(`${f.label} · garimpar ${descGarimpo(l)}`));
		}
		const regrasAnt = (sisLocal.ultimaCfg && sisLocal.ultimaCfg.craftRegras) || [];
		if (JSON.stringify(regrasAnt) !== JSON.stringify(n.craftRegras)) n.craftRegras.forEach((r, i) => linhas.push(`Regra de craft ${i + 1}${r.ativo ? '' : ' (desligada)'}: ${r.lote} × ${r.receita} em ${sisValorLegivel('starbase', r.starbase)}${r.produtoMin ? ' · se produto < ' + r.produtoMin : ''}${r.produtoMax ? ' · até ' + r.produtoMax : ''}${r.lucroPct >= 0 ? ' · lucro ≥ ' + r.lucroPct + '%' : ''}${r.lucroUsd >= 0 ? ' · lucro ≥ US$ ' + r.lucroUsd : ''}${r.encadear ? ' · encadeia' : ''}${r.cond ? ` · só se ${r.cond.itens.map((c) => `${(cargoItems.find((it) => it.token === c.mint) || {}).name || c.mint} em ${sisValorLegivel('starbase', c.sb || r.starbase)} ${c.op === '>' ? 'acima' : 'abaixo'} de ${c.qtd}`).join(r.cond.modo === 'ou' ? ' OU ' : ' E ')}` : ''}`));
		for (const pk in n.frotas) if (!userFleets.some(f => f.publicKey.toString() === pk) && n.frotas[pk].assignment) linhas.push(`${n.frotas[pk].nome}: frota não existe neste navegador (será ignorada)`);
		for (let i = 0; i < n.global.craftingJobs; i++) {
			const a = JSON.parse(await GM.getValue('craft' + (i + 1), '{}')); const x = n.crafts[i] || {};
			for (const k in SIS_CRAFT) { const va = a[k] === undefined || a[k] === '' ? SIS_CRAFT[k][1] : a[k]; const vn = x[k] === undefined ? SIS_CRAFT[k][1] : x[k]; if (String(va) !== String(vn)) linhas.push(`craft${i + 1} · ${SIS_ROTULOS[k] || k}: ${sisValorLegivel(k, va)} → ${sisValorLegivel(k, vn)}`); }
		}
		return linhas;
	}
	async function sisReceberConfig(cfg) {
		if (sisAplicando || (sisPendente && sisPendente.versao === cfg.versao) || +sisLocal.recusada === +cfg.versao) return;
		const h = await sisSha256(String(cfg.dados));
		if (h !== cfg.hash) { sisEvento('erro', '', `Configuração v${cfg.versao} ignorada: o hash não confere`); return; }
		let dados = null, erro = '', diff = [];
		try { dados = sisValidarConfig(JSON.parse(cfg.dados)); diff = await sisDiferencas(dados); } catch (e) { erro = e.message || String(e); }
		sisPendente = { versao: +cfg.versao, hash: cfg.hash, dados, erro, diff, resumo: String(cfg.resumo || ''), autor: String(cfg.autor || ''), criado_em: String(cfg.criado_em || ''), espelho: !!cfg.espelho };
		sisAtualizarUI();
		// [1.4.0] versão já aceita em outro navegador do perfil (primário): aplica sozinha se a opção local estiver ligada
		if (cfg.espelho && !erro && sisLocal.espelharCfg !== false) {
			sisEvento('info', '', `Configuração v${cfg.versao} já aceita no navegador primário: aplicando neste navegador (espelho).`);
			await sisAplicar(); return;
		}
		// [1.5.2] só mudou o que está com "Aceite no navegador" desligado na configuração JÁ aceita (frotas e/ou crafting): aplica sozinha
		const auto = !erro && dados ? sisAutoAplicavel(sisLocal.ultimaCfg, dados) : '';
		if (auto) {
			sisEvento('info', '', `Configuração v${cfg.versao}: só mudou ${auto} (sem aceite no navegador) — aplicando sozinha.`);
			await sisAplicar(); return;
		}
		sisAbrirAceite();
	}
	// [1.5.2] a mudança é só em partes sem aceite? Decide pela configuração aceita (a), nunca pela nova (n): mudar as chaves sempre pede aceite.
	const SIS_GLOBAL_CRAFT = ['craftingTxMultiplier', 'craftingTxAffectsAutoFee', 'craftingJobs', 'craftColetar', 'craftModo'];
	function sisAutoAplicavel(a, n) {
		if (!a || a.completa !== true || !a.global || !n || !n.global) return '';
		const fr = a.global.aceiteFrotas === false, cr = a.global.aceiteCrafting === false; if (!fr && !cr) return '';
		const igual = (x, y) => JSON.stringify(x === undefined ? null : x) === JSON.stringify(y === undefined ? null : y);
		const partes = new Set();
		for (const k of new Set([...Object.keys(a.global), ...Object.keys(n.global)])) { if (igual(a.global[k], n.global[k])) continue; if (!cr || !SIS_GLOBAL_CRAFT.includes(k)) return ''; partes.add('o crafting'); }
		if (!igual(a.frotas || {}, n.frotas || {})) { if (!fr) return ''; partes.add('as frotas'); }
		for (const k of ['crafts', 'craftRegras']) if (!igual(a[k] || [], n[k] || [])) { if (!cr) return ''; partes.add('o crafting'); }
		return [...partes].join(' e ');
	}
	async function sisResponderAceite(aceito, resultado) {
		const p = sisPendente; if (!p) return false;
		const r = await sisPost('aceite', { perfil: userProfileAcct.toString(), versao: p.versao, hash: p.hash, aceito, resultado, espelho: !!p.espelho, inst: sisInstCorpo(false) });
		if (r.status !== 200 && p.espelho) { sisEvento('erro', '', `Espelho da v${p.versao}: o dashboard não confirmou (${(r.j && r.j.erro) || r.status}).`); return false; }
		if (r.status !== 200) { alert('Sis-Bot: não consegui avisar o dashboard (' + ((r.j && r.j.erro) || r.status) + '). Tente de novo.'); return false; }
		return true;
	}
	async function sisRecusar() {
		const p = sisPendente; if (!p) return;
		if (!(await sisResponderAceite(false, { motivo: p.erro || 'recusada pelo membro' }))) return;
		sisLocal.recusada = p.versao; await sisSalvarLocal();
		sisPendente = null; sisFecharAceite(); sisAtualizarUI();
	}
	// fleets/crafts no meio de dock, carga, descarga ou reabastecimento terminam essa etapa antes da troca
	async function sisEsperarEtapas(msg, limiteMs = 300000) {
		const ocupado = ['Mine Starting', 'Mining Stop', 'Unloading', 'Loading', 'Docking', 'Undocking', 'Refueling'];
		const t0 = Date.now();
		while (enableAssistant && Date.now() - t0 < limiteMs) {
			const f = userFleets.filter(x => ocupado.some(v => String(x.state || '').startsWith(v))).map(x => x.label);
			for (let i = 1; i < globalSettings.craftingJobs + 1; i++) {
				const c = JSON.parse(await GM.getValue('craft' + i, '{}'));
				if (['Starting:', 'Upgrade Starting', 'Completing:', 'Upgrade Completing'].some(v => String(c.state || '').startsWith(v))) f.push('craft' + i);
			}
			if (!f.length) return;
			setTxt(msg, `Esperando terminar a etapa em andamento: ${f.join(', ')}…`);
			await wait(3000);
		}
	}
	// Aplica sem recarregar a página: grava no GM e atualiza na memória as frotas, os crafts e os ajustes, como o SLYA fazia ao salvar.
	// Só recarrega quando o próprio SLYA exigiria (mais trabalhos de crafting com o Sis-Bot rodando ou mudança nas frotas ignoradas).
	async function sisAplicar() {
		const p = sisPendente; if (!p || !p.dados || sisAplicando) return;
		sisAplicando = true;
		const msg = document.querySelector('#sisAceiteMsg');
		const padraoAnt = globalSettings.scanBlockPattern, compAnt = globalSettings.scanBlockLength;
		try {
			await sisEsperarEtapas(msg);
			setTxt(msg, 'Aplicando…');
			const n = p.dados; const resultado = {};
			const g = JSON.parse(await GM.getValue(settingsGmKey, '{}'));
			const jobsAnt = globalSettings.craftingJobs, excluirAnt = String(globalSettings.excludeFleets || '').trim();
			for (const k in SIS_GLOBAL) g[k] = n.global[k];
			const gravar = [[settingsGmKey, JSON.stringify(g)]]; // tudo é calculado antes; só grava se nada falhar
			const memoria = [];
			globalSettings.scanBlockPattern = n.global.scanBlockPattern; globalSettings.scanBlockLength = n.global.scanBlockLength;
			for (const fl of userFleets) {
				const pk = fl.publicKey.toString();
				const atual = JSON.parse(await GM.getValue(pk, '{}'));
				const c = n.frotas[pk] || { assignment: '' };
				const novo = Object.assign({}, atual, { name: fl.label });
				for (const k in SIS_FROTA) novo[k] = c[k] === undefined ? SIS_FROTA[k][1] : c[k];
				// abastecer e garimpar são decididos pelo Sis-Bot: o motor local do SLYA não opera essa frota
				novo.sisCerebro = sisEhCerebro(novo); novo.sisTarefa = novo.sisCerebro ? novo.assignment : '';
				novo.sisDe = novo.sisCerebro ? { starbase: novo.starbase, dest: novo.dest } : null; // só para mostrar as diferenças no próximo aceite
				if (novo.sisCerebro) { novo.assignment = ''; novo.starbase = ''; novo.dest = ''; novo.mineResource = ''; }
				// [1.5.0] frota em missão: a tarefa do motor local fica guardada (volta quando a missão terminar)
				const emMissao = !!atual.sisMissaoDe && sisEmMissao.has(pk);
				if (!emMissao) delete novo.sisMissaoDe;
				let moveType = novo.subwarpPref == 1 ? 'subwarp' : (novo.subwarpPref == 2 ? 'warpsubwarp' : (novo.subwarpPref == 3 ? 'warp-subwarp-warp' : 'warp'));
				novo.scanBlock = [];
				if (novo.assignment) {
					const sb = ConvertCoords(novo.starbase), ds = ConvertCoords(novo.dest);
					const warpCost = calcWarpFuelReq(fl, sb, ds, moveType == 'warpsubwarp');
					if (warpCost > fl.fuelCapacity) {
						const subwarpCost = calculateSubwarpFuelBurn(fl, calculateMovementDistance(sb, ds));
						if (subwarpCost * 2 > fl.fuelCapacity) { resultado[pk] = `${fl.label}: combustível não alcança ida e volta — ficou sem tarefa`; novo.assignment = ''; }
						else { moveType = 'subwarp'; resultado[pk] = `${fl.label}: warp não alcança — usando subwarp`; }
					}
					if (novo.dest) novo.scanBlock = buildScanBlock(ds[0], ds[1], novo.scanPattern, novo.scanPatternLength);
				}
				novo.moveType = moveType;
				// rota nova: a frota não volta para a rota antiga. Um warp já iniciado termina (a SAGE não cancela warp);
				// na chegada, o transporte segue para a starbase NOVA (carrega pelo manifesto novo) e o minerador/scanner decide pela rota nova.
				const mudouRota = atual.dest !== novo.dest || atual.starbase !== novo.starbase || atual.assignment !== novo.assignment;
				if (mudouRota) novo.moveTarget = novo.assignment === 'Transport' ? novo.starbase : '';
				if (!resultado[pk] && n.frotas[pk]) resultado[pk] = 'ok';
				if (emMissao && novo.assignment && !novo.sisCerebro) { novo.sisMissaoDe = { assignment: novo.assignment }; novo.assignment = ''; }
				else if (emMissao) delete novo.sisMissaoDe;
				gravar.push([pk, JSON.stringify(novo)]);
				memoria.push([fl, novo, mudouRota]);
			}
			globalSettings.scanBlockPattern = padraoAnt; globalSettings.scanBlockLength = compAnt;
			for (const pk in n.frotas) if (!resultado[pk]) resultado[pk] = `${n.frotas[pk].nome}: frota não encontrada neste navegador`;
			for (let i = 1; i <= n.global.craftingJobs; i++) {
				const atual = JSON.parse(await GM.getValue('craft' + i, '{}'));
				if (n.global.craftModo === 'regras') { // trabalhos passam a ser definidos pelo cérebro; os que estão produzindo continuam até terminar
					gravar.push(['craft' + i, JSON.stringify(Object.assign({}, atual, { label: 'craft' + i, state: atual.state || 'Idle', item: atual.craftingId ? (atual.item || '') : '', amount: atual.craftingId ? atual.amount : 0, sisUnica: !!atual.craftingId }))]);
					continue;
				}
				const c = n.crafts[i - 1] || {};
				const novo = Object.assign({}, atual, { label: 'craft' + i, state: atual.state || 'Idle', coordinates: c.coordinates || '', crew: c.crew || 0, item: c.item || '',
					amount: c.amount || 0, belowAmount: c.belowAmount || '', special: c.special || '', feeAtlas: atual.feeAtlas || 0, craftingCoords: atual.craftingCoords || '', craftingId: atual.craftingId || 0 });
				gravar.push(['craft' + i, JSON.stringify(novo)]);
			}
			for (const [k, v] of gravar) await GM.setValue(k, v);
			// memória: o loop de cada frota relê a tarefa do GM a cada volta; rota, scan e movimento ficam em userFleets
			for (const [fl, x, mudouRota] of memoria) {
				Object.assign(fl, { mineResource: x.mineResource, destCoord: x.dest, starbaseCoord: x.starbase, moveType: x.moveType, scanBlock: x.scanBlock,
					scanMin: x.scanMin || 15, scanMin2: x.scanMin2, scanMin3: x.scanMin3, scanSearchDist: x.scanSearchDist || 15, scanClusterFactor: x.scanClusterFactor || 50,
					scanNeighborhoodMinGood: x.scanNeighborhoodMinGood || 3, scanCheckWhileCooldownLeft: x.scanCheckWhileCooldownLeft, scanBypassPercent: x.scanBypassPercent || 4,
					scanHomeAtPercent: x.scanHomeAtPercent, scanPattern: x.scanPattern, scanPatternLength: x.scanPatternLength, scanMove: x.scanMove });
				if (mudouRota) { fl.moveTarget = x.moveTarget || ''; fl.scanBlockIdx = 0; fl.scanAutoMoveTo = null; fl.resupplying = false; }
				else if (!x.scanMove) fl.scanBlockIdx = 0;
			}
			await loadGlobalSettings();
			if (!globalSettings.automaticFee) currentFee = globalSettings.priorityFee;
			else currentFee = Math.max(globalSettings.automaticFeeMin, Math.min(globalSettings.automaticFeeMax, currentFee));
			sisAtualizarChipTaxa();
			const recarregar = (enableAssistant && n.global.craftingJobs > jobsAnt) || String(n.global.excludeFleets || '').trim() !== excluirAnt;
			if (!(await sisResponderAceite(true, resultado))) { sisAplicando = false; setTxt(msg, 'A configuração foi aplicada, mas o dashboard não confirmou. Ele vai reenviar; aceite de novo.'); return; }
			sisLocal.cfgAceita = { versao: p.versao, hash: p.hash }; sisLocal.recusada = 0;
			sisLocal.plano = sisMontarPlano(n); sisLocal.ultimaCfg = { completa: true, frotas: n.frotas, crafts: n.crafts, craftRegras: n.craftRegras, global: Object.assign({}, n.global) }; // [1.5.2] completa: para aplicar sozinha
			await sisSalvarLocal();
			sisPendente = null;
			sisAtualizarUI();
			sisSyncAgora();
			if (recarregar) {
				await GM.setValue('sisRetomar', enableAssistant ? '1' : '');
				setTxt(msg, 'Aplicada. Esta mudança (frotas ignoradas ou mais trabalhos de crafting) precisa recarregar a página…');
				if (enableAssistant) await toggleAssistant();
				setTimeout(() => location.reload(), 1500);
				return;
			}
			sisAplicando = false;
			setTxt(msg, 'Aplicada sem recarregar a página. O Sis-Bot continua rodando com a configuração nova.');
			setTimeout(sisFecharAceite, 2500);
		} catch (e) {
			sisAplicando = false;
			globalSettings.scanBlockPattern = padraoAnt; globalSettings.scanBlockLength = compAnt;
			sisEvento('erro', '', 'Falha ao aplicar configuração: ' + (e.message || e));
			setTxt(msg, 'Falha ao aplicar: ' + (e.message || e));
		}
	}
	// intenção do membro: iniciar (botão ou comando do dashboard) liga; parar (botão, comando ou teto local) desliga;
	// parada por erro não muda. Depois de recarregar a página o Sis-Bot volta a rodar sozinho se estava ligado.
	function sisMarcarIntencao(acao) {
		if (acao === 'erro') return;
		if (sisInst.paradaSistema && acao === 'parar') { sisInst.paradaSistema = false; return; }
		const v = acao === 'iniciar';
		if (sisLocal.querRodando !== v) { sisLocal.querRodando = v; sisSalvarLocal(); }
	}
	// 1.2.5: ao abrir a página, o plano aceito é remontado com as regras desta versão do script (antes ficava o montado no aceite,
	// por uma versão anterior — a 1.2.4 recusava o Hydrocarbon com o plano montado pela 1.2.3)
	async function sisRemontarPlano() {
		try {
			const u = sisLocal.ultimaCfg; const pl = sisLocal.plano; if (!u || !pl) return;
			const g = u.global || { craftingJobs: pl.craft ? pl.craft.jobs : 0, craftModo: pl.craft && pl.craft.ativo ? 'regras' : 'fixo' };
			sisLocal.plano = sisMontarPlano({ global: g, frotas: u.frotas || {}, craftRegras: u.craftRegras || [] }); await sisSalvarLocal();
		} catch (e) { cLog(1, 'Sis-Bot: remontar o plano aceito', e); }
	}
	async function sisAoIniciar() {
		await sisRemontarPlano();
		const retomar = await GM.getValue('sisRetomar', '');
		if (retomar) await GM.setValue('sisRetomar', '');
		if (!enableAssistant && (retomar || (sisLocal.querRodando && sisLocal.retomarAuto !== false && sisLocal.token))) {
			sisPainelMostrar(); toggleAssistant();
			if (!retomar) sisEvento('info', '', 'Sis-Bot retomado sozinho depois de recarregar a página (estava rodando).');
		}
		sisEvento('info', '', `Sis-Bot ${GM_info.script.version} iniciado com ${userFleets.length} frotas`);
		sisAgendar(2);
	}

	// ---------- painel local (Sis-Ágora) e janela de aceite — só textContent/value, nunca innerHTML com dados
	function sisAtualizarUI() {
		const b = document.querySelector('#sisBtn > span');
		const papelCurto = sisAbaDuplicada ? ' · outra aba' : sisInst.papel === 'reserva' ? ' · reserva' : sisInst.papel === 'bloqueada' ? ' · bloqueado' : sisInst.papel === 'saindo' ? ' · saindo' : sisInst.papel === 'sem_contato' ? ' · sem contato' : '';
		if (b && initComplete) setTxt(b, (sisPendente ? 'Sis-Ágora ●' : 'Sis-Ágora') + papelCurto);
		const ii = document.querySelector('#sisInstInfo');
		if (ii) { setTxt(ii, (() => { try { return customKeypair ? '' : '⚠ assina pela extensão da carteira (não assume sozinho; configure a hot wallet abaixo) · '; } catch (e) { return ''; } })() + sisInstTexto() + (sisInst.reservas ? ` · ${sisInst.reservas} reserva(s) no ar` : '') + (sisInst.max > 1 ? ` · até ${sisInst.max} navegadores liberados` : '')); ii.style.color = sisPrimarioAgora() || sisInst.papel === 'unico' ? 'lightgreen' : (sisInst.papel === 'bloqueada' || sisAbaDuplicada || sisInst.papel === 'sem_contato' ? 'tomato' : '#ffbe4d'); }
		const btn = document.querySelector('#sisBtn');
		if (btn) btn.style.color = sisPendente ? '#ffbe4d' : (sisUltimo.ok === false ? 'tomato' : '');
		const con = document.querySelector('#sisConexao');
		if (con) {
			const quando = sisUltimo.em ? ' · ' + TimeToStr(new Date(sisUltimo.em)) : '';
			setTxt(con, !sisLocal.token ? 'Sem token: cole o token gerado no dashboard.' : (sisUltimo.ok ? `Conectado ao perfil ${sisUltimo.perfil}${quando}` : `Sem conexão: ${sisUltimo.msg}${quando}`));
			con.style.color = sisUltimo.ok ? 'lightgreen' : (sisLocal.token ? 'tomato' : 'inherit');
		}
		setTxt(document.querySelector('#sisCfgInfo'), (sisLocal.cfgAceita ? `Configuração aplicada: v${sisLocal.cfgAceita.versao}` : 'Nenhuma configuração do dashboard aplicada ainda') + (sisPendente ? ` · v${sisPendente.versao} aguardando seu aceite` : '') + (sisLocal.modoLocal ? ' · MODO LOCAL (ignorando a VPS)' : ''));
		setTxt(document.querySelector('#sisFila'), `${sisFila.eventos.length} eventos e ${sisFila.producao.length} registros de produção na fila · ${sisTxHora.length} transações na última hora`);
		const pend = document.querySelector('#sisVerPendente'); if (pend) pend.style.display = sisPendente ? 'inline-block' : 'none';
		const rl = document.querySelector('#sisRpcLista');
		if (rl) setTxt(rl, sisRpcResumo().map(r => `${r.proprio ? '★ ' : ''}${r.nome} [${r.host}, ${r.origem}]: ${r.estado === 'ok' ? 'ok' : 'fora por ' + r.volta_em_s + ' s'}${r.ms ? ' · ' + r.ms + ' ms' : ''} · ${r.ok} ok / ${r.erros} erros${r.sem.length ? ' · sem ' + r.sem.join(', ') : ''}${r.ult && r.estado !== 'ok' ? ' · ' + r.ult : ''}`).join(' | '));
	}
	function sisPreencherPainel() {
		document.querySelector('#sisToken').value = ''; document.querySelector('#sisToken').placeholder = sisLocal.token ? 'token salvo (deixe vazio para manter)' : 'cole aqui o token do dashboard';
		document.querySelector('#sisTokenClear').checked = false;
		document.querySelector('#mySecretKey').value = ''; document.querySelector('#mySecretKey').placeholder = globalSettings.mySecretKey ? 'chave salva (deixe vazio para manter)' : 'nenhuma chave: usa Solflare/Phantom';
		document.querySelector('#mySecretKeyClear').checked = false;
		document.querySelector('#sisRpcs').value = sisRpcProprios.join('\n');
		document.querySelector('#sisTetoTaxa').value = sisLocal.tetoTaxa;
		document.querySelector('#sisTetoMult').value = sisLocal.tetoMultCraft;
		document.querySelector('#sisTetoTx').value = sisLocal.tetoTxHora;
		document.querySelector('#sisComandos').checked = !!sisLocal.comandosRemotos;
		document.querySelector('#sisModoLocal').checked = !!sisLocal.modoLocal;
		document.querySelector('#sisRetomarAuto').checked = sisLocal.retomarAuto !== false;
		document.querySelector('#sisAceitarRpcs').checked = sisLocal.aceitarRpcs !== false;
		document.querySelector('#sisInstNome').value = sisInstLocal.nome || '';
		document.querySelector('#sisEspelhar').checked = sisLocal.espelharCfg !== false;
		document.querySelector('#debugLogLevel').value = globalSettings.debugLogLevel;
		document.querySelector('#statusPanelOpacity').value = globalSettings.statusPanelOpacity;
		document.querySelector('#fleetsPerColumn').value = globalSettings.fleetsPerColumn;
		sisAtualizarUI();
	}
	function sisAbrirPainel() { const m = document.querySelector('#sisModal'); if (!m) return; if (m.style.display === 'none') { sisPreencherPainel(); m.style.display = 'block'; } else m.style.display = 'none'; }
	async function sisSalvarPainel() {
		const tok = document.querySelector('#sisToken').value.trim();
		if (tok && !/^sisbot_[A-Za-z0-9_-]{40,60}$/.test(tok)) { alert('Sis-Bot: token inválido. Copie o token inteiro do dashboard (começa com sisbot_).'); return; }
		const trocouToken = tok && tok !== sisLocal.token;
		if (document.querySelector('#sisTokenClear').checked) sisLocal.token = '';
		if (tok) sisLocal.token = tok;
		const lim = (v, a, b, p) => { const n = parseInt(v); return isNaN(n) ? p : Math.max(a, Math.min(b, n)); };
		sisLocal.tetoTaxa = lim(document.querySelector('#sisTetoTaxa').value, 0, 50000, SIS_PADRAO_LOCAL.tetoTaxa);
		sisLocal.tetoMultCraft = lim(document.querySelector('#sisTetoMult').value, 1, 500, SIS_PADRAO_LOCAL.tetoMultCraft);
		sisLocal.tetoTxHora = lim(document.querySelector('#sisTetoTx').value, 0, 100000, SIS_PADRAO_LOCAL.tetoTxHora);
		sisLocal.comandosRemotos = document.querySelector('#sisComandos').checked;
		sisLocal.modoLocal = document.querySelector('#sisModoLocal').checked;
		sisLocal.retomarAuto = document.querySelector('#sisRetomarAuto').checked;
		sisLocal.espelharCfg = document.querySelector('#sisEspelhar').checked;
		{ const nn = document.querySelector('#sisInstNome').value.replace(/[\x00-\x1F<>]+/g, ' ').trim().slice(0, 40);
		  if (nn && nn !== sisInstLocal.nome) { sisInstLocal.nome = nn; sisInst.renomear = nn; try { await GM.setValue(SIS_INST_KEY, JSON.stringify(sisInstLocal)); } catch (e) { } } }
		{ const antesRpc = sisLocal.aceitarRpcs !== false; sisLocal.aceitarRpcs = document.querySelector('#sisAceitarRpcs').checked;
		  if (antesRpc && !sisLocal.aceitarRpcs) { await sisRpcReceberDash({ versao: '', lista: [] }); } else if (!antesRpc && sisLocal.aceitarRpcs) { sisRpcDash.versao = '-'; } }
		if (trocouToken) { sisLocal.cfgAceita = null; sisLocal.recusada = 0; sisCatalogoEnviar = true; }
		await sisSalvarLocal();
		// [1.3.0] RPCs próprios (só https; até 8). Valem depois de recarregar a página.
		{
			const linhas = document.querySelector('#sisRpcs').value.split(/[\s,]+/).map(x => x.trim()).filter(Boolean);
			const ruins = linhas.filter(u => !/^https:\/\/[^\s"'<>]+$/.test(u));
			if (ruins.length) { alert('Sis-Bot: RPC inválido (use https://…): ' + ruins.map(sisRpcMascarar).join(', ')); return; }
			const novos = [...new Set(linhas)].slice(0, 8);
			if (novos.join('\n') !== sisRpcProprios.join('\n')) {
				await GM.setValue(SIS_RPC_KEY, JSON.stringify(novos));
				alert('Sis-Bot: RPCs salvos. Recarregue a página para usar a nova lista.');
			}
		}
		// campos locais do SLYA (gravados direto nas configurações deste navegador)
		const g = JSON.parse(await GM.getValue(settingsGmKey, '{}'));
		const campo = document.querySelector('#mySecretKey'); const digitado = campo.value.trim();
		const antes = g.mySecretKey || '';
		if (document.querySelector('#mySecretKeyClear').checked) g.mySecretKey = '';
		if (digitado) {
			try { const arr = JSON.parse(digitado); if (!Array.isArray(arr) || arr.length !== 64) throw 0; } catch (e) { campo.value = ''; alert('Sis-Bot: chave inválida. Use o formato [12, 34, ...] com 64 números.'); return; }
			g.mySecretKey = digitado;
		}
		campo.value = '';
		g.debugLogLevel = lim(document.querySelector('#debugLogLevel').value, 0, 9, 3);
		g.statusPanelOpacity = lim(document.querySelector('#statusPanelOpacity').value, 1, 100, 75);
		g.fleetsPerColumn = lim(document.querySelector('#fleetsPerColumn').value, 0, 100, 0);
		await GM.setValue(settingsGmKey, JSON.stringify(g));
		await loadGlobalSettings();
		sisPreencherPainel();
		if ((g.mySecretKey || '') !== antes) alert('Sis-Bot: chave da hot wallet alterada. Recarregue a página para usar a nova chave.');
		sisSyncAgora();
	}
	function sisAbrirAceite() {
		const m = document.querySelector('#sisAceiteModal'); const p = sisPendente; if (!m || !p) return;
		setTxt(document.querySelector('#sisAceiteTitulo'), `Configuração v${p.versao} do dashboard`);
		setTxt(document.querySelector('#sisAceiteInfo'), `Enviada por ${p.autor || '?'} em ${p.criado_em}. ${p.resumo}`);
		setTxt(document.querySelector('#sisAceiteErro'), p.erro ? 'Esta configuração não passou na conferência deste navegador e só pode ser recusada: ' + p.erro : '');
		setTxt(document.querySelector('#sisAceiteMsg'), enableAssistant ? 'Ao aceitar, o Sis-Bot aplica as mudanças sem parar e sem recarregar a página (frotas no meio de dock/carga terminam essa etapa antes).' : 'Ao aceitar, as mudanças são gravadas na hora.');
		const ul = document.querySelector('#sisAceiteLista'); ul.replaceChildren();
		(p.diff.length ? p.diff : ['Nenhuma mudança em relação ao que está neste navegador.']).slice(0, 300).forEach(l => ul.appendChild(celula('li', l)));
		if (p.diff.length > 300) ul.appendChild(celula('li', `… e mais ${p.diff.length - 300} mudanças`));
		document.querySelector('#sisAceitar').disabled = !!p.erro;
		m.style.display = 'block';
	}
	function sisFecharAceite() { const m = document.querySelector('#sisAceiteModal'); if (m) m.style.display = 'none'; }

	// =====================================================================================================
	// [Sis-Ágora] painel de Status e Estatísticas — só DOM e textContent
	// 1.1.9: controles (Start/Stop, Sis-Ágora, Tools) no cabeçalho do painel; abas por tarefa; seções; paginação;
	// 3 modos de exibição; tamanho da fonte; ordem manual (setas ▲▼ ou arrastando a linha pelo ⋮⋮).
	// As preferências ficam só neste navegador (GM 'sisPainelUI'); nada disso vai para o dashboard.
	// =====================================================================================================
	const sisPainel = { linhas: new Map(), tarefas: new Map(), ordens: new Map(), pods: new Map(), rr: 0, filtro: '', filtroCor: '', pagina: 1, pags: 1, assinatura: '', arrastando: null, lendo: new Set(), seguir: '', renderTimer: null };
	const SIS_TAREFA_NOME = { Mine: 'Minerar', Transport: 'Transportar', Scan: 'Scan', Supply: 'Abastecer', Garimpo: 'Garimpar', Prospectar: 'Prospectar SDU' };
	const SIS_COR_NOME = { minerando: 'minerando', viagem: 'em viagem', operando: 'operando', scan: 'escaneando', craft: 'crafting', erro: 'com erro', parado: 'parada', livre: 'livre' };
	// ordem das seções na lista (chave = tarefa da frota; 'Craft' = processos de crafting)
	const SIS_SECOES = [['Supply', 'Abastecer'], ['Garimpo', 'Garimpar'], ['Prospectar', 'Prospectar SDU'], ['Mine', 'Minerar'], ['Transport', 'Transportar'], ['Scan', 'Scan'], ['', 'Sem tarefa'], ['Craft', 'Crafting']];
	const SIS_UI_KEY = 'sisPainelUI';
	const SIS_POR_PAG = [10, 20, 50, 100];
	const sisUI = { modo: 2, fonte: 100, aba: 'todas', porPag: 20, ordem: [], recolhido: false, fechadas: [], telaCheia: false };
	try {
		const g = JSON.parse(await GM.getValue(SIS_UI_KEY, '{}')) || {};
		if ([1, 2, 3].includes(+g.modo)) sisUI.modo = +g.modo;
		if (g.fonte) sisUI.fonte = Math.max(70, Math.min(160, parseInt(g.fonte) || 100));
		if (typeof g.aba === 'string' && (g.aba === 'todas' || SIS_SECOES.some(([k]) => k === g.aba))) sisUI.aba = g.aba;
		if (SIS_POR_PAG.includes(+g.porPag)) sisUI.porPag = +g.porPag;
		if (Array.isArray(g.ordem)) sisUI.ordem = g.ordem.filter((x) => typeof x === 'string').slice(0, 600);
		sisUI.recolhido = !!g.recolhido;
		sisUI.telaCheia = !!g.telaCheia;
		if (Array.isArray(g.fechadas)) sisUI.fechadas = g.fechadas.filter((x) => typeof x === 'string' && SIS_SECOES.some(([k]) => k === x));
	} catch (e) { }
	let sisUITimer = null;
	function sisSalvarUI() { clearTimeout(sisUITimer); sisUITimer = setTimeout(() => { try { GM.setValue(SIS_UI_KEY, JSON.stringify(sisUI)); } catch (e) { } }, 300); }

	function sisEl(tag, cls, txt) { const e = document.createElement(tag); if (cls) e.className = cls; if (txt !== undefined) setTxt(e, txt); return e; }
	const sisFmt = (n, d = 0) => (n === undefined || n === null || isNaN(n)) ? '—' : Number(n).toLocaleString('pt-BR', { maximumFractionDigits: d, minimumFractionDigits: d });
	function sisCorEstado(s) {
		s = String(s || '');
		if (/ERROR|Erro|falh/i.test(s)) return 'erro';
		if (/STOP|Stopp/i.test(s)) return 'parado';
		if (/^Mine|Mining|minera/i.test(s)) return 'minerando';
		if (/Scan/i.test(s)) return 'scan';
		if (/Warp|Subwarp|Move|viagem|salto|chegou/i.test(s)) return 'viagem';
		if (/Load|Unload|Dock|Undock|Refuel|carreg|descarreg|atrac|suprim|abastec|Resupply/i.test(s)) return 'operando';
		if (/Craft|Upgrade|Starting|Completing|Claim/i.test(s)) return 'craft';
		return 'livre';
	}
	// "Mine [23:17]", "Subwarp [2,-23] 23:20" → quanto falta
	function sisFaltam(s) {
		const m = /(?:\[|\s)(\d\d):(\d\d)\]?\s*$/.exec(String(s || '')) || /\[(\d\d):(\d\d)\]/.exec(String(s || ''));
		if (!m) return '';
		const agora = new Date(); const alvo = new Date(agora); alvo.setHours(+m[1], +m[2], 0, 0);
		let d = (alvo - agora) / 60000; if (d < -720) d += 1440; if (d > 720) d -= 1440;
		if (d <= 0) return 'agora';
		return d >= 60 ? `${Math.floor(d / 60)} h ${String(Math.round(d % 60)).padStart(2, '0')} min` : `${Math.ceil(d)} min`;
	}
	function sisNomeCoord(c) {
		if (!c) return '';
		const [x, y] = Array.isArray(c) ? c : String(c).split(',').map((v) => v.trim());
		if (x === undefined || y === undefined || x === '' || y === '') return '';
		const t = validTargets.find((t) => t.x == x && t.y == y);
		return t ? `${t.name} [${x},${y}]` : `[${x},${y}]`;
	}
	// 30.210 → 30,2k · 1.250.000 → 1,25M
	const sisCurto = (n) => { if (n === undefined || n === null || isNaN(n)) return '—'; n = +n; const a = Math.abs(n);
		return a >= 1e6 ? sisFmt(n / 1e6, a >= 1e7 ? 1 : 2) + 'M' : a >= 1e4 ? sisFmt(n / 1e3, 1) + 'k' : sisFmt(n); };
	const sisHa = (ms) => { const s = Math.max(0, Math.round((Date.now() - ms) / 1000)); return s < 90 ? `há ${s} s` : `há ${Math.round(s / 60)} min`; };
	function sisMedidor(rot, v, max, cls) {
		const w = sisEl('div', 'sis-med ' + (cls || '')); const tem = v !== undefined && v !== null && !isNaN(v);
		const p = max && tem ? Math.max(0, Math.min(100, v / max * 100)) : null;
		const top = sisEl('div', 'sis-med-top'); top.append(sisEl('span', '', rot), sisEl('em', '', p !== null ? sisFmt(p) + '%' : ''));
		const b = sisEl('div', 'sis-med-barra'); const i = sisEl('i'); i.style.width = (p || 0).toFixed(1) + '%'; if (p !== null && p < 15) i.classList.add('baixo'); b.append(i);
		const val = sisEl('b', 'sis-med-val', !tem ? '—' : max ? `${sisCurto(v)} / ${sisCurto(max)}` : sisCurto(v));
		w.append(top, b, val);
		w.title = `${rot}: ${tem ? sisFmt(v) : 'lendo…'}${max ? ' de ' + sisFmt(max) : ''}`;
		return w;
	}
	// modo Linha: "Comb 82%" com um traço de nível embaixo
	function sisMiniRec(rot, v, max, cls) {
		const tem = v !== undefined && v !== null && !isNaN(v); const p = max && tem ? Math.max(0, Math.min(100, v / max * 100)) : null;
		const s = sisEl('span', 'sis-mr ' + cls, `${rot} ${p !== null ? sisFmt(p) + '%' : (tem ? sisCurto(v) : '—')}`);
		s.style.setProperty('--p', (p || 0).toFixed(1) + '%'); if (p !== null && p < 15) s.classList.add('baixo');
		s.title = `${rot}: ${tem ? sisFmt(v) : 'lendo…'}${max ? ' de ' + sisFmt(max) : ''}`;
		return s;
	}
	function sisDet(k, v) { const d = sisEl('div'); const vv = sisEl('span', 'v', v); vv.title = v; d.append(sisEl('span', 'k', k), vv); return d; }
	function sisBotao(txt, titulo, fn, cls) {
		const b = sisEl('button', 'sis-mini' + (cls ? ' ' + cls : ''), txt); b.type = 'button'; b.title = titulo;
		b.addEventListener('click', (e) => { e.stopPropagation(); try { const r = fn(); if (r && r.catch) r.catch((er) => cLog(2, 'Sis-Bot: painel', er)); } catch (er) { cLog(2, 'Sis-Bot: painel', er); } });
		return b;
	}
	function sisTarefaDe(r) {
		if (r.craft) return 'Craft';
		const t = sisPainel.tarefas.get(r.pk) || {}; const k = t.sis || t.tarefa || '';
		return SIS_TAREFA_NOME[k] ? k : '';
	}
	// a tarefa vem das sincronizações; para a frota não ficar em "Sem tarefa" até a 1ª sincronização, lê do que está gravado
	function sisLerTarefa(pk) {
		if (sisPainel.tarefas.has(pk) || sisPainel.lendo.has(pk)) return;
		sisPainel.lendo.add(pk);
		Promise.resolve(GM.getValue(pk, '{}')).then((v) => {
			const g = JSON.parse(v || '{}') || {};
			if (!sisPainel.tarefas.has(pk)) sisPainel.tarefas.set(pk, { tarefa: g.assignment || '', sis: g.sisCerebro ? (g.sisTarefa || 'Supply') : '' });
			sisAgendarRender();
		}).catch(() => {}).finally(() => sisPainel.lendo.delete(pk));
	}

	function sisCriarLinhaFrota(fleet, pk) {
		const el = sisEl('div', 'sis-st-linha'); el.dataset.pk = pk;
		const grip = sisEl('span', 'sis-grip', '⋮⋮'); grip.title = 'Arraste para mudar a ordem';
		const c1 = sisEl('div', 'sis-st-c1'); const nome = sisEl('b', 'sis-st-nome'); const chips = sisEl('div', 'sis-st-chips');
		const tarefa = sisEl('span', 'sis-chip tarefa'); const mov = sisEl('span', 'sis-chip mov');
		chips.append(tarefa, mov); c1.append(nome, chips);
		const c2 = sisEl('div', 'sis-st-c2'); const est = sisEl('div', 'sis-st-estado'); const estTxt = sisEl('span', 'sis-st-esttxt');
		const tempo = sisEl('span', 'sis-tempo'); tempo.title = 'Há quanto tempo a frota está neste estado';
		est.append(sisEl('i', 'sis-core'), estTxt, tempo); const sub = sisEl('div', 'sis-st-sub sis-sub-estado'); const ciclo = sisEl('div', 'sis-st-sub sis-ciclo'); c2.append(est, sub, ciclo);
		const c3 = sisEl('div', 'sis-st-c3'); const onde = sisEl('div', 'sis-st-onde'); const alvo = sisEl('div', 'sis-st-sub sis-sub-alvo'); c3.append(onde, alvo);
		const falta = sisEl('div', 'sis-st-falta'); const mini = sisEl('div', 'sis-mini-rec'); const meds = sisEl('div', 'sis-meds'); const det = sisEl('div', 'sis-st-det');
		const acoes = sisEl('div', 'sis-st-acoes');
		const up = sisBotao('▲', 'Subir na lista', () => sisMover(pk, -1), 'seta'); const down = sisBotao('▼', 'Descer na lista', () => sisMover(pk, 1), 'seta');
		acoes.append(up, down,
			sisBotao('■', 'Parar esta frota (termina a operação em andamento)', () => { const f = sisPainel.linhas.get(pk).fleet; if (!f.stopping && !String(f.state).includes('STOPPED')) { f.stopping = true; updateFleetState(f, 'Stopping ...'); } }),
			sisBotao('↻', 'Reiniciar a frota (parada ou com erro)', () => resetFleetState(sisPainel.linhas.get(pk).fleet)));
		el.append(grip, c1, c2, c3, falta, mini, meds, det, acoes);
		sisLigarArrasto(el, grip, pk);
		return { el, pk, nome, tarefa, mov, estTxt, tempo, ciclo, sub, onde, alvo, falta, mini, meds, det, up, down, fleet };
	}
	function sisCriarLinhaCraft(fleet, pk) {
		const el = sisEl('div', 'sis-st-linha craft'); el.dataset.pk = pk;
		const grip = sisEl('span', 'sis-grip', '⋮⋮'); grip.title = 'Arraste para mudar a ordem';
		const nome = sisEl('b', 'sis-st-nome'); const item = sisEl('div', 'sis-st-sub');
		const c1 = sisEl('div', 'sis-st-c1 craft'); c1.append(nome, item);
		const c2 = sisEl('div', 'sis-st-c2'); const est = sisEl('div', 'sis-st-estado'); const estTxt = sisEl('span', 'sis-st-esttxt'); est.append(sisEl('i', 'sis-core'), estTxt);
		const sub = sisEl('div', 'sis-st-sub'); c2.append(est, sub);
		const acoes = sisEl('div', 'sis-st-acoes');
		const up = sisBotao('▲', 'Subir na lista', () => sisMover(pk, -1), 'seta'); const down = sisBotao('▼', 'Descer na lista', () => sisMover(pk, 1), 'seta');
		acoes.append(up, down);
		el.append(grip, c1, c2, acoes);
		sisLigarArrasto(el, grip, pk);
		return { el, pk, nome, item, estTxt, sub, up, down, craft: true, fleet };
	}
	// tempo no estado, ordem do Sis-Bot esperando há X, ciclo atual e último/média (atualiza a cada 1 s)
	function sisPintarTempos(r) {
		if (!r.tempo || r.craft) return;
		const f = r.fleet; const t0 = sisDesdeEstado(r.pk, f.state);
		setTxt(r.tempo, sisTempo(Date.now() - t0));
		const ord = sisPainel.ordens.get(r.pk);
		if (ord && r.sub && r.sub.textContent) { const txt = r.sub.textContent.replace(/ · \d+(?::\d\d){0,2}(?: s)?$/, ''); if (/Sis-Bot:/.test(txt)) setTxt(r.sub, `${txt} · ${sisTempo(Date.now() - ord.desde)}`); }
		const c = sisCiclos[r.pk];
		const partes = [];
		if (c && +c.t0 > 0) partes.push(`ciclo ${sisTempo(Date.now() - +c.t0)}`);
		if (c && c.ultimo) partes.push(`último ${sisTempo(c.ultimo * 1000)}${c.ultimo_ok === 0 ? ' (falhou)' : ''}`);
		if (c && c.media) partes.push(`média ${sisTempo(c.media * 1000)}`);
		setTxt(r.ciclo, partes.join(' · ')); r.ciclo.style.display = partes.length ? '' : 'none';
		r.ciclo.title = 'Ciclo = da 1ª ordem do plano do Sis-Bot até o fim do plano (inclui viagens, mineração e todas as transações)';
	}
	function sisPintarLinha(r) {
		const f = r.fleet; const cor = sisCorEstado(f.state); r.el.dataset.cor = cor;
		setTxt(r.estTxt, f.state || '—'); r.estTxt.title = f.state || '';
		const falta = sisFaltam(f.state);
		if (r.craft) {
			const coords = f.craftingId && f.craftingCoords ? f.craftingCoords : f.coordinates;
			setTxt(r.nome, `${f.label}${coords ? ' · ' + sisNomeCoord(coords) : ''}`);
			setTxt(r.item, f.item ? `${f.amount ? sisFmt(f.amount) + ' × ' : ''}${f.item}${f.sisUnica ? ' (lote do Sis-Bot)' : ''}` : 'sem receita');
			setTxt(r.sub, falta ? `termina em ${falta}` : '');
			return;
		}
		const t = sisPainel.tarefas.get(r.pk) || {}; const tarefa = sisTarefaDe(r);
		setTxt(r.nome, f.label); r.nome.title = f.label; r.nome.style.color = f.fontColor && f.fontColor !== 'white' ? f.fontColor : '';
		const emMis = sisEmMissao.has(r.pk);
		setTxt(r.tarefa, (emMis ? '◆ MISSÃO · ' : '') + (SIS_TAREFA_NOME[tarefa] || 'sem tarefa')); r.tarefa.dataset.t = tarefa; r.tarefa.dataset.missao = emMis ? '1' : '';
		r.tarefa.title = emMis ? 'Frota em missão (lançada no dashboard): quando terminar, volta para a tarefa dela' : '';
		setTxt(r.mov, f.moveType || ''); r.mov.style.display = f.moveType ? '' : 'none';
		const ord = sisPainel.ordens.get(r.pk);
		const rotFalta = { minerando: 'para em', viagem: 'chega em', erro: 'nova tentativa em', scan: 'próximo scan em', operando: 'termina em' }[cor] || 'em';
		setTxt(r.sub, [falta && `${rotFalta} ${falta}`, ord && `Sis-Bot: ${ord.rotulo || ord.acao} · ${sisTempo(Date.now() - ord.desde)}`].filter(Boolean).join(' · '));
		sisPintarTempos(r);
		setTxt(r.falta, falta ? `${rotFalta} ${falta}` : (ord ? '…' : '')); r.falta.title = ord ? `Sis-Bot: ${ord.rotulo || ord.acao}` : '';
		setTxt(r.onde, sisNomeCoord(f.startingCoords) || (cor === 'viagem' ? 'em viagem' : '—'));
		setTxt(r.alvo, f.moveTarget ? `→ ${sisNomeCoord(f.moveTarget)}` : (f.destCoord && !t.sis ? `destino ${sisNomeCoord(f.destCoord)}` : ''));
		const p = sisPainel.pods.get(r.pk) || {};
		if (sisUI.modo === 1) {
			r.mini.replaceChildren(sisMiniRec('Comb', p.comb ?? f.fuelCnt, f.fuelCapacity, 'comb'), sisMiniRec('Mun', p.mun, f.ammoCapacity, 'mun'), sisMiniRec('Carga', p.carga, f.cargoCapacity, 'carga'));
			return;
		}
		const meds = [sisMedidor('Combustível', p.comb ?? f.fuelCnt, f.fuelCapacity, 'comb'), sisMedidor('Munição', p.mun, f.ammoCapacity, 'mun'),
			sisMedidor('Carga', p.carga, f.cargoCapacity, 'carga'), sisMedidor(f.scanCost > 0 ? 'Comida · SDU' : 'Comida', p.comida ?? f.foodCnt, 0, 'comida')];
		if (f.scanCost > 0) setTxt(meds[3].querySelector('b'), `${sisCurto(p.comida ?? f.foodCnt)} · ${sisCurto(f.sduCnt)}`);
		r.meds.replaceChildren(...meds);
		if (sisUI.modo === 3) {
			r.det.replaceChildren(
				sisDet('Movimento', f.moveType || '—'),
				sisDet('Ordem do Sis-Bot', ord ? `${ord.rotulo || ord.acao} · ${sisHa(ord.desde)}` : 'nenhuma agora'),
				sisDet('Destino da tarefa', f.destCoord ? sisNomeCoord(f.destCoord) : '—'),
				sisDet('Chegada / término', falta || '—'),
				sisDet('Leitura dos porões', p.em ? sisHa(p.em) : 'aguardando a vez'),
				sisDet('Frota', `${r.pk.slice(0, 4)}…${r.pk.slice(-4)}${f.emprestada ? ' · emprestada' : ''}`));
		}
	}
	function sisStatusLinha(fleet) {
		const pk = fleet.publicKey ? fleet.publicKey.toString() : fleet.label;
		let r = sisPainel.linhas.get(pk);
		if (!r) {
			r = fleet.publicKey ? sisCriarLinhaFrota(fleet, pk) : sisCriarLinhaCraft(fleet, pk);
			sisPainel.linhas.set(pk, r); r.fleet = fleet;
			if (fleet.publicKey) sisLerTarefa(pk);
			sisAgendarRender();
			return;
		}
		r.fleet = fleet; if (r.el.isConnected) sisPintarLinha(r);
	}
	function sisAgendarRender() {
		if (sisPainel.renderTimer) return;
		sisPainel.renderTimer = setTimeout(() => { sisPainel.renderTimer = null; try { sisRenderLista(); } catch (e) { cLog(2, 'Sis-Bot: painel', e); } }, 80);
	}

	// ---------- ordem manual (vale para o painel; não muda a prioridade configurada no dashboard)
	function sisItensOrdenados() {
		let pos = new Map(); sisUI.ordem.forEach((pk, i) => pos.set(pk, i));
		let novo = false;
		for (const pk of sisPainel.linhas.keys()) if (!pos.has(pk)) { pos.set(pk, sisUI.ordem.length); sisUI.ordem.push(pk); novo = true; }
		if (sisUI.ordem.length > 600) { sisUI.ordem = sisUI.ordem.filter((pk) => sisPainel.linhas.has(pk)); pos = new Map(); sisUI.ordem.forEach((pk, i) => pos.set(pk, i)); novo = true; }
		if (novo) sisSalvarUI();
		return [...sisPainel.linhas.values()].sort((a, b) => pos.get(a.pk) - pos.get(b.pk));
	}
	function sisReordenar(pk, alvoPk, onde) {
		if (pk === alvoPk) return;
		sisItensOrdenados();
		const o = sisUI.ordem.filter((x) => x !== pk); let i = o.indexOf(alvoPk); if (i < 0) return;
		if (onde === 'depois') i++;
		o.splice(i, 0, pk); sisUI.ordem = o; sisPainel.seguir = pk;
		sisSalvarUI(); sisRenderLista(true);
	}
	// setas: troca com a vizinha visível da mesma seção
	function sisMover(pk, d) {
		const r = sisPainel.linhas.get(pk); if (!r) return; const k = sisTarefaDe(r);
		const sec = sisItensOrdenados().filter((x) => sisTarefaDe(x) === k && sisPassaFiltro(x));
		const i = sec.findIndex((x) => x.pk === pk); const j = i + d;
		if (i < 0 || j < 0 || j >= sec.length) return;
		sisReordenar(pk, sec[j].pk, d > 0 ? 'depois' : 'antes');
	}
	function sisLimparAlvos() { document.querySelectorAll('#sisStatusFrotas [data-alvo]').forEach((e) => { delete e.dataset.alvo; }); }
	// arrastar: só pelo ⋮⋮ e só dentro da mesma seção
	function sisLigarArrasto(el, grip, pk) {
		grip.addEventListener('mousedown', () => { el.draggable = true; });
		grip.addEventListener('mouseup', () => { if (!sisPainel.arrastando) el.draggable = false; });
		el.addEventListener('dragstart', (e) => {
			if (!el.draggable) { e.preventDefault(); return; }
			sisPainel.arrastando = pk; el.classList.add('sis-arrastando');
			try { e.dataTransfer.effectAllowed = 'move'; e.dataTransfer.setData('text/plain', pk); } catch (er) { }
		});
		el.addEventListener('dragend', () => { el.draggable = false; el.classList.remove('sis-arrastando'); sisPainel.arrastando = null; sisLimparAlvos(); sisRenderLista(true); });
		el.addEventListener('dragover', (e) => {
			const de = sisPainel.arrastando; if (!de || de === pk) return;
			const a = sisPainel.linhas.get(de), b = sisPainel.linhas.get(pk);
			if (!a || !b || sisTarefaDe(a) !== sisTarefaDe(b)) return;
			e.preventDefault(); try { e.dataTransfer.dropEffect = 'move'; } catch (er) { }
			const q = el.getBoundingClientRect();
			const onde = (sisUI.modo === 3 ? e.clientX > q.left + q.width / 2 : e.clientY > q.top + q.height / 2) ? 'depois' : 'antes';
			if (el.dataset.alvo !== onde) { sisLimparAlvos(); el.dataset.alvo = onde; }
		});
		el.addEventListener('drop', (e) => {
			e.preventDefault(); const de = sisPainel.arrastando; const onde = el.dataset.alvo; delete el.dataset.alvo;
			if (de && onde) { sisPainel.arrastando = null; sisReordenar(de, pk, onde); }
		});
	}

	// ---------- lista: abas, seções e paginação
	function sisPassaFiltro(r) {
		if (sisPainel.filtro && !String(r.fleet && r.fleet.label || '').toLowerCase().includes(sisPainel.filtro)) return false;
		if (sisPainel.filtroCor && sisCorEstado(r.fleet && r.fleet.state) !== sisPainel.filtroCor) return false;
		return true;
	}
	function sisCabecalhoSecao(s, fechada, cont) {
		const h = sisEl('div', 'sis-sec' + (fechada ? ' fechada' : '')); h.dataset.t = s.k;
		const unid = s.k === 'Craft' ? ['processo', 'processos'] : ['frota', 'frotas'];
		h.append(sisEl('i', 'sis-sec-seta', fechada ? '▸' : '▾'), sisEl('span', '', s.nome + (cont ? ' (continuação)' : '')),
			sisEl('b', '', s.itens.length === s.total ? `${sisFmt(s.total)} ${unid[s.total === 1 ? 0 : 1]}` : `${sisFmt(s.itens.length)} de ${sisFmt(s.total)}`));
		h.title = fechada ? 'Mostrar esta seção' : 'Recolher esta seção';
		h.addEventListener('click', () => { const f = new Set(sisUI.fechadas); if (f.has(s.k)) f.delete(s.k); else f.add(s.k); sisUI.fechadas = [...f]; sisSalvarUI(); sisRenderLista(true); });
		return h;
	}
	function sisRenderAbas(porSec, total, aba) {
		const alvo = document.querySelector('#sisStatusAbas'); if (!alvo) return;
		const sig = aba + '|' + total + '|' + SIS_SECOES.map(([k]) => porSec.get(k).length).join(',');
		if (alvo.dataset.sig === sig) return; alvo.dataset.sig = sig;
		const bt = (k, nome, n) => {
			const b = sisEl('button', 'sis-aba' + (k === aba ? ' ativa' : '')); b.type = 'button'; b.dataset.t = k;
			b.append(sisEl('span', '', nome), sisEl('b', '', sisFmt(n)));
			b.addEventListener('click', () => { sisUI.aba = k; sisPainel.pagina = 1; sisSalvarUI(); sisRenderLista(true); });
			return b;
		};
		const l = [bt('todas', 'Todas', total)];
		for (const [k, nome] of SIS_SECOES) if (porSec.get(k).length) l.push(bt(k, nome, porSec.get(k).length));
		alvo.replaceChildren(...l);
	}
	function sisRenderPaginas(n, pags, ini, qtd) {
		const w = document.querySelector('#sisStatusPag'); if (!w) return;
		sisPainel.pags = pags; w.classList.toggle('uma', pags <= 1);
		setTxt(document.querySelector('#sisPagTxt'), n ? `${sisFmt(ini + 1)}–${sisFmt(ini + qtd)} de ${sisFmt(n)}${pags > 1 ? ` · página ${sisPainel.pagina} de ${pags}` : ''}` : 'nada para mostrar');
		w.querySelectorAll('button[data-p]').forEach((b) => { const p = b.dataset.p; b.disabled = (p === 'ini' || p === 'ant') ? sisPainel.pagina <= 1 : sisPainel.pagina >= pags; });
	}
	function sisRenderLista(forcar) {
		const corpo = document.querySelector('#sisStatusFrotas'); if (!corpo || sisPainel.arrastando) return;
		const itens = sisItensOrdenados();
		const porSec = new Map(SIS_SECOES.map(([k]) => [k, []]));
		for (const r of itens) porSec.get(sisTarefaDe(r)).push(r);
		const aba = sisUI.aba !== 'todas' && porSec.has(sisUI.aba) && porSec.get(sisUI.aba).length ? sisUI.aba : 'todas';
		sisRenderAbas(porSec, itens.length, aba);
		const fechadas = new Set(sisUI.fechadas);
		const secoes = SIS_SECOES.filter(([k]) => aba === 'todas' || k === aba)
			.map(([k, nome]) => ({ k, nome, total: porSec.get(k).length, itens: porSec.get(k).filter(sisPassaFiltro) }))
			.filter((s) => s.itens.length);
		const lista = []; for (const s of secoes) if (!fechadas.has(s.k)) s.itens.forEach((r) => lista.push(r));
		const pags = Math.max(1, Math.ceil(lista.length / sisUI.porPag));
		if (sisPainel.seguir) { const i = lista.findIndex((r) => r.pk === sisPainel.seguir); if (i >= 0) sisPainel.pagina = Math.floor(i / sisUI.porPag) + 1; }
		sisPainel.pagina = Math.max(1, Math.min(pags, sisPainel.pagina));
		const ini = (sisPainel.pagina - 1) * sisUI.porPag; const fatia = lista.slice(ini, ini + sisUI.porPag);
		const naPagina = new Set(fatia.map((r) => r.pk));
		const partes = []; const assin = [aba, sisUI.modo, sisPainel.pagina, itens.length];
		for (const s of secoes) {
			const fechada = fechadas.has(s.k);
			const daPag = fechada ? [] : s.itens.filter((r) => naPagina.has(r.pk));
			if (!fechada && !daPag.length) continue;
			const cont = !fechada && daPag[0] !== s.itens[0];
			partes.push({ sec: s, fechada, cont }); assin.push(`#${s.k}:${s.itens.length}/${s.total}:${fechada ? 1 : 0}:${cont ? 1 : 0}`);
			daPag.forEach((r) => { partes.push({ r, primeiro: r === s.itens[0], ultimo: r === s.itens[s.itens.length - 1] }); assin.push(r.pk); });
		}
		for (const p of partes) if (p.r) { try { sisPintarLinha(p.r); } catch (e) { cLog(3, 'Sis-Bot: painel', e); } p.r.up.disabled = p.primeiro; p.r.down.disabled = p.ultimo; }
		const sig = assin.join('|');
		if (forcar || sig !== sisPainel.assinatura) {
			sisPainel.assinatura = sig;
			const els = partes.map((p) => p.r ? p.r.el : sisCabecalhoSecao(p.sec, p.fechada, p.cont));
			if (!els.length) els.push(sisEl('div', 'sis-vazio', itens.length ? 'Nenhuma frota com este filtro.' : 'Carregando as frotas…'));
			corpo.replaceChildren(...els);
		}
		if (sisPainel.seguir) {
			const r = sisPainel.linhas.get(sisPainel.seguir); sisPainel.seguir = '';
			if (r && r.el.isConnected) { r.el.classList.remove('sis-movida'); void r.el.offsetWidth; r.el.classList.add('sis-movida'); }
		}
		sisRenderPaginas(lista.length, pags, ini, fatia.length);
	}

	// ---------- painel: montagem, modo, fonte, recolher
	function sisPainelIniciar() {
		const st = document.querySelector('#assistStatus'); if (!st) return;
		const fi = document.querySelector('#sisStatusFiltro');
		if (fi) fi.addEventListener('input', () => { sisPainel.filtro = fi.value.trim().toLowerCase(); sisPainel.pagina = 1; sisRenderLista(true); });
		document.querySelectorAll('#sisModo button').forEach((b) => b.addEventListener('click', () => { sisUI.modo = +b.dataset.m; sisSalvarUI(); sisPainelAplicarUI(); }));
		document.querySelectorAll('#sisFonte button').forEach((b) => b.addEventListener('click', () => { sisUI.fonte = Math.max(70, Math.min(160, sisUI.fonte + (b.dataset.f === '+' ? 10 : -10))); sisSalvarUI(); sisPainelAplicarUI(); }));
		const fv = document.querySelector('#sisFonteVal'); if (fv) fv.addEventListener('click', () => { sisUI.fonte = 100; sisSalvarUI(); sisPainelAplicarUI(); });
		document.querySelectorAll('#sisStatusPag button[data-p]').forEach((b) => b.addEventListener('click', () => {
			const p = b.dataset.p; sisPainel.pagina = p === 'ini' ? 1 : p === 'fim' ? sisPainel.pags : sisPainel.pagina + (p === 'prox' ? 1 : -1);
			sisRenderLista(true); const c = st.querySelector('.sis-st-corpo'); if (c) c.scrollTop = 0;
		}));
		const pp = document.querySelector('#sisPorPag');
		if (pp) pp.addEventListener('change', () => { sisUI.porPag = SIS_POR_PAG.includes(+pp.value) ? +pp.value : 20; sisPainel.pagina = 1; sisSalvarUI(); sisRenderLista(true); });
		const tc = document.querySelector('#sisTelaCheia'); if (tc) tc.addEventListener('click', (e) => { e.stopPropagation(); sisPainelTelaCheia(); });
		// Esc sai da tela cheia (se nenhuma janela estiver aberta por cima)
		document.addEventListener('keydown', (e) => { if (e.key === 'Escape' && sisUI.telaCheia && !document.querySelector('.assist-modal[style*="block"]')) sisPainelTelaCheia(false); });
		st.style.display = 'block';
		sisPainelAplicarUI();
	}
	function sisPainelAplicarUI() {
		const st = document.querySelector('#assistStatus'); if (!st) return;
		st.dataset.modo = String(sisUI.modo);
		st.style.fontSize = (12.5 * sisUI.fonte / 100).toFixed(2) + 'px';
		st.classList.toggle('sis-recolhido', !!sisUI.recolhido);
		st.classList.toggle('sis-tela-cheia', !!sisUI.telaCheia && !sisUI.recolhido);
		const tc = document.querySelector('#sisTelaCheia'); if (tc) { tc.classList.toggle('ativo', !!sisUI.telaCheia); tc.title = sisUI.telaCheia ? 'Voltar ao tamanho normal (Esc)' : 'Ocupar a página inteira'; tc.style.display = sisUI.recolhido ? 'none' : ''; }
		const x = st.querySelector('.assist-modal-header-right .assist-modal-close');
		if (x) { setTxt(x, sisUI.recolhido ? '▾' : '▴'); x.title = sisUI.recolhido ? 'Expandir o painel' : 'Recolher o painel'; }
		document.querySelectorAll('#sisModo button').forEach((b) => b.classList.toggle('ativo', +b.dataset.m === sisUI.modo));
		setTxt(document.querySelector('#sisFonteVal'), sisUI.fonte + '%');
		const pp = document.querySelector('#sisPorPag'); if (pp) pp.value = String(sisUI.porPag);
		const cab = document.querySelector('#sisStatusCab');
		if (cab) cab.replaceChildren(...(sisUI.modo === 1 ? ['', 'Frota', 'Estado', 'Onde → destino', 'Falta', 'Comb. · Mun. · Carga', ''] : ['', 'Frota', 'Estado', 'Onde', 'Combustível · Munição · Carga · Comida', '']).map((t) => sisEl('span', '', t)));
		sisPainel.linhas.forEach((r) => { if (!r.craft) { r.meds.replaceChildren(); r.mini.replaceChildren(); r.det.replaceChildren(); } });
		try { sisRenderLista(true); if (!sisUI.recolhido) sisStatusResumo(); } catch (e) { cLog(2, 'Sis-Bot: painel', e); }
	}
	// o painel fica sempre na tela; o ▴ do cabeçalho (e Tools › Status) recolhe/expande a lista
	// tela cheia: o painel ocupa a página inteira (fica gravado); recolher sai da tela cheia
	function sisPainelTelaCheia(v) { sisUI.telaCheia = v === undefined ? !sisUI.telaCheia : !!v; if (sisUI.telaCheia) sisUI.recolhido = false; sisSalvarUI(); sisPainelAplicarUI(); }
	function sisPainelRecolher(v) { sisUI.recolhido = v === undefined ? !sisUI.recolhido : !!v; if (sisUI.recolhido) sisUI.telaCheia = false; sisSalvarUI(); sisPainelAplicarUI(); }
	function sisPainelMostrar() { const st = document.querySelector('#assistStatus'); if (st) st.style.display = 'block'; }
	const sisPainelVisivel = () => { const e = document.querySelector('#assistStatus'); return !!e && e.style.display !== 'none' && !sisUI.recolhido; };
	function sisChip(txt, cls, titulo) { const c = sisEl('span', 'sis-chip ' + (cls || ''), txt); if (titulo) c.title = titulo; return c; }
	function sisStatusResumo() {
		const alvo = document.querySelector('#sisStatusResumo'); if (!alvo) return;
		const cont = {}; let frotas = 0;
		sisPainel.linhas.forEach((r) => { if (r.craft) return; frotas++; const c = sisCorEstado(r.fleet.state); cont[c] = (cont[c] || 0) + 1; });
		const m = sisResumoTx(true), sm = sisResumoTx(false);
		const quando = sisUltimo.em ? TimeToStr(new Date(sisUltimo.em)) : '—';
		const chips = [
			sisChip(enableAssistant ? '● Rodando' : '○ Parado', enableAssistant ? 'ok' : 'cinza'),
			sisChip(sisUltimo.ok ? `Dashboard conectado · ${quando}` : `Dashboard: ${sisUltimo.msg}`, sisUltimo.ok ? 'azul' : 'erro', 'Última sincronização com o Sis-Ágora'),
			sisChip(sisLocal.cfgAceita ? `Config v${sisLocal.cfgAceita.versao}` : 'sem configuração', sisPendente ? 'aviso' : '', sisPendente ? `v${sisPendente.versao} aguardando o seu aceite` : 'Configuração aplicada neste navegador'),
			sisChip(`${sisTxHora.length} tx/h${parseInt(sisLocal.tetoTxHora) ? ' de ' + sisLocal.tetoTxHora : ''}`, '', 'Transações na última hora (teto local)'),
			sisChip(`Confirmação ${sm.n ? sisFmt(sm.mediana, 1) + ' s' : '—'} · várias instr. ${m.n ? sisFmt(m.mediana, 1) + ' s' : '—'}`, m.n && m.mediana > (+globalSettings.multiIxFeeAutoSeg || 30) ? 'aviso' : '', 'Mediana do tempo de confirmação nos últimos 30 min'),
		];
		if (sisFeeAuto.ligada) chips.push(sisChip(`taxa mín. automática ${globalSettings.minPriorityFeeForMultiIx}`, 'aviso'));
		if (sisLocal.modoLocal) chips.push(sisChip('MODO LOCAL', 'aviso'));
		// contadores por estado: clicar filtra a lista por esse estado
		const contagem = sisEl('div', 'sis-st-cont');
		for (const k of ['minerando', 'viagem', 'operando', 'scan', 'erro', 'parado', 'livre']) {
			if (!cont[k] && sisPainel.filtroCor !== k) continue;
			const ativo = sisPainel.filtroCor === k;
			const c = sisEl('button', 'sis-cont' + (ativo ? ' ativo' : '')); c.type = 'button'; c.dataset.cor = k;
			c.title = ativo ? 'Mostrar todas as frotas' : `Mostrar só as frotas ${SIS_COR_NOME[k]}`;
			c.append(sisEl('i', 'sis-core'), sisEl('b', '', cont[k] || 0), sisEl('span', '', SIS_COR_NOME[k]));
			c.addEventListener('click', () => { sisPainel.filtroCor = ativo ? '' : k; sisPainel.pagina = 1; sisStatusResumo(); sisRenderLista(true); });
			contagem.append(c);
		}
		const linha1 = sisEl('div', 'sis-st-chips-topo'); linha1.append(...chips);
		alvo.replaceChildren(linha1, contagem);
		setTxt(document.querySelector('#sisStatusInfo'), `${frotas} frotas · ${sisFila.eventos.length} eventos na fila · v${GM_info.script.version}`);
	}
	// porão, combustível e munição de uma frota por vez — só das frotas na página aberta do painel: 3 leituras a cada 12 s
	async function sisAtualizarPods() {
		if (!sisPainelVisivel() || !initComplete || !userFleets.length) return;
		const vis = [...sisPainel.linhas.values()].filter((r) => !r.craft && r.el.isConnected && r.fleet && r.fleet.cargoHold);
		if (!vis.length) return;
		const fl = vis[sisPainel.rr++ % vis.length].fleet;
		try {
			const ler = async (pod) => (await solanaReadConnection.getParsedTokenAccountsByOwner(pod, { programId: tokenProgramPK })).value;
			const [carga, tanque, banco] = await Promise.all([ler(fl.cargoHold), ler(fl.fuelTank), ler(fl.ammoBank)]);
			const soma = (l) => l.reduce((n, { account }) => n + account.data.parsed.info.tokenAmount.uiAmount, 0);
			const ocupado = carga.reduce((n, { account }) => { const i = account.data.parsed.info; const it = cargoItems.find((c) => c.token === i.mint); return n + i.tokenAmount.uiAmount * (it && it.size ? it.size : 1); }, 0);
			const food = sageGameAcct.account.mints.food.toString();
			const comida = carga.filter(({ account }) => account.data.parsed.info.mint === food).reduce((n, { account }) => n + account.data.parsed.info.tokenAmount.uiAmount, 0);
			sisPainel.pods.set(fl.publicKey.toString(), { carga: ocupado, comb: soma(tanque), mun: soma(banco), comida, em: Date.now() });
			const r = sisPainel.linhas.get(fl.publicKey.toString()); if (r && r.el.isConnected) sisPintarLinha(r);
		} catch (e) { cLog(3, 'Sis-Bot: painel', e); }
	}
	setInterval(() => { if (sisPainelVisivel()) { try { sisStatusResumo(); sisRenderLista(); } catch (e) { cLog(2, 'Sis-Bot: painel', e); } } }, 3000);
	setInterval(sisAtualizarPods, 12000);
	// cartões no topo das Estatísticas
	function sisStatsResumo() {
		const g = transactionStats.groups || {}; const horas = Math.max(1 / 60, (Date.now() / 1000 - transactionStats.start) / 3600);
		const tot = (nome) => { const k = Object.keys(g).find((x) => x.toLowerCase().includes(nome)); return k && g[k].TOTAL ? g[k].TOTAL : null; };
		const fees = tot('fee'); const conf = tot('confirm');
		const m = sisResumoTx(true), sm = sisResumoTx(false);
		const card = (rot, val, det, cls) => { const c = sisEl('div', 'sis-card ' + (cls || '')); c.append(sisEl('div', 'rot', rot), sisEl('div', 'val', val), sisEl('div', 'det', det)); return c; };
		const w = sisEl('div', 'sis-cards');
		w.append(
			card('Transações', sisFmt(conf ? conf.count : 0), `${sisFmt((conf ? conf.count : 0) / horas, 1)} por hora · ${sisTxHora.length} na última hora`),
			card('SOL em taxas', fees ? fees.value.toFixed(6) : '0', `${fees ? (fees.value / horas).toFixed(6) : '0'} SOL/h · taxa atual ${currentFee}`, 'laranja'),
			card('Confirmação (30 min)', sm.n ? `${sisFmt(sm.mediana, 1)} s` : '—', `várias instruções ${m.n ? sisFmt(m.mediana, 1) + ' s (90%: ' + sisFmt(m.p90, 1) + ' s)' : '—'}`, m.n && m.mediana > (+globalSettings.multiIxFeeAutoSeg || 30) ? 'aviso' : 'ciano'),
			card('RPC', sisFmt(solanaReadCount + solanaWriteCount), `${sisFmt(solanaReadCount)} leituras · ${sisFmt(solanaWriteCount)} escritas · ${sisFmt(solanaErrorCount)} erros`, solanaErrorCount ? 'aviso' : 'roxo'),
		);
		return w;
	}
	function sisCssTema(opac) {
		const a = Math.max(0.55, Math.min(1, +opac || 0.9));
		return ` #assistStatus, #assistStats, #assistStarbaseStatus { background: rgba(8, 19, 28, ${a}) !important; opacity: 1 !important; border: 1px solid rgba(140, 190, 235, .18); border-radius: 14px; box-shadow: 0 18px 50px rgba(0, 0, 0, .55); color: #eef4f9; font-family: 'Plus Jakarta Sans', system-ui, -apple-system, 'Segoe UI', sans-serif; overflow: hidden; }
		#assistStats { width: min(720px, 94vw); }
		.assist-modal-header { display: flex !important; align-items: center; flex-wrap: nowrap; background: linear-gradient(90deg, #0f1f2d, #152a3b) !important; border-bottom: 1px solid rgba(77, 167, 239, .45) !important; color: #eef4f9 !important; padding: 10px 16px !important; gap: 10px; font-weight: 600; }
		.assist-modal-header small { color: #a9bccb; font-weight: 500; }
		.assist-modal-header-right { color: #a9bccb !important; }
		.assist-modal-header-right .assist-modal-close:hover { color: #f07b7b; }
		.assist-modal-content { background: #0b1824 !important; border: 1px solid rgba(140, 190, 235, .18) !important; border-radius: 14px; color: #eef4f9; font-family: 'Plus Jakarta Sans', system-ui, -apple-system, 'Segoe UI', sans-serif; box-shadow: 0 18px 50px rgba(0, 0, 0, .55) !important; }
		.assist-modal { background-color: rgba(3, 10, 16, .6) !important; }
		.assist-modal-body { font-size: 12.5px !important; color: #eef4f9; }
		.assist-btn, .assist-btn-alt, .assist-modal-btn { background: #0f1f2d !important; color: #4da7ef !important; border: 1px solid rgba(77, 167, 239, .35) !important; border-radius: 9px !important; font-family: inherit; transition: background .15s, border-color .15s; }
		.assist-btn { padding: 3px 10px; }
		.assist-btn:hover, .assist-btn-alt:hover, .assist-modal-btn:hover, .assist-modal-save:hover { background: rgba(77, 167, 239, .16) !important; border-color: #4da7ef !important; }
		.dropdown { background: #0b1824 !important; border: 1px solid rgba(140, 190, 235, .18); border-radius: 12px; overflow: hidden; box-shadow: 0 18px 50px rgba(0, 0, 0, .55) !important; }
		.dropdown .assist-btn-alt { border: 0 !important; border-radius: 0 !important; border-bottom: 1px solid rgba(140, 190, 235, .09) !important; }
		.assist-modal-content input, .assist-modal-content select, .assist-modal-content textarea, #assistStatus input, #assistStatus select { background: #08131c !important; color: #eef4f9 !important; border: 1px solid rgba(140, 190, 235, .18) !important; border-radius: 6px; padding: .3em .6em; font-family: inherit; }
		.sis-corpo h3 { color: #4da7ef; font-weight: 600; }
		.tooltip .tooltiptext { background: #0f1f2d !important; border-color: rgba(140, 190, 235, .18) !important; }
		#assistStats td { color: #a9bccb; padding: 2px 10px 2px 0 !important; border-bottom: 1px solid rgba(140, 190, 235, .06); }
		#assistStats tr[style*="opacity"] td { color: #4da7ef; text-transform: uppercase; font-size: 11px; letter-spacing: .04em; padding-top: 8px !important; }

		/* ---------------- painel de Status (1.1.9) — tamanhos em em: o controle de fonte escala o painel inteiro ---------------- */
		#assistStatus { --t1: #eef4f9; --t2: #a9bccb; --t3: #7890a3; --az: #4da7ef; --linha: rgba(140, 190, 235, .09); width: min(1280px, 97vw); overflow: visible !important; font-size: 12.5px; }
		#assistStatus [data-cor="minerando"] { --c: #7fd49a; } #assistStatus [data-cor="viagem"] { --c: #4da7ef; } #assistStatus [data-cor="operando"] { --c: #e7c76a; }
		#assistStatus [data-cor="scan"] { --c: #8f80f3; } #assistStatus [data-cor="craft"] { --c: #44d7f7; } #assistStatus [data-cor="erro"] { --c: #f07b7b; }
		#assistStatus [data-cor="parado"] { --c: #50636f; } #assistStatus [data-cor="livre"] { --c: #7890a3; }
		#assistStatus .assist-modal-header { font-size: 1.04em; padding: .55em .9em !important; gap: .55em; flex-wrap: wrap; border-radius: 14px 14px 0 0; position: relative; }
		#assistStatus.sis-recolhido .assist-modal-header { border-radius: 14px; border-bottom-color: transparent !important; }
		#assistStatus.sis-recolhido .sis-st-corpo { display: none; }
		#assistStatus.sis-recolhido { width: auto; max-width: 97vw; }
		#assistStatus.sis-tela-cheia { position: fixed !important; top: 0 !important; left: 0 !important; right: 0 !important; bottom: 0 !important; width: auto !important; max-width: none !important; border-radius: 0 !important; border: 0; z-index: 50 !important; }
		#assistStatus.sis-tela-cheia .assist-status-content { display: flex; flex-direction: column; height: 100%; }
		#assistStatus.sis-tela-cheia .assist-modal-header { border-radius: 0; cursor: default !important; }
		#assistStatus.sis-tela-cheia .sis-st-corpo { flex: 1 1 auto; min-height: 0; max-height: none; padding: 1em 1.6em 1.2em !important; }
		.assist-modal { z-index: 60 !important; } #assistStats, #assistStarbaseStatus { z-index: 55 !important; }
		.sis-hbtn { font: inherit; font-size: .8em; line-height: 1; width: 1.7em; height: 1.7em; margin-right: .35em; padding: 0; vertical-align: middle; border-radius: 4px; border: 1px solid rgba(140, 190, 235, .22); background: transparent; color: var(--t2); cursor: pointer; }
		.sis-hbtn:hover { color: var(--t1); border-color: #4da7ef; background: rgba(77, 167, 239, .14); } .sis-hbtn.ativo { color: #03111a; background: #4da7ef; border-color: #4da7ef; }
		#assistStatus .assist-modal-header .sis-chip { font-size: .8em; }
		#assistStatus .assist-modal-header-right { font-size: 1.25em; padding-left: .3em; margin-left: auto !important; }
		.sis-st-titulo { white-space: nowrap; margin-right: .2em; }
		.sis-st-ctl { display: inline-flex; align-items: center; gap: .35em; position: relative; }
		.sis-st-ctl > .assist-btn { margin: 0 !important; padding: .28em .85em; font-size: .88em; line-height: 1.35; cursor: pointer; }
		.sis-st-ctl > .assist-btn span { font-size: inherit !important; }
		#autoScanBtn { color: #7fd49a !important; border-color: rgba(127, 212, 154, .45) !important; }
		#assistStatus .dropdown { top: calc(100% + 6px); left: auto; right: 0; margin: 0; min-width: 15em; z-index: 6; font-size: .9em; }
		#assistStatus .dropdown .assist-btn-alt { padding: .75em 1.1em; } #assistStatus .dropdown .assist-btn-alt span { font-size: inherit !important; }
		#assistStatus .sis-st-corpo { max-height: calc(100vh - 190px); overflow: auto; padding: .85em 1.05em 1em !important; font-size: 1em !important; }
		.sis-st-chips-topo, .sis-st-cont, .sis-st-chips { display: flex; flex-wrap: wrap; gap: .45em; align-items: center; }
		.sis-st-cont { margin: .55em 0 0; gap: .3em; }
		.sis-cont { display: inline-flex; align-items: center; gap: .45em; font: inherit; font-size: .9em; color: var(--t2); background: transparent; border: 1px solid transparent; border-radius: 4px; padding: .22em .6em; cursor: pointer; }
		.sis-cont b { color: var(--t1); font-variant-numeric: tabular-nums; }
		.sis-cont:hover { border-color: var(--linha); background: rgba(255, 255, 255, .03); }
		.sis-cont.ativo { border-color: var(--c); background: color-mix(in srgb, var(--c) 14%, transparent); color: var(--t1); }
		.sis-chip { display: inline-flex; align-items: center; padding: .15em .65em; border-radius: 99px; font-size: .88em; line-height: 1.45; color: #a9bccb; background: rgba(255, 255, 255, .06); border: 1px solid rgba(140, 190, 235, .12); white-space: nowrap; }
		.sis-chip.ok { color: #7fd49a; background: rgba(127, 212, 154, .12); } .sis-chip.azul { color: #79a8f0; background: rgba(121, 168, 240, .13); } .sis-chip.erro { color: #f07b7b; background: rgba(240, 123, 123, .12); }
		.sis-chip.tarefa[data-missao="1"] { color: #ffd166 !important; border-color: rgba(255, 209, 102, .6); background: rgba(255, 209, 102, .12); box-shadow: 0 0 .7em rgba(255, 209, 102, .35); font-weight: 700; animation: sisMissao 2.2s ease-in-out infinite; }
		@keyframes sisMissao { 50% { box-shadow: 0 0 1.3em rgba(255, 209, 102, .7); } }
		.sis-chip.aviso { color: #e7c76a; background: rgba(231, 199, 106, .12); } .sis-chip.cinza { color: #7890a3; }
		.sis-chip.tarefa[data-t="Mine"], .sis-chip.tarefa[data-t="Garimpo"] { color: #7fd49a; } .sis-chip.tarefa[data-t="Supply"], .sis-chip.tarefa[data-t="Transport"] { color: #f2a65a; } .sis-chip.tarefa[data-t="Scan"] { color: #8f80f3; } .sis-chip.tarefa[data-t="Prospectar"] { color: #b6a6ff; }

		/* abas por tarefa */
		.sis-abas { display: flex; flex-wrap: wrap; gap: .35em; margin: .75em 0 .1em; padding-bottom: .6em; border-bottom: 1px solid var(--linha); }
		.sis-aba { --ac: var(--az); display: inline-flex; align-items: center; gap: .55em; padding: .4em .95em .4em .85em; font: inherit; font-size: .92em; color: var(--t2); background: rgba(255, 255, 255, .035); border: 0; cursor: pointer;
			clip-path: polygon(.55em 0, 100% 0, 100% calc(100% - .55em), calc(100% - .55em) 100%, 0 100%, 0 .55em); box-shadow: inset 0 0 0 1px rgba(140, 190, 235, .16); transition: color .15s, background .15s; }
		.sis-aba b { font-size: .86em; font-weight: 700; color: var(--t3); font-variant-numeric: tabular-nums; }
		.sis-aba:hover { color: var(--t1); box-shadow: inset 0 0 0 1px var(--ac); }
		.sis-aba.ativa { color: #03111a; background: var(--ac); box-shadow: none; } .sis-aba.ativa b { color: rgba(3, 17, 26, .7); }
		.sis-aba[data-t="Supply"], .sis-aba[data-t="Transport"] { --ac: #f2a65a; } .sis-aba[data-t="Garimpo"], .sis-aba[data-t="Mine"] { --ac: #7fd49a; }
		.sis-aba[data-t="Scan"] { --ac: #8f80f3; } .sis-aba[data-t="Prospectar"] { --ac: #b6a6ff; } .sis-aba[data-t="Craft"] { --ac: #44d7f7; } .sis-aba[data-t=""] { --ac: #a9bccb; }

		/* barra: filtro, modo, fonte */
		.sis-st-filtro { display: flex; align-items: center; flex-wrap: wrap; gap: .6em; margin: .6em 0 .55em; color: var(--t3); font-size: .92em; }
		#sisStatusFiltro { width: 15em; font-size: 1em !important; } #sisStatusInfo { margin-left: auto; }
		.sis-seg { display: inline-flex; align-items: stretch; border: 1px solid rgba(140, 190, 235, .18); border-radius: 4px; overflow: hidden; background: rgba(0, 0, 0, .15); }
		.sis-seg button, .sis-seg span { font: inherit; font-size: .9em; padding: .32em .75em; background: transparent; color: var(--t2); border: 0; border-right: 1px solid rgba(140, 190, 235, .12); cursor: pointer; }
		.sis-seg > :last-child { border-right: 0; } .sis-seg button:hover { color: var(--t1); background: rgba(77, 167, 239, .1); }
		.sis-seg button.ativo { color: #03111a; background: var(--az); }
		.sis-seg span { color: var(--t1); min-width: 3.4em; text-align: center; font-variant-numeric: tabular-nums; }

		/* linhas — sinalização: núcleo hexagonal com anel girando, cantos de HUD e trilho de energia animado na base */
		.sis-st-cabecalho, .sis-st-linha { display: grid; gap: .9em; align-items: center; }
		.sis-st-cabecalho { padding: .2em .8em .3em .5em; color: var(--t3); font-size: .84em; text-transform: uppercase; letter-spacing: .06em; }
		#assistStatus[data-modo="3"] .sis-st-cabecalho { display: none; }
		.sis-st-linha { --c: #7890a3; position: relative; padding: .6em .8em .65em .5em; margin-bottom: .32em; border-radius: 4px; overflow: hidden;
			background: linear-gradient(100deg, color-mix(in srgb, var(--c) 7%, transparent), rgba(255, 255, 255, .015) 40%); border: 1px solid var(--linha); transition: background .2s, opacity .2s; }
		.sis-st-linha:hover { background: linear-gradient(100deg, color-mix(in srgb, var(--c) 13%, transparent), rgba(77, 167, 239, .05) 60%); }
		.sis-st-linha::before { content: ''; position: absolute; inset: 0; pointer-events: none; opacity: .9;
			background: linear-gradient(var(--c), var(--c)) left top / .85em 1.5px no-repeat, linear-gradient(var(--c), var(--c)) left top / 1.5px .85em no-repeat,
				linear-gradient(var(--c), var(--c)) right bottom / .85em 1.5px no-repeat, linear-gradient(var(--c), var(--c)) right bottom / 1.5px .85em no-repeat; }
		.sis-st-linha::after { content: ''; position: absolute; left: 1em; right: 1em; bottom: 0; height: 1px; pointer-events: none; background: linear-gradient(90deg, transparent, color-mix(in srgb, var(--c) 45%, transparent), transparent); }
		.sis-st-linha[data-cor="viagem"]::after { height: 2px; background: repeating-linear-gradient(90deg, var(--c) 0 .55em, transparent .55em 1.1em);
			-webkit-mask-image: linear-gradient(90deg, transparent, #000 12%, #000 88%, transparent); mask-image: linear-gradient(90deg, transparent, #000 12%, #000 88%, transparent); animation: sis-marcha .8s linear infinite; }
		.sis-st-linha[data-cor="minerando"]::after, .sis-st-linha[data-cor="operando"]::after, .sis-st-linha[data-cor="scan"]::after, .sis-st-linha[data-cor="craft"]::after {
			height: 1.5px; background: linear-gradient(90deg, transparent, var(--c), transparent) no-repeat; background-size: 30% 100%; animation: sis-varre 2.8s ease-in-out infinite; }
		.sis-st-linha[data-cor="operando"]::after { animation-duration: 1.3s; }
		.sis-st-linha[data-cor="erro"] { background: linear-gradient(100deg, color-mix(in srgb, var(--c) 17%, transparent), rgba(240, 123, 123, .03) 55%); border-color: rgba(240, 123, 123, .28); }
		.sis-st-linha[data-cor="erro"]::after { height: 2px; background: var(--c); animation: sis-pisca 1.1s ease-in-out infinite; }
		.sis-st-linha[data-cor="parado"] { opacity: .72; } .sis-st-linha[data-cor="parado"]::before, .sis-st-linha[data-cor="livre"]::before { opacity: .4; }
		@keyframes sis-marcha { to { background-position: 1.1em 0; } }
		@keyframes sis-varre { 0% { background-position: -45% 0; } 100% { background-position: 145% 0; } }
		@keyframes sis-pisca { 0%, 100% { opacity: 1; } 50% { opacity: .25; } }
		@keyframes sis-gira { to { transform: rotate(360deg); } }
		@keyframes sis-ping { 0% { transform: scale(.55); opacity: .95; } 100% { transform: scale(1.9); opacity: 0; } }
		@keyframes sis-flash { 0% { box-shadow: 0 0 0 1px #4da7ef, 0 0 18px rgba(77, 167, 239, .5); } 100% { box-shadow: 0 0 0 0 transparent; } }
		.sis-st-linha.sis-movida { animation: sis-flash 1.2s ease-out; }
		.sis-core { position: relative; display: inline-block; flex: none; width: .9em; height: .9em; filter: drop-shadow(0 0 .28em color-mix(in srgb, var(--c, #7890a3) 75%, transparent)); }
		.sis-core::before { content: ''; position: absolute; inset: 0; background: var(--c, #7890a3); clip-path: polygon(50% 0, 93% 25%, 93% 75%, 50% 100%, 7% 75%, 7% 25%); }
		.sis-core::after { content: ''; position: absolute; inset: -.32em; border-radius: 50%; border: 1.5px solid transparent; }
		#assistStatus [data-cor="minerando"] .sis-core::after, #assistStatus [data-cor="viagem"] .sis-core::after, #assistStatus [data-cor="operando"] .sis-core::after, #assistStatus [data-cor="craft"] .sis-core::after {
			border-top-color: var(--c); border-right-color: color-mix(in srgb, var(--c) 35%, transparent); animation: sis-gira 1.5s linear infinite; }
		#assistStatus [data-cor="viagem"] .sis-core::after { animation-duration: .85s; }
		#assistStatus [data-cor="scan"] .sis-core::after { border-color: var(--c); animation: sis-ping 1.6s ease-out infinite; }
		#assistStatus [data-cor="erro"] .sis-core { animation: sis-pisca .9s ease-in-out infinite; }
		#assistStatus [data-cor="parado"] .sis-core, #assistStatus [data-cor="livre"] .sis-core { filter: none; } #assistStatus [data-cor="parado"] .sis-core::before { background: color-mix(in srgb, var(--c) 60%, transparent); }
		.sis-grip { color: var(--t3); cursor: grab; user-select: none; text-align: center; font-size: .95em; letter-spacing: -.2em; line-height: 1; opacity: .45; }
		.sis-st-linha:hover .sis-grip { opacity: 1; color: var(--az); } .sis-grip:active { cursor: grabbing; }
		.sis-st-linha.sis-arrastando { opacity: .35; }
		.sis-st-linha[data-alvo="antes"] { box-shadow: 0 -2px 0 #4da7ef, 0 0 14px rgba(77, 167, 239, .3); } .sis-st-linha[data-alvo="depois"] { box-shadow: 0 2px 0 #4da7ef, 0 0 14px rgba(77, 167, 239, .3); }
		#assistStatus[data-modo="3"] .sis-st-linha[data-alvo="antes"] { box-shadow: -2px 0 0 #4da7ef, 0 0 14px rgba(77, 167, 239, .3); } #assistStatus[data-modo="3"] .sis-st-linha[data-alvo="depois"] { box-shadow: 2px 0 0 #4da7ef, 0 0 14px rgba(77, 167, 239, .3); }
		.sis-st-c1, .sis-st-c2, .sis-st-c3 { min-width: 0; }
		.sis-st-nome { display: block; font-size: 1.04em; margin-bottom: .3em; color: var(--t1); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
		.sis-st-estado { display: flex; align-items: center; gap: .7em; color: var(--t1); min-width: 0; } .sis-st-esttxt { white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
		.sis-st-sub { color: var(--t3); font-size: .92em; margin-top: .25em; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
		.sis-tempo { margin-left: .45em; font-size: .8em; font-variant-numeric: tabular-nums; color: var(--t3); padding: 0 .35em; border-radius: 3px; background: rgba(255,255,255,.06); white-space: nowrap; }
		.sis-ciclo { font-size: .82em; opacity: .85; }
		.sis-st-onde { color: var(--t2); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
		.sis-st-falta { color: var(--t2); font-size: .92em; white-space: nowrap; font-variant-numeric: tabular-nums; }
		.sis-meds { display: grid; grid-template-columns: repeat(4, minmax(0, 1fr)); gap: .8em; }
		.sis-med { min-width: 0; } .sis-med-top { display: flex; justify-content: space-between; gap: .3em; font-size: .84em; color: var(--t3); white-space: nowrap; } .sis-med-top em { font-style: normal; color: var(--t3); }
		.sis-med-val { display: block; margin-top: .25em; font-size: .92em; font-weight: 600; color: var(--t2); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
		.sis-med-barra { height: .4em; margin-top: .25em; border-radius: 99px; background: rgba(255, 255, 255, .07); overflow: hidden; } .sis-med-barra i { display: block; height: 100%; border-radius: 99px; background: #4da7ef; }
		.sis-med.comb .sis-med-barra i { background: #f2a65a; } .sis-med.mun .sis-med-barra i { background: #8f80f3; } .sis-med.carga .sis-med-barra i { background: #44d7f7; } .sis-med-barra i.baixo { background: #f07b7b !important; }
		.sis-mr { --mc: var(--az); position: relative; flex: 1 1 0; min-width: 0; padding-bottom: .4em; font-size: .86em; color: var(--t2); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; font-variant-numeric: tabular-nums; }
		.sis-mr::before, .sis-mr::after { content: ''; position: absolute; left: 0; bottom: 0; height: 2px; border-radius: 2px; }
		.sis-mr::before { right: 0; background: rgba(255, 255, 255, .07); } .sis-mr::after { width: var(--p, 0%); background: var(--mc); box-shadow: 0 0 5px var(--mc); }
		.sis-mr.comb { --mc: #f2a65a; } .sis-mr.mun { --mc: #8f80f3; } .sis-mr.carga { --mc: #44d7f7; } .sis-mr.baixo { --mc: #f07b7b; color: #f07b7b; }
		.sis-st-acoes { display: flex; gap: .25em; justify-content: flex-end; }
		.sis-mini { width: 1.9em; height: 1.9em; border-radius: 4px; border: 1px solid rgba(140, 190, 235, .18); background: #0f1f2d; color: var(--t2); cursor: pointer; font-size: .92em; line-height: 1; padding: 0; }
		.sis-mini:hover:not(:disabled) { color: var(--t1); border-color: #4da7ef; background: rgba(77, 167, 239, .16); }
		.sis-mini.seta { color: var(--t3); font-size: .78em; } .sis-mini:disabled { opacity: .25; cursor: default; }
		.sis-st-det { grid-template-columns: repeat(2, minmax(0, 1fr)); gap: .25em 1.1em; font-size: .9em; padding-top: .5em; border-top: 1px dashed var(--linha); }
		.sis-st-det > div { display: flex; justify-content: space-between; gap: .6em; min-width: 0; } .sis-st-det .k { color: var(--t3); white-space: nowrap; }
		.sis-st-det .v { color: var(--t2); text-align: right; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }

		/* modos: 1 = Linha, 2 = Padrão, 3 = Completo */
		#assistStatus .sis-st-falta, #assistStatus .sis-mini-rec, #assistStatus .sis-st-det { display: none; }
		#assistStatus[data-modo="1"] .sis-st-linha, #assistStatus[data-modo="1"] .sis-st-cabecalho { grid-template-columns: 1.2em minmax(150px, 1.3fr) minmax(170px, 1.5fr) minmax(150px, 1.3fr) minmax(80px, .6fr) minmax(200px, 1.3fr) 8.4em; }
		#assistStatus[data-modo="1"] .sis-st-linha { padding-top: .36em; padding-bottom: .4em; margin-bottom: .2em; }
		#assistStatus[data-modo="1"] .sis-st-c1:not(.craft) { display: flex; align-items: center; gap: .5em; } #assistStatus[data-modo="1"] .sis-st-c1:not(.craft) .sis-st-nome { margin: 0; flex: 0 1 auto; min-width: 0; }
		#assistStatus[data-modo="1"] .sis-chip.mov, #assistStatus[data-modo="1"] .sis-sub-estado, #assistStatus[data-modo="1"] .sis-meds { display: none !important; }
		#assistStatus[data-modo="1"] .sis-st-c3 { display: flex; gap: .45em; align-items: baseline; } #assistStatus[data-modo="1"] .sis-sub-alvo { margin: 0; }
		#assistStatus[data-modo="1"] .sis-st-falta { display: block; } #assistStatus[data-modo="1"] .sis-mini-rec { display: flex; gap: .65em; }
		#assistStatus[data-modo="2"] .sis-st-linha, #assistStatus[data-modo="2"] .sis-st-cabecalho { grid-template-columns: 1.2em minmax(180px, 1.1fr) minmax(210px, 1.3fr) minmax(150px, .9fr) minmax(330px, 2fr) 8.4em; }
		#assistStatus[data-modo="1"] .sis-st-linha.craft, #assistStatus[data-modo="2"] .sis-st-linha.craft { grid-template-columns: 1.2em minmax(180px, 1.1fr) minmax(0, 3fr) 8.4em; }
		#assistStatus[data-modo="3"] #sisStatusFrotas { display: grid; grid-template-columns: repeat(auto-fill, minmax(min(100%, 26em), 1fr)); gap: .55em; align-items: start; }
		#assistStatus[data-modo="3"] .sis-sec, #assistStatus[data-modo="3"] .sis-vazio { grid-column: 1 / -1; }
		#assistStatus[data-modo="3"] .sis-st-linha { grid-template-columns: 1.2em minmax(0, 1fr) auto; grid-template-areas: "g n a" "e e e" "o o o" "m m m" "d d d"; gap: .55em .7em; padding: .75em .85em .85em .6em; margin: 0; }
		#assistStatus[data-modo="3"] .sis-st-linha.craft { grid-template-areas: "g n a" "e e e"; }
		#assistStatus[data-modo="3"] .sis-grip { grid-area: g; } #assistStatus[data-modo="3"] .sis-st-c1 { grid-area: n; } #assistStatus[data-modo="3"] .sis-st-acoes { grid-area: a; }
		#assistStatus[data-modo="3"] .sis-st-c2 { grid-area: e; padding: .45em .6em; background: rgba(0, 0, 0, .2); border: 1px solid var(--linha); border-radius: 3px; }
		#assistStatus[data-modo="3"] .sis-st-c3 { grid-area: o; display: flex; gap: .5em; align-items: baseline; } #assistStatus[data-modo="3"] .sis-sub-alvo { margin: 0; }
		#assistStatus[data-modo="3"] .sis-meds { grid-area: m; } #assistStatus[data-modo="3"] .sis-st-det { grid-area: d; display: grid; }

		/* seções e paginação */
		.sis-sec { --sc: var(--t2); display: flex; align-items: center; gap: .55em; margin: .95em 0 .4em; color: var(--sc); font-size: .84em; font-weight: 700; text-transform: uppercase; letter-spacing: .14em; cursor: pointer; user-select: none; }
		#sisStatusFrotas > .sis-sec:first-child { margin-top: .15em; }
		#assistStatus[data-modo="3"] .sis-sec { margin: .5em 0 0; }
		.sis-sec::after { content: ''; flex: 1; height: 1px; background: linear-gradient(90deg, color-mix(in srgb, var(--sc) 55%, transparent), transparent); }
		.sis-sec b { font-weight: 600; letter-spacing: .03em; text-transform: none; color: var(--t2); padding: .06em .6em; border: 1px solid color-mix(in srgb, var(--sc) 35%, transparent); border-radius: 3px; }
		.sis-sec-seta { font-style: normal; width: 1em; } .sis-sec.fechada { opacity: .65; } .sis-sec:hover { opacity: 1; filter: brightness(1.2); }
		.sis-sec[data-t="Supply"], .sis-sec[data-t="Transport"] { --sc: #f2a65a; } .sis-sec[data-t="Garimpo"], .sis-sec[data-t="Mine"] { --sc: #7fd49a; } .sis-sec[data-t="Prospectar"] { --sc: #b6a6ff; } .sis-sec[data-t="Scan"] { --sc: #8f80f3; } .sis-sec[data-t="Craft"] { --sc: #44d7f7; }
		.sis-vazio { padding: 1.4em; text-align: center; color: var(--t3); border: 1px dashed var(--linha); border-radius: 4px; }
		.sis-pag { display: flex; align-items: center; justify-content: flex-end; flex-wrap: wrap; gap: .4em; margin-top: .75em; color: var(--t3); font-size: .9em; }
		.sis-pag button { min-width: 2.1em; height: 2.1em; font: inherit; border-radius: 3px; border: 1px solid rgba(140, 190, 235, .18); background: #0f1f2d; color: var(--t2); cursor: pointer; }
		.sis-pag button:hover:not(:disabled) { color: var(--t1); border-color: #4da7ef; } .sis-pag button:disabled { opacity: .3; cursor: default; }
		.sis-pag.uma button { display: none; } #sisPagTxt { padding: 0 .5em; font-variant-numeric: tabular-nums; }
		.sis-pag label { margin-left: .8em; display: inline-flex; gap: .4em; align-items: center; } #sisPorPag { font-size: 1em !important; padding: .15em .35em !important; }

		.sis-cards { display: grid; grid-template-columns: repeat(4, minmax(0, 1fr)); gap: 8px; margin: 10px 0; }
		.sis-card { padding: 9px 11px; border-radius: 12px; background: #0f1f2d; border: 1px solid rgba(140, 190, 235, .12); border-top: 2px solid #4da7ef; }
		.sis-card.laranja { border-top-color: #f2a65a; } .sis-card.ciano { border-top-color: #44d7f7; } .sis-card.roxo { border-top-color: #8f80f3; } .sis-card.aviso { border-top-color: #e7c76a; }
		.sis-card .rot { font-size: 10.5px; color: #7890a3; text-transform: uppercase; letter-spacing: .04em; } .sis-card .val { font-size: 18px; font-weight: 600; margin: 2px 0; color: #eef4f9; } .sis-card .det { font-size: 11px; color: #a9bccb; }
		/* [1.4.3] efeitos sempre visíveis: ignora "efeitos de animação" desligados no sistema (prefers-reduced-motion) */
		@media (max-width: 900px) { #assistStatus[data-modo="1"] .sis-st-cabecalho, #assistStatus[data-modo="2"] .sis-st-cabecalho { display: none; }
			#assistStatus[data-modo="1"] .sis-st-linha, #assistStatus[data-modo="2"] .sis-st-linha { grid-template-columns: 1.2em minmax(0, 1fr) minmax(0, 1fr) auto !important; }
			#assistStatus[data-modo="1"] .sis-mini-rec, #assistStatus[data-modo="2"] .sis-meds { grid-column: 2 / -1; } .sis-cards { grid-template-columns: 1fr 1fr; } }`;
	}
	const scanningPatterns = ['square', 'ring', 'spiral', 'up', 'down', 'left', 'right', 'sly', 'auto(1)', 'auto(1+)', 'auto(1,2hv)', 'auto(1,2hv+)', 'auto(1,2hv++)'];
	await loadGlobalSettings();

	let errorLog = [];
	let errorLogIndex = 0;
	let errorLogMaxEntries = 30;
	async function loadErrorLog() {
		let savedErrorLog = await GM.getValue('ErrorLog', '{ "index": 0, "messages": [] }');
		let parsedErrorLog = JSON.parse(savedErrorLog);
		errorLogIndex = parsedErrorLog.index;
		errorLog = parsedErrorLog.messages;
	}
	await loadErrorLog();
	async function logError(msg, fleetName) {
		sisEvento(/Recoverable connection error \(still/.test(String(msg)) ? 'info' : 'erro', fleetName || '', msg);
		let timeStamp = "[" + new Date(Date.now()).toLocaleString("en-GB", { hour12: false }) + "]";
		errorLog[errorLogIndex] = timeStamp + " " + (fleetName ? (fleetName + " ") : '') + msg;
		errorLogIndex++;
		if(errorLogIndex >= errorLogMaxEntries) errorLogIndex = 0;
		let newErrorLog = { "index": errorLogIndex, "messages": errorLog };
		await GM.setValue('ErrorLog', JSON.stringify(newErrorLog));
	}
	let oldOnUnhandledRejection = window.onunhandledrejection;
	window.onunhandledrejection = function(errorEvent) {
		logError("Unhandled exception: " + errorEvent.reason.message + (!!errorEvent.reason.stack ? ("\nStack: " + errorEvent.reason.stack) : '') );
		if(oldOnUnhandledRejection) oldOnUnhandledRejection(errorEvent);
	};

	function cLog(level, ...args) {	if(level <= globalSettings.debugLogLevel) console.log(...args); }

	// ===== [Sis-Ágora] DOM seguro =====
	// Regra: nenhum dado externo (nomes de frota, perfil, receita, starbase, estados, respostas de servidores) entra via innerHTML.
	// Texto -> textContent. Opções de <select> -> createElement. Modelos fixos do próprio script -> DOMParser (setHtmlEstatico).
	function setTxt(el, v) { if(el) el.textContent = (v === undefined || v === null) ? '' : String(v); }
	function preencherOpcoes(select, itens) {
		for (const item of itens) {
			const opt = document.createElement('option');
			if (Array.isArray(item)) { opt.value = String(item[0]); opt.textContent = String(item[1]); }
			else { opt.value = String(item); opt.textContent = String(item); }
			select.appendChild(opt);
		}
	}
	// Só para modelos CONSTANTES escritos neste arquivo (nunca concatenar dados externos aqui).
	function setHtmlEstatico(alvo, html) {
		const doc = new DOMParser().parseFromString('<!DOCTYPE html><html><body>' + html + '</body></html>', 'text/html');
		const nodes = Array.from(doc.body.childNodes).map(n => document.importNode(n, true));
		alvo.replaceChildren(...nodes);
	}
	function celula(tag, texto, attrs) {
		const c = document.createElement(tag);
		if (attrs) for (const k in attrs) c.setAttribute(k, attrs[k]);
		setTxt(c, texto);
		return c;
	}
	// ===== [Sis-Ágora] Segredos =====
	// mySecretKey (e influxAuth) nunca vão para o DOM, console, log de erros, export ou backup.
	function semSegredos(obj) {
		const c = Object.assign({}, obj);
		for (const k of CAMPOS_SECRETOS) if (k in c) c[k] = c[k] ? '(oculto)' : '';
		return c;
	}
	function wait(ms) {	return new Promise(resolve => {	setTimeout(resolve, ms); }); }
	function TimeToStr(date) { return date.toLocaleTimeString("en-GB", { hour12: false, hour: "2-digit", minute: "2-digit" }); }
	function TimeStamp() { return `[${TimeToStr(new Date(Date.now()))}]`; }
	function FleetTimeStamp(fleetName) { return `[${fleetName}] ${TimeStamp()}` }
	function BoolToStr(bool) { return bool ? 'Y' : 'N' };
	function CoordsValid(c) { return !isNaN(parseInt(c[0])) && !isNaN(parseInt(c[1])); }
	function ConvertCoords(coords) { return coords.split(',').map(coord => parseInt(coord.trim()));	}
	function CoordsEqual(a, b) {
		return Array.isArray(a) && Array.isArray(b) &&
			a.length === 2 && b.length === 2 &&
			Number(a[0]) === Number(b[0]) &&
			Number(a[1]) === Number(b[1])
	}
	function parseIntDefault(value, defaultValue) {
		const intValue = parseInt(value);
		return !intValue && intValue !== 0 ? defaultValue : intValue;
	}
	function parseBoolDefault(value, defaultValue) {
		if(typeof value == "boolean") return value;
		if(typeof value == "string")	return value === "true" || value === "false" ? value === "true" : defaultValue;
		return defaultValue;
	}
	function parseStringDefault(value, defaultValue) {
		if(typeof value == "string")	return value;
		return defaultValue;
	}
	function parseStringDefaultForced(value, defaultValue) {
		if(typeof value == "string" && value.length)	return value;
		return defaultValue;
	}


	function parseIntKMG(val) {
		let multiplier = val.substr(-1).toLowerCase();
		if(multiplier == "k")
			return parseInt(val) * 1000;
		else if (multiplier == "m")
			return parseInt(val) * 1000000;
		else if (multiplier == "g")
			return parseInt(val) * 1000000000;
		else
			return parseInt(val);
	}

	async function loadGlobalSettings() {
		const rawSettingsData = await GM.getValue(settingsGmKey, '{}');
		globalSettings = JSON.parse(rawSettingsData);
		globalSettings = {
			// Priority Fee added to each transaction in Lamports. Set to 0 (zero) to disable priority fees. 1 Lamport = 0.000000001 SOL
			priorityFee: parseIntDefault(globalSettings.priorityFee, 1),
			minPriorityFeeForMultiIx: parseIntDefault(globalSettings.minPriorityFeeForMultiIx, 0),

			//autofee
			automaticFee: parseBoolDefault(globalSettings.automaticFee, false),
			automaticFeeStep: parseIntDefault(globalSettings.automaticFeeStep, 80),
			automaticFeeMin: parseIntDefault(globalSettings.automaticFeeMin, 1),
			automaticFeeMax: parseIntDefault(globalSettings.automaticFeeMax, 10000),
			automaticFeeTimeMin: parseIntDefault(globalSettings.automaticFeeTimeMin, 10),
			automaticFeeTimeMax: parseIntDefault(globalSettings.automaticFeeTimeMax, 70),

			craftingTxMultiplier: parseIntDefault(globalSettings.craftingTxMultiplier, 200),
			craftingTxAffectsAutoFee: parseBoolDefault(globalSettings.craftingTxAffectsAutoFee, true),

			transportKeep1: parseBoolDefault(globalSettings.transportKeep1, false),
			transportLoadUnloadSingleTx: parseBoolDefault(globalSettings.transportLoadUnloadSingleTx, false),
			transportUnloadsUnknownRSS: parseBoolDefault(globalSettings.transportUnloadsUnknownRSS, false),
			minerUnloadsAll: parseBoolDefault(globalSettings.minerUnloadsAll, false),
			minerSupplySingleTx: parseBoolDefault(globalSettings.minerSupplySingleTx, false),
			minerKeep1: parseBoolDefault(globalSettings.minerKeep1, false),
			starbaseKeep1: parseBoolDefault(globalSettings.starbaseKeep1, false),
			queueExitWarpSubwarp: parseBoolDefault(globalSettings.queueExitWarpSubwarp, false),



			scanMapURL: parseStringDefaultForced(globalSettings.scanMapURL,'https://slya.de/sdu.json'),


			fleetsPerColumn: parseIntDefault(globalSettings.fleetsPerColumn, 0),

			//Percentage of the priority fees above should be used for all actions except scanning
			//lowPriorityFeeMultiplier: parseIntDefault(globalSettings.lowPriorityFeeMultiplier, 10),

            //Save profile selection to speed up future initialization
            saveProfile: parseBoolDefault(globalSettings.saveProfile, true),
            savedProfile: globalSettings.savedProfile && globalSettings.savedProfile.length > 0 ? globalSettings.savedProfile : [],

			//How many milliseconds to wait before re-reading the chain for confirmation
			confirmationCheckingDelay: parseIntDefault(globalSettings.confirmationCheckingDelay, 2000),

			//How much console logging you want to see (higher number = more, 0 = none)
			debugLogLevel: parseIntDefault(globalSettings.debugLogLevel, 3),

            //The number of crafting jobs enabled in config
            craftingJobs: parseIntDefault(globalSettings.craftingJobs, 4),

            //Subwarp when the distance is 1 diagonal sector or less
            subwarpShortDist: parseBoolDefault(globalSettings.subwarpShortDist, true),

			//Determines if your transports should use their ammo banks to move ammo (in addition to their cargo holds)
			transportUseAmmoBank: parseBoolDefault(globalSettings.transportUseAmmoBank, true),

			//Should transport fleet stop completely if there's an error (example: not enough resource/fuel/etc.)
			transportStopOnError: parseBoolDefault(globalSettings.transportStopOnError, true),

			//If refueling at the source, should transport fleets fill fuel to 100%?
			transportFuel100: parseBoolDefault(globalSettings.transportFuel100, true),

			//Should fleets always load the full ordered amount of consumables needed for operation (fuel, ammo, food)? (otherwise fractions are allowed, but a fleet may do an additional loop until it realizes there is not enough of the rss available)
			fleetForceConsumableAmount: parseBoolDefault(globalSettings.fleetForceConsumableAmount, true),

			//Valid patterns: square, ring, spiral, up, down, left, right, sly
			scanBlockPattern: scanningPatterns.includes(globalSettings.scanBlockPattern) ? globalSettings.scanBlockPattern : 'square',

			//Length of the line-based patterns (only applies to up, down, left and right)
			scanBlockLength: parseIntDefault(globalSettings.scanBlockLength, 5),

			//Start from the beginning of the pattern after resupplying at starbase?
			scanBlockResetAfterResupply: parseBoolDefault(globalSettings.scanBlockResetAfterResupply, false),

			//When true, scanning fleet set to scanMove with low fuel will return to base to resupply fuel + toolkits
			scanResupplyOnLowFuel: parseBoolDefault(globalSettings.scanResupplyOnLowFuel, false),

			//Number of seconds to wait after a successful scan to allow sector to regenerate
			scanSectorRegenTime: parseIntDefault(globalSettings.scanSectorRegenTime, 90),

			//Number of seconds to wait when sectors probabilities are too low
			scanPauseTime: parseIntDefault(globalSettings.scanPauseTime, 600),

			//Number of seconds to scan a low probability sector before giving up and moving on (or pausing)
			scanStrikeCount: parseIntDefault(globalSettings.scanStrikeCount, 3),

			//List of fleets that are handled manually or in another instance (they are excluded here)
			excludeFleets: parseStringDefault(globalSettings.excludeFleets,''),

			//How transparent the status panel should be (1 = completely opaque)
			statusPanelOpacity: parseIntDefault(globalSettings.statusPanelOpacity, 75),

			//Should assistant automatically start after initialization is complete?
			autoStartScript: parseBoolDefault(globalSettings.autoStartScript, false),

			//How many fleets need to stall before triggering an automatic page reload? (0 = never trigger)
			reloadPageOnFailedFleets: parseIntDefault(globalSettings.reloadPageOnFailedFleets, 0),

			//Custom secret key (replaces Solflare/Phantom signing)
			mySecretKey: parseStringDefault(globalSettings.mySecretKey,''),
		}

		// [Sis-Ágora] tetos locais: valem acima de qualquer configuração vinda da VPS
		const tetoTaxa = Math.max(0, Math.min(50000, parseInt(sisLocal.tetoTaxa) || 0));
		for (const k of ['priorityFee', 'minPriorityFeeForMultiIx', 'automaticFeeMin', 'automaticFeeMax']) globalSettings[k] = Math.min(globalSettings[k], tetoTaxa);
		globalSettings.craftingTxMultiplier = Math.min(globalSettings.craftingTxMultiplier, Math.max(1, Math.min(500, parseInt(sisLocal.tetoMultCraft) || 200)));
		try { if (currentFee > tetoTaxa) currentFee = tetoTaxa; } catch (e) { }
		// [Sis-Ágora] 1.2.0: ajustes que só o Sis-Bot tem (fora da lista do SLYA) também entram no globalSettings
		// (antes ficavam só no GM: a taxa automática p/ várias instruções nunca chegava a ligar)
		try { const bruto = JSON.parse(rawSettingsData); for (const k in SIS_GLOBAL) if (!(k in globalSettings)) { try { globalSettings[k] = sisCampo(k, bruto[k], SIS_GLOBAL[k], 'Ajustes'); } catch (e) { globalSettings[k] = SIS_GLOBAL[k][1]; } } } catch (e) { for (const k in SIS_GLOBAL) if (!(k in globalSettings)) globalSettings[k] = SIS_GLOBAL[k][1]; }
		cLog(2, 'SYSTEM: Global Settings loaded', semSegredos(globalSettings));
	}

	//statsadd start
	//Transaction statistics by Risingson/EveEye, small improvements by Swift42
	let transactionStats={ "start": (Math.round(Date.now() / 1000)), "groups":{} };
	async function alterStats(group,name,val,unit,precision) {
		//let stats = JSON.parse(await GM.getValue('statistics', '{}'));
		let started = new Date(transactionStats.start*1000);
		if (!transactionStats.groups[group]) transactionStats.groups[group]={"TOTAL":{"count":0,"value":0,"last":0,"unit":unit,"precision":precision}};
		if (name && !transactionStats.groups[group][name]) transactionStats.groups[group][name]={"count":0,"value":0,"last":0};
		if (name) {
			transactionStats.groups[group][name].count += 1;
			transactionStats.groups[group][name].value += val;
			transactionStats.groups[group][name].last = val;
		}
		transactionStats.groups[group].TOTAL.count += 1;
		transactionStats.groups[group].TOTAL.value += val;
		transactionStats.groups[group].TOTAL.last = val;

		// update ui ([Sis-Ágora] montado com DOM, sem innerHTML)
		let groups = transactionStats.groups;
		const tabela = document.createElement('table');
		const tr1 = document.createElement('tr');
		tr1.appendChild(celula('td', 'Started: '+started.toLocaleDateString()+' '+started.toLocaleTimeString()+' / Hours passed: '+((Date.now()-started)/1000/60/60).toFixed(2), {colspan: '4'}));
		tabela.appendChild(tr1);
        let tempTotalRequests = solanaReadCount + solanaWriteCount;
        let tempMinsPassed = (Date.now()-started)/1000/60;
		const tr2 = document.createElement('tr');
		tr2.appendChild(celula('td', 'RPC Requests: ' + solanaReadCount + ' reads | ' + solanaWriteCount + ' writes | ' + (tempTotalRequests / tempMinsPassed).toFixed(2) + ' per minute | ' + solanaErrorCount + ' errors', {colspan: '4'}));
		tabela.appendChild(tr2);
		for (let group in groups) {
			const trG = document.createElement('tr');
			trG.style.opacity = '0.66';
			const unit = groups[group].TOTAL.unit;
			trG.appendChild(celula('td', group));
			trG.appendChild(celula('td', 'Count', {align: 'right'}));
			trG.appendChild(celula('td', 'Total ' + unit, {align: 'right'}));
			trG.appendChild(celula('td', 'Average ' + unit, {align: 'right'}));
			trG.appendChild(celula('td', 'Last ' + unit, {align: 'right'}));
			tabela.appendChild(trG);
			let precision = +groups[group].TOTAL.precision;
			for (let item in groups[group]) {
				let avg = groups[group][item].value/groups[group][item].count;
				const tr = document.createElement('tr');
				tr.appendChild(celula('td', item));
				tr.appendChild(celula('td', groups[group][item].count, {align: 'right'}));
				tr.appendChild(celula('td', groups[group][item].value.toFixed(precision), {align: 'right'}));
				tr.appendChild(celula('td', avg.toFixed(precision), {align: 'right'}));
				tr.appendChild(celula('td', groups[group][item].last.toFixed(precision), {align: 'right'}));
				tabela.appendChild(tr);
			}
		}
		const statsEl = document.querySelector('#assistStatsContent');
		if (statsEl) statsEl.replaceChildren(sisStatsResumo(), tabela);
	}
	//statsadd end

	//autofee
	async function alterFees(seconds,opName) {
		if((!globalSettings.craftingTxAffectsAutoFee) && (opName.includes('CRAFT') || opName.includes('UPGRADE'))) return;
		const proportionFee = (globalSettings.automaticFeeMax <= globalSettings.automaticFeeMin) ? 1 : (currentFee - globalSettings.automaticFeeMin) / ( globalSettings.automaticFeeMax - globalSettings.automaticFeeMin );
		let thresholdTime = (globalSettings.automaticFeeTimeMax - globalSettings.automaticFeeTimeMin) * proportionFee + globalSettings.automaticFeeTimeMin;
		if(thresholdTime < globalSettings.automaticFeeTimeMin) { thresholdTime = globalSettings.automaticFeeTimeMin; }
		if(thresholdTime > globalSettings.automaticFeeTimeMax) { thresholdTime = globalSettings.automaticFeeTimeMax; }
		let change=0;
		if(globalSettings.automaticFee) {
			if(seconds == -1) {
				// tx was resent. We need to adapt fast, so use max fee increase here
				change = globalSettings.automaticFeeStep;
			} else {
				if(seconds < thresholdTime) {
					let factor = (thresholdTime - seconds) / (globalSettings.automaticFeeTimeMax - globalSettings.automaticFeeTimeMin);
					if(factor > 1) { factor = 1; }
					change = Math.round(factor * globalSettings.automaticFeeStep * -1);
				} else {
					let factor = (seconds - thresholdTime) / (globalSettings.automaticFeeTimeMax - globalSettings.automaticFeeTimeMin);
					if(factor > 1) { factor = 1; }
					change = Math.round(factor * globalSettings.automaticFeeStep);
				}
			}
			currentFee += change;
			if(currentFee < globalSettings.automaticFeeMin) {
				currentFee = globalSettings.automaticFeeMin;
			}
			if(currentFee > globalSettings.automaticFeeMax) {
				currentFee = globalSettings.automaticFeeMax;
			}
		} else {
			currentFee = globalSettings.priorityFee;
		}
		cLog(3, `Fee change data: Seconds `, seconds, `, thresholdTime `, thresholdTime, `, change `, change, `, new fee: `, currentFee);

		sisAtualizarChipTaxa();
	}

	async function doProxyStuff(target, origMethod, args, rpcs, proxyType)
	{
		function isConnectivityError(error) {
			const m = String(error && error.message || error);
			return (
				(Number(m.slice(0,3)) > 299) ||
				(m === 'Failed to fetch') ||
				m.includes('failed to get') ||
				m.includes('failed to send') ||
				m.includes('NetworkError') ||
				m.includes('Unable to complete request') ||
				m.includes('Sis-Bot: nenhum RPC') ||
				m.includes('429') || m.includes('503') || m.includes('timeout') || m.includes('aborted')
			);
		}
		let espera = 1000;
		for (let tentativa = 1; ; tentativa++) {
			try {
				return await origMethod.apply(target, args);
			} catch (error1) {
				solanaErrorCount++;
				if (!isConnectivityError(error1)) { logError('Unrecoverable connection error: ' + sisRpcMascarar(error1)); return undefined; }
				cLog(2, `${proxyType} CONNECTION ERROR (tentativa ${tentativa}): `, sisRpcMascarar(error1));
				if (tentativa === 1 || tentativa % 10 === 0) logError(`Nenhum RPC respondeu (${proxyType}); tentando de novo em ${Math.round(espera / 1000)} s: ` + sisRpcMascarar(error1));
				try { sisRpcAtualizarChip(); } catch (e) { }
				await wait(espera);
				espera = Math.min(espera * 2, 30000);
			}
		}
	}
	const writeConnectionProxy = {
		get(target, key, receiver) {
			const origMethod = target[key];
			if(typeof origMethod === 'function'){
				return async function (...args) {
					solanaWriteCount++;
					return await doProxyStuff(target, origMethod, args, writeRPCs, 'WRITE');
				}
			}
		},
	}
	const readConnectionProxy = {
		get(target, key, receiver) {
			const origMethod = target[key];
			if(typeof origMethod === 'function'){
				return async function (...args) {
					solanaReadCount++;
					return await doProxyStuff(target, origMethod, args, readRPCs, 'READ');
				}
			}
		},
	}

	// [Sis-Ágora 1.3.0] as duas conexões usam o pool (sisRpcFetch): leitura escolhe o melhor RPC, envio vai para todos
	const rawSolanaReadConnection = sisRpcNovaConexao();
	const solanaReadConnection = new Proxy(rawSolanaReadConnection, readConnectionProxy);
	const rawSolanaWriteConnection = sisRpcNovaConexao();
	const solanaWriteConnection = new Proxy(rawSolanaWriteConnection, writeConnectionProxy);
    let cachedEpochInfo = {'blockHeight': 0, 'lastUpdated': 0, 'isUpdating': false};
	let solanaReadCount = 0;
	let solanaWriteCount = 0;
	let tokenCheckCounter = 0;
	let fleetStatusCount=0;
	let fleetStatusCurColumn=0;
    let globalErrorTracker = {'firstErrorTime': 0, 'errorCount': 0};
    cLog(1, `RPCs (em ordem): ${sisRpcPool.map(e => e.nome + ' [' + e.host + ']').join(', ')}`);

	//Not sure what this does, but it seems to do some reads, so sticking it on the read connection
	const anchorProvider = new BrowserAnchor.anchor.AnchorProvider(solanaReadConnection, null, null);
	window.Buffer = BrowserBuffer.Buffer.Buffer;

	//IDL Definitions
    //const sageIDL = await BrowserAnchor.anchor.Program.fetchIdl(sageProgramPK, anchorProvider);
    const sageIDL = {version: "0.1.0",name: "sage",instructions: [{name: "activateGameState",accounts: [{name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !0,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !0,isSigner: !1,docs: ["The [`GameState`] account"]}],args: [{name: "input",type: {defined: "ManageGameInput"}}]}, {name: "addConnection",accounts: [{name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for adding the connection"]}, {name: "sector1",isMut: !0,isSigner: !1,docs: ["The first connected sector"]}, {name: "sector2",isMut: !0,isSigner: !1,docs: ["The second connected sector"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The system program"]}],args: [{name: "subCoordinates1",type: {array: ["i64", 2]}}, {name: "flags1",type: "u8"}, {name: "subCoordinates2",type: {array: ["i64", 2]}}, {name: "flags2",type: "u8"}, {name: "keyIndex",type: "u16"}]}, {name: "addRental",accounts: [{name: "ownerProfile",isMut: !1,isSigner: !1,docs: ["The fleet owner's profile."]}, {name: "ownerKey",isMut: !1,isSigner: !0,docs: ["The key on the owner profile with renting permissions."]}, {name: "invalidator",isMut: !1,isSigner: !0,docs: ["The fleet rental invalidator - this is a signer to help make sure the fleet won't be locked."]}, {name: "subProfile",isMut: !1,isSigner: !1,docs: ["The profile to rent to."]}, {name: "fleet",isMut: !0,isSigner: !1,docs: ["The fleet to rent out."]}, {name: "gameAccounts",accounts: [{name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}],args: [{name: "ownerKeyIndex",type: "u16"}]}, {name: "addShipEscrow",accounts: [{name: "funder",isMut: !0,isSigner: !0,docs: ["The funder - pays for account rent"]}, {name: "sagePlayerProfile",isMut: !0,isSigner: !1,docs: ["The [`SagePlayerProfile`] account"]}, {name: "originTokenAccount",isMut: !0,isSigner: !1,docs: ["The Origin Token Account"]}, {name: "ship",isMut: !1,isSigner: !1,docs: ["The [`Ship`] Account"]}, {name: "shipEscrowTokenAccount",isMut: !0,isSigner: !1,docs: ["The Escrow Token Account"]}, {name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !0,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "gameAccountsAndProfile",accounts: [{name: "gameAndProfileAndFaction",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "profileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["The Token Program"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The Solana System program"]}],args: [{name: "input",type: {defined: "AddShipEscrowInput"}}]}, {name: "addShipToFleet",accounts: [{name: "gameAccountsFleetAndOwner",accounts: [{name: "gameFleetAndOwner",accounts: [{name: "fleetAndOwner",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key on the profile."]}, {name: "owningProfile",isMut: !1,isSigner: !1,docs: ["The profile that owns the fleet."]}, {name: "owningProfileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "fleet",isMut: !0,isSigner: !1,docs: ["The fleet."]}]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the new `Fleet`"]}, {name: "fleetShips",isMut: !0,isSigner: !1,docs: ["The [`FleetShips`] account"]}, {name: "ship",isMut: !1,isSigner: !1,docs: ["The [`Ship`] Account"]}, {name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !0,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The Solana System program"]}],args: [{name: "input",type: {defined: "AddShipToFleetInput"}}]}, {name: "burnCraftingConsumables",accounts: [{name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !1,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "craftingInstance",isMut: !1,isSigner: !1,docs: ["The [`CraftingInstance`] account"]}, {name: "craftingProcess",isMut: !0,isSigner: !1,docs: ["The crafting process account"]}, {name: "craftingFacility",isMut: !1,isSigner: !1,docs: ["The crafting facility account"]}, {name: "craftingRecipe",isMut: !1,isSigner: !1,docs: ["The crafting recipe"]}, {name: "gameAccounts",accounts: [{name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "tokenFrom",isMut: !0,isSigner: !1,docs: ["The source account of the tokens - owner should be `crafting_process`"]}, {name: "tokenMint",isMut: !0,isSigner: !1,docs: ["The token mint"]}, {name: "craftingProgram",isMut: !1,isSigner: !1,docs: ["The Crafting Program"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["The [Token] program"]}],args: [{name: "input",type: {defined: "IngredientIndexInput"}}]}, {name: "cancelCraftingProcess",accounts: [{name: "fundsTo",isMut: !0,isSigner: !1,docs: ["The funds_to - receives rent refund"]}, {name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !0,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "craftingInstance",isMut: !0,isSigner: !1,docs: ["The [`CraftingInstance`] account to cancel"]}, {name: "craftingProcess",isMut: !0,isSigner: !1,docs: ["The crafting process account"]}, {name: "craftingFacility",isMut: !0,isSigner: !1,docs: ["The `CraftingFacility` account"]}, {name: "gameAccountsAndProfile",accounts: [{name: "gameAndProfileAndFaction",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "profileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "craftingProgram",isMut: !1,isSigner: !1,docs: ["The Crafting Program"]}],args: [{name: "input",type: {defined: "KeyIndexInput"}}]}, {name: "changeRental",accounts: [{name: "subProfileInvalidator",isMut: !1,isSigner: !0,docs: ["The fleet rental invalidator"]}, {name: "newSubProfile",isMut: !1,isSigner: !1,docs: ["The new sub profile"]}, {name: "fleet",isMut: !0,isSigner: !1,docs: ["The fleet"]}, {name: "gameAccounts",accounts: [{name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}],args: []}, {name: "claimCraftingNonConsumables",accounts: [{name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !1,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "craftingInstance",isMut: !1,isSigner: !1,docs: ["The [`CraftingInstance`] account"]}, {name: "craftingProcess",isMut: !0,isSigner: !1,docs: ["The crafting process account"]}, {name: "craftingFacility",isMut: !1,isSigner: !1,docs: ["The crafting facility account"]}, {name: "craftingRecipe",isMut: !1,isSigner: !1,docs: ["The crafting recipe"]}, {name: "cargoPodTo",isMut: !0,isSigner: !1,docs: ["The destination cargo pod account"]}, {name: "cargoType",isMut: !1,isSigner: !1,docs: ["The Cargo Type Account"]}, {name: "cargoStatsDefinition",isMut: !1,isSigner: !1,docs: ["The cargo stats definition account"]}, {name: "gameAccounts",accounts: [{name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "tokenFrom",isMut: !0,isSigner: !1,docs: ["The source account of the tokens - owner should be `crafting_process`"]}, {name: "tokenTo",isMut: !0,isSigner: !1,docs: ["The destination account of the tokens - owner should be `cargo_pod_to`"]}, {name: "tokenMint",isMut: !0,isSigner: !1,docs: ["The token mint"]}, {name: "craftingProgram",isMut: !1,isSigner: !1,docs: ["The Crafting Program"]}, {name: "cargoProgram",isMut: !1,isSigner: !1,docs: ["The Cargo Program"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["The [Token] program"]}],args: [{name: "input",type: {defined: "IngredientIndexInput"}}]}, {name: "claimCraftingOutputs",accounts: [{name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !1,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "craftingInstance",isMut: !1,isSigner: !1,docs: ["The [`CraftingInstance`] account"]}, {name: "craftingProcess",isMut: !0,isSigner: !1,docs: ["The [`CraftingProcess`] account"]}, {name: "craftingFacility",isMut: !1,isSigner: !1,docs: ["The crafting facility account"]}, {name: "craftingRecipe",isMut: !1,isSigner: !1,docs: ["The crafting [`Recipe`]"]}, {name: "craftableItem",isMut: !1,isSigner: !1,docs: ["The craftable item"]}, {name: "cargoPodTo",isMut: !0,isSigner: !1,docs: ["The destination cargo pod account"]}, {name: "cargoType",isMut: !1,isSigner: !1,docs: ["The Cargo Type Account"]}, {name: "cargoStatsDefinition",isMut: !1,isSigner: !1,docs: ["The cargo stats definition account"]}, {name: "gameAccounts",accounts: [{name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "tokenFrom",isMut: !0,isSigner: !1,docs: ["The source account of the tokens - owner should be `craftable_item`"]}, {name: "tokenTo",isMut: !0,isSigner: !1,docs: ["The destination account of the tokens - owner should be `cargo_pod_to`"]}, {name: "craftingProgram",isMut: !1,isSigner: !1,docs: ["The Crafting Program"]}, {name: "cargoProgram",isMut: !1,isSigner: !1,docs: ["The Cargo Program"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["The [Token] program"]}],args: [{name: "input",type: {defined: "IngredientIndexInput"}}]}, {name: "closeCraftingProcess",accounts: [{name: "fundsTo",isMut: !0,isSigner: !1,docs: ["The funds_to - receives rent refund"]}, {name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !0,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "craftingInstance",isMut: !0,isSigner: !1,docs: ["The [`CraftingInstance`] account to close"]}, {name: "craftingProcess",isMut: !0,isSigner: !1,docs: ["The crafting process account"]}, {name: "craftingRecipe",isMut: !1,isSigner: !1,docs: ["The crafting recipe account"]}, {name: "craftingFacility",isMut: !0,isSigner: !1,docs: ["The `CraftingFacility` account"]}, {name: "gameAccountsAndProfile",accounts: [{name: "gameAndProfileAndFaction",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "profileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "craftingXpAccounts",accounts: [{name: "userPointsAccount",isMut: !0,isSigner: !1,docs: ["The User Points Account"]}, {name: "pointsCategory",isMut: !1,isSigner: !1,docs: ["The Points Category Account"]}, {name: "pointsModifierAccount",isMut: !1,isSigner: !1,docs: ["The Points Modifier Account"]}]}, {name: "councilRankXpAccounts",accounts: [{name: "userPointsAccount",isMut: !0,isSigner: !1,docs: ["The User Points Account"]}, {name: "pointsCategory",isMut: !1,isSigner: !1,docs: ["The Points Category Account"]}, {name: "pointsModifierAccount",isMut: !1,isSigner: !1,docs: ["The Points Modifier Account"]}]}, {name: "progressionConfig",isMut: !1,isSigner: !1,docs: ["The [`ProgressionConfig`] account"]}, {name: "pointsProgram",isMut: !1,isSigner: !1,docs: ["The points program"]}, {name: "craftingProgram",isMut: !1,isSigner: !1,docs: ["The Crafting Program"]}],args: [{name: "input",type: {defined: "KeyIndexInput"}}]}, {name: "closeDisbandedFleet",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key on the player profile."]}, {name: "playerProfile",isMut: !1,isSigner: !1,docs: ["The player profile."]}, {name: "fundsTo",isMut: !0,isSigner: !1,docs: ["The funds_to - receives rent refund"]}, {name: "disbandedFleet",isMut: !0,isSigner: !1,docs: ["The [`DisbandedFleet`] account"]}, {name: "fleetShips",isMut: !0,isSigner: !1,docs: ["The [`FleetShips`] account"]}],args: [{name: "input",type: {defined: "CloseDisbandedFleetInput"}}]}, {name: "closeFleetCargoPodTokenAccount",accounts: [{name: "gameAccountsFleetAndOwner",accounts: [{name: "gameFleetAndOwner",accounts: [{name: "fleetAndOwner",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key on the profile."]}, {name: "owningProfile",isMut: !1,isSigner: !1,docs: ["The profile that owns the fleet."]}, {name: "owningProfileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "fleet",isMut: !0,isSigner: !1,docs: ["The fleet."]}]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "cargoPod",isMut: !0,isSigner: !1,docs: ["The cargo pod, owned by the fleet"]}, {name: "cargoType",isMut: !1,isSigner: !1,docs: ["The cargo type account"]}, {name: "cargoStatsDefinition",isMut: !1,isSigner: !1,docs: ["The cargo stats definition account"]}, {name: "token",isMut: !0,isSigner: !1,docs: ["The destination token account - owned by the `cargo_pod`"]}, {name: "tokenMint",isMut: !0,isSigner: !1,docs: ["The mint of the token accounts"]}, {name: "fundsTo",isMut: !0,isSigner: !1,docs: ["The funds_to - receives rent refund"]}, {name: "cargoProgram",isMut: !1,isSigner: !1,docs: ["The Cargo Program"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["Token Program"]}],args: [{name: "input",type: {defined: "KeyIndexInput"}}]}, {name: "closeStarbaseCargoTokenAccount",accounts: [{name: "fundsTo",isMut: !0,isSigner: !1,docs: ["The funds_to - receives rent refund"]}, {name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !1,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "gameAccountsAndProfile",accounts: [{name: "gameAndProfileAndFaction",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "profileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "cargoPod",isMut: !0,isSigner: !1,docs: ["The new cargo pod"]}, {name: "cargoType",isMut: !1,isSigner: !1,docs: ["The cargo type account"]}, {name: "cargoStatsDefinition",isMut: !1,isSigner: !1,docs: ["The cargo stats definition account"]}, {name: "token",isMut: !0,isSigner: !1,docs: ["The source token account - owned by the `cargo_pod`"]}, {name: "tokenMint",isMut: !0,isSigner: !1,docs: ["The mint of the token accounts"]}, {name: "cargoProgram",isMut: !1,isSigner: !1,docs: ["The Cargo Program"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["Token Program"]}],args: [{name: "input",type: {defined: "KeyIndexInput"}}]}, {name: "closeUpgradeProcess",accounts: [{name: "fundsTo",isMut: !0,isSigner: !1,docs: ["The funds_to - receives rent refund"]}, {name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !0,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "resourceCraftingInstance",isMut: !0,isSigner: !1,docs: ["The [`CraftingInstance`] account to close"]}, {name: "resourceCraftingProcess",isMut: !0,isSigner: !1,docs: ["The crafting process account"]}, {name: "resourceRecipe",isMut: !1,isSigner: !1,docs: ["The crafting recipe for the upgrade resource"]}, {name: "resourceCraftingFacility",isMut: !0,isSigner: !1,docs: ["The `CraftingFacility` account"]}, {name: "gameAccountsAndProfile",accounts: [{name: "gameAndProfileAndFaction",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "profileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "craftingProgram",isMut: !1,isSigner: !1,docs: ["The Crafting Program"]}],args: [{name: "input",type: {defined: "KeyIndexInput"}}]}, {name: "completeStarbaseUpgrade",accounts: [{name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the new crafting process"]}, {name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !0,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !1,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "craftingFacility",isMut: !0,isSigner: !1,docs: ["The `CraftingFacility` account for crafting at this `Starbase`"]}, {name: "upgradeFacility",isMut: !1,isSigner: !1,docs: ["The `CraftingFacility` account for starbase upgrades"]}, {name: "upgradeRecipe",isMut: !1,isSigner: !1,docs: ["The crafting recipe for the starbase upgrade"]}, {name: "newRecipeCategory",isMut: !1,isSigner: !1,docs: ["The crafting recipe category for the next `Starbase` level"]}, {name: "craftingDomain",isMut: !1,isSigner: !1,docs: ["The crafting domain"]}, {name: "gameAccountsAndProfile",accounts: [{name: "gameAndProfileAndFaction",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "profileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "craftingProgram",isMut: !1,isSigner: !1,docs: ["The Crafting Program"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["Solana System program"]}],args: [{name: "input",type: {defined: "KeyIndexInput"}}]}, {name: "copyGameState",accounts: [{name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the new `GameState`"]}, {name: "oldGameState",isMut: !1,isSigner: !1,docs: ["The old [`GameState`] account"]}, {name: "newGameState",isMut: !0,isSigner: !1,docs: ["The [`GameState`] account", "This will and should fail if there already exists a `GameState`for the desired `update_id`"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The system program"]}],args: [{name: "input",type: {defined: "ManageGameInput"}}]}, {name: "createCargoPod",accounts: [{name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the new crafting process"]}, {name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !1,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "cargoPod",isMut: !0,isSigner: !1,docs: ["The new cargo pod"]}, {name: "cargoStatsDefinition",isMut: !1,isSigner: !1,docs: ["The cargo stats definition account"]}, {name: "gameAccountsAndProfile",accounts: [{name: "gameAndProfileAndFaction",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "profileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "cargoProgram",isMut: !1,isSigner: !1,docs: ["The Cargo Program"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The system program"]}],args: [{name: "input",type: {defined: "StarbaseCreateCargoPodInput"}}]}, {name: "createCertificateMint",accounts: [{name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the new crafting process"]}, {name: "starbase",isMut: !1,isSigner: !1,docs: ["The Starbase to create a certificate mint for"]}, {name: "cargoMint",isMut: !1,isSigner: !1,docs: ["The mint to create a certificate mint for"]}, {name: "certificateMint",isMut: !0,isSigner: !1,docs: ["The new certificate mint"]}, {name: "cargoType",isMut: !1,isSigner: !1,docs: ["The cargo type to associated with the `cargo_mint`", "Included to ensure that this instruction can only be called for valid cargo types"]}, {name: "rent",isMut: !1,isSigner: !1,docs: ["The rent sysvar"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["The token program"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The system program"]}],args: []}, {name: "createCraftingProcess",accounts: [{name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the new crafting process"]}, {name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !0,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "craftingInstance",isMut: !0,isSigner: !1,docs: ["The [`CraftingInstance`] account to initialize"]}, {name: "craftingFacility",isMut: !0,isSigner: !1,docs: ["The `CraftingFacility` account"]}, {name: "craftingProcess",isMut: !0,isSigner: !1,docs: ["The crafting process account (NOT initialized)"]}, {name: "craftingRecipe",isMut: !1,isSigner: !1,docs: ["The crafting recipe"]}, {name: "craftingDomain",isMut: !1,isSigner: !1,docs: ["The crafting domain"]}, {name: "gameAccountsAndProfile",accounts: [{name: "gameAndProfileAndFaction",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "profileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "craftingProgram",isMut: !1,isSigner: !1,docs: ["The Crafting Program"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["Solana System program"]}],args: [{name: "input",type: {defined: "StarbaseCreateCraftingProcessInput"}}]}, {name: "createFleet",accounts: [{name: "gameAccountsAndProfile",accounts: [{name: "gameAndProfileAndFaction",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "profileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the new `Fleet`"]}, {name: "fleet",isMut: !0,isSigner: !1,docs: ["The [`Fleet`] account"]}, {name: "fleetShips",isMut: !0,isSigner: !1,docs: ["The [`FleetShips`] account"]}, {name: "cargoHold",isMut: !0,isSigner: !1,docs: ["The new fleet `cargo_hold` cargo pod (not initialized)"]}, {name: "fuelTank",isMut: !0,isSigner: !1,docs: ["The new fleet `fuel_tank` cargo pod (not initialized)"]}, {name: "ammoBank",isMut: !0,isSigner: !1,docs: ["The new fleet `ammo_bank` cargo pod (not initialized)"]}, {name: "ship",isMut: !1,isSigner: !1,docs: ["The [`Ship`] Account - represents the first ship in the new fleet"]}, {name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !0,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "cargoStatsDefinition",isMut: !1,isSigner: !1,docs: ["The cargo stats definition account"]}, {name: "cargoProgram",isMut: !1,isSigner: !1,docs: ["The Cargo Program"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The Solana System program"]}],args: [{name: "input",type: {defined: "CreateFleetInput"}}]}, {name: "createStarbaseUpgradeResourceProcess",accounts: [{name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the new crafting process"]}, {name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !0,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "craftingInstance",isMut: !0,isSigner: !1,docs: ["The [`CraftingInstance`] account to initialize"]}, {name: "upgradeFacility",isMut: !0,isSigner: !1,docs: ["The `CraftingFacility` account"]}, {name: "craftingProcess",isMut: !0,isSigner: !1,docs: ["The crafting process account (NOT initialized)"]}, {name: "craftingRecipe",isMut: !1,isSigner: !1,docs: ["The crafting recipe"]}, {name: "craftingDomain",isMut: !1,isSigner: !1,docs: ["The crafting domain"]}, {name: "gameAccountsAndProfile",accounts: [{name: "gameAndProfileAndFaction",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "profileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "craftingProgram",isMut: !1,isSigner: !1,docs: ["The Crafting Program"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["Solana System program"]}],args: [{name: "input",type: {defined: "StarbaseCreateCraftingProcessInput"}}]}, {name: "depositCargoToFleet",accounts: [{name: "gameAccountsFleetAndOwner",accounts: [{name: "gameFleetAndOwner",accounts: [{name: "fleetAndOwner",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key on the profile."]}, {name: "owningProfile",isMut: !1,isSigner: !1,docs: ["The profile that owns the fleet."]}, {name: "owningProfileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "fleet",isMut: !0,isSigner: !1,docs: ["The fleet."]}]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "fundsTo",isMut: !0,isSigner: !1,docs: ["The funds_to - receives rent refund"]}, {name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !1,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "cargoPodFrom",isMut: !0,isSigner: !1,docs: ["The origin cargo pod"]}, {name: "cargoPodTo",isMut: !0,isSigner: !1,docs: ["The destination cargo pod"]}, {name: "cargoType",isMut: !1,isSigner: !1,docs: ["The cargo type account"]}, {name: "cargoStatsDefinition",isMut: !1,isSigner: !1,docs: ["The cargo stats definition account"]}, {name: "tokenFrom",isMut: !0,isSigner: !1,docs: ["The source token account - owned by the `cargo_pod_from`"]}, {name: "tokenTo",isMut: !0,isSigner: !1,docs: ["The destination token account - owned by the `cargo_pod_to`"]}, {name: "tokenMint",isMut: !0,isSigner: !1,docs: ["The mint of the token accounts"]}, {name: "cargoProgram",isMut: !1,isSigner: !1,docs: ["The Cargo Program"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["Token Program"]}],args: [{name: "input",type: {defined: "DepositCargoToFleetInput"}}]}, {name: "depositCargoToGame",accounts: [{name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !1,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "cargoPod",isMut: !0,isSigner: !1,docs: ["The new cargo pod"]}, {name: "cargoType",isMut: !1,isSigner: !1,docs: ["The cargo type account"]}, {name: "cargoStatsDefinition",isMut: !1,isSigner: !1,docs: ["The cargo stats definition account"]}, {name: "gameAccountsAndProfile",accounts: [{name: "gameAndProfileAndFaction",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "profileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "tokenFrom",isMut: !0,isSigner: !1,docs: ["The source token account - owned by the `key`"]}, {name: "tokenTo",isMut: !0,isSigner: !1,docs: ["The destination token account - owned by the `cargo_pod`"]}, {name: "cargoProgram",isMut: !1,isSigner: !1,docs: ["The Cargo Program"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["Token Program"]}],args: [{name: "input",type: {defined: "CargoToGameInput"}}]}, {name: "depositCraftingIngredient",accounts: [{name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !1,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "craftingInstance",isMut: !1,isSigner: !1,docs: ["The [`CraftingInstance`] account"]}, {name: "craftingFacility",isMut: !1,isSigner: !1,docs: ["The [`CraftingFacility`](crafting::CraftingFacility) account"]}, {name: "craftingProcess",isMut: !0,isSigner: !1,docs: ["The crafting process account"]}, {name: "cargoPodFrom",isMut: !0,isSigner: !1,docs: ["The source cargo pod account"]}, {name: "craftingRecipe",isMut: !1,isSigner: !1,docs: ["The crafting recipe"]}, {name: "cargoType",isMut: !1,isSigner: !1,docs: ["The Cargo Type Account"]}, {name: "cargoStatsDefinition",isMut: !1,isSigner: !1,docs: ["The cargo stats definition account"]}, {name: "gameAccountsAndProfile",accounts: [{name: "gameAndProfileAndFaction",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "profileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "tokenFrom",isMut: !0,isSigner: !1,docs: ["The source account of the tokens - owner should be `cargo_pod_from`"]}, {name: "tokenTo",isMut: !0,isSigner: !1,docs: ["The destination account of the tokens - owner should be `crafting_process`"]}, {name: "craftingProgram",isMut: !1,isSigner: !1,docs: ["The Crafting Program"]}, {name: "cargoProgram",isMut: !1,isSigner: !1,docs: ["The Cargo Program"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["The [Token] program"]}],args: [{name: "input",type: {defined: "StarbaseDepositCraftingIngredientInput"}}]}, {name: "depositStarbaseUpkeepResource",accounts: [{name: "fundsTo",isMut: !0,isSigner: !1,docs: ["The funds_to - receives rent refund"]}, {name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !0,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !1,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "cargoPodFrom",isMut: !0,isSigner: !1,docs: ["The source cargo pod account"]}, {name: "cargoType",isMut: !1,isSigner: !1,docs: ["The Cargo Type Account"]}, {name: "cargoStatsDefinition",isMut: !1,isSigner: !1,docs: ["The cargo stats definition account"]}, {name: "tokenFrom",isMut: !0,isSigner: !1,docs: ["The source account of the tokens - owner should be `cargo_pod_from`"]}, {name: "tokenMint",isMut: !0,isSigner: !1,docs: ["The token mint"]}, {name: "gameAccountsAndProfile",accounts: [{name: "gameAndProfileAndFaction",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "profileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "resourceRecipe",isMut: !1,isSigner: !1,docs: ["The crafting recipe for the upkeep resource"]}, {name: "loyaltyPointsAccounts",accounts: [{name: "userPointsAccount",isMut: !0,isSigner: !1,docs: ["The User Points Account"]}, {name: "pointsCategory",isMut: !1,isSigner: !1,docs: ["The Points Category Account"]}, {name: "pointsModifierAccount",isMut: !1,isSigner: !1,docs: ["The Points Modifier Account"]}]}, {name: "progressionConfig",isMut: !1,isSigner: !1,docs: ["The [`ProgressionConfig`] account"]}, {name: "pointsProgram",isMut: !1,isSigner: !1,docs: ["The points program"]}, {name: "cargoProgram",isMut: !1,isSigner: !1,docs: ["The Cargo Program"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["The [Token] program"]}],args: [{name: "input",type: {defined: "DepositStarbaseUpkeepResourceInput"}}]}, {name: "deregisterMineItem",accounts: [{name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "fundsTo",isMut: !0,isSigner: !1,docs: ["Where the closing funds go."]}, {name: "mineItem",isMut: !0,isSigner: !1,docs: ["The [`MineItem`] account"]}],args: [{name: "input",type: {defined: "KeyIndexInput"}}]}, {name: "deregisterProgressionConfig",accounts: [{name: "fundsTo",isMut: !0,isSigner: !1,docs: ["Where the closing funds go."]}, {name: "progressionConfig",isMut: !0,isSigner: !1,docs: ["The [`ProgressionConfig`] account"]}, {name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The Solana System program"]}],args: [{name: "input",type: {defined: "KeyIndexInput"}}]}, {name: "deregisterResource",accounts: [{name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "fundsTo",isMut: !0,isSigner: !1,docs: ["Where the closing funds go."]}, {name: "mineItem",isMut: !0,isSigner: !1,docs: ["The [`MineItem`] account"]}, {name: "resource",isMut: !0,isSigner: !1,docs: ["The [`Resource`] account"]}, {name: "location",isMut: !0,isSigner: !1,docs: ["The Location address"]}],args: [{name: "input",type: {defined: "KeyIndexInput"}}]}, {name: "deregisterStarbase",accounts: [{name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "fundsTo",isMut: !0,isSigner: !1,docs: ["Where the closing funds go."]}, {name: "starbase",isMut: !0,isSigner: !1,docs: ["The [`Starbase`] account"]}],args: [{name: "input",type: {defined: "KeyIndexInput"}}]}, {name: "deregisterSurveyDataUnitTracker",accounts: [{name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "fundsTo",isMut: !0,isSigner: !1,docs: ["Where the closing funds go."]}, {name: "surveyDataUnitTracker",isMut: !0,isSigner: !1,docs: ["The [`SurveyDataUnitTracker`] account"]}],args: [{name: "input",type: {defined: "DeregisterSurveyDataUnitTrackerInput"}}]}, {name: "disbandFleet",accounts: [{name: "gameAccountsAndProfile",accounts: [{name: "gameAndProfileAndFaction",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "profileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder - pays for account rent"]}, {name: "disbandedFleet",isMut: !0,isSigner: !1,docs: ["The [`DisbandedFleet`] account"]}, {name: "fleet",isMut: !0,isSigner: !1,docs: ["The [`Fleet`] account", "The fleet can only be disbanded by the `owner_profile` and not `sub_profile`", "Fleet cannot be disbanded while rented"]}, {name: "fleetShips",isMut: !0,isSigner: !1,docs: ["The [`FleetShips`] account"]}, {name: "cargoHold",isMut: !0,isSigner: !1,docs: ["The fleet `cargo_hold` cargo pod"]}, {name: "fuelTank",isMut: !0,isSigner: !1,docs: ["The fleet `fuel_tank` cargo pod"]}, {name: "ammoBank",isMut: !0,isSigner: !1,docs: ["The fleet `ammo_bank` cargo pod"]}, {name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !1,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "cargoProgram",isMut: !1,isSigner: !1,docs: ["The Cargo Program"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The Solana System program"]}],args: [{name: "input",type: {defined: "DisbandFleetInput"}}]}, {name: "disbandedFleetToEscrow",accounts: [{name: "gameAccountsAndProfile",accounts: [{name: "gameAndProfileAndFaction",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "profileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder - pays for account rent"]}, {name: "disbandedFleet",isMut: !0,isSigner: !1,docs: ["The [`DisbandedFleet`] account"]}, {name: "fleetShips",isMut: !0,isSigner: !1,docs: ["The [`FleetShips`] account"]}, {name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !0,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "ship",isMut: !1,isSigner: !1,docs: ["The [`Ship`] Account"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The Solana System program"]}],args: [{name: "input",type: {defined: "DisbandedFleetToEscrowInput"}}]}, {name: "discoverSector",accounts: [{name: "gameAccountsFleetAndOwner",accounts: [{name: "gameFleetAndOwner",accounts: [{name: "fleetAndOwner",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key on the profile."]}, {name: "owningProfile",isMut: !1,isSigner: !1,docs: ["The profile that owns the fleet."]}, {name: "owningProfileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "fleet",isMut: !1,isSigner: !1,docs: ["The fleet."]}]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the new sector"]}, {name: "sector",isMut: !0,isSigner: !1,docs: ["The [`Sector`] account"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The Solana System program"]}],args: [{name: "input",type: {defined: "DiscoverSectorInput"}}]}, {name: "drainMineItemBank",accounts: [{name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "fundsTo",isMut: !0,isSigner: !1,docs: ["Where the closing funds go."]}, {name: "mineItem",isMut: !0,isSigner: !1,docs: ["The [`MineItem`] account"]}, {name: "tokenFrom",isMut: !0,isSigner: !1,docs: ["The mine item token bank to drain"]}, {name: "tokenTo",isMut: !0,isSigner: !1,docs: ["Where to send tokens from the bank"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["The [Token] program"]}],args: [{name: "input",type: {defined: "KeyIndexInput"}}]}, {name: "drainSurveyDataUnitsBank",accounts: [{name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "fundsTo",isMut: !0,isSigner: !1,docs: ["Where the closing rent refunds go."]}, {name: "surveyDataUnitTracker",isMut: !1,isSigner: !1,docs: ["The [`SurveyDataUnitTracker`] account"]}, {name: "surveyDataUnitTrackerSigner",isMut: !1,isSigner: !1,docs: ["The `SurveyDataUnitTracker` signer"]}, {name: "tokenFrom",isMut: !0,isSigner: !1,docs: ["The SDU token bank to drain"]}, {name: "tokenTo",isMut: !0,isSigner: !1,docs: ["Where to send tokens from the bank"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["The [Token] program"]}],args: [{name: "input",type: {defined: "KeyIndexInput"}}]}, {name: "fleetStateHandler",accounts: [{name: "fleet",isMut: !0,isSigner: !1,docs: ["The fleet."]}],args: []}, {name: "forceDisbandFleet",accounts: [{name: "funder",isMut: !0,isSigner: !0,docs: ["The funder - pays for account rent"]}, {name: "disbandedFleet",isMut: !0,isSigner: !1,docs: ["The new [`DisbandedFleet`] account"]}, {name: "fleet",isMut: !0,isSigner: !1,docs: ["The [`Fleet`] account"]}, {name: "fleetShips",isMut: !0,isSigner: !1,docs: ["The [`FleetShips`] account"]}, {name: "cargoHold",isMut: !0,isSigner: !1,docs: ["The fleet `cargo_hold` cargo pod"]}, {name: "fuelTank",isMut: !0,isSigner: !1,docs: ["The fleet `fuel_tank` cargo pod"]}, {name: "ammoBank",isMut: !0,isSigner: !1,docs: ["The fleet `ammo_bank` cargo pod"]}, {name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !1,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "ship",isMut: !1,isSigner: !1,docs: ["The [`Ship`]", "Must provide at least one ship that is invalid for this instruction"]}, {name: "gameAccounts",accounts: [{name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "cargoProgram",isMut: !1,isSigner: !1,docs: ["The Cargo Program"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The Solana System program"]}],args: [{name: "input",type: {defined: "ForcedDisbandFleetInput"}}]}, {name: "forceDropFleetCargo",accounts: [{name: "fleet",isMut: !0,isSigner: !1,docs: ["The `Fleet` Account"]}, {name: "fleetShips",isMut: !0,isSigner: !1,docs: ["The [`FleetShips`] account"]}, {name: "cargoPod",isMut: !0,isSigner: !1,docs: ["The origin cargo pod"]}, {name: "cargoType",isMut: !1,isSigner: !1,docs: ["The `cargo_type` for the token"]}, {name: "cargoStatsDefinition",isMut: !1,isSigner: !1,docs: ["The cargo stats definition"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}, {name: "tokenFrom",isMut: !0,isSigner: !1,docs: ["The source token account - owned by the `cargo_pod`"]}, {name: "tokenMint",isMut: !0,isSigner: !1,docs: ["The mint of the token account"]}, {name: "cargoProgram",isMut: !1,isSigner: !1,docs: ["The Cargo Program"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["Token Program"]}],args: []}, {name: "idleToLoadingBay",accounts: [{name: "gameAccountsFleetAndOwner",accounts: [{name: "gameFleetAndOwner",accounts: [{name: "fleetAndOwner",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key on the profile."]}, {name: "owningProfile",isMut: !1,isSigner: !1,docs: ["The profile that owns the fleet."]}, {name: "owningProfileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "fleet",isMut: !0,isSigner: !1,docs: ["The fleet."]}]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !0,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}],args: [{name: "keyIndex",type: "u16"}]}, {name: "idleToRespawn",accounts: [{name: "gameAccountsFleetAndOwner",accounts: [{name: "gameFleetAndOwner",accounts: [{name: "fleetAndOwner",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key on the profile."]}, {name: "owningProfile",isMut: !1,isSigner: !1,docs: ["The profile that owns the fleet."]}, {name: "owningProfileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "fleet",isMut: !0,isSigner: !1,docs: ["The fleet."]}]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "atlasTokenFrom",isMut: !0,isSigner: !1,docs: ["Source Token account for ATLAS, owned by the player"]}, {name: "atlasTokenTo",isMut: !0,isSigner: !1,docs: ["Vault Token account for ATLAS"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["The Solana Token Program"]}],args: [{name: "input",type: {defined: "IdleToRespawnInput"}}]}, {name: "initGame",accounts: [{name: "signer",isMut: !0,isSigner: !0,docs: ["The entity calling this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The sector permissions [`Profile`]"]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the new game"]}, {name: "gameId",isMut: !0,isSigner: !0,docs: ["The [`Game`] account"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The system program"]}],args: []}, {name: "initGameState",accounts: [{name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the new `GameState`"]}, {name: "gameState",isMut: !0,isSigner: !1,docs: ["The [`GameState`] account"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The system program"]}],args: [{name: "input",type: {defined: "InitGameStateInput"}}]}, {name: "invalidateRental",accounts: [{name: "subProfileInvalidator",isMut: !1,isSigner: !0,docs: ["The fleet rental invalidator"]}, {name: "fleet",isMut: !0,isSigner: !1,docs: ["The fleet"]}],args: []}, {name: "invalidateShip",accounts: [{name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !0,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "ship",isMut: !0,isSigner: !1,docs: ["The current [`Ship`] account"]}],args: [{name: "keyIndex",type: "u16"}]}, { name: "loadFleetCrew", accounts: [{ name: "fleetAndOwner", accounts: [{ name: "key", isMut: !1, isSigner: !0, docs: ["The key on the profile."] }, { name: "owningProfile", isMut: !1, isSigner: !1, docs: ["The profile that owns the fleet."] }, { name: "owningProfileFaction", isMut: !1, isSigner: !1, docs: ["The faction that the profile belongs to."] }, { name: "fleet", isMut: !0, isSigner: !1, docs: ["The fleet."] }] }, { name: "starbaseAndStarbasePlayer", accounts: [{ name: "starbase", isMut: !1, isSigner: !1, docs: ["The [`Starbase`] account"] }, { name: "starbasePlayer", isMut: !0, isSigner: !1, docs: ["The [`StarbasePlayer`] Account"] }] }, { name: "gameId", isMut: !1, isSigner: !1, docs: ["The [`Game`] account"] }], args: [{ name: "input", type: { defined: "FleetCrewInput" } }] }, { name: "unloadFleetCrew", accounts: [{ name: "fleetAndOwner", accounts: [{ name: "key", isMut: !1, isSigner: !0, docs: ["The key on the profile."] }, { name: "owningProfile", isMut: !1, isSigner: !1, docs: ["The profile that owns the fleet."] }, { name: "owningProfileFaction", isMut: !1, isSigner: !1, docs: ["The faction that the profile belongs to."] }, { name: "fleet", isMut: !0, isSigner: !1, docs: ["The fleet."] }] }, { name: "starbaseAndStarbasePlayer", accounts: [{ name: "starbase", isMut: !1, isSigner: !1, docs: ["The [`Starbase`] account"] }, { name: "starbasePlayer", isMut: !0, isSigner: !1, docs: ["The [`StarbasePlayer`] Account"] }] }, { name: "gameId", isMut: !1, isSigner: !1, docs: ["The [`Game`] account"] }], args: [{ name: "input", type: { defined: "FleetCrewInput" } }] }, {name: "loadingBayToIdle",accounts: [{name: "gameAccountsFleetAndOwner",accounts: [{name: "gameFleetAndOwner",accounts: [{name: "fleetAndOwner",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key on the profile."]}, {name: "owningProfile",isMut: !1,isSigner: !1,docs: ["The profile that owns the fleet."]}, {name: "owningProfileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "fleet",isMut: !0,isSigner: !1,docs: ["The fleet."]}]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !0,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}],args: [{name: "keyIndex",type: "u16"}]}, {name: "mineAsteroidToRespawn",accounts: [{name: "gameAccountsFleetAndOwner",accounts: [{name: "gameFleetAndOwner",accounts: [{name: "fleetAndOwner",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key on the profile."]}, {name: "owningProfile",isMut: !1,isSigner: !1,docs: ["The profile that owns the fleet."]}, {name: "owningProfileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "fleet",isMut: !0,isSigner: !1,docs: ["The fleet."]}]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "resource",isMut: !0,isSigner: !1,docs: ["The [`Resource`] account"]}, {name: "planet",isMut: !0,isSigner: !1,docs: ["The [`Planet`] account"]}, {name: "atlasTokenFrom",isMut: !0,isSigner: !1,docs: ["Source Token account for ATLAS, owned by the player"]}, {name: "atlasTokenTo",isMut: !0,isSigner: !1,docs: ["Vault Token account for ATLAS"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["The Solana Token Program"]}],args: [{name: "input",type: {defined: "MineAsteroidToRespawnInput"}}]}, {name: "mintCertificate",accounts: [{name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !1,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "gameAccountsAndProfile",accounts: [{name: "gameAndProfileAndFaction",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "profileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "cargoMint",isMut: !1,isSigner: !1,docs: ["The mint of the cargo in question"]}, {name: "certificateMint",isMut: !0,isSigner: !1,docs: ["The cargo certificate mint"]}, {name: "certificateTokenTo",isMut: !0,isSigner: !1,docs: ["The token account where certificates are minted to"]}, {name: "cargoTokenFrom",isMut: !0,isSigner: !1,docs: ["The source token account for the cargo - owned by the `cargo_pod`"]}, {name: "cargoTokenTo",isMut: !0,isSigner: !1,docs: ["The destination token account for the cargo - owned by the Starbase"]}, {name: "cargoPod",isMut: !0,isSigner: !1,docs: ["The cargo pod to take from"]}, {name: "cargoType",isMut: !1,isSigner: !1,docs: ["The cargo type account"]}, {name: "cargoStatsDefinition",isMut: !1,isSigner: !1,docs: ["The cargo stats definition account"]}, {name: "cargoProgram",isMut: !1,isSigner: !1,docs: ["The Cargo Program"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["The token program"]}],args: [{name: "keyIndex",type: "u16"}, {name: "amount",type: "u64"}]}, {name: "redeemCertificate",accounts: [{name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !1,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "gameAccountsAndProfile",accounts: [{name: "gameAndProfileAndFaction",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "profileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "cargoMint",isMut: !1,isSigner: !1,docs: ["The mint of the cargo in question"]}, {name: "certificateMint",isMut: !0,isSigner: !1,docs: ["The cargo certificate mint"]}, {name: "certificateOwnerAuthority",isMut: !1,isSigner: !0,docs: ["Owner of the certificates"]}, {name: "certificateTokenFrom",isMut: !0,isSigner: !1,docs: ["The source token account for the cargo certificate - owned by the `certificate_owner_authority`"]}, {name: "cargoTokenFrom",isMut: !0,isSigner: !1,docs: ["The source token account for the cargo - owned by the Starbase"]}, {name: "cargoTokenTo",isMut: !0,isSigner: !1,docs: ["The destination token account for the cargo - owned by the `cargo_pod`"]}, {name: "cargoPod",isMut: !0,isSigner: !1,docs: ["The cargo pod to send to"]}, {name: "cargoType",isMut: !1,isSigner: !1,docs: ["The cargo type account"]}, {name: "cargoStatsDefinition",isMut: !1,isSigner: !1,docs: ["The cargo stats definition account"]}, {name: "cargoProgram",isMut: !1,isSigner: !1,docs: ["The Cargo Program"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["The token program"]}],args: [{name: "keyIndex",type: "u16"}, {name: "amount",type: "u64"}]}, {name: "registerMineItem",accounts: [{name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the new mine item"]}, {name: "mineItem",isMut: !0,isSigner: !1,docs: ["The [`MineItem`] account"]}, {name: "mint",isMut: !0,isSigner: !1,docs: ["The mint address representing the mine item"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The Solana System program"]}],args: [{name: "input",type: {defined: "RegisterMineItemInput"}}]}, {name: "registerPlanet",accounts: [{name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the new game"]}, {name: "planet",isMut: !0,isSigner: !0,docs: ["The [`Planet`] account"]}, {name: "sector",isMut: !0,isSigner: !1,docs: ["The [`Sector`] account"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The Solana System program"]}],args: [{name: "input",type: {defined: "RegisterPlanetInput"}}]}, {name: "registerProgressionConfig",accounts: [{name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the new star base"]}, {name: "progressionConfig",isMut: !0,isSigner: !1,docs: ["The [`ProgressionConfig`] account"]}, {name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The Solana System program"]}],args: [{name: "input",type: {defined: "RegisterProgressionConfigInput"}}]}, {name: "registerResource",accounts: [{name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the new resource"]}, {name: "resource",isMut: !0,isSigner: !1,docs: ["The [`Resource`] account"]}, {name: "location",isMut: !0,isSigner: !1,docs: ["The Location address"]}, {name: "mineItem",isMut: !0,isSigner: !1,docs: ["The [`MineItem`] account"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The Solana System program"]}],args: [{name: "input",type: {defined: "RegisterResourceInput"}}]}, {name: "registerSagePlayerProfile",accounts: [{name: "profile",isMut: !1,isSigner: !1,docs: ["The player permissions [`Profile`]"]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the new profile"]}, {name: "sagePlayerProfile",isMut: !0,isSigner: !1,docs: ["The `SagePlayerProfile` account"]}, {name: "gameAccounts",accounts: [{name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The Solana System program"]}],args: []}, {name: "registerSagePointModifier",accounts: [{name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder - pays for account rent"]}, {name: "pointsCategory",isMut: !1,isSigner: !1,docs: ["The `PointCategory`"]}, {name: "pointsModifier",isMut: !0,isSigner: !1,docs: ["The `PointsModifier` account to be created in Points CPI"]}, {name: "pointsProgram",isMut: !1,isSigner: !1,docs: ["The points program"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The Solana System program"]}],args: [{name: "input",type: {defined: "RegisterSagePointsModifierInput"}}]}, {name: "registerSector",accounts: [{name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the new sector"]}, {name: "discoverer",isMut: !1,isSigner: !1,docs: ["The discoverer of this sector"]}, {name: "sector",isMut: !0,isSigner: !1,docs: ["The [`Sector`] account"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The Solana System program"]}],args: [{name: "coordinates",type: {array: ["i64", 2]}}, {name: "name",type: {array: ["u8", 64]}}, {name: "keyIndex",type: "u16"}]}, {name: "registerShip",accounts: [{name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the new game"]}, {name: "ship",isMut: !0,isSigner: !0,docs: ["The [`Ship`] account"]}, {name: "mint",isMut: !1,isSigner: !1,docs: ["The mint address representing the [`Ship`]"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The Solana System program"]}],args: [{name: "input",type: {defined: "RegisterShipInput"}}]}, {name: "registerStar",accounts: [{name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder - pays for account rent"]}, {name: "star",isMut: !0,isSigner: !0,docs: ["The [`Star`] account"]}, {name: "sector",isMut: !0,isSigner: !1,docs: ["The [`Sector`] account"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The Solana System program"]}],args: [{name: "input",type: {defined: "RegisterStarInput"}}]}, {name: "registerStarbase",accounts: [{name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the new star base"]}, {name: "starbase",isMut: !0,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "sector",isMut: !1,isSigner: !1,docs: ["The [`Sector`] account"]}, {name: "gameStateAndProfile",accounts: [{name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The Solana System program"]}],args: [{name: "input",type: {defined: "RegisterStarbaseInputUnpacked"}}]}, {name: "registerStarbasePlayer",accounts: [{name: "funder",isMut: !0,isSigner: !0,docs: ["The funder -- pays account rent"]}, {name: "gameAccounts",accounts: [{name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "sagePlayerProfile",isMut: !1,isSigner: !1,docs: ["The [`SagePlayerProfile`] account"]}, {name: "profileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the player belongs to."]}, {name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !0,isSigner: !1,docs: ["The [`StarbasePlayer`] account to initialize"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The system program"]}],args: []}, {name: "registerSurveyDataUnitTracker",accounts: [{name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the new `SurveyDataUnitTracker`"]}, {name: "surveyDataUnitTracker",isMut: !0,isSigner: !0,docs: ["The [`SurveyDataUnitTracker`] account"]}, {name: "sduMint",isMut: !1,isSigner: !1,docs: ["The Survey Data Unit Mint"]}, {name: "resourceMint",isMut: !1,isSigner: !1,docs: ["The mint of the resource spent when scanning for SDUs"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The Solana System program"]}],args: [{name: "input",type: {defined: "RegisterSurveyDataUnitTrackerInput"}}]}, {name: "removeCargoPod",accounts: [{name: "fundsTo",isMut: !0,isSigner: !1,docs: ["The funds_to - receives rent refund"]}, {name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !1,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "cargoPod",isMut: !0,isSigner: !1,docs: ["The cargo pod (should be empty)"]}, {name: "gameAccountsAndProfile",accounts: [{name: "gameAndProfileAndFaction",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "profileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "cargoProgram",isMut: !1,isSigner: !1,docs: ["The Cargo Program"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The system program"]}],args: [{name: "input",type: {defined: "StarbaseRemoveCargoPodInput"}}]}, {name: "removeConnection",accounts: [{name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "fundsTo",isMut: !0,isSigner: !1,docs: ["Where the rent refund funds from the connections go to."]}, {name: "sector1",isMut: !0,isSigner: !1,docs: ["The first sector to remove from"]}, {name: "sector2",isMut: !0,isSigner: !1,docs: ["The second sector to remove from"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The system program"]}],args: [{name: "sector1Index",type: "u16"}, {name: "sector2Index",type: "u16"}, {name: "keyIndex",type: "u16"}]}, {name: "removeInvalidShipEscrow",accounts: [{name: "gameAccountsAndProfile",accounts: [{name: "gameAndProfileAndFaction",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "profileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "fundsTo",isMut: !0,isSigner: !1,docs: ["The funds_to - receives rent refund"]}, {name: "sagePlayerProfile",isMut: !0,isSigner: !1,docs: ["The [`SagePlayerProfile`] account"]}, {name: "destinationTokenAccount",isMut: !0,isSigner: !1,docs: ["The Destination Token Account"]}, {name: "ship",isMut: !1,isSigner: !1,docs: ["The [`Ship`] Account"]}, {name: "shipEscrowTokenAccount",isMut: !0,isSigner: !1,docs: ["The Escrow Token Account"]}, {name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !0,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["The Token Program"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The Solana System program"]}],args: [{name: "input",type: {defined: "RemoveShipEscrowInput"}}]}, {name: "removeShipEscrow",accounts: [{name: "gameAccountsAndProfile",accounts: [{name: "gameAndProfileAndFaction",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "profileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "fundsTo",isMut: !0,isSigner: !1,docs: ["The funds_to - receives rent refund"]}, {name: "sagePlayerProfile",isMut: !0,isSigner: !1,docs: ["The [`SagePlayerProfile`] account"]}, {name: "destinationTokenAccount",isMut: !0,isSigner: !1,docs: ["The Destination Token Account"]}, {name: "ship",isMut: !1,isSigner: !1,docs: ["The [`Ship`] Account"]}, {name: "shipEscrowTokenAccount",isMut: !0,isSigner: !1,docs: ["The Escrow Token Account"]}, {name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !0,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["The Token Program"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The Solana System program"]}],args: [{name: "input",type: {defined: "RemoveShipEscrowInput"}}]}, {name: "respawnToLoadingBay",accounts: [{name: "gameAccountsFleetAndOwner",accounts: [{name: "gameFleetAndOwner",accounts: [{name: "fleetAndOwner",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key on the profile."]}, {name: "owningProfile",isMut: !1,isSigner: !1,docs: ["The profile that owns the fleet."]}, {name: "owningProfileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "fleet",isMut: !0,isSigner: !1,docs: ["The fleet."]}]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !0,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "cargoHold",isMut: !1,isSigner: !1,docs: ["The fleet `cargo_hold` cargo pod"]}, {name: "fuelTank",isMut: !1,isSigner: !1,docs: ["The fleet `fuel_tank` cargo pod"]}, {name: "ammoBank",isMut: !1,isSigner: !1,docs: ["The fleet `ammo_bank` cargo pod"]}],args: [{name: "input",type: {defined: "RespawnToLoadingBayInput"}}]}, {name: "scanForSurveyDataUnits",accounts: [{name: "gameAccountsFleetAndOwner",accounts: [{name: "gameFleetAndOwner",accounts: [{name: "fleetAndOwner",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key on the profile."]}, {name: "owningProfile",isMut: !1,isSigner: !1,docs: ["The profile that owns the fleet."]}, {name: "owningProfileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "fleet",isMut: !0,isSigner: !1,docs: ["The fleet."]}]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "surveyDataUnitTracker",isMut: !1,isSigner: !1,docs: ["The [`SurveyDataUnitTracker`] account"]}, {name: "surveyDataUnitTrackerSigner",isMut: !1,isSigner: !1,docs: ["The `SurveyDataUnitTracker` signer"]}, {name: "cargoHold",isMut: !0,isSigner: !1,docs: ["The general cargo hold cargo pod for the fleet"]}, {name: "sector",isMut: !0,isSigner: !1,docs: ["The [`Sector`] account"]}, {name: "sduTokenFrom",isMut: !0,isSigner: !1,docs: ["Source token account for the SDU, owned by `survey_data_unit_tracker_signer`"]}, {name: "sduTokenTo",isMut: !0,isSigner: !1,docs: ["Destination token account for the SDU, owned by cargo_hold"]}, {name: "resourceTokenFrom",isMut: !0,isSigner: !1,docs: ["Token account for `resource_mint`, owned by fleet"]}, {name: "resourceMint",isMut: !0,isSigner: !1,docs: ["The resource that is spent when scanning for Survey Data Units"]}, {name: "sduCargoType",isMut: !1,isSigner: !1,docs: ["The cargo type of the SDU"]}, {name: "resourceCargoType",isMut: !1,isSigner: !1,docs: ["The cargo type of `resource_mint`"]}, {name: "cargoStatsDefinition",isMut: !1,isSigner: !1,docs: ["The cargo stats definition"]}, {name: "dataRunningXpAccounts",accounts: [{name: "userPointsAccount",isMut: !0,isSigner: !1,docs: ["The User Points Account"]}, {name: "pointsCategory",isMut: !1,isSigner: !1,docs: ["The Points Category Account"]}, {name: "pointsModifierAccount",isMut: !1,isSigner: !1,docs: ["The Points Modifier Account"]}]}, {name: "councilRankXpAccounts",accounts: [{name: "userPointsAccount",isMut: !0,isSigner: !1,docs: ["The User Points Account"]}, {name: "pointsCategory",isMut: !1,isSigner: !1,docs: ["The Points Category Account"]}, {name: "pointsModifierAccount",isMut: !1,isSigner: !1,docs: ["The Points Modifier Account"]}]}, {name: "progressionConfig",isMut: !1,isSigner: !1,docs: ["The [`ProgressionConfig`] account"]}, {name: "pointsProgram",isMut: !1,isSigner: !1,docs: ["The points program"]}, {name: "cargoProgram",isMut: !1,isSigner: !1,docs: ["The cargo program"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["The token program"]}, {name: "instructionsSysvar",isMut: !1,isSigner: !1,docs: ["Solana Instructions Sysvar"]}, {name: "recentSlothashes",isMut: !1,isSigner: !1,docs: ["Solana recent slothashes"]}],args: [{name: "input",type: {defined: "ScanForSurveyDataUnitsInput"}}]}, {name: "setNextShip",accounts: [{name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "ship",isMut: !0,isSigner: !1,docs: ["The current [`Ship`] account"]}, {name: "nextShip",isMut: !0,isSigner: !1,docs: ["The next [`Ship`] account"]}],args: [{name: "keyIndex",type: "u16"}]}, {name: "startCraftingProcess",accounts: [{name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !1,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "craftingInstance",isMut: !1,isSigner: !1,docs: ["The [`CraftingInstance`] account"]}, {name: "craftingProcess",isMut: !0,isSigner: !1,docs: ["The [`CraftingProcess`] account"]}, {name: "craftingRecipe",isMut: !0,isSigner: !1,docs: ["The crafting [`Recipe`]"]}, {name: "craftingFacility",isMut: !1,isSigner: !1,docs: ["The `CraftingFacility` account"]}, {name: "gameAccountsAndProfile",accounts: [{name: "gameAndProfileAndFaction",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "profileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "craftingProgram",isMut: !1,isSigner: !1,docs: ["The Crafting Program"]}],args: [{name: "input",type: {defined: "KeyIndexInput"}}]}, {name: "startMiningAsteroid",accounts: [{name: "gameAccountsFleetAndOwner",accounts: [{name: "gameFleetAndOwner",accounts: [{name: "fleetAndOwner",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key on the profile."]}, {name: "owningProfile",isMut: !1,isSigner: !1,docs: ["The profile that owns the fleet."]}, {name: "owningProfileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "fleet",isMut: !0,isSigner: !1,docs: ["The fleet."]}]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "fleetFuelTokenAccount",isMut: !1,isSigner: !1,docs: ["The fleet fuel token account - owned by the `fuel_tank`"]}, {name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !1,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "mineItem",isMut: !1,isSigner: !1,docs: ["The [`MineItem`] account"]}, {name: "resource",isMut: !0,isSigner: !1,docs: ["The [`Resource`] account"]}, {name: "planet",isMut: !0,isSigner: !1,docs: ["The [`Planet`] account"]}],args: [{name: "input",type: {defined: "KeyIndexInput"}}]}, {name: "startStarbaseUpgrade",accounts: [{name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the new crafting process"]}, {name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !0,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !1,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "upgradeFacility",isMut: !0,isSigner: !1,docs: ["The `CraftingFacility` account for starbase upgrades"]}, {name: "upgradeRecipe",isMut: !1,isSigner: !1,docs: ["The crafting recipe for the starbase upgrade"]}, {name: "gameAccountsAndProfile",accounts: [{name: "gameAndProfileAndFaction",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "profileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["Solana System program"]}],args: [{name: "input",type: {defined: "KeyIndexInput"}}]}, {name: "startSubwarp",accounts: [{name: "gameAccountsFleetAndOwner",accounts: [{name: "gameFleetAndOwner",accounts: [{name: "fleetAndOwner",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key on the profile."]}, {name: "owningProfile",isMut: !1,isSigner: !1,docs: ["The profile that owns the fleet."]}, {name: "owningProfileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "fleet",isMut: !0,isSigner: !1,docs: ["The fleet."]}]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}],args: [{name: "input",type: {defined: "StartSubwarpInput"}}]}, {name: "stopCraftingProcess",accounts: [{name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !1,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "craftingInstance",isMut: !1,isSigner: !1,docs: ["The [`CraftingInstance`] account"]}, {name: "craftingProcess",isMut: !0,isSigner: !1,docs: ["The [`CraftingProcess`] account"]}, {name: "craftingRecipe",isMut: !0,isSigner: !1,docs: ["The crafting [`Recipe`]"]}, {name: "craftingFacility",isMut: !1,isSigner: !1,docs: ["The `CraftingFacility` account"]}, {name: "gameAccountsAndProfile",accounts: [{name: "gameAndProfileAndFaction",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "profileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "craftingProgram",isMut: !1,isSigner: !1,docs: ["The Crafting Program"]}],args: [{name: "input",type: {defined: "KeyIndexInput"}}]}, {name: "stopMiningAsteroid",accounts: [{name: "gameAccountsFleetAndOwner",accounts: [{name: "gameFleetAndOwner",accounts: [{name: "fleetAndOwner",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key on the profile."]}, {name: "owningProfile",isMut: !1,isSigner: !1,docs: ["The profile that owns the fleet."]}, {name: "owningProfileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "fleet",isMut: !0,isSigner: !1,docs: ["The fleet."]}]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "mineItem",isMut: !1,isSigner: !1,docs: ["The [`MineItem`] account"]}, {name: "resource",isMut: !0,isSigner: !1,docs: ["The [`Resource`] account"]}, {name: "planet",isMut: !0,isSigner: !1,docs: ["The [`Planet`] account"]}, {name: "fuelTank",isMut: !0,isSigner: !1,docs: ["The fuel tank cargo pod"]}, {name: "cargoType",isMut: !1,isSigner: !1,docs: ["The cargo type account for fuel"]}, {name: "cargoStatsDefinition",isMut: !1,isSigner: !1,docs: ["The cargo stats definition account"]}, {name: "tokenFrom",isMut: !0,isSigner: !1,docs: ["The source token account - owned by the `fuel_tank`"]}, {name: "tokenMint",isMut: !0,isSigner: !1,docs: ["The mint of the token account"]}, {name: "pilotXpAccounts",accounts: [{name: "userPointsAccount",isMut: !0,isSigner: !1,docs: ["The User Points Account"]}, {name: "pointsCategory",isMut: !1,isSigner: !1,docs: ["The Points Category Account"]}, {name: "pointsModifierAccount",isMut: !1,isSigner: !1,docs: ["The Points Modifier Account"]}]}, {name: "miningXpAccounts",accounts: [{name: "userPointsAccount",isMut: !0,isSigner: !1,docs: ["The User Points Account"]}, {name: "pointsCategory",isMut: !1,isSigner: !1,docs: ["The Points Category Account"]}, {name: "pointsModifierAccount",isMut: !1,isSigner: !1,docs: ["The Points Modifier Account"]}]}, {name: "councilRankXpAccounts",accounts: [{name: "userPointsAccount",isMut: !0,isSigner: !1,docs: ["The User Points Account"]}, {name: "pointsCategory",isMut: !1,isSigner: !1,docs: ["The Points Category Account"]}, {name: "pointsModifierAccount",isMut: !1,isSigner: !1,docs: ["The Points Modifier Account"]}]}, {name: "progressionConfig",isMut: !1,isSigner: !1,docs: ["The [`ProgressionConfig`] account"]}, {name: "pointsProgram",isMut: !1,isSigner: !1,docs: ["The points program"]}, {name: "cargoProgram",isMut: !1,isSigner: !1,docs: ["The Cargo Program"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["Token Program"]}],args: [{name: "input",type: {defined: "StopMiningAsteroidInput"}}]}, {name: "stopSubwarp",accounts: [{name: "gameAccountsFleetAndOwner",accounts: [{name: "gameFleetAndOwner",accounts: [{name: "fleetAndOwner",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key on the profile."]}, {name: "owningProfile",isMut: !1,isSigner: !1,docs: ["The profile that owns the fleet."]}, {name: "owningProfileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "fleet",isMut: !0,isSigner: !1,docs: ["The fleet."]}]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}],args: [{name: "input",type: {defined: "StopSubwarpInput"}}]}, {name: "submitStarbaseUpgradeResource",accounts: [{name: "fundsTo",isMut: !0,isSigner: !1,docs: ["The funds_to - receives rent refund"]}, {name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !0,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !1,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "resourceCraftingInstance",isMut: !1,isSigner: !1,docs: ["The [`CraftingInstance`] account"]}, {name: "resourceCraftingProcess",isMut: !0,isSigner: !1,docs: ["The crafting process account"]}, {name: "resourceCraftingFacility",isMut: !1,isSigner: !1,docs: ["The crafting facility account"]}, {name: "upgradeProcessRecipe",isMut: !1,isSigner: !1,docs: ["The crafting recipe for the submission of resources used in the upgrade process"]}, {name: "starbaseUpgradeRecipe",isMut: !1,isSigner: !1,docs: ["The crafting recipe for the starbase upgrade"]}, {name: "resourceRecipe",isMut: !1,isSigner: !1,docs: ["The crafting recipe for the resource being submitted", "i.e. the recipe for crafting the resource whose mint would be `token_mint`"]}, {name: "cargoPodTo",isMut: !0,isSigner: !1,docs: ["The destination cargo pod account"]}, {name: "cargoType",isMut: !1,isSigner: !1,docs: ["The Cargo Type Account"]}, {name: "cargoStatsDefinition",isMut: !1,isSigner: !1,docs: ["The cargo stats definition account"]}, {name: "tokenFrom",isMut: !0,isSigner: !1,docs: ["The source account of the tokens - owner should be `crafting_process`"]}, {name: "tokenTo",isMut: !0,isSigner: !1,docs: ["The destination account of the tokens - owner should be `cargo_pod_to` (receives any refunds)"]}, {name: "tokenMint",isMut: !0,isSigner: !1,docs: ["The token mint"]}, {name: "gameAccountsAndProfile",accounts: [{name: "gameAndProfileAndFaction",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "profileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "loyaltyPointsAccounts",accounts: [{name: "userPointsAccount",isMut: !0,isSigner: !1,docs: ["The User Points Account"]}, {name: "pointsCategory",isMut: !1,isSigner: !1,docs: ["The Points Category Account"]}, {name: "pointsModifierAccount",isMut: !1,isSigner: !1,docs: ["The Points Modifier Account"]}]}, {name: "progressionConfig",isMut: !1,isSigner: !1,docs: ["The [`ProgressionConfig`] account"]}, {name: "pointsProgram",isMut: !1,isSigner: !1,docs: ["The points program"]}, {name: "craftingProgram",isMut: !1,isSigner: !1,docs: ["The Crafting Program"]}, {name: "cargoProgram",isMut: !1,isSigner: !1,docs: ["The Cargo Program"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["The [Token] program"]}],args: [{name: "input",type: {defined: "SubmitStarbaseUpgradeResourceInput"}}]}, {name: "syncStarbasePlayer",accounts: [{name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !0,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "gameAccounts",accounts: [{name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}],args: []}, {name: "syncStarbaseUpgradeIngredients",accounts: [{name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the new crafting process"]}, {name: "starbase",isMut: !0,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "upgradeRecipe",isMut: !1,isSigner: !1,docs: ["The crafting recipe for the starbase upgrade"]}, {name: "gameAccountsAndProfile",accounts: [{name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["Solana System program"]}],args: [{name: "input",type: {defined: "KeyIndexInput"}}]}, {name: "transferCargoAtStarbase",accounts: [{name: "fundsTo",isMut: !0,isSigner: !1,docs: ["The funds_to - receives rent refund"]}, {name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !1,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "cargoPodFrom",isMut: !0,isSigner: !1,docs: ["The origin cargo pod"]}, {name: "cargoPodTo",isMut: !0,isSigner: !1,docs: ["The destination cargo pod"]}, {name: "cargoType",isMut: !1,isSigner: !1,docs: ["The cargo type account"]}, {name: "cargoStatsDefinition",isMut: !1,isSigner: !1,docs: ["The cargo stats definition account"]}, {name: "gameAccountsAndProfile",accounts: [{name: "gameAndProfileAndFaction",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "profileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "tokenFrom",isMut: !0,isSigner: !1,docs: ["The source token account - owned by the `cargo_pod_from`"]}, {name: "tokenTo",isMut: !0,isSigner: !1,docs: ["The destination token account - owned by the `cargo_pod_to`"]}, {name: "tokenMint",isMut: !0,isSigner: !1,docs: ["The mint of the token accounts"]}, {name: "cargoProgram",isMut: !1,isSigner: !1,docs: ["The Cargo Program"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["Token Program"]}],args: [{name: "input",type: {defined: "StarbaseTransferCargoInput"}}]}, {name: "transferCargoWithinFleet",accounts: [{name: "gameAccountsFleetAndOwner",accounts: [{name: "gameFleetAndOwner",accounts: [{name: "fleetAndOwner",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key on the profile."]}, {name: "owningProfile",isMut: !1,isSigner: !1,docs: ["The profile that owns the fleet."]}, {name: "owningProfileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "fleet",isMut: !0,isSigner: !1,docs: ["The fleet."]}]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "cargoPodFrom",isMut: !0,isSigner: !1,docs: ["The origin cargo pod"]}, {name: "cargoPodTo",isMut: !0,isSigner: !1,docs: ["The destination cargo pod"]}, {name: "cargoType",isMut: !1,isSigner: !1,docs: ["The cargo type account"]}, {name: "cargoStatsDefinition",isMut: !1,isSigner: !1,docs: ["The cargo stats definition account"]}, {name: "tokenFrom",isMut: !0,isSigner: !1,docs: ["The source token account - owned by the `cargo_pod_from`"]}, {name: "tokenTo",isMut: !0,isSigner: !1,docs: ["The destination token account - owned by the `cargo_pod_to`"]}, {name: "tokenMint",isMut: !0,isSigner: !1,docs: ["The mint of the token accounts"]}, {name: "fundsTo",isMut: !0,isSigner: !1,docs: ["The funds_to - receives rent refund"]}, {name: "cargoProgram",isMut: !1,isSigner: !1,docs: ["The Cargo Program"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["Token Program"]}],args: [{name: "input",type: {defined: "TransferCargoWithinFleetInput"}}]}, {name: "updateGame",accounts: [{name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !0,isSigner: !1,docs: ["The [`Game`] account"]}]}],args: [{name: "input",type: {defined: "UpdateGameInput"}}]}, {name: "updateGameState",accounts: [{name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !0,isSigner: !1,docs: ["The [`GameState`] account"]}],args: [{name: "input",type: {defined: "UpdateGameStateInput"}}]}, {name: "updateMineItem",accounts: [{name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "mineItem",isMut: !0,isSigner: !1,docs: ["The [`MineItem`] account"]}],args: [{name: "input",type: {defined: "UpdateMineItemInput"}}]}, {name: "updatePlanet",accounts: [{name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "planet",isMut: !0,isSigner: !1,docs: ["The [`Planet`] account"]}],args: [{name: "input",type: {defined: "UpdatePlanetInput"}}]}, {name: "updateProgressionConfig",accounts: [{name: "progressionConfig",isMut: !0,isSigner: !1,docs: ["The [`ProgressionConfig`] account"]}, {name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The Solana System program"]}],args: [{name: "input",type: {defined: "UpdateProgressionConfigInput"}}]}, {name: "updateResource",accounts: [{name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "mineItem",isMut: !1,isSigner: !1,docs: ["The [`MineItem`] account"]}, {name: "resource",isMut: !0,isSigner: !1,docs: ["The [`Resource`] account"]}],args: [{name: "input",type: {defined: "UpdateResourceInput"}}]}, {name: "updateShip",accounts: [{name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "ship",isMut: !0,isSigner: !1,docs: ["The [`Ship`] account"]}],args: [{name: "input",type: {defined: "UpdateShipInput"}}]}, {name: "updateShipEscrow",accounts: [{name: "oldShip",isMut: !1,isSigner: !1,docs: ["The old [`Ship`] Account"]}, {name: "next",isMut: !1,isSigner: !1,docs: ["The address indicated as `next` in the `old_ship` account"]}, {name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !0,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "gameAccounts",accounts: [{name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}],args: [{name: "input",type: {defined: "UpdateShipEscrowInput"}}]}, {name: "updateShipInFleet",accounts: [{name: "fleet",isMut: !0,isSigner: !1,docs: ["The [`Fleet`] account"]}, {name: "fleetShips",isMut: !0,isSigner: !1,docs: ["The [`FleetShips`] account"]}, {name: "oldShip",isMut: !1,isSigner: !1,docs: ["The old [`Ship`] Account"]}, {name: "next",isMut: !1,isSigner: !1,docs: ["The address indicated as `next` in the `old_ship` account"]}, {name: "gameAccounts",accounts: [{name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}],args: [{name: "input",type: {defined: "UpdateShipFleetInput"}}]}, {name: "updateStar",accounts: [{name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "star",isMut: !0,isSigner: !1,docs: ["The [`Star`] account"]}],args: [{name: "input",type: {defined: "UpdateStarInput"}}]}, {name: "updateStarbase",accounts: [{name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the new game"]}, {name: "starbase",isMut: !0,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The Solana System program"]}],args: [{name: "input",type: {defined: "UpdateStarbaseInput"}}]}, {name: "updateSurveyDataUnitTracker",accounts: [{name: "gameAndProfile",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "surveyDataUnitTracker",isMut: !0,isSigner: !1,docs: ["The [`SurveyDataUnitTracker`] account"]}],args: [{name: "input",type: {defined: "UpdateSurveyDataUnitTrackerInput"}}]}, {name: "warpLane",accounts: [{name: "gameAccountsFleetAndOwner",accounts: [{name: "gameFleetAndOwner",accounts: [{name: "fleetAndOwner",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key on the profile."]}, {name: "owningProfile",isMut: !1,isSigner: !1,docs: ["The profile that owns the fleet."]}, {name: "owningProfileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "fleet",isMut: !0,isSigner: !1,docs: ["The fleet."]}]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "fromStarbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "toStarbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "fromSector",isMut: !1,isSigner: !1,docs: ["The Sector account representing the fleet`s  current sector"]}, {name: "toSector",isMut: !1,isSigner: !1,docs: ["The Sector account that `Fleet` will move to"]}, {name: "fuelTank",isMut: !0,isSigner: !1,docs: ["The fuel tank cargo pod"]}, {name: "cargoType",isMut: !1,isSigner: !1,docs: ["The `Cargo Type` Account"]}, {name: "statsDefinition",isMut: !1,isSigner: !1,docs: ["The `CargoStatsDefinition` for the cargo type"]}, {name: "fuelTokenFrom",isMut: !0,isSigner: !1,docs: ["The fuel source token account - owned by the `fuel_tank`"]}, {name: "fuelMint",isMut: !0,isSigner: !1,docs: ["Token Mint - The fuel mint"]}, {name: "feeTokenFrom",isMut: !0,isSigner: !1,docs: ["The fee source token account"]}, {name: "feeTokenTo",isMut: !0,isSigner: !1,docs: ["The fee destination token account"]}, {name: "feeMint",isMut: !0,isSigner: !1,docs: ["Fee Token Mint"]}, {name: "cargoProgram",isMut: !1,isSigner: !1,docs: ["The Cargo Program"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["Token Program"]}],args: [{name: "input",type: {defined: "WarpLaneInput"}}]}, {name: "warpToCoordinate",accounts: [{name: "gameAccountsFleetAndOwner",accounts: [{name: "gameFleetAndOwner",accounts: [{name: "fleetAndOwner",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key on the profile."]}, {name: "owningProfile",isMut: !1,isSigner: !1,docs: ["The profile that owns the fleet."]}, {name: "owningProfileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "fleet",isMut: !0,isSigner: !1,docs: ["The fleet."]}]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "fuelTank",isMut: !0,isSigner: !1,docs: ["The fuel tank cargo pod"]}, {name: "cargoType",isMut: !1,isSigner: !1,docs: ["The cargo type account for fuel"]}, {name: "statsDefinition",isMut: !1,isSigner: !1,docs: ["The cargo stats definition account"]}, {name: "tokenFrom",isMut: !0,isSigner: !1,docs: ["The source token account - owned by the `fuel_tank`"]}, {name: "tokenMint",isMut: !0,isSigner: !1,docs: ["Token Mint - The fuel mint"]}, {name: "cargoProgram",isMut: !1,isSigner: !1,docs: ["The Cargo Program"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["Token Program"]}],args: [{name: "input",type: {defined: "WarpToCoordinateInput"}}]}, {name: "withdrawCargoFromFleet",accounts: [{name: "gameAccountsFleetAndOwner",accounts: [{name: "gameFleetAndOwner",accounts: [{name: "fleetAndOwner",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key on the profile."]}, {name: "owningProfile",isMut: !1,isSigner: !1,docs: ["The profile that owns the fleet."]}, {name: "owningProfileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "fleet",isMut: !0,isSigner: !1,docs: ["The fleet."]}]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !1,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "cargoPodFrom",isMut: !0,isSigner: !1,docs: ["The origin cargo pod, owned by the fleet"]}, {name: "cargoPodTo",isMut: !0,isSigner: !1,docs: ["The destination cargo pod, owned by the Starbase player"]}, {name: "cargoType",isMut: !1,isSigner: !1,docs: ["The cargo type account"]}, {name: "cargoStatsDefinition",isMut: !1,isSigner: !1,docs: ["The cargo stats definition account"]}, {name: "tokenFrom",isMut: !0,isSigner: !1,docs: ["The source token account - owned by the `cargo_pod_from`"]}, {name: "tokenTo",isMut: !0,isSigner: !1,docs: ["The destination token account - owned by the `cargo_pod_to`"]}, {name: "tokenMint",isMut: !0,isSigner: !1,docs: ["The mint of the token accounts"]}, {name: "fundsTo",isMut: !0,isSigner: !1,docs: ["The funds_to - receives rent refund"]}, {name: "cargoProgram",isMut: !1,isSigner: !1,docs: ["The Cargo Program"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["Token Program"]}],args: [{name: "input",type: {defined: "WithdrawCargoFromFleetInput"}}]}, {name: "withdrawCargoFromGame",accounts: [{name: "fundsTo",isMut: !0,isSigner: !1,docs: ["The funds_to - receives rent refund"]}, {name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !1,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "gameAccountsAndProfile",accounts: [{name: "gameAndProfileAndFaction",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "profileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "cargoPod",isMut: !0,isSigner: !1,docs: ["The new cargo pod"]}, {name: "cargoType",isMut: !1,isSigner: !1,docs: ["The cargo type account"]}, {name: "cargoStatsDefinition",isMut: !1,isSigner: !1,docs: ["The cargo stats definition account"]}, {name: "tokenFrom",isMut: !0,isSigner: !1,docs: ["The source token account - owned by the `cargo_pod`"]}, {name: "tokenTo",isMut: !0,isSigner: !1,docs: ["The destination token account - owned by the `key`"]}, {name: "tokenMint",isMut: !0,isSigner: !1,docs: ["The mint of the token accounts"]}, {name: "cargoProgram",isMut: !1,isSigner: !1,docs: ["The Cargo Program"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["Token Program"]}],args: [{name: "input",type: {defined: "CargoToGameInput"}}]}, {name: "withdrawCraftingIngredient",accounts: [{name: "starbaseAndStarbasePlayer",accounts: [{name: "starbase",isMut: !1,isSigner: !1,docs: ["The [`Starbase`] account"]}, {name: "starbasePlayer",isMut: !1,isSigner: !1,docs: ["The [`StarbasePlayer`] Account"]}]}, {name: "craftingInstance",isMut: !1,isSigner: !1,docs: ["The [`CraftingInstance`] account"]}, {name: "craftingFacility",isMut: !1,isSigner: !1,docs: ["The [`CraftingFacility`](crafting::CraftingFacility) account"]}, {name: "craftingProcess",isMut: !0,isSigner: !1,docs: ["The crafting process account"]}, {name: "cargoPodTo",isMut: !0,isSigner: !1,docs: ["The destination cargo pod account"]}, {name: "craftingRecipe",isMut: !1,isSigner: !1,docs: ["The crafting recipe"]}, {name: "cargoType",isMut: !1,isSigner: !1,docs: ["The Cargo Type Account"]}, {name: "cargoStatsDefinition",isMut: !1,isSigner: !1,docs: ["The cargo stats definition account"]}, {name: "gameAccountsAndProfile",accounts: [{name: "gameAndProfileAndFaction",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] account"]}, {name: "profileFaction",isMut: !1,isSigner: !1,docs: ["The faction that the profile belongs to."]}, {name: "gameId",isMut: !1,isSigner: !1,docs: ["The [`Game`] account"]}]}, {name: "gameState",isMut: !1,isSigner: !1,docs: ["The [`GameState`] account"]}]}, {name: "tokenFrom",isMut: !0,isSigner: !1,docs: ["The source account of the tokens - owner should be `crafting_process`"]}, {name: "tokenTo",isMut: !0,isSigner: !1,docs: ["The destination account of the tokens - owner should be `cargo_pod_to`"]}, {name: "tokenMint",isMut: !0,isSigner: !1,docs: ["The token mint"]}, {name: "craftingProgram",isMut: !1,isSigner: !1,docs: ["The Crafting Program"]}, {name: "cargoProgram",isMut: !1,isSigner: !1,docs: ["The Cargo Program"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["The [Token] program"]}],args: [{name: "input",type: {defined: "StarbaseWithdrawCraftingIngredientInput"}}]}],accounts: [{name: "craftingInstance",docs: ["This account is used to store relevant information for a crafting process instance"],type: {kind: "struct",fields: [{name: "version",docs: ["The data version of this account"],type: "u8"}, {name: "seqId",docs: ["The sequence id for the `Starbase`"],type: "u16"}, {name: "authority",docs: ["The authority over the `CraftingInstance`"],type: "publicKey"}, {name: "craftingProcess",docs: ["The `CraftingProcess` account address"],type: "publicKey"}, {name: "instanceType",docs: ["the planet type"],type: "u8"}, {name: "numCrew",docs: ["The number of crew taking part in the crafting process"],type: "u64"}, {name: "bump",docs: ["Bump of Account PDA"],type: "u8"}]}}, {name: "disbandedFleet",docs: ["Keeps track of a fleet while it is disbanded"],type: {kind: "struct",fields: [{name: "version",docs: ["The data version of this account."],type: "u8"}, {name: "gameId",docs: ["The game id this belongs to."],type: "publicKey"}, {name: "ownerProfile",docs: ["The owner's profile."],type: "publicKey"}, {name: "starbase",docs: ["The `Starbase` at which the original `Fleet` was disbanded."],type: "publicKey"}, {name: "fleetLabel",docs: ["The label or name of the disbanded fleet."],type: {array: ["u8", 32]}}, {name: "fleetShips",docs: ["The `FleetShips` account belonging to the original `Fleet` that was disbanded."],type: "publicKey"}, {name: "bump",docs: ["The disbanded fleet's bump."],type: "u8"}]}}, {name: "fleet",docs: ["A `SAGE` fleet."],type: {kind: "struct",fields: [{name: "version",docs: ["The data version of this account."],type: "u8"}, {name: "gameId",docs: ["The game id this belongs to."],type: "publicKey"}, {name: "ownerProfile",docs: ["The owner's profile."],type: "publicKey"}, {name: "fleetShips",docs: ["Fleet Ships Key"],type: "publicKey"}, {name: "subProfile",docs: ["The fleet's sub-authority.", "If [`Some`] will have the exclusive ability to interact with this fleet."],type: {defined: "OptionalNonSystemPubkey"}}, {name: "subProfileInvalidator",docs: ["The authority for revoking a sun-authority."],type: "publicKey"}, {name: "faction",docs: ["The faction of the profile."],type: "u8"}, {name: "fleetLabel",docs: ["The label or name of the fleet."],type: {array: ["u8", 32]}}, {name: "shipCounts",docs: ["The number of ships in the fleet."],type: {defined: "ShipCounts"}}, {name: "warpCooldownExpiresAt",docs: ["The time at which the warp cooldown expires"],type: "i64"}, {name: "scanCooldownExpiresAt",docs: ["The time at which the scan cooldown expires"],type: "i64"}, {name: "stats",docs: ["The fleet's stats."],type: {defined: "ShipStats"}}, {name: "cargoHold",docs: ["The Cargo pod representing the fleet's cargo hold"],type: "publicKey"}, {name: "fuelTank",docs: ["The Cargo pod representing the fleet's fuel tank"],type: "publicKey"}, {name: "ammoBank",docs: ["The Cargo pod representing the fleet's ammo bank"],type: "publicKey"}, {name: "updateId",docs: ["The update id for the `Fleet`"],type: "u64"}, {name: "bump",docs: ["The fleet's bump."],type: "u8"}]}}, {name: "fleetShips",docs: ["Keeps track of a the individual ships that make up a fleet"],type: {kind: "struct",fields: [{name: "version",docs: ["The data version of this account."],type: "u8"}, {name: "fleet",docs: ["The `Fleet` account this belongs to"],type: "publicKey"}, {name: "fleetShipsInfoCount",docs: ["List length of `RemainingData`"],type: "u32"}, {name: "bump",docs: ["The disbanded fleet's bump."],type: "u8"}]}}, {name: "game",docs: ["Global Game Configuration variables"],type: {kind: "struct",fields: [{name: "version",docs: ["The data version of this account."],type: "u8"}, {name: "updateId",docs: ["The sequence id for updates."],type: "u64"}, {name: "profile",docs: ["The [`Profile`](player_profile::state::Profile) that handles the sector program permissions"],type: "publicKey"}, {name: "gameState",docs: ["The associated `GameState` account."],type: "publicKey"}, {name: "points",docs: ["Points setting"],type: {defined: "Points"}}, {name: "cargo",docs: ["Cargo settings"],type: {defined: "Cargo"}}, {name: "crafting",docs: ["Crafting settings"],type: {defined: "Crafting"}}, {name: "mints",docs: ["mint related settings"],type: {defined: "Mints"}}, {name: "vaults",docs: ["vault related settings"],type: {defined: "Vaults"}}, {name: "riskZones",docs: ["Data for risk zones"],type: {defined: "RiskZonesData"}}]}}, {name: "gameState",docs: ["Keeps track of variables that may change frequently during a `Game` session"],type: {kind: "struct",fields: [{name: "version",docs: ["The data version of this account"],type: "u8"}, {name: "updateId",docs: ["The sequence id for updates"],type: "u64"}, {name: "gameId",docs: ["The `Game` that this belongs to"],type: "publicKey"}, {name: "fleet",docs: ["Fleet settings"],type: {defined: "FleetInfo"}}, {name: "misc",docs: ["Miscellaneous settings"],type: {defined: "MiscVariables"}}, {name: "bump",docs: ["PDA bump"],type: "u8"}]}}, {name: "mineItem",docs: ["Represents a token registered as an item that can be mined"],type: {kind: "struct",fields: [{name: "version",docs: ["The data version of this account."],type: "u8"}, {name: "gameId",docs: ["the game_id account this item is registered with"],type: "publicKey"}, {name: "name",docs: ["The name of the `MineItem`"],type: {array: ["u8", 64]}}, {name: "mint",docs: ["the mint representing the items mined"],type: "publicKey"}, {name: "resourceHardness",docs: ["How hard it is to mine this item -> Ranges from 1-10"],type: "u16"}, {name: "numResourceAccounts",docs: ["The number of resource accounts for this mine item"],type: "u64"}, {name: "bump",docs: ["bump for PDA"],type: "u8"}]}}, {name: "planet",docs: ["Planet"],type: {kind: "struct",fields: [{name: "version",docs: ["The data version of this account."],type: "u8"}, {name: "name",docs: ["The name of this `Planet`"],type: {array: ["u8", 64]}}, {name: "gameId",docs: ["the `Game` that this belongs to"],type: "publicKey"}, {name: "sector",docs: ["the sector that this belongs to"],type: {array: ["i64", 2]}}, {name: "subCoordinates",docs: ["sub_coordinates as [x, y]"],type: {array: ["i64", 2]}}, {name: "planetType",docs: ["the planet type"],type: "u8"}, {name: "position",docs: ["the planet position"],type: "u8"}, {name: "size",docs: ["size"],type: "u64"}, {name: "maxHp",docs: ["maximum health"],type: "u64"}, {name: "currentHealth",docs: ["The current health of the `Planet`."],type: "u64"}, {name: "amountMined",docs: ["the cumulative amount mined from this `Asteroid`"],type: "u64"}, {name: "numResources",docs: ["the number of resources at this `Asteroid`"],type: "u8"}, {name: "numMiners",docs: ["the number of entities currently mining at this `Asteroid`"],type: "u64"}]}}, {name: "progressionConfig",docs: ["Progression Config"],type: {kind: "struct",fields: [{name: "version",docs: ["the data version of this account."],type: "u8"}, {name: "gameId",docs: ["the `Game` that this belongs to"],type: "publicKey"}, {name: "dailyLpLimit",docs: ["the daily limit for Loyalty Points (LP)"],type: "u64"}, {name: "dailyCouncilRankXpLimit",docs: ["the daily limit for Council Rank Experience Points (CRXP)"],type: "u64"}, {name: "dailyPilotXpLimit",docs: ["the daily limit for Pilot License Experience Points (PXP)"],type: "u64"}, {name: "dailyDataRunningXpLimit",docs: ["the daily limit for Data Running Experience Points (DRXP)"],type: "u64"}, {name: "dailyMiningXpLimit",docs: ["the daily limit for Mining Experience Points (MXP)"],type: "u64"}, {name: "dailyCraftingXpLimit",docs: ["the daily limit for Crafting Experience Points (CXP)"],type: "u64"}, {name: "numItems",docs: ["number of progression items being tracked"],type: "u16"}]}}, {name: "resource",docs: ["Represents a mine-able item existing at a particular location (e.g. a planet)"],type: {kind: "struct",fields: [{name: "version",docs: ["The data version of this account."],type: "u8"}, {name: "gameId",docs: ["the game_id pubkey"],type: "publicKey"}, {name: "location",docs: ["the location's pubkey"],type: "publicKey"}, {name: "mineItem",docs: ["the mine item pubkey"],type: "publicKey"}, {name: "locationType",docs: ["the location type"],type: "u8"}, {name: "systemRichness",docs: ["How abundant the resource is at the location -> Ranges from 1-5"],type: "u16"}, {name: "amountMined",docs: ["the cumulative amount mined from this resource"],type: "u64"}, {name: "numMiners",docs: ["the number of entities currently mining this resource"],type: "u64"}, {name: "bump",docs: ["bump for PDA"],type: "u8"}]}}, {name: "sagePlayerProfile",docs: ["A `SAGE` player's profile."],type: {kind: "struct",fields: [{name: "version",docs: ["The data version of this account"],type: "u8"}, {name: "playerProfile",docs: ["The `Profile` key"],type: "publicKey"}, {name: "gameId",docs: ["The id of the `Game`"],type: "publicKey"}, {name: "bump",docs: ["Bump of Account PDA"],type: "u8"}]}}, {name: "sector",docs: ["Sector"],type: {kind: "struct",fields: [{name: "version",docs: ["The data version of this account."],type: "u8"}, {name: "gameId",docs: ["the game_id that this belongs to"],type: "publicKey"}, {name: "coordinates",docs: ["coordinates as [x, y]"],type: {array: ["i64", 2]}}, {name: "discoverer",docs: ["The discoverer of this sector"],type: "publicKey"}, {name: "name",docs: ["The name of this sector"],type: {array: ["u8", 64]}}, {name: "numStars",docs: ["the number of stars in this system"],type: "u16"}, {name: "numPlanets",docs: ["the number of planets in this system"],type: "u16"}, {name: "numMoons",docs: ["the number of moons in this system"],type: "u16"}, {name: "numAsteroidBelts",docs: ["the number of num_asteroid belts in this system"],type: "u16"}, {name: "lastScanTime",docs: ["The last time the `Sector` was scanned"],type: "i64"}, {name: "lastScanChance",docs: ["The probability of finding SDUs in the `Sector` from the last time it was scanned"],type: "u32"}, {name: "bump",docs: ["PDA bump"],type: "u8"}, {name: "numConnections",docs: ["the number of connections in this system"],type: "u16"}]}}, {name: "ship",docs: ["This account represents a Ship"],type: {kind: "struct",fields: [{name: "version",docs: ["The data version of this account."],type: "u8"}, {name: "gameId",docs: ["the game_id account this Ship is registered with"],type: "publicKey"}, {name: "mint",docs: ["the mint representing the Ship"],type: "publicKey"}, {name: "name",docs: ["The name of this `Ship`"],type: {array: ["u8", 64]}}, {name: "sizeClass",docs: ["the ship's size class"],type: "u8"}, {name: "stats",docs: ["The ship's stats"],type: {defined: "ShipStats"}}, {name: "updateId",docs: ["The `update_id` for the `Ship`"],type: "u64"}, {name: "maxUpdateId",docs: ["The max `Game` `update_id` that the `Ship` is valid for"],type: "u64"}, {name: "next",docs: ["the next `Ship` account to use when this `Ship` is updated"],type: {defined: "OptionalNonSystemPubkey"}}]}}, {name: "star",docs: ["`Star` account"],type: {kind: "struct",fields: [{name: "version",docs: ["The data version of this account."],type: "u8"}, {name: "name",docs: ["The name of this `Star`"],type: {array: ["u8", 64]}}, {name: "gameId",docs: ["the game_id that this belongs to"],type: "publicKey"}, {name: "sector",docs: ["the sector that this belongs to"],type: {array: ["i64", 2]}}, {name: "size",docs: ["size"],type: "u64"}, {name: "subCoordinates",docs: ["sub_coordinates as [x, y]"],type: {array: ["i64", 2]}}, {name: "starType",docs: ["the star type"],type: "u8"}]}}, {name: "starbase",docs: ["Starbase"],type: {kind: "struct",fields: [{name: "version",docs: ["The data version of this `Starbase` account."],type: "u8"}, {name: "gameId",docs: ["the game_id that this `Starbase` belongs to"],type: "publicKey"}, {name: "sector",docs: ["the sector that this `Starbase` belongs to"],type: {array: ["i64", 2]}}, {name: "craftingFacility",docs: ["the [`CraftingFacility`] to use for crafting at this `Starbase`"],type: "publicKey"}, {name: "upgradeFacility",docs: ["the [`CraftingFacility`] to use for upgrade jobs at this `Starbase`"],type: "publicKey"}, {name: "name",docs: ["The name of this `Starbase`"],type: {array: ["u8", 64]}}, {name: "subCoordinates",docs: ["coordinates as [x, y]"],type: {array: ["i64", 2]}}, {name: "faction",docs: ["The faction of the `Starbase`."],type: "u8"}, {name: "bump",docs: ["bump for PDA"],type: "u8"}, {name: "seqId",docs: ["The sequence id for the `Starbase`"],type: "u16"}, {name: "state",docs: ["The state of the `Starbase`. Is a [`StarbaseState`]."],type: "u8"}, {name: "level",docs: ["The level of the `Starbase`."],type: "u8"}, {name: "hp",docs: ["The `Starbase` health points."],type: "u64"}, {name: "sp",docs: ["The `Starbase` shield points."],type: "u64"}, {name: "sectorRingAvailable",docs: ["The planet position (`sector::state::Ring`) available for this `Starbase`"],type: "u8"}, {name: "upgradeState",docs: ["The `Starbase` upgrade state"],type: "u8"}, {name: "upgradeIngredientsChecksum",docs: ["used to check if expected upgrade ingredients have been supplied"],type: {array: ["u8", 16]}}, {name: "numUpgradeIngredients",docs: ["number of ingredients needed for starbase upgrade"],type: "u8"}, {name: "upkeepAmmoBalance",docs: ["The balance of ammo for upkeep"],type: "u64"}, {name: "upkeepAmmoLastUpdate",docs: ["The last time ammo for upkeep was updated (Local time)"],type: "i64"}, {name: "upkeepAmmoGlobalLastUpdate",docs: ["The last time ammo for upkeep was updated (Global time)"],type: "i64"}, {name: "upkeepFoodBalance",docs: ["The balance of food for upkeep"],type: "u64"}, {name: "upkeepFoodLastUpdate",docs: ["The last time food for upkeep was updated (Local time)"],type: "i64"}, {name: "upkeepFoodGlobalLastUpdate",docs: ["The last time food for upkeep was updated (Global time)"],type: "i64"}, {name: "upkeepToolkitBalance",docs: ["The balance of toolkits for upkeep"],type: "u64"}, {name: "upkeepToolkitLastUpdate",docs: ["The last time toolkits for upkeep was updated (Local time)"],type: "i64"}, {name: "upkeepToolkitGlobalLastUpdate",docs: ["The last time toolkits for upkeep was updated (Global time)"],type: "i64"}, {name: "builtDestroyedTimestamp",docs: ["The last time the starbase was built or destroyed"],type: "i64"}]}}, {name: "starbasePlayer",docs: ["The `SAGE` player info within a `Starbase`"],type: {kind: "struct",fields: [{name: "version",docs: ["The data version of this account"],type: "u8"}, {name: "playerProfile",docs: ["The `Profile` key"],type: "publicKey"}, {name: "gameId",docs: ["The id of the `Game`"],type: "publicKey"}, {name: "starbase",docs: ["The `Starbase` key"],type: "publicKey"}, {name: "sagePlayerProfile",docs: ["The `SagePlayerProfile` key"],type: "publicKey"}, {name: "bump",docs: ["Bump of Account PDA"],type: "u8"}, {name: "shipEscrowCount",docs: ["List length of `RemainingData`"],type: "u32"}, {name: "totalCrewOld",docs: ["The total OLD crew members from the player's fleets at the Starbase"],type: "u32"}, {name: "totalCrew",docs: ["The total crew members from the player's fleets at the Starbase"],type: "u32"}, {name: "busyCrew",docs: ["The number of crew members that is engaged/busy and not available"],type: "u64"}, {name: "updateId",docs: ["The `Game` update id"],type: "u64"}, {name: "updatedShipEscrowCount",docs: ["Number of updated items in `RemainingData` list", "This will be `ship_escrow_count` when all ships in escrow are up-to-date"],type: "u32"}]}}, {name: "surveyDataUnitTracker",docs: ["Survey Data Unit (SDU) Tracker"],type: {kind: "struct",fields: [{name: "version",docs: ["The data version of this account."],type: "u8"}, {name: "gameId",docs: ["The game_id that this belongs to"],type: "publicKey"}, {name: "sduMint",docs: ["The Survey Data Unit Mint"],type: "publicKey"}, {name: "resourceMint",docs: ["The mint of the resource spent when scanning for SDUs"],type: "publicKey"}, {name: "signer",docs: ["The signer for this account"],type: "publicKey"}, {name: "signerBump",docs: ["The signer for this account"],type: "u8"}, {name: "coordinatesRange",docs: ["The valid coordinates range", "e.g. a value of [-50, 50] means that coordinates from [-50, -50] to [50, 50] are valid for SDU scanning"],type: {array: ["i64", 2]}}, {name: "cssCoordinates",docs: ["The locations of the central space stations (CSS) of the three factions"],type: {array: [{array: ["i64", 2]}, 3]}}, {name: "originCoordinates",docs: ['The co-ordinates of the "origin"; used in calculating SDU probability'],type: {array: ["i64", 2]}}, {name: "cssMaxDistance",docs: ["The max distance from the nearest CSS; used in calculating SDU probability"],type: "u32"}, {name: "originMaxDistance",docs: ["The max distance from the `origin_coordinates`; used in calculating SDU probability"],type: "u32"}, {name: "distanceWeighting",docs: ["The distance weighting; used in calculating SDU probability"],type: "u32"}, {name: "tMax",docs: ["The maximum time before SDU probability at a location changes"],type: "i64"}, {name: "xMul",docs: ["Multiplier in the X dimension; used in noise function"],type: "u32"}, {name: "yMul",docs: ["Multiplier in the Y dimension; used in noise function"],type: "u32"}, {name: "zMul",docs: ["Multiplier in the Z dimension; used in noise function"],type: "u32"}, {name: "sduMaxPerSector",docs: ["The maximum number of SDUs that can be found per scan per sector"],type: "u32"}, {name: "scanChanceRegenPeriod",docs: ["The amount of time in seconds that it takes for a sector scan chance to fully regenerate"],type: "i16"}]}}],types: [{name: "AddShipEscrowInput",docs: ["Struct for data input for `AddShipEscrow`"],type: {kind: "struct",fields: [{name: "shipAmount",docs: ["Amount of `Ship` tokens to transfer to escrow"],type: "u64"}, {name: "index",docs: ["Index of `WrappedShipEscrow` in remaining data of `StarbasePlayer`", "Some index `WrappedShipEscrow`, or None for new `WrappedShipEscrow`"],type: {option: "u32"}}]}}, {name: "AddShipToFleetInput",docs: ["Struct for data input for that has `key_index`"],type: {kind: "struct",fields: [{name: "shipAmount",docs: ["Number of ships to add to the fleet"],type: "u8"}, {name: "shipEscrowIndex",docs: ["Index of `WrappedShipEscrow` in remaining data of `StarbasePlayer`"],type: "u32"}, {name: "fleetShipInfoIndex",docs: ["Index of `FleetShipsInfo` in remaining data of `FleetShips`"],type: {option: "u32"}}, {name: "keyIndex",docs: ["the index of the key in the player profile"],type: "u16"}]}}, {name: "Cargo",docs: ["Variables for the Cargo program"],type: {kind: "struct",fields: [{name: "statsDefinition",docs: ["The cargo stats definition account"],type: "publicKey"}]}}, {name: "CargoStats",docs: ["A ship's cargo stats"],type: {kind: "struct",fields: [{name: "cargoCapacity",docs: ["the capacity of the ship's cargo hold"],type: "u32"}, {name: "fuelCapacity",docs: ["the capacity of the ship's fuel tank"],type: "u32"}, {name: "ammoCapacity",docs: ["the capacity of the ship's ammo bank"],type: "u32"}, {name: "ammoConsumptionRate",docs: ["the amount of ammo consumed per second by the ship when doing non-combat activities e.g. mining"],type: "u32"}, {name: "foodConsumptionRate",docs: ["the amount of food consumed per second by the ship when doing non-combat activities e.g. mining"],type: "u32"}, {name: "miningRate",docs: ["the amount of resources that can be mined by a ship per second"],type: "u32"}, {name: "upgradeRate",docs: ["the amount of upgrade material that is consumed by a ship per second while upgrading a Starbase"],type: "u32"}, {name: "cargoTransferRate",docs: ["the amount of cargo that a ship can transfer per second to another ship outside of dock"],type: "u32"}, {name: "tractorBeamGatherRate",docs: ["the amount of cargo that the ship can gather per second using its tractor beam"],type: "u32"}]}}, {name: "CargoStatsUnpacked",docs: ["Unpacked version of [`CargoStats`]"],type: {kind: "struct",fields: [{name: "cargoCapacity",docs: ["the capacity of the ship's cargo hold"],type: "u32"}, {name: "fuelCapacity",docs: ["the capacity of the ship's fuel tank"],type: "u32"}, {name: "ammoCapacity",docs: ["the capacity of the ship's ammo bank"],type: "u32"}, {name: "ammoConsumptionRate",docs: ["the amount of ammo consumed per second by the ship when doing non-combat activities e.g. mining"],type: "u32"}, {name: "foodConsumptionRate",docs: ["the amount of food consumed per second by the ship when doing non-combat activities e.g. mining"],type: "u32"}, {name: "miningRate",docs: ["the amount of resources that can be mined by a ship per second"],type: "u32"}, {name: "upgradeRate",docs: ["the amount of upgrade material that is consumed by a ship per second while upgrading a Starbase"],type: "u32"}, {name: "cargoTransferRate",docs: ["the amount of cargo that a ship can transfer per second to another ship outside of dock"],type: "u32"}, {name: "tractorBeamGatherRate",docs: ["the amount of cargo that the ship can gather per second using its tractor beam"],type: "u32"}]}}, {name: "CargoToGameInput",docs: ["Struct for data input to `DepositCargoToGame`"],type: {kind: "struct",fields: [{name: "amount",docs: ["cargo amount"],type: "u64"}, {name: "keyIndex",docs: ["the index of the key in the player profile"],type: "u16"}]}}, {name: "CloseDisbandedFleetInput",docs: ["Struct for data input for that has `key_index`"],type: {kind: "struct",fields: [{name: "keyIndex",docs: ["the index of the key in the player profile"],type: "u16"}]}}, {name: "Crafting",docs: ["Variables for the Crafting program"],type: {kind: "struct",fields: [{name: "domain",docs: ["The crafting domain account"],type: "publicKey"}]}}, {name: "CraftingInstanceType",docs: ["Represents the type of `CraftingInstance`"],type: {kind: "enum",variants: [{name: "StarbaseCrafting"}, {name: "StarbaseUpgradeMaterial"}]}}, {name: "CreateFleetInput",docs: ["Struct for data input for that has `key_index`"],type: {kind: "struct",fields: [{name: "shipAmount",docs: ["Number of ships to add to the fleet"],type: "u8"}, {name: "fleetLabel",docs: ["the fleet label"],type: {array: ["u8", 32]}}, {name: "shipEscrowIndex",docs: ["Index of `WrappedShipEscrow` in remaining data of `StarbasePlayer`"],type: "u32"}, {name: "cargoHoldSeeds",docs: ["cargo hold seeds"],type: {array: ["u8", 32]}}, {name: "fuelTankSeeds",docs: ["fuel tank seeds"],type: {array: ["u8", 32]}}, {name: "ammoBankSeeds",docs: ["ammo bank seeds"],type: {array: ["u8", 32]}}, {name: "keyIndex",docs: ["the index of the key in the player profile"],type: "u16"}]}}, {name: "DepositCargoToFleetInput",docs: ["Struct for data input to `DepositCargoToFleet`"],type: {kind: "struct",fields: [{name: "amount",docs: ["cargo amount"],type: "u64"}, {name: "keyIndex",docs: ["the index of the key in the player profile"],type: "u16"}]}}, {name: "DepositStarbaseUpkeepResourceInput",docs: ["Submit starbase upkeep resource inputs"],type: {kind: "struct",fields: [{name: "pointsProgramPermissionsKeyIndex",docs: ["the index of the points program permissions in the player profile"],type: "u16"}, {name: "sagePermissionsKeyIndex",docs: ["the index of the key in sage permissions in the player profile"],type: "u16"}, {name: "resourceType",docs: ["the resource type"],type: "u8"}, {name: "resourceIndex",docs: ["the index of the resource represented by `token_mint` in the `resource_recipe` ingredients list"],type: "u16"}, {name: "amount",docs: ["the amount"],type: "u64"}, {name: "epochIndex",docs: ["the index of the epoch in the `RedemptionConfig` account"],type: "u16"}]}}, {name: "DeregisterSurveyDataUnitTrackerInput",docs: ["Struct for data input that has `key_index`"],type: {kind: "struct",fields: [{name: "keyIndex",docs: ["the index of the key in the sector permissions profile"],type: "u16"}]}}, {name: "DisbandFleetInput",docs: ["Struct for data input for that has `key_index`"],type: {kind: "struct",fields: [{name: "keyIndex",docs: ["the index of the key in the player profile"],type: "u16"}]}}, {name: "DisbandedFleetToEscrowInput",docs: ["Struct for data input for that has `key_index`"],type: {kind: "struct",fields: [{name: "shipAmount",docs: ["Number of ships to add to the fleet"],type: "u16"}, {name: "shipEscrowIndex",docs: ["Index of `WrappedShipEscrow` in remaining data of `StarbasePlayer`"],type: {option: "u32"}}, {name: "fleetShipInfoIndex",docs: ["Index of `FleetShipsInfo` in remaining data of `FleetShips`"],type: "u32"}, {name: "keyIndex",docs: ["the index of the key in the player profile"],type: "u16"}]}}, {name: "DiscoverSectorInput",docs: ["Struct for data input for `DiscoverSector`"],type: {kind: "struct",fields: [{name: "coordinates",docs: ["The coordinates of the new `Sector`"],type: {array: ["i64", 2]}}, {name: "keyIndex",docs: ["the index of the key in the player profile"],type: "u16"}]}}, {name: "FactionsStarbaseLevelInfo",docs: ["`Starbase` levels discriminated by faction"],type: {kind: "struct",fields: [{name: "mud",docs: ["Mud Starbase Levels Info"],type: {array: [{defined: "StarbaseLevelInfo"}, 7]}}, {name: "oni",docs: ["Oni Starbase Levels Info"],type: {array: [{defined: "StarbaseLevelInfo"}, 7]}}, {name: "ustur",docs: ["Ustur Starbase Levels Info"],type: {array: [{defined: "StarbaseLevelInfo"}, 7]}}]}}, { name: "FleetCrewInput", docs: ["Struct for data input to `FleetCrew`"], type: { kind: "struct", fields: [{ name: "count", docs: ["passenger count"], type: "u16" }, { name: "keyIndex", docs: ["the index of the key in the player profile"], type: "u16" }] } }, {name: "FleetInfo",docs: ["Variables for the Fleet program"],type: {kind: "struct",fields: [{name: "starbaseLevels",docs: ["`Starbase` levels discriminated by faction"],type: {defined: "FactionsStarbaseLevelInfo"}}, {name: "upkeep",docs: ["`Starbase` upkeep discriminated by level"],type: {defined: "StarbaseUpkeepLevels"}}, {name: "maxFleetSize",docs: ["Maximum `Fleet` size allowed"],type: "u32"}]}}, {name: "FleetInput",docs: ["Struct for data input to Update fleet settings"],type: {kind: "struct",fields: [{name: "starbaseLevelInfoArray",docs: ["`Starbase` Level Info array"],type: {option: {vec: {defined: "StarbaseLevelInfoArrayInput"}}}}, {name: "upkeepInfoArray",docs: ["`Starbase` Level Info array"],type: {option: {vec: {defined: "StarbaseUpkeepInfoArrayInput"}}}}, {name: "maxFleetSize",docs: ["Maximum `Fleet` size allowed"],type: {option: "u32"}}]}}, {name: "FleetShipsInfo",docs: ["Struct that represents info on a single ship type in a fleet"],type: {kind: "struct",fields: [{name: "ship",docs: ["The `Ship` account address"],type: "publicKey"}, {name: "amount",docs: ["The `Ship` token amount in escrow"],type: "u64"}, {name: "updateId",docs: ["The update id for the `Ship`"],type: "u64"}]}}, {name: "ForcedDisbandFleetInput",docs: ["Struct for data input for that has `key_index`"],type: {kind: "struct",fields: [{name: "fleetShipInfoIndex",docs: ["Index of `FleetShipsInfo` in remaining data of `FleetShips`"],type: "u32"}]}}, {name: "Idle",docs: ["The data for the [`FleetStateData::Idle`](crate::state_machine::FleetStateData::Idle) state"],type: {kind: "struct",fields: [{name: "sector",docs: ["The star system the fleet is in"],type: {array: ["i64", 2]}}]}}, {name: "IdleToRespawnInput",docs: ["Struct for data input to initialize an `IdleToRespawn` Ix"],type: {kind: "struct",fields: [{name: "keyIndex",docs: ["index of the key in the player profile"],type: "u16"}]}}, {name: "IngredientIndexInput",docs: ["Struct for data input that has `key_index`"],type: {kind: "struct",fields: [{name: "ingredientIndex",docs: ["the index of the recipe output"],type: "u16"}]}}, {name: "InitGameStateInput",docs: ["Struct for data input to `InitGameState`"],type: {kind: "struct",fields: [{name: "keyIndex",docs: ["the index of the key in the sector permissions profile"],type: "u16"}]}}, {name: "KeyIndexInput",docs: ["Struct for data input that has `key_index`"],type: {kind: "struct",fields: [{name: "keyIndex",docs: ["the index of the key in the sector permissions profile"],type: "u16"}]}}, {name: "LocationType",docs: ["Represents different types of locations that a `Resource` might be found"],type: {kind: "enum",variants: [{name: "Planet"}]}}, {name: "ManageGameInput",docs: ["Struct for data input to managing Game accounts"],type: {kind: "struct",fields: [{name: "keyIndex",docs: ["the index of the key in the sector permissions profile"],type: "u16"}]}}, {name: "MineAsteroid",docs: ["The data for the [`FleetStateData::MineAsteroid`](crate::state_machine::FleetStateData::MineAsteroid) state"],type: {kind: "struct",fields: [{name: "asteroid",docs: ["The `Asteroid` the `Fleet` is mining (Must be an asteroid belt)"],type: "publicKey"}, {name: "resource",docs: ["The `Resource` being mined on the `Asteroid`"],type: "publicKey"}, {name: "start",docs: ["The timestamp at which mining activity started"],type: "i64"}, {name: "end",docs: ["The timestamp at which mining activity stops"],type: "i64"}, {name: "amountMined",docs: ["The cumulative amount mined"],type: "u64"}, {name: "lastUpdate",docs: ["The last time the `Fleet` was updated"],type: "i64"}]}}, {name: "MineAsteroidToRespawnInput",docs: ["Struct for data input for `MineAsteroidToRespawnInput`"],type: {kind: "struct",fields: [{name: "keyIndex",docs: ["the index of the key in the player profile"],type: "u16"}]}}, {name: "Mints",docs: ["Token mints"],type: {kind: "struct",fields: [{name: "atlas",docs: ["ATLAS token mint"],type: "publicKey"}, {name: "polis",docs: ["POLIS token mint"],type: "publicKey"}, {name: "ammo",docs: ["ammunition"],type: "publicKey"}, {name: "food",docs: ["food"],type: "publicKey"}, {name: "fuel",docs: ["fuel"],type: "publicKey"}, {name: "repairKit",docs: ["repair kit"],type: "publicKey"}]}}, {name: "MiscStats",docs: ["A ship's miscellaneous stats"],type: {kind: "struct",fields: [{name: 'requiredCrew', docs: ['Number of crew required to operate the ship'], type: 'u16' }, { name: 'passengerCapacity', docs: ['Number of crew that the ship can carry as passengers'], type: 'u16' }, { name: 'crewCount', docs: ['Total number of crew on the ship (required crew + passengers on board) / Will be zero until `CREW_FEATURE` is enabled'], type: 'u16'}, { name: 'rentedCrew', docs: ['Total number of crew that were rented out together with the fleet'], type: 'u16'}, {name: "respawnTime",docs: ["the time it takes the ship to respawn"],type: "u16"}, {name: "scanCoolDown",docs: ["the time it takes the ship to be able to scan again after scanning"],type: "u16"}, {name: "sduPerScan",docs: ["The number of SDUs that can be found while scanning"],type: "u32"}, {name: "scanCost",docs: ["the amount of resource required to do a scan"],type: "u32"}, {name: "placeholder",docs: ["TODO: this is a placeholder stats for use in the future"],type: "u32"}, {name: "placeholder2",docs: ["TODO: this is a placeholder stats for use in the future"],type: "u32"}, {name: "placeholder3",docs: ["TODO: this is a placeholder stats for use in the future"],type: "u32"}]}}, {name: "MiscStatsUnpacked",docs: ["Unpacked version of [`MiscStats`]"],type: {kind: "struct",fields: [{name: 'requiredCrew', docs: ['Number of crew required to operate the ship'], type: 'u16' }, { name: 'passengerCapacity', docs: ['Number of crew that the ship can carry as passengers'], type: 'u16' }, { name: 'crewCount', docs: ['Total number of crew on the ship (required crew + passengers on board) / Will be zero until `CREW_FEATURE` is enabled'], type: 'u16'}, { name: 'rentedCrew', docs: ['Total number of crew that were rented out together with the fleet'], type: 'u16'}, {name: "respawnTime",docs: ["the time it takes the ship to respawn"],type: "u16"}, {name: "scanCoolDown",docs: ["the time it takes the ship to be able to scan again after scanning"],type: "u16"}, {name: "sduPerScan",docs: ["The number of SDUs that can be found while scanning"],type: "u32"}, {name: "scanCost",docs: ["the amount of resource required to do a scan"],type: "u32"}, {name: "placeholder",docs: ["TODO: this is a placeholder stats for use in the future"],type: "u32"}, {name: "placeholder2",docs: ["TODO: this is a placeholder stats for use in the future"],type: "u32"}, {name: "placeholder3",docs: ["TODO: this is a placeholder stats for use in the future"],type: "u32"}]}}, {name: "MiscVariables",docs: ["Miscellaneous game state variables"],type: {kind: "struct",fields: [{name: "warpLaneFuelCostReduction",docs: ['Percentage by which the "warp lane" movement type reduces warp fuel cost'],type: "i16"}, {name: "respawnFee",docs: ["Respawn fee; You cannot enter into the respawning state without paying this fee", "Since ATLAS has 8 decimal places, units are in the smallest value of ATLAS possible."],type: "u64"}, {name: "upkeepMiningEmissionsPenalty",docs: ["Percentage by which to reduce the asteroid mining rate if a starbase ammo upkeep coffer is empty"],type: "i16"}]}}, {name: "MiscVariablesInput",docs: ["Struct for data input to update miscellaneous settings"],type: {kind: "struct",fields: [{name: "warpLaneFuelCostReduction",docs: ['Percentage by which the "warp lane" movement type reduces warp fuel cost'],type: {option: "i16"}}, {name: "upkeepMiningEmissionsPenalty",docs: ["Percentage by which to reduce the asteroid mining rate if a starbase ammo upkeep coffer is empty"],type: {option: "i16"}}, {name: "respawnFee",docs: ["Respawn fee, charged in ATLAS"],type: {option: "u64"}}]}}, {name: "MoveSubwarp",docs: ["The data for the [`FleetStateData::MoveSubwarp`] state"],type: {kind: "struct",fields: [{name: "fromSector",docs: ["The sector the fleet is coming from"],type: {array: ["i64", 2]}}, {name: "toSector",docs: ["The sector the fleet is going to"],type: {array: ["i64", 2]}}, {name: "currentSector",docs: ["The sector the fleet is currently in"],type: {array: ["i64", 2]}}, {name: "departureTime",docs: ["When the fleet started subwarp"],type: "i64"}, {name: "arrivalTime",docs: ["When the fleet will finish subwarp"],type: "i64"}, {name: "fuelExpenditure",docs: ["The fuel cost of the subwarp"],type: "u64"}, {name: "lastUpdate",docs: ["The last update time"],type: "i64"}]}}, {name: "MoveWarp",docs: ["The data for the [`FleetStateData::MoveWarp`] state"],type: {kind: "struct",fields: [{name: "fromSector",docs: ["The star system the fleet is coming from"],type: {array: ["i64", 2]}}, {name: "toSector",docs: ["The star system the fleet is going to"],type: {array: ["i64", 2]}}, {name: "warpStart",docs: ["When the fleet started warping"],type: "i64"}, {name: "warpFinish",docs: ["When the warp will end"],type: "i64"}]}}, {name: "MovementStats",docs: ["A ship's movement stats"],type: {kind: "struct",fields: [{name: "subwarpSpeed",docs: ["the amount of distance that the ship can cover in one second while sub-warping"],type: "u32"}, {name: "warpSpeed",docs: ["the amount of distance that the ship can cover in one second while warping"],type: "u32"}, {name: "maxWarpDistance",docs: ["the max distance that the ship can warp"],type: "u16"}, {name: "warpCoolDown",docs: ["the time it takes the ship to be able to warp again after a warp"],type: "u16"}, {name: "subwarpFuelConsumptionRate",docs: ["the amount of fuel consumed by the ship when sub-warp moving"],type: "u32"}, {name: "warpFuelConsumptionRate",docs: ["the amount of fuel consumed by the ship when warp moving"],type: "u32"}, {name: "planetExitFuelAmount",docs: ["the amount of fuel required to exit a planet"],type: "u32"}]}}, {name: "MovementStatsUnpacked",docs: ["Unpacked version of [`MovementStats`]"],type: {kind: "struct",fields: [{name: "subwarpSpeed",docs: ["the amount of distance that the ship can cover in one second while sub-warping"],type: "u32"}, {name: "warpSpeed",docs: ["the amount of distance that the ship can cover in one second while warping"],type: "u32"}, {name: "maxWarpDistance",docs: ["the max distance that the ship can warp"],type: "u16"}, {name: "warpCoolDown",docs: ["the time it takes the ship to be able to warp again after a warp"],type: "u16"}, {name: "subwarpFuelConsumptionRate",docs: ["the amount of fuel consumed by the ship when sub-warp moving"],type: "u32"}, {name: "warpFuelConsumptionRate",docs: ["the amount of fuel consumed by the ship when warp moving"],type: "u32"}, {name: "planetExitFuelAmount",docs: ["the amount of fuel required to exit a planet"],type: "u32"}]}}, {name: "OptionalNonSystemPubkey",docs: ["A pubkey sized option that is none if set to the system program."],type: {kind: "struct",fields: [{name: "key",type: "publicKey"}]}}, {name: "PlanetType",docs: ["Represents different types a `Planet` could be"],type: {kind: "enum",variants: [{name: "Terrestrial"}, {name: "Volcanic"}, {name: "Barren"}, {name: "AsteroidBelt"}, {name: "GasGiant"}, {name: "IceGiant"}, {name: "Dark"}]}}, {name: "Points",docs: ["Variables for the Points program"],type: {kind: "struct",fields: [{name: "lpCategory",docs: ["Represents the points category & modifier to use for Loyalty Points (LP)"],type: {defined: "SagePointsCategory"}}, {name: "councilRankXpCategory",docs: ["Represents the points category & modifier to use for Council Rank Experience Points (CRXP)"],type: {defined: "SagePointsCategory"}}, {name: "pilotXpCategory",docs: ["Represents the points category & modifier to use for Pilot License Experience Points (PXP)"],type: {defined: "SagePointsCategory"}}, {name: "dataRunningXpCategory",docs: ["Represents the points category & modifier to use for Data Running Experience Points (DRXP)"],type: {defined: "SagePointsCategory"}}, {name: "miningXpCategory",docs: ["Represents the points category & modifier to use for Mining Experience Points (MXP)"],type: {defined: "SagePointsCategory"}}, {name: "craftingXpCategory",docs: ["Represents the points category & modifier to use for Crafting Experience Points (CXP)"],type: {defined: "SagePointsCategory"}}]}}, {name: "ProgressionItem",docs: ["Progression Item"],type: {kind: "struct",fields: [{name: "value",docs: ["The progression points value"],type: "u32"}]}}, {name: "ProgressionItemInput",docs: ["Progression Item for Data Input"],type: {kind: "struct",fields: [{name: "itemType",docs: ["The type of progression item"],type: "u8"}, {name: "item",docs: ["The progression item"],type: {defined: "ProgressionItem"}}]}}, {name: "ProgressionItemInputUnpacked",docs: ["Unpacked version of [`ProgressionItemInput`]"],type: {kind: "struct",fields: [{name: "itemType",docs: ["The type of progression item"],type: "u8"}, {name: "item",docs: ["The progression item"],type: {defined: "ProgressionItem"}}]}}, {name: "ProgressionItemType",docs: ["The different types of progression items"],type: {kind: "enum",variants: [{name: "Subwarp"}, {name: "Warp"}, {name: "WarpLane"}, {name: "AsteroidExit"}, {name: "ScanUnsuccessful"}, {name: "ScanSuccessful"}, {name: "Mining"}, {name: "Crafting"}, {name: "Upkeep"}, {name: "Upgrade"}]}}, {name: "ProgressionItemUnpacked",docs: ["Unpacked version of [`ProgressionItem`]"],type: {kind: "struct",fields: [{name: "value",docs: ["The progression points value"],type: "u32"}]}}, {name: "RegisterMineItemInput",docs: ["Struct for data input to Register a Resource"],type: {kind: "struct",fields: [{name: "name",docs: ["The name of the `MineItem`"],type: {array: ["u8", 64]}}, {name: "resourceHardness",docs: ["How hard it is to mine this item"],type: "u16"}, {name: "keyIndex",docs: ["the index of the key in the fleet permissions profile"],type: "u16"}]}}, {name: "RegisterPlanetInput",docs: ["Struct for data input to Register Planet"],type: {kind: "struct",fields: [{name: "name",docs: ["`Planet` name"],type: {array: ["u8", 64]}}, {name: "size",docs: ["`Planet` size"],type: "u64"}, {name: "maxHp",docs: ["`Planet` max health"],type: "u64"}, {name: "subCoordinates",docs: ["`Planet` sub_coordinates"],type: {array: ["i64", 2]}}, {name: "planetType",docs: ["`Planet` type"],type: "u8"}, {name: "position",docs: ["`Planet` position"],type: "u8"}, {name: "keyIndex",docs: ["the index of the key in the sector permissions profile"],type: "u16"}]}}, {name: "RegisterProgressionConfigInput",docs: ["Data input for RegisterProgressionConfig"],type: {kind: "struct",fields: [{name: "dailyLpLimit",docs: ["the daily limit for Loyalty Points (LP)"],type: {option: "u64"}}, {name: "dailyCouncilRankXpLimit",docs: ["the daily limit for Council Rank Experience Points (CRXP)"],type: {option: "u64"}}, {name: "dailyPilotXpLimit",docs: ["the daily limit for Pilot License Experience Points (PXP)"],type: {option: "u64"}}, {name: "dailyDataRunningXpLimit",docs: ["the daily limit for Data Running Experience Points (DRXP)"],type: {option: "u64"}}, {name: "dailyMiningXpLimit",docs: ["the daily limit for Mining Experience Points (MXP)"],type: {option: "u64"}}, {name: "dailyCraftingXpLimit",docs: ["the daily limit for Crafting Experience Points (CXP)"],type: {option: "u64"}}, {name: "keyIndex",docs: ["the index of the key in the sector permissions profile"],type: "u16"}]}}, {name: "RegisterResourceInput",docs: ["Struct for data input to Register a Resource"],type: {kind: "struct",fields: [{name: "locationType",docs: ["`Resource` location type"],type: "u8"}, {name: "systemRichness",docs: ["`Resource` `system_richness`"],type: "u16"}, {name: "keyIndex",docs: ["the index of the key in the fleet permissions profile"],type: "u16"}]}}, {name: "RegisterSagePointsModifierInput",docs: ["Struct for data input to register a points modifier for SAGE program"],type: {kind: "struct",fields: [{name: "pointsCategoryType",docs: ["The points category type of the modifier"],type: "u8"}, {name: "keyIndex",docs: ["the index of the key in the player profile"],type: "u16"}]}}, {name: "RegisterShipInput",docs: ["Struct for data input to Register Ship"],type: {kind: "struct",fields: [{name: "name",docs: ["The `Ship` name/label"],type: {array: ["u8", 64]}}, {name: "sizeClass",docs: ["the ship's size class"],type: {defined: "SizeClass"}}, {name: "stats",docs: ["The stats for the ship"],type: {defined: "ShipStatsUnpacked"}}, {name: "keyIndex",docs: ["the index of the key in the sector permissions profile"],type: "u16"}, {name: "isActive",docs: ["Whether the ship is initialized to active (`update_id == current_update_id`)"],type: "bool"}]}}, {name: "RegisterStarInput",docs: ["Struct for data input to Register Star"],type: {kind: "struct",fields: [{name: "name",docs: ["`Star` name"],type: {array: ["u8", 64]}}, {name: "size",docs: ["`Star` size"],type: "u64"}, {name: "subCoordinates",docs: ["`Star` sub_coordinates"],type: {array: ["i64", 2]}}, {name: "starType",docs: ["`Star` type"],type: "u8"}, {name: "keyIndex",docs: ["the index of the key in the sector permissions profile"],type: "u16"}]}}, {name: "RegisterStarbaseInput",docs: ["Struct for data input to Register `Starbase`"],type: {kind: "struct",fields: [{name: "name",docs: ["`Starbase` name"],type: {array: ["u8", 64]}}, {name: "subCoordinates",docs: ["`Starbase` coordinates"],type: {array: ["i64", 2]}}, {name: "starbaseLevelIndex",docs: ["The index representing the level of the `Starbase` in the game variables."],type: "u8"}, {name: "faction",docs: ["The `Starbase` faction"],type: "u8"}, {name: "keyIndex",docs: ["the index of the key in the fleet permissions profile"],type: "u16"}]}}, {name: "RegisterStarbaseInputUnpacked",docs: ["Unpacked version of [`RegisterStarbaseInput`]"],type: {kind: "struct",fields: [{name: "name",docs: ["`Starbase` name"],type: {array: ["u8", 64]}}, {name: "subCoordinates",docs: ["`Starbase` coordinates"],type: {array: ["i64", 2]}}, {name: "starbaseLevelIndex",docs: ["The index representing the level of the `Starbase` in the game variables."],type: "u8"}, {name: "faction",docs: ["The `Starbase` faction"],type: "u8"}, {name: "keyIndex",docs: ["the index of the key in the fleet permissions profile"],type: "u16"}]}}, {name: "RegisterSurveyDataUnitTrackerInput",docs: ["Struct for data input to Register SurveyDataUnitTracker"],type: {kind: "struct",fields: [{name: "coordinatesRange",docs: ["The valid coordinates range", "e.g. a value of [-50, 50] means that coordinates from [-50, -50] to [50, 50] are valid for SDU scanning"],type: {array: ["i64", 2]}}, {name: "cssCoordinates",docs: ["The locations of the central space stations (CSS) of the three factions"],type: {array: [{array: ["i64", 2]}, 3]}}, {name: "originCoordinates",docs: ['The co-ordinates of the "origin"; used in calculating SDU probability'],type: {array: ["i64", 2]}}, {name: "cssMaxDistance",docs: ["The max distance from the nearest CSS; used in calculating SDU probability"],type: "u32"}, {name: "originMaxDistance",docs: ["The max distance from the `origin_coordinates`; used in calculating SDU probability"],type: "u32"}, {name: "distanceWeighting",docs: ["The distance weighting; used in calculating SDU probability"],type: "u32"}, {name: "tMax",docs: ["The maximum time before SDU probability at a location changes"],type: "i64"}, {name: "xMul",docs: ["Multiplier in the X dimension; used in noise function"],type: "u32"}, {name: "yMul",docs: ["Multiplier in the Y dimension; used in noise function"],type: "u32"}, {name: "zMul",docs: ["Multiplier in the Z dimension; used in noise function"],type: "u32"}, {name: "sduMaxPerSector",docs: ["The maximum number of SDUs that can be found per scan per sector"],type: "u32"}, {name: "scanChanceRegenPeriod",docs: ["The amount of time in seconds that it takes for a sector scan chance to fully regenerate"],type: "i16"}, {name: "keyIndex",docs: ["the index of the key in the sector permissions profile"],type: "u16"}]}}, {name: "RemoveShipEscrowInput",docs: ["Struct for data input for `RemoveShipEscrow`"],type: {kind: "struct",fields: [{name: "shipAmount",docs: ["Amount of `Ship` tokens to transfer from escrow"],type: "u64"}, {name: "permissionKeyIndex",docs: ["the index of the `ProfileKey` in `Profile` with required permissions"],type: "u16"}, {name: "shipEscrowIndex",docs: ["Index of `WrappedShipEscrow` in remaining data of `StarbasePlayer`"],type: "u32"}]}}, {name: "Respawn",docs: ["The data for the [`FleetStateData::Respawn`](crate::state_machine::FleetStateData::Respawn) state"],type: {kind: "struct",fields: [{name: "sector",docs: ["The star system the fleet was in when it entered the `Respawn` state"],type: {array: ["i64", 2]}}, {name: "start",docs: ["The time `Respawn` started"],type: "i64"}]}}, {name: "RespawnToLoadingBayInput",docs: ["Struct for data input to `RespawnToLoadingBay`"],type: {kind: "struct",fields: [{name: "keyIndex",docs: ["the index of the key in the player profile"],type: "u16"}]}}, {name: "RiskZoneData",docs: ["`RiskZone` center and radius"],type: {kind: "struct",fields: [{name: "center",docs: ["Risk zone center"],type: {array: ["i64", 2]}}, {name: "radius",docs: ["Risk zone radius"],type: "u64"}]}}, {name: "RiskZoneDataUnpacked",docs: ["Unpacked version of [`RiskZoneData`]"],type: {kind: "struct",fields: [{name: "center",docs: ["Risk zone center"],type: {array: ["i64", 2]}}, {name: "radius",docs: ["Risk zone radius"],type: "u64"}]}}, {name: "RiskZonesData",docs: ["[`RiskZoneData`] for [`RiskZones`]"],type: {kind: "struct",fields: [{name: "mudSecurityZone",docs: ["Mud security zone"],type: {defined: "RiskZoneData"}}, {name: "oniSecurityZone",docs: ["Oni security zone"],type: {defined: "RiskZoneData"}}, {name: "usturSecurityZone",docs: ["Ustur security zone"],type: {defined: "RiskZoneData"}}, {name: "highRiskZone",docs: ["High risk zone"],type: {defined: "RiskZoneData"}}, {name: "mediumRiskZone",docs: ["Medium risk zone"],type: {defined: "RiskZoneData"}}]}}, {name: "RiskZonesDataUnpacked",docs: ["Unpacked version of [`RiskZonesData`]"],type: {kind: "struct",fields: [{name: "mudSecurityZone",docs: ["Mud security zone"],type: {defined: "RiskZoneData"}}, {name: "oniSecurityZone",docs: ["Oni security zone"],type: {defined: "RiskZoneData"}}, {name: "usturSecurityZone",docs: ["Ustur security zone"],type: {defined: "RiskZoneData"}}, {name: "highRiskZone",docs: ["High risk zone"],type: {defined: "RiskZoneData"}}, {name: "mediumRiskZone",docs: ["Medium risk zone"],type: {defined: "RiskZoneData"}}]}}, {name: "SagePointsCategory",docs: ["Represents a points category & modifier as defined in the Points program"],type: {kind: "struct",fields: [{name: "category",docs: ["The points category"],type: "publicKey"}, {name: "modifier",docs: ["The points category modifier"],type: "publicKey"}, {name: "modifierBump",docs: ["The points category modifier bump"],type: "u8"}]}}, {name: "ScanForSurveyDataUnitsInput",docs: ["Struct for data input to Scan For Survey Data Units"],type: {kind: "struct",fields: [{name: "keyIndex",docs: ["The index of the key in the player profile"],type: "u16"}]}}, {name: "SectorConnection",docs: ["Connection between sectors"],type: {kind: "struct",fields: [{name: "connectionSector",docs: ["The sector connected to"],type: "publicKey"}, {name: "subCoordinates",docs: ["The location of the connection"],type: {array: ["i64", 2]}}, {name: "flags",docs: ["Connection flags"],type: "u8"}]}}, {name: "SectorRing",docs: ["Represents the orbital position of a `Planet` in the `Sector`"],type: {kind: "enum",variants: [{name: "Inner"}, {name: "Mid"}, {name: "Outer"}]}}, {name: "ShipCounts",docs: ["Ship counts for a fleet."],type: {kind: "struct",fields: [{name: "total",docs: ["The total number of ships in the fleet."],type: "u32"}, {name: "updated",docs: ["Used when updating a fleet.", "Value is 0 when fleet update is in progress"],type: "u32"}, {name: "xxSmall",docs: ["The number of xx small ships in the fleet."],type: "u16"}, {name: "xSmall",docs: ["The number of x small ships in the fleet."],type: "u16"}, {name: "small",docs: ["The number of small ships in the fleet."],type: "u16"}, {name: "medium",docs: ["The number of medium ships in the fleet."],type: "u16"}, {name: "large",docs: ["The number of large ships in the fleet."],type: "u16"}, {name: "capital",docs: ["The number of capital ships in the fleet."],type: "u16"}, {name: "commander",docs: ["The number of commander ships in the fleet."],type: "u16"}, {name: "titan",docs: ["The number of titan ships in the fleet."],type: "u16"}]}}, {name: "ShipCountsUnpacked",docs: ["Unpacked version of [`ShipCounts`]"],type: {kind: "struct",fields: [{name: "total",docs: ["The total number of ships in the fleet."],type: "u32"}, {name: "updated",docs: ["Used when updating a fleet.", "Value is 0 when fleet update is in progress"],type: "u32"}, {name: "xxSmall",docs: ["The number of xx small ships in the fleet."],type: "u16"}, {name: "xSmall",docs: ["The number of x small ships in the fleet."],type: "u16"}, {name: "small",docs: ["The number of small ships in the fleet."],type: "u16"}, {name: "medium",docs: ["The number of medium ships in the fleet."],type: "u16"}, {name: "large",docs: ["The number of large ships in the fleet."],type: "u16"}, {name: "capital",docs: ["The number of capital ships in the fleet."],type: "u16"}, {name: "commander",docs: ["The number of commander ships in the fleet."],type: "u16"}, {name: "titan",docs: ["The number of titan ships in the fleet."],type: "u16"}]}}, {name: "ShipSizes",docs: ["Ship sizes."],type: {kind: "struct",fields: [{name: "xxSmall",docs: ["The size of xx small ships"],type: "u8"}, {name: "xSmall",docs: ["The size of x small ships"],type: "u8"}, {name: "small",docs: ["The size of small ships"],type: "u8"}, {name: "medium",docs: ["The size of medium ships"],type: "u8"}, {name: "large",docs: ["The size of large ships"],type: "u8"}, {name: "capital",docs: ["The size of capital ships"],type: "u8"}, {name: "commander",docs: ["The size of commander ships"],type: "u8"}, {name: "titan",docs: ["The size of titan ships"],type: "u8"}]}}, {name: "ShipStats",docs: ["A ship's stats"],type: {kind: "struct",fields: [{name: "movementStats",docs: ["Movement stats for the ship"],type: {defined: "MovementStats"}}, {name: "cargoStats",docs: ["Cargo stats for the ship"],type: {defined: "CargoStats"}}, {name: "miscStats",docs: ["Miscellaneous stats for the ship"],type: {defined: "MiscStats"}}]}}, {name: "ShipStatsUnpacked",docs: ["Unpacked version of [`ShipStats`]"],type: {kind: "struct",fields: [{name: "movementStats",docs: ["Movement stats for the ship"],type: {defined: "MovementStats"}}, {name: "cargoStats",docs: ["Cargo stats for the ship"],type: {defined: "CargoStats"}}, {name: "miscStats",docs: ["Miscellaneous stats for the ship"],type: {defined: "MiscStats"}}]}}, {name: "SizeClass",docs: ["Represents different types of Ships"],type: {kind: "enum",variants: [{name: "XxSmall"}, {name: "XSmall"}, {name: "Small"}, {name: "Medium"}, {name: "Large"}, {name: "Capital"}, {name: "Commander"}, {name: "Titan"}]}}, {name: "StarType",docs: ["Represents different types of Stars"],type: {kind: "enum",variants: [{name: "WhiteDwarf"}, {name: "RedDwarf"}, {name: "Solar"}, {name: "HotBlue"}, {name: "RedGiant"}]}}, {name: "StarbaseCreateCargoPodInput",docs: ["Struct for data input to `StarbaseCreateCargoPod`"],type: {kind: "struct",fields: [{name: "podSeeds",docs: ["cargo pod seeds"],type: {array: ["u8", 32]}}, {name: "keyIndex",docs: ["the index of the key in the player profile"],type: "u16"}]}}, {name: "StarbaseCreateCraftingProcessInput",docs: ["Struct for data input to create a `CraftingProcess`"],type: {kind: "struct",fields: [{name: "craftingId",docs: ["crafting id"],type: "u64"}, {name: "recipeCategoryIndex",docs: ["the index of the recipe's category"],type: "u16"}, {name: "quantity",docs: ["quantity of outputs to craft"],type: "u64"}, {name: "numCrew",docs: ["number of crew members to use for this crafting process"],type: "u64"}, {name: "keyIndex",docs: ["the index of the key in the player profile"],type: "u16"}]}}, {name: "StarbaseDepositCraftingIngredientInput",docs: ["Struct for data input to deposit an ingredient"],type: {kind: "struct",fields: [{name: "amount",docs: ["the amount of ingredient to deposit"],type: "u64"}, {name: "ingredientIndex",docs: ["the index of the recipe ingredient"],type: "u16"}, {name: "keyIndex",docs: ["the index of the key in the player profile"],type: "u16"}]}}, {name: "StarbaseLevelInfo",docs: ["Information associated with `Starbase` levels"],type: {kind: "struct",fields: [{name: "recipeForUpgrade",docs: ["The crafting recipe required to upgrade a `Starbase` to this level"],type: "publicKey"}, {name: "recipeCategoryForLevel",docs: ["The crafting recipe category enabled for crafting at a `Starbase` of this level."],type: "publicKey"}, {name: "hp",docs: ["The `Starbase` health points for this level."],type: "u64"}, {name: "sp",docs: ["The `Starbase` shield points for this level."],type: "u64"}, {name: "sectorRingAvailable",docs: ["The planet position `Ring` available for this level"],type: "u8"}, {name: "warpLaneMovementFee",docs: ['Fee charged for the "warp lane" movement type which is meant to be charged in ATLAS', "Since ATLAS has 8 decimal places, units are in the smallest value of ATLAS possible."],type: "u64"}]}}, {name: "StarbaseLevelInfoArrayInput",docs: ["Struct for data input to Update Starbase Level Settings"],type: {kind: "struct",fields: [{name: "level",docs: ["The level of the `Starbase`."],type: "u8"}, {name: "faction",docs: ["The `Starbase` faction."],type: "u8"}, {name: "hp",docs: ["The `Starbase` health points for this level."],type: "u64"}, {name: "sp",docs: ["The `Starbase` shield points for this level."],type: "u64"}, {name: "sectorRingAvailable",docs: ["The planet position `Ring` available for this level"],type: {defined: "SectorRing"}}, {name: "warpLaneMovementFee",docs: ['Fee charged for the "warp lane" movement type which is meant to be charged in ATLAS'],type: "u64"}]}}, {name: "StarbaseLoadingBay",docs: ["The data for the [`FleetStateData::StarbaseLoadingBay`] state"],type: {kind: "struct",fields: [{name: "starbase",docs: ["The `Starbase` is in the loading bay of"],type: "publicKey"}, {name: "lastUpdate",docs: ["The last time this fleet was updated"],type: "i64"}]}}, {name: "StarbaseRemoveCargoPodInput",docs: ["Struct for data input to `StarbaseRemoveCargoPod`"],type: {kind: "struct",fields: [{name: "keyIndex",docs: ["the index of the key in the player profile"],type: "u16"}]}}, {name: "StarbaseState",docs: ["The state of a `Starbase`."],type: {kind: "enum",variants: [{name: "Active"}, {name: "Destroyed"}]}}, {name: "StarbaseTransferCargoInput",docs: ["Struct for data input to `DepositCargoToGame`"],type: {kind: "struct",fields: [{name: "amount",docs: ["cargo amount"],type: "u64"}, {name: "keyIndex",docs: ["the index of the key in the player profile"],type: "u16"}]}}, {name: "StarbaseUpgradeState",docs: ["The state of a `Starbase`."],type: {kind: "enum",variants: [{name: "NotStarted"}, {name: "Started"}, {name: "Completed"}]}}, {name: "StarbaseUpkeepInfo",docs: ["Information associated with `Starbase` upkeep"],type: {kind: "struct",fields: [{name: "ammoReserve",docs: ["The maximum amount of ammo that can be committed upkeep by players", "If 0 (zero) then ammo upkeep is disabled"],type: "u64"}, {name: "ammoDepletionRate",docs: ["The per second rate at which the ammo reserve is emptied"],type: "u32"}, {name: "foodReserve",docs: ["The maximum amount of food that can be committed upkeep by players", "If 0 (zero) then food upkeep is disabled"],type: "u64"}, {name: "foodDepletionRate",docs: ["The per second rate at which the food reserve is emptied"],type: "u32"}, {name: "toolkitReserve",docs: ["The maximum amount of toolkits that can be committed upkeep by players", "If 0 (zero) then toolkit upkeep is disabled"],type: "u64"}, {name: "toolkitDepletionRate",docs: ["The per second rate at which the toolkit reserve is emptied"],type: "u32"}]}}, {name: "StarbaseUpkeepInfoArrayInput",docs: ["Struct for data input to Update Starbase Upkeep Settings"],type: {kind: "struct",fields: [{name: "level",docs: ["The level of the `Starbase`."],type: "u8"}, {name: "info",docs: ["The stats for the ship"],type: {defined: "StarbaseUpkeepInfoUnpacked"}}]}}, {name: "StarbaseUpkeepInfoUnpacked",docs: ["Unpacked version of [`StarbaseUpkeepInfo`]"],type: {kind: "struct",fields: [{name: "ammoReserve",docs: ["The maximum amount of ammo that can be committed upkeep by players", "If 0 (zero) then ammo upkeep is disabled"],type: "u64"}, {name: "ammoDepletionRate",docs: ["The per second rate at which the ammo reserve is emptied"],type: "u32"}, {name: "foodReserve",docs: ["The maximum amount of food that can be committed upkeep by players", "If 0 (zero) then food upkeep is disabled"],type: "u64"}, {name: "foodDepletionRate",docs: ["The per second rate at which the food reserve is emptied"],type: "u32"}, {name: "toolkitReserve",docs: ["The maximum amount of toolkits that can be committed upkeep by players", "If 0 (zero) then toolkit upkeep is disabled"],type: "u64"}, {name: "toolkitDepletionRate",docs: ["The per second rate at which the toolkit reserve is emptied"],type: "u32"}]}}, {name: "StarbaseUpkeepLevels",docs: ["Information on `Starbase` upkeep by level"],type: {kind: "struct",fields: [{name: "level0",docs: ["Upkeep info. for a level 0 `Starbase`"],type: {defined: "StarbaseUpkeepInfo"}}, {name: "level1",docs: ["Upkeep info. for a level 1 `Starbase`"],type: {defined: "StarbaseUpkeepInfo"}}, {name: "level2",docs: ["Upkeep info. for a level 2 `Starbase`"],type: {defined: "StarbaseUpkeepInfo"}}, {name: "level3",docs: ["Upkeep info. for a level 3 `Starbase`"],type: {defined: "StarbaseUpkeepInfo"}}, {name: "level4",docs: ["Upkeep info. for a level 4 `Starbase`"],type: {defined: "StarbaseUpkeepInfo"}}, {name: "level5",docs: ["Upkeep info. for a level 5 `Starbase`"],type: {defined: "StarbaseUpkeepInfo"}}, {name: "level6",docs: ["Upkeep info. for a level 6 `Starbase`"],type: {defined: "StarbaseUpkeepInfo"}}]}}, {name: "StarbaseUpkeepLevelsUnpacked",docs: ["Unpacked version of [`StarbaseUpkeepLevels`]"],type: {kind: "struct",fields: [{name: "level0",docs: ["Upkeep info. for a level 0 `Starbase`"],type: {defined: "StarbaseUpkeepInfo"}}, {name: "level1",docs: ["Upkeep info. for a level 1 `Starbase`"],type: {defined: "StarbaseUpkeepInfo"}}, {name: "level2",docs: ["Upkeep info. for a level 2 `Starbase`"],type: {defined: "StarbaseUpkeepInfo"}}, {name: "level3",docs: ["Upkeep info. for a level 3 `Starbase`"],type: {defined: "StarbaseUpkeepInfo"}}, {name: "level4",docs: ["Upkeep info. for a level 4 `Starbase`"],type: {defined: "StarbaseUpkeepInfo"}}, {name: "level5",docs: ["Upkeep info. for a level 5 `Starbase`"],type: {defined: "StarbaseUpkeepInfo"}}, {name: "level6",docs: ["Upkeep info. for a level 6 `Starbase`"],type: {defined: "StarbaseUpkeepInfo"}}]}}, {name: "StarbaseWithdrawCraftingIngredientInput",docs: ["Struct for data input to withdraw an ingredient"],type: {kind: "struct",fields: [{name: "amount",docs: ["the amount of ingredient to withdraw"],type: "u64"}, {name: "ingredientIndex",docs: ["the index of the recipe ingredient"],type: "u16"}, {name: "keyIndex",docs: ["the index of the key in the player profile"],type: "u16"}]}}, {name: "StartSubwarpInput",docs: ["Struct for data input to initialize an `SubwarpMovement`"],type: {kind: "struct",fields: [{name: "toSector",docs: ["The destination coordinates"],type: {array: ["i64", 2]}}, {name: "keyIndex",docs: ["The index of the key in the player profile"],type: "u16"}]}}, {name: "StopMiningAsteroidInput",docs: ["Struct for data input for `StopMiningAsteroidInput`"],type: {kind: "struct",fields: [{name: "keyIndex",docs: ["the index of the key in the player profile"],type: "u16"}]}}, {name: "StopSubwarpInput",docs: ["Struct for data input to stop an `SubwarpMovement`"],type: {kind: "struct",fields: [{name: "keyIndex",docs: ["The index of the key in the player profile"],type: "u16"}]}}, {name: "SubmitStarbaseUpgradeResourceInput",docs: ["Submit starbase upgrade resource inputs"],type: {kind: "struct",fields: [{name: "pointsProgramPermissionsKeyIndex",docs: ["the index of the point program permissions in the player profile"],type: "u16"}, {name: "sagePermissionsKeyIndex",docs: ["the index of the key in sage permissions in the player profile"],type: "u16"}, {name: "upgradeProcessRecipeInputIndex",docs: ["the index of the resource in the upgrade_process_recipe ingredients list", "The resource is a non-consumable in this recipe"],type: "u16"}, {name: "starbaseUpgradeRecipeInputIndex",docs: ["the index of the resource in the upgrade recipe", "The resource is a consumable in this recipe"],type: "u16"}, {name: "resourceRecipeOutputIndex",docs: ["the index of the resource represented by `token_mint` in the `resource_recipe` ingredients list", "The resource is an output in this recipe"],type: "u16"}, {name: "epochIndex",docs: ["the index of the epoch in the `RedemptionConfig` account"],type: "u16"}]}}, {name: "TransferCargoWithinFleetInput",docs: ["Struct for data input to `TransferCargoWithinFleet`"],type: {kind: "struct",fields: [{name: "amount",docs: ["cargo amount"],type: "u64"}, {name: "keyIndex",docs: ["the index of the key in the player profile"],type: "u16"}]}}, {name: "UpdateGameInput",docs: ["Struct for data input to Update instruction"],type: {kind: "struct",fields: [{name: "cargo",docs: ["Cargo settings"],type: "u8"}, {name: "crafting",docs: ["Crafting settings"],type: "u8"}, {name: "mints",docs: ["Mints"],type: "u8"}, {name: "vaults",docs: ["Vaults"],type: "u8"}, {name: "points",docs: ["Points settings"],type: "u8"}, {name: "riskZones",docs: ["Data for risk zones"],type: {option: {defined: "RiskZonesDataUnpacked"}}}, {name: "keyIndex",docs: ["the index of the key in the sector permissions profile"],type: "u16"}]}}, {name: "UpdateGameStateInput",docs: ["Struct for data input to Update instruction"],type: {kind: "struct",fields: [{name: "fleet",docs: ["Fleet settings"],type: {option: {defined: "FleetInput"}}}, {name: "misc",docs: ["Miscellaneous settings"],type: {option: {defined: "MiscVariablesInput"}}}, {name: "keyIndex",docs: ["the index of the key in the sector permissions profile"],type: "u16"}]}}, {name: "UpdateMineItemInput",docs: ["Struct for data input to Register a Resource"],type: {kind: "struct",fields: [{name: "name",docs: ["The name of the `MineItem`"],type: {option: {array: ["u8", 64]}}}, {name: "resourceHardness",docs: ["How hard it is to mine this item"],type: {option: "u16"}}, {name: "keyIndex",docs: ["the index of the key in the fleet permissions profile"],type: "u16"}]}}, {name: "UpdatePlanetInput",docs: ["Struct for data input to Update Planet"],type: {kind: "struct",fields: [{name: "name",docs: ["`Planet` name"],type: {option: {array: ["u8", 64]}}}, {name: "size",docs: ["`Planet` size"],type: {option: "u64"}}, {name: "maxHp",docs: ["`Planet` max_hp"],type: {option: "u64"}}, {name: "keyIndex",docs: ["the index of the key in the sector permissions profile"],type: "u16"}]}}, {name: "UpdateProgressionConfigInput",docs: ["Struct for data input for `UpdateProgressionConfig`"],type: {kind: "struct",fields: [{name: "keyIndex",docs: ["the index of the key in the permissions profile"],type: "u16"}, {name: "dailyLpLimit",docs: ["the daily limit for Loyalty Points (LP)"],type: {option: "u64"}}, {name: "dailyCouncilRankXpLimit",docs: ["the daily limit for Council Rank Experience Points (CRXP)"],type: {option: "u64"}}, {name: "dailyPilotXpLimit",docs: ["the daily limit for Pilot License Experience Points (PXP)"],type: {option: "u64"}}, {name: "dailyDataRunningXpLimit",docs: ["the daily limit for Data Running Experience Points (DRXP)"],type: {option: "u64"}}, {name: "dailyMiningXpLimit",docs: ["the daily limit for Mining Experience Points (MXP)"],type: {option: "u64"}}, {name: "dailyCraftingXpLimit",docs: ["the daily limit for Crafting Experience Points (CXP)"],type: {option: "u64"}}, {name: "items",docs: ["the progression items"],type: {option: {vec: {defined: "ProgressionItemInputUnpacked"}}}}]}}, {name: "UpdateResourceInput",docs: ["Struct for data input to Update Resource"],type: {kind: "struct",fields: [{name: "systemRichness",docs: ["`Resource` richness"],type: {option: "u16"}}, {name: "keyIndex",docs: ["the index of the key in the fleet permissions profile"],type: "u16"}]}}, {name: "UpdateShipEscrowInput",docs: ["Struct for data input for `UpdateShipEscrow`"],type: {kind: "struct",fields: [{name: "shipEscrowIndex",docs: ["Index of `WrappedShipEscrow` in remaining data of `StarbasePlayer`"],type: "u32"}]}}, {name: "UpdateShipFleetInput",docs: ["Struct for data input for that has `key_index`"],type: {kind: "struct",fields: [{name: "shipAmount",docs: ["Number of ships to add to the fleet"],type: "u16"}, {name: "fleetShipInfoIndex",docs: ["Index of `FleetShipsInfo` in remaining data of `FleetShips`"],type: "u32"}]}}, {name: "UpdateShipInput",docs: ["Struct for data input to Update Ship"],type: {kind: "struct",fields: [{name: "name",docs: ["The `Ship` name/label"],type: {array: ["u8", 64]}}, {name: "sizeClass",docs: ["the ship's size class"],type: {defined: "SizeClass"}}, {name: "stats",docs: ["The stats for the ship"],type: {defined: "ShipStatsUnpacked"}}, {name: "keyIndex",docs: ["the index of the key in the sector permissions profile"],type: "u16"}]}}, {name: "UpdateStarInput",docs: ["Struct for data input to Update Star"],type: {kind: "struct",fields: [{name: "name",docs: ["`Star` name"],type: {option: {array: ["u8", 64]}}}, {name: "size",docs: ["`Star` size"],type: {option: "u64"}}, {name: "starType",docs: ["`Star` type"],type: {option: "u8"}}, {name: "keyIndex",docs: ["the index of the key in the sector permissions profile"],type: "u16"}]}}, {name: "UpdateStarbaseInput",docs: ["Struct for data input to Update `Starbase`"],type: {kind: "struct",fields: [{name: "name",docs: ["`Starbase` name"],type: {option: {array: ["u8", 64]}}}, {name: "subCoordinates",docs: ["`Starbase` coordinates"],type: {option: {array: ["i64", 2]}}}, {name: "keyIndex",docs: ["the index of the key in the sector permissions profile"],type: "u16"}]}}, {name: "UpdateSurveyDataUnitTrackerInput",docs: ["Struct for data input to Update SurveyDataUnitTracker"],type: {kind: "struct",fields: [{name: "coordinatesRange",docs: ["The valid coordinates range", "e.g. a value of [-50, 50] means that coordinates from [-50, -50] to [50, 50] are valid for SDU scanning"],type: {option: {array: ["i64", 2]}}}, {name: "cssCoordinates",docs: ["The locations of the central space stations (CSS) of the three factions"],type: {option: {array: [{array: ["i64", 2]}, 3]}}}, {name: "originCoordinates",docs: ['The co-ordinates of the "origin"; used in calculating SDU probability'],type: {option: {array: ["i64", 2]}}}, {name: "cssMaxDistance",docs: ["The max distance from the nearest CSS; used in calculating SDU probability"],type: {option: "u32"}}, {name: "originMaxDistance",docs: ["The max distance from the `origin_coordinates`; used in calculating SDU probability"],type: {option: "u32"}}, {name: "distanceWeighting",docs: ["The distance weighting; used in calculating SDU probability"],type: {option: "u32"}}, {name: "tMax",docs: ["The maximum time before SDU probability at a location changes"],type: {option: "i64"}}, {name: "xMul",docs: ["Multiplier in the X dimension; used in noise function"],type: {option: "u32"}}, {name: "yMul",docs: ["Multiplier in the Y dimension; used in noise function"],type: {option: "u32"}}, {name: "zMul",docs: ["Multiplier in the Z dimension; used in noise function"],type: {option: "u32"}}, {name: "sduMaxPerSector",docs: ["The maximum number of SDUs that can be found per scan per sector"],type: {option: "u32"}}, {name: "scanChanceRegenPeriod",docs: ["The amount of time in seconds that it takes for a sector scan chance to fully regenerate"],type: {option: "i16"}}, {name: "keyIndex",docs: ["the index of the key in the sector permissions profile"],type: "u16"}]}}, {name: "UpkeepResourceType",docs: ["The different types of upkeep resources"],type: {kind: "enum",variants: [{name: "Ammo"}, {name: "Food"}, {name: "Toolkit"}]}}, {name: "Vaults",docs: ["Token vaults"],type: {kind: "struct",fields: [{name: "atlas",docs: ["ATLAS token mint"],type: "publicKey"}, {name: "polis",docs: ["POLIS token mint"],type: "publicKey"}]}}, {name: "WarpLaneInput",docs: ["Struct for data input to initialize a `WarpLane`"],type: {kind: "struct",fields: [{name: "keyIndex",docs: ["The index of the key in the player profile"],type: "u16"}, {name: "toSectorIndex",docs: ["Index of the to_sector in `SectorConnections` of the from_sector"],type: "u16"}, {name: "fromSectorIndex",docs: ["Index of the from_sector in `SectorConnections` of the to_sector"],type: "u16"}]}}, {name: "WarpToCoordinateInput",docs: ["Struct for data input to initialize a `WarpToCoordinate`"],type: {kind: "struct",fields: [{name: "keyIndex",docs: ["The index of the key in the player profile"],type: "u16"}, {name: "toSector",docs: ["The destination coordinates"],type: {array: ["i64", 2]}}]}}, {name: "WithdrawCargoFromFleetInput",docs: ["Struct for data input to `WithdrawCargoFromFleet`"],type: {kind: "struct",fields: [{name: "amount",docs: ["cargo amount"],type: "u64"}, {name: "keyIndex",docs: ["the index of the key in the player profile"],type: "u16"}]}}, {name: "WrappedShipEscrow",docs: ["Wrapped `Ship` escrow info"],type: {kind: "struct",fields: [{name: "ship",docs: ["The `Ship` account address"],type: "publicKey"}, {name: "amount",docs: ["The `Ship` token amount in escrow"],type: "u64"}, {name: "updateId",docs: ["The update id for the `Ship`"],type: "u64"}]}}],errors: [{code: 6e3,name: "IncorrectAdminAddress",msg: "Incorrect admin address."}, {code: 6001,name: "MissingRemainingAccount",msg: "An expected remaining account is missing."}, {code: 6002,name: "NoStargateConnectionsAvailable",msg: "No Stargate connections available."}, {code: 6003,name: "StargatesNotConnected",msg: "The provided Stargates are not connected."}, {code: 6004,name: "InvalidPlanetType",msg: "Invalid Planet Type."}, {code: 6005,name: "InvalidRingType",msg: "Invalid Ring Type."}, {code: 6006,name: "InvalidStarType",msg: "Invalid Star Type."}, {code: 6007,name: "InvalidOrInactiveGame",msg: "Invalid Or Inactive Game"}, {code: 6008,name: "InvalidShipSizeClass",msg: "Invalid Ship Size Class."}, {code: 6009,name: "IncorrectAccountSize",msg: "Incorrect Account Size."}, {code: 6010,name: "UpdateIdMismatch",msg: "The update_id is mismatched."}, {code: 6011,name: "AlreadyActive",msg: "The account is already active."}, {code: 6012,name: "InactiveAccount",msg: "The account is inactive."}, {code: 6013,name: "InvalidGame",msg: "The game account is invalid."}, {code: 6014,name: "InvalidGameState",msg: "The game state account is invalid."}, {code: 6015,name: "InvalidSector",msg: "The sector account is invalid."}, {code: 6016,name: "IncorrectVarsAccountAddress",msg: "Incorrect sage game_id account address."}, {code: 6017,name: "InsufficientFuel",msg: "Insufficient Fuel to complete movement"}, {code: 6018,name: "DistanceGreaterThanMax",msg: "Distance of movement is greater than the allowed maximum"}, {code: 6019,name: "NumericOverflow",msg: "Numeric overflow"}, {code: 6020,name: "InvalidLocationType",msg: "Invalid Location Type."}, {code: 6021,name: "LocationTypeNotSupported",msg: "The provided location type is not supported."}, {code: 6022,name: "IncorrectMineItem",msg: "Incorrect mine item address."}, {code: 6023,name: "IncorrectAuthorityAddress",msg: "Incorrect authority address."}, {code: 6024,name: "IncorrectResourceAddress",msg: "Incorrect resource address."}, {code: 6025,name: "IncorrectMintAuthority",msg: "Incorrect mint authority."}, {code: 6026,name: "MintAuthorityIsNone",msg: "The mint authority should exist."}, {code: 6027,name: "InvalidCurrentFleetState",msg: "The current fleet state is not valid."}, {code: 6028,name: "InvalidCurrentStarbaseState",msg: "The current starbase state is not valid."}, {code: 6029,name: "AuthorityMismatch",msg: "Authority mismatch"}, {code: 6030,name: "MintMismatch",msg: "Mint mismatch"}, {code: 6031,name: "TokenMismatch",msg: "Incorrect token address."}, {code: 6032,name: "OwnerMismatch",msg: "Owner mismatch"}, {code: 6033,name: "GameMismatch",msg: "Game ID mismatch"}, {code: 6034,name: "ProfileMismatch",msg: "Profile mismatch"}, {code: 6035,name: "SagePlayerProfileMismatch",msg: "SagePlayerProfile mismatch"}, {code: 6036,name: "StarbaseMismatch",msg: "Starbase mismatch"}, {code: 6037,name: "FactionMismatch",msg: "Faction mismatch"}, {code: 6038,name: "SeqIdMismatch",msg: "Sequence id mismatch"}, {code: 6039,name: "ShipMismatch",msg: "Ship mismatch"}, {code: 6040,name: "CargoPodMismatch",msg: "Cargo Pod mismatch"}, {code: 6041,name: "PlanetMismatch",msg: "Planet mismatch"}, {code: 6042,name: "MineItemMismatch",msg: "MineItem mismatch"}, {code: 6043,name: "LocationMismatch",msg: "Location mismatch"}, {code: 6044,name: "InvalidEscrowKey",msg: "Escrow key not found in remaining data"}, {code: 6045,name: "InvalidShipAmount",msg: "Insufficient Ship token amount"}, {code: 6046,name: "InvalidShipHangarSpaceAmount",msg: "Insufficient Ship hangar space amount"}, {code: 6047,name: "InvalidCrewAmount",msg: "Invalid crew amount"}, {code: 6048,name: "InvalidState",msg: "Invalid state"}, {code: 6049,name: "InvalidDistance",msg: "Invalid distance"}, {code: 6050,name: "NotAtCentralSpaceStation",msg: "Not at central space station"}, {code: 6051,name: "ShipNotExpected",msg: "The instruction does not expect a ship account"}, {code: 6052,name: "AddressMismatch",msg: "Address mismatch"}, {code: 6053,name: "InvalidSectorConnection",msg: "Invalid sector connection"}, {code: 6054,name: "InvalidStarbaseLevel",msg: "Invalid Starbase level"}, {code: 6055,name: "InvalidStarbaseUpgradeRecipeCategory",msg: "Invalid Starbase upgrade recipe category"}, {code: 6056,name: "HangarUpgradeNotPossible",msg: "Hangar upgrade not Possible"}, {code: 6057,name: "DisbandedFleetNotEmpty",msg: "Disbanded fleet not empty"}, {code: 6058,name: "FaultyMovement",msg: "Faulty movement"}, {code: 6059,name: "IncorrectHandleRawAccount",msg: "Incorrect Account Type for Handle Raw"}, {code: 6060,name: "InsufficientShipCargoCapacity",msg: "Insufficient Ship Cargo Capacity"}, {code: 6061,name: "FleetDoesNotNeedUpdate",msg: "Fleet does not need update"}, {code: 6062,name: "MustDisbandFleet",msg: "Must disband fleet"}, {code: 6063,name: "CannotForceDisbandFleet",msg: "Cannot force-disband fleet"}, {code: 6064,name: "ShipMismatchOrAlreadyUpdated",msg: "Ship mismatch or already updated"}, {code: 6065,name: "ShipAlreadyUpdated",msg: "Ship already updated"}, {code: 6066,name: "InvalidNextShipAddress",msg: "Invalid next ship address"}, {code: 6067,name: "InvalidShipForForcedDisband",msg: "Ship is not valid for forced disband of fleet"}, {code: 6068,name: "InvalidWarpRange",msg: "Warp range exceeded"}, {code: 6069,name: "InvalidIngredient",msg: "Invalid Ingredient"}, {code: 6070,name: "StarbaseUpgradeNotInProgress",msg: "Starbase Upgrade Not in progress"}, {code: 6071,name: "FleetNotInQueue",msg: "Fleet Not in queue"}, {code: 6072,name: "NeedCleanStarbaseUpgradeQueue",msg: "Need to clean Starbase upgrade queue"}, {code: 6073,name: "PlanetNotReachable",msg: "Planet Not Reachable"}, {code: 6074,name: "RespawnNotPossible",msg: "Respawn Not Possible"}, {code: 6075,name: "InvalidMovement",msg: "Cannot enter enemy faction's Security Zone"}, {code: 6076,name: "CargoAmountAboveZero",msg: "The Cargo Pod contains a non-zero amount of the Cargo Type"}, {code: 6077,name: "InvalidCargoPod",msg: "The Cargo Pod is invalid"}, {code: 6078,name: "InvalidZoneCoordinates",msg: "Invalid Zone Coordinates"}, {code: 6079,name: "RespawnTimeNotElapsed",msg: "Respawn time not elapsed"}, {code: 6080,name: "ActiveAccount",msg: "The Account is Active"}, {code: 6081,name: "StarbasePlayerMismatch",msg: "Starbase Player mismatch"}, {code: 6082,name: "AlreadyProcessed",msg: "The account has already been processed"}, {code: 6083,name: "InvalidAmount",msg: "The amount is invalid"}, {code: 6084,name: "WarpIsOnCooldown",msg: "Warp is on cooldown"}, {code: 6085,name: "ProgramMismatch",msg: "Program Mismatch"}, {code: 6086,name: "MustBeOnlyInstruction",msg: "Current Instruction Is Not Only Instruction"}, {code: 6087,name: "InvalidTime",msg: "Invalid Time"}, {code: 6088,name: "ScanIsOnCooldown",msg: "Scanning is on cooldown"}, {code: 6089,name: "InvalidFleetSize",msg: "Invalid Fleet Size"}, {code: 6090,name: "InactiveFeature",msg: "The feature is inactive"}, {code: 6091,name: "ZeroShipsAdded",msg: "Zero ships added to fleet"}, {code: 6092,name: "InvalidNoiseSeed",msg: "Invalid Noise Seed"}, {code: 6093,name: "InvalidType",msg: "Invalid type"}, {code: 6094,name: "RentedFleet",msg: "Rented Fleet"}, {code: 6095,name: "GenericInvalid",msg: "Generic invalid data"}]};
    const profileIDL = {version: "0.7.3",name: "player_profile",instructions: [{name: "acceptRoleInvitation",accounts: [{name: "newMember",isMut: !1,isSigner: !1,docs: ["The new member"]}, {name: "roleAccount",isMut: !0,isSigner: !1,docs: ["The role which the player is joining"]}, {name: "roleMembershipAccount",isMut: !0,isSigner: !1,docs: ["The role membership account for the new member"]}],args: [{name: "keyIndex",type: "u16"}, {name: "keyIndexInRoleAccount",type: "u16"}, {name: "keyIndexInMembershipAccount",type: "u16"}]}, {name: "addExistingMemberToRole",accounts: [{name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for reallocation."]}, {name: "newMember",isMut: !1,isSigner: !1,docs: ["The profile of the member to be added to the role"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The profile which the role belongs to."]}, {name: "roleMembershipAccount",isMut: !0,isSigner: !1,docs: ["The role membership account for the new member"]}, {name: "roleAccount",isMut: !0,isSigner: !1,docs: ["The role which the player is joining"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The system program"]}],args: [{name: "keyIndex",type: "u16"}, {name: "keyIndexInMembershipAccount",type: "u16"}]}, {name: "addKeys",accounts: [{name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the profile."]}, {name: "key",isMut: !1,isSigner: !0,docs: ["Key with [`ProfilePermissions::ADD_KEYS`] permission to add keys."]}, {name: "profile",isMut: !0,isSigner: !1,docs: ["The profile to add to"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The system program"]}],args: [{name: "keyAddIndex",type: "u16"}, {name: "keyPermissionsIndex",type: "u16"}, {name: "keysToAdd",type: {vec: {defined: "AddKeyInput"}}}]}, {name: "adjustAuth",accounts: [{name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the profile."]}, {name: "profile",isMut: !0,isSigner: !1,docs: ["The profile to create"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The system program"]}],args: [{name: "authIndexes",type: {vec: "u16"}}, {name: "newKeyPermissions",type: {vec: {defined: "AddKeyInput"}}}, {name: "removeRange",type: {array: ["u16", 2]}}, {name: "newKeyThreshold",type: "u8"}]}, {name: "createProfile",accounts: [{name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the new profile."]}, {name: "profile",isMut: !0,isSigner: !0,docs: ["The profile to create"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The system program"]}],args: [{name: "keyPermissions",type: {vec: {defined: "AddKeyInput"}}}, {name: "keyThreshold",type: "u8"}]}, {name: "createRole",accounts: [{name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the transaction"]}, {name: "profile",isMut: !0,isSigner: !1,docs: ["The [`Profile`] account that the role is being created for"]}, {name: "newRoleAccount",isMut: !0,isSigner: !1,docs: ["The role account being created"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The system program"]}],args: [{name: "keyIndex",type: "u16"}]}, {name: "inviteMemberToRole",accounts: [{name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the new profile."]}, {name: "newMember",isMut: !1,isSigner: !1,docs: ["The profile of the user to be added to the role"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The profile which the role belongs to."]}, {name: "roleMembershipAccount",isMut: !0,isSigner: !1,docs: ["The role membership account for the new member"]}, {name: "roleAccount",isMut: !0,isSigner: !1,docs: ["The role which the player is joining"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The system program"]}],args: [{name: "keyIndex",type: "u16"}]}, {name: "joinRole",accounts: [{name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the new profile."]}, {name: "newMember",isMut: !1,isSigner: !1,docs: ["The new member joining the role"]}, {name: "roleMembershipAccount",isMut: !0,isSigner: !1,docs: ["The role membership account for the new member"]}, {name: "roleAccount",isMut: !0,isSigner: !1,docs: ["The role which the player is joining"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The system program"]}],args: [{name: "keyIndex",type: "u16"}]}, {name: "leaveRole",accounts: [{name: "funder",isMut: !0,isSigner: !1,docs: ["The funder to receive the rent allocation."]}, {name: "member",isMut: !1,isSigner: !1,docs: ["The member leaving the role"]}, {name: "roleMembershipAccount",isMut: !0,isSigner: !1,docs: ["The role membership account for the member"]}, {name: "roleAccount",isMut: !0,isSigner: !1,docs: ["The role which the player is leaving"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The system program"]}],args: [{name: "keyIndex",type: "u16"}, {name: "keyIndexInRoleAccount",type: "u16"}, {name: "keyIndexInMembershipAccount",type: "u16"}]}, {name: "removeKeys",accounts: [{name: "funder",isMut: !0,isSigner: !1,docs: ["The funder for the profile."]}, {name: "key",isMut: !1,isSigner: !0,docs: ["Key with [`ProfilePermissions::REMOVE_KEYS`] permission to add keys."]}, {name: "profile",isMut: !0,isSigner: !1,docs: ["The profile to remove from"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The system program"]}],args: [{name: "keyIndex",type: "u16"}, {name: "keysToRemove",type: {array: ["u16", 2]}}]}, {name: "removeMemberFromRole",accounts: [{name: "funder",isMut: !0,isSigner: !1,docs: ["The funder to receive the rent allocation"]}, {name: "member",isMut: !1,isSigner: !1,docs: ["The profile of the user to be added to the role"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The profile which the role belongs to."]}, {name: "roleMembershipAccount",isMut: !0,isSigner: !1,docs: ["The role membership account for the member"]}, {name: "roleAccount",isMut: !0,isSigner: !1,docs: ["The role which the player is being removed from"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The system program"]}],args: [{name: "keyIndex",type: "u16"}, {name: "keyIndexInRoleAccount",type: "u16"}, {name: "keyIndexInMembershipAccount",type: "u16"}]}, {name: "removeRole",accounts: [{name: "funder",isMut: !0,isSigner: !1,docs: ["The funder for the transaction"]}, {name: "profile",isMut: !0,isSigner: !1,docs: ["The Profile that the role is being removed from"]}, {name: "roleAccount",isMut: !0,isSigner: !1,docs: ["The role being removed"]}, {name: "roleNameAccount",isMut: !0,isSigner: !1,docs: ["The role name account (if it exists)"]}],args: [{name: "roleNameBump",type: "u8"}, {name: "keyIndex",type: "u16"}]}, {name: "setName",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized to change the name."]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the name size change."]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The profile to set the name for."]}, {name: "name",isMut: !0,isSigner: !1,docs: ["The name account."]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The system program."]}],args: [{name: "keyIndex",type: "u16"}, {name: "name",type: "bytes"}]}, {name: "setRoleAcceptingMembers",accounts: [{name: "profile",isMut: !1,isSigner: !1,docs: ["The profile which owns the role being modified."]}, {name: "roleAccount",isMut: !0,isSigner: !1,docs: ["The role account to set as accepting members."]}],args: [{name: "keyIndex",type: "u16"}]}, {name: "setRoleAuthorizer",accounts: [{name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the name size change."]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The profile to set the name for."]}, {name: "roleAccount",isMut: !0,isSigner: !1,docs: ["The role account to set the authorizer for."]}, {name: "authorizer",isMut: !1,isSigner: !1,docs: ["The authorizer account to set."]}],args: [{name: "keyIndex",type: "u16"}]}, {name: "setRoleName",accounts: [{name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the name size change."]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The profile which the role belongs to"]}, {name: "role",isMut: !1,isSigner: !1,docs: ["The role to set the name for."]}, {name: "name",isMut: !0,isSigner: !1,docs: ["The name account."]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The system program."]}],args: [{name: "keyIndex",type: "u16"}, {name: "name",type: "bytes"}]}, {name: "setRoleNotAcceptingMembers",accounts: [{name: "profile",isMut: !1,isSigner: !1,docs: ["The profile which owns the role being modified."]}, {name: "roleAccount",isMut: !0,isSigner: !1,docs: ["The role account to set as not accepting members."]}],args: [{name: "keyIndex",type: "u16"}]}],accounts: [{name: "playerName",docs: ["Stores a players name on-chain."],type: {kind: "struct",fields: [{name: "version",docs: ["The data version of this account."],type: "u8"}, {name: "profile",docs: ["The profile this name is for."],type: "publicKey"}, {name: "bump",docs: ["The bump for this account."],type: "u8"}]}}, {name: "profile",docs: ["A player profile."],type: {kind: "struct",fields: [{name: "version",docs: ["The data version of this account."],type: "u8"}, {name: "authKeyCount",docs: ["The number of auth keys on the account"],type: "u16"}, {name: "keyThreshold",docs: ["The number of auth keys needed to update the profile."],type: "u8"}, {name: "nextSeqId",docs: ["The next sequence number for a new role."],type: "u64"}, {name: "createdAt",docs: ["When the profile was created."],type: "i64"}]}}, {name: "profileRoleMembership",docs: ["A players roles for a given profile", "Remaining data contains an unordered list of [`RoleMembership`](RoleMembership) structs"],type: {kind: "struct",fields: [{name: "version",docs: ["The data version of this account."],type: "u8"}, {name: "profile",docs: ["The Profile this belongs to"],type: "publicKey"}, {name: "member",docs: ["The members profile pubkey"],type: "publicKey"}, {name: "bump",docs: ["PDA bump"],type: "u8"}]}}, {name: "role",docs: ["A Role associated with a Profile. A Role contains an unordered list of Role Members in its", "remaining data which lists all of the members who carry this role."],type: {kind: "struct",fields: [{name: "version",docs: ["The data version of this account."],type: "u8"}, {name: "profile",docs: ["Profile that this role belongs to"],type: "publicKey"}, {name: "authorizer",docs: ["Origin authority of the account"],type: "publicKey"}, {name: "roleSeqId",docs: ["Roles seq_id"],type: "u64"}, {name: "acceptingNewMembers",docs: ["Is role accepting new members"],type: "u8"}, {name: "bump",docs: ["The name of the rank", "TODO: Add instruction to use `player-name` as the label", "PDA bump"],type: "u8"}]}}],types: [{name: "AddKeyInput",docs: ["Struct for adding a key"],type: {kind: "struct",fields: [{name: "scope",docs: ["The block of permissions"],type: "publicKey"}, {name: "expireTime",docs: ["The expire time of the key to add"],type: "i64"}, {name: "permissions",docs: ["The permissions for the key"],type: {array: ["u8", 8]}}]}}, {name: "MemberStatus",docs: ["Represents potential membership statuses for a player with a role"],type: {kind: "enum",variants: [{name: "Inactive"}, {name: "Active"}]}}, {name: "ProfileKey",docs: ["A key on a profile."],type: {kind: "struct",fields: [{name: "key",docs: ["The key."],type: "publicKey"}, {name: "scope",docs: ["The key for the permissions."],type: "publicKey"}, {name: "expireTime",docs: ["The expire time for this key.", "If `<0` does not expire."],type: "i64"}, {name: "permissions",docs: ["The permissions for the key."],type: {array: ["u8", 8]}}]}}, {name: "RoleMembership",docs: ["Represents a members status in a role"],type: {kind: "struct",fields: [{name: "key",docs: ["The member or role key associated with this membership"],type: "publicKey"}, {name: "status",docs: ["The members role status"],type: "u8"}]}}],errors: [{code: 6e3,name: "KeyIndexOutOfBounds",msg: "Key index out of bounds"}, {code: 6001,name: "ProfileMismatch",msg: "Profile did not match profile key"}, {code: 6002,name: "KeyMismatch",msg: "Key did not match profile key"}, {code: 6003,name: "ScopeMismatch",msg: "Scope did not match profile scope"}, {code: 6004,name: "KeyExpired",msg: "Key expired"}, {code: 6005,name: "KeyMissingPermissions",msg: "Key is missing permissions"}, {code: 6006,name: "PermissionsMismatch",msg: "Permissions dont match available"}, {code: 6007,name: "AuthKeyCannotExpire",msg: "Auth keys cannot expire"}, {code: 6008,name: "AuthKeyMustSign",msg: "New auth keys must be signers"}, {code: 6009,name: "DuplicateAuthKey",msg: "Duplicate key when adjusting auth keys"}, {code: 6010,name: "RoleAuthorityAlreadySet",msg: "Role authority has already been set"}, {code: 6011,name: "RoleNotAcceptingMembers",msg: "Role is not accepting new members"}, {code: 6012,name: "RoleMembershipMismatch",msg: "Role membership is not as expected"}, {code: 6013,name: "RoleLimitExceeded",msg: "Role limit exceeded"}, {code: 6014,name: "RoleHasMembers",msg: "Cannot remove role with members"}, {code: 6015,name: "FeatureNotImplemented",msg: "This feature is not yet support"}]};
    const cargoIDL = {version: "0.1.0",name: "cargo2",docs: ["The `cargo2` program"],instructions: [{name: "addCargo",docs: ["Adds cargo to a [`CargoPod`](state::CargoPod).", "Requires the authority to sign."],accounts: [{name: "authority",isMut: !1,isSigner: !0,docs: ["Authority for the cargo pod"]}, {name: "signerOriginAccount",isMut: !1,isSigner: !0,docs: ["Signer for Cargo Token Transfer"]}, {name: "statsDefinition",isMut: !1,isSigner: !1,docs: ["The [CargoStatsDefinition] for the cargo type"]}, {name: "cargoPod",isMut: !0,isSigner: !1,docs: ["The [CargoPod] Account"]}, {name: "cargoType",isMut: !1,isSigner: !1,docs: ["The Cargo Type Account"]}, {name: "originTokenAccount",isMut: !0,isSigner: !1,docs: ["The Origin Token Account"]}, {name: "cargoTokenAccount",isMut: !0,isSigner: !1,docs: ["The Cargo Token Account"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["Token Program"]}],args: [{name: "cargoAmount",type: "u64"}]}, {name: "closeCargoPod",docs: ["Closes the [`CargoPod`](state::CargoPod) if it has no open token accounts.", "Requires the authority to sign."],accounts: [{name: "funder",isMut: !0,isSigner: !1,docs: ["The account to return the rent"]}, {name: "authority",isMut: !1,isSigner: !0,docs: ["The authority for the pod account"]}, {name: "cargoPod",isMut: !0,isSigner: !1,docs: ["The [CargoPod] Account"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The system Program"]}],args: []}, {name: "closeTokenAccount",docs: ["Closes and burns any excess tokens in a given token account within a [`CargoPod`](state::CargoPod).", "Requires the authority to sign."],accounts: [{name: "funder",isMut: !0,isSigner: !1,docs: ["The account to return the rent"]}, {name: "authority",isMut: !1,isSigner: !0,docs: ["The authority for [CargoPod] account"]}, {name: "cargoPod",isMut: !0,isSigner: !1,docs: ["The [CargoPod] account"]}, {name: "cargoTokenAccount",isMut: !0,isSigner: !1,docs: ["The Cargo Token Account"]}, {name: "cargoType",isMut: !1,isSigner: !1,docs: ["The Cargo Type Account", "If the token account belongs to a registered Cargo Type then this account must be a valid Cargo Type", "However, to allow closing token account that are not valid Cargo this is an unchecked account"]}, {name: "mint",isMut: !0,isSigner: !1,docs: ["The Token Mint"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["The Token Program"]}],args: []}, {name: "consumeCargo",docs: ["Consumes cargo from a [`CargoPod`](state::CargoPod), burning the amount.", "Requires the authority to sign."],accounts: [{name: "authority",isMut: !1,isSigner: !0,docs: ["Authority for the cargo pod"]}, {name: "statsDefinition",isMut: !1,isSigner: !1,docs: ["The [CargoStatsDefinition] for the cargo type"]}, {name: "cargoPod",isMut: !0,isSigner: !1,docs: ["The [CargoPod] Account"]}, {name: "cargoType",isMut: !1,isSigner: !1,docs: ["The Cargo Type Account"]}, {name: "cargoTokenAccount",isMut: !0,isSigner: !1,docs: ["The Cargo Token Account"]}, {name: "tokenMint",isMut: !0,isSigner: !1,docs: ["Token Mint"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["Token Program"]}],args: [{name: "cargoAmount",type: "u64"}]}, {name: "initCargoPod",docs: ["Inits a new [`CargoPod`](state::CargoPod) account for the given [`CargoStatsDefinition`](state::CargoStatsDefinition) and authority."],accounts: [{name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the new cargo pod"]}, {name: "authority",isMut: !1,isSigner: !0,docs: ["The authority for the new cargo pod"]}, {name: "cargoPod",isMut: !0,isSigner: !1,docs: ["The new cargo pod"]}, {name: "statsDefinition",isMut: !1,isSigner: !1,docs: ["The definition of tracked stats"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The system program"]}],args: [{name: "podSeeds",type: {array: ["u8", 32]}}]}, {name: "initCargoType",docs: ["Inits a new [`CargoType`](state::CargoType) account for the given [`CargoStatsDefinition`](state::CargoStatsDefinition)."],accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The cargo permissions [`Profile`].", "Is going to act as the authority for the new definition."]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the cargo type"]}, {name: "mint",isMut: !1,isSigner: !1,docs: ["The mint for the new cargo type"]}, {name: "statsDefinition",isMut: !1,isSigner: !1,docs: ["The definition for the cargo type"]}, {name: "cargoType",isMut: !0,isSigner: !1,docs: ["The cargo type to init"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The system program"]}],args: [{name: "input",type: {defined: "InitCargoTypeInput"}}]}, {name: "initCargoTypeForNextSeqId",docs: ["Creates a new cargo type for the next `seq_id`."],accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The cargo permissions [`Profile`].", "Is going to act as the authority for the new definition."]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the cargo type"]}, {name: "mint",isMut: !1,isSigner: !1,docs: ["The mint for the new cargo type"]}, {name: "statsDefinition",isMut: !1,isSigner: !1,docs: ["The definition for the cargo type"]}, {name: "cargoType",isMut: !0,isSigner: !1,docs: ["The cargo type to init"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The system program"]}],args: [{name: "input",type: {defined: "InitCargoTypeInput"}}]}, {name: "initCargoTypeFromOldCargoType",docs: ["Creates a new cargo type for the next `seq_id` from a given cargo type."],accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The cargo permissions [`Profile`].", "Is going to act as the authority for the new definition."]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the cargo type"]}, {name: "statsDefinition",isMut: !1,isSigner: !1,docs: ["The definition for the cargo type"]}, {name: "oldCargoType",isMut: !1,isSigner: !1,docs: ["The old Cargo Type Account"]}, {name: "cargoType",isMut: !0,isSigner: !1,docs: ["The cargo type to init"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The system program"]}],args: [{name: "input",type: {defined: "InitCargoTypeFromOldCargoTypeInput"}}]}, {name: "initDefinition",docs: ["Inits a [`CargoStatsDefinition`](state::CargoStatsDefinition) account."],accounts: [{name: "profile",isMut: !1,isSigner: !1,docs: ["The cargo permissions [`Profile`](Profile).", "Is going to act as the authority for the new definition."]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the new definition"]}, {name: "statsDefinition",isMut: !0,isSigner: !0,docs: ["The new definition"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The system program"]}],args: [{name: "input",type: {defined: "InitDefinitionInput"}}]}, {name: "legitimizeCargo",docs: ["Legitimizes cargo in a [`CargoPod`](state::CargoPod) that was added outside of [`add_cargo`] or other cargo ix.", "Requires the authority to sign."],accounts: [{name: "authority",isMut: !1,isSigner: !0,docs: ["Authority for the cargo pod"]}, {name: "statsDefinition",isMut: !1,isSigner: !1,docs: ["The [CargoStatsDefinition] for the cargo type"]}, {name: "cargoPod",isMut: !0,isSigner: !1,docs: ["The [CargoPod] Account"]}, {name: "cargoType",isMut: !1,isSigner: !1,docs: ["The Cargo Type Account"]}, {name: "cargoTokenAccount",isMut: !0,isSigner: !1,docs: ["The Cargo Token Account"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["Token Program"]}],args: [{name: "cargoAmount",type: "u64"}]}, {name: "mintTo",docs: ["Mints tokens directly to a [`CargoPod`](state::CargoPod).", "Requires the authority to sign."],accounts: [{name: "authority",isMut: !1,isSigner: !0,docs: ["Authority for the [`CargoPod`] Account"]}, {name: "mintAuthority",isMut: !1,isSigner: !0,docs: ["The mint Authority"]}, {name: "statsDefinition",isMut: !1,isSigner: !1,docs: ["The [CargoStatsDefinition] for the cargo type"]}, {name: "cargoPod",isMut: !0,isSigner: !1,docs: ["The [`CargoPod`] Account"]}, {name: "cargoType",isMut: !1,isSigner: !1,docs: ["The [`CargoType`] Account"]}, {name: "cargoTokenAccount",isMut: !0,isSigner: !1,docs: ["The Cargo Token Account"]}, {name: "tokenMint",isMut: !0,isSigner: !1,docs: ["The Cargo token mint"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["Token Program"]}],args: [{name: "mintAmount",type: "u64"}]}, {name: "removeCargo",docs: ["Removes cargo from a [`CargoPod`](state::CargoPod) to a given token account.", "Requires the authority to sign."],accounts: [{name: "authority",isMut: !1,isSigner: !0,docs: ["Authority for the cargo pod"]}, {name: "statsDefinition",isMut: !1,isSigner: !1,docs: ["The [CargoStatsDefinition] for the cargo type"]}, {name: "cargoPod",isMut: !0,isSigner: !1,docs: ["The [CargoPod] Account"]}, {name: "cargoType",isMut: !1,isSigner: !1,docs: ["The Cargo Type Account"]}, {name: "destinationTokenAccount",isMut: !0,isSigner: !1,docs: ["The Destination Token Account"]}, {name: "cargoTokenAccount",isMut: !0,isSigner: !1,docs: ["Cargo Token Account"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["Token Program"]}],args: [{name: "cargoAmount",type: "u64"}]}, {name: "transferAuthority",docs: ["Transfers authority of a [`CargoPod`](state::CargoPod) to a new authority.", "Requires both authorities to sign."],accounts: [{name: "originPodAuthority",isMut: !1,isSigner: !0,docs: ["Authority for the cargo pod"]}, {name: "newPodAuthority",isMut: !1,isSigner: !0,docs: ["New authority for the cargo pod"]}, {name: "cargoPod",isMut: !0,isSigner: !1,docs: ["The [CargoPod] Account"]}],args: []}, {name: "transferCargo",docs: ["Transfers cargo between [`CargoPod`](state::CargoPod)s.", "Requires both authorities to sign."],accounts: [{name: "originPodAuthority",isMut: !1,isSigner: !0,docs: ["Authority for the origin cargo pod"]}, {name: "destinationPodAuthority",isMut: !1,isSigner: !0,docs: ["Authority for the destination cargo pod"]}, {name: "statsDefinition",isMut: !1,isSigner: !1,docs: ["The [CargoStatsDefinition] for the cargo type"]}, {name: "originCargoPod",isMut: !0,isSigner: !1,docs: ["The Origin [CargoPod] Account"]}, {name: "destinationCargoPod",isMut: !0,isSigner: !1,docs: ["The Destination [CargoPod] Account"]}, {name: "cargoType",isMut: !1,isSigner: !1,docs: ["The Cargo Type Account"]}, {name: "originTokenAccount",isMut: !0,isSigner: !1,docs: ["The Origin Token Account"]}, {name: "destinationTokenAccount",isMut: !0,isSigner: !1,docs: ["The Destination Token Account"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["Token Program"]}],args: [{name: "cargoAmount",type: "u64"}]}, {name: "updateCargoPod",docs: ["Updates a [`CargoPod`](state::CargoPod) account to have the newest sequence id from the [`CargoDefinition`](state::CargoStatsDefinition).", "This is the first step to update a [`CargoPod`](state::CargoPod) to a new [`CargoStatsDefinition`](state::CargoStatsDefinition).", "Permissionless function."],accounts: [{name: "cargoPod",isMut: !0,isSigner: !1,docs: ["The cargo pod to update"]}, {name: "statsDefinition",isMut: !1,isSigner: !1,docs: ["The definition of tracked stats"]}],args: []}, {name: "updateDefinition",docs: ["Updates a [`CargoStatsDefinition`](state::CargoStatsDefinition) account.", "Will advance the `seq_id` unless `rollback` is set to true."],accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The cargo permissions [`Profile`](Profile).", "Is going to act as the authority for the new definition."]}, {name: "statsDefinition",isMut: !0,isSigner: !1,docs: ["The [CargoStatsDefinition]"]}],args: [{name: "input",type: {defined: "UpdateDefinitionInput"}}]}, {name: "updatePodTokenAccount",docs: ["Updates a [`CargoPod`](state::CargoPod)s token account to have the same sequence id as the [`CargoPod`](state::CargoPod).", "This must be called after [`update_cargo_pod`].", "Permissionless function."],accounts: [{name: "statsDefinition",isMut: !1,isSigner: !1,docs: ["The [CargoStatsDefinition] for the cargo type"]}, {name: "cargoPod",isMut: !0,isSigner: !1,docs: ["The [CargoPod] Account"]}, {name: "oldCargoType",isMut: !1,isSigner: !1,docs: ["The previous version(`seq_id`) Cargo Type"]}, {name: "cargoType",isMut: !1,isSigner: !1,docs: ["The updated Cargo Type Account"]}, {name: "cargoTokenAccount",isMut: !0,isSigner: !1,docs: ["The Cargo Token Account"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["Token Program"]}],args: []}],accounts: [{name: "cargoPod",docs: ["A pod that can store any number of resources and tracks stats given a definition."],type: {kind: "struct",fields: [{name: "version",docs: ["The data version of this account."],type: "u8"}, {name: "statsDefinition",docs: ["The definition of tracked stats."],type: "publicKey"}, {name: "authority",docs: ["The authority for this pod."],type: "publicKey"}, {name: "openTokenAccounts",docs: ["The number of open token accounts in this pod."],type: "u8"}, {name: "podSeeds",docs: ["The seeds of the signer for this pod."],type: {array: ["u8", 32]}}, {name: "podBump",docs: ["The bump of the signer for this pod."],type: "u8"}, {name: "seqId",docs: ["The sequence id for the definition"],type: "u16"}, {name: "unupdatedTokenAccounts",docs: ["The number of unupdated token accounts in this pod. If this is greater than zero means the pod is frozen and only can withdraw cargo but not deposit."],type: "u8"}]}}, {name: "cargoStatsDefinition",docs: ["A definition of cargo stats."],type: {kind: "struct",fields: [{name: "version",docs: ["The data version of this account."],type: "u8"}, {name: "authority",docs: ["The authority for this definition."],type: "publicKey"}, {name: "defaultCargoType",docs: ["The default cargo type. System program (all 0s) if none."],type: "publicKey"}, {name: "statsCount",docs: ["The number of stats in this definition."],type: "u16"}, {name: "seqId",docs: ["The sequence id for the definition"],type: "u16"}]}}, {name: "cargoType",docs: ["The stats for a given cargo type (token mint)."],type: {kind: "struct",fields: [{name: "version",docs: ["The data version of this account."],type: "u8"}, {name: "statsDefinition",docs: ["The definition this follows"],type: "publicKey"}, {name: "mint",docs: ["The mint the cargo type is for"],type: "publicKey"}, {name: "creator",docs: ["TODO: remove this"],type: "publicKey"}, {name: "bump",docs: ["The bump for this account"],type: "u8"}, {name: "statsCount",docs: ["The number of stats in this definition."],type: "u16"}, {name: "seqId",docs: ["The sequence id for the definition"],type: "u16"}]}}],types: [{name: "InitCargoTypeFromOldCargoTypeInput",docs: ["Struct for data input for this IX"],type: {kind: "struct",fields: [{name: "keyIndex",docs: ["the index of the key in the cargo permissions profile"],type: "u16"}, {name: "newValues",docs: ["vector with values for all stats tracked by the definition"],type: {option: {vec: "u64"}}}]}}, {name: "InitCargoTypeInput",docs: ["Struct for data input for this IX"],type: {kind: "struct",fields: [{name: "keyIndex",docs: ["the index of the key in the cargo permissions profile"],type: "u16"}, {name: "values",docs: ["vector with values for all stats tracked by the definition"],type: {vec: "u64"}}]}}, {name: "InitDefinitionInput",docs: ["Struct for data input for [`InitDefinition`]"],type: {kind: "struct",fields: [{name: "cargoStats",docs: ["the count of stats the definition has"],type: "u16"}]}}, {name: "UpdateDefinitionInput",docs: ["Struct for data input for this IX"],type: {kind: "struct",fields: [{name: "keyIndex",docs: ["the index of the key in the cargo permissions profile"],type: "u16"}, {name: "rollback",docs: ["flag that if present means we need to decrease the definition seq_id"],type: {option: "bool"}}]}}],errors: [{code: 6e3,name: "StatOutOfBounds",msg: "A given stat was out of bounds"}, {code: 6001,name: "TooManyStats",msg: "There are too many stats"}, {code: 6002,name: "InvalidRentFunder",msg: "Rent funder was not owned by the system program or this program"}, {code: 6003,name: "TooFewStats",msg: "Popped a stat when there are no stats left"}, {code: 6004,name: "MissingSystemProgram",msg: "System program is missing when needed"}, {code: 6005,name: "InvalidCargoStat",msg: "Cargo stat data was invalid"}, {code: 6006,name: "InvalidCargoStatSize",msg: "Cargo stat size data was invalid"}, {code: 6007,name: "InvalidCargoType",msg: "Cargo type is invalid"}, {code: 6008,name: "WrongNumberOfDefinitions",msg: "Wrong number of definitions provided to init a cargo type"}, {code: 6009,name: "InvalidValueForStat",msg: "Invalid value provided for stat"}, {code: 6010,name: "NumericOverflow",msg: "Math overflow"}, {code: 6011,name: "AuthorityMismatch",msg: "Authority mismatch"}, {code: 6012,name: "StatsDefinitionMismatch",msg: "Stats definition mismatch"}, {code: 6013,name: "MintMismatch",msg: "Mint mismatch"}, {code: 6014,name: "OwnerMismatch",msg: "Owner mismatch"}, {code: 6015,name: "InvalidDelegation",msg: "Delegated amount is invalid"}, {code: 6016,name: "FrozenPod",msg: "The pod is frozen"}, {code: 6017,name: "UnupdatedCargoPodAccount",msg: "Unupdated CargoPod Account"}, {code: 6018,name: "InvalidSeqId",msg: "Invalid seq_id"}, {code: 6019,name: "UnupdatedTokenAccount",msg: "Unupdated token account"}, {code: 6020,name: "OpenTokenAccounts",msg: "Cargo Pod has token accounts open"}, {code: 6021,name: "NonZeroDelegation",msg: "Non Zero Delegated Amount"}, {code: 6022,name: "InvalidPreviousType",msg: "Invalid previous cargo_type account"}, {code: 6023,name: "InsufficientCargoAmount",msg: "Insufficient cargo amount"}, {code: 6024,name: "InsufficientTokenAmount",msg: "Insufficient token amount"}, {code: 6025,name: "PodTokenAccountAlreadyUpdated",msg: "Pod Token Account Already Updated"}]};
    const profileFactionIDL = {version: "0.7.1",name: "profile_faction",instructions: [{name: "chooseFaction",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key with auth permissions."]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder for the transaction."]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The profile to change faction for."]}, {name: "faction",isMut: !0,isSigner: !1,docs: ["The faction to change to."]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The system program."]}],args: [{name: "keyIndex",type: "u16"}, {name: "faction",type: {defined: "Faction"}}]}],accounts: [{name: "profileFactionAccount",docs: ["Stores a profiles enlisted faction on-chain."],type: {kind: "struct",fields: [{name: "version",docs: ["The data version of this account."],type: "u8"}, {name: "profile",docs: ["The profile this faction enlistment is for."],type: "publicKey"}, {name: "faction",docs: ["The faction of the profile."],type: "u8"}, {name: "bump",docs: ["The bump for this account."],type: "u8"}]}}],types: [{name: "Faction",docs: ["A faction that a player can belong to."],type: {kind: "enum",variants: [{name: "Unaligned"}, {name: "MUD"}, {name: "ONI"}, {name: "Ustur"}]}}]};
    //const pointsIDL = await BrowserAnchor.anchor.Program.fetchIdl(pointsProgramId, anchorProvider);
    const pointsIDL = JSON.parse('{"version": "0.1.0","name": "points","instructions": [{"name": "addPointCategoryLevel","accounts": [{"name": "key","isMut": false,"isSigner": true,"docs": ["The key authorized for this instruction"]},{"name": "profile","isMut": false,"isSigner": false,"docs": ["The points permissions [`Profile`](Profile)"]},{"name": "funder","isMut": true,"isSigner": true,"docs": ["The funder - pays for account rent"]},{"name": "category","isMut": true,"isSigner": false,"docs": ["The [PointCategory]"]},{"name": "systemProgram","isMut": false,"isSigner": false,"docs": ["The system program"]}],"args": [{"name": "input","type": {"defined": "AddPointCategoryLevelInput"}}]},{"name": "createPointCategory","accounts": [{"name": "profile","isMut": false,"isSigner": false,"docs": ["The points permissions [`Profile`](player_profile::state::Profile)"]},{"name": "funder","isMut": true,"isSigner": true,"docs": ["The funder - pays for account rent"]},{"name": "category","isMut": true,"isSigner": true,"docs": ["The [`PointCategory`]"]},{"name": "systemProgram","isMut": false,"isSigner": false,"docs": ["The Solana System program"]}],"args": [{"name": "input","type": {"defined": "CreatePointCategoryInput"}}]},{"name": "createUserPointAccount","accounts": [{"name": "userProfile","isMut": false,"isSigner": false,"docs": ["The user [`Profile`](Profile)"]},{"name": "funder","isMut": true,"isSigner": true,"docs": ["The funder - pays for account rent"]},{"name": "pointCategoryAccount","isMut": false,"isSigner": false,"docs": ["The [`PointCategory`]"]},{"name": "userPointsAccount","isMut": true,"isSigner": false,"docs": ["The [`UserPointsAccount`]"]},{"name": "systemProgram","isMut": false,"isSigner": false,"docs": ["The Solana System Program"]}],"args": []},{"name": "createUserPointAccountWithLicense","accounts": [{"name": "userProfile","isMut": false,"isSigner": false,"docs": ["The user [`Profile`](Profile)"]},{"name": "funder","isMut": true,"isSigner": true,"docs": ["The funder - pays for account rent"]},{"name": "category","isMut": false,"isSigner": false,"docs": ["The [`PointCategory`]"]},{"name": "userPointsAccount","isMut": true,"isSigner": false,"docs": ["The [`UserPointsAccount`]"]},{"name": "tokenAccountOwner","isMut": false,"isSigner": true,"docs": ["The owner of [`user_token_account`](CreateUserPointAccountWithLicense::user_token_account)"]},{"name": "userTokenAccount","isMut": true,"isSigner": false,"docs": ["The token account for the license to burn"]},{"name": "mintOrVault","isMut": true,"isSigner": false,"docs": ["The mint address for the license to burn"]},{"name": "tokenProgram","isMut": false,"isSigner": false,"docs": ["The Solana Token Program"]},{"name": "systemProgram","isMut": false,"isSigner": false,"docs": ["The Solana System Program"]}],"args": []},{"name": "decrementLevel","accounts": [{"name": "category","isMut": false,"isSigner": false,"docs": ["The [`PointCategory`]"]},{"name": "userPointsAccount","isMut": true,"isSigner": false,"docs": ["The [`UserPointsAccount`]"]},{"name": "pointsModifierAccount","isMut": false,"isSigner": true,"docs": ["The [`PointsModifier`] account"]}],"args": [{"name": "input","type": {"defined": "DecrementLevelInput"}}]},{"name": "decrementPoints","accounts": [{"name": "category","isMut": false,"isSigner": false,"docs": ["The [`PointCategory`]"]},{"name": "userPointsAccount","isMut": true,"isSigner": false,"docs": ["The [`UserPointsAccount`]"]},{"name": "pointsModifierAccount","isMut": false,"isSigner": true,"docs": ["The [`PointsModifier`] account"]}],"args": [{"name": "pointsAmount","type": "u64"}]},{"name": "deregisterPointModifier","accounts": [{"name": "key","isMut": false,"isSigner": true,"docs": ["The key authorized for this instruction"]},{"name": "profile","isMut": false,"isSigner": false,"docs": ["The points permissions [`Profile`](Profile)"]},{"name": "fundsTo","isMut": true,"isSigner": false,"docs": ["The funds_to - receives rent refund"]},{"name": "category","isMut": false,"isSigner": false,"docs": ["The [`PointCategory`]"]},{"name": "pointsModifierAccount","isMut": true,"isSigner": false,"docs": ["The [`PointsModifier`] account"]},{"name": "systemProgram","isMut": false,"isSigner": false,"docs": ["The Solana System program"]}],"args": [{"name": "keyIndex","type": "u16"}]},{"name": "incrementLevel","accounts": [{"name": "category","isMut": false,"isSigner": false,"docs": ["The [`PointCategory`]"]},{"name": "userPointsAccount","isMut": true,"isSigner": false,"docs": ["The [`UserPointsAccount`]"]},{"name": "pointsModifierAccount","isMut": false,"isSigner": true,"docs": ["The [`PointsModifier`] account"]}],"args": [{"name": "input","type": {"defined": "IncrementLevelInput"}}]},{"name": "incrementLevelBeyondThreshold","accounts": [{"name": "category","isMut": false,"isSigner": false,"docs": ["The [`PointCategory`]"]},{"name": "userPointsAccount","isMut": true,"isSigner": false,"docs": ["The [`UserPointsAccount`]"]},{"name": "pointsModifierAccount","isMut": false,"isSigner": true,"docs": ["The [`PointsModifier`] account"]}],"args": []},{"name": "incrementPoints","accounts": [{"name": "category","isMut": false,"isSigner": false,"docs": ["The [`PointCategory`]"]},{"name": "userPointsAccount","isMut": true,"isSigner": false,"docs": ["The [`UserPointsAccount`]"]},{"name": "pointsModifierAccount","isMut": false,"isSigner": true,"docs": ["The [`PointsModifier`] account"]}],"args": [{"name": "input","type": {"defined": "IncrementPointsInput"}}]},{"name": "registerPointModifier","accounts": [{"name": "key","isMut": false,"isSigner": true,"docs": ["The key authorized for this instruction"]},{"name": "profile","isMut": false,"isSigner": false,"docs": ["The points permissions [`Profile`](player_profile::state::Profile)"]},{"name": "funder","isMut": true,"isSigner": true,"docs": ["The funder - pays for account rent"]},{"name": "category","isMut": false,"isSigner": false,"docs": ["The [`PointCategory`]"]},{"name": "pointsModifierAccount","isMut": true,"isSigner": true,"docs": ["The [`PointsModifier`] account"]},{"name": "systemProgram","isMut": false,"isSigner": false,"docs": ["The Solana System program"]}],"args": [{"name": "canIncrement","type": "bool"},{"name": "canDecrement","type": "bool"},{"name": "keyIndex","type": "u16"}]},{"name": "removePointCategoryLevel","accounts": [{"name": "key","isMut": false,"isSigner": true,"docs": ["The key authorized for this instruction"]},{"name": "profile","isMut": false,"isSigner": false,"docs": ["The points permissions [`Profile`](Profile)"]},{"name": "fundsTo","isMut": true,"isSigner": false,"docs": ["The funds_to - receives rent refund"]},{"name": "category","isMut": true,"isSigner": false,"docs": ["The [PointCategory]"]},{"name": "systemProgram","isMut": false,"isSigner": false,"docs": ["The system program"]}],"args": [{"name": "input","type": {"defined": "RemovePointCategoryLevelInput"}}]},{"name": "spendPoints","accounts": [{"name": "spender","isMut": false,"isSigner": true,"docs": ["The entity calling this instruction"]},{"name": "spenderProfile","isMut": false,"isSigner": false,"docs": ["The profile of the spender"]},{"name": "category","isMut": false,"isSigner": false,"docs": ["The [`PointCategory`]"]},{"name": "userPointsAccount","isMut": true,"isSigner": false,"docs": ["The [`UserPointsAccount`]"]}],"args": [{"name": "pointsAmount","type": "u64"},{"name": "keyIndex","type": "u16"}]},{"name": "updatePointCategory","accounts": [{"name": "key","isMut": false,"isSigner": true,"docs": ["The key authorized for this instruction"]},{"name": "profile","isMut": false,"isSigner": false,"docs": ["The points permissions [`Profile`](Profile)"]},{"name": "category","isMut": true,"isSigner": false,"docs": ["The [PointCategory]"]},{"name": "systemProgram","isMut": false,"isSigner": false,"docs": ["The Solana System program"]}],"args": [{"name": "input","type": {"defined": "UpdatePointCategoryInput"}}]}],"accounts": [{"name": "PointCategory","docs": ["PDA for each specific type of Points"],"type": {"kind": "struct","fields": [{"name": "version","docs": ["The data version of this account."],"type": "u8"},{"name": "profile","docs": ["the managing profile"],"type": "publicKey"},{"name": "tokenRequired","docs": ["if this is true then token_mint should be a mint address"],"type": "u8"},{"name": "tokenMint","docs": ["the token mint"],"type": "publicKey"},{"name": "tokenQty","docs": ["the token qty"],"type": "u64"},{"name": "transferTokensToVault","docs": ["if this is true then token_vault should be a token address"],"type": "u8"},{"name": "tokenVault","docs": ["the token vault"],"type": "publicKey"},{"name": "pointLimit","docs": ["point limit"],"type": "u64"},{"name": "isSpendable","docs": ["is spendable?"],"type": "u8"},{"name": "postLevelsUpgradeThreshold","docs": ["the number of points required to upgrade a level after a user gets to the last level as set","in the levels array.If 0, this is turned off."],"type": "u64"}]}},{"name": "PointsModifier","docs": ["PDA containing one account that can modify an Points type"],"type": {"kind": "struct","fields": [{"name": "version","docs": ["The data version of this account."],"type": "u8"},{"name": "pointCategory","docs": ["point_category"],"type": "publicKey"},{"name": "canIncrement","docs": ["can_increment"],"type": "u8"},{"name": "canDecrement","docs": ["can_decrement"],"type": "u8"}]}},{"name": "UserPointsAccount","docs": ["PDA of an User for each specific type of Points"],"type": {"kind": "struct","fields": [{"name": "version","docs": ["The data version of this account."],"type": "u8"},{"name": "profile","docs": ["The profile this account is for"],"type": "publicKey"},{"name": "pointCategory","docs": ["point_category"],"type": "publicKey"},{"name": "earnedPoints","docs": ["earned_points"],"type": "u64"},{"name": "spentPoints","docs": ["spent_points"],"type": "u64"},{"name": "level","docs": ["The current level; 0 when no level"],"type": "u16"},{"name": "dailyEarnedPoints","docs": ["daily earned_points"],"type": "u64"},{"name": "lastEarnedPointsTimestamp","docs": ["The last timestamp at which points were earned"],"type": "i64"},{"name": "bump","docs": ["PDA bump"],"type": "u8"}]}}],"types": [{"name": "AddPointCategoryLevelInput","docs": ["Struct for data add points category level"],"type": {"kind": "struct","fields": [{"name": "level","docs": ["the level"],"type": "u16"},{"name": "points","docs": ["the amount of points required for this level"],"type": "u64"},{"name": "tokenQty","docs": ["the quantity of tokens required for this level"],"type": {"option": "u64"}},{"name": "license","docs": ["the type of license"],"type": "u8"},{"name": "keyIndex","docs": ["the index of the key in the player profile"],"type": "u16"}]}},{"name": "CreatePointCategoryInput","docs": ["Struct for data input to create a points category"],"type": {"kind": "struct","fields": [{"name": "license","docs": ["token_required"],"type": {"defined": "LicenseType"}},{"name": "pointLimit","docs": ["point_limit"],"type": {"option": "u64"}},{"name": "isSpendable","docs": ["is_spendable"],"type": "bool"},{"name": "keyIndex","docs": ["the index of the key in the crafting permissions profile"],"type": "u16"}]}},{"name": "DecrementLevelInput","docs": ["Struct for data input to decrement a user points account level"],"type": {"kind": "struct","fields": [{"name": "decrementToLevelIndex","docs": ["the index of the level to decrement to"],"type": "u16"}]}},{"name": "IncrementLevelInput","docs": ["Struct for data input to increment a user points account level"],"type": {"kind": "struct","fields": [{"name": "nextLevelIndex","docs": ["the index of the users desired level"],"type": "u16"}]}},{"name": "IncrementPointsInput","docs": ["Input data to increment points"],"type": {"kind": "struct","fields": [{"name": "points","docs": ["the amount of points to increment"],"type": "u64"},{"name": "dailyPointsLimit","docs": ["the daily points limit"],"type": {"option": "u64"}}]}},{"name": "LicenseType","docs": ["The type of license to use for the category"],"type": {"kind": "enum","variants": [{"name": "None"},{"name": "Burn","fields": [{"name": "quantity","docs": ["The quantity to burn"],"type": "u64"}]},{"name": "Vault","fields": [{"name": "quantity","docs": ["The quantity to transfer"],"type": "u64"}]}]}},{"name": "OptionalNonSystemPubkey","docs": ["A pubkey sized option that is none if set to the system program."],"type": {"kind": "struct","fields": [{"name": "key","type": "publicKey"}]}},{"name": "PointsLevel","docs": ["Defines an individual points level"],"type": {"kind": "struct","fields": [{"name": "level","docs": ["the level"],"type": "u16"},{"name": "points","docs": ["the amount of points required for this level","the total points required for the level i.e. the amount needed to upgrade to this level from level 0"],"type": "u64"},{"name": "tokenVault","docs": ["the token mint or vault","this allows for upgrading levels to require tokens (e.g. a piloting license)"],"type": "publicKey"},{"name": "tokenQty","docs": ["the token qty"],"type": "u64"}]}},{"name": "PointsLevelLicenseType","docs": ["The license type required for an individual points level","Can be:","1. None: no license","2. Burn: requires a license and burns tokens","3. Vault: requires a license and transfers tokens to vault"],"type": {"kind": "enum","variants": [{"name": "None"},{"name": "Burn"},{"name": "Vault"}]}},{"name": "RemovePointCategoryLevelInput","docs": ["Struct for data input to remove points category level"],"type": {"kind": "struct","fields": [{"name": "levelIndex","docs": ["the index of the level in the array"],"type": "u16"},{"name": "keyIndex","docs": ["the index of the key in the player profile"],"type": "u16"}]}},{"name": "UpdatePointCategoryInput","docs": ["Struct for data input to Update point categories"],"type": {"kind": "struct","fields": [{"name": "pointLimit","docs": ["point limit"],"type": {"option": "u64"}},{"name": "newLicense","docs": ["new licence for category"],"type": {"option": {"defined": "LicenseType"}}},{"name": "isSpendable","docs": ["is_spendable"],"type": {"option": "bool"}},{"name": "postLevelsUpgradeThreshold","docs": ["the number of points required to upgrade a level after a user gets to the last level as set in the levels array."],"type": {"option": "u64"}},{"name": "keyIndex","docs": ["the index of the key in the player profile"],"type": "u16"}]}}],"errors": [{"code": 6000,"name": "IncorrectAdminAddress","msg": "Incorrect admin address."},{"code": 6001,"name": "IncorrectMintAddress","msg": "Incorrect mint address."},{"code": 6002,"name": "IncorrectTokenAddress","msg": "Incorrect token address."},{"code": 6003,"name": "IncorrectModifierAddress","msg": "Incorrect modifier address."},{"code": 6004,"name": "IncorrectPointCategoryAddress","msg": "Incorrect owner address."},{"code": 6005,"name": "IncorrectProgramAddress","msg": "Incorrect program address."},{"code": 6006,"name": "IncorrectOwner","msg": "Incorrect Point Category."},{"code": 6007,"name": "IncrementNotAllowed","msg": "Not allowed to increment Points."},{"code": 6008,"name": "DecrementNotAllowed","msg": "Not allowed to decrement Points."},{"code": 6009,"name": "InsufficientTokenToBurn","msg": "Insufficient token licenses to burn."},{"code": 6010,"name": "LicenseRequired","msg": "License required to this category of Points."},{"code": 6011,"name": "LicenseNotRequired","msg": "License NOT required to this category of Points."},{"code": 6012,"name": "TokenMintRequired","msg": "Token Mint account is required."},{"code": 6013,"name": "TokenVaultRequired","msg": "Token Vault account is required."},{"code": 6014,"name": "NotEnoughEarnedPoints","msg": "Not enough earned points."},{"code": 6015,"name": "NotEnoughPoints","msg": "Not enough points."},{"code": 6016,"name": "SpendingNotAllowed","msg": "Points modifier not allowed to spend points."},{"code": 6017,"name": "TokenQuantityExpected","msg": "The token quantity must be provided."},{"code": 6018,"name": "NumericOverflow","msg": "Numeric overflow."},{"code": 6019,"name": "InvalidLevel","msg": "Invalid Level."},{"code": 6020,"name": "PointLevelRequired","msg": "The Point Level is required."},{"code": 6021,"name": "LevelUpgradeNotAllowed","msg": "Level Upgrade Not Allowed."},{"code": 6022,"name": "InvalidIndex","msg": "The Index is invalid."}]}');
    const pointsStoreIDL = JSON.parse('{"version":"0.1.0","name":"points_store","instructions":[{"name":"addRedemptionEpoch","accounts":[{"name":"key","isMut":false,"isSigner":true,"docs":["The key authorized for this instruction"]},{"name":"profile","isMut":false,"isSigner":false,"docs":["The [`Profile`] that handles the points store program permissions"]},{"name":"funder","isMut":true,"isSigner":true,"docs":["The funder for the new `RedemptionConfig`."]},{"name":"config","isMut":true,"isSigner":false,"docs":["The `RedemptionConfig`."]},{"name":"systemProgram","isMut":false,"isSigner":false,"docs":["The system program."]}],"args":[{"name":"input","type":{"defined":"AddRedemptionEpochInput"}}]},{"name":"buy","accounts":[{"name":"key","isMut":true,"isSigner":true,"docs":["This is a key in the `user_profile` that is authorized to spend user points."]},{"name":"userProfile","isMut":false,"isSigner":false,"docs":["The users [`Profile`]"]},{"name":"store","isMut":false,"isSigner":false,"docs":["The store to buy from."]},{"name":"storeSigner","isMut":false,"isSigner":false,"docs":["The stores signer."]},{"name":"bank","isMut":true,"isSigner":false,"docs":["The stores bank."]},{"name":"userPointsAccount","isMut":true,"isSigner":false,"docs":["The users points account."]},{"name":"tokensTo","isMut":true,"isSigner":false,"docs":["The account where the tokens will be sent."]},{"name":"pointCategory","isMut":false,"isSigner":false,"docs":["The category of points to spend."]},{"name":"pointsProgram","isMut":false,"isSigner":false,"docs":["The points program."]},{"name":"tokenProgram","isMut":false,"isSigner":false,"docs":["The token program."]}],"args":[{"name":"amount","type":"u64"},{"name":"keyIndex","type":"u16"},{"name":"expectedPrice","type":"u64"}]},{"name":"changeStorePrice","accounts":[{"name":"key","isMut":false,"isSigner":true,"docs":["The key authorized for this instruction"]},{"name":"profile","isMut":false,"isSigner":false,"docs":["The points permissions [`Profile`]"]},{"name":"store","isMut":true,"isSigner":false,"docs":["The store to change the price of."]}],"args":[{"name":"input","type":{"defined":"ChangeStorePriceInputUnpacked"}}]},{"name":"claimTokens","accounts":[{"name":"key","isMut":false,"isSigner":true,"docs":["The key authorized for this instruction"]},{"name":"fundsTo","isMut":true,"isSigner":false,"docs":["Where the user_redemption_accounts funds will be sent."]},{"name":"userProfile","isMut":false,"isSigner":false,"docs":["The players [`Profile`]"]},{"name":"userRedemptionAccount","isMut":true,"isSigner":false,"docs":["The [`UserRedemption`]"]},{"name":"config","isMut":true,"isSigner":false,"docs":["The `RedemptionConfig`."]},{"name":"configSigner","isMut":false,"isSigner":false,"docs":["The configs signer."]},{"name":"bank","isMut":true,"isSigner":false,"docs":["The configs bank."]},{"name":"tokensTo","isMut":true,"isSigner":false,"docs":["The account where the tokens will be sent."]},{"name":"tokenProgram","isMut":false,"isSigner":false,"docs":["The token program."]}],"args":[{"name":"input","type":{"defined":"ClaimTokensInput"}}]},{"name":"closeRedemptionConfig","accounts":[{"name":"key","isMut":false,"isSigner":true,"docs":["The key authorized for this instruction"]},{"name":"profile","isMut":false,"isSigner":false,"docs":["The points permissions [`Profile`]"]},{"name":"config","isMut":true,"isSigner":false,"docs":["The `RedemptionConfig` to close."]},{"name":"configSigner","isMut":false,"isSigner":false,"docs":["The configs signer"]},{"name":"bank","isMut":true,"isSigner":false,"docs":["The configs bank."]},{"name":"fundsTo","isMut":true,"isSigner":false,"docs":["Where the configs funds will be sent."]},{"name":"tokensTo","isMut":true,"isSigner":false,"docs":["Where the remaining config items are sent."]},{"name":"tokenProgram","isMut":false,"isSigner":false,"docs":["The token program."]},{"name":"systemProgram","isMut":false,"isSigner":false,"docs":["The system program"]}],"args":[{"name":"keyIndex","type":"u16"}]},{"name":"closeStore","accounts":[{"name":"key","isMut":false,"isSigner":true,"docs":["The key authorized for this instruction"]},{"name":"profile","isMut":false,"isSigner":false,"docs":["The points permissions [`Profile`]"]},{"name":"store","isMut":true,"isSigner":false,"docs":["The store to close."]},{"name":"storeSigner","isMut":false,"isSigner":false,"docs":["The stores signer"]},{"name":"bank","isMut":true,"isSigner":false,"docs":["The stores bank."]},{"name":"fundsTo","isMut":true,"isSigner":false,"docs":["Where the stores funds will be sent."]},{"name":"tokensTo","isMut":true,"isSigner":false,"docs":["Where the remaining store items are sent."]},{"name":"tokenProgram","isMut":false,"isSigner":false,"docs":["The token program."]},{"name":"systemProgram","isMut":false,"isSigner":false,"docs":["The system program"]}],"args":[{"name":"keyIndex","type":"u16"}]},{"name":"contributeToRedemption","accounts":[{"name":"key","isMut":false,"isSigner":true,"docs":["The key authorized for this instruction"]},{"name":"profile","isMut":false,"isSigner":false,"docs":["The players [`Profile`]"]},{"name":"profileFaction","isMut":false,"isSigner":false,"docs":["The faction that the profile belongs to."]},{"name":"pointCategory","isMut":false,"isSigner":false,"docs":["The category of points to spend."]},{"name":"userPointsAccount","isMut":true,"isSigner":false,"docs":["The users points account."]},{"name":"userRedemptionAccount","isMut":true,"isSigner":false,"docs":["The [`UserRedemption`]"]},{"name":"config","isMut":true,"isSigner":false,"docs":["The `RedemptionConfig` to be created."]},{"name":"pointsProgram","isMut":false,"isSigner":false,"docs":["The points program."]}],"args":[{"name":"input","type":{"defined":"ContributeToRedemptionInput"}}]},{"name":"createPointsStore","accounts":[{"name":"profile","isMut":false,"isSigner":false,"docs":["The [`Profile`] that handles the points store program permissions"]},{"name":"store","isMut":true,"isSigner":true,"docs":["The store to be created."]},{"name":"storeSigner","isMut":false,"isSigner":false,"docs":["The signer for the store."]},{"name":"pointCategory","isMut":false,"isSigner":false,"docs":["The category for points to spend."]},{"name":"funder","isMut":true,"isSigner":true,"docs":["The funder for the new store."]},{"name":"bank","isMut":false,"isSigner":false,"docs":["The bank for the tokens that can be bought"]},{"name":"systemProgram","isMut":false,"isSigner":false,"docs":["The system program."]}],"args":[{"name":"input","type":{"defined":"CreatePointsStoreInputUnpacked"}}]},{"name":"createRedemptionConfig","accounts":[{"name":"profile","isMut":false,"isSigner":false,"docs":["The [`Profile`] that handles the points store program permissions"]},{"name":"config","isMut":true,"isSigner":true,"docs":["The `RedemptionConfig` to be created."]},{"name":"configSigner","isMut":false,"isSigner":false,"docs":["The signer for the `RedemptionConfig`."]},{"name":"pointCategory","isMut":false,"isSigner":false,"docs":["The category for points to redeem."]},{"name":"funder","isMut":true,"isSigner":true,"docs":["The funder for the new `RedemptionConfig`."]},{"name":"bank","isMut":false,"isSigner":false,"docs":["The bank for the tokens that can be redeemed"]},{"name":"systemProgram","isMut":false,"isSigner":false,"docs":["The system program."]}],"args":[{"name":"input","type":{"defined":"CreateRedemptionConfigInput"}}]},{"name":"removeRedemptionEpoch","accounts":[{"name":"key","isMut":false,"isSigner":true,"docs":["The key authorized for this instruction"]},{"name":"profile","isMut":false,"isSigner":false,"docs":["The [`Profile`] that handles the points store program permissions"]},{"name":"funder","isMut":true,"isSigner":true,"docs":["The funder for the new `RedemptionConfig`."]},{"name":"config","isMut":true,"isSigner":false,"docs":["The `RedemptionConfig`."]},{"name":"systemProgram","isMut":false,"isSigner":false,"docs":["The system program."]}],"args":[{"name":"input","type":{"defined":"RemoveRedemptionEpochInput"}}]},{"name":"removeStoreItems","accounts":[{"name":"key","isMut":false,"isSigner":true,"docs":["The key authorized for this instruction"]},{"name":"profile","isMut":false,"isSigner":false,"docs":["The points permissions [`Profile`]"]},{"name":"store","isMut":false,"isSigner":false,"docs":["The store to remove items from."]},{"name":"storeSigner","isMut":false,"isSigner":false,"docs":["The stores signer."]},{"name":"bank","isMut":true,"isSigner":false,"docs":["The stores bank."]},{"name":"tokensTo","isMut":true,"isSigner":false,"docs":["Where the removed items will be sent."]},{"name":"tokenProgram","isMut":false,"isSigner":false,"docs":["The token program."]}],"args":[{"name":"amount","type":"u64"},{"name":"keyIndex","type":"u16"}]},{"name":"startRedemption","accounts":[{"name":"key","isMut":false,"isSigner":true,"docs":["The key authorized for this instruction"]},{"name":"funder","isMut":true,"isSigner":true,"docs":["The funder - pays for account rent"]},{"name":"userProfile","isMut":false,"isSigner":false,"docs":["The players [`Profile`]"]},{"name":"profileFaction","isMut":false,"isSigner":false,"docs":["The faction that the profile belongs to."]},{"name":"pointCategory","isMut":false,"isSigner":false,"docs":["The category of points to spend."]},{"name":"userPointsAccount","isMut":true,"isSigner":false,"docs":["The users points account."]},{"name":"userRedemptionAccount","isMut":true,"isSigner":true,"docs":["The [`UserRedemption`]"]},{"name":"config","isMut":true,"isSigner":false,"docs":["The `RedemptionConfig` to be created."]},{"name":"pointsProgram","isMut":false,"isSigner":false,"docs":["The points program."]},{"name":"systemProgram","isMut":false,"isSigner":false,"docs":["The Solana System Program"]}],"args":[{"name":"input","type":{"defined":"StartRedemptionInput"}}]},{"name":"updateRedemptionEpoch","accounts":[{"name":"key","isMut":false,"isSigner":true,"docs":["The key authorized for this instruction"]},{"name":"profile","isMut":false,"isSigner":false,"docs":["The [`Profile`] that handles the points store program permissions"]},{"name":"config","isMut":true,"isSigner":false,"docs":["The `RedemptionConfig`."]}],"args":[{"name":"input","type":{"defined":"UpdateRedemptionEpochInput"}}]}],"accounts":[{"name":"PointsStore","docs":["A set of tokens that can be bought with points from a category."],"type":{"kind":"struct","fields":[{"name":"version","docs":["The version of data in this struct."],"type":"u8"},{"name":"pointCategory","docs":["The category of points that can be spent"],"type":"publicKey"},{"name":"profile","docs":["The [`Profile`] that handles the points store program permissions"],"type":"publicKey"},{"name":"bank","docs":["The token wallet that stores the purchasable tokens."],"type":"publicKey"},{"name":"price","docs":["The points cost of each token."],"type":"u64"},{"name":"signerBump","docs":["The bump of the signer for the points store."],"type":"u8"}]}},{"name":"RedemptionConfig","docs":["Configuration to allow use of points to redeem tokens."],"type":{"kind":"struct","fields":[{"name":"version","docs":["The version of data in this struct."],"type":"u8"},{"name":"pointCategory","docs":["The category of points that can be used to redeem tokens"],"type":"publicKey"},{"name":"profile","docs":["The [`Profile`] that handles the points store program permissions"],"type":"publicKey"},{"name":"faction","docs":["The faction"],"type":"u8"},{"name":"bank","docs":["The token account that stores the redeemable tokens."],"type":"publicKey"},{"name":"signerBump","docs":["The bump of the signer for the points store."],"type":"u8"},{"name":"allowOnlyCurrentEpoch","docs":["Allow only the current epoch to be redeemed"],"type":"u8"}]}},{"name":"UserRedemption","docs":["User account for token redemptions for a certain epoch"],"type":{"kind":"struct","fields":[{"name":"version","docs":["The version of data in this struct."],"type":"u8"},{"name":"profile","docs":["The profile this account is for"],"type":"publicKey"},{"name":"pointCategory","docs":["The category of points that are being redeemed"],"type":"publicKey"},{"name":"userPointsAccount","docs":["The user points account"],"type":"publicKey"},{"name":"config","docs":["The redemption config account"],"type":"publicKey"},{"name":"points","docs":["The points submitted by the user to be redeemed for this epoch"],"type":"u64"},{"name":"dayIndex","docs":["The index of the day (24-hour period) that represents this epoch"],"type":"i64"}]}}],"types":[{"name":"AddRedemptionEpochInput","docs":["Input for `AddRedemptionEpoch`"],"type":{"kind":"struct","fields":[{"name":"totalTokens","docs":["The total tokens that can redeemed for this epoch"],"type":"u64"},{"name":"dayIndex","docs":["The index of the day (24 hour period) that represents this epoch"],"type":"i64"},{"name":"keyIndex","docs":["the index of the key in the player profile"],"type":"u16"}]}},{"name":"ChangeStorePriceInput","docs":["Input to [`ChangeStorePrice`]"],"type":{"kind":"struct","fields":[{"name":"newPrice","docs":["The price for the store"],"type":"u64"},{"name":"keyIndex","docs":["The key permissions index"],"type":"u16"}]}},{"name":"ChangeStorePriceInputUnpacked","docs":["Unpacked version of [`ChangeStorePriceInput`]"],"type":{"kind":"struct","fields":[{"name":"newPrice","docs":["The price for the store"],"type":"u64"},{"name":"keyIndex","docs":["The key permissions index"],"type":"u16"}]}},{"name":"ClaimTokensInput","docs":["Input for `ClaimTokens`"],"type":{"kind":"struct","fields":[{"name":"epochIndex","docs":["the index of the epoch in the `RedemptionConfig` account"],"type":"u16"},{"name":"keyIndex","docs":["the index of the key in the player profile"],"type":"u16"}]}},{"name":"ContributeToRedemptionInput","docs":["Input for `StartRedemption`"],"type":{"kind":"struct","fields":[{"name":"points","docs":["The number of points to submit"],"type":"u64"},{"name":"epochIndex","docs":["the index of the epoch in the `RedemptionConfig` account"],"type":"u16"},{"name":"keyIndex","docs":["the index of the key that can spend points in the player profile"],"type":"u16"}]}},{"name":"CreatePointsStoreInput","docs":["Input to [`CreatePointsStore`]"],"type":{"kind":"struct","fields":[{"name":"price","docs":["The price for the store"],"type":"u64"}]}},{"name":"CreatePointsStoreInputUnpacked","docs":["Unpacked version of [`CreatePointsStoreInput`]"],"type":{"kind":"struct","fields":[{"name":"price","docs":["The price for the store"],"type":"u64"}]}},{"name":"CreateRedemptionConfigInput","docs":["Input for `CreateRedemptionConfig`"],"type":{"kind":"struct","fields":[{"name":"faction","docs":["the faction that uses this `RedemptionConfig"],"type":"u8"},{"name":"allowOnlyCurrentEpoch","docs":["Allow only the current epoch to be redeemed"],"type":"bool"}]}},{"name":"RedemptionEpoch","docs":["Represents token redemptions for a 24-hour period"],"type":{"kind":"struct","fields":[{"name":"totalPoints","docs":["The total points submitted by all users to be redeemed for this epoch"],"type":"u64"},{"name":"redeemedPoints","docs":["The total tokens have been redeemed for this epoch"],"type":"u64"},{"name":"totalTokens","docs":["The total tokens that can redeemed for this epoch"],"type":"u64"},{"name":"redeemedTokens","docs":["The tokens that have been redeemed for this epoch"],"type":"u64"},{"name":"dayIndex","docs":["The index of the day (24-hour period) that represents this epoch"],"type":"i64"}]}},{"name":"RemoveRedemptionEpochInput","docs":["Input for `RemoveRedemptionEpoch`"],"type":{"kind":"struct","fields":[{"name":"epochIndex","docs":["The index of epoch in the `RedemptionConfig` epoch array"],"type":"u16"},{"name":"keyIndex","docs":["the index of the key in the player profile"],"type":"u16"}]}},{"name":"StartRedemptionInput","docs":["Input for `StartRedemption`"],"type":{"kind":"struct","fields":[{"name":"points","docs":["The number of points to submit"],"type":"u64"},{"name":"epochIndex","docs":["the index of the epoch in the `RedemptionConfig` account"],"type":"u16"},{"name":"keyIndex","docs":["the index of the key that can spend points in the player profile"],"type":"u16"}]}},{"name":"UpdateRedemptionEpochInput","docs":["Input for `UpdateRedemptionEpoch`"],"type":{"kind":"struct","fields":[{"name":"totalTokens","docs":["The total tokens that can redeemed for this epoch"],"type":"u64"},{"name":"epochIndex","docs":["The index of epoch in the `RedemptionConfig` epoch array"],"type":"u16"},{"name":"keyIndex","docs":["the index of the key in the player profile"],"type":"u16"}]}}],"errors":[{"code":6000,"name":"IncorrectProfileAddress","msg":"Incorrect profile address."},{"code":6001,"name":"IncorrectBankAddress","msg":"Incorrect bank address."},{"code":6002,"name":"IncorrectMintAddress","msg":"Incorrect mint address."},{"code":6003,"name":"EpochHasRedemptions","msg":"Epoch Has Redemptions."},{"code":6004,"name":"EpochHasSubmissions","msg":"Epoch Has Submissions."},{"code":6005,"name":"MustRemoveAllEpochs","msg":"Must Remove All Epochs."},{"code":6006,"name":"FactionMismatch","msg":"Faction mismatch"},{"code":6007,"name":"EpochHasNoTokensAvailable","msg":"Epoch Has No Tokens Available."},{"code":6008,"name":"EpochHasNoPointsAvailable","msg":"Epoch Has No Points Available."},{"code":6009,"name":"EpochMismatch","msg":"Epoch Mismatch."},{"code":6010,"name":"ConfigMismatch","msg":"Config Mismatch."}]}');
    //const craftingIDL = await BrowserAnchor.anchor.Program.fetchIdl(craftingProgramPK, anchorProvider);
    const craftingIDL = {version: "0.1.0",name: "crafting2",instructions: [{name: "addConsumableInputToRecipe",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The crafting permissions [`Profile`]"]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder - pays for account rent"]}, {name: "recipe",isMut: !0,isSigner: !1,docs: ["The [Recipe] account"]}, {name: "domain",isMut: !1,isSigner: !1,docs: ["The [Domain] account"]}, {name: "mint",isMut: !1,isSigner: !1,docs: ["The Mint Account"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The System program"]}],args: [{name: "input",type: {defined: "RecipeIngredients"}}]}, {name: "addCraftingFacilityRecipeCategory",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The crafting permissions [`Profile`]"]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder - pays for account rent"]}, {name: "recipeCategory",isMut: !1,isSigner: !1,docs: ["The [`RecipeCategory`] account"]}, {name: "craftingFacility",isMut: !0,isSigner: !1,docs: ["The [`CraftingFacility`] account"]}, {name: "domain",isMut: !1,isSigner: !1,docs: ["the [Domain] account"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The Solana System program"]}],args: [{name: "input",type: {defined: "KeyIndexInput"}}]}, {name: "addNonConsumableInputToRecipe",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The crafting permissions [`Profile`]"]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder - pays for account rent"]}, {name: "recipe",isMut: !0,isSigner: !1,docs: ["The [Recipe] account"]}, {name: "domain",isMut: !1,isSigner: !1,docs: ["The [Domain] account"]}, {name: "mint",isMut: !1,isSigner: !1,docs: ["The Cargo Type Account"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The System program"]}],args: [{name: "input",type: {defined: "RecipeIngredients"}}]}, {name: "addOutputToRecipe",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The crafting permissions [`Profile`]"]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder - pays for account rent"]}, {name: "recipe",isMut: !0,isSigner: !1,docs: ["The [Recipe] account"]}, {name: "domain",isMut: !1,isSigner: !1,docs: ["The [Domain] account"]}, {name: "craftableItem",isMut: !1,isSigner: !1,docs: ["The Craftable Item that the output represents"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The System program"]}],args: [{name: "input",type: {defined: "RecipeIngredients"}}]}, {name: "addRecipeIngredient",accounts: [{name: "location",isMut: !1,isSigner: !0,docs: ["The entity calling this instruction, this is the Crafting Facilitys location", "Having the location as the signer is an implicit check that this is the correct location"]}, {name: "authority",isMut: !1,isSigner: !0,docs: ["The [`CraftingProcess`] authority"]}, {name: "craftingProcess",isMut: !0,isSigner: !1,docs: ["The [`CraftingProcess`] account"]}, {name: "recipe",isMut: !1,isSigner: !1,docs: ["The [`Recipe`] account"]}, {name: "craftingFacility",isMut: !1,isSigner: !1,docs: ["The [`CraftingFacility`] account"]}, {name: "tokenFrom",isMut: !0,isSigner: !1,docs: ["The recipe ingredient token account."]}, {name: "tokenTo",isMut: !0,isSigner: !1,docs: ["The token account owned by the `crafting_process` which holds the ingredient in escrow"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["The [Token] program"]}],args: [{name: "input",type: {defined: "AddRecipeIngredientInput"}}]}, {name: "burnConsumableIngredient",accounts: [{name: "craftingProcess",isMut: !0,isSigner: !1,docs: ["The [`CraftingProcess`] account"]}, {name: "recipe",isMut: !1,isSigner: !1,docs: ["The [`Recipe`] account"]}, {name: "tokenFrom",isMut: !0,isSigner: !1,docs: ["The token account owned by the `crafting_process` which holds the ingredient in escrow"]}, {name: "mint",isMut: !0,isSigner: !1,docs: ["The mint of the consumable recipe ingredient"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["The [Token] program"]}],args: [{name: "input",type: {defined: "BurnConsumableIngredientInput"}}]}, {name: "cancelCraftingProcess",accounts: [{name: "location",isMut: !1,isSigner: !0,docs: ["The entity calling this instruction, this is the Crafting Facilitys location", "Having the location as the signer is an implicit check that this is the correct location"]}, {name: "authority",isMut: !1,isSigner: !0,docs: ["The [`CraftingProcess`] authority"]}, {name: "fundsTo",isMut: !0,isSigner: !1,docs: ["The funds_to - receives rent refund"]}, {name: "craftingProcess",isMut: !0,isSigner: !1,docs: ["The [`CraftingProcess`] account"]}, {name: "craftingFacility",isMut: !0,isSigner: !1,docs: ["The [`CraftingFacility`] account"]}],args: []}, {name: "claimNonConsumableIngredient",accounts: [{name: "authority",isMut: !1,isSigner: !1,docs: ["The owner/authority of crafting_process account"]}, {name: "craftingProcess",isMut: !0,isSigner: !1,docs: ["The [`CraftingProcess`] account"]}, {name: "recipe",isMut: !1,isSigner: !1,docs: ["The [`Recipe`] account"]}, {name: "tokenFrom",isMut: !0,isSigner: !1,docs: ["The token account owned by the `crafting_process` which holds the ingredient in escrow"]}, {name: "tokenTo",isMut: !0,isSigner: !1,docs: ["The token account to receive the non-consumable ingredient."]}, {name: "mint",isMut: !0,isSigner: !1,docs: ["The mint of the recipe ingredient"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["The [Token] program"]}],args: [{name: "input",type: {defined: "ClaimNonConsumableIngredientInput"}}]}, {name: "claimRecipeOutput",accounts: [{name: "authority",isMut: !1,isSigner: !1,docs: ["The owner/authority of crafting_process account"]}, {name: "craftingProcess",isMut: !0,isSigner: !1,docs: ["The [`CraftingProcess`] account"]}, {name: "recipe",isMut: !1,isSigner: !1,docs: ["The [`Recipe`] account"]}, {name: "tokenFrom",isMut: !0,isSigner: !1,docs: ["The token account owned by `craftable_item`."]}, {name: "tokenTo",isMut: !0,isSigner: !1,docs: ["The recipe ingredient token account."]}, {name: "craftableItem",isMut: !1,isSigner: !1,docs: ["The craftable_item (this is also the mint authority)"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["The [Token] program"]}],args: [{name: "input",type: {defined: "ClaimRecipeOutputInput"}}]}, {name: "closeCraftingProcess",accounts: [{name: "authority",isMut: !1,isSigner: !0,docs: ["The [`CraftingProcess`] authority"]}, {name: "fundsTo",isMut: !0,isSigner: !1,docs: ["The funds_to - receives rent refund"]}, {name: "craftingProcess",isMut: !0,isSigner: !1,docs: ["The [`CraftingProcess`] account"]}, {name: "recipe",isMut: !1,isSigner: !1,docs: ["The [`Recipe`] account"]}, {name: "craftingFacility",isMut: !0,isSigner: !1,docs: ["The [`CraftingFacility`] account"]}],args: []}, {name: "createCraftingProcess",accounts: [{name: "location",isMut: !1,isSigner: !0,docs: ["The entity calling this instruction, this is the Crafting Facilitys location", "Having the location as the signer is an implicit check that this is the correct location"]}, {name: "authority",isMut: !1,isSigner: !0,docs: ["The owner/authority for the new crafting_process account"]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder - pays for account rent"]}, {name: "craftingProcess",isMut: !0,isSigner: !1,docs: ["The [`CraftingProcess`] account"]}, {name: "recipe",isMut: !1,isSigner: !1,docs: ["The [`Recipe`] account"]}, {name: "craftingFacility",isMut: !0,isSigner: !1,docs: ["The [`CraftingFacility`] account"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["Solana System program"]}],args: [{name: "input",type: {defined: "CreateCraftingProcessInput"}}]}, {name: "deregisterCraftingFacility",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The crafting permissions [`Profile`]"]}, {name: "fundsTo",isMut: !0,isSigner: !1,docs: ["The funds_to - receives rent refund"]}, {name: "craftingFacility",isMut: !0,isSigner: !1,docs: ["The [`CraftingFacility`] account"]}, {name: "domain",isMut: !1,isSigner: !1,docs: ["the [Domain] account"]}],args: [{name: "input",type: {defined: "KeyIndexInput"}}]}, {name: "deregisterRecipeCategory",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The crafting permissions [`Profile`]"]}, {name: "fundsTo",isMut: !0,isSigner: !1,docs: ["The funds_to - receives rent refund"]}, {name: "recipeCategory",isMut: !0,isSigner: !1,docs: ["The [RecipeCategory] account"]}, {name: "domain",isMut: !1,isSigner: !1,docs: ["the [Domain] account"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["Solana System program"]}],args: [{name: "input",type: {defined: "KeyIndexInput"}}]}, {name: "drainCraftableItemBank",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The crafting permissions [`Profile`]"]}, {name: "fundsTo",isMut: !0,isSigner: !1,docs: ["The funds_to - receives rent refund"]}, {name: "craftableItem",isMut: !0,isSigner: !1,docs: ["The Craftable Item account"]}, {name: "domain",isMut: !1,isSigner: !1,docs: ["the [omain account"]}, {name: "tokenFrom",isMut: !0,isSigner: !1,docs: ["The craftable item token bank to drain"]}, {name: "tokenTo",isMut: !0,isSigner: !1,docs: ["Where to send tokens from the bank"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["The token program"]}],args: [{name: "input",type: {defined: "KeyIndexInput"}}]}, {name: "initializeDomain",accounts: [{name: "signer",isMut: !0,isSigner: !0,docs: ["The entity calling this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The [`Profile`] that handles the crafting program permissions"]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder - pays for account rent"]}, {name: "domain",isMut: !0,isSigner: !0,docs: ["The [Domain] account"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["Solana System program"]}],args: [{name: "namespace",type: {array: ["u8", 32]}}]}, {name: "legitimizeRecipeIngredient",accounts: [{name: "location",isMut: !1,isSigner: !0,docs: ["The entity calling this instruction, this is the Crafting Facilitys location", "Having the location as the signer is an implicit check that this is the correct location"]}, {name: "authority",isMut: !1,isSigner: !0,docs: ["The [`CraftingProcess`] authority"]}, {name: "craftingProcess",isMut: !0,isSigner: !1,docs: ["The [`CraftingProcess`] account"]}, {name: "recipe",isMut: !1,isSigner: !1,docs: ["The [`Recipe`] account"]}, {name: "craftingFacility",isMut: !1,isSigner: !1,docs: ["The [`CraftingFacility`] account"]}, {name: "tokenTo",isMut: !0,isSigner: !1,docs: ["The token account owned by the `crafting_process` which holds the ingredient in escrow"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["The [Token] program"]}],args: [{name: "input",type: {defined: "LegitimizeRecipeIngredientInput"}}]}, {name: "registerCraftableItem",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The crafting permissions [`Profile`]"]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder - pays for account rent"]}, {name: "mint",isMut: !0,isSigner: !1,docs: ["The mint of the Craftable Item to be registered"]}, {name: "craftableItem",isMut: !0,isSigner: !1,docs: ["The Craftable Item to be registered"]}, {name: "domain",isMut: !1,isSigner: !1,docs: ["the [Domain] account"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The Solana System program"]}],args: [{name: "input",type: {defined: "RegisterCraftableItemInput"}}]}, {name: "registerCraftingFacility",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The crafting permissions [`Profile`]"]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder - pays for account rent"]}, {name: "craftingFacility",isMut: !0,isSigner: !0,docs: ["The [`CraftingFacility`] account"]}, {name: "domain",isMut: !1,isSigner: !1,docs: ["the [Domain] account"]}, {name: "location",isMut: !1,isSigner: !1,docs: ["The Location address"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The Solana System program"]}],args: [{name: "input",type: {defined: "RegisterCraftingFacilityInput"}}]}, {name: "registerRecipe",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The crafting permissions [`Profile`]"]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder - pays for account rent"]}, {name: "recipe",isMut: !0,isSigner: !0,docs: ["The [Recipe] account"]}, {name: "recipeCategory",isMut: !0,isSigner: !1,docs: ["The [RecipeCategory] account"]}, {name: "domain",isMut: !1,isSigner: !1,docs: ["The [Domain] account"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The Solana System program"]}],args: [{name: "input",type: {defined: "RegisterRecipeInput"}}]}, {name: "registerRecipeCategory",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The crafting permissions [`Profile`]"]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder - pays for account rent"]}, {name: "recipeCategory",isMut: !0,isSigner: !0,docs: ["The [RecipeCategory] account"]}, {name: "domain",isMut: !1,isSigner: !1,docs: ["the [Domain] account"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["Solana System program"]}],args: [{name: "input",type: {defined: "RegisterRecipeCategoryInput"}}]}, {name: "removeConsumableInputFromRecipe",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The crafting permissions [`Profile`]"]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder - pays for account rent"]}, {name: "recipe",isMut: !0,isSigner: !1,docs: ["The [Recipe] account"]}, {name: "domain",isMut: !1,isSigner: !1,docs: ["The [Domain] account"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The System program"]}],args: [{name: "input",type: {defined: "RemoveRecipeIngredients"}}]}, {name: "removeCraftingFacilityRecipeCategory",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The crafting permissions [`Profile`]"]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder - pays for account rent"]}, {name: "recipeCategory",isMut: !1,isSigner: !1,docs: ["The recipe category to remove."]}, {name: "craftingFacility",isMut: !0,isSigner: !1,docs: ["The [`CraftingFacility`] account"]}, {name: "domain",isMut: !1,isSigner: !1,docs: ["the [Domain] account"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The Solana System program"]}],args: [{name: "input",type: {defined: "RemoveCraftingFacilityRecipeCategoryInput"}}]}, {name: "removeNonConsumableInputFromRecipe",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The crafting permissions [`Profile`]"]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder - pays for account rent"]}, {name: "recipe",isMut: !0,isSigner: !1,docs: ["The [Recipe] account"]}, {name: "domain",isMut: !1,isSigner: !1,docs: ["The [Domain] account"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The System program"]}],args: [{name: "input",type: {defined: "RemoveRecipeIngredients"}}]}, {name: "removeOutputFromRecipe",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The crafting permissions [`Profile`]"]}, {name: "funder",isMut: !0,isSigner: !0,docs: ["The funder - pays for account rent"]}, {name: "recipe",isMut: !0,isSigner: !1,docs: ["The [Recipe] account"]}, {name: "domain",isMut: !1,isSigner: !1,docs: ["The [Domain] account"]}, {name: "systemProgram",isMut: !1,isSigner: !1,docs: ["The System program"]}],args: [{name: "input",type: {defined: "RemoveRecipeIngredients"}}]}, {name: "removeRecipeIngredient",accounts: [{name: "location",isMut: !1,isSigner: !0,docs: ["The entity calling this instruction, this is the Crafting Facilitys location", "Having the location as the signer is an implicit check that this is the correct location"]}, {name: "authority",isMut: !1,isSigner: !0,docs: ["The [`CraftingProcess`] authority"]}, {name: "craftingProcess",isMut: !0,isSigner: !1,docs: ["The [`CraftingProcess`] account"]}, {name: "recipe",isMut: !1,isSigner: !1,docs: ["The [`Recipe`] account"]}, {name: "craftingFacility",isMut: !1,isSigner: !1,docs: ["The [`CraftingFacility`] account"]}, {name: "tokenFrom",isMut: !0,isSigner: !1,docs: ["The token account owned by the `crafting_process` which holds the ingredient in escrow"]}, {name: "tokenTo",isMut: !0,isSigner: !1,docs: ["The token account that receives the recipe ingredient."]}, {name: "mint",isMut: !0,isSigner: !1,docs: ["The mint of the recipe ingredient"]}, {name: "tokenProgram",isMut: !1,isSigner: !1,docs: ["The [`Token`] program"]}],args: [{name: "input",type: {defined: "RemoveRecipeIngredientInput"}}]}, {name: "startCraftingProcess",accounts: [{name: "location",isMut: !1,isSigner: !0,docs: ["The entity calling this instruction, this is the Crafting Facilitys location", "Having the location as the signer is an implicit check that this is the correct location"]}, {name: "craftingProcess",isMut: !0,isSigner: !1,docs: ["The [`CraftingProcess`] account"]}, {name: "recipe",isMut: !0,isSigner: !1,docs: ["The [`Recipe`] account"]}, {name: "craftingFacility",isMut: !1,isSigner: !1,docs: ["The [`CraftingFacility`] account"]}],args: [{name: "input",type: {defined: "StartCraftingProcessInput"}}]}, {name: "stopCraftingProcess",accounts: [{name: "location",isMut: !1,isSigner: !0,docs: ["The entity calling this instruction, this is the Crafting Facilitys location", "Having the location as the signer is an implicit check that this is the correct location"]}, {name: "craftingProcess",isMut: !0,isSigner: !1,docs: ["The [`CraftingProcess`] account"]}, {name: "recipe",isMut: !0,isSigner: !1,docs: ["The [`Recipe`] account"]}, {name: "craftingFacility",isMut: !1,isSigner: !1,docs: ["The [`CraftingFacility`] account"]}],args: [{name: "input",type: {defined: "StopCraftingProcessInput"}}]}, {name: "updateCraftingFacility",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The crafting permissions [`Profile`]"]}, {name: "craftingFacility",isMut: !0,isSigner: !1,docs: ["The [`CraftingFacility`] account"]}, {name: "domain",isMut: !1,isSigner: !1,docs: ["the [Domain] account"]}],args: [{name: "input",type: {defined: "UpdateCraftingFacilityInput"}}]}, {name: "updateDomain",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The crafting permissions [`Profile`]"]}, {name: "newProfile",isMut: !1,isSigner: !1,docs: ["The new crafting permissions profile"]}, {name: "domain",isMut: !0,isSigner: !1,docs: ["the [Domain] account"]}],args: [{name: "input",type: {defined: "KeyIndexInput"}}]}, {name: "updateRecipe",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The crafting permissions [`Profile`]"]}, {name: "recipe",isMut: !0,isSigner: !1,docs: ["The [Recipe] to update"]}, {name: "domain",isMut: !1,isSigner: !1,docs: ["the [Domain] account"]}],args: [{name: "input",type: {defined: "UpdateRecipeInput"}}]}, {name: "updateRecipeCategory",accounts: [{name: "key",isMut: !1,isSigner: !0,docs: ["The key authorized for this instruction"]}, {name: "profile",isMut: !1,isSigner: !1,docs: ["The crafting permissions [`Profile`]"]}, {name: "recipe",isMut: !0,isSigner: !1,docs: ["The [Recipe] to update"]}, {name: "recipeCategoryOld",isMut: !0,isSigner: !1,docs: ["The old [RecipeCategory] account"]}, {name: "recipeCategoryNew",isMut: !0,isSigner: !1,docs: ["The new [RecipeCategory] account"]}, {name: "domain",isMut: !1,isSigner: !1,docs: ["the [Domain] account"]}],args: [{name: "input",type: {defined: "KeyIndexInput"}}]}],accounts: [{name: "craftableItem",docs: ["This PDA represents a token registered as an item that can be crafted"],type: {kind: "struct",fields: [{name: "version",docs: ["The data version of this account."],type: "u8"}, {name: "domain",docs: ["the domain"],type: "publicKey"}, {name: "mint",docs: ["the mint representing the items crafted"],type: "publicKey"}, {name: "creator",docs: ["TODO: remove this"],type: "publicKey"}, {name: "namespace",docs: ["the name of this account"],type: {array: ["u8", 32]}}, {name: "bump",docs: ["bump for PDA"],type: "u8"}]}}, {name: "craftingFacility",docs: ["Represents a crafting facility account"],type: {kind: "struct",fields: [{name: "version",docs: ["The data version of this account."],type: "u8"}, {name: "domain",docs: ["the domain"],type: "publicKey"}, {name: "location",docs: ["the locations pubkey"],type: "publicKey"}, {name: "locationType",docs: ["the location type"],type: "u8"}, {name: "maxConcurrentProcesses",docs: ["the max. number of concurrent crafting processes that can be handled by this facility", "if 0 then there is no max"],type: "u32"}, {name: "numConcurrentProcesses",docs: ["the current number of concurrent crafting processes"],type: "u32"}, {name: "efficiency",docs: ["the efficiency rate for this crafting_facility (as basis points)"],type: "u32"}, {name: "numRecipeCategories",docs: ["number of recipe categories"],type: "u32"}]}}, {name: "craftingProcess",docs: ["This account represents crafting in progress"],type: {kind: "struct",fields: [{name: "version",docs: ["The data version of this account."],type: "u8"}, {name: "craftingId",docs: ["the crafting id"],type: "u64"}, {name: "authority",docs: ["the owner/authority of this crafting process"],type: "publicKey"}, {name: "recipe",docs: ["the recipe"],type: "publicKey"}, {name: "craftingFacility",docs: ["the crafting facility"],type: "publicKey"}, {name: "inputsChecksum",docs: ["used to check if expected inputs have been supplied"],type: {array: ["u8", 16]}}, {name: "outputsChecksum",docs: ["used to check if expected outputs have been claimed"],type: {array: ["u8", 16]}}, {name: "quantity",docs: ["Quantity of outputs to craft"],type: "u64"}, {name: "status",docs: ["The status of this crafting process"],type: "u8"}, {name: "startTime",docs: ["the start timestamp"],type: "i64"}, {name: "endTime",docs: ["the end timestamp"],type: "i64"}, {name: "denyPermissionlessClaiming",docs: ["Whether or not to deny permission-less claiming.  True when > 0"],type: "u8"}, {name: "useLocalTime",docs: ["Whether or not to local time supplied by the location.  True when > 0"],type: "u8"}, {name: "bump",docs: ["bump for PDA"],type: "u8"}]}}, {name: "domain",docs: ["This account allows for a crafting domain which allows an admin to set up", "craftable items, recipes and crafting facilities tied to a domain that the control"],type: {kind: "struct",fields: [{name: "version",docs: ["The data version of this account."],type: "u8"}, {name: "profile",docs: ["The [`Profile`] that handles the crafting program permissions"],type: "publicKey"}, {name: "namespace",docs: ["the namespace"],type: {array: ["u8", 32]}}]}}, {name: "recipe",docs: ["The definition of a recipe."],type: {kind: "struct",fields: [{name: "version",docs: ["The data version of this account."],type: "u8"}, {name: "domain",docs: ["the domain"],type: "publicKey"}, {name: "category",docs: ["The Recipe Category"],type: "publicKey"}, {name: "creator",docs: ["TODO: remove this"],type: "publicKey"}, {name: "duration",docs: ["The time required to craft this Recipe."],type: "i64"}, {name: "minDuration",docs: ["The minimum time required to craft this Recipe."],type: "i64"}, {name: "namespace",docs: ["The name of this recipe."],type: {array: ["u8", 32]}}, {name: "status",docs: ["The status of the recipe"],type: "u8"}, {name: "feeAmount",docs: ["The amount to charge when this recipe is used"],type: "u64"}, {name: "feeRecipient",docs: ["The token account that receives the `fee_amount`.  If [`None`] the recipe requires no fees when used."],type: {defined: "OptionalNonSystemPubkey"}}, {name: "usageCount",docs: ["The number of times that this recipe has been used"],type: "u64"}, {name: "usageLimit",docs: ["The maximum number of times that this recipe can be used"],type: "u64"}, {name: "value",docs: ["The value of this recipe e.g. might be economic value", "The precise meaning of value is left to the person creating recipes to determine"],type: "u64"}, {name: "consumablesCount",docs: ["The number of consumable inputs in this recipe."],type: "u8"}, {name: "nonConsumablesCount",docs: ["The number of non-consumable inputs in this recipe."],type: "u8"}, {name: "outputsCount",docs: ["The number of outputs from this recipe."],type: "u8"}, {name: "totalCount",docs: ["The number of all inputs and outputs in this recipe."],type: "u16"}]}}, {name: "recipeCategory",docs: ["A Crafting Recipe Category account"],type: {kind: "struct",fields: [{name: "version",docs: ["The data version of this account."],type: "u8"}, {name: "domain",docs: ["the domain"],type: "publicKey"}, {name: "creator",docs: ["TODO: remove this"],type: "publicKey"}, {name: "recipeCount",docs: ["The number of recipes in this category."],type: "u32"}, {name: "namespace",docs: ["the name of this account"],type: {array: ["u8", 32]}}]}}],types: [{name: "AddRecipeIngredientInput",docs: ["Struct for data input to add an ingredient"],type: {kind: "struct",fields: [{name: "amount",docs: ["the amount of ingredient to deposit"],type: "u64"}, {name: "ingredientIndex",docs: ["the index of the recipe ingredient"],type: "u16"}]}}, {name: "BurnConsumableIngredientInput",docs: ["Struct for data input to burn a consumable ingredient"],type: {kind: "struct",fields: [{name: "ingredientIndex",docs: ["the index of the consumable recipe ingredient"],type: "u16"}, {name: "localTime",docs: ["the current time as supplied by the location; required if `use_local_time` is set on the `CraftingProcess`"],type: {option: "i64"}}]}}, {name: "ClaimNonConsumableIngredientInput",docs: ["Struct for data input to claim a non-consumable ingredient"],type: {kind: "struct",fields: [{name: "ingredientIndex",docs: ["the index of the recipe ingredient"],type: "u16"}, {name: "localTime",docs: ["the current time as supplied by the location; required if `use_local_time` is set on the `CraftingProcess`"],type: {option: "i64"}}]}}, {name: "ClaimRecipeOutputInput",docs: ["Struct for data input to claim an output"],type: {kind: "struct",fields: [{name: "ingredientIndex",docs: ["the index of the recipe output"],type: "u16"}, {name: "localTime",docs: ["the current time as supplied by the location; required if `use_local_time` is set on the `CraftingProcess`"],type: {option: "i64"}}]}}, {name: "CreateCraftingProcessInput",docs: ["Struct for data input to create a `CraftingProcess`"],type: {kind: "struct",fields: [{name: "craftingId",docs: ["crafting id"],type: "u64"}, {name: "recipeCategoryIndex",docs: ["the index of the recipes category"],type: "u16"}, {name: "quantity",docs: ["quantity of outputs to craft"],type: "u64"}, {name: "denyPermissionlessClaiming",docs: ["Whether or not to deny permission-less claiming.  True when > 0"],type: "bool"}, {name: "useLocalTime",docs: ["Whether or not to local time supplied by the location.  True when > 0"],type: "bool"}]}}, {name: "KeyIndexInput",docs: ["Struct for data input for that has `key_index`"],type: {kind: "struct",fields: [{name: "keyIndex",docs: ["the index of the key in the crafting permissions profile"],type: "u16"}]}}, {name: "LegitimizeRecipeIngredientInput",docs: ["Struct for data input to legitimize an ingredient"],type: {kind: "struct",fields: [{name: "amount",docs: ["the amount of ingredient to deposit"],type: "u64"}, {name: "ingredientIndex",docs: ["the index of the recipe ingredient"],type: "u16"}]}}, {name: "LocationType",docs: ["Represents different types of locations that a `CraftingFacility` might be found"],type: {kind: "enum",variants: [{name: "Starbase"}]}}, {name: "OptionalNonSystemPubkey",docs: ["A pubkey sized option that is none if set to the system program."],type: {kind: "struct",fields: [{name: "key",type: "publicKey"}]}}, {name: "ProcessStatus",docs: ["The `CraftingProcess` `status`"],type: {kind: "enum",variants: [{name: "Initialized"}, {name: "Started"}]}}, {name: "RecipeIngredients",docs: ["A copy of the [`RecipeInputsOutputs`] struct --> to be used as inputs into the program, avoiding using Borsch Serialization with Zero Copy serialization"],type: {kind: "struct",fields: [{name: "amount",docs: ["Amount of Input/Output required/produced."],type: "u64"}, {name: "mint",docs: ["Mint pubkey of the Input/Output."],type: "publicKey"}, {name: "keyIndex",docs: ["the index of the key in the crafting permissions profile"],type: "u16"}]}}, {name: "RecipeInputsOutputs",docs: ["The [`RecipeInputsOutputs`] struct."],type: {kind: "struct",fields: [{name: "amount",docs: ["Amount of Input/Output required/produced."],type: "u64"}, {name: "mint",docs: ["Mint pubkey of the Input/Output."],type: "publicKey"}]}}, {name: "RecipeStatus",docs: ["Represents the `Recipe` status"],type: {kind: "enum",variants: [{name: "Initializing"}, {name: "Active"}, {name: "Deactivated"}]}}, {name: "RegisterCraftableItemInput",docs: ["used as input struct when registering a `CraftableItem`"],type: {kind: "struct",fields: [{name: "namespace",docs: ["the `CraftableItem` namespace"],type: {array: ["u8", 32]}}, {name: "keyIndex",docs: ["the index of the key in the crafting permissions profile"],type: "u16"}]}}, {name: "RegisterCraftingFacilityInput",docs: ["Struct for data input to Register a `CraftingFacility`"],type: {kind: "struct",fields: [{name: "locationType",docs: ["`CraftingFacility` location type"],type: {defined: "LocationType"}}, {name: "efficiency",docs: ["`CraftingFacility` efficiency"],type: "u32"}, {name: "maxConcurrentProcesses",docs: ["the max. number of concurrent crafting processes that can be handled by this facility", "if 0 then there is no max"],type: "u32"}, {name: "keyIndex",docs: ["the index of the key in the crafting permissions profile"],type: "u16"}]}}, {name: "RegisterRecipeCategoryInput",docs: ["Struct for data input for `RegisterRecipeCategory`"],type: {kind: "struct",fields: [{name: "namespace",docs: ["the `RecipeCategory` namespace"],type: {array: ["u8", 32]}}, {name: "keyIndex",docs: ["the index of the key in the crafting permissions profile"],type: "u16"}]}}, {name: "RegisterRecipeInput",docs: ["used as input struct when registering a Recipe"],type: {kind: "struct",fields: [{name: "duration",docs: ["the recipe duration (seconds)"],type: "i64"}, {name: "minDuration",docs: ["the recipe minimum duration (seconds)"],type: "i64"}, {name: "namespace",docs: ["the recipe namespace"],type: {array: ["u8", 32]}}, {name: "feeAmount",docs: ["the recipe fee amount"],type: {option: "u64"}}, {name: "usageLimit",docs: ["the maximum number of times that this recipe can be used; if not provided a limit will not be set"],type: {option: "u64"}}, {name: "value",docs: ["the recipes value"],type: {option: "u64"}}, {name: "keyIndex",docs: ["the index of the key in the crafting permissions profile"],type: "u16"}]}}, {name: "RemoveCraftingFacilityRecipeCategoryInput",docs: ["Struct for data input for removing a [`CraftingFacility`] `RecipeCategory`"],type: {kind: "struct",fields: [{name: "recipeCategoryIndex",docs: ["the index of the `RecipeCategory` in the [`CraftingFacility`]"],type: "u16"}, {name: "keyIndex",docs: ["the index of the key in the crafting permissions profile"],type: "u16"}]}}, {name: "RemoveRecipeIngredientInput",docs: ["Struct for data input to remove an ingredient"],type: {kind: "struct",fields: [{name: "amount",docs: ["the amount of ingredient to withdraw/remove"],type: "u64"}, {name: "ingredientIndex",docs: ["the index of the recipe ingredient"],type: "u16"}]}}, {name: "RemoveRecipeIngredients",docs: ["used as input struct when removing recipe ingredients"],type: {kind: "struct",fields: [{name: "keyIndex",docs: ["the index of the key in the crafting permissions profile"],type: "u16"}, {name: "ingredientIndex",docs: ["the index of the recipe ingredient"],type: "u16"}]}}, {name: "StartCraftingProcessInput",docs: ["Inputs for `StartCraftingProcess`"],type: {kind: "struct",fields: [{name: "recipeDurationOverride",docs: ["recipe duration override"],type: {option: "u64"}}, {name: "localTime",docs: ["the current time as supplied by the location; required if `use_local_time` is set on the `CraftingProcess`"],type: {option: "i64"}}]}}, {name: "StopCraftingProcessInput",docs: ["Inputs for `StopCraftingProcess`"],type: {kind: "struct",fields: [{name: "localTime",docs: ["the current time as supplied by the location; required if `use_local_time` is set on the `CraftingProcess`"],type: {option: "i64"}}]}}, {name: "UpdateCraftingFacilityInput",docs: ["Struct for data input to Update `CraftingFacility`"],type: {kind: "struct",fields: [{name: "efficiency",docs: ["`CraftingFacility` efficiency"],type: {option: "u32"}}, {name: "maxConcurrentProcesses",docs: ["the max. number of concurrent crafting processes that can be handled by this facility", "if 0 then there is no max"],type: {option: "u32"}}, {name: "keyIndex",docs: ["the index of the key in the crafting permissions profile"],type: "u16"}]}}, {name: "UpdateRecipeInput",docs: ["Struct for data input to Update `Recipe`"],type: {kind: "struct",fields: [{name: "duration",docs: ["`Recipe` duration"],type: {option: "i64"}}, {name: "minDuration",docs: ["`Recipe` minimum duration"],type: {option: "i64"}}, {name: "feeAmount",docs: ["`Recipe` fee amount"],type: {option: "u64"}}, {name: "status",docs: ["`Recipe` status"],type: {option: "u8"}}, {name: "usageLimit",docs: ["the maximum number of times that this recipe can be used"],type: {option: "u64"}}, {name: "value",docs: ["the recipes value"],type: {option: "u64"}}, {name: "keyIndex",docs: ["the index of the key in the crafting permissions profile"],type: "u16"}]}}, {name: "WrappedRecipeCategory",docs: ["Wrapped recipe category", "We are wrapping this around Pubkey so that we can use Anchors coder to decode it"],type: {kind: "struct",fields: [{name: "id",docs: ["The recipe category public key"],type: "publicKey"}]}}],errors: [{code: 6e3,name: "IncorrectProfileAddress",msg: "Incorrect profile address."}, {code: 6001,name: "IncorrectAuthority",msg: "The authority is incorrect."}, {code: 6002,name: "NumericOverflow",msg: "Numeric overflow"}, {code: 6003,name: "MintAuthorityIsNone",msg: "The mint authority should exist."}, {code: 6004,name: "AdminNotMintAuthority",msg: "Admin wallet is not the mint authority of the entered token."}, {code: 6005,name: "IncorrectMintAddress",msg: "Incorrect mint address entered."}, {code: 6006,name: "LocationTypeNotSupported",msg: "The provided location type is not supported."}, {code: 6007,name: "IncorrectLocation",msg: "The location is incorrect."}, {code: 6008,name: "IncorrectDomain",msg: "The domain is incorrect."}, {code: 6009,name: "IncorrectRecipe",msg: "The recipe is incorrect."}, {code: 6010,name: "IncorrectRecipeCategory",msg: "The recipe category is incorrect."}, {code: 6011,name: "IncorrectCraftingFacility",msg: "The crafting facility is incorrect."}, {code: 6012,name: "OverCraftingFacilityCapacity",msg: "The crafting facility capacity has been exceeded."}, {code: 6013,name: "InsufficientAmount",msg: "The amount is insufficient."}, {code: 6014,name: "AmountRequired",msg: "An amount greater than 0 must be provided."}, {code: 6015,name: "AmountTooMuch",msg: "The amount is more than what is required."}, {code: 6016,name: "InputAlreadySupplied",msg: "The input has already been supplied."}, {code: 6017,name: "InputNotSupplied",msg: "The input has NOT been supplied."}, {code: 6018,name: "OutputAlreadyClaimed",msg: "The output has already been claimed."}, {code: 6019,name: "CraftingProcessAlreadyStarted",msg: "The crafting process has already started."}, {code: 6020,name: "CraftingProcessNotStarted",msg: "The crafting process has not started."}, {code: 6021,name: "CraftingProcessNotCompleted",msg: "The crafting process has not completed."}, {code: 6022,name: "CraftingProcessAlreadyCompleted",msg: "The crafting process has already completed."}, {code: 6023,name: "IngredientNotAnInput",msg: "The ingredient is not an input."}, {code: 6024,name: "IngredientNotAConsumableInput",msg: "The ingredient is not a consumable input."}, {code: 6025,name: "IngredientNotANonConsumableInput",msg: "The ingredient is not a non-consumable input."}, {code: 6026,name: "IngredientNotAnOutput",msg: "The ingredient is not an output."}, {code: 6027,name: "IncorrectInputsChecksum",msg: "The inputs checksum is incorrect."}, {code: 6028,name: "IncorrectOutputsChecksum",msg: "The outputs checksum is incorrect."}, {code: 6029,name: "RecipeCountNotZero",msg: "The recipe count is not zero."}, {code: 6030,name: "RecipeCategoryCountNotZero",msg: "The recipe category count is not zero."}, {code: 6031,name: "ConcurrentProcessCountNotZero",msg: "The concurrent process count is not zero."}, {code: 6032,name: "KeyIndexOutOfBounds",msg: "Key index out of bounds."}, {code: 6033,name: "InvalidDuration",msg: "Duration is invalid."}, {code: 6034,name: "MissingRequiredSignature",msg: "Missing required signature."}, {code: 6035,name: "AddressMismatchATA",msg: "ATA address does not match"}, {code: 6036,name: "CannotUpdateDeactivatedRecipe",msg: "Cannot Update Deactivated Recipe"}, {code: 6037,name: "InvalidRecipeStatus",msg: "Invalid Recipe Status"}, {code: 6038,name: "EfficiencyIsZero",msg: "Crafting Facility Efficiency Is Zero"}, {code: 6039,name: "AddressMismatch",msg: "Address Mismatch"}, {code: 6040,name: "RecipeLimitExceeded",msg: "Recipe Limit Exceeded"}]};


	const maxResWeight = 6;

	let userPublicKey = null;
	let userProfileAcct = null;
	let userProfileKeyIdx = 0;
    let pointsProfileKeyIdx = 0;
	let userProfileFactionAcct = null;
    let userRedemptionConfigAcct = null;
	let userFleetAccts = null;
	let userFleets = [];
    let userXpAccounts = {
        userCouncilRankXpAccounts: {},
        userDataRunningXpAccounts: {},
        userPilotingXpAccounts: {},
        userMiningXpAccounts: {},
        userCraftingXpAccounts: {},
        userLPAccounts: {},
    };
    let starbaseData = [];
    let planetData = [];
    let minableResourceData = null;
    let starbasePlayerData = [];
    let validTargets = [];

	let currentFee = globalSettings.priorityFee; //autofee

	let sageProgram = new BrowserAnchor.anchor.Program(sageIDL, sageProgramPK, anchorProvider);
	//let [sageGameAcct] = await sageProgram.account.game.all();
    let sageGameAccts = await sageProgram.account.game.all();
    let sageGameAcct = null;
    for (let curAcct of sageGameAccts) {
        if(curAcct.account.gameState.toString()!='11111111111111111111111111111111') {
            sageGameAcct = curAcct;
            break;
        }
    }

	let cargoStatsDefinitionAcctPK = sageGameAcct.account.cargo.statsDefinition;
	let [sageSDUTrackerAcct] = await sageProgram.account.surveyDataUnitTracker.all();
	let profileProgram = new BrowserAnchor.anchor.Program(profileIDL, profileProgramPK, anchorProvider);
    let pointsProgram = new BrowserAnchor.anchor.Program(pointsIDL, pointsProgramId, anchorProvider);
    let pointsStoreProgram = new BrowserAnchor.anchor.Program(pointsStoreIDL, pointsStoreProgramId, anchorProvider);
    let craftingProgram = new BrowserAnchor.anchor.Program(craftingIDL, craftingProgramPK, anchorProvider);
	let cargoProgram = new BrowserAnchor.anchor.Program(cargoIDL, cargoProgramPK, anchorProvider);
	//let [cargoStatsDefinitionAcct] = await cargoProgram.account.cargoStatsDefinition.all();
	let cargoStatsDefinitionAccts = await cargoProgram.account.cargoStatsDefinition.all();
	//we need to select the correct account:
	let cargoStatsDefinitionAcct = cargoStatsDefinitionAccts.find(item => item.publicKey.toString() === cargoStatsDefinitionAcctPK.toString());
	let cargoStatsDefSeqId = cargoStatsDefinitionAcct.account.seqId;
	let seqBN = new BrowserAnchor.anchor.BN(cargoStatsDefSeqId);
	let seqArr = seqBN.toTwos(64).toArrayLike(BrowserBuffer.Buffer.Buffer, "be", 2);
	let seq58 = bs58.encode(seqArr);
    let cargoItems = [];
    let craftableItems = [];
    let mineItems = [];
    let craftRecipes = [];
    let upgradeRecipes = [];
    let addressLookupTables = [];

	const cargoTypes = await cargoProgram.account.cargoType.all([
			{
				 memcmp: {
						 offset: 108,//75,
							bytes: seq58,
					},
			},
			{
				memcmp: {
					offset: 9,
					bytes: cargoStatsDefinitionAcct.publicKey.toBase58()
				}
			},
	]);

    cLog(0,'getResourceTokens()');
    await getResourceTokens();

    cLog(0,'getCraftRecipes()');
    await getCraftRecipes();

    cLog(0,'getALTs()');
    await getALTs();

    console.log('craftRecipes: ', craftRecipes);
    console.log('upgradeRecipes: ', upgradeRecipes);

    let sduItem = cargoItems.find(item => item.name === 'Survey Data Unit');
	let fuelItem = cargoItems.find(item => item.token === sageGameAcct.account.mints.fuel.toString());
	let foodItem = cargoItems.find(item => item.token === sageGameAcct.account.mints.food.toString());
	let toolItem = cargoItems.find(item => item.token === sageGameAcct.account.mints.repairKit.toString());
	let ammoItem = cargoItems.find(item => item.token === sageGameAcct.account.mints.ammo.toString());

    let [progressionConfigAcct] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
        [
            BrowserBuffer.Buffer.Buffer.from("ProgressionConfig"),
            sageGameAcct.publicKey.toBuffer()
        ],
        sageProgramPK
    );

    async function getALTs() {
        for (let lookupTableAddress of addressLookupTableAddresses) {
            let lookupTableAccount = await solanaReadConnection.getAddressLookupTable(lookupTableAddress);
            addressLookupTables.push(lookupTableAccount.value);
        }
    }

    async function getStarbaseUpkeepStatus() {
	let factionStarbases = await sageProgram.account.starbase.all([
		{
			memcmp: {
				offset: 201,
				bytes: [userProfileFactionAcct.account.faction]
			}
		}
	]);
        let starbasesStatus = [];
	for(let target of validTargets) {
		starbasesStatus.push(await upkeepBalance(factionStarbases.find(sb => ((sb.account.sector[0].toNumber() + ',' + sb.account.sector[1].toNumber()) == (target.x + ',' + target.y)))));
	}
        return starbasesStatus;
    }

    async function upkeepBalance(starbase) {
        let gameStateAcct = await sageProgram.account.gameState.fetch(sageGameAcct.account.gameState);
        let sbLevel = gameStateAcct.fleet.upkeep['level' + starbase.account.level];
        let upkeepFoodBalance = Math.max(0, starbase.account.upkeepFoodBalance.toNumber() - (Math.floor(Date.now() / 1000) - starbase.account.upkeepFoodGlobalLastUpdate.toNumber()) * (sbLevel.foodDepletionRate/100));
        let foodBalanceHr = (upkeepFoodBalance / (sbLevel.foodDepletionRate/100)) / 60 / 60;
        let upkeepToolBalance = Math.max(0, starbase.account.upkeepToolkitBalance.toNumber() - (Math.floor(Date.now() / 1000) - starbase.account.upkeepToolkitGlobalLastUpdate.toNumber()) * (sbLevel.toolkitDepletionRate/100));
        let toolBalanceHr = (upkeepToolBalance / (sbLevel.toolkitDepletionRate/100)) / 60 / 60;
        //let [name] = (new TextDecoder().decode(new Uint8Array(starbase.account.name)).replace(/\0/g, '')).split(' (');
        //name = name === 'ONI Central Space Station' ? 'ONI CSS' : name;
	let target = validTargets.find(target => (target.x + ',' + target.y) == starbase.account.sector[0].toNumber() + ',' + starbase.account.sector[1].toNumber());
	let name = target?.name;
        let coords = starbase.account.sector[0].toNumber() + ', ' + starbase.account.sector[1].toNumber();
        return {name: `${name} [LVL ${starbase.account.level}]`, coords: coords, foodBalanceHr: foodBalanceHr.toFixed(2), foodBalancePerc: (upkeepFoodBalance / sbLevel.foodReserve).toFixed(2), toolBalanceHr: toolBalanceHr.toFixed(2) || 'N/A', toolBalancePerc: (upkeepToolBalance / sbLevel.toolkitReserve).toFixed(2) || 'N/A'}
    }

    async function getSolanaClockTime() {
        let solanaClock = await solanaReadConnection.getAccountInfo(new solanaWeb3.PublicKey('SysvarC1ock11111111111111111111111111111111'));
        let solanaTime = solanaClock.data.readBigInt64LE(8 * 4);
    }
	async function influxStarbaseCargoHold() {} // [Sis-Ágora] estoque das starbases já vem das sincronizações da VPS


    async function getCargoTypeSizes(cargoTypes) {
	let publicKeys = [];
	let cargoTypeSizes = [];
	for(let i=0; i < cargoTypes.length; i++) {
		publicKeys.push(cargoTypes[i].publicKey);
	}
	// 100 is the max data size of getMultipleAccountsInfo
	for (let i = 0; i < publicKeys.length; i += 100) {
		let publicKeysSlice = publicKeys.slice(i, i + 100);
		let cargoTypeAccts = await solanaReadConnection.getMultipleAccountsInfo(publicKeysSlice);
		for(let j=0; j < cargoTypeAccts.length; j++) {
			let cargoTypeDataExtra = cargoTypeAccts[j].data.subarray(110);
			let cargoTypeDataExtraBuff = BrowserBuffer.Buffer.Buffer.from(cargoTypeDataExtra);
			cargoTypeSizes[i + j] = cargoTypeDataExtraBuff.readUIntLE(0, 8);
		}
	}
	return cargoTypeSizes;
    }
    async function getResourceTokens() {
        mineItems = await sageProgram.account.mineItem.all();
        craftableItems = await craftingProgram.account.craftableItem.all();

	let cargoTypeSizes = await getCargoTypeSizes(cargoTypes);
        for (let resource of mineItems) {
            let cargoTypeIndex = cargoTypes.findIndex(item => item.account.mint.toString() === resource.account.mint.toString());
            let cargoName = (new TextDecoder().decode(new Uint8Array(resource.account.name)).replace(/\0/g, ''));
            let cargoSize = cargoTypeSizes[cargoTypeIndex];
            cargoItems.push({'name': cargoName, 'token': resource.account.mint.toString(), 'size': cargoSize});
        }
        for (let craftable of craftableItems) {
            let cargoTypeIndex = cargoTypes.findIndex(item => item.account.mint.toString() === craftable.account.mint.toString());
            let cargoName = (new TextDecoder().decode(new Uint8Array(craftable.account.namespace)).replace(/\0/g, ''));
            let cargoSize = cargoTypeSizes[cargoTypeIndex];
            cargoItems.push({'name': cargoName, 'token': craftable.account.mint.toString(), 'size': cargoSize});
        }
        cargoItems.sort(function (a, b) { return a.name.toUpperCase().localeCompare(b.name.toUpperCase()); });
    }

    async function getCraftRecipes() {
        let craftingDomain = sageGameAcct.account.crafting.domain;
        //const allCraftCategories = await craftingProgram.account.recipeCategory.all();
	//we need to select the correct craftingDomain:
        const allCraftCategories = await craftingProgram.account.recipeCategory.all([
			{
				memcmp: {
					offset: 9,
					bytes: craftingDomain.toBase58(),
				},
			},

        ]);
        let upgradeCategory = allCraftCategories.find(item => (new TextDecoder().decode(new Uint8Array(item.account.namespace)).replace(/\0/g, '')) === 'Upgrade');

        let statusBN = new BrowserAnchor.anchor.BN(2);
        let statusArr = statusBN.toTwos(64).toArrayLike(BrowserBuffer.Buffer.Buffer, "be", 2);
        let status58 = bs58.encode(statusArr);
        const allCraftRecipes = await craftingProgram.account.recipe.all([
            {
                memcmp: {
                    offset: 152,
                    bytes: status58,
                },
            },
            //we only select the recipes for the correct crafting domain
            {
		memcmp: {
			offset: 9,
			bytes: craftingDomain.toBase58(),
		},
            },
        ]);

	let publicKeys = [];
	let recipeDatas = [];
	for(let i=0; i < allCraftRecipes.length; i++) {
		publicKeys.push(allCraftRecipes[i].publicKey);
	}
	// 100 is the max data size of getMultipleAccountsInfo
	for (let i = 0; i < publicKeys.length; i += 100) {
		let publicKeysSlice = publicKeys.slice(i, i + 100);
		let recipeAcctInfos = await solanaReadConnection.getMultipleAccountsInfo(publicKeysSlice);
		for(let j=0; j < recipeAcctInfos.length; j++) {
			recipeDatas[i + j] = recipeAcctInfos[j].data.subarray(223);
		}
	}

	let recipeIdx = 0;
        for (let craftRecipe of allCraftRecipes) {
            let recipeName = (new TextDecoder().decode(new Uint8Array(craftRecipe.account.namespace)).replace(/\0/g, ''));
            let recipeInputOutput = [];
            let recipeData = recipeDatas[recipeIdx];
            let recipeIter = 0;
            while (recipeData.length >= 40) {
                let currIngredient = recipeData.subarray(0, 40);
                let ingredientDecoded = craftingProgram.coder.types.decode('RecipeInputsOutputs', currIngredient);
                recipeInputOutput.push({mint: ingredientDecoded.mint, amount: ingredientDecoded.amount.toNumber(), idx: recipeIter});
                recipeData = recipeData.subarray(40);
                recipeIter += 1;
            }
            if (craftRecipe.account.category.toString() === upgradeCategory.publicKey.toString()) {
                upgradeRecipes.push({'name': recipeName, 'publicKey': craftRecipe.publicKey, 'category': craftRecipe.account.category, 'domain': craftRecipe.account.domain, 'feeRecipient': craftRecipe.account.feeRecipient.key, 'duration': craftRecipe.account.duration.toNumber(), 'input': recipeInputOutput, 'output': []});
            } else {
                craftRecipes.push({'name': recipeName, 'publicKey': craftRecipe.publicKey, 'category': craftRecipe.account.category, 'domain': craftRecipe.account.domain, 'feeAmount': craftRecipe.account.feeAmount.toNumber()/100000000, 'feeRecipient': craftRecipe.account.feeRecipient.key, 'duration': craftRecipe.account.duration.toNumber(), 'input': recipeInputOutput.slice(0, -1), 'output': recipeInputOutput.slice(-1)[0]});
            }
            recipeIdx++;
        }
        upgradeRecipes.sort(function (a, b) { return a.name.toUpperCase().localeCompare(b.name.toUpperCase()); });
        craftRecipes.sort(function (a, b) { return a.name.toUpperCase().localeCompare(b.name.toUpperCase()); });
    }


	function createPDA(derived, derivedFrom1, derivedFrom2, fleet, send = true) {
			return new Promise(async resolve => {
					const keys = [{
							pubkey: userPublicKey,
							isSigner: true,
							isWritable: true
					}, {
							pubkey: derived,
							isSigner: false,
							isWritable: true
					}, {
							pubkey: derivedFrom1,
							isSigner: false,
							isWritable: false
					}, {
							pubkey: derivedFrom2,
							isSigner: false,
							isWritable: false
					}, {
							pubkey: solanaWeb3.SystemProgram.programId,
							isSigner: false,
							isWritable: false
					}, {
							pubkey: tokenProgramPK,
							isSigner: false,
							isWritable: false
					}];
					let tx = {instruction: new solanaWeb3.TransactionInstruction({
							keys: keys,
							programId: programPK,
							//data: []
					})}
					let txResult = tx
                    if (send) txResult = await txSignAndSend(tx, fleet, 'CreatePDA', 100);
					resolve(txResult);
			});
	}

    function buildXpAccounts(xpCategory, userXpAccounts, userXpAccountGroup) {
        return new Promise(async resolve => {
            let [userXpAccount] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
                [
                    BrowserBuffer.Buffer.Buffer.from("UserPointsAccount"),
                    xpCategory.toBuffer(),
                    userProfileAcct.toBuffer()
                ],
                pointsProgramId
            );

            let sageGamePointsCategory = Object.values(sageGameAcct.account.points).find(item => item.category.toString() == xpCategory.toString());

            userXpAccounts[userXpAccountGroup] = {
                userPointsAccount: userXpAccount,
                pointsCategory: xpCategory,
                pointsModifierAccount: sageGamePointsCategory.modifier
            }

            resolve(userXpAccounts[userXpAccountGroup]);
        });
    }

	async function getAccountInfo(fleetName, reason, params) {
		cLog(3, `${FleetTimeStamp(fleetName)} get ${reason}`);
		return await solanaReadConnection.getAccountInfo(params);
	}

    function getFleetState(fleetAcctInfo, fleet) {
        let remainingData = fleetAcctInfo.data.subarray(439);
        let fleetState = 'Unknown';
        let extra = null;

	if(fleet && fleet.exitWarpSubwarpPending) {
		let sector = null;
		if(fleet.exitWarpSubwarpPending == 1) sector = sageProgram.coder.types.decode('MoveWarp', remainingData.subarray(1));
		if(fleet.exitWarpSubwarpPending == 2) sector = sageProgram.coder.types.decode('MoveSubwarp', remainingData.subarray(1));
		if(sector.toSector) { // only continue if there is a target sector. Otherwise we will assume a RPC mismatch and reset the pending state (after waiting some time to be sure)
			fleetState = 'Idle';
			if(fleet.exitWarpSubwarpPending == 2) {
				fleet.exitSubwarpWillBurnFuel = sector.fuelExpenditure.toNumber();
			}
			extra = [sector.toSector[0].toNumber(), sector.toSector[1].toNumber()];
			return [fleetState, extra];
		} else {
			cLog(1, `${FleetTimeStamp(fleet.label)} Exit warp/subwarp was pending, but no target sector found, resetting pending state`);
			logError('Exit warp/subwarp was pending, but no target sector found, resetting pending state', fleet.label);
			fleet.exitWarpSubwarpPending = 0;
		}
	}

        switch(remainingData[0]) {
            case 0:
                fleetState = 'StarbaseLoadingBay';
                extra = sageProgram.coder.types.decode('StarbaseLoadingBay', remainingData.subarray(1));
                break;
            case 1: {
                fleetState = 'Idle';
                let sector = sageProgram.coder.types.decode('Idle', remainingData.subarray(1));
                extra = [sector.sector[0].toNumber(), sector.sector[1].toNumber()]
                break;
            }
            case 2:
                fleetState = 'MineAsteroid';
                extra = sageProgram.coder.types.decode('MineAsteroid', remainingData.subarray(1));
                break;
            case 3:
                fleetState = 'MoveWarp';
                extra = sageProgram.coder.types.decode('MoveWarp', remainingData.subarray(1));
                break;
            case 4:
                fleetState = 'MoveSubwarp';
                extra = sageProgram.coder.types.decode('MoveSubwarp', remainingData.subarray(1));
                break;
            case 5:
                fleetState = 'Respawn';
                break;
            case 6:
                fleetState = "StarbaseUpgrade";
                break;
            case 7:
                fleetState = "ReadyToExitWarp";
                break;
        }
        return [fleetState, extra];
    }

    function getBalanceChange(txResult, targetAcct) {
        let acctIdx = txResult.transaction.message.staticAccountKeys.findIndex(item => item.toString() === targetAcct);
        let preBalanceObj = txResult.meta.preTokenBalances.find(item => item.accountIndex === acctIdx);
        let preBalance = preBalanceObj && preBalanceObj.uiTokenAmount && preBalanceObj.uiTokenAmount.uiAmount ? preBalanceObj.uiTokenAmount.uiAmount : 0;
        let postBalanceObj = txResult.meta.postTokenBalances.find(item => item.accountIndex === acctIdx);
        let postBalance = postBalanceObj && postBalanceObj.uiTokenAmount && postBalanceObj.uiTokenAmount.uiAmount ? postBalanceObj.uiTokenAmount.uiAmount : 0;
        return {preBalance: preBalance, postBalance: postBalance}
    }

    function calculateMovementDistance(orig, dest) {
        return dest ? Math.sqrt((orig[0] - dest[0]) ** 2 + (orig[1] - dest[1]) ** 2) : 0
    }

    function calculateWarpTime(fleet, distance) {
        return fleet.warpSpeed > 0 ? distance / (fleet.warpSpeed / 1e6) : 0
    }

	function calcNextWarpPoint(warpRange, startCoords, endCoords) {
		const [startX, startY] = [Number(startCoords[0]), Number(startCoords[1])];
		const [endX, endY] = [Number(endCoords[0]), Number(endCoords[1])];
		const moveDist = calculateMovementDistance([startX, startY], [endX, endY]);
		const realWarpRange = warpRange / 100;
		const warpCount = realWarpRange > 0 ? moveDist / realWarpRange : 1;

		if(warpCount <= 1) return endCoords; // In range for single jump?

        // Calculate raw distance
        let slope = Math.abs(endX - startX) > 0 ? Math.abs((endY - startY) / (endX - startX)) : Math.abs((endY - startY));
        let dx = realWarpRange / Math.sqrt(slope ** 2 + 1);
        let dy = dx * slope;

        // Calculate the middle point destination
        let intX = startX > endX ? startX - parseInt(dx) : startX + parseInt(dx);
        let intY = startY > endY ? startY - parseInt(dy) : startY + parseInt(dy);
        let potentialCoords = [[intX,intY],[intX-1,intY-1],[intX-1,intY],[intX-1,intY+1],[intX,intY-1],[intX,intY+1],[intX+1,intY-1],[intX+1,intY],[intX+1,intY+1]];

        // Determine the optimal route based on total jumps and total distance
        let bestCoords = potentialCoords.reduce((best, val) => {
            let remainingDistOld = Math.sqrt((best[0] - endX) ** 2 + (best[1] - endY) ** 2);
            let remainingDistNew = Math.sqrt((val[0] - endX) ** 2 + (val[1] - endY) ** 2);
            let travelDistOld = Math.sqrt((startX - best[0]) ** 2 + (startY - best[1]) ** 2);
            let travelDistNew = Math.sqrt((startX - val[0]) ** 2 + (startY - val[1]) ** 2);
            if (globalSettings.subwarpShortDist && remainingDistOld > 0 && remainingDistNew < remainingDistOld && remainingDistNew < 1.5 && travelDistNew <= realWarpRange) {
                return val;
            } else if (globalSettings.subwarpShortDist && remainingDistOld < 1.5 && remainingDistNew >= 1.5) {
                return best;
            } else if (remainingDistNew <= realWarpRange && travelDistNew <= realWarpRange && remainingDistNew+travelDistNew < remainingDistOld+travelDistOld) {
                return val;
            } else if (remainingDistNew <= realWarpRange && travelDistNew <= realWarpRange && remainingDistOld > realWarpRange) {
                return val;
            } else if (remainingDistNew > realWarpRange && travelDistNew <= realWarpRange && remainingDistNew < remainingDistOld) {
                return val;
            } else if (travelDistOld <= realWarpRange) {
                return best;
            }
        });

		//Calculate and return waypoint coordinates
		return bestCoords;
	}

	function calcWarpFuelReq(fleet, startCoords, endCoords, warpSubwarp) {
		if(!CoordsValid(startCoords) || !CoordsValid(endCoords)) {
			cLog(4, `${FleetTimeStamp(fleet.label)} calcWarpFuelReq: Bad coords`, startCoords, endCoords);
			return 0;
		}
		if(CoordsEqual(startCoords, endCoords)) {
			cLog(4, `${FleetTimeStamp(fleet.label)} calcWarpFuelReq: Same coords`, startCoords, endCoords);
			return 0;
		}

		const [startX, startY] = [Number(startCoords[0]), Number(startCoords[1])];

		let jumps = 0;
		let fuelRequired = 0;
		let curWP = [startX, startY];

		while(!CoordsEqual(curWP, endCoords)) {
			const nextWP = calcNextWarpPoint(fleet.maxWarpDistance, curWP, endCoords);
			const distance = calculateMovementDistance(curWP, nextWP);
			fuelRequired += Math.ceil(distance * (fleet.warpFuelConsumptionRate / 100));
			curWP = nextWP;
			jumps++;
			if(warpSubwarp) {
				const distToTarget = calculateMovementDistance(curWP, endCoords);
				fuelRequired += calculateSubwarpFuelBurn(fleet, distToTarget);
				break;
			}
		};

		//cLog(4, `${FleetTimeStamp(fleet.label)} calcWarpFuelReq: ${fuelRequired} fuel over ${jumps} jumps`);
		return fuelRequired;
	};

    function calculateSubwarpTime(fleet, distance) {
        return fleet.subwarpSpeed > 0 ? distance / (fleet.subwarpSpeed / 1e6) : 0
    }

    function calculateSubwarpFuelBurn(fleet, distance) {
        return distance * (fleet.subwarpFuelConsumptionRate / 100)
    }

    function calculateMiningDuration(cargoCapacity, miningRate, resourceHardness, systemRichness) {
        return resourceHardness > 0 ? Math.ceil(cargoCapacity / (((miningRate / 10000) * (systemRichness / 100)) / (resourceHardness / 100))) : 0;
    }

    async function getSectorFromCoords(x, y) {
        return new Promise(async resolve => {
            let xBN = new BrowserAnchor.anchor.BN(x);
            let xArr = xBN.toTwos(64).toArrayLike(BrowserBuffer.Buffer.Buffer, "le", 8);
            let yBN = new BrowserAnchor.anchor.BN(y);
            let yArr = yBN.toTwos(64).toArrayLike(BrowserBuffer.Buffer.Buffer, "le", 8);
            let [sector] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
                [
                    BrowserBuffer.Buffer.Buffer.from("Sector"),
                    sageGameAcct.publicKey.toBuffer(),
                    xArr,
                    yArr
                ],
                sageProgramPK
            );
            resolve(sector);
        });
    }

    async function getStarbaseFromCoords(x, y, getLive = false) {
        return new Promise(async resolve => {
            let xBN = new BrowserAnchor.anchor.BN(x);
            let xArr = xBN.toTwos(64).toArrayLike(BrowserBuffer.Buffer.Buffer, "le", 8);
            let x58 = bs58.encode(xArr);
            let yBN = new BrowserAnchor.anchor.BN(y);
            let yArr = yBN.toTwos(64).toArrayLike(BrowserBuffer.Buffer.Buffer, "le", 8);
            let y58 = bs58.encode(yArr);

            let cachedStarbase = starbaseData.find(item => item.coords[0] == x && item.coords[1] == y);
            let starbase = cachedStarbase && cachedStarbase.starbase;
            let needUpdate = cachedStarbase && Date.now() - cachedStarbase.lastUpdated > 1000*60*60*24 ? true : false;

            if (!starbase || needUpdate || getLive) {
                [starbase] = await sageProgram.account.starbase.all([
                    {
                        memcmp: {
                            offset: 41,
                            bytes: x58
                        }
                    },
                    {
                        memcmp: {
                            offset: 49,
                            bytes: y58
                        }
                    },
                ]);
		//race-condition fixed: because of the previous "await", it is possible that we end up with two concurrent reads and two identical cache entries. So we need to make sure that an existing entry is always overwritten
		//also when expired entry is read again and just pushed to the array, find() will still find the expired first entry and not the updated one. This would lead to a broken cache. So again we need to overwrite the existing entry.
		let cachedStarbaseIdx = starbaseData.findIndex(item => item.coords[0] == x && item.coords[1] == y);
                if(cachedStarbaseIdx >= 0) {
                    starbaseData[cachedStarbaseIdx].lastUpdated = Date.now();
                    starbaseData[cachedStarbaseIdx].starbase = starbase;
                } else {
                    starbaseData.push({coords: [x,y], lastUpdated: Date.now(), starbase: starbase});
		}
            }

            resolve(starbase);
        });
    }

    async function getPlanetsFromCoords(x, y) {
        return new Promise(async resolve => {
            let xBN = new BrowserAnchor.anchor.BN(x);
            let xArr = xBN.toTwos(64).toArrayLike(BrowserBuffer.Buffer.Buffer, "le", 8);
            let x58 = bs58.encode(xArr);
            let yBN = new BrowserAnchor.anchor.BN(y);
            let yArr = yBN.toTwos(64).toArrayLike(BrowserBuffer.Buffer.Buffer, "le", 8);
            let y58 = bs58.encode(yArr);

            let cachedPlanet = planetData.find(item => item.coords[0] == x && item.coords[1] == y);
            let planets = cachedPlanet && cachedPlanet.planets;
            let needUpdate = cachedPlanet && Date.now() - cachedPlanet.lastUpdated > 1000*60*60*24 ? true : false;

            if (!planets || needUpdate) {
                planets = await sageProgram.account.planet.all([
                    {
                        memcmp: {
                            offset: 105,
                            bytes: x58
                        }
                    },
                    {
                        memcmp: {
                            offset: 113,
                            bytes: y58
                        }
                    },
                ]);
		//race-condition fixed: because of the previous "await", it is possible that we end up with two concurrent reads and two identical cache entries. So we need to make sure that an existing entry is always overwritten
		//also when expired entry is read again and just pushed to the array, find() will still find the expired first entry and not the updated one. This would lead to a broken cache. So again we need to overwrite the existing entry.
		let cachedPlanetIdx = planetData.findIndex(item => item.coords[0] == x && item.coords[1] == y);
                if(cachedPlanetIdx >= 0) {
                    planetData[cachedPlanetIdx].lastUpdated = Date.now();
                    planetData[cachedPlanetIdx].planets = planets;
                } else {
                    planetData.push({coords: [x,y], lastUpdated: Date.now(), planets: planets});
		}
            }

            resolve(planets);
        });
    }

    async function getMineableResourceFromPlanet(planet, mineItem) {
        let needUpdate = minableResourceData && minableResourceData.lastUpdated && Date.now() - minableResourceData.lastUpdated < 1000*60*60*24 ? false : true;

        if (needUpdate) {
            let mineableResources = await sageProgram.account.resource.all();
            minableResourceData = {lastUpdated: Date.now(), mineableResources: mineableResources};
        }

        let mineableResource = minableResourceData.mineableResources.find(item => item.account.location.toString() === planet && item.account.mineItem.toString() === mineItem);

        return mineableResource;
    }

    async function getStarbasePlayer(userProfile, starbase) {
        return new Promise(async resolve => {
            //starbasePlayerData
            let cachedStarbasePlayer = starbasePlayerData.find(item => item.userProfile == userProfile.toBase58() && item.starbase == starbase.toBase58());
            let starbasePlayer = cachedStarbasePlayer && cachedStarbasePlayer.starbasePlayer;
            let needUpdate = cachedStarbasePlayer && Date.now() - cachedStarbasePlayer.lastUpdated > 1000*60*60*24 ? true : false;

            if (!starbasePlayer || needUpdate) {
                [starbasePlayer] = await sageProgram.account.starbasePlayer.all([
                    {
                        memcmp: {
                            offset: 9,
                            bytes: userProfile.toBase58()
                        }
                    },
                    {
                        memcmp: {
                            offset: 73,
                            bytes: starbase.toBase58()
                        }
                    },
                ]);
		//race-condition fixed: because of the previous "await", it is possible that we end up with two concurrent reads and two identical cache entries. So we need to make sure that an existing entry is always overwritten
		//also when expired entry is read again and just pushed to the array, find() will still find the expired first entry and not the updated one. This would lead to a broken cache. So again we need to overwrite the existing entry.
		let cachedStarbasePlayerDataIdx = starbasePlayerData.findIndex(item => item.userProfile == userProfile.toBase58() && item.starbase == starbase.toBase58());
		if(cachedStarbasePlayerDataIdx >= 0) {
			starbasePlayerData[cachedStarbasePlayerDataIdx].lastUpdated = Date.now();
			starbasePlayerData[cachedStarbasePlayerDataIdx].starbasePlayer = starbasePlayer;
		} else {
                	starbasePlayerData.push({userProfile: userProfile.toBase58(), starbase: starbase.toBase58(), lastUpdated: Date.now(), starbasePlayer: starbasePlayer});
		}
            }

            resolve(starbasePlayer);
        });
    }

    async function getUserRedemptionAccount() {
        let currentDayIndex = Math.floor(Date.now() / 86400000);
        let currentDayIdxBN = new BrowserAnchor.anchor.BN(currentDayIndex);
        let currentDayIdxArr = currentDayIdxBN.toTwos(64).toArrayLike(BrowserBuffer.Buffer.Buffer, "le", 2);
        let currentDayIdx58 = bs58.encode(currentDayIdxArr);

        let [userRedemption] = await pointsStoreProgram.account.userRedemption.all([
            {
                memcmp: {
                    offset: 9,
                    bytes: userProfileAcct.toBase58(),
                },
            },
            {
                memcmp: {
                    offset: 145,
                    bytes: currentDayIdx58,
                },
            },
        ]);

        return userRedemption ? userRedemption.publicKey : false;
    }


	async function getFleetFuelToken(fleet) {
		const [token] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
			[
				fleet.fuelTank.toBuffer(),
				tokenProgramPK.toBuffer(),
				new solanaWeb3.PublicKey(fuelItem.token).toBuffer()
			],
			programPK
		);

		return token;
	}
	async function getFleetAmmoToken(fleet) {
		const [token] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
			[
				fleet.ammoBank.toBuffer(),
				tokenProgramPK.toBuffer(),
				sageGameAcct.account.mints.ammo.toBuffer()
			],
			programPK
		);

		return token;
	}
	async function getFleetCargoToken(fleet, tokenPK) {
		const [token] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
			[
				fleet.cargoHold.toBuffer(),
				tokenProgramPK.toBuffer(),
				tokenPK.toBuffer()
			],
			programPK
		);

		return token;
	}


    // localGetEpochInfo caches the last block height and estimates the current block height if the cached value has been requested recently.
    async function localGetEpochInfo(fleet) {
	const cachedValueExpires = 15000, useBlockTime = 450;
        let curTimestamp = Date.now(), localBlockHeight = 0, loopCounter = 0;
	// if another async call is already requesting the current blocktime, we wait for the result, but max 2.5 seconds (= 20 * 125ms)
        while (cachedEpochInfo.isUpdating && (curTimestamp - cachedEpochInfo.lastUpdated) > cachedValueExpires && loopCounter < 20) {
		await wait(125);
		loopCounter++;
		if(loopCounter >= 20 && cachedEpochInfo.isUpdating) cLog(1,`${FleetTimeStamp(fleet.label)} Concurrent read of block height took too long, forcing read`);
	}
        if ((curTimestamp - cachedEpochInfo.lastUpdated) > cachedValueExpires) {
		// We request the current block height. We use a try/catch block, so if something goes wrong, we can be sure that "isUpdating" is reset
		try {
			cachedEpochInfo.isUpdating = true;
			let {blockHeight: curBlockHeight} = await solanaReadConnection.getEpochInfo({ commitment: 'confirmed' });
			cachedEpochInfo.blockHeight = curBlockHeight;
			cachedEpochInfo.lastUpdated = Date.now(); // we must use the current time (instead of the previously saved timestamp), because it is possible that several re-reads occured (it would then lead to a block height that's way too high). Also it is possible that the while loop from above took some time.
			localBlockHeight = curBlockHeight;
			cLog(3,`${FleetTimeStamp(fleet.label)} using requested block height of`, localBlockHeight);
			cachedEpochInfo.isUpdating = false;
		} catch(error) {
			// something went wrong, we reset "isUpdating" and use the cached value
			localBlockHeight = cachedEpochInfo.blockHeight + Math.round((Date.now() - cachedEpochInfo.lastUpdated) / useBlockTime);
			cachedEpochInfo.isUpdating = false;
			cLog(1,`${FleetTimeStamp(fleet.label)} Uncaught error in localGetEpochInfo`, error);
		}
        } else {
		// We estimate the current block height and use the current timestamp for the calculation, because it is possible that the above while loop took some time. The average block time is 420ms, but just to be sure we use a little more (450ms), so a tx doesn't expire too early.
		localBlockHeight = cachedEpochInfo.blockHeight + Math.round((Date.now() - cachedEpochInfo.lastUpdated) / useBlockTime);
		cLog(3,`${FleetTimeStamp(fleet.label)} using estimated block height of`, localBlockHeight);
        }
        return localBlockHeight;
    }

let signatureStatusQueue = [];

function requestSignatureStatus(txHash) {
  return new Promise((resolve, reject) => {
	signatureStatusQueue.push({ txHash, resolve, reject });
  });
}

// The handler processes the queue and calls the corresponding callbacks with the result
async function signatureStatusHandler() {
	const currentHashes = signatureStatusQueue.splice(0, signatureStatusQueue.length);
	if (currentHashes.length > 0) {

		const txHashes = currentHashes.map(req => req.txHash);
		cLog(3,`Requesting`, currentHashes.length, `signature statuses at once`);

		try {
			const signatureStatuses = await solanaReadConnection.getSignatureStatuses(txHashes);
			//cLog(3,`Got signature results:`, signatureStatuses);
			for (let i = 0 ; i < currentHashes.length; i++) {
				const { resolve } = currentHashes[i];
				const signatureStatus = { value: signatureStatuses.value[i] };
				resolve(signatureStatus);
			}
		} catch(error) {
			// If something goes wrong, we reject each request. If a promise of the queue was already resolved in the "try" block, the reject does (correctly) nothing and won't throw an error
			logError('Error: Rejecting all signature checks - ' + error);
			for (const req of currentHashes) {
				req.reject(error);
			}
		}
	}
	setTimeout(() => { signatureStatusHandler(); }, 1000);
}
setTimeout(() => { signatureStatusHandler(); }, 1000);

async function sendAndConfirmTx(txSerialized, lastValidBlockHeight, txHash, fleet, opName) {
	let curBlockHeight = await localGetEpochInfo(fleet);
	let retryCount = 0;
	// loop until block height exceeded and give the RPC a little more time (20 blocks = ~8 seconds) to prevent race conditions (and take into account that the estimated block time can be slightly off). The loop doesn't need a wait time, because it is enforced by the signature queue (2 seconds)
        while (curBlockHeight <= lastValidBlockHeight + 20) {
		// we initially send the tx. Also we resend the tx every fourth loop (=~8 seconds), because: "With many transactions in the network queue, our initial send might get stuck behind a large backlog. While the blockhash is still valid, the transaction must still be seen and processed by validators. Continuously resending it can increase the chance that our transaction “bubbles up” to the front of processing queues.
		if((retryCount % 4) == 0) {
			txHash = await solanaWriteConnection.sendRawTransaction(txSerialized, {skipPreflight: true, maxRetries: 0, preflightCommitment: 'confirmed'});
			cLog(3,`${FleetTimeStamp(fleet.label)} <${opName}>`,(retryCount > 0 ? 'TRYING 🌐 ' : '') + 'txHash', txHash, `/ last valid block`, lastValidBlockHeight, `/ cur block`, curBlockHeight);
			if (!txHash) return {txHash, confirmation: {name: 'TransactionExpiredBlockheightExceededError'}};
		}
		try {
			const signatureStatus = await requestSignatureStatus(txHash);
			if (signatureStatus.value && ['confirmed','finalized'].includes(signatureStatus.value.confirmationStatus)) {
				cLog(3,`${FleetTimeStamp(fleet.label)} <${opName}> SIGNATURE FOUND ✅`);
				return {txHash, confirmation: signatureStatus};
			} else if (signatureStatus.value && signatureStatus.value.err) {
				cLog(3,`${FleetTimeStamp(fleet.label)} <${opName}> Err`,signatureStatus.value.err);
				return {txHash, confirmation: signatureStatus}
			}
		} catch(err) {
			cLog(1,`${FleetTimeStamp(fleet.label)} <${opName}> Signature exception:`,err);
		}
		curBlockHeight = await localGetEpochInfo(fleet); // todo: if for some reason the request of the block height takes a very long time (e.g. 20 seconds) and the block height is near the block limit, it is possible that the loop will exit without doing a final signature check.
		retryCount++;
	}
        return {txHash, confirmation: {name: 'TransactionExpiredBlockheightExceededError'}};
}



	function sisTxOriginal(ix, fleet, opName, priorityFeeMultiplier, extraSigner=false) {
		return new Promise(async resolve => {
			const fleetName = fleet ? fleet.label : 'unknown';
			sisContarTx(fleetName);
			let macroOpStart = Date.now();
			if(!priorityFeeMultiplier) priorityFeeMultiplier = 10; // globalSettings.lowPriorityFeeMultiplier;
			priorityFeeMultiplier = priorityFeeMultiplier / 100;

			let priorityFeeMin = 1;
			if(ix.constructor === Array) priorityFeeMin = Math.max(1, Math.min(globalSettings.minPriorityFeeForMultiIx, 50000) * 5);

			if(fleet.exitWarpSubwarpPending) {
				let exitWarpSubwarpTx = null;
				if(fleet.exitWarpSubwarpPending == 1) exitWarpSubwarpTx = await execExitWarp(fleet, true);
				else exitWarpSubwarpTx = await execExitSubwarp(fleet, true);
				if(ix.constructor === Array) {
					ix = [exitWarpSubwarpTx].concat(ix);
				} else {
					ix = [exitWarpSubwarpTx,ix];
				}
				fleet.exitWarpSubwarpPending = 0;
				fleet.exitSubwarpWillBurnFuel = 0;
			}

			let confirmed = false;
			let sisReenvios = 0;
			while (!confirmed) {
				//let tx = new solanaWeb3.Transaction();

				//the fee is applied to the default compute limit and it is in microLamports. The default compute limit is 200k and microLamports to Lamports is 1M, therefore: 1M / 200k = we need to multiply by 5
				//const priorityFee = currentFee ? Math.max(1, Math.ceil(priorityFeeMultiplier * currentFee * 5)) : 0;
				const priorityFee = currentFee ? Math.max(priorityFeeMin, Math.ceil(priorityFeeMultiplier * currentFee * 5)) : 0;

				cLog(4,`${FleetTimeStamp(fleetName)} <${opName}> 💳 Fee ${Math.ceil(priorityFee / 5)} lamp`);
                let instructions = [];
                let ixsJogo = [];
                if (ix.constructor === Array) {
					ix.forEach(item => ixsJogo.push(item.instruction))
				} else {
					ixsJogo.push(ix.instruction);
				}
                // [Sis-Ágora 1.3.1] blockhash, simulação (limite de CU) e taxa da rede em paralelo: uma ida ao RPC em vez de três
                const gravaveis = [...new Set(ixsJogo.flatMap(i => (i.keys || []).filter(k => k.isWritable).map(k => k.pubkey.toString())))];
                const usarRede = !!(globalSettings.automaticFee && priorityFee > 0);
                const [latestBH, limiteCU, rede] = await Promise.all([
                    solanaReadConnection.getLatestBlockhash('confirmed'),
                    sisLimiteCU(solanaReadConnection, ixsJogo, 'EkSnNWid2cvwEVnVx9aBqawnmiCNiDgp3gUdkDPTKN1N'), // replaceRecentBlockhash: o blockhash da simulação não importa
                    usarRede ? sisTaxaRede(solanaReadConnection, gravaveis) : Promise.resolve(0),
                ]);
                // piso = taxa paga agora nas mesmas contas (p75), limitado pelos tetos; só com taxa automática
                let taxaUsada = priorityFee;
                if (usarRede && rede > taxaUsada) {
                    let teto = Math.max(1, globalSettings.automaticFeeMax) * 5;
                    try { if (sisLocal && sisLocal.tetoTaxa >= 0) teto = Math.min(teto, Math.max(1, sisLocal.tetoTaxa) * 5); } catch (e) { }
                    taxaUsada = Math.min(rede, teto);
                    cLog(4, `${FleetTimeStamp(fleetName)} <${opName}> 💳 taxa da rede ${rede} µlamp/CU → usando ${taxaUsada}`);
                }
                // [1.3.2] mínimo/máximo de lamports do usuário (por tx, com o CU que a tx vai pedir) e teto local
                const cuEfetivo = limiteCU > 0 ? limiteCU : Math.min(1400000, 200000 * Math.max(1, ixsJogo.length));
                taxaUsada = sisAplicarLimitesTaxa(taxaUsada, cuEfetivo, sisReenvios);
                if (sisReenvios) cLog(2, `${FleetTimeStamp(fleetName)} <${opName}> 💳 reenvio ${sisReenvios}: taxa reforçada para ${Math.ceil(taxaUsada * cuEfetivo / 1e6)} lamports`);
                try { sisTaxaUltima = { base: priorityFee, rede, usada: taxaUsada, cu: limiteCU, cuEfetivo, reenvios: sisReenvios, em: Date.now() }; sisAtualizarChipTaxa(); } catch (e) { }
                if (limiteCU > 0) instructions.push(solanaWeb3.ComputeBudgetProgram.setComputeUnitLimit({units: limiteCU}));
                if (taxaUsada > 0) instructions.push(solanaWeb3.ComputeBudgetProgram.setComputeUnitPrice({microLamports: taxaUsada}));
                ixsJogo.forEach(i => instructions.push(i));
                let messageV0 = new solanaWeb3.TransactionMessage({
                    payerKey: userPublicKey,
                    recentBlockhash: latestBH.blockhash,
                    instructions,
                }).compileToV0Message(addressLookupTables);
                let tx = new solanaWeb3.VersionedTransaction(messageV0);
                if (extraSigner) tx.sign([extraSigner]);
				let txSigned = null;
                cLog(4,`${FleetTimeStamp(fleetName)} <${opName}> tx: `, tx);

		const signStart = Date.now();

                try {
                    if(customKeypair) {
                        tx.sign([customKeypair]);
                        txSigned = [tx];
                    } else if (typeof solflare === 'undefined') {
                        txSigned = phantom && phantom.solana ? await phantom.solana.signAllTransactions([tx]) : solana.signAllTransactions([tx]);
                    } else {
                        txSigned = await solflare.signAllTransactions([tx]);
                    }
                } catch (error1) {
                    cLog(2,`${FleetTimeStamp(fleetName)} <${opName}> Wallet extension error`, error1);
                    await wait(1000);
                    if(customKeypair) {
                        tx.sign([customKeypair]);
                        txSigned = [tx];
                    } else if (typeof solflare === 'undefined') {
                        txSigned = phantom && phantom.solana ? await phantom.solana.signAllTransactions([tx]) : solana.signAllTransactions([tx]);
                    } else {
                        txSigned = await solflare.signAllTransactions([tx]);
                    }
                }

		const signMsTaken = Date.now() - signStart;
		if(signMsTaken>2000) cLog(2,`${FleetTimeStamp(fleetName)} <${opName}> WARNING: Signing of tx took`,signMsTaken,`milliseconds`);
		{
			const tEl = document.getElementById('assist-modal-time');
			if (tEl) {
				const sp = celula('span', (signMsTaken/1000).toFixed(1) + 's');
				sp.style.color = signMsTaken >= 2000 ? (signMsTaken >= 20000 ? 'Red' : 'Yellow') : 'inherit';
				tEl.replaceChildren(document.createTextNode('Last sign time: '), sp);
			}
		}

                cLog(4,`${FleetTimeStamp(fleetName)} <${opName}> txSigned: `, txSigned);
				let txSerialized = await txSigned[0].serialize();
                cLog(4,`${FleetTimeStamp(fleetName)} <${opName}> txSerialized: `, txSerialized);

				let microOpStart = Date.now();
				cLog(2,`${FleetTimeStamp(fleetName)} <${opName}> SEND ➡️ lastValidBlockHeight: `, latestBH.lastValidBlockHeight);
                // Adding a 25 block buffer before considering a transaction expired
				let response = await sendAndConfirmTx(txSerialized, latestBH.lastValidBlockHeight, null, fleet, opName);
				let txHash = response.txHash;
				let confirmation = response.confirmation;
				//let txResult = txHash ? await solanaReadConnection.getTransaction(txHash, {commitment: 'confirmed', preflightCommitment: 'confirmed', maxSupportedTransactionVersion: 1}) : undefined;
				let txResult = txHash ? await solanaReadConnection.getTransaction(txHash, {commitment: 'confirmed', maxSupportedTransactionVersion: 1}) : undefined;
                if ((confirmation.value && confirmation.value.err && confirmation.value.err.InstructionError) || (txResult && txResult.meta && txResult.meta.err && txResult.meta.err.InstructionError)) {
                    if (globalErrorTracker.firstErrorTime === 0) globalErrorTracker.firstErrorTime = Date.now();
                    if (Date.now() < globalErrorTracker.firstErrorTime + 600000) {
                        globalErrorTracker.errorCount++
                    } else {
                        globalErrorTracker.firstErrorTime = Date.now();
                        globalErrorTracker.errorCount = 1;
                    }
                    updateFleetState(fleet, 'ERROR: Ix Error');
                    cLog(2,`${FleetTimeStamp(fleetName)} <${opName}> ERROR ❌ The instruction resulted in an error.`);
                    let ixError = txResult && txResult.meta && txResult.meta.logMessages ? txResult.meta.logMessages : 'Unknown';
                    console.log(FleetTimeStamp(fleetName), ' txResult.logMessages: ', ixError);
                    logError('ix error: ' + ixError, fleetName);
                }

				const confirmationTimeStr = `${Date.now() - microOpStart}ms`;

				if (confirmation && confirmation.name == 'TransactionExpiredBlockheightExceededError' && !txResult) {
					cLog(2,`${FleetTimeStamp(fleetName)} <${opName}> CONFIRM ❌ ${confirmationTimeStr}`);
					cLog(2,`${FleetTimeStamp(fleetName)} <${opName}> RESEND 🔂`);
					await alterStats('Txs Resent',opName,(Date.now() - macroOpStart)/1000,'Seconds',1); //statsadd
					await alterFees(-1, opName); //autofee
					sisReenvios++;
					continue; //retart loop to try again
				}

				let tryCount = 1;
				if (!confirmation.name) {
					if(!txResult) cLog(3,`${FleetTimeStamp(fleetName)} <${opName}> Polling transaction until successful`);
					while (!txResult) {
						tryCount++;
						if(tryCount >= 130) {
							// couldn't find the transaction, it is possible a block re-org happened, so try to send the tx again
							cLog(1,`${FleetTimeStamp(fleetName)} <${opName}> No transaction found after ${tryCount} tries, possible block re-org, resending the tx`);
							break;
						}
						if((tryCount % 10) == 0) { cLog(3,`${FleetTimeStamp(fleetName)} <${opName}> Still polling the transaction`); }
						//txResult = await solanaReadConnection.getTransaction(txHash, {commitment: 'confirmed', preflightCommitment: 'confirmed', maxSupportedTransactionVersion: 1});
						txResult = await solanaReadConnection.getTransaction(txHash, {commitment: 'confirmed', maxSupportedTransactionVersion: 1});
						if(!txResult) await wait(1000);
					}
					if(tryCount >= 130) {
						continue;
					}
				}

				if(tryCount > 1) cLog(3, `${FleetTimeStamp(fleetName)} Got txResult in ${tryCount} tries`, txResult);
				cLog(4, `${FleetTimeStamp(fleetName)} txResult`, txResult);
				cLog(2,`${FleetTimeStamp(fleetName)} <${opName}> CONFIRM ✅ ${confirmationTimeStr}`);
				confirmed = true;

				const fullMsTaken = Date.now() - macroOpStart;
				sisMedirTx(ix.constructor === Array ? ix.length : 1, fullMsTaken, fleet, txResult);
				const secondsTaken = Math.round(fullMsTaken / 1000);
				cLog(1,`${FleetTimeStamp(fleetName)} <${opName}> Completed 🏁 ${secondsTaken}s`);

				await alterStats('SOL Fees',undefined,txResult.meta.fee*0.000000001,'SOL',7); // undefined name => only totals tracked //statsadd
				let statGroup = ((confirmation && confirmation.value && confirmation.value.err && confirmation.value.err.InstructionError) || (txResult && txResult.meta && txResult.meta.err && txResult.meta.err.InstructionError)) ? 'Txs IxErrors' : 'Txs Confirmed'; //statsadd
				await alterStats(statGroup,opName,fullMsTaken/1000,'Seconds',1); //statsadd
				await alterFees(fullMsTaken/1000, opName); //autofee

				resolve(txResult);
			}
		});
	}

    async function execScan(fleet, fleetCoords) {
        return new Promise(async resolve => {
            let sector = await getSectorFromCoords(fleetCoords[0], fleetCoords[1]);
            try {
                let sectorExists = await sageProgram.account.sector.fetch(sector);
            } catch (err) {
                let txDiscover = { instruction: await sageProgram.methods.discoverSector({coordinates: [new BrowserAnchor.anchor.BN(fleetCoords[0]), new BrowserAnchor.anchor.BN(fleetCoords[1])], keyIndex: new BrowserAnchor.anchor.BN(userProfileKeyIdx)}).accountsStrict({
                    gameAccountsFleetAndOwner: {
                        gameFleetAndOwner: {
                            fleetAndOwner: {
                                fleet: fleet.publicKey,
                                owningProfile: userProfileAcct,
                                owningProfileFaction: userProfileFactionAcct.publicKey,
                                key: userPublicKey
                            },
                            gameId: sageGameAcct.publicKey
                        },
                        gameState: sageGameAcct.account.gameState
                    },
                    funder: userPublicKey,
                    sector: sector,
                    systemProgram: solanaWeb3.SystemProgram.programId
                }).instruction()}
                updateFleetState(fleet, `Scanning [${TimeToStr(new Date(Date.now()))}]`);
                let txResult = await txSignAndSend(txDiscover, fleet, 'SCAN', 100);
            }

			let sduCargoTypeAcct = cargoTypes.find(item => item.account.mint.toString() == sduItem.token);
			let foodCargoTypeAcct = cargoTypes.find(item => item.account.mint.toString() == sageGameAcct.account.mints.food);
			let [sduTokenFrom] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
				[
						sageSDUTrackerAcct.account.signer.toBuffer(),
						tokenProgramPK.toBuffer(),
						new solanaWeb3.PublicKey(sduItem.token).toBuffer()
				],
				programPK
			);

            let tx = { instruction: await sageProgram.methods.scanForSurveyDataUnits({keyIndex: new BrowserAnchor.anchor.BN(userProfileKeyIdx)}).accountsStrict({
                gameAccountsFleetAndOwner: {
                    gameFleetAndOwner: {
                        fleetAndOwner: {
                            fleet: fleet.publicKey, // sageProgram.fleet.publicKey
                            owningProfile: userProfileAcct, // from API
                            owningProfileFaction: userProfileFactionAcct.publicKey,
                            key: userPublicKey // wallet.publicKey
                        },
                        gameId: sageGameAcct.publicKey // sageProgram.game.publicKey
                    },
                    gameState: sageGameAcct.account.gameState // sageProgram.game.gameState
                },
                sector: sector,
                surveyDataUnitTracker: sageSDUTrackerAcct.publicKey, // sageProgram.SurveyDataUnitTracker.publicKey
                surveyDataUnitTrackerSigner: sageSDUTrackerAcct.account.signer, // sageProgram.SurveyDataUnitTracker.signer
                cargoHold: fleet.cargoHold, // sageProgram.fleet.cargoHold
                sduCargoType: sduCargoTypeAcct.publicKey, // cargoProgram.cargoType - memcmp offset 41 'SDUsgfSZaDhhZ76U3ZgvtFiXsfnHbf2VrzYxjBZ5YbM'
                resourceCargoType: foodCargoTypeAcct.publicKey, // cargoProgram.cargoType - memcmp offset 41 'tooLsNYLiVqzg8o4m3L2Uetbn62mvMWRqkog6PQeYKL'
                cargoStatsDefinition: sageGameAcct.account.cargo.statsDefinition, // sageProgram.game.cargo
                sduTokenFrom: sduTokenFrom, // calculated
                sduTokenTo: fleet.sduToken,
                resourceTokenFrom: fleet.foodToken,
                resourceMint: sageGameAcct.account.mints.food, // sageProgram.game.repairKit
                dataRunningXpAccounts: userXpAccounts.userDataRunningXpAccounts,
                councilRankXpAccounts: userXpAccounts.userCouncilRankXpAccounts,
                progressionConfig: progressionConfigAcct,
                pointsProgram: pointsProgramId,
                cargoProgram: cargoProgramPK, // static
                tokenProgram: tokenProgramPK, // static
                recentSlothashes: new solanaWeb3.PublicKey('SysvarS1otHashes111111111111111111111111111'), // static
                instructionsSysvar: new solanaWeb3.PublicKey('Sysvar1nstructions1111111111111111111111111') // static
            }).instruction()}

            updateFleetState(fleet, `Scanning [${TimeToStr(new Date(Date.now()))}]`);

            let txResult = await txSignAndSend(tx, fleet, 'SCAN', 100);

            resolve(txResult);
        });
    }

	async function execSubwarp(fleet, destX, destY, moveTime) {
		return new Promise(async resolve => {
			let tx = { instruction: await sageProgram.methods.startSubwarp({keyIndex: new BrowserAnchor.anchor.BN(userProfileKeyIdx), toSector: [new BrowserAnchor.anchor.BN(destX), new BrowserAnchor.anchor.BN(destY)]}).accountsStrict({
				gameAccountsFleetAndOwner: {
					gameFleetAndOwner: {
						fleetAndOwner: {
							fleet: fleet.publicKey,
							owningProfile: userProfileAcct,
							owningProfileFaction: userProfileFactionAcct.publicKey,
							key: userPublicKey
						},
						gameId: sageGameAcct.publicKey
					},
					gameState: sageGameAcct.account.gameState
				},
				}).instruction()
			}

			const coordStr = `[${destX},${destY}]`;
			cLog(1,`${FleetTimeStamp(fleet.label)} Subwarping to ${coordStr}`);
			updateFleetState(fleet, 'Subwarping');

			let txResult = await txSignAndSend(tx, fleet, 'SUBWARP', 10);

			const travelEndTime = TimeToStr(new Date(Date.now()+(moveTime * 1000)));
			const newFleetState = `Subwarp ${coordStr} ${travelEndTime}`;
			updateFleetState(fleet, newFleetState);

			resolve(txResult);
		});
	}

    async function execExitSubwarp(fleet, returnTx) {
        return new Promise(async resolve => {
            let fuelCargoTypeAcct = cargoTypes.find(item => item.account.mint.toString() == sageGameAcct.account.mints.fuel);
            let tx = { instruction: await sageProgram.methods.fleetStateHandler().accountsStrict({
                fleet: fleet.publicKey
            }).remainingAccounts([
                {
                    pubkey: userProfileAcct,
                    isSigner: false,
                    isWritable: true
                },
                {
                    pubkey: fleet.fuelTank,
                    isSigner: false,
                    isWritable: true
                },
                {
                    pubkey: fuelCargoTypeAcct.publicKey,
                    isSigner: false,
                    isWritable: false
                },
                {
                    pubkey: sageGameAcct.account.cargo.statsDefinition,
                    isSigner: false,
                    isWritable: false
                },
                {
                    pubkey: fleet.fuelToken,
                    isSigner: false,
                    isWritable: true
                },
                {
                    pubkey: sageGameAcct.account.mints.fuel,
                    isSigner: false,
                    isWritable: true
                },
                {
                    pubkey: userXpAccounts.userPilotingXpAccounts.userPointsAccount,
                    isSigner: false,
                    isWritable: true
                },
                {
                    pubkey: userXpAccounts.userPilotingXpAccounts.pointsCategory,
                    isSigner: false,
                    isWritable: false
                },
                {
                    pubkey: userXpAccounts.userPilotingXpAccounts.pointsModifierAccount,
                    isSigner: false,
                    isWritable: false
                },
                {
                    pubkey: userXpAccounts.userCouncilRankXpAccounts.userPointsAccount,
                    isSigner: false,
                    isWritable: true
                },
                {
                    pubkey: userXpAccounts.userCouncilRankXpAccounts.pointsCategory,
                    isSigner: false,
                    isWritable: false
                },
                {
                    pubkey: userXpAccounts.userCouncilRankXpAccounts.pointsModifierAccount,
                    isSigner: false,
                    isWritable: false
                },
                {
                    pubkey: progressionConfigAcct,
                    isSigner: false,
                    isWritable: false
                },
                {
                    pubkey: sageGameAcct.publicKey,
                    isSigner: false,
                    isWritable: false
                },
                {
                    pubkey: pointsProgramId,
                    isSigner: false,
                    isWritable: false
                },
                {
                    pubkey: cargoProgramPK,
                    isSigner: false,
                    isWritable: false
                },
                {
                    pubkey: tokenProgramPK,
                    isSigner: false,
                    isWritable: false
                },
            ]).instruction()}

			cLog(1,`${FleetTimeStamp(fleet.label)} Exiting Subwarp`);
			updateFleetState(fleet, 'Exiting Subwarp');

			//let txResult = await txSignAndSend(tx, fleet, 'EXIT SUBWARP', 100);
			let txResult;
			if(returnTx) {
				txResult = tx;
			} else {
				txResult = await txSignAndSend(tx, fleet, 'EXIT SUBWARP', 100);
			}

			cLog(1,`${FleetTimeStamp(fleet.label)} Idle 💤`);
			updateFleetState(fleet, 'Idle');

			resolve(txResult);
		});
	}

    async function execWarp(fleet, destX, destY, moveTime) {
        return new Promise(async resolve => {
			let fuelCargoTypeAcct = cargoTypes.find(item => item.account.mint.toString() == sageGameAcct.account.mints.fuel);
            let tx = { instruction: await sageProgram.methods.warpToCoordinate({keyIndex: new BrowserAnchor.anchor.BN(userProfileKeyIdx), toSector: [new BrowserAnchor.anchor.BN(destX), new BrowserAnchor.anchor.BN(destY)]}).accountsStrict({
                gameAccountsFleetAndOwner: {
                    gameFleetAndOwner: {
                        fleetAndOwner: {
                            fleet: fleet.publicKey,
                            owningProfile: userProfileAcct,
                            owningProfileFaction: userProfileFactionAcct.publicKey,
                            key: userPublicKey
                        },
                        gameId: sageGameAcct.publicKey
                    },
                    gameState: sageGameAcct.account.gameState
                },
                fuelTank: fleet.fuelTank,
                cargoType: fuelCargoTypeAcct.publicKey,
                statsDefinition: sageGameAcct.account.cargo.statsDefinition,
                tokenFrom: fleet.fuelToken,
                tokenMint: sageGameAcct.account.mints.fuel,
                cargoProgram: cargoProgramPK,
                tokenProgram: tokenProgramPK
            }).instruction()}

            const coordStr = `[${destX},${destY}]`;
            cLog(1,`${FleetTimeStamp(fleet.label)} Warping to ${coordStr}`);
            updateFleetState(fleet, 'Warping');

            let txResult = await txSignAndSend(tx, fleet, 'WARP', 100);

            const travelEndTime = TimeToStr(new Date(Date.now()+(moveTime * 1000 + 10000)));
            const newFleetState = `Warp ${coordStr} ${travelEndTime}`;
            updateFleetState(fleet, newFleetState);

            fleet.warpCoolDownFinish = Date.now() + fleet.warpCooldown * 1000 + 2000;

            resolve({txResult, warpCooldownFinished: fleet.warpCoolDownFinish});
        });
    }

    async function execExitWarp(fleet, returnTx) {
        return new Promise(async resolve => {
            let tx = { instruction: await sageProgram.methods.fleetStateHandler().accountsStrict({
                fleet: fleet.publicKey
            }).remainingAccounts([
                {
                    pubkey: userXpAccounts.userPilotingXpAccounts.userPointsAccount,
                    isSigner: false,
                    isWritable: true
                },
                {
                    pubkey: userXpAccounts.userPilotingXpAccounts.pointsCategory,
                    isSigner: false,
                    isWritable: false
                },
                {
                    pubkey: userXpAccounts.userPilotingXpAccounts.pointsModifierAccount,
                    isSigner: false,
                    isWritable: false
                },
                {
                    pubkey: userXpAccounts.userCouncilRankXpAccounts.userPointsAccount,
                    isSigner: false,
                    isWritable: true
                },
                {
                    pubkey: userXpAccounts.userCouncilRankXpAccounts.pointsCategory,
                    isSigner: false,
                    isWritable: false
                },
                {
                    pubkey: userXpAccounts.userCouncilRankXpAccounts.pointsModifierAccount,
                    isSigner: false,
                    isWritable: false
                },
                {
                    pubkey: userProfileAcct,
                    isSigner: false,
                    isWritable: false
                },
                {
                    pubkey: progressionConfigAcct,
                    isSigner: false,
                    isWritable: false
                },
                {
                    pubkey: sageGameAcct.publicKey,
                    isSigner: false,
                    isWritable: false
                },
                {
                    pubkey: pointsProgramId,
                    isSigner: false,
                    isWritable: false
                },
            ]).instruction()}

            cLog(1,`${FleetTimeStamp(fleet.label)} Exiting Warp`);
            updateFleetState(fleet, 'Exiting Warp');

            //let txResult = await txSignAndSend(tx, fleet, 'EXIT WARP', 10);
            let txResult;
            if(returnTx) {
            	txResult = tx;
            } else {
            	txResult = await txSignAndSend(tx, fleet, 'EXIT WARP', 10);
            }

            cLog(1,`${FleetTimeStamp(fleet.label)} Idle 💤`);
            updateFleetState(fleet, 'Idle');

            resolve(txResult);
        });
    }

    async function execRegisterStarbasePlayer(fleet, dockCoords) {
        let starbaseX = dockCoords.split(',')[0].trim();
        let starbaseY = dockCoords.split(',')[1].trim();
        let starbase = await getStarbaseFromCoords(starbaseX, starbaseY);

        let [sagePlayerProfile] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
			[
				BrowserBuffer.Buffer.Buffer.from("sage_player_profile"),
				userProfileAcct.toBuffer(),
                sageGameAcct.publicKey.toBuffer()
			],
			sageProgramPK
		);

        const bumpArrBuff = new ArrayBuffer(2)
        const bumpData = new DataView(bumpArrBuff);
        bumpData.setUint16(0, starbase.seqId, !0);
        let [starbasePlayer] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
			[
				BrowserBuffer.Buffer.Buffer.from("starbase_player"),
				starbase.publicKey.toBuffer(),
                sagePlayerProfile.toBuffer(),
                new Uint8Array(bumpData.buffer)
			],
			sageProgramPK
		);

        let tx = { instruction: await sageProgram.methods.registerStarbasePlayer().accountsStrict({
            profileFaction: userProfileFactionAcct.publicKey,
            funder: userPublicKey,
            gameAccounts: {
                gameId: sageGameAcct.publicKey,
                gameState: sageGameAcct.account.gameState
            },
            sagePlayerProfile: sagePlayerProfile,
            starbase: starbase.publicKey,
            starbasePlayer: starbasePlayer,
            systemProgram: solanaWeb3.SystemProgram.programId
        }).instruction()}
        await txSignAndSend(tx, fleet, 'Register Starbase Player');

        return starbasePlayer;
    }

    async function execDock(fleet, dockCoords, returnTx) {
        return new Promise(async resolve => {
            let starbaseX = dockCoords.split(',')[0].trim();
            let starbaseY = dockCoords.split(',')[1].trim();
            let starbase = await getStarbaseFromCoords(starbaseX, starbaseY);
            let starbasePlayer = await getStarbasePlayer(userProfileAcct,starbase.publicKey);
            starbasePlayer = starbasePlayer ? starbasePlayer.publicKey : await execRegisterStarbasePlayer(fleet, dockCoords);
            let tx = { instruction: await sageProgram.methods.idleToLoadingBay(new BrowserAnchor.anchor.BN(userProfileKeyIdx)).accountsStrict({
                gameAccountsFleetAndOwner: {
                    gameFleetAndOwner: {
                        fleetAndOwner: {
                            fleet: fleet.publicKey,
                            owningProfile: userProfileAcct,
                            owningProfileFaction: userProfileFactionAcct.publicKey,
                            key: userPublicKey
                        },
                        gameId: sageGameAcct.publicKey
                    },
                    gameState: sageGameAcct.account.gameState
                },
                starbaseAndStarbasePlayer: {
                    starbase: starbase.publicKey,
                    starbasePlayer: starbasePlayer
                }
            }).instruction()}

            cLog(1,`${FleetTimeStamp(fleet.label)} Docking`);
            updateFleetState(fleet, 'Docking');

            let txResult;
            if(returnTx) {
	        txResult = tx;
            } else {
                txResult = await txSignAndSend(tx, fleet, 'DOCK', 10);
            }
            //let txResult = await txSignAndSend(tx, fleet, 'DOCK', 10);

            cLog(1,`${FleetTimeStamp(fleet.label)} Docked`);
            updateFleetState(fleet, 'Docked');

            resolve(txResult);
        });
    }

	async function execUndock(fleet, dockCoords, returnTx) {
			return new Promise(async resolve => {
					let starbaseX = dockCoords.split(',')[0].trim();
					let starbaseY = dockCoords.split(',')[1].trim();
					let starbase = await getStarbaseFromCoords(starbaseX, starbaseY);
					let starbasePlayer = await getStarbasePlayer(userProfileAcct,starbase.publicKey);
					let tx = { instruction: await sageProgram.methods.loadingBayToIdle(new BrowserAnchor.anchor.BN(userProfileKeyIdx)).accountsStrict({
							gameAccountsFleetAndOwner: {
									gameFleetAndOwner: {
											fleetAndOwner: {
													fleet: fleet.publicKey,
													owningProfile: userProfileAcct,
													owningProfileFaction: userProfileFactionAcct.publicKey,
													key: userPublicKey
											},
											gameId: sageGameAcct.publicKey
									},
									gameState: sageGameAcct.account.gameState
							},
							starbaseAndStarbasePlayer: {
									starbase: starbase.publicKey,
									starbasePlayer: starbasePlayer.publicKey
							}
					}).remainingAccounts([{
							pubkey: starbase.publicKey,
							isSigner: false,
							isWritable: false
					}]).instruction()}

					cLog(1,`${FleetTimeStamp(fleet.label)} Undocking`);
					updateFleetState(fleet, 'Undocking');

					let txResult;
					if(returnTx) {
						txResult = tx;
					} else {
						txResult = await txSignAndSend(tx, fleet, 'UNDOCK', 10);
					}
					//let txResult = await txSignAndSend(tx, fleet, 'UNDOCK', 10);

					//await wait(2000);
					updateFleetState(fleet, 'Idle');

					resolve(txResult);
			});
	}

	async function execStartupUndock(i, assignment) {
		const fleet = userFleets[i];
		cLog(1,`${FleetTimeStamp(fleet.label)} Undock ${assignment} Startup`);

		if(assignment == 'Transport' || assignment == 'Mine') {
			const fleetAcctInfo = await solanaReadConnection.getAccountInfo(fleet.publicKey);
			const [fleetState, extra] = getFleetState(fleetAcctInfo, fleet);
			if (fleetState === 'StarbaseLoadingBay') {
				const starbase = await sageProgram.account.starbase.fetch(extra.starbase);
				const coords = starbase.sector[0].toNumber() + ',' + starbase.sector[1].toNumber();
				await execUndock(fleet, coords);
			}
		}
		else if(assignment == 'Scan') {
			//Try undocking from Starbase
			await execUndock(fleet, fleet.starbaseCoord);

			//Make sure all supplies are topped off
			const fleetsCoords = [fleet.starbaseCoord.split(',')[0].trim(), fleet.starbaseCoord.split(',')[1].trim()];
			await handleResupply(i, fleetsCoords);
		}

		cLog(1,`${FleetTimeStamp(fleet.label)} Undock Startup Complete`);
	}

	async function execCreateCargoPod(fleet, dockCoords) {
		let starbaseX = dockCoords.split(',')[0].trim();
		let starbaseY = dockCoords.split(',')[1].trim();
		let starbase = await getStarbaseFromCoords(starbaseX, starbaseY);
		let starbasePlayer = await getStarbasePlayer(userProfileAcct,starbase.publicKey);
		let cargoPodData = {
			keyIndex: new BrowserAnchor.anchor.BN(userProfileKeyIdx),
			podSeeds: Array.from(solanaWeb3.Keypair.generate().publicKey.toBuffer())
		}

		let [cargoPod] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
			[
				BrowserBuffer.Buffer.Buffer.from("cargo_pod"),
				BrowserBuffer.Buffer.Buffer.from(cargoPodData.podSeeds),
			],
			cargoProgramPK
		);

        let tx = { instruction: await sageProgram.methods.createCargoPod(cargoPodData).accountsStrict({
            funder: userPublicKey,
            starbaseAndStarbasePlayer: {
                starbase: starbase.publicKey,
                starbasePlayer: starbasePlayer.publicKey
            },
            gameAccountsAndProfile: {
                gameAndProfileAndFaction: {
                    key: userPublicKey,
                    profile: userProfileAcct,
                    profileFaction: userProfileFactionAcct.publicKey,
                    gameId: sageGameAcct.publicKey
                },
                gameState: sageGameAcct.account.gameState
            },
            cargoPod: cargoPod,
            cargoStatsDefinition: sageGameAcct.account.cargo.statsDefinition,
            cargoProgram: cargoProgramPK,
            systemProgram: solanaWeb3.SystemProgram.programId
        }).instruction(),
        signers: [userPublicKey,userPublicKey]
        }

		await txSignAndSend(tx, fleet, 'Create CargoPod');

		return cargoPod;
	}

	async function execCargoFromFleetToStarbase(fleet, fleetCargoPod, tokenMint, dockCoords, amount, returnTx) {
			return new Promise(async resolve => {
					let starbaseX = dockCoords.split(',')[0].trim();
					let starbaseY = dockCoords.split(',')[1].trim();
					let starbase = await getStarbaseFromCoords(starbaseX, starbaseY);
					let starbasePlayer = await getStarbasePlayer(userProfileAcct,starbase.publicKey);
					let starbasePlayerCargoHolds = await cargoProgram.account.cargoPod.all([
							{
									memcmp: {
											offset: 41,
											bytes: starbasePlayer.publicKey.toBase58(),
									},
							},
					]);

					let starbasePlayerCargoHold = starbasePlayerCargoHolds.find(item => item.account.openTokenAccounts > 0);
					starbasePlayerCargoHold = starbasePlayerCargoHold ? starbasePlayerCargoHold.publicKey : starbasePlayerCargoHolds.length > 0 ? starbasePlayerCargoHolds[0].publicKey : await execCreateCargoPod(fleet, dockCoords);

					if(!fleet.state.includes('ERROR')) {
						await influxStarbaseCargoHold(starbaseX,starbaseY,starbasePlayerCargoHold);
					}

					let [starbaseCargoToken] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
							[
									starbasePlayerCargoHold.toBuffer(),
									tokenProgramPK.toBuffer(),
									new solanaWeb3.PublicKey(tokenMint).toBuffer()
							],
							programPK
					);
					let [fleetResourceToken] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
							[
									fleetCargoPod.toBuffer(),
									tokenProgramPK.toBuffer(),
									new solanaWeb3.PublicKey(tokenMint).toBuffer()
							],
							programPK
					);
					let fleetCurrentPod = await solanaReadConnection.getParsedTokenAccountsByOwner(fleetCargoPod, {programId: tokenProgramPK});
					let currentResource = fleetCurrentPod.value.find(item => item.account.data.parsed.info.mint === tokenMint);
					let fleetResourceAcct = currentResource ? currentResource.pubkey : fleetResourceToken;
					let resourceCargoTypeAcct = cargoTypes.find(item => item.account.mint.toString() == tokenMint);
					await getAccountInfo(fleet.label, 'Starbase cargo token', starbaseCargoToken) || await createPDA(starbaseCargoToken, starbasePlayerCargoHold, new solanaWeb3.PublicKey(tokenMint), fleet);
					let tx = { instruction: await sageProgram.methods.withdrawCargoFromFleet({ amount: new BrowserAnchor.anchor.BN(amount), keyIndex: new BrowserAnchor.anchor.BN(userProfileKeyIdx) }).accountsStrict({
							gameAccountsFleetAndOwner: {
									gameFleetAndOwner: {
											fleetAndOwner: {
													fleet: fleet.publicKey,
													owningProfile: userProfileAcct,
													owningProfileFaction: userProfileFactionAcct.publicKey,
													key: userPublicKey
											},
											gameId: sageGameAcct.publicKey
									},
									gameState: sageGameAcct.account.gameState
							},
							starbaseAndStarbasePlayer: {
									starbase: starbase.publicKey,
									starbasePlayer: starbasePlayer.publicKey
							},
							cargoPodFrom: fleetCargoPod, // fleet.cargoHold,
							cargoPodTo: starbasePlayerCargoHold,
							cargoType: resourceCargoTypeAcct.publicKey,
							cargoStatsDefinition: sageGameAcct.account.cargo.statsDefinition,
							tokenFrom: fleetResourceAcct,
							tokenTo: starbaseCargoToken,
							tokenMint: tokenMint,
							fundsTo: userPublicKey,
							cargoProgram: cargoProgramPK,
							tokenProgram: tokenProgAddy
					}).remainingAccounts([{
							pubkey: starbase.publicKey,
							isSigner: false,
							isWritable: false
					}]).instruction()}
					//let txResult = await txSignAndSend(tx, fleet, 'UNLOAD', 100);
					let txResult;
					if(returnTx) {
						txResult = tx;
					} else {
						txResult = await txSignAndSend(tx, fleet, 'UNLOAD', 100);
					}
					resolve(txResult);

					if(!!sisLocal.token) {
						let starbaseName = validTargets.find(target => (target.x + ',' + target.y) == (starbaseX + ',' + starbaseY))?.name;
						let rssName = cargoItems.find(r => r.token == tokenMint)?.name;
						const fleetPK = fleet.publicKey.toString();
						const fleetSavedData = await GM.getValue(fleetPK, '{}');
						const fleetParsedData = JSON.parse(fleetSavedData);
						const assignment = fleetParsedData.assignment;
						let loadType = 'cargo_out';
						if(fleetCargoPod == fleet.fuelTank) loadType = 'fuel_out';
						if(fleetCargoPod == fleet.ammoBank) loadType = 'ammo_out';
						await sendToInflux(`fleetrss,fleet=${influxEscape(fleet.label)},starbase=${influxEscape(starbaseName)},sectorX=${starbaseX},sectorY=${starbaseY},rss=${influxEscape(rssName)},assignment=${assignment},type=${loadType} amount=${amount}`);
					}

			});
	}

	async function createScannerPDAs(fleet) {
		cLog(2,`${FleetTimeStamp(fleet.label)} Maintaining Scanner PDAs`);
		await getAccountInfo(fleet.label, 'fleet SDU token', fleet.sduToken) || await createPDA(fleet.sduToken, fleet.cargoHold, new solanaWeb3.PublicKey(sduItem.token), fleet);
		if(fleet.scanCost > 0) {
			await getAccountInfo(fleet.label, 'fleet food token', fleet.foodToken) || await createPDA(fleet.foodToken, fleet.cargoHold, new solanaWeb3.PublicKey(foodItem.token), fleet);
		}
		await getAccountInfo(fleet.label, 'fleet fuel token', fleet.fuelToken) || await createPDA(fleet.fuelToken, fleet.fuelTank, new solanaWeb3.PublicKey(fuelItem.token), fleet);
	}

	async function execCargoFromStarbaseToFleet(fleet, cargoPodTo, tokenTo, tokenMint, cargoType, dockCoords, amount, forceAmount, returnTx, alreadyLoadedInTransaction) {
		return new Promise(async resolve => {
			let txResult = {};
			let starbaseX = dockCoords.split(',')[0].trim();
			let starbaseY = dockCoords.split(',')[1].trim();
			let starbase = await getStarbaseFromCoords(starbaseX, starbaseY);
			let starbasePlayer = await getStarbasePlayer(userProfileAcct,starbase.publicKey);
			let starbasePlayerCargoHolds = await cargoProgram.account.cargoPod.all([
				{
						memcmp: {
								offset: 41,
								bytes: starbasePlayer.publicKey.toBase58(),
						},
				},
			]);
			let starbasePlayerCargoHold = starbasePlayerCargoHolds[0];
			let mostFound = 0;
            for (let cargoHold of starbasePlayerCargoHolds) {
                if (cargoHold.account && cargoHold.account.openTokenAccounts > 0) {
                    let cargoHoldTokens = await solanaReadConnection.getParsedTokenAccountsByOwner(cargoHold.publicKey, {programId: tokenProgramPK});

                    if(!fleet.state.includes('ERROR')) {
			await influxStarbaseCargoHold(starbaseX,starbaseY,cargoHoldTokens);
                    }

                    let cargoHoldFound = cargoHoldTokens.value.find(item => item.account.data.parsed.info.mint === tokenMint && item.account.data.parsed.info.tokenAmount.uiAmount >= amount);
                    if (cargoHoldFound) {
                        starbasePlayerCargoHold = cargoHold;
                        mostFound = cargoHoldFound.account.data.parsed.info.tokenAmount.uiAmount;
                        break;
                    } else {
                        let cargoHoldFound = cargoHoldTokens.value.find(item => item.account.data.parsed.info.mint === tokenMint && item.account.data.parsed.info.tokenAmount.uiAmount >= mostFound);
                        if (cargoHoldFound) {
                            starbasePlayerCargoHold = cargoHold;
                            mostFound = cargoHoldFound.account.data.parsed.info.tokenAmount.uiAmount;
                        }
                    }
                }
            }

			if(alreadyLoadedInTransaction) {
				mostFound = mostFound - alreadyLoadedInTransaction;
			}

			//amount = amount > mostFound ? mostFound : amount;
			let orgAmount = amount;
			if(globalSettings.starbaseKeep1) {
				// don't close the token account; also prevents a race condition if a transporter loads and closes the account while a miner unloads in the same moment
				amount = amount >= mostFound ? mostFound-1 : amount;
			} else {
				amount = amount > mostFound ? mostFound : amount;
			}

			//if (amount > 0) {
			if ((!forceAmount && amount > 0) || (forceAmount && amount >= orgAmount)) {
				//Make sure fleet token account exists
				const tokenMintPK = new solanaWeb3.PublicKey(tokenMint)
				await getAccountInfo(fleet.label, 'fleet cargo token', tokenTo) || await createPDA(tokenTo, cargoPodTo, tokenMintPK, fleet);

				let [starbaseCargoToken] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
					[
							starbasePlayerCargoHold.publicKey.toBuffer(),
							tokenProgramPK.toBuffer(),
							tokenMintPK.toBuffer()
					],
					programPK
				);

				//Get/create source account (why?)
				//await getAccountInfo(fleet.label, 'Starbase cargo token', starbaseCargoToken) || await createPDA(starbaseCargoToken, starbasePlayerCargoHold.publicKey, new solanaWeb3.PublicKey(tokenMint), fleet);

				//Build tx
				let tx = { instruction: await sageProgram.methods.depositCargoToFleet({ amount: new BrowserAnchor.anchor.BN(amount), keyIndex: new BrowserAnchor.anchor.BN(userProfileKeyIdx) }).accountsStrict({
						gameAccountsFleetAndOwner: {
								gameFleetAndOwner: {
										fleetAndOwner: {
												fleet: fleet.publicKey,
												owningProfile: userProfileAcct,
												owningProfileFaction: userProfileFactionAcct.publicKey,
												key: userPublicKey
										},
										gameId: sageGameAcct.publicKey
								},
								gameState: sageGameAcct.account.gameState
						},
						fundsTo: userPublicKey,
						starbaseAndStarbasePlayer: {
								starbase: starbase.publicKey,
								starbasePlayer: starbasePlayer.publicKey
						},
						cargoPodFrom: starbasePlayerCargoHold.publicKey,
						cargoPodTo: cargoPodTo,
						cargoType: cargoType.publicKey,
						cargoStatsDefinition: sageGameAcct.account.cargo.statsDefinition,
						tokenFrom: starbaseCargoToken,
						tokenTo: tokenTo,
						tokenMint: tokenMint,
						cargoProgram: cargoProgramPK,
						tokenProgram: tokenProgAddy
				}).remainingAccounts([{
						pubkey: starbase.publicKey,
						isSigner: false,
						isWritable: false
				}]).instruction()}

				//Send tx
				//txResult = {amount: amount, result: await txSignAndSend(tx, fleet, 'LOAD', 100)};
				if(returnTx) {
					txResult = {amount: amount, tx: tx };
				} else {
					txResult = {amount: amount, result: await txSignAndSend(tx, fleet, 'LOAD', 100)};
				}
			}
			else txResult = {name: "NotEnoughResource"};

			resolve(txResult);

			if(!!sisLocal.token && amount > 0) {
				let starbaseName = validTargets.find(target => (target.x + ',' + target.y) == (starbaseX + ',' + starbaseY))?.name;
				let rssName = cargoItems.find(r => r.token == tokenMint)?.name;
				const fleetPK = fleet.publicKey.toString();
				const fleetSavedData = await GM.getValue(fleetPK, '{}');
				const fleetParsedData = JSON.parse(fleetSavedData);
				const assignment = fleetParsedData.assignment;
				let loadType = 'cargo_in';
				if(cargoPodTo == fleet.fuelTank) loadType = 'fuel_in';
				if(cargoPodTo == fleet.ammoBank) loadType = 'ammo_in';
				await sendToInflux(`fleetrss,fleet=${influxEscape(fleet.label)},starbase=${influxEscape(starbaseName)},sectorX=${starbaseX},sectorY=${starbaseY},rss=${influxEscape(rssName)},assignment=${assignment},type=${loadType} amount=${amount}`);
			}

		});
	}

    async function execStartMining(fleet, mineItem, sageResource, planet) {
        return new Promise(async resolve => {
            let resourceToken = fleet.mineResource;
            let targetX = fleet.destCoord.split(',')[0].trim();
            let targetY = fleet.destCoord.split(',')[1].trim();
            let starbase = await getStarbaseFromCoords(targetX, targetY);
            let starbasePlayer = await getStarbasePlayer(userProfileAcct,starbase.publicKey);
            starbasePlayer = starbasePlayer ? starbasePlayer.publicKey : await execRegisterStarbasePlayer(fleet, fleet.destCoord);
            let tx = { instruction: await sageProgram.methods.startMiningAsteroid({keyIndex: new BrowserAnchor.anchor.BN(userProfileKeyIdx)}).accountsStrict({
                gameAccountsFleetAndOwner: {
                    gameFleetAndOwner: {
                        fleetAndOwner: {
                            fleet: fleet.publicKey,
                            owningProfile: userProfileAcct,
                            owningProfileFaction: userProfileFactionAcct.publicKey,
                            key: userPublicKey
                        },
                        gameId: sageGameAcct.publicKey
                    },
                    gameState: sageGameAcct.account.gameState
                },
                starbaseAndStarbasePlayer: {
                    starbase: starbase.publicKey,
                    starbasePlayer: starbasePlayer
                },
                mineItem : mineItem.publicKey,
                resource: sageResource.publicKey,
                planet: planet.publicKey,
                fleetFuelTokenAccount: fleet.fuelToken,
            }).instruction()}

            cLog(1,`${FleetTimeStamp(fleet.label)} Mining Start ...`);
            updateFleetState(fleet, 'Mine Starting')

            let txResult = await txSignAndSend(tx, fleet, 'START MINING', 10);
            resolve(txResult);
        });
    }

    async function execStopMining(fleet, sageResource, sageResourceAcctInfo, mineItem, resourceToken) {
        return new Promise(async resolve => {
            let planet = sageResourceAcctInfo.location;
            let targetX = fleet.destCoord.split(',')[0].trim();
            let targetY = fleet.destCoord.split(',')[1].trim();
            let starbase = await getStarbaseFromCoords(targetX, targetY);
            //let starbasePlayer = await getStarbasePlayer(userProfileAcct,starbase.publicKey);

            let [planetResourceToken] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
                [
                    mineItem.toBuffer(),
                    tokenProgramPK.toBuffer(),
                    resourceToken.toBuffer()
                ],
                programPK
            );
            let [fleetResourceToken] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
                [
                    fleet.cargoHold.toBuffer(),
                    tokenProgramPK.toBuffer(),
                    resourceToken.toBuffer()
                ],
                programPK
            );
            let [fleetFoodToken] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
                [
                    fleet.cargoHold.toBuffer(),
                    tokenProgramPK.toBuffer(),
                    sageGameAcct.account.mints.food.toBuffer()
                ],
                programPK
            );
            const fleetAmmoToken = await getFleetAmmoToken(fleet);
            const fleetCurrentCargo = await solanaReadConnection.getParsedTokenAccountsByOwner(fleet.cargoHold, {programId: tokenProgramPK});
            const currentFood = fleetCurrentCargo.value.find(item => item.account.data.parsed.info.mint === sageGameAcct.account.mints.food.toString());
            const fleetFoodAcct = currentFood ? currentFood.pubkey : fleetFoodToken;

            let fleetCurrentAmmoBank = await solanaReadConnection.getParsedTokenAccountsByOwner(fleet.ammoBank, {programId: tokenProgramPK});
            let currentAmmo = fleetCurrentAmmoBank.value.find(item => item.account.data.parsed.info.mint === sageGameAcct.account.mints.ammo.toString());
            let fleetAmmoAcct = currentAmmo ? currentAmmo.pubkey : fleetAmmoToken;
            await solanaReadConnection.getAccountInfo(fleetAmmoAcct) || await createPDA(fleetAmmoAcct, fleet.ammoBank, sageGameAcct.account.mints.ammo, fleet);

            const accInfo = await getAccountInfo(fleet.label, 'fleet resource token', fleetResourceToken);
            cLog(2, `${FleetTimeStamp(fleet.label)} Mining getAccountInfo result`,accInfo);
            if(!accInfo) {
                const cpda = await createPDA(fleetResourceToken, fleet.cargoHold, resourceToken, fleet);

                cLog(2, `${FleetTimeStamp(fleet.label)} Mining createPDA result`, cpda);
            }
            let foodCargoTypeAcct = cargoTypes.find(item => item.account.mint.toString() == sageGameAcct.account.mints.food);
            let ammoCargoTypeAcct = cargoTypes.find(item => item.account.mint.toString() == sageGameAcct.account.mints.ammo);
            let resourceCargoTypeAcct = cargoTypes.find(item => item.account.mint.toString() == resourceToken.toString());
            let tx1 = { instruction: await sageProgram.methods.fleetStateHandler().accountsStrict({
                fleet: fleet.publicKey
            }).remainingAccounts([
                {
                    pubkey: fleet.cargoHold,
                    isSigner: false,
                    isWritable: true
                },
                {
                    pubkey: fleet.ammoBank,
                    isSigner: false,
                    isWritable: true
                },
                {
                    pubkey: mineItem,
                    isSigner: false,
                    isWritable: false
                },
                {
                    pubkey: sageResource,
                    isSigner: false,
                    isWritable: true
                },
                {
                    pubkey: planet,
                    isSigner: false,
                    isWritable: true
                },
                {
                    pubkey: starbase.publicKey,
                    isSigner: false,
                    isWritable: false
                },
                {
                    pubkey: fleetFoodAcct,
                    isSigner: false,
                    isWritable: true
                },
                {
                    pubkey: fleetAmmoAcct,
                    isSigner: false,
                    isWritable: true
                },
                {
                    pubkey: planetResourceToken,
                    isSigner: false,
                    isWritable: true
                },
                {
                    pubkey: fleetResourceToken,
                    isSigner: false,
                    isWritable: true
                },
                {
                    pubkey: sageGameAcct.account.mints.food,
                    isSigner: false,
                    isWritable: true
                },
                {
                    pubkey: sageGameAcct.account.mints.ammo,
                    isSigner: false,
                    isWritable: true
                },
                {
                    pubkey: foodCargoTypeAcct.publicKey,
                    isSigner: false,
                    isWritable: false
                },
                {
                    pubkey: ammoCargoTypeAcct.publicKey,
                    isSigner: false,
                    isWritable: false
                },
                {
                    pubkey: resourceCargoTypeAcct.publicKey,
                    isSigner: false,
                    isWritable: false
                },
                {
                    pubkey: sageGameAcct.account.cargo.statsDefinition,
                    isSigner: false,
                    isWritable: false
                },
                {
                    pubkey: sageGameAcct.account.gameState,
                    isSigner: false,
                    isWritable: false
                },
                {
                    pubkey: sageGameAcct.publicKey,
                    isSigner: false,
                    isWritable: false
                },
                {
                    pubkey: cargoProgramPK,
                    isSigner: false,
                    isWritable: false
                },
                {
                    pubkey: tokenProgramPK,
                    isSigner: false,
                    isWritable: false
                },
            ]).instruction()}
            updateFleetState(fleet, `Mining Stop`);
            //let tx1Result = await txSignAndSend(tx1, fleet, 'STOP MINING (fleetStateHandler)', 100);

            let fuelCargoTypeAcct = cargoTypes.find(item => item.account.mint.toString() == sageGameAcct.account.mints.fuel);
            let tx2 = { instruction: await sageProgram.methods.stopMiningAsteroid({keyIndex: new BrowserAnchor.anchor.BN(userProfileKeyIdx)}).accountsStrict({
                gameAccountsFleetAndOwner: {
                    gameFleetAndOwner: {
                        fleetAndOwner: {
                            fleet: fleet.publicKey,
                            owningProfile: userProfileAcct,
                            owningProfileFaction: userProfileFactionAcct.publicKey,
                            key: userPublicKey
                        },
                        gameId: sageGameAcct.publicKey
                    },
                    gameState: sageGameAcct.account.gameState
                },
                mineItem: mineItem,
                resource: sageResource,
                planet: planet,
                fuelTank : fleet.fuelTank,
                cargoType: fuelCargoTypeAcct.publicKey,
                cargoStatsDefinition: sageGameAcct.account.cargo.statsDefinition,
                tokenFrom: fleet.fuelToken,
                tokenMint: sageGameAcct.account.mints.fuel,
                miningXpAccounts: userXpAccounts.userMiningXpAccounts,
                pilotXpAccounts: userXpAccounts.userPilotingXpAccounts,
                councilRankXpAccounts: userXpAccounts.userCouncilRankXpAccounts,
                progressionConfig: progressionConfigAcct,
                cargoProgram: cargoProgramPK,
                pointsProgram: pointsProgramId,
                tokenProgram: tokenProgramPK,
            }).instruction()}

            cLog(1,`${FleetTimeStamp(fleet.label)} Mining Stop`);
            updateFleetState(fleet, 'Mining Stop')

            //let txResult = await txSignAndSend(tx2, fleet, 'STOP MINING', 100);
            let txResult = await txSignAndSend([tx1,tx2], fleet, 'STOP MINING', 100);

            //await wait(2000);
            cLog(1,`${FleetTimeStamp(fleet.label)} Idle 💤`);
            updateFleetState(fleet, 'Idle');

            resolve(txResult);

            if(!fleet.state.includes('ERROR') && txResult && txResult.meta && !!sisLocal.token) {
		let postTokenBalances = txResult.meta.postTokenBalances;
		let preTokenBalances = txResult.meta.preTokenBalances;
		let preFoodCnt = 0;
		let postFoodCnt = 0;
		let preAmmoCnt = 0;
		let postAmmoCnt = 0;
		let preRssCnt = 0;
		let postRssCnt = 0;
		for (let bal of preTokenBalances) {
			if (bal.mint == resourceToken.toString() && bal.owner==fleet.cargoHold) { preRssCnt = bal.uiTokenAmount.uiAmount; }
			if (bal.mint == sageGameAcct.account.mints.food.toString()) { preFoodCnt = bal.uiTokenAmount.uiAmount; }
			if (bal.mint == sageGameAcct.account.mints.ammo.toString()) { preAmmoCnt = bal.uiTokenAmount.uiAmount; }
		}
		for (let bal of postTokenBalances) {
			if (bal.mint == resourceToken.toString() && bal.owner==fleet.cargoHold) { postRssCnt = bal.uiTokenAmount.uiAmount; }
			if (bal.mint == sageGameAcct.account.mints.food.toString()) { postFoodCnt = bal.uiTokenAmount.uiAmount; }
			if (bal.mint == sageGameAcct.account.mints.ammo.toString()) { postAmmoCnt = bal.uiTokenAmount.uiAmount; }
		}
		let minedAmount = postRssCnt - preRssCnt;
		let burnedFood = preFoodCnt - postFoodCnt;
		let burnedAmmo = preAmmoCnt - postAmmoCnt;

		let minedRssName = cargoItems.find(r => r.token == resourceToken.toString())?.name;
		let starbaseName = validTargets.find(target => (target.x + ',' + target.y) == targetX + ',' + targetY )?.name;
		await sendToInflux(`mining,fleet=${influxEscape(fleet.label)},starbase=${influxEscape(starbaseName)},sectorX=${targetX},sectorY=${targetY},rss=${influxEscape(minedRssName)} burnedFuel=${fleet.planetExitFuelAmount},burnedFood=${burnedFood},burnedAmmo=${burnedAmmo},amount=${minedAmount}`);
            }

        });
    }

    async function execStartCrafting(starbase, starbasePlayer, starbasePlayerCargoHoldsAndTokens, craftingRecipe, craftAmount, userCraft) {
        return new Promise(async resolve => {
            let transactions = [];
            let numCrew = craftAmount > 1 ? userCraft.crew : 1;

            let facility = craftRecipes.some(item => item.name === craftingRecipe.name) ? starbase.account.craftingFacility : starbase.account.upgradeFacility;

            let craftingFacilityAcct = await solanaReadConnection.getAccountInfo(facility);
            let craftingFacilityData = craftingFacilityAcct.data.subarray(90);
            let craftIter = 0;
            let craftingFacilityRecipeCategories = [];
            while (craftingFacilityData.length >= 32) {
                let currRecipeCategory = craftingFacilityData.subarray(0, 32);
                let recipeCategoryDecoded = craftingProgram.coder.types.decode('WrappedRecipeCategory', currRecipeCategory);
                craftingFacilityRecipeCategories.push({recipeCategory: recipeCategoryDecoded.id, idx: craftIter});
                craftingFacilityData = craftingFacilityData.subarray(32);
                craftIter += 1;
            }

            let recipeCategoryIndex = craftingFacilityRecipeCategories.find(item => item.recipeCategory.toString() === craftingRecipe.category.toString());
            if (!recipeCategoryIndex) {
                updateFleetState(userCraft, 'ERROR: Craft Unavailable');
                resolve({name: "CraftUnavailable"});
                return;
            }

            let tempBytes = new Uint8Array(256);
            let tempRandomBytes = crypto.getRandomValues(tempBytes);
            let formattedRandomBytes = BrowserBuffer.Buffer.Buffer.from(tempRandomBytes).readUIntLE(0, 6);
            let bnRandomBytes = new BrowserAnchor.anchor.BN(formattedRandomBytes);
            let [craftingProcess] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
                [
                    BrowserBuffer.Buffer.Buffer.from("CraftingProcess"),
                    facility.toBuffer(),
                    craftingRecipe.publicKey.toBuffer(),
                    bnRandomBytes.toArrayLike(BrowserBuffer.Buffer.Buffer, "le", 8)
                ],
                craftingProgramPK
            );

            let [craftingInstance] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
                [
                    BrowserBuffer.Buffer.Buffer.from("CraftingInstance"),
                    starbasePlayer.toBuffer(),
                    craftingProcess.toBuffer(),
                ],
                sageProgramPK
            );

            let createProcessMethod = craftRecipes.some(item => item.name === craftingRecipe.name) ? sageProgram.methods.createCraftingProcess : sageProgram.methods.createStarbaseUpgradeResourceProcess;
            let facilityLabel = craftRecipes.some(item => item.name === craftingRecipe.name) ? 'craftingFacility' : 'upgradeFacility';

            let tx1 = { instruction: await createProcessMethod({
                keyIndex: new BrowserAnchor.anchor.BN(userProfileKeyIdx),
                craftingId: new BrowserAnchor.anchor.BN(formattedRandomBytes),
                recipeCategoryIndex: new BrowserAnchor.anchor.BN(recipeCategoryIndex.idx),
                quantity: new BrowserAnchor.anchor.BN(craftAmount),
                numCrew: new BrowserAnchor.anchor.BN(numCrew)
            }).accountsStrict({
                funder: userPublicKey,
                starbaseAndStarbasePlayer: {
                    starbase: starbase.publicKey,
                    starbasePlayer: starbasePlayer
                },
                gameAccountsAndProfile: {
                    gameAndProfileAndFaction: {
                        gameId: sageGameAcct.publicKey,
                        key: userPublicKey,
                        profile: userProfileAcct,
                        profileFaction: userProfileFactionAcct.publicKey
                    },
                    gameState: sageGameAcct.account.gameState
                },
                craftingInstance: craftingInstance,
                craftingProcess: craftingProcess,
                [facilityLabel]: facility,
                craftingRecipe: craftingRecipe.publicKey,
                craftingDomain: craftingRecipe.domain,
                craftingProgram: craftingProgramPK,
                systemProgram: solanaWeb3.SystemProgram.programId
            }).instruction()}
            transactions.push(tx1);

            for (let ingredient of craftingRecipe.input) {
                let starbasePlayerCargoHold = getStarbasePlayerCargoMaxItem(starbasePlayerCargoHoldsAndTokens, ingredient.mint.toString());
                let cargoTypeAcct = cargoTypes.find(item => item.account.mint.toString() == ingredient.mint.toString());

                let [starbaseCargoToken] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
                    [
                        starbasePlayerCargoHold.starbasePlayerCargoHold.toBuffer(),
                        tokenProgramPK.toBuffer(),
                        ingredient.mint.toBuffer()
                    ],
                    programPK
                );

                let [ingredientToken] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
                    [
                        craftingProcess.toBuffer(),
                        tokenProgramPK.toBuffer(),
                        ingredient.mint.toBuffer(),
                    ],
                    programPK
                );
                if (!await getAccountInfo(userCraft.label, 'Crafting ingredient token', ingredientToken)) transactions.push(await createPDA(ingredientToken, craftingProcess, ingredient.mint, userCraft, false));

                let tx = { instruction: await sageProgram.methods.depositCraftingIngredient({
                    amount: new BrowserAnchor.anchor.BN(craftAmount * ingredient.amount),
                    keyIndex: new BrowserAnchor.anchor.BN(userProfileKeyIdx),
                    ingredientIndex: ingredient.idx
                }).accountsStrict({
                    starbaseAndStarbasePlayer: {
                        starbase: starbase.publicKey,
                        starbasePlayer: starbasePlayer
                    },
                    gameAccountsAndProfile: {
                        gameAndProfileAndFaction: {
                            gameId: sageGameAcct.publicKey,
                            key: userPublicKey,
                            profile: userProfileAcct,
                            profileFaction: userProfileFactionAcct.publicKey
                        },
                        gameState: sageGameAcct.account.gameState
                    },
                    craftingInstance: craftingInstance,
                    craftingProcess: craftingProcess,
                    craftingFacility: facility,
                    craftingRecipe: craftingRecipe.publicKey,
                    cargoPodFrom: starbasePlayerCargoHold.starbasePlayerCargoHold,
                    cargoType: cargoTypeAcct.publicKey,
                    cargoStatsDefinition: sageGameAcct.account.cargo.statsDefinition,
                    tokenFrom: starbaseCargoToken,
                    tokenTo: ingredientToken,
                    craftingProgram: craftingProgramPK,
                    cargoProgram: cargoProgramPK,
                    tokenProgram: tokenProgramPK
                }).instruction()}
                transactions.push(tx);
            }

            let [signerFeeMintToken] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
                [
                    userPublicKey.toBuffer(),
                    tokenProgramPK.toBuffer(),
                    sageGameAcct.account.mints.atlas.toBuffer(),
                ],
                programPK
            );

            let startCraftProcRemainingAccts = [
                {
                    pubkey: craftingRecipe.feeRecipient,
                    isSigner: false,
                    isWritable: false
                }, {
                    pubkey: userPublicKey,
                    isSigner: true,
                    isWritable: true
                }, {
                    pubkey: signerFeeMintToken,
                    isSigner: false,
                    isWritable: true
                }];

            if (craftRecipes.some(item => item.name === craftingRecipe.name)) {
                let [craftingAtlasToken] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
                    [
                        craftingProcess.toBuffer(),
                        tokenProgramPK.toBuffer(),
                        sageGameAcct.account.mints.atlas.toBuffer(),
                    ],
                    programPK
                );

                if (!await getAccountInfo(userCraft.label, 'Crafting atlas token', craftingAtlasToken)) transactions.push(await createPDA(craftingAtlasToken, craftingProcess, sageGameAcct.account.mints.atlas, userCraft, false));

                startCraftProcRemainingAccts.push({
                    pubkey: craftingAtlasToken,
                    isSigner: false,
                    isWritable: true
                });
            }

            startCraftProcRemainingAccts.push({
                pubkey: tokenProgramPK,
                isSigner: false,
                isWritable: false
            });

            let tx2 = { instruction: await sageProgram.methods.startCraftingProcess({keyIndex: new BrowserAnchor.anchor.BN(userProfileKeyIdx)}).accountsStrict({
                starbaseAndStarbasePlayer: {
                    starbase: starbase.publicKey,
                    starbasePlayer: starbasePlayer
                },
                craftingInstance: craftingInstance,
                craftingProcess: craftingProcess,
                craftingRecipe: craftingRecipe.publicKey,
                craftingFacility: facility,
                gameAccountsAndProfile: {
                    gameAndProfileAndFaction: {
                        gameId: sageGameAcct.publicKey,
                        key: userPublicKey,
                        profile: userProfileAcct,
                        profileFaction: userProfileFactionAcct.publicKey
                    },
                    gameState: sageGameAcct.account.gameState
                },
                craftingProgram: craftingProgramPK
                }).remainingAccounts(startCraftProcRemainingAccts).instruction()}
            transactions.push(tx2);

            //let txResult = {craftingId: formattedRandomBytes, result: await txSignAndSend(transactions, userCraft, 'START CRAFTING', Math.min(globalSettings.craftingTxMultiplier, 500) )};
            let txResult = {craftingId: formattedRandomBytes, feeAtlas: 0, result: await txSliceAndSend(transactions, userCraft, 'START CRAFTING', Math.min(globalSettings.craftingTxMultiplier, 500), 6 )};

            if (!userCraft.state.includes('ERROR')) {
	            let postTokenBalances = txResult.result.meta.postTokenBalances;
	            let feeAccount = txResult.result.transaction.message.staticAccountKeys.map((key) => key.toBase58())[3];
	            for (var b in postTokenBalances) {
	            	if (postTokenBalances[b].mint=='ATLASXmbPQxBUYbxPsV97usA3fPQYEqzQBUHgiFCUsXx' && postTokenBalances[b].owner==feeAccount && postTokenBalances[b].uiTokenAmount.uiAmount) {
	            		await alterStats('ATLAS Fees','Crafting',postTokenBalances[b].uiTokenAmount.uiAmount,'ATLAS',4);
				txResult.feeAtlas = postTokenBalances[b].uiTokenAmount.uiAmount;
	            	}
	            }
	    }

            resolve(txResult);
        });
    }

    async function execCompleteCrafting(starbase, starbasePlayer, starbasePlayerCargoHoldsAndTokens, craftingProcess, userCraft) {
        return new Promise(async resolve => {
            let transactions = [];

            let craftRecipe = craftRecipes.find(item => item.publicKey.toString() === craftingProcess.recipe.toString());

            let influxStr = '';
            let starbaseName = validTargets.find(target => (target.x + ',' + target.y) == (starbase.account.sector[0].toNumber() + ',' + starbase.account.sector[1].toNumber()))?.name;;
            let outputName = cargoItems.find(r => r.token == craftRecipe.output.mint.toString())?.name;

            // status:
            // <2 = not ready
            // 2 = ready
            // 3 = at least 1 ingredient was burned, but the craftingProcess is still open
            if (craftingProcess.status < 2) {
		updateFleetState(userCraft, 'ERROR: Invalid crafting state');
		resolve({});
		return;
            }

            //if (craftingProcess.status == 2) {
                for (let ingredient of craftRecipe.input) {
                    let [ingredientToken] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
                        [
                            craftingProcess.craftingProcess.toBuffer(),
                            tokenProgramPK.toBuffer(),
                            ingredient.mint.toBuffer(),
                        ],
                        programPK
                    );

                    //we don't need to check for PDA. Also we don't need to create one.
                    //if (!await getAccountInfo(userCraft.label, 'Crafting ingredient token', ingredientToken)) transactions.push(await createPDA(ingredientToken, craftingProcess, ingredient.mint, userCraft, false));

                    // input Checksum has a bit for each ingredient.
                    // so if we have 3 ingredients, the initial value is 1+2+4=7
                    // when we claim the ingredient with index 1, the checksum is 1+4=5 now
                    // we need to make sure it works with more than 8 ingredients
                    const bitValue = craftingProcess.inputsChecksum[0] + craftingProcess.inputsChecksum[1] * 256;
                    const ingredientBurned = ((bitValue >> ingredient.idx) & 1) == 0;
                    if(ingredientBurned) { cLog(1,`${FleetTimeStamp(userCraft.label)} Ingredient`, ingredient.idx, `was already burned, skipping ...`); continue; }

                    let inputName = cargoItems.find(r => r.token == ingredient.mint.toString())?.name;
                    influxStr += (influxStr.length ? "\n" : "") + `crafting,starbase=${influxEscape(starbaseName)},sectorX=${starbase.account.sector[0].toNumber()},sectorY=${starbase.account.sector[1].toNumber()},input=${influxEscape(inputName)},output=${influxEscape(outputName)},craftingID=${userCraft.craftingId},type=Input fee=${userCraft.feeAtlas ? userCraft.feeAtlas : 0},amount=${craftingProcess.quantity * ingredient.amount}`;

                    let tx = { instruction: await sageProgram.methods.burnCraftingConsumables({
                        ingredientIndex: ingredient.idx
                    }).accountsStrict({
                        starbaseAndStarbasePlayer: {
                            starbase: starbase.publicKey,
                            starbasePlayer: starbasePlayer
                        },
                        gameAccounts: {
                            gameId: sageGameAcct.publicKey,
                            gameState: sageGameAcct.account.gameState
                        },
                        craftingInstance: craftingProcess.craftingInstance,
                        craftingProcess: craftingProcess.craftingProcess,
                        craftingFacility: starbase.account.craftingFacility,
                        craftingRecipe: craftingProcess.recipe,
                        tokenFrom: ingredientToken,
                        tokenMint: ingredient.mint,
                        craftingProgram: craftingProgramPK,
                        tokenProgram: tokenProgramPK
                    }).instruction()}
                    transactions.push(tx);
                }

		//outputToken was not used anywhere, removed

                let starbasePlayerCargoHold = getStarbasePlayerCargoMaxItem(starbasePlayerCargoHoldsAndTokens, craftRecipe.output.mint.toString());
                starbasePlayerCargoHold = starbasePlayerCargoHold ? starbasePlayerCargoHold.starbasePlayerCargoHold : starbasePlayerCargoHoldsAndTokens.length > 0 ? starbasePlayerCargoHoldsAndTokens[0].starbasePlayerCargoHold : await execCreateCargoPod(userCraft, userCraft.coordinates);

                let cargoTypeAcct = cargoTypes.find(item => item.account.mint.toString() == craftRecipe.output.mint.toString());

                let [starbaseCargoToken] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
                    [
                        starbasePlayerCargoHold.toBuffer(),
                        tokenProgramPK.toBuffer(),
                        craftRecipe.output.mint.toBuffer()
                    ],
                    programPK
                );

                await getAccountInfo(userCraft.label, 'Starbase cargo token', starbaseCargoToken) || await createPDA(starbaseCargoToken, starbasePlayerCargoHold, craftRecipe.output.mint, userCraft);

                let craftableItem = craftableItems.find(item => item.account.domain.toString() === craftRecipe.domain.toBase58() && item.account.mint.toString() === craftRecipe.output.mint.toBase58());

                let [outputFrom] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
                    [
                        craftableItem.publicKey.toBuffer(),
                        tokenProgramPK.toBuffer(),
                        craftRecipe.output.mint.toBuffer()
                    ],
                    programPK
                );
                if (!await getAccountInfo(userCraft.label, 'Crafting ingredient token', outputFrom)) transactions.push(await createPDA(outputFrom, craftableItem.publicKey, craftRecipe.output.mint, userCraft, false));

                //if the output checksum is 0, we haven't claimed the output yet.
                if(!craftingProcess.outputsChecksum[0])
                {
                    	influxStr += (influxStr.length ? "\n" : "") + `crafting,starbase=${influxEscape(starbaseName)},sectorX=${starbase.account.sector[0].toNumber()},sectorY=${starbase.account.sector[1].toNumber()},output=${influxEscape(outputName)},craftingID=${userCraft.craftingId},type=Output fee=${userCraft.feeAtlas ? userCraft.feeAtlas : 0},amount=${craftingProcess.quantity}`;

	                let tx1 = { instruction: await sageProgram.methods.claimCraftingOutputs({
	                    ingredientIndex: craftRecipe.output.idx
	                }).accountsStrict({
	                    starbaseAndStarbasePlayer: {
	                        starbase: starbase.publicKey,
	                        starbasePlayer: starbasePlayer
	                    },
	                    gameAccounts: {
	                        gameId: sageGameAcct.publicKey,
	                        gameState: sageGameAcct.account.gameState
	                    },
	                    craftingInstance: craftingProcess.craftingInstance,
	                    craftingProcess: craftingProcess.craftingProcess,
	                    craftingFacility: starbase.account.craftingFacility,
	                    craftingRecipe: craftingProcess.recipe,
	                    craftableItem: craftableItem.publicKey,
	                    cargoPodTo: starbasePlayerCargoHold,
	                    cargoType: cargoTypeAcct.publicKey,
	                    cargoStatsDefinition: sageGameAcct.account.cargo.statsDefinition,
	                    tokenFrom: outputFrom,
	                    tokenTo: starbaseCargoToken,
	                    craftingProgram: craftingProgramPK,
	                    cargoProgram: cargoProgramPK,
	                    tokenProgram: tokenProgramPK
	                }).instruction()}
	                transactions.push(tx1);
		} else {
			cLog(1,`${FleetTimeStamp(userCraft.label)} Output was already claimed, skipping ...`);
		}

                //cLog(1,`${FleetTimeStamp(userCraft.label)} Completing craft (2 transactions)`);
                //let tx1Result = await txSignAndSend(transactions, userCraft, 'COMPLETING CRAFT TX1');
            //}

            let [craftingAtlasToken] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
                [
                    craftingProcess.craftingProcess.toBuffer(),
                    tokenProgramPK.toBuffer(),
                    sageGameAcct.account.mints.atlas.toBuffer(),
                ],
                programPK
            );

            let tx2 = { instruction: await sageProgram.methods.closeCraftingProcess({keyIndex: new BrowserAnchor.anchor.BN(userProfileKeyIdx)}).accountsStrict({
                fundsTo: userPublicKey,
                starbaseAndStarbasePlayer: {
                    starbase: starbase.publicKey,
                    starbasePlayer: starbasePlayer
                },
                gameAccountsAndProfile: {
                    gameAndProfileAndFaction: {
                        gameId: sageGameAcct.publicKey,
                        key: userPublicKey,
                        profile: userProfileAcct,
                        profileFaction: userProfileFactionAcct.publicKey
                    },
                    gameState: sageGameAcct.account.gameState
                },
                craftingInstance: craftingProcess.craftingInstance,
                craftingProcess: craftingProcess.craftingProcess,
                craftingRecipe: craftingProcess.recipe,
                craftingFacility: starbase.account.craftingFacility,
                craftingXpAccounts: userXpAccounts.userCraftingXpAccounts,
                councilRankXpAccounts: userXpAccounts.userCouncilRankXpAccounts,
                progressionConfig: progressionConfigAcct,
                pointsProgram: pointsProgramId,
                craftingProgram: craftingProgramPK
                }).remainingAccounts([
                {
                    pubkey: craftingAtlasToken,
                    isSigner: false,
                    isWritable: true
                }, {
                    pubkey: sageGameAcct.account.vaults.atlas,
                    isSigner: false,
                    isWritable: true
                }, {
                    pubkey: tokenProgramPK,
                    isSigner: false,
                    isWritable: false
                }
            ]).instruction()}
            transactions.push(tx2);

            //let txResult = await txSignAndSend(tx2, userCraft, 'COMPLETING CRAFT TX2');
            //let txResult = await txSignAndSend(transactions, userCraft, 'COMPLETING CRAFT', Math.min(globalSettings.craftingTxMultiplier, 500) );
            let txResult = await txSliceAndSend(transactions, userCraft, 'COMPLETING CRAFT', Math.min(globalSettings.craftingTxMultiplier, 500), 6);

            if(!userCraft.state.includes('ERROR')) {
		await sendToInflux(influxStr);
            }

            // Allow RPC to catch up (to be sure the crew is available before starting the next job)
            await wait(4000);

            resolve(txResult);
        });
    }

    async function execCompleteUpgrade(starbase, starbasePlayer, starbasePlayerCargoHoldsAndTokens, craftingProcess, userCraft) {
        return new Promise(async resolve => {
            let transactions = [];

            let craftingRecipe = upgradeRecipes.find(item => item.publicKey.toString() === craftingProcess.recipe.toString());
            let starbaseUpgradeRecipe = upgradeRecipes.find(item => item.name === 'SB Tier ' + (starbase.account.level+1));
            let starbaseUpgradeRecipeInput = starbaseUpgradeRecipe.input.find(item => item.mint.toString() === craftingRecipe.input[0].mint.toString());
            let itemRecipe = craftRecipes.find(item => item.output.mint.toString() === craftingRecipe.input[0].mint.toString());

            let starbasePlayerCargoHold = getStarbasePlayerCargoMaxItem(starbasePlayerCargoHoldsAndTokens, craftingRecipe.input[0].mint.toString());
            starbasePlayerCargoHold = starbasePlayerCargoHold ? starbasePlayerCargoHold.starbasePlayerCargoHold : starbasePlayerCargoHoldsAndTokens.length > 0 ? starbasePlayerCargoHoldsAndTokens[0].starbasePlayerCargoHold : await execCreateCargoPod(userCraft, userCraft.coordinates);

            let cargoTypeAcct = cargoTypes.find(item => item.account.mint.toString() == craftingRecipe.input[0].mint.toString());

            let [starbaseCargoToken] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
                [
                    starbasePlayerCargoHold.toBuffer(),
                    tokenProgramPK.toBuffer(),
                    craftingRecipe.input[0].mint.toBuffer()
                ],
                programPK
            );

            let craftableItem = craftableItems.find(item => item.account.domain.toString() === craftingRecipe.domain.toBase58() && item.account.mint.toString() === craftingRecipe.input[0].mint.toBase58());

            let [outputFrom] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
                [
                    craftableItem.publicKey.toBuffer(),
                    tokenProgramPK.toBuffer(),
                    craftingRecipe.input[0].mint.toBuffer()
                ],
                programPK
            );

            let [ingredientToken] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
                [
                    craftingProcess.craftingProcess.toBuffer(),
                    tokenProgramPK.toBuffer(),
                    craftingRecipe.input[0].mint.toBuffer(),
                ],
                programPK
            );

            let userRedemptionPubkey = await getUserRedemptionAccount();
            let newRedemption = false;
            let userRedemptionAcct;
            if (!userRedemptionPubkey) {
                userRedemptionAcct = new solanaWeb3.Keypair();
                userRedemptionPubkey = userRedemptionAcct.publicKey;
                newRedemption = true;
            }

            let remainingAccounts = [
                {
                    pubkey: userRedemptionPubkey,
                    isSigner: newRedemption,
                    isWritable: true
                },
                {
                    pubkey: userRedemptionConfigAcct,
                    isSigner: false,
                    isWritable: true
                },
                {
                    pubkey: pointsStoreProgramId,
                    isSigner: false,
                    isWritable: false
                }
            ];

            let ixSigners = [userPublicKey];

            if (newRedemption) {
                remainingAccounts.push({
                    pubkey: solanaWeb3.SystemProgram.programId,
                    isSigner: false,
                    isWritable: false
                });
                remainingAccounts.push({
                    pubkey: userPublicKey,
                    isSigner: true,
                    isWritable: true
                });
                ixSigners.push(userRedemptionAcct);
            }

            let currDayIndex = Math.floor(Date.now() / 86400000);
            let redemptionConfig = await solanaReadConnection.getAccountInfo(userRedemptionConfigAcct);
            let redemptionConfigData = redemptionConfig.data.subarray(112);
            let epochIndex = 0;
            while (redemptionConfigData.length >= 44) {
                let currEpoch = redemptionConfigData.subarray(0, 40);
                let epochDecoded = pointsStoreProgram.coder.types.decode('RedemptionEpoch', currEpoch);
                if (epochDecoded.dayIndex.toNumber() === currDayIndex) {
                    break;
                }
                redemptionConfigData = redemptionConfigData.subarray(40);
                epochIndex += 1;
            }

            let tx1 = { instruction: await sageProgram.methods.submitStarbaseUpgradeResource({
                sagePermissionsKeyIndex: new BrowserAnchor.anchor.BN(userProfileKeyIdx),
                pointsProgramPermissionsKeyIndex: new BrowserAnchor.anchor.BN(pointsProfileKeyIdx),
                upgradeProcessRecipeInputIndex: craftingRecipe.input[0].idx,
                starbaseUpgradeRecipeInputIndex: starbaseUpgradeRecipeInput.idx,
                resourceRecipeOutputIndex: itemRecipe.output.idx,
                epochIndex: epochIndex,
            }).accountsStrict({
                fundsTo: userPublicKey,
                starbaseAndStarbasePlayer: {
                    starbase: starbase.publicKey,
                    starbasePlayer: starbasePlayer
                },
                gameAccountsAndProfile: {
                    gameAndProfileAndFaction: {
                        gameId: sageGameAcct.publicKey,
                        key: userPublicKey,
                        profile: userProfileAcct,
                        profileFaction: userProfileFactionAcct.publicKey
                    },
                    gameState: sageGameAcct.account.gameState
                },
                resourceCraftingInstance: craftingProcess.craftingInstance,
                resourceCraftingProcess: craftingProcess.craftingProcess,
                resourceCraftingFacility: starbase.account.upgradeFacility,
                upgradeProcessRecipe: craftingProcess.recipe,
                starbaseUpgradeRecipe: starbaseUpgradeRecipe.publicKey,
                resourceRecipe: itemRecipe.publicKey,
                cargoPodTo: starbasePlayerCargoHold,
                cargoType: cargoTypeAcct.publicKey,
                cargoStatsDefinition: sageGameAcct.account.cargo.statsDefinition,
                tokenFrom: ingredientToken,
                tokenTo: starbaseCargoToken,
                tokenMint: craftingRecipe.input[0].mint,
                loyaltyPointsAccounts: userXpAccounts.userLPAccounts,
                progressionConfig: progressionConfigAcct,
                pointsProgram: pointsProgramId,
                craftingProgram: craftingProgramPK,
                cargoProgram: cargoProgramPK,
                tokenProgram: tokenProgramPK
            }).remainingAccounts(remainingAccounts).instruction(),
                      signers: ixSigners};
            transactions.push(tx1);

            let tx2 = { instruction: await sageProgram.methods.closeUpgradeProcess({
                keyIndex: new BrowserAnchor.anchor.BN(userProfileKeyIdx)
            }).accountsStrict({
                fundsTo: userPublicKey,
                starbaseAndStarbasePlayer: {
                    starbase: starbase.publicKey,
                    starbasePlayer: starbasePlayer
                },
                gameAccountsAndProfile: {
                    gameAndProfileAndFaction: {
                        gameId: sageGameAcct.publicKey,
                        key: userPublicKey,
                        profile: userProfileAcct,
                        profileFaction: userProfileFactionAcct.publicKey
                    },
                    gameState: sageGameAcct.account.gameState
                },
                resourceCraftingInstance: craftingProcess.craftingInstance,
                resourceCraftingProcess: craftingProcess.craftingProcess,
                resourceRecipe: craftingRecipe.publicKey,
                resourceCraftingFacility: starbase.account.upgradeFacility,
                craftingProgram: craftingProgramPK
            }).instruction()};
            transactions.push(tx2);

            let txResult = await txSignAndSend(transactions, userCraft, 'COMPLETING UPGRADE', Math.min(globalSettings.craftingTxMultiplier, 500), userRedemptionAcct);

            resolve(txResult);
        });
    }

    // TESTING - this is not compatible with Player Profiles

	async function execLoadFleetAmmo(fleet, starbaseCoords, amount, returnTx) {
		const ammoMint = sageGameAcct.account.mints.ammo;
		const parsedTokenAccounts = await solanaReadConnection.getParsedTokenAccountsByOwner(fleet.ammoBank, {programId: tokenProgramPK});
		const parsedTokenAccount = parsedTokenAccounts.value.find(item => item.account.data.parsed.info.mint === ammoMint.toString());
		const ammoCargoTypeAcct = cargoTypes.find(item => item.account.mint.toString() == ammoMint);
		const [fleetAmmoToken] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
			[
				fleet.ammoBank.toBuffer(),
				tokenProgramPK.toBuffer(),
				sageGameAcct.account.mints.ammo.toBuffer()
			],
			programPK
		);
		const currentAmmoCnt = parsedTokenAccount ? parsedTokenAccount.account.data.parsed.info.tokenAmount.uiAmount : 0;
		const resAmmoMax = Math.min(fleet.ammoCapacity, amount);

		let amountLoaded = 0;
		let transaction = null;
		if (currentAmmoCnt < resAmmoMax) {
			amountLoaded = resAmmoMax - currentAmmoCnt;
			cLog(1,`${FleetTimeStamp(fleet.label)} Loading Ammobanks: ${amountLoaded}`);
			let resp = await execCargoFromStarbaseToFleet(
				fleet,
				fleet.ammoBank,
				parsedTokenAccount ? parsedTokenAccount.pubkey : fleetAmmoToken,
				ammoMint.toString(),
				ammoCargoTypeAcct,
				starbaseCoords,
				amountLoaded,
				false,
				returnTx
			);
			if(resp && !resp.name) {
				transaction = resp.tx;
			}
		}

		return { amountLoaded, transaction };
	}



	async function resetFleetState(fleet) {
		if ((fleet.state.includes('ERROR') && !fleet.state.includes('⌛')) || fleet.state.includes('STOPPED')) {
			let userFleetIndex = userFleets.findIndex(item => {return item.publicKey == fleet.publicKey});
			cLog(1,`${FleetTimeStamp(fleet.label)} Manual request for resetting the fleet state`);
			updateFleetState(fleet,'ERROR: Trying to restart ...',true); // keep string "ERROR" for now to prevent an early start of operateFleet()

			fleet.exitWarpSubwarpPending = 0;
			fleet.exitSubwarpWillBurnFuel = 0;

			let fleetAcctInfo = await getAccountInfo(fleet.label, 'full fleet info', fleet.publicKey);
			let [fleetState, extra] = getFleetState(fleetAcctInfo);
			let fleetCoords = fleetState == 'Idle' && extra ? extra : [];

			//now we have all necessary info, let's do the reset
			fleet.startingCoords = fleetCoords;
			fleet.iterCnt=0;
			fleet.resupplying=false;
			fleet.justResupplied=false;
			fleet.moveTarget = '';
			fleet.stopping = false;
			fleet.scanAutoMoveTo = null;
			fleet.scanForceResupply = false;
			fleet.scanStartForcedResupply = false;
			fleet.scanLastFuelAmount = undefined;

			//updateFleetState(fleet, fleetState, true);
			updateFleetState(fleet, 'Starting', true);
			fleet.state = fleetState; // overwrite "starting" with the real state but don't display it - just like in toggleAssistant
		}
		else {
			fleet.stopping = true;
			updateFleetState(fleet, 'Stopping ...');
		}
	}


	function updateAssistStatus(fleet) {
		sisRegistrarStatus(fleet);
		try { sisStatusLinha(fleet); } catch (e) { cLog(2, 'Sis-Bot: painel', e); }
	}

    async function updateAssistStarbaseStatus(starbases) {
        document.querySelectorAll('#assistStarbaseStatus .assist-modal-body table .assist-starbase-row').forEach(e => e.remove());

		for (let starbase of starbases) {
			let starbaseRow = document.createElement('tr');
			starbaseRow.classList.add('assist-starbase-row');
			let starbaseLabel = document.createElement('span');
			setTxt(starbaseLabel, starbase.name);
			let starbaseLabelTd = document.createElement('td');
			starbaseLabelTd.appendChild(starbaseLabel);
			let starbaseCoords = document.createElement('span');
			setTxt(starbaseCoords, starbase.coords);
			let starbaseCoordsTd = document.createElement('td');
			starbaseCoordsTd.appendChild(starbaseCoords);
			let starbaseFood = document.createElement('span');
			setTxt(starbaseFood, `${starbase.foodBalanceHr}hr [${(starbase.foodBalancePerc*100).toFixed(2)}%]`);
			let starbaseFoodTd = document.createElement('td');
			starbaseFoodTd.appendChild(starbaseFood);
			let starbaseTools = document.createElement('span');
			setTxt(starbaseTools, `${starbase.toolBalanceHr}hr [${(starbase.toolBalancePerc*100).toFixed(2)}%]`);
			let starbaseToolsTd = document.createElement('td');
			starbaseToolsTd.appendChild(starbaseTools);
			starbaseRow.appendChild(starbaseLabelTd);
			starbaseRow.appendChild(starbaseCoordsTd);
			starbaseRow.appendChild(starbaseFoodTd);
			starbaseRow.appendChild(starbaseToolsTd);
			let targetElem = document.querySelector('#assistStarbaseStatus .assist-modal-body table');
			targetElem.appendChild(starbaseRow);
		}
	}

	function updateFleetState(fleet, newState, overrideError) {
        if ((typeof fleet.state == 'undefined') || !fleet.state.includes('ERROR') || overrideError) {
            fleet.state = newState;
            updateAssistStatus(fleet);
        }
	}

	function buildScanBlock(destX, destY, overridePattern, overridePatternLength) {
		let { scanBlockPattern, scanBlockLength }  = globalSettings;
		if(typeof overridePattern != "undefined" && overridePattern != '') {
			scanBlockPattern = overridePattern;
		}
		if(overridePatternLength) {
			scanBlockLength = overridePatternLength;
		}

		destX = Number(destX);
		destY = Number(destY);
		let scanBlock = [];
		if (isNaN(destX) || isNaN(destY)) return scanBlock;

		const tip = scanBlockLength - 1;

		if(scanBlockPattern == 'ring') {
			scanBlock.push([destX, destY]);
			scanBlock.push([destX+1, destY]);
			scanBlock.push([destX+2, destY]);
			scanBlock.push([destX+2, destY+1]);
			scanBlock.push([destX+2, destY+2]);
			scanBlock.push([destX+1, destY+2]);
			scanBlock.push([destX, destY+2]);
			scanBlock.push([destX, destY+1]);
		}
		else if(scanBlockPattern == 'spiral') {
			scanBlock.push([destX, destY]);
			scanBlock.push([destX-1, destY-1]);
			scanBlock.push([destX, destY-1]);
			scanBlock.push([destX+1, destY-1]);
			scanBlock.push([destX+1, destY]);
			scanBlock.push([destX+1, destY+1]);
			scanBlock.push([destX, destY+1]);
			scanBlock.push([destX-1, destY+1]);
			scanBlock.push([destX-1, destY]);
		}
		else if(scanBlockPattern == 'sly') {
			scanBlock.push([destX, destY]);
			scanBlock.push([destX-1, destY+1]);
			scanBlock.push([destX-2, destY+1]);
			scanBlock.push([destX-3, destY]);
			scanBlock.push([destX-3, destY-1]);
			scanBlock.push([destX-2, destY-2]);
			scanBlock.push([destX-1, destY-2]);
			scanBlock.push([destX, destY-3]);
			scanBlock.push([destX, destY-4]);
			scanBlock.push([destX-1, destY-5]);
			scanBlock.push([destX-2, destY-5]);
			scanBlock.push([destX-3, destY-4]);
		}
		else if(scanBlockPattern == 'up') {
			for(let i=0; i < scanBlockLength; i++) scanBlock.push([destX, destY + i]);
			for(let i=0; i < scanBlockLength; i++) scanBlock.push([destX + 1, destY + (tip - i)]);
		}
		else if(scanBlockPattern == 'down') {
			for(let i=0; i < scanBlockLength; i++) scanBlock.push([destX, destY - i]);
			for(let i=0; i < scanBlockLength; i++) scanBlock.push([destX + 1, destY - (tip - i)]);
		}
		else if(scanBlockPattern == 'left') {
			for(let i=0; i < scanBlockLength; i++) scanBlock.push([destX - i, destY]);
			for(let i=0; i < scanBlockLength; i++) scanBlock.push([destX - (tip - i), destY + 1]);
		}
		else if(scanBlockPattern == 'right') {
			for(let i=0; i < scanBlockLength; i++) scanBlock.push([destX + i, destY]);
			for(let i=0; i < scanBlockLength; i++) scanBlock.push([destX + (tip - i), destY + 1]);
		}
		else {
			//Default to square
			scanBlock.push([destX, destY]);
			scanBlock.push([destX+1, destY]);
			scanBlock.push([destX+1, destY+1]);
			scanBlock.push([destX, destY+1]);
		}

		return scanBlock;
	}

	async function clearErrors() {
		errorLog = [];
		errorLogIndex = 0;
		let newErrorLog = { "index": errorLogIndex, "messages": errorLog };
		await GM.setValue('ErrorLog', JSON.stringify(newErrorLog));
		await reloadErrors();
	}
	async function reloadErrors() {
		await assistErrorToggle();
		await assistErrorToggle();
	}




	async function assistErrorToggle() {
		let targetElem = document.querySelector('#errorModal');
		if (targetElem.style.display === 'none') {
			targetElem.style.display = 'block';
			let importText = document.querySelector('#errorText');
			let curIdx=errorLogIndex - 1;
			if(curIdx < 0) curIdx = errorLogMaxEntries - 1;
			importText.value = '';
			for(let i=0; i<errorLogMaxEntries; i++) {
				if(!errorLog[curIdx]) break;
				importText.value += errorLog[curIdx] + "\n\n";
				curIdx--;
				if(curIdx<0) curIdx = errorLogMaxEntries - 1;
			}
		} else {
			targetElem.style.display = 'none';
		}
	}

	async function assistStatToggle(el) { //statsadd
		let targetElem = document.querySelector(el);
		if (targetElem.style.display === 'none') {
		    targetElem.style.display = 'block';
		} else {
		    targetElem.style.display = 'none';
		}
	}



	// [Sis-Ágora] o campo #mySecretKey é só de escrita: vazio = mantém a chave salva; marcado "apagar" = remove.





    async function assistStatusToggle() { sisPainelRecolher(); } // [Sis-Ágora] o painel de Status fica sempre na tela: fechar/Status = recolher ou expandir


    async function assistStarbaseStatusToggle() {
        let targetElem = document.querySelector('#assistStarbaseStatus');
        if (targetElem.style.display === 'none') {
            let starbases = await getStarbaseUpkeepStatus();
            updateAssistStarbaseStatus(starbases);
            targetElem.style.display = 'block';
        } else {
            targetElem.style.display = 'none';
        }
    }


	async function assistProfileToggle(profiles) {
		return new Promise(async resolve => {
				let targetElem = document.querySelector('#profileModal');
				if (targetElem.style.display === 'none' && profiles) {
						targetElem.style.display = 'block';
						let contentElem = document.querySelector('#profileDiv');
						let profileSelect = document.createElement('select');
						profileSelect.size = profiles.length + 1;
						profileSelect.style.padding = '2px 10px';
						preencherOpcoes(profileSelect, profiles.map(profile => [profile.profile, profile.name + '  [' + profile.profile + ']']));
						contentElem.append(profileSelect);
						profileSelect.onchange = function() {
								cLog(2, 'assistProfileToggle: profileSelect.value', profileSelect.value);
								let selected = profiles.find(o => o.profile === profileSelect.value);
								assistProfileToggle(null);
								resolve(selected);
						}
				} else {
						targetElem.style.display = 'none';
						resolve(null);
				}
		});
	}

	async function handleUndockAll() {
		for (let i=0, n=userFleets.length; i < n; i++) {
			let fleetAcctInfo = await solanaReadConnection.getAccountInfo(userFleets[i].publicKey);
			let [fleetState, extra] = getFleetState(fleetAcctInfo);
			if (fleetState === 'StarbaseLoadingBay') {
				let starbase = await sageProgram.account.starbase.fetch(extra.starbase);
				let coords = starbase.sector[0].toNumber() + ',' + starbase.sector[1].toNumber();
				await execUndock(userFleets[i], coords);
			}
		}
	}


	async function handleMovement(i, moveDist, moveX, moveY, isStarbaseAndWarpSubwarp) {
		let moveTime = 1;
		let warpCooldownFinished = 0;
		let fleetAcctInfo = await getAccountInfo(userFleets[i].label, 'full fleet info', userFleets[i].publicKey);
		let [fleetState, extra] = getFleetState(fleetAcctInfo, userFleets[i]);

		//Fleet idle and needs to be moved?
		if (fleetState == 'Idle' && extra.length > 1 && moveDist && moveX !== null && moveX !== '' && moveY != null && moveY !== '') {
			if (extra[0] !== moveX || extra[1] !== moveY) {
				let warpCost = calcWarpFuelReq(userFleets[i], extra, [moveX, moveY], isStarbaseAndWarpSubwarp);
				cLog(4, `${FleetTimeStamp(userFleets[i].label)} warpCost: ${warpCost}`);
				let subwarpCost = calculateSubwarpFuelBurn(userFleets[i], moveDist);
				let fleetCurrentFuelTank = await solanaReadConnection.getParsedTokenAccountsByOwner(userFleets[i].fuelTank, {programId: tokenProgramPK});
				let currentFuel = fleetCurrentFuelTank.value.find(item => item.account.data.parsed.info.mint === sageGameAcct.account.mints.fuel.toString());
				let currentFuelCnt = currentFuel ? currentFuel.account.data.parsed.info.tokenAmount.uiAmount - userFleets[i].exitSubwarpWillBurnFuel : 0;

				let fleetCurrentCargo = await solanaReadConnection.getParsedTokenAccountsByOwner(userFleets[i].cargoHold, {programId: tokenProgramPK});
				let currentCargoFuel = fleetCurrentCargo.value.find(item => item.account.data.parsed.info.mint === sageGameAcct.account.mints.fuel.toString());
				let currentCargoFuelCnt = currentCargoFuel ? currentCargoFuel.account.data.parsed.info.tokenAmount.uiAmount : 0;

				let shortSubwarp = moveDist < 1.5 && globalSettings.subwarpShortDist ? true : false;
				let fleetAcctData = sageProgram.coder.accounts.decode('fleet', fleetAcctInfo.data);
				let warpCooldownExpiresAt = fleetAcctData.warpCooldownExpiresAt.toNumber() * 1000;
				let forceSubwarp = false;
				if(!shortSubwarp && userFleets[i].moveType == 'warp-subwarp-warp' && Date.now() < warpCooldownExpiresAt) {
					let warpCooldownLeft = (warpCooldownExpiresAt - Date.now()) / 1000;
					let fullDistanceLeft = calculateMovementDistance(extra, [moveX,moveY]);
					let subwarpDistanceWhileCooldown = warpCooldownLeft * (userFleets[i].subwarpSpeed / 1e6);
					let maxSubwarpDistancePastCooldown = (userFleets[i].warpCooldown - warpCooldownLeft) * (userFleets[i].subwarpSpeed / 1e6); //how far could we subwarp in the past cooldown time?
					//if the past warp cooldown is too low that it would've allowed a subwarp, we subwarp if we can do at least 0.5 AU in the remaining time. If a lot more of the cooldown is done, we check if we can do at least 0.71 AU because moveX and moveY can be off by 0.49999 (=0.5) each (due to rounding) and sqrt(0.5*0.5+0.5*0.5)=0.707, so it is possible that the previous subwarp was 0.707 units off, but we don't want to subwarp again
					if(fullDistanceLeft > 0 && ((maxSubwarpDistancePastCooldown < 0.5 && subwarpDistanceWhileCooldown >= 0.5) || subwarpDistanceWhileCooldown >= 0.71)) {
						if(subwarpDistanceWhileCooldown < 1.415) subwarpDistanceWhileCooldown = 1.415; // we subwarp at least 1 sector in the direction of the target and we choose 1.415 (=sqrt(2) rounded up), so all 8 directions have the same chance
						let distanceRatio = Math.min(1, subwarpDistanceWhileCooldown / fullDistanceLeft);
						if(distanceRatio < 1) {
							let moveXNew = Math.round((moveX-extra[0]) * distanceRatio) + extra[0];
							let moveYNew = Math.round((moveY-extra[1]) * distanceRatio) + extra[1];
							let moveDistNew = calculateMovementDistance(extra, [moveXNew,moveYNew]);
							//if we would be near the target after subwarp, we subwarp the full distance if "subwarpShortDist" is enabled, so we only set the new coords if this isn't the case
							if(!globalSettings.subwarpShortDist || moveDist - moveDistNew >= 1.5) {
								moveX = moveXNew;
								moveY = moveYNew;
								moveDist = moveDistNew;

								//if we don't subwarp the full way to the target, we need to save the target (exactly as in the warp block below), just in case SLYA gets reloaded
								const fleetPK = userFleets[i].publicKey.toString();
								const fleetSavedData = await GM.getValue(fleetPK, '{}');
								const fleetParsedData = JSON.parse(fleetSavedData);
								fleetParsedData.moveTarget = userFleets[i].moveTarget;
								await GM.setValue(fleetPK, JSON.stringify(fleetParsedData));
							}
						}
						forceSubwarp = true;
					}
				}

				//Should a warp be attempted?
				//if (userFleets[i].moveType == 'warp' && (currentFuelCnt + currentCargoFuelCnt) >= warpCost && !shortSubwarp) {
				if ((userFleets[i].moveType == 'warp' || userFleets[i].moveType == 'warp-subwarp-warp' || isStarbaseAndWarpSubwarp) && (currentFuelCnt + currentCargoFuelCnt) >= warpCost && !shortSubwarp && !forceSubwarp) {
					//let fleetAcctData = sageProgram.coder.accounts.decode('fleet', fleetAcctInfo.data);
					//let warpCooldownExpiresAt = fleetAcctData.warpCooldownExpiresAt.toNumber() * 1000;

					//Wait for cooldown
					while (Date.now() < warpCooldownExpiresAt) {
						if (!userFleets[i].state.includes('Warp C/D')) {
							const warpCDExpireTimeStr = `[${TimeToStr(new Date(warpCooldownExpiresAt))}]`;
							cLog(1,`${FleetTimeStamp(userFleets[i].label)} Awaiting Warp C/D ${warpCDExpireTimeStr}`);
							updateFleetState(userFleets[i], `Warp C/D ${warpCDExpireTimeStr}`);
						}

						//await wait(Math.max(1000, warpCooldownExpiresAt - Date.now()));
						if(warpCooldownExpiresAt - Date.now() < 5000) await wait(Math.max(1000, warpCooldownExpiresAt - Date.now()));
						else await wait(5000);
						if(userFleets[i].stopping) return;
					}
					await wait(2000); //Extra wait to ensure accuracy

					const fleetPK = userFleets[i].publicKey.toString();
					const fleetSavedData = await GM.getValue(fleetPK, '{}');
					const fleetParsedData = JSON.parse(fleetSavedData);
					const assignment = fleetParsedData.assignment;

					//Calculate next warp point if more than 1 is needed to arrive at final destination
					if (moveDist > userFleets[i].maxWarpDistance / 100) {
						[moveX, moveY] = calcNextWarpPoint(userFleets[i].maxWarpDistance, extra, [moveX, moveY]);

						//Saves temporary waypoints for transports in case the page is refreshed mid-journey while using warp
						//const fleetPK = userFleets[i].publicKey.toString();
						//const fleetSavedData = await GM.getValue(fleetPK, '{}');
						//const fleetParsedData = JSON.parse(fleetSavedData);
						//cLog(3, `${FleetTimeStamp(userFleets[i].label)} moveTargets`, fleetParsedData.moveTarget, userFleets[i].moveTarget);
						fleetParsedData.moveTarget = userFleets[i].moveTarget;
						await GM.setValue(fleetPK, JSON.stringify(fleetParsedData));

						//Update distance based on new warp target
						moveDist = calculateMovementDistance(extra, [moveX,moveY]);
					}

					moveTime = calculateWarpTime(userFleets[i], moveDist);
					const warpResult = await execWarp(userFleets[i], moveX, moveY, moveTime);
					await sendToInflux(`movement,fleet=${influxEscape(userFleets[i].label)},fromX=${extra[0]},fromY=${extra[1]},toX=${moveX},toY=${moveY},assignment=${assignment} type="warp",burnedFuel=${moveDist*(userFleets[i].warpFuelConsumptionRate/100)},moveTime=${moveTime},moveDist=${moveDist}`);
					if(userFleets[i].scanLastFuelAmount) userFleets[i].scanLastFuelAmount -= moveDist*(userFleets[i].warpFuelConsumptionRate/100);
					warpCooldownFinished = warpResult.warpCooldownFinished;
				} else if (currentFuelCnt + currentCargoFuelCnt >= subwarpCost) {
					moveTime = calculateSubwarpTime(userFleets[i], moveDist);
					await execSubwarp(userFleets[i], moveX, moveY, moveTime);
					const fleetPK = userFleets[i].publicKey.toString();
					const fleetSavedData = await GM.getValue(fleetPK, '{}');
					const fleetParsedData = JSON.parse(fleetSavedData);
					const assignment = fleetParsedData.assignment;
					await sendToInflux(`movement,fleet=${influxEscape(userFleets[i].label)},fromX=${extra[0]},fromY=${extra[1]},toX=${moveX},toY=${moveY},assignment=${assignment} type="subwarp",burnedFuel=${moveDist*(userFleets[i].subwarpFuelConsumptionRate/100)},moveTime=${moveTime},moveDist=${moveDist}`);
					if(userFleets[i].scanLastFuelAmount) userFleets[i].scanLastFuelAmount -= moveDist*(userFleets[i].subwarpFuelConsumptionRate/100);
				} else {
					cLog(1,`${FleetTimeStamp(userFleets[i].label)} Unable to move, lack of fuel`);
					updateFleetState(userFleets[i], 'ERROR: Not enough fuel');
					sisEvento('alerta', userFleets[i].label, 'Sem combustível suficiente para se mover');
				}
			}
		}

		await wait(2000); //Allow time for RPC to update
		fleetAcctInfo = await getAccountInfo(userFleets[i].label, 'full fleet info', userFleets[i].publicKey);
		[fleetState, extra] = getFleetState(fleetAcctInfo, userFleets[i]);
		let warpFinish = fleetState == 'MoveWarp' ? extra.warpFinish.toNumber() * 1000 : 0;
		let subwarpFinish = fleetState == 'MoveSubwarp' ? extra.arrivalTime.toNumber() * 1000 : 0;
		let endTime = warpFinish > subwarpFinish ? warpFinish : subwarpFinish;

		const calcEndTime = Date.now() + moveTime * 1000;
		cLog(3, `${FleetTimeStamp(userFleets[i].label)} Expected arrival (chain): ${TimeToStr(new Date(endTime))}`);
		cLog(3, `${FleetTimeStamp(userFleets[i].label)} Expected arrival (calc): ${TimeToStr(new Date(calcEndTime))}`);

		//Sometimes the chain returns null, use calculated time as fallback
		if(!endTime) endTime = calcEndTime;

		userFleets[i].moveEnd = endTime;
		await wait(moveTime * 1000);
		while (endTime > Date.now()) {
			const newFleetState = 'Move [' + TimeToStr(new Date(endTime)) + ']';
			updateFleetState(userFleets[i], newFleetState);
			await wait(Math.max(1000, endTime - Date.now()));
		}

		//await wait(2000);

		let localQueueExitWarpSubwarp = false;
		if(globalSettings.queueExitWarpSubwarp) {
			//exclude scanning fleets from this feature, because we can't bundle a scan instruction with any other ix
			let fleetSavedData = await GM.getValue(userFleets[i].publicKey.toString(), '{}');
			let fleetParsedData = JSON.parse(fleetSavedData);
			if (fleetParsedData.assignment != 'Scan') localQueueExitWarpSubwarp = true;
		}

		if (fleetState == 'MoveWarp') {
			if (localQueueExitWarpSubwarp) {
				userFleets[i].exitWarpSubwarpPending = 1;
				updateFleetState(userFleets[i], 'Idle');
			} else {
				await execExitWarp(userFleets[i]);
			}
		} else if (fleetState == 'MoveSubwarp'){
			if (localQueueExitWarpSubwarp) {
				userFleets[i].exitWarpSubwarpPending = 2;
				updateFleetState(userFleets[i], 'Idle');
			} else {
				await execExitSubwarp(userFleets[i]);
			}
		}

		fleetAcctInfo = await getAccountInfo(userFleets[i].label, 'full fleet info', userFleets[i].publicKey);
		[fleetState, extra] = getFleetState(fleetAcctInfo, userFleets[i]);
		if (fleetState == 'Idle' && extra) {
				let targetX = userFleets[i].moveTarget != '' && userFleets[i].moveTarget.split(',').length > 1 ? userFleets[i].moveTarget.split(',')[0].trim() : '';
				let targetY = userFleets[i].moveTarget != '' && userFleets[i].moveTarget.split(',').length > 1 ? userFleets[i].moveTarget.split(',')[1].trim() : '';
				if (extra[0] == targetX && extra[1] == targetY) {
						//bugfix: moveTarget is string based, not an array!
						//userFleets[i].moveTarget = [];
						userFleets[i].moveTarget = '';
						let fleetSavedData = await GM.getValue(userFleets[i].publicKey.toString(), '{}');
						let fleetParsedData = JSON.parse(fleetSavedData);
						let fleetPK = userFleets[i].publicKey.toString();
						fleetParsedData.moveTarget = userFleets[i].moveTarget;
						await GM.setValue(fleetPK, JSON.stringify(fleetParsedData));
				}
		}

		return warpCooldownFinished;
	}

	async function saveScanEnd(i) {
		const fleetPK = userFleets[i].publicKey.toString();
		const fleetSavedData = await GM.getValue(fleetPK, '{}');
		const fleetParsedData = JSON.parse(fleetSavedData);
		fleetParsedData.scanEnd = userFleets[i].scanEnd;
		await GM.setValue(fleetPK, JSON.stringify(fleetParsedData));
	}

	async function handleScan(i, fleetCoords, destCoords) {
		let fleetCurrentCargo = await solanaReadConnection.getParsedTokenAccountsByOwner(userFleets[i].cargoHold, {programId: tokenProgramPK});
		let cargoCnt = fleetCurrentCargo.value.reduce((n, {account}) => n + account.data.parsed.info.tokenAmount.uiAmount * cargoItems.find(r => r.token == account.data.parsed.info.mint).size, 0);
		let currentFoodAcct = fleetCurrentCargo.value.find(item => item.pubkey.toString() === userFleets[i].foodToken.toString());
		let currentFoodCnt = currentFoodAcct && currentFoodAcct.account.data.parsed.info.delegatedAmount ? currentFoodAcct.account.data.parsed.info.delegatedAmount.uiAmount : 0;

		if(
			//Cargo full check for data runner fleets
			((userFleets[i].scanCost == 0) && (userFleets[i].cargoCapacity - cargoCnt < userFleets[i].sduPerScan)) ||
			//Food count check for regular scanning fleets
			(currentFoodCnt < userFleets[i].scanCost)
			|| (userFleets[i].scanForceResupply)
		) {
			await handleResupply(i, fleetCoords);
			return;
		}

		let moved = false;
		if ((fleetCoords[0] !== destCoords[0] || fleetCoords[1] !== destCoords[1])) {
			if (!userFleets[i].state.includes('Warp C/D')) {
				const starbaseCoords = ConvertCoords(userFleets[i].starbaseCoord);
				//cLog(4, `${FleetTimeStamp(userFleets[i].label)} starbaseCoords: ${starbaseCoords}`);

				let isStarbaseAndWarpSubwarp = (userFleets[i].moveType == 'warpsubwarp' && fleetCoords[0] == starbaseCoords[0] && fleetCoords[1] == starbaseCoords[1]);

				let fuelNeeded = 0;
				if (userFleets[i].moveType == 'warp' || userFleets[i].moveType == 'warp-subwarp-warp' || isStarbaseAndWarpSubwarp) {
					const warpCostFromFleetToDest = calcWarpFuelReq(userFleets[i], fleetCoords, destCoords, isStarbaseAndWarpSubwarp);
					//const warpCostFromDestToStarbase = calcWarpFuelReq(userFleets[i], destCoords, starbaseCoords);
					const warpCostFromDestToStarbase = userFleets[i].moveType == 'warpsubwarp' ? calculateSubwarpFuelBurn(userFleets[i], calculateMovementDistance(destCoords, starbaseCoords)) : calcWarpFuelReq(userFleets[i], destCoords, starbaseCoords);
					fuelNeeded = warpCostFromFleetToDest + warpCostFromDestToStarbase;
					cLog(4, `${FleetTimeStamp(userFleets[i].label)} Warp cost to target: ${warpCostFromFleetToDest}`);
					cLog(4, `${FleetTimeStamp(userFleets[i].label)} Warp cost to return: ${warpCostFromDestToStarbase}`);
					cLog(4, `${FleetTimeStamp(userFleets[i].label)} Warp cost total: ${fuelNeeded}`);
				} else {
					const distToTarget = calculateMovementDistance(fleetCoords, destCoords);
					const distFromTargetToStarbase = calculateMovementDistance(destCoords, starbaseCoords);
					fuelNeeded = calculateSubwarpFuelBurn(userFleets[i], distToTarget + distFromTargetToStarbase);
					cLog(4, `${FleetTimeStamp(userFleets[i].label)} Subwarp cost total: ${fuelNeeded}`);
				}

				const fleetCurrentFuelTank = await solanaReadConnection.getParsedTokenAccountsByOwner(userFleets[i].fuelTank, {programId: tokenProgramPK});
				const currentFuel = fleetCurrentFuelTank.value.find(item => item.account.data.parsed.info.mint === sageGameAcct.account.mints.fuel.toString());
				const currentFuelCnt = currentFuel ? currentFuel.account.data.parsed.info.tokenAmount.uiAmount : 0;
				const fuelReadout = `Fuel: need ${Math.round(fuelNeeded)} / have ${Math.round(currentFuelCnt)}`;
				userFleets[i].scanLastFuelAmount = currentFuelCnt;
				if (currentFuelCnt > fuelNeeded) {
						cLog(1, `${FleetTimeStamp(userFleets[i].label)}`, fuelReadout);
						let moveDist = calculateMovementDistance(fleetCoords, destCoords);
						if (moveDist > 0) {
								moved = true;
								const scanEndsIn = Math.max(0, userFleets[i].scanEnd - Date.now());
								//Clamp the scan end time to the cooldown if it is higher (due to paused scanning)
								userFleets[i].scanEnd = (scanEndsIn > userFleets[i].scanCooldown * 1000 ? userFleets[i].scanCooldown * 1000 : scanEndsIn) + Date.now(); //fixed a wrong time calculation
								await saveScanEnd(i);
								await handleMovement(i, moveDist, destCoords[0], destCoords[1], isStarbaseAndWarpSubwarp);
								cLog(1,`${FleetTimeStamp(userFleets[i].label)} Movement finished`);
								userFleets[i].scanStrikes = 0;
						} else {
								cLog(1,`${FleetTimeStamp(userFleets[i].label)} Skipping movement`);
						}
				} else {
					cLog(1, `${FleetTimeStamp(userFleets[i].label)} ${fuelReadout} (low)`);
					if(globalSettings.scanResupplyOnLowFuel) {
						await handleResupply(i, fleetCoords);
						moved = true;
					} else {
						cLog(3, `${FleetTimeStamp(userFleets[i].label)} Moved:`, moved);
					}
				}
			}
		}

		if(userFleets[i].stopping) return;

		if (!moved && Date.now() > userFleets[i].scanEnd) {
			userFleets[i].lastScanCoord = userFleets[i].destCoord;
			if(!userFleets[i].scanStrikes) userFleets[i].scanStrikes = 0;
			const scanResult = await execScan(userFleets[i], fleetCoords);
			const changesSDU = scanResult ? getBalanceChange(scanResult, userFleets[i].sduToken.toString()) : {postBalance: userFleets[i].sduCnt, preBalance: userFleets[i].sduCnt};
			const changesFood = scanResult ? getBalanceChange(scanResult, userFleets[i].foodToken.toString()) : {postBalance: userFleets[i].foodCnt - userFleets[i].scanCost, preBalance: userFleets[i].foodCnt};
			const scanConditionLog = scanResult && scanResult.meta.logMessages ? scanResult.meta.logMessages.find(item => item.startsWith("Program log: SDU probability:")) : null;
			const scanCondition = scanConditionLog ? (Number(scanConditionLog.split(' ').pop())*100).toFixed(4) : 0;
			userFleets[i].foodCnt = changesFood.postBalance;
			userFleets[i].sduCnt = changesSDU.postBalance;

			let sduFound = 0;
			if (changesSDU.postBalance != changesSDU.preBalance) {
					sduFound = changesSDU.postBalance - changesSDU.preBalance;
					userFleets[i].scanSkipCnt = 0;
			}
			await alterStats('SDUs found',undefined,sduFound,'SDU',0);

			cLog(1,`${FleetTimeStamp(userFleets[i].label)} 📡 ${Math.round(scanCondition)}%${sduFound > 0 ? ` | 💰 FOUND: ${sduFound}` : ''}`);
			if(scanCondition < userFleets[i].scanMin2) {
				userFleets[i].scanStrikes = globalSettings.scanStrikeCount;
				cLog(3,`${FleetTimeStamp(userFleets[i].label)} ⚡⚡️ Instant strike out`);
			} else if(scanCondition < userFleets[i].scanMin3 && sduFound > 0) {
				userFleets[i].scanStrikes = globalSettings.scanStrikeCount;
				cLog(3,`${FleetTimeStamp(userFleets[i].label)} ⚡⚡️ Instant strike out`);
			} else if(scanCondition < userFleets[i].scanMin) {
				userFleets[i].scanStrikes++;
				cLog(3,`${FleetTimeStamp(userFleets[i].label)} ⚡️ Strike ${userFleets[i].scanStrikes} / ${globalSettings.scanStrikeCount}`);
			} else {
				userFleets[i].scanStrikes = 0;
				cLog(3,`${FleetTimeStamp(userFleets[i].label)} 🔙 Strikes reset`);
			}

			const struckOut = userFleets[i].scanStrikes >= globalSettings.scanStrikeCount;
			if(struckOut) {
				cLog(3,`${FleetTimeStamp(userFleets[i].label)} 🔺 Sector struck out`);
				userFleets[i].scanSkipCnt++;
			}

			let scanBlockPattern = globalSettings.scanBlockPattern;
			if(userFleets[i].scanPattern) scanBlockPattern=userFleets[i].scanPattern;

			//Iterate pattern positioning id (or reset to 0 if reached end)
			if(userFleets[i].scanMove && struckOut)
				if(!scanBlockPattern.includes('auto')) {
					userFleets[i].scanBlockIdx = userFleets[i].scanBlockIdx > userFleets[i].scanBlock.length - 2 ? 0 : userFleets[i].scanBlockIdx+1;
				} else {
					await handleScanAutoMovement(i, fleetCoords);
				}

			const needPause =
				(struckOut && !userFleets[i].scanMove) ||
				(userFleets[i].scanMove && userFleets[i].scanSkipCnt >= userFleets[i].scanBlock.length - 1 && !scanBlockPattern.includes('auto'))

            let newState;
			if (needPause) {
				//userFleets[i].scanEnd = Date.now() + (globalSettings.scanPauseTime * 1000);
				userFleets[i].scanEnd = Date.now() + Math.max(globalSettings.scanPauseTime * 1000, userFleets[i].scanCooldown * 1000 + 2000);
				//userFleets[i].state = `Scanning Paused [${TimeToStr(new Date(userFleets[i].scanEnd))}]`;
                newState = `Scanning Paused [${TimeToStr(new Date(userFleets[i].scanEnd))}]`;
				cLog(1,`${FleetTimeStamp(userFleets[i].label)} Scanning Paused due to low probability [${TimeToStr(new Date(userFleets[i].scanEnd))}]`);
				userFleets[i].scanSkipCnt = 0;
				userFleets[i].scanStrikes = 0;
			} else {
				let scanDelayMs = userFleets[i].scanCooldown * 1000 + 2000;
				if(sduFound) scanDelayMs = Math.max(scanDelayMs, globalSettings.scanSectorRegenTime * 1000);
				userFleets[i].scanEnd = Date.now() + scanDelayMs;
				//userFleets[i].state = `Scanned [${Math.round(scanCondition)}%]${sduFound ? ` +${sduFound}` : ''}`;
                newState = `Scanned [${Math.round(scanCondition)}%]${sduFound ? ` +${sduFound}` : ''}`;
			}

			//updateAssistStatus(userFleets[i]);
            updateFleetState(userFleets[i], newState);

			//Start resupply/unload immediately rather than waiting for scan cooldown
			//if(currentFoodCnt - userFleets[i].scanCost < userFleets[i].scanCost) userFleets[i].scanEnd = Date.now();
			if(((userFleets[i].scanCost == 0) && (userFleets[i].cargoCapacity - cargoCnt - sduFound < userFleets[i].sduPerScan)) || (currentFoodCnt - userFleets[i].scanCost < userFleets[i].scanCost)) userFleets[i].scanEnd = Date.now();

			//save the scan end time, so after a reload the fleet waits accordingly
			await saveScanEnd(i);

			let burnedFood = changesFood.preBalance - changesFood.postBalance;
			await sendToInflux(`sdu,fleet=${influxEscape(userFleets[i].label)},sectorX=${fleetCoords[0]},sectorY=${fleetCoords[1]} amount=${sduFound},burnedFood=${burnedFood},chance=${scanCondition},cargoRoomLeft=${userFleets[i].cargoCapacity - cargoCnt - sduFound}`);

		}
		else if (!moved && Date.now() < userFleets[i].scanEnd && userFleets[i].state == 'Idle') {
			userFleets[i].lastScanCoord = userFleets[i].destCoord;
			const scanCDExpireTimeStr = `[${TimeToStr(new Date(userFleets[i].scanEnd))}]`;
			updateFleetState(userFleets[i], 'Waiting for scan cooldown ' + scanCDExpireTimeStr);
			//await wait(userFleets[i].scanEnd - Date.now());
		}
	}

	function influxEscape(val) {
		return String(val ?? '').replaceAll("\\","\\\\").replaceAll(" ","\\ ").replaceAll(",","\\,").replaceAll("=","\\="); // [Sis-Ágora] nome ausente não pode derrubar a operação
	}

	// [Sis-Ágora] as linhas que iam para o Influx viram registros de produção enviados à VPS
	async function sendToInflux(msg) { try { sisInfluxParaProducao(msg); } catch (e) { cLog(1, 'SIS: produção', e); } }

	async function handleResupply(i, fleetCoords) {
		const errorWaitTime = 8 * 60 * 1000;
		const errorFuelRatio = 0.75;

		async function unloadSDU() {
			cLog(1,`${FleetTimeStamp(userFleets[i].label)} Unloading SDU`);

			//Get SDU count
			const fleetCurrentCargo = await solanaReadConnection.getParsedTokenAccountsByOwner(userFleets[i].cargoHold, {programId: tokenProgramPK});
			const currentSduCnt = fleetCurrentCargo.value.find(item => item.pubkey.toString() === userFleets[i].sduToken.toString())
			const sduCount = currentSduCnt ? currentSduCnt.account.data.parsed.info.tokenAmount.uiAmount : 0;

			//Unload all but 1 to keep SDU token account open
			if (sduCount > 1) {
				const sduToUnload = sduCount - 1;
				await execCargoFromFleetToStarbase(userFleets[i], userFleets[i].cargoHold, sduItem.token, userFleets[i].starbaseCoord, sduToUnload);
				cLog(2,`${FleetTimeStamp(userFleets[i].label)} Unloaded ${sduToUnload} SDU`);
				userFleets[i].sduCnt = 1;
			}
		}

		async function loadFood() {
			cLog(1,`${FleetTimeStamp(userFleets[i].label)} Loading Food`);

			//Update this just in case it's missing
			//await getAccountInfo(userFleets[i].label, 'fleet repair kit token', userFleets[i].repairKitToken) || await createPDA(userFleets[i].repairKitToken, userFleets[i].cargoHold, new solanaWeb3.PublicKey(toolItem.token), userFleets[i]);

			//Calculate occupied cargo count
			const preLoadCargo = await solanaReadConnection.getParsedTokenAccountsByOwner(userFleets[i].cargoHold, {programId: tokenProgramPK});
			const preLoadCargoCount = preLoadCargo.value.reduce((n, {account}) => n + account.data.parsed.info.tokenAmount.uiAmount, 0);

			let foodCargoTypeAcct = cargoTypes.find(item => item.account.mint.toString() == sageGameAcct.account.mints.food);
			//Fill remaining cargo with Food
			await execCargoFromStarbaseToFleet(
				userFleets[i],
				userFleets[i].cargoHold,
				userFleets[i].foodToken,
				foodItem.token,
				foodCargoTypeAcct,
				userFleets[i].starbaseCoord,
				userFleets[i].cargoCapacity - preLoadCargoCount
			);

			//Wait for RPC to catch up
			await wait(2000);

			//Update food count
			const foodCargo = await solanaReadConnection.getTokenAccountBalance(userFleets[i].foodToken);
			userFleets[i].foodCnt = foodCargo.value.amount;

			//Was enough food loaded?
			if (userFleets[i].foodCnt < userFleets[i].scanCost) {
				cLog(1,`${FleetTimeStamp(userFleets[i].label)} ERROR: Not enough food at starbase - waiting for more`);
				updateFleetState(userFleets[i], `ERROR: No food ⌛ ${TimeToStr(new Date(Date.now() + errorWaitTime))}`);

				//Wait a while before trying again
				await wait(errorWaitTime);
				await loadFood();
			} else {
				if(userFleets[i].state.includes('ERROR: No food ⌛')) {
					updateFleetState(userFleets[i], `Enough food loaded`, true);
				}
			}
		}

		async function loadFuel() {
			cLog(1,`${FleetTimeStamp(userFleets[i].label)} Refueling`);
			let fleetCurrentFuel = await solanaReadConnection.getParsedTokenAccountsByOwner(userFleets[i].fuelTank, {programId: tokenProgramPK});
			let currentFuelCnt = fleetCurrentFuel.value.find(item => item.pubkey.toString() === userFleets[i].fuelToken.toString())

			let fuelCargoTypeAcct = cargoTypes.find(item => item.account.mint.toString() == sageGameAcct.account.mints.fuel);
			await execCargoFromStarbaseToFleet(
				userFleets[i],
				userFleets[i].fuelTank,
				userFleets[i].fuelToken,
				fuelItem.token, fuelCargoTypeAcct,
				userFleets[i].starbaseCoord,
				userFleets[i].fuelCapacity - currentFuelCnt.account.data.parsed.info.tokenAmount.uiAmount
			);

			//Allow rpc to catch up
			await wait(2000);

			//Update fuel count
			fleetCurrentFuel = await solanaReadConnection.getParsedTokenAccountsByOwner(userFleets[i].fuelTank, {programId: tokenProgramPK});
			currentFuelCnt = fleetCurrentFuel.value.find(item => item.pubkey.toString() === userFleets[i].fuelToken.toString())
			userFleets[i].fuelCnt = currentFuelCnt.account.data.parsed.info.tokenAmount.uiAmount;

			//Was enough fuel loaded?
			if(userFleets[i].fuelCnt < userFleets[i].fuelCapacity * errorFuelRatio) {
				cLog(1,`${FleetTimeStamp(userFleets[i].label)} ERROR: Not enough fuel at starbase - waiting for more`);
				updateFleetState(userFleets[i], `ERROR: No Fuel ⌛ ${TimeToStr(new Date(Date.now() + errorWaitTime))}`);

				//Wait a while before trying again
				await wait(errorWaitTime);
				await loadFuel();
			} else {
				if(userFleets[i].state.includes('ERROR: No Fuel ⌛')) {
					updateFleetState(userFleets[i], `Enough fuel loaded`, true);
				}
			}
		}

		userFleets[i].resupplying = true;
		if(globalSettings.scanBlockResetAfterResupply) userFleets[i].scanBlockIdx = 0;
		const baseCoords = ConvertCoords(userFleets[i].starbaseCoord);

		if (CoordsEqual(fleetCoords, baseCoords)) { //At starbase
			cLog(1,`${FleetTimeStamp(userFleets[i].label)} Resupply: Docking at starbase`);
			await execDock(userFleets[i], userFleets[i].starbaseCoord);

			updateFleetState(userFleets[i], 'Unloading');
			await unloadSDU();

			updateFleetState(userFleets[i], 'Loading');
			if (userFleets[i].scanCost > 0) await loadFood();
			await loadFuel();

			//Redundancy check to ensure all PDAs are present
			await createScannerPDAs(userFleets[i]);

			//Update last op to prevent fleet stall flagging
			userFleets[i].lastOp = Date.now();

			userFleets[i].scanAutoMoveTo = null;
			userFleets[i].scanStartForcedResupply = false;
			userFleets[i].scanForceResupply = false;
			userFleets[i].scanLastFuelAmount = undefined;

			//Undock
			await execUndock(userFleets[i], userFleets[i].starbaseCoord);
		} else { //Not at starbase - move there
			let moveDist = calculateMovementDistance(fleetCoords, baseCoords);
			if (moveDist > 0 && !userFleets[i].state.includes('Warp C/D')) {
				cLog(1,`${FleetTimeStamp(userFleets[i].label)} Resupply: Moving to base`);
				updateFleetState(userFleets[i], 'Idle');
				await handleMovement(i, moveDist, baseCoords[0], baseCoords[1]);
				cLog(1,`${FleetTimeStamp(userFleets[i].label)} Resupply: Arrived at base`);
			} else {
				cLog(1,`${FleetTimeStamp(userFleets[i].label)} Resupply: Already at base`);
			}
		}

		userFleets[i].resupplying = false;
	}

	async function handleMining(i, fleetState, fleetCoords, fleetMining) {
		let destX = userFleets[i].destCoord.split(',')[0].trim();
		let destY = userFleets[i].destCoord.split(',')[1].trim();
		let starbaseX = userFleets[i].starbaseCoord.split(',')[0].trim();
		let starbaseY = userFleets[i].starbaseCoord.split(',')[1].trim();
        let mineItem = mineItems.find(item => item.account.mint.toString() === userFleets[i].mineResource);
		let resourceHardness = mineItem.account.resourceHardness;
		let planets = await getPlanetsFromCoords(destX, destY);
		let sageResource = null;
		let planet = null;
		for (let planetCheck of planets) {
            let resourceCheck = await getMineableResourceFromPlanet(planetCheck.publicKey.toString(), mineItem.publicKey.toString());
			if (sageResource === null && resourceCheck && resourceCheck.publicKey) {
				sageResource = resourceCheck;
				planet = planetCheck
			}
		}
		let systemRichness = 0;
		if (sageResource && sageResource.account) {
			systemRichness = sageResource.account.systemRichness;
		} else {
			let resShort = cargoItems.find(r => r.token == userFleets[i].mineResource).name;
			cLog(1,`${FleetTimeStamp(userFleets[i].label)} ERROR: ${resShort} not found at mining location`);
			updateFleetState(userFleets[i], `ERROR: ${resShort} not found at mining location`);
            return;
		}
		// fleet PDA
		let [fleetResourceToken] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
				[
						userFleets[i].cargoHold.toBuffer(),
						tokenProgramPK.toBuffer(),
						new solanaWeb3.PublicKey(userFleets[i].mineResource).toBuffer()
				],
				programPK
		);
		let [fleetFoodToken] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
				[
						userFleets[i].cargoHold.toBuffer(),
						tokenProgramPK.toBuffer(),
						sageGameAcct.account.mints.food.toBuffer()
				],
				programPK
		);
		let [fleetAmmoToken] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
				[
						userFleets[i].ammoBank.toBuffer(),
						tokenProgramPK.toBuffer(),
						sageGameAcct.account.mints.ammo.toBuffer()
				],
				programPK
		);
		let [fleetCargoAmmoToken] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
				[
						userFleets[i].cargoHold.toBuffer(),
						tokenProgramPK.toBuffer(),
						sageGameAcct.account.mints.ammo.toBuffer()
				],
				programPK
		);
		let [fleetFuelToken] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
				[
						userFleets[i].fuelTank.toBuffer(),
						tokenProgramPK.toBuffer(),
						new solanaWeb3.PublicKey(fuelItem.token).toBuffer()
				],
				programPK
		);

		let fleetCurrentFuelTank = await solanaReadConnection.getParsedTokenAccountsByOwner(userFleets[i].fuelTank, {programId: tokenProgramPK});
		let currentFuel = fleetCurrentFuelTank.value.find(item => item.account.data.parsed.info.mint === sageGameAcct.account.mints.fuel.toString());
		let fleetFuelAcct = currentFuel ? currentFuel.pubkey : fleetFuelToken;
		let currentFuelCnt = currentFuel ? currentFuel.account.data.parsed.info.tokenAmount.uiAmount - userFleets[i].exitSubwarpWillBurnFuel : 0;
		let fleetCurrentCargo = await solanaReadConnection.getParsedTokenAccountsByOwner(userFleets[i].cargoHold, {programId: tokenProgramPK});
		//todo: cargoCnt currently assumes that 1 rss always takes 1 of the cargo room
		let cargoCnt = fleetCurrentCargo.value.reduce((n, {account}) => n + account.data.parsed.info.tokenAmount.uiAmount, 0);
		let currentFood = fleetCurrentCargo.value.find(item => item.account.data.parsed.info.mint === sageGameAcct.account.mints.food.toString());
		let fleetFoodAcct = currentFood ? currentFood.pubkey : fleetFoodToken;
		let currentFoodCnt = currentFood ? currentFood.account.data.parsed.info.tokenAmount.uiAmount : 0;
		let currentResource = fleetCurrentCargo.value.find(item => item.account.data.parsed.info.mint === userFleets[i].mineResource);
		let fleetResourceAcct = currentResource ? currentResource.pubkey : fleetResourceToken;
		let currentResourceCnt = currentResource ? currentResource.account.data.parsed.info.tokenAmount.uiAmount : 0;
		let fleetCurrentAmmoBank = await solanaReadConnection.getParsedTokenAccountsByOwner(userFleets[i].ammoBank, {programId: tokenProgramPK});
		let currentAmmo = fleetCurrentAmmoBank.value.find(item => item.account.data.parsed.info.mint === sageGameAcct.account.mints.ammo.toString());
		let fleetAmmoAcct = currentAmmo ? currentAmmo.pubkey : fleetAmmoToken;
		let currentAmmoCnt = currentAmmo ? currentAmmo.account.data.parsed.info.tokenAmount.uiAmount : 0;

		let miningDuration = calculateMiningDuration(userFleets[i].cargoCapacity - cargoCnt, userFleets[i].miningRate, resourceHardness, systemRichness);
		let foodForDuration = Math.ceil(miningDuration * (userFleets[i].foodConsumptionRate / 10000));
		let ammoForDuration = Math.ceil(miningDuration * (userFleets[i].ammoConsumptionRate / 10000));
		ammoForDuration = Math.min(userFleets[i].ammoCapacity, ammoForDuration);

		let distToTarget = calculateMovementDistance(fleetCoords, [destX,destY]);
		let distReturn = calculateMovementDistance([destX,destY], [starbaseX,starbaseY]);

		cLog(4, `${FleetTimeStamp(userFleets[i].label)} handleMining -> fleet:`, fleetCoords, `starbase:`, [starbaseX, starbaseY], `target:`, [destX, destY]);

		let isStarbaseAndWarpSubwarp = (userFleets[i].moveType == 'warpsubwarp' && ((fleetCoords[0] == starbaseX && fleetCoords[1] == starbaseY) || (fleetCoords[0] == destX && fleetCoords[1] == destY)));
		let isSourceStarbaseAndWarpSubwarp = (userFleets[i].moveType == 'warpsubwarp' && (fleetCoords[0] == starbaseX && fleetCoords[1] == starbaseY));

		const warpCostToTarget = fleetCoords.length == 2 ? calcWarpFuelReq(userFleets[i], fleetCoords, [destX, destY], isSourceStarbaseAndWarpSubwarp) : 0;
		let warpCost = warpCostToTarget +  calcWarpFuelReq(userFleets[i], [destX, destY], [starbaseX, starbaseY], userFleets[i].moveType == 'warpsubwarp') + userFleets[i].planetExitFuelAmount;
		let halfWarpCost = warpCostToTarget + calculateSubwarpFuelBurn(userFleets[i], distReturn) + userFleets[i].planetExitFuelAmount;
		let subwarpCost = calculateSubwarpFuelBurn(userFleets[i], distToTarget) + calculateSubwarpFuelBurn(userFleets[i], distReturn) + userFleets[i].planetExitFuelAmount;
		let fuelNeeded = userFleets[i].planetExitFuelAmount;
		if (userFleets[i].moveType == 'warp' || userFleets[i].moveType == 'warp-subwarp-warp' || isStarbaseAndWarpSubwarp) {
			fuelNeeded += userFleets[i].fuelCapacity < warpCost ? userFleets[i].fuelCapacity < halfWarpCost ? subwarpCost : halfWarpCost : warpCost;
		} else fuelNeeded += subwarpCost;

		async function handleMineMovement() {
			if (userFleets[i].moveTarget && userFleets[i].moveTarget !== '') {
				let targetX = userFleets[i].moveTarget.split(',').length > 1 ? userFleets[i].moveTarget.split(',')[0].trim() : '';
				let targetY = userFleets[i].moveTarget.split(',').length > 1 ? userFleets[i].moveTarget.split(',')[1].trim() : '';
				let moveDist = calculateMovementDistance(fleetCoords, [targetX,targetY]);
				if (moveDist > 0) {
					let warpCooldownFinished = await handleMovement(i, moveDist, targetX, targetY, isStarbaseAndWarpSubwarp);
				} else {
					cLog(1,`${FleetTimeStamp(userFleets[i].label)} Idle 💤`);
					updateFleetState(userFleets[i], 'Idle');
				}
			} else {
				const msg = 'ERROR: Fleet must start at Target or Starbase';
				cLog(1,`${FleetTimeStamp(userFleets[i].label)} Mining - ${msg}`);
				updateFleetState(userFleets[i], msg);
			}
		}

		if(fleetState !== 'Idle') userFleets[i].justResupplied=false;

		//Not mining?
		if (fleetState === 'Idle') {
			let errorResource = [];
			let needSupplies = false;

			//Hard-coded 60 second duration check: no point resuming mining if it'll take less than 1 minute to finish
			if (miningDuration < 60) {
				cLog(1,`${FleetTimeStamp(userFleets[i].label)} Supplies low, only ${miningDuration} seconds left`);
				needSupplies = true;
			}
			else if (currentFuelCnt < fuelNeeded || currentAmmoCnt < ammoForDuration || currentFoodCnt < foodForDuration) {
				needSupplies = true;
			}

			//sometimes the RPC didn't catch up when we send the combined resupply tx - even after the loop wait time of 10 seconds. So we try to wait another 15+10 seconds if we think that's happening
			if(needSupplies && userFleets[i].justResupplied) {
				cLog(1,`${FleetTimeStamp(userFleets[i].label)} Fleet just resupplied, but it is trying to resupply again. RPC sync problems? We wait some time to be sure`);
				updateFleetState(userFleets[i], 'Waiting ...');
				await wait(15000);
				updateFleetState(userFleets[i], 'Idle');
				userFleets[i].justResupplied=false;
				return;
			}
			userFleets[i].justResupplied=false;

			let minerSupplySingleTx=globalSettings.minerSupplySingleTx;

			//Needs Resupply?
			if (needSupplies) {
				cLog(1,`${FleetTimeStamp(userFleets[i].label)} Need resupply`);

				//Recalulate requirements based on total cargo cap
				miningDuration = calculateMiningDuration(userFleets[i].cargoCapacity, userFleets[i].miningRate, resourceHardness, systemRichness);
				foodForDuration = Math.ceil(miningDuration * (userFleets[i].foodConsumptionRate / 10000));
				ammoForDuration = Math.ceil(miningDuration * (userFleets[i].ammoConsumptionRate / 10000));
				ammoForDuration = Math.min(userFleets[i].ammoCapacity, ammoForDuration);

				cLog(2, `${FleetTimeStamp(userFleets[i].label)} Calculated miningDuration: ${miningDuration}`);
				cLog(2, `${FleetTimeStamp(userFleets[i].label)} fuel: ${currentFuelCnt}/${fuelNeeded}`);
				cLog(2, `${FleetTimeStamp(userFleets[i].label)} ammo: ${currentAmmoCnt}/${ammoForDuration}`);
				//cLog(2, `${FleetTimeStamp(userFleets[i].label)} ammoForDuration: ${ammoForDuration} = miningDuration ${miningDuration} * (ammoConsumptionRate ${userFleets[i].ammoConsumptionRate} / 10000)`);
				cLog(2, `${FleetTimeStamp(userFleets[i].label)} food: ${currentFoodCnt}/${foodForDuration}`);

				if (fleetCoords[0] == starbaseX && fleetCoords[1] == starbaseY) {
					let transactions = [];
					let resp = await execDock(userFleets[i], userFleets[i].starbaseCoord, minerSupplySingleTx);
					if(minerSupplySingleTx && resp) {
						transactions.push(resp);
					}
					//await execDock(userFleets[i], userFleets[i].starbaseCoord);
					cLog(1,`${FleetTimeStamp(userFleets[i].label)} Unloading resource`);
					updateFleetState(userFleets[i], `Unloading`);

					let unloadAmount = 0;
					if(globalSettings.minerUnloadsAll) {
						for(let currentRes of fleetCurrentCargo.value) {

							// don't unload food
							if(currentRes.account.data.parsed.info.mint === sageGameAcct.account.mints.food.toString()) continue;

							let amountToUnload = currentRes.account.data.parsed.info.tokenAmount.uiAmount;
							if(globalSettings.minerKeep1 && amountToUnload > 0) { amountToUnload -= 1; }
							if (amountToUnload > 0) {
								resp = await execCargoFromFleetToStarbase(userFleets[i], userFleets[i].cargoHold, currentRes.account.data.parsed.info.mint, userFleets[i].starbaseCoord, amountToUnload, minerSupplySingleTx);
								if(minerSupplySingleTx && resp) {
									transactions.push(resp);
								}
								unloadAmount += amountToUnload;
							}
						}
					}
					else {
						unloadAmount = currentResourceCnt;
						if(globalSettings.minerKeep1 && unloadAmount > 0) { unloadAmount -= 1; }
						if (unloadAmount > 0) {
							resp = await execCargoFromFleetToStarbase(userFleets[i], userFleets[i].cargoHold, userFleets[i].mineResource, userFleets[i].starbaseCoord, unloadAmount, minerSupplySingleTx);
							if(minerSupplySingleTx && resp) {
								transactions.push(resp);
							}
							//await wait(2000);
						}
					}

					//if (currentFuelCnt < userFleets[i].fuelCapacity) {
					if (currentFuelCnt < fuelNeeded) {
						cLog(1,`${FleetTimeStamp(userFleets[i].label)} Loading fuel`);
						updateFleetState(userFleets[i], `Loading`);
						let fuelCargoTypeAcct = cargoTypes.find(item => item.account.mint.toString() == sageGameAcct.account.mints.fuel);
						let fuelResp = await execCargoFromStarbaseToFleet(userFleets[i], userFleets[i].fuelTank, fleetFuelAcct, sageGameAcct.account.mints.fuel.toString(), fuelCargoTypeAcct, userFleets[i].starbaseCoord, userFleets[i].fuelCapacity - currentFuelCnt, globalSettings.fleetForceConsumableAmount, minerSupplySingleTx);
						if (fuelResp && fuelResp.name == 'NotEnoughResource') {
							cLog(1,`${FleetTimeStamp(userFleets[i].label)} ERROR: Not enough fuel`);
							errorResource.push('fuel');
						}
						else if(minerSupplySingleTx && fuelResp.tx) {
							transactions.push(fuelResp.tx);
						}
						//await wait(2000);
					} else { cLog(1,`${FleetTimeStamp(userFleets[i].label)} Fuel loading skipped: ${currentFuelCnt} / ${fuelNeeded}`); }

					if (currentAmmoCnt < ammoForDuration) {
						cLog(1,`${FleetTimeStamp(userFleets[i].label)} Loading ammo`);
						updateFleetState(userFleets[i], `Loading`);
						let ammoCargoTypeAcct = cargoTypes.find(item => item.account.mint.toString() == sageGameAcct.account.mints.ammo);
						let ammoResp = await execCargoFromStarbaseToFleet(userFleets[i], userFleets[i].ammoBank, fleetAmmoAcct, sageGameAcct.account.mints.ammo.toString(), ammoCargoTypeAcct, userFleets[i].starbaseCoord, userFleets[i].ammoCapacity - currentAmmoCnt, globalSettings.fleetForceConsumableAmount, minerSupplySingleTx);
						if (ammoResp && ammoResp.name == 'NotEnoughResource') {
							cLog(1,`${FleetTimeStamp(userFleets[i].label)} ERROR: Not enough ammo`);
							errorResource.push('ammo');
						}
						else if(minerSupplySingleTx && ammoResp.tx) {
							transactions.push(ammoResp.tx);
						}
						//await wait(2000);
					} else { cLog(1,`${FleetTimeStamp(userFleets[i].label)} Ammo loading skipped: ${currentAmmoCnt} / ${ammoForDuration}`); }

					// removed the following block, because we already have all the data, "miningduration" only needs to take the unload amount into account
					miningDuration = calculateMiningDuration(userFleets[i].cargoCapacity - cargoCnt + unloadAmount + currentFoodCnt, userFleets[i].miningRate, resourceHardness, systemRichness);
					foodForDuration = Math.ceil(miningDuration * (userFleets[i].foodConsumptionRate / 10000)) + (globalSettings.minerKeep1 ? 1 : 0);
					if (currentFoodCnt < foodForDuration) {
						cLog(1,`${FleetTimeStamp(userFleets[i].label)} Loading food`);
						updateFleetState(userFleets[i], `Loading`);
						let foodCargoTypeAcct = cargoTypes.find(item => item.account.mint.toString() == sageGameAcct.account.mints.food);
						let foodResp = await execCargoFromStarbaseToFleet(userFleets[i], userFleets[i].cargoHold, fleetFoodAcct, sageGameAcct.account.mints.food.toString(), foodCargoTypeAcct, userFleets[i].starbaseCoord, foodForDuration - currentFoodCnt, globalSettings.fleetForceConsumableAmount, minerSupplySingleTx);
						if (foodResp && foodResp.name == 'NotEnoughResource') {
							cLog(1,`${FleetTimeStamp(userFleets[i].label)} ERROR: Not enough food`);
							errorResource.push('food');
						}
						else if(minerSupplySingleTx && foodResp.tx) {
							transactions.push(foodResp.tx);
						}
						//await wait(2000);
					} else { cLog(1,`${FleetTimeStamp(userFleets[i].label)} Food loading skipped: ${currentFoodCnt} / ${foodForDuration}`); }

					updateFleetState(userFleets[i], `Loading`);

					if (errorResource.length > 0) {
						updateFleetState(userFleets[i], `ERROR: Not enough ${errorResource.toString()}`);
						sisEvento('alerta', userFleets[i].label, 'Falta na starbase: ' + errorResource.toString());
					} else {
						//await execUndock(userFleets[i], userFleets[i].starbaseCoord);
						resp = await execUndock(userFleets[i], userFleets[i].starbaseCoord, minerSupplySingleTx);
						if(minerSupplySingleTx && resp) {
							updateFleetState(userFleets[i], `Executing resupply tx`);
							transactions.push(resp);
							//console.log(transactions);
							await txSliceAndSend(transactions, userFleets[i], 'RESUPPLY', 100, 6);
							updateFleetState(userFleets[i], 'Idle');
						}
						userFleets[i].justResupplied=true;
					}
					//await wait(2000);
					//userFleets[i].moveTarget = userFleets[i].destCoord;
				} else {
					if(userFleets[i].stopping) return;
					userFleets[i].moveTarget = userFleets[i].starbaseCoord;
					await handleMineMovement();
				}
			}

			//At mining area?
			else if (fleetCoords[0] == destX && fleetCoords[1] == destY) {
		if(userFleets[i].stopping) return;
                fleetCurrentCargo = await solanaReadConnection.getParsedTokenAccountsByOwner(userFleets[i].cargoHold, {programId: tokenProgramPK});
                cargoCnt = fleetCurrentCargo.value.reduce((n, {account}) => n + account.data.parsed.info.tokenAmount.uiAmount, 0);
                currentFood = fleetCurrentCargo.value.find(item => item.account.data.parsed.info.mint === sageGameAcct.account.mints.food.toString());
                fleetFoodAcct = currentFood ? currentFood.pubkey : fleetFoodToken;
                currentFoodCnt = currentFood ? currentFood.account.data.parsed.info.tokenAmount.uiAmount : 0;
                miningDuration = calculateMiningDuration(userFleets[i].cargoCapacity - cargoCnt + currentFoodCnt, userFleets[i].miningRate, resourceHardness, systemRichness);
				await execStartMining(userFleets[i], mineItem, sageResource, planet);
				if (userFleets[i].state.slice(0, 5) !== 'ERROR') updateFleetState(userFleets[i], 'Mine [' + TimeToStr(new Date(Date.now()+(miningDuration * 1000))) + ']')

				//Wait for data to propagate through the RPCs
				await wait(20000);

				//Fetch update mining state from chain
				const fleetAcctInfo = await getAccountInfo(userFleets[i].label, 'full fleet info', userFleets[i].publicKey);
				const [fleetState, extra] = getFleetState(fleetAcctInfo, userFleets[i]);
				cLog(4, `${FleetTimeStamp(userFleets[i].label)} chain fleet state: ${fleetState}`);
				fleetMining = fleetState == 'MineAsteroid' ? extra : null;
			}

			//Move to mining area
			else {
				if(userFleets[i].stopping) return;
				userFleets[i].moveTarget = userFleets[i].destCoord;
				await handleMineMovement();
			}
		}

		if(userFleets[i].stopping) return;

		//Already mining?
		if (userFleets[i].state.slice(0, 4) === 'Mine' && fleetMining) {
            let maxMiningDuration = calculateMiningDuration(userFleets[i].cargoCapacity, userFleets[i].miningRate, resourceHardness, systemRichness);
            let mineTimePassed = (Date.now() / 1000) - fleetMining.start.toNumber();
            let foodConsumed = Math.ceil(mineTimePassed * (userFleets[i].foodConsumptionRate / 10000));
            let ammoConsumed = Math.ceil(mineTimePassed * (userFleets[i].ammoConsumptionRate / 10000));
            let resourceMined = Math.ceil(mineTimePassed * ((userFleets[i].miningRate/10000) * (systemRichness/100)) / (resourceHardness/100));
            let timeFoodRemaining = Math.ceil((currentFoodCnt - foodConsumed) / (userFleets[i].foodConsumptionRate / 10000));
            let timeAmmoRemaining = userFleets[i].ammoConsumptionRate > 0 ? Math.ceil((currentAmmoCnt - ammoConsumed) / (userFleets[i].ammoConsumptionRate / 10000)) : maxMiningDuration;

            //wrong calculation, fixed: we must not subtract the foodConsumed from the simulated cargo space, because the food doesn't exist anymore
            //we instead ignore the food needed for consumption completely

            //let simCurrentCargo = userFleets[i].cargoCapacity - cargoCnt - resourceMined + currentFoodCnt - foodConsumed;
            let foodMaxConsumed = Math.ceil(maxMiningDuration * (userFleets[i].foodConsumptionRate / 10000));
            let simCurrentCargo = userFleets[i].cargoCapacity - cargoCnt - resourceMined + foodMaxConsumed;

            let timeCargoRemaining = calculateMiningDuration(simCurrentCargo, userFleets[i].miningRate, resourceHardness, systemRichness);
            let timeLimit = Math.min(timeFoodRemaining, timeAmmoRemaining, timeCargoRemaining);
            let mineEnd = Date.now()+(timeLimit * 1000);
			userFleets[i].mineEnd = mineEnd;
			updateFleetState(userFleets[i], 'Mine [' + TimeToStr(new Date(mineEnd)) + ']')
			let sageResourceAcctInfo = await sageProgram.account.resource.fetch(fleetMining.resource);
			let mineItem = await sageProgram.account.mineItem.fetch(sageResourceAcctInfo.mineItem);
			if (Date.now() > (mineEnd)) {
				await execStopMining(userFleets[i], fleetMining.resource, sageResourceAcctInfo, sageResourceAcctInfo.mineItem, mineItem.mint);
            }
		}
	}

	function hasTransportManifest(manifest) {
		for (const entry of manifest)
			if(entry.res && entry.amt) return true;

		return false;
	}

    async function checkCargo(currentManifest, destinationManifest, fleet) {
        const fleetCurrentCargo = await solanaReadConnection.getParsedTokenAccountsByOwner(fleet.cargoHold, {programId: tokenProgramPK});
        const cargoCnt = fleetCurrentCargo.value.reduce((n, {account}) => n + account.data.parsed.info.tokenAmount.uiAmount * cargoItems.find(r => r.token == account.data.parsed.info.mint).size, 0);
        let needToLoad = false;
        let needToUnload = false;
        for (const entry of destinationManifest) {
            if (entry.res && entry.amt > 0) {
                let currentCargoObj = fleetCurrentCargo.value.find(item => item.account.data.parsed.info.mint === entry.res);
                let currentCargoResAmt = currentCargoObj ? currentCargoObj.account.data.parsed.info.tokenAmount.uiAmount : 0;
                if (currentCargoResAmt < entry.amt) needToLoad = true;
                if (currentCargoResAmt > entry.amt) {
                    needToUnload = true;
                    currentManifest.push({res: entry.res, amt: currentCargoResAmt - entry.amt, extra: true});
                }
            }
        }

        for (const entry of currentManifest) {
            if (entry.res && entry.amt > 0) {
                let currentCargoObj = fleetCurrentCargo.value.find(item => item.account.data.parsed.info.mint === entry.res);
                let currentCargoResAmt = currentCargoObj ? currentCargoObj.account.data.parsed.info.tokenAmount.uiAmount : 0;
                if (currentCargoResAmt > (globalSettings.transportKeep1 ? 1 : 0)) needToUnload = true;
            }
        }

	//we check if the fleet has any cargo which isn't part of the current or destination manifest - to unload it
	if(globalSettings.transportUnloadsUnknownRSS) {
		for(let currentRes of fleetCurrentCargo.value) {
			if(currentManifest.findIndex(item => item.res == currentRes.account.data.parsed.info.mint) < 0 && destinationManifest.findIndex(item => item.res == currentRes.account.data.parsed.info.mint) < 0) {
				let amountToUnload = currentRes.account.data.parsed.info.tokenAmount.uiAmount;
				if(globalSettings.transportKeep1 && amountToUnload > 0) { amountToUnload -= 1; }
				if(amountToUnload > 0) {
					needToUnload = true;
					cLog(3,`${FleetTimeStamp(fleet.label)} Found rss that is not part of any transport manifest, unloading`, amountToUnload, 'of', currentRes.account.data.parsed.info.mint);
					currentManifest.push({res: currentRes.account.data.parsed.info.mint, amt: amountToUnload });
				}
			}
		}
	}

        return {currentManifest, destinationManifest, needToLoad, needToUnload};
    }

	async function handleCrewUnloading(fleet, starbaseCoords, amount, returnTx) {
	return new Promise(async resolve => {

	    let starbaseX = starbaseCoords.split(',')[0].trim();
	    let starbaseY = starbaseCoords.split(',')[1].trim();
	    let starbase = await getStarbaseFromCoords(starbaseX, starbaseY);
	    let starbasePlayer = await getStarbasePlayer(userProfileAcct,starbase.publicKey);
	    starbasePlayer = starbasePlayer ? starbasePlayer.publicKey : await execRegisterStarbasePlayer(fleet, starbaseCoords);

			let txUnload = { instruction: await sageProgram.methods.unloadFleetCrew({count: new BrowserAnchor.anchor.BN(amount), keyIndex: new BrowserAnchor.anchor.BN(userProfileKeyIdx) }).accountsStrict({
				fleetAndOwner: {
						key: userPublicKey,
						owningProfile: userProfileAcct,
						owningProfileFaction: userProfileFactionAcct.publicKey,
						fleet: fleet.publicKey
				},
				starbaseAndStarbasePlayer: {
					starbase: starbase.publicKey,
					starbasePlayer: starbasePlayer
				},
				gameId: sageGameAcct.publicKey
			}).remainingAccounts([{
					pubkey: starbase.publicKey,
					isSigner: false,
					isWritable: false
			}]).instruction()}
			cLog(1,`${FleetTimeStamp(fleet.label)} Unloading crew`);
			updateFleetState(fleet, 'Unloading crew');
			//let txResult = await txSignAndSend(txUnload, fleet, 'UNLOAD CREW', 100);

			let txResult;
			if(returnTx) {
				txResult = txUnload;
			} else {
				txResult = await txSignAndSend(txUnload, fleet, 'UNLOAD CREW', 100);
			}

			resolve(txResult);
		});
	}
	async function handleCrewLoading(fleet, starbaseCoords, amount, returnTx) {
		return new Promise(async resolve => {

		    let starbaseX = starbaseCoords.split(',')[0].trim();
		    let starbaseY = starbaseCoords.split(',')[1].trim();
		    let starbase = await getStarbaseFromCoords(starbaseX, starbaseY);
		    let starbasePlayer = await getStarbasePlayer(userProfileAcct,starbase.publicKey);
		    starbasePlayer = starbasePlayer ? starbasePlayer.publicKey : await execRegisterStarbasePlayer(fleet, starbaseCoords);

		    let starbasePlayerInfo = await sageProgram.account.starbasePlayer.fetch(starbasePlayer);
		    let availableCrew = starbasePlayerInfo.totalCrew - starbasePlayerInfo.busyCrew.toNumber();
		    cLog(3,`${FleetTimeStamp(fleet.label)} Available crew at starbase:`, availableCrew);
		    if(availableCrew < amount) amount = availableCrew;

		    let txResult;
		    if(amount <= 0) {
				txResult = {name: "NotEnoughCrew"};
		    } else {

			    let txLoad = { instruction: await sageProgram.methods.loadFleetCrew({count: new BrowserAnchor.anchor.BN(amount), keyIndex: new BrowserAnchor.anchor.BN(userProfileKeyIdx) }).accountsStrict({
					fleetAndOwner: {
							key: userPublicKey,
							owningProfile: userProfileAcct,
							owningProfileFaction: userProfileFactionAcct.publicKey,
							fleet: fleet.publicKey
					},
					starbaseAndStarbasePlayer: {
						starbase: starbase.publicKey,
						starbasePlayer: starbasePlayer
					},
					gameId: sageGameAcct.publicKey
			    }).remainingAccounts([{
						pubkey: starbase.publicKey,
						isSigner: false,
						isWritable: false
			    }]).instruction()}
			    cLog(1,`${FleetTimeStamp(fleet.label)} Loading crew`);
			    updateFleetState(fleet, 'Loading crew');
			    //txResult = await txSignAndSend(txLoad, fleet, 'LOAD CREW', 100);
			    if(returnTx) {
			    	txResult = txLoad;
			    } else {
				txResult = await txSignAndSend(txLoad, fleet, 'LOAD CREW', 100);
			    }
		    }

		    resolve(txResult);
		});
	}

    async function handleTransport(i, fleetState, fleetCoords) {
        const [destX, destY] = ConvertCoords(userFleets[i].destCoord);
        const [starbaseX, starbaseY] = ConvertCoords(userFleets[i].starbaseCoord);

        const fleetParsedData = JSON.parse(await GM.getValue(userFleets[i].publicKey.toString(), '{}'));
        let targetCargoManifest = [
            {res: fleetParsedData.transportResource1, amt: fleetParsedData.transportResource1Perc, crew: fleetParsedData.transportResource1Crew },
            {res: fleetParsedData.transportResource2, amt: fleetParsedData.transportResource2Perc,},
            {res: fleetParsedData.transportResource3, amt: fleetParsedData.transportResource3Perc,},
            {res: fleetParsedData.transportResource4, amt: fleetParsedData.transportResource4Perc,},
        ];
        let starbaseCargoManifest = [
            {res: fleetParsedData.transportSBResource1, amt: fleetParsedData.transportSBResource1Perc, crew: fleetParsedData.transportSBResource1Crew},
            {res: fleetParsedData.transportSBResource2, amt: fleetParsedData.transportSBResource2Perc,},
            {res: fleetParsedData.transportSBResource3, amt: fleetParsedData.transportSBResource3Perc,},
            {res: fleetParsedData.transportSBResource4, amt: fleetParsedData.transportSBResource4Perc,},
        ];
        const hasTargetManifest = hasTransportManifest(targetCargoManifest);
        const hasStarbaseManifest = hasTransportManifest(starbaseCargoManifest);

        //let moveDist = calculateMovementDistance([starbaseX,starbaseY], [destX,destY]);
        if (fleetState === 'Idle') {
            // Fleet at starbase?
            if (fleetCoords[0] == starbaseX && fleetCoords[1] == starbaseY) {
                userFleets[i].resupplying = true;

                let checkCargoResult = await checkCargo(starbaseCargoManifest, targetCargoManifest, userFleets[i]);
                starbaseCargoManifest = checkCargoResult.currentManifest;
                targetCargoManifest = checkCargoResult.destinationManifest;

		let needToUnloadCrew = 0;
		if((starbaseCargoManifest[0].crew > 0) && (userFleets[i].passengerCapacity > 0) && (userFleets[i].crewCount - userFleets[i].requiredCrew > 0)) {
			needToUnloadCrew = userFleets[i].crewCount - userFleets[i].requiredCrew;
		}
		let needToLoadCrew = 0;
		if((targetCargoManifest[0].crew > 0) && (userFleets[i].passengerCapacity > 0) && ((userFleets[i].requiredCrew + userFleets[i].passengerCapacity - userFleets[i].crewCount - needToUnloadCrew) > 0)) {
			needToLoadCrew = Math.min(userFleets[i].requiredCrew + userFleets[i].passengerCapacity - userFleets[i].crewCount, targetCargoManifest[0].crew);
		}
		if(starbaseCargoManifest[0].crew > 0 || targetCargoManifest[0].crew > 0) cLog(3, `${FleetTimeStamp(userFleets[i].label)} crew:`, userFleets[i].crewCount, 'passengerCapacity:', userFleets[i].passengerCapacity, 'required crew:', userFleets[i].requiredCrew, 'load:', needToLoadCrew, 'unload:', needToUnloadCrew);

                const fuelData = await getFleetFuelData(userFleets[i], [starbaseX, starbaseY], [destX, destY], true);
		//previously we only checked for "fuelData.fuelNeeded > 0" below, but fuelNeeded is always greater than 0 - it is just the fuel needed for the warp/subwarp.
		//this broke the check if the fleet needs to do the dock/undock sequence and the sequence was always executed
		//We need to explicitly calculate the needed fuel minus the available fuel, just like in handleTransportRefueling()
		const fuelEntry = targetCargoManifest.find(e => e.res === sageGameAcct.account.mints.fuel.toString()) || {amt: 0};
		const totalFuel = fuelData.fuelNeeded + fuelEntry.amt;
		let fuelToAdd = Math.min(fuelData.capacity, totalFuel) - fuelData.amount;
		// Check for "Fuel to 100% for transporters" (roundTrip only = source starbase)
		if(fuelToAdd > 0 && globalSettings.transportFuel100 && fuelToAdd < fuelData.capacity - fuelData.amount) fuelToAdd = fuelData.capacity - fuelData.amount;

		cLog(3,`${FleetTimeStamp(userFleets[i].label)} Fuel needed`, fuelData.fuelNeeded, '/ fuel found', fuelData.amount, '/ fuel to add', fuelToAdd, '/ needToLoad', checkCargoResult.needToLoad, '/ needToUnload', checkCargoResult.needToUnload, '/ needToLoadCrew', needToLoadCrew, '/ needToUnloadCrew', needToUnloadCrew);

                //if (checkCargoResult.needToLoad || checkCargoResult.needToUnload || fuelData.fuelNeeded > 0 || needToLoadCrew || needToUnloadCrew) {
                if (checkCargoResult.needToLoad || checkCargoResult.needToUnload || fuelToAdd > 0 || needToLoadCrew > 0 || needToUnloadCrew > 0) {
                    //await execDock(userFleets[i], userFleets[i].starbaseCoord);
                    let transportLoadUnloadSingleTx=globalSettings.transportLoadUnloadSingleTx;
                    let transactions = [];
                    let unloadedAmountInTransaction = 0;

                    let resp = await execDock(userFleets[i], userFleets[i].starbaseCoord, transportLoadUnloadSingleTx);
                    if(transportLoadUnloadSingleTx && resp) {
			transactions.push(resp);
                    }

                    if(needToUnloadCrew) {
			resp = await handleCrewUnloading(userFleets[i], userFleets[i].starbaseCoord, needToUnloadCrew, transportLoadUnloadSingleTx);
			if(transportLoadUnloadSingleTx && resp) {
				transactions.push(resp);
			}
                    }
                    if(needToLoadCrew) {
			let crewResp = await handleCrewLoading(userFleets[i], userFleets[i].starbaseCoord, needToLoadCrew, transportLoadUnloadSingleTx);
			if (crewResp && crewResp.name == 'NotEnoughCrew') {
				if(globalSettings.transportStopOnError) {
					cLog(1,`${FleetTimeStamp(userFleets[i].label)} Transporting - ERROR: Not enough crew`);
					updateFleetState(userFleets[i], 'ERROR: Not enough crew');
					return;
				} else {
					cLog(1,`${FleetTimeStamp(userFleets[i].label)} Not enough crew`);
				}
			}
			else if(transportLoadUnloadSingleTx && crewResp) {
				transactions.push(crewResp);
			}
                    }

                    if (hasStarbaseManifest || checkCargoResult.needToUnload) {
                        resp = await handleTransportUnloading(userFleets[i], userFleets[i].starbaseCoord, starbaseCargoManifest, transportLoadUnloadSingleTx);
                        if(transportLoadUnloadSingleTx) {
				transactions = transactions.concat(resp.transactions);
				unloadedAmountInTransaction = resp.unloadedAmount;
			}
                    } else cLog(1,`${FleetTimeStamp(userFleets[i].label)} Unloading skipped - No resources specified`);

                    //Refueling at Starbase
                    let refuelResp = await handleTransportRefueling(userFleets[i], userFleets[i].starbaseCoord, [starbaseX, starbaseY], [destX, destY], true, 0, targetCargoManifest, transportLoadUnloadSingleTx);
                    if (refuelResp.status === 0) {
                        userFleets[i].state = refuelResp.detail;
                        return;
                    } else if(transportLoadUnloadSingleTx && refuelResp && refuelResp.transactions) {
			transactions = transactions.concat(refuelResp.transactions);
                    }

                    let fuelIndex = targetCargoManifest.findIndex(e => e.res === sageGameAcct.account.mints.fuel.toString());
                    if (fuelIndex > -1) {
                        targetCargoManifest[fuelIndex].amt = targetCargoManifest[fuelIndex].amt - refuelResp.amount;
                        //when using a combined load tx, we need to take into account the amount loaded into the fuel tank, because the starbase still reports the original amount (otherwise we may get an ix error instead a "NotEnoughResource" error)
                        if(transportLoadUnloadSingleTx && refuelResp.alreadyLoaded) targetCargoManifest[fuelIndex].alreadyLoadedInTransaction = refuelResp.alreadyLoaded;
                    }

                    //Loading at Starbase
                    if (hasTargetManifest) {
                        const loadedCargo = await handleTransportLoading(i, userFleets[i].starbaseCoord, targetCargoManifest, transportLoadUnloadSingleTx, transportLoadUnloadSingleTx ? unloadedAmountInTransaction : 0);
                        cLog(4,`${FleetTimeStamp(userFleets[i].label)} loadedCargo: `, loadedCargo.success);
                        if(!loadedCargo.success && globalSettings.transportStopOnError) {
                            //const newFleetState = `ERROR: No more cargo to load`;
                            //cLog(1,`${FleetTimeStamp(userFleets[i].label)} ${newFleetState}`);
                            //userFleets[i].state = newFleetState;
                            cLog(1,`${FleetTimeStamp(userFleets[i].label)} ERROR: Unexpected error on cargo load.`);
                            return;
                        } else if(transportLoadUnloadSingleTx) {
                            transactions = transactions.concat(loadedCargo.transactions);
			}
                    } else cLog(1,`${FleetTimeStamp(userFleets[i].label)} Loading skipped - No resources specified`);

                    let undockResult = await execUndock(userFleets[i], userFleets[i].starbaseCoord, transportLoadUnloadSingleTx);
                    if(transportLoadUnloadSingleTx) {
			updateFleetState(userFleets[i], 'Exec tx bundle');
			transactions.push(undockResult);
			undockResult = await txSliceAndSend(transactions, userFleets[i], 'LOAD/UNLOAD', 100, 5);
			updateFleetState(userFleets[i], 'Idle');
                    }
                    //cLog(4,`${FleetTimeStamp(userFleets[i].label)} userFleets[i]: `, undockResult);
                    let fleetState = await solanaReadConnection.getAccountInfoAndContext(userFleets[i].publicKey, {minContextSlot: undockResult.slot});
                }
                userFleets[i].moveTarget = userFleets[i].destCoord;
                userFleets[i].resupplying = false;
                cLog(3,`${FleetTimeStamp(userFleets[i].label)} userFleets[i]: `, userFleets[i]);
            }

            // Fleet at target?
            else if (fleetCoords[0] == destX && fleetCoords[1] == destY) {
                userFleets[i].resupplying = true;

                let checkCargoResult = await checkCargo(targetCargoManifest, starbaseCargoManifest, userFleets[i]);
                targetCargoManifest = checkCargoResult.currentManifest;
                starbaseCargoManifest = checkCargoResult.destinationManifest;

		let needToUnloadCrew = 0;
		if((targetCargoManifest[0].crew > 0) && (userFleets[i].passengerCapacity > 0) && (userFleets[i].crewCount - userFleets[i].requiredCrew > 0)) {
			needToUnloadCrew = userFleets[i].crewCount - userFleets[i].requiredCrew;
		}
		let needToLoadCrew = 0;
		if((starbaseCargoManifest[0].crew > 0) && (userFleets[i].passengerCapacity > 0) && ((userFleets[i].requiredCrew + userFleets[i].passengerCapacity - userFleets[i].crewCount - needToUnloadCrew) > 0)) {
			needToLoadCrew = Math.min(userFleets[i].requiredCrew + userFleets[i].passengerCapacity - userFleets[i].crewCount, starbaseCargoManifest[0].crew);
		}
		if(starbaseCargoManifest[0].crew > 0 || targetCargoManifest[0].crew > 0) cLog(3, `${FleetTimeStamp(userFleets[i].label)} crew:`, userFleets[i].crewCount, 'passengerCapacity:', userFleets[i].passengerCapacity, 'required crew:', userFleets[i].requiredCrew, 'load:', needToLoadCrew, 'unload:', needToUnloadCrew);

                const fuelData = await getFleetFuelData(userFleets[i], [destX, destY], [starbaseX, starbaseY], false);
		//previously we only checked for "fuelData.fuelNeeded > 0" below, but fuelNeeded is always greater than 0 - it is just the fuel needed for the warp/subwarp.
		//We need to explicitly calculate the needed fuel minus the available fuel, just like in handleTransportRefueling()
		const fuelEntry = starbaseCargoManifest.find(e => e.res === sageGameAcct.account.mints.fuel.toString()) || {amt: 0};
		const totalFuel = fuelData.fuelNeeded + fuelEntry.amt;
		let fuelToAdd = Math.min(fuelData.capacity, totalFuel) - fuelData.amount;

		cLog(3,`${FleetTimeStamp(userFleets[i].label)} Fuel needed`, fuelData.fuelNeeded, '/ fuel found', fuelData.amount, '/ fuel to add', fuelToAdd, '/ needToLoad', checkCargoResult.needToLoad, '/ needToUnload', checkCargoResult.needToUnload, '/ needToLoadCrew', needToLoadCrew, '/ needToUnloadCrew', needToUnloadCrew);

                if (checkCargoResult.needToLoad || checkCargoResult.needToUnload || fuelToAdd > 0 || needToLoadCrew > 0 || needToUnloadCrew > 0) {
                    //await execDock(userFleets[i], userFleets[i].destCoord);
                    let transportLoadUnloadSingleTx=globalSettings.transportLoadUnloadSingleTx;
                    let transactions = [];
                    let unloadedAmountInTransaction = 0;

                    let resp = await execDock(userFleets[i], userFleets[i].destCoord, transportLoadUnloadSingleTx);
                    if(transportLoadUnloadSingleTx && resp) {
			transactions.push(resp);
                    }

                    if(needToUnloadCrew) {
			resp = await handleCrewUnloading(userFleets[i], userFleets[i].destCoord, needToUnloadCrew, transportLoadUnloadSingleTx);
			if(transportLoadUnloadSingleTx && resp) {
				transactions.push(resp);
			}
                    }
                    if(needToLoadCrew) {
			let crewResp = await handleCrewLoading(userFleets[i], userFleets[i].destCoord, needToLoadCrew, transportLoadUnloadSingleTx);
			if (crewResp && crewResp.name == 'NotEnoughCrew') {
				if(globalSettings.transportStopOnError) {
					cLog(1,`${FleetTimeStamp(userFleets[i].label)} Transporting - ERROR: Not enough crew`);
					updateFleetState(userFleets[i], 'ERROR: Not enough crew');
					return;
				} else {
					cLog(1,`${FleetTimeStamp(userFleets[i].label)} Not enough crew`);
				}
			} else if(transportLoadUnloadSingleTx && crewResp) {
				transactions.push(crewResp);
			}
                    }

                    //Unloading at Target
                    let fuelUnloadDeficit = 0; //How far short of the manifest was the amount of fuel unloaded?
                    if (hasTargetManifest || checkCargoResult.needToUnload) {
                        const unloadResult = await handleTransportUnloading(userFleets[i], userFleets[i].destCoord, targetCargoManifest, transportLoadUnloadSingleTx);
                        fuelUnloadDeficit = unloadResult.fuelUnloadDeficit;
                        if(transportLoadUnloadSingleTx) {
				transactions = transactions.concat(unloadResult.transactions);
				unloadedAmountInTransaction = unloadResult.unloadedAmount;
			}
                    } else cLog(1,`${FleetTimeStamp(userFleets[i].label)} Unloading skipped - No resources specified`);

                    //Refueling at Target
                    let refuelResp = await handleTransportRefueling(userFleets[i], userFleets[i].destCoord, [destX, destY], [starbaseX, starbaseY], false, fuelUnloadDeficit, starbaseCargoManifest, transportLoadUnloadSingleTx);
                    if (refuelResp.status === 0) {
                        userFleets[i].state = refuelResp.detail;
                        return;
                    } else if(transportLoadUnloadSingleTx && refuelResp && refuelResp.transactions) {
			transactions = transactions.concat(refuelResp.transactions);
                    }

                    let fuelIndex = starbaseCargoManifest.findIndex(e => e.res === sageGameAcct.account.mints.fuel.toString());
                    if (fuelIndex > -1) {
                        starbaseCargoManifest[fuelIndex].amt = starbaseCargoManifest[fuelIndex].amt - refuelResp.amount;
                    }

                    //Loading at Target
                    if(hasStarbaseManifest) {
                        const loadedCargo = await handleTransportLoading(i, userFleets[i].destCoord, starbaseCargoManifest, transportLoadUnloadSingleTx, transportLoadUnloadSingleTx ? unloadedAmountInTransaction : 0);
                        cLog(4,`${FleetTimeStamp(userFleets[i].label)} loadedCargo: `, loadedCargo.success);
                        if(!loadedCargo.success && globalSettings.transportStopOnError) {
                            //const newFleetState = `ERROR: No more cargo to load`;
                            //cLog(1,`${FleetTimeStamp(userFleets[i].label)} ${newFleetState}`);
                            //userFleets[i].state = newFleetState;
                            cLog(1,`${FleetTimeStamp(userFleets[i].label)} ERROR: Unexpected error on cargo load.`);
                            return;
                        } else if(transportLoadUnloadSingleTx) {
				transactions = transactions.concat(loadedCargo.transactions);
			}
                    } else cLog(1,`${FleetTimeStamp(userFleets[i].label)} Loading skipped - No resources specified`);

                    let undockResult = await execUndock(userFleets[i], userFleets[i].destCoord, transportLoadUnloadSingleTx);
                    if(transportLoadUnloadSingleTx) {
			transactions.push(undockResult);
			updateFleetState(userFleets[i], 'Exec tx bundle');
			undockResult = await txSliceAndSend(transactions, userFleets[i], 'LOAD/UNLOAD', 100, 5);
			updateFleetState(userFleets[i], 'Idle');
                    }
                    //cLog(4,`${FleetTimeStamp(userFleets[i].label)} userFleets[i]: `, undockResult);
                    let fleetState = await solanaReadConnection.getAccountInfoAndContext(userFleets[i].publicKey, {minContextSlot: undockResult.slot});
                }
                userFleets[i].moveTarget = userFleets[i].starbaseCoord;
                userFleets[i].resupplying = false;
                cLog(3,`${FleetTimeStamp(userFleets[i].label)} userFleets[i]: `, userFleets[i]);
            }

            if(userFleets[i].stopping) return;

            if (userFleets[i].moveTarget !== '') {
                const targetX = userFleets[i].moveTarget.split(',').length > 1 ? userFleets[i].moveTarget.split(',')[0].trim() : '';
                const targetY = userFleets[i].moveTarget.split(',').length > 1 ? userFleets[i].moveTarget.split(',')[1].trim() : '';
                const moveDist = calculateMovementDistance(fleetCoords, [targetX,targetY]);
		let isStarbaseAndWarpSubwarp = userFleets[i].moveType == 'warpsubwarp' && ((fleetCoords[0] == starbaseX && fleetCoords[1] == starbaseY) || (fleetCoords[0] == destX && fleetCoords[1] == destY));
                await handleMovement(i, moveDist, targetX, targetY, isStarbaseAndWarpSubwarp);
            } else {
                cLog(1,`${FleetTimeStamp(userFleets[i].label)} Transporting - ERROR: Fleet must start at Target or Starbase`);
                updateFleetState(userFleets[i], 'ERROR: Fleet must start at Target or Starbase');
            }
        }
    }

	async function getFleetFuelData(fleet, currentPos, targetPos, roundTrip = true) {
		const moveDist = calculateMovementDistance(currentPos, targetPos);
		const fleetCurrentFuelTank = await solanaReadConnection.getParsedTokenAccountsByOwner(fleet.fuelTank, {programId: tokenProgramPK});
		const token = fleetCurrentFuelTank.value.find(item => item.account.data.parsed.info.mint === sageGameAcct.account.mints.fuel.toString());
		const account = token ? token.pubkey : await getFleetFuelToken(fleet);
		let amount = token ? token.account.data.parsed.info.tokenAmount.uiAmount - fleet.exitSubwarpWillBurnFuel : 0;
		// [1.5.7] reabastecer do Sis-Bot: não passa da menor leitura conferida antes (sisTetoComb)
		if (Number.isFinite(fleet.sisTetoComb) && fleet.sisTetoComb < amount) amount = fleet.sisTetoComb;
        const warpCost = calcWarpFuelReq(fleet, currentPos, targetPos, fleet.moveType == 'warpsubwarp');
        const subwarpCost = Math.ceil(calculateSubwarpFuelBurn(fleet, moveDist));

        //Calculate fuel needed
        const costMultiplier = roundTrip ? 2 : 1;
        let fuelNeeded = 0;
        if (fleet.moveType == 'warp' || fleet.moveType == 'warpsubwarp' || fleet.moveType == 'warp-subwarp-warp') {
            fuelNeeded = warpCost * costMultiplier;
            if (fuelNeeded > fleet.fuelCapacity) fuelNeeded = roundTrip ? warpCost + subwarpCost : subwarpCost;
        } else {
            fuelNeeded = subwarpCost * costMultiplier;
        }

		//cLog(4, `${FleetTimeStamp(fleet.label)} getFleetFuelData -> calcWarpFuelReq:`, currentPos, targetPos);
		return {
			account,
			token,
			amount,
			capacity: fleet.fuelCapacity,
			warpCost: warpCost,
			subwarpCost: subwarpCost,
            fuelNeeded: fuelNeeded
		}
	}

	async function fuelFleet(fleet, dockCoords, account, amount, returnTx) {
		cLog(1,`${FleetTimeStamp(fleet.label)} Filling fuel tank: ${amount}`);
		let fuelCargoTypeAcct = cargoTypes.find(item => item.account.mint.toString() == sageGameAcct.account.mints.fuel);
		const fuelResp = await execCargoFromStarbaseToFleet(
			fleet,
			fleet.fuelTank,
			account,
			sageGameAcct.account.mints.fuel.toString(),
			fuelCargoTypeAcct,
			dockCoords,
			amount,
			globalSettings.fleetForceConsumableAmount,
			returnTx
		);

		return fuelResp;
	}

	async function handleTransportRefueling(fleet, starbaseCoord, currentPos, targetPos, roundTrip = true, amountToDropOff = 0, transportManifest, returnTx) {
		cLog(1,`${FleetTimeStamp(fleet.label)} ⛽ Refueling`);
		updateFleetState(fleet, 'Refueling');
        let fuelResp = {status: 0, detail: '', amount: 0};

		const fuelData = await getFleetFuelData(fleet, currentPos, targetPos, roundTrip);


		if(fuelData.fuelNeeded > fuelData.capacity) {
			cLog(1,`${FleetTimeStamp(fleet.label)} ERROR: Fuel tank too small for round trip`);
            fuelResp.detail = 'ERROR: Fuel tank too small for round trip';
			return fuelResp;
		}

        const fuelEntry = transportManifest.find(e => e.res === sageGameAcct.account.mints.fuel.toString()) || {amt: 0};

		//Log fuel readouts
		const extraFuel = Math.floor(fuelData.amount - fuelData.fuelNeeded);
		cLog(2, `${FleetTimeStamp(fleet.label)} Current Fuel: ${fuelData.amount}`);
		cLog(2, `${FleetTimeStamp(fleet.label)} Warp Cost: ${fuelData.warpCost}`);
		cLog(2, `${FleetTimeStamp(fleet.label)} Subwarp Cost: ${fuelData.subwarpCost}`);
        cLog(2, `${FleetTimeStamp(fleet.label)} Fuel To Transport: ${fuelEntry.amt}`);
		cLog(2, `${FleetTimeStamp(fleet.label)} Extra Fuel: ${extraFuel}`);

		let transactions = [];
		let alreadyLoaded=0;

		//Unload extra fuel from tank
		if(amountToDropOff > 0) {
			const fuelToUnload = Math.min(amountToDropOff, extraFuel);
			if (fuelToUnload > 0) {
				cLog(1,`${FleetTimeStamp(fleet.label)} Unloading extra fuel: ${fuelToUnload}`);
				let resp = await execCargoFromFleetToStarbase(fleet, fleet.fuelTank, sageGameAcct.account.mints.fuel.toString(), starbaseCoord, fuelToUnload, returnTx);
				alreadyLoaded -= fuelToUnload;
				if(returnTx && resp) {
					transactions.push(resp);
				}
			}
		}

		//Calculate amount of fuel to add to the tank
        const totalFuel = fuelData.fuelNeeded + fuelEntry.amt;
		let fuelToAdd = Math.min(fuelData.capacity, totalFuel) - fuelData.amount;
		fuelResp.alreadyLoaded = alreadyLoaded;

		//Bail if already has enough
		if (fuelToAdd <= 0) {
            fuelResp.status = 1;
            fuelResp.amount = fuelData.amount + fuelToAdd - fuelData.fuelNeeded;
            if(transactions.length > 0) fuelResp.transactions = transactions;
            return fuelResp;
        }

        if(globalSettings.transportFuel100 && roundTrip && fuelToAdd < fuelData.capacity - fuelData.amount) {
            fuelToAdd = fuelData.capacity - fuelData.amount;
        }

		//Put in the fuel
        let execResp = await fuelFleet(fleet, starbaseCoord, fuelData.account, fuelToAdd, returnTx);

        if (execResp && execResp.name == 'NotEnoughResource') {
			cLog(1,`${FleetTimeStamp(fleet.label)} ERROR: Not enough fuel`);
			sisEvento('alerta', fleet.label, 'Combustível insuficiente na starbase');
            fuelResp.detail = 'ERROR: Not enough fuel';
		} else {
            fuelResp.status = 1;
            fuelResp.amount = fuelData.amount + fuelToAdd - fuelData.fuelNeeded;
            alreadyLoaded += fuelToAdd;
            fuelResp.alreadyLoaded = alreadyLoaded;
            if(returnTx && execResp.tx) {
		transactions.push(execResp.tx);
            }
        }
		if(transactions.length > 0) fuelResp.transactions = transactions;

		return fuelResp
	}


	//new approach: squeeze as much as possible into a transaction by calculating the exact tx sizes, only make another tx if the max tx size is exceeded
	async function txSliceAndSend(transactions, fleet, opName, priorityFeeMultiplier, maxInstructionsPerTx) {
		// TODO: maxInstructionsPerTx is not longer used, remove later
		let txResult;
		let curPos=0;
		let curSize=1;
		let lastBlock=null;
		let allTxSlices=[];

		if(fleet.exitWarpSubwarpPending == 1) { transactions = [await execExitWarp(fleet, true)].concat(transactions); fleet.exitWarpSubwarpPending = 0; }
		if(fleet.exitWarpSubwarpPending == 2) { transactions = [await execExitSubwarp(fleet, true)].concat(transactions); fleet.exitWarpSubwarpPending = 0; fleet.exitSubwarpWillBurnFuel = 0; }

		while(curPos+curSize <= transactions.length) {

			let instructions = [];
			const priorityFee = 5;
			let curBlock = transactions.slice(curPos, curPos + curSize);

			instructions.push(solanaWeb3.ComputeBudgetProgram.setComputeUnitLimit({units: 1400000}));
			instructions.push(solanaWeb3.ComputeBudgetProgram.setComputeUnitPrice({microLamports: priorityFee}));
			curBlock.forEach(item => instructions.push(item.instruction))

			let latestBH = 'EkSnNWid2cvwEVnVx9aBqawnmiCNiDgp3gUdkDPTKN1N'; //dummy block hash
			let messageV0 = new solanaWeb3.TransactionMessage({
				payerKey: userPublicKey,
				recentBlockhash: latestBH,
				instructions,
			}).compileToV0Message(addressLookupTables);
			let tx = new solanaWeb3.VersionedTransaction(messageV0);
			//if (extraSigner) tx.sign([extraSigner]);
			let txSize = 9999;
			try {
				txSize = tx.serialize().length;
			} catch (err) {
				// couldn't serialize due to "RangeError: encoding overruns Uint8Array", so we reached the limit, txSize is still 9999 now and we will use the tx from the previous loop
			}

			// check the size (use 1216 as max size instead of 1232, just to be sure, so we have 16 bytes of free space)
			// the first ix should always fit into a tx
			if(curSize > 1 && txSize > 1216) {
				allTxSlices.push(lastBlock);
				lastBlock = null;
				curPos += curSize - 1;
				curSize = 1;
			} else {
				curSize += 1;
			}

			lastBlock = curBlock;
		}
		if(lastBlock) {
			allTxSlices.push(lastBlock);
		}

		for (const transactionsSlice of allTxSlices) {
			if(transactionsSlice.length == 1)
				txResult = await txSignAndSend(transactionsSlice[0], fleet, opName, priorityFeeMultiplier );
			else
				txResult = await txSignAndSend(transactionsSlice, fleet, opName, priorityFeeMultiplier );
			if(fleet.state.includes('ERROR')) break;
		}
		return txResult;
	}

	async function handleTransportUnloading(fleet, starbaseCoord, transportManifest, returnTx) {
		cLog(1,`${FleetTimeStamp(fleet.label)} 🚚 Unloading Transport`);
		updateFleetState(fleet, 'Unloading');

		const ammoMint = sageGameAcct.account.mints.ammo.toString();
		const fleetCurrentCargo = await solanaReadConnection.getParsedTokenAccountsByOwner(fleet.cargoHold, {programId: tokenProgramPK});

		let transactions = [];
		let unloadedAmount = 0;

		//Unloading resources from manifest
		let fuelUnloadDeficit = 0;
		let ammoUnloadDeficit = 0;
		for (const entry of transportManifest) {
			if (entry.res !== '' && entry.amt > 0) {
				const isFuel = entry.res === sageGameAcct.account.mints.fuel.toString();
				const isAmmo = entry.res === ammoMint;
				const currentRes = fleetCurrentCargo.value.find(item => item.account.data.parsed.info.mint === entry.res);
				const currentResCnt = currentRes ? currentRes.account.data.parsed.info.tokenAmount.uiAmount : 0;

				if(isFuel) fuelUnloadDeficit = entry.amt;
				if(isAmmo) ammoUnloadDeficit = entry.amt;
				let amountToUnload = entry.extra ? Math.min(currentResCnt, entry.amt) : currentResCnt;
				if(globalSettings.transportKeep1 && amountToUnload > 0) { amountToUnload -= 1; }
				if (amountToUnload > 0) {
					cLog(1,`${FleetTimeStamp(fleet.label)} Unloading ${amountToUnload} ${entry.res}`);
					let resp = await execCargoFromFleetToStarbase(fleet, fleet.cargoHold, entry.res, starbaseCoord, amountToUnload, returnTx);
					if(returnTx && resp) {
						transactions.push(resp);
					}
					if(isFuel) fuelUnloadDeficit -= amountToUnload;
					if(isAmmo) ammoUnloadDeficit -= amountToUnload;
					unloadedAmount += amountToUnload * cargoItems.find(r => r.token == entry.res).size;
				} else {
					cLog(1,`${FleetTimeStamp(fleet.label)} Unload ${entry.res} skipped - none found in ship's cargo hold`);
				}
				//if (resource == sageGameAcct.account.mints.fuel.toString() && resMax < resAmt) extraFuel = resAmt - resMax;
				//if (resource == ammoMint.toString() && resMax < resAmt) ammoToUnload = resAmt - resMax;
			}
		}

		//Ammo bank unloading
		const ammoEntry = globalSettings.transportUseAmmoBank ? transportManifest.find(e => e.res === ammoMint) : undefined;
		if (ammoEntry) {
			let fleetCurrentAmmoBank = await solanaReadConnection.getParsedTokenAccountsByOwner(fleet.ammoBank, {programId: tokenProgramPK});
			let currentAmmo = fleetCurrentAmmoBank.value.find(item => item.account.data.parsed.info.mint === ammoMint);
			let currentAmmoCnt = currentAmmo ? currentAmmo.account.data.parsed.info.tokenAmount.uiAmount : 0;
			let ammoToUnload = Math.min(currentAmmoCnt, ammoUnloadDeficit);
			if(globalSettings.transportKeep1 && ammoToUnload > 0) { ammoToUnload -= 1; }
			if (ammoToUnload > 0) {
				cLog(1,`${FleetTimeStamp(fleet.label)} Unloading Ammobanks: ${ammoToUnload}`);
				let resp = await execCargoFromFleetToStarbase(fleet, fleet.ammoBank, ammoMint, starbaseCoord, ammoToUnload, returnTx);
				if(returnTx && resp) {
					transactions.push(resp);
				}
			}
		}

		return { fuelUnloadDeficit, transactions, unloadedAmount };
	}

	async function handleTransportLoading(i, starbaseCoords, transportManifest, returnTx, alreadyUnloadedInTransaction) {
		cLog(1,`${FleetTimeStamp(userFleets[i].label)} 📦 Loading Transport`);
		updateFleetState(userFleets[i], 'Loading');

		//Use ammo banks if possible
		const ammoEntry = globalSettings.transportUseAmmoBank ? transportManifest.find(e => e.res === sageGameAcct.account.mints.ammo.toString()) : undefined;
		let resp = null;
		let transactions = [];
		let ammoLoadingIntoAmmoBank = ammoEntry ? (resp = await execLoadFleetAmmo(userFleets[i], starbaseCoords, ammoEntry.amt, returnTx)).amountLoaded : 0;
		if(returnTx && resp && resp.transaction) {
			transactions.push(resp.transaction);
			//when using a combined load tx, we need to take into account the amount loaded into the fuel tank, because the starbase still reports the original amount (otherwise we may get an ix error instead a "NotEnoughResource" error)
			ammoEntry.alreadyLoadedInTransaction = ammoLoadingIntoAmmoBank;
		}

        //Calculate remaining free cargo space
        cLog(2,`${FleetTimeStamp(userFleets[i].label)} Calculating cargoSpace ...`);
        const fleetCurrentCargo = await solanaReadConnection.getParsedTokenAccountsByOwner(userFleets[i].cargoHold, {programId: tokenProgramPK});
        const cargoCnt = fleetCurrentCargo.value.reduce((n, {account}) => n + account.data.parsed.info.tokenAmount.uiAmount * cargoItems.find(r => r.token == account.data.parsed.info.mint).size, 0);
        let cargoSpace = userFleets[i].cargoCapacity - cargoCnt;
        if(alreadyUnloadedInTransaction) cargoSpace += alreadyUnloadedInTransaction;
        const startingCargoSpace = cargoSpace;
        let expectedCnt = 0;
        cLog(2,`${FleetTimeStamp(userFleets[i].label)} cargoSpace remaining: ${cargoSpace}`);

		let notEnoughInfo = '';
		for (const entry of transportManifest) {
			if (entry.res && entry.amt > 0) {
				if(cargoSpace < 1) {
					cLog(1,`${FleetTimeStamp(userFleets[i].label)} Cargo full - remaining loading process skipped`);
					break;
				}

				const [fleetResourceToken] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
					[
						userFleets[i].cargoHold.toBuffer(),
						tokenProgramPK.toBuffer(),
						new solanaWeb3.PublicKey(entry.res).toBuffer()
					],
					programPK
				);
				const currentRes = fleetCurrentCargo.value.find(item => item.account.data.parsed.info.mint === entry.res);
				const fleetResAcct = currentRes ? currentRes.pubkey : fleetResourceToken;
				const resCargoTypeAcct = cargoTypes.find(item => item.account.mint.toString() == entry.res);
				const currentResAmt = currentRes ? currentRes.account.data.parsed.info.tokenAmount.uiAmount : 0;

				//Deduct ammo already loaded into ammobank if applicable
				const isAmmo = entry.res === sageGameAcct.account.mints.ammo.toString();
				//For ammo SLYA didn't check the ammo in the cargo room, added
				const resMax = Math.floor(Math.min(cargoSpace / cargoItems.find(r => r.token == entry.res).size, isAmmo ? entry.amt - ammoLoadingIntoAmmoBank - currentResAmt : entry.amt - currentResAmt));
				expectedCnt += resMax;
				if (resMax > 0) {
					cLog(1,`${FleetTimeStamp(userFleets[i].label)} Attempting to load ${resMax} ${entry.res} from ${starbaseCoords}`);
					const resp = await execCargoFromStarbaseToFleet(
						userFleets[i],
						userFleets[i].cargoHold,
						fleetResAcct,
						entry.res,
						resCargoTypeAcct,
						starbaseCoords,
						resMax,
						false,
						returnTx,
						((returnTx && entry.alreadyLoadedInTransaction) ? entry.alreadyLoadedInTransaction : 0)
					);
                    cLog(1,`${FleetTimeStamp(userFleets[i].label)} Loaded ${resp.amount} ${entry.res}: `, resp);
                    cargoSpace -= resp && resp.amount ? cargoItems.find(r => r.token == entry.res).size * resp.amount : 0;

					if (resp && resp.name == 'NotEnoughResource') {
						const resShort = cargoItems.find(r => r.token == entry.res).name;
						cLog(1,`${FleetTimeStamp(userFleets[i].label)} Not enough ${resShort}`);
						notEnoughInfo += 'Not enough ' + resShort + '\n';
					}
					else if(returnTx && resp && resp.tx) {
						transactions.push(resp.tx);
					}
				}
			}
		}

		//Return true if cargo was added
		//const fleetCurrentCargo = await solanaReadConnection.getParsedTokenAccountsByOwner(userFleets[i].cargoHold, {programId: tokenProgramPK});
		//const cargoCnt = fleetCurrentCargo.value.reduce((n, {account}) => n + account.data.parsed.info.tokenAmount.uiAmount, 0);

		//cLog(3,`${FleetTimeStamp(userFleets[i].label)} Loading finished with ${cargoCnt} total cargo loaded`);

	        if (startingCargoSpace == cargoSpace && expectedCnt > 0) {
	            updateFleetState(userFleets[i], 'ERROR: No cargo loaded');
	            cLog(2,`${FleetTimeStamp(userFleets[i].label)} ERROR: No cargo loaded`);
	            sisEvento('alerta', userFleets[i].label, 'Nenhuma carga carregada: ' + notEnoughInfo);
	        }
		return { success: !userFleets[i].state.includes('ERROR'), transactions: transactions};
	}

	function startupScanBlockCheck(i, fleetCoords) {
		if(!userFleets[i].startupScanBlockCheck && fleetCoords.length > 0) {
			//Calculate current scanBlockIdx
			userFleets[i].startupScanBlockCheck = true;

			if(userFleets[i].scanMove) {
					cLog(2, `${FleetTimeStamp(userFleets[i].label)} Checking scanBlock`, userFleets[i].scanBlock);
					for (let s=0; s < userFleets[i].scanBlock.length - 1; s++) {
							const testCoords = userFleets[i].scanBlock[s];
							if (fleetCoords[0] == testCoords[0] && fleetCoords[1] == testCoords[1])
							{
									if(userFleets[i].scanBlockIdx != s) {
											userFleets[i].scanBlockIdx = s;
											cLog(2, `${FleetTimeStamp(userFleets[i].label)} Resuming scanBlockIdx: ${s}`);
									}
									break;
							}
					}
			}
		}
	}

	function updateFleetMiscStats(fleet, fleetAcctInfo) {
		let fleetAcctData = sageProgram.coder.accounts.decode('fleet', fleetAcctInfo.data);
		fleet.requiredCrew = fleetAcctData.stats.miscStats.requiredCrew;
		fleet.passengerCapacity = fleetAcctData.stats.miscStats.passengerCapacity;
		fleet.crewCount = fleetAcctData.stats.miscStats.crewCount;
	}

	let lastScanMapRead = 0;
	let lastScanMapCache=null;
	let lastScanMapIsUpdating = false;
	async function readScanMap() {

		//if a concurrent call is reading the map, we wait until the read is done (up to 10 seconds)
		let loopCounter = 0;
        	while (lastScanMapIsUpdating && loopCounter < 50) {
			await wait(200);
			loopCounter++;
		}
		if(lastScanMapRead <= Date.now()-30*1000) {

			lastScanMapIsUpdating = true;
			lastScanMapCache = null;
			lastScanMapRead = Date.now();
			let message = '';
			try {
				cLog(1, 'Requesting the SDU file');
				const response = await sisFetchMapaSdu(globalSettings.scanMapURL);
				if (!response.ok) {
					message = 'Error while reading the SDU file: ' + response.status + ' ' + response.statusText;
				} else {
					const result = await response.json();
					if(result.timestamp < Date.now() - 60*10*1000) {
						cLog(1, 'Error: SDU file too old');
					} else {
						lastScanMapCache = result.scans;
					}
				}
			} catch(error) {
				message = 'Error while requesting the SDU file: ' + error.message;
			}
			lastScanMapIsUpdating = false;
			if(message) cLog(1, message);
		} else {
			//console.log('Reading cached scan map');
		}

		return lastScanMapCache;
	}

	async function scanTargetStillValid(i,fleetCoords) {
		let scanData = await readScanMap();
		if(!scanData) {
			userFleets[i].fontColor = 'yellow';
			updateAssistStatus(userFleets[i]);
			return true;
		}
		let scanX = fleetCoords[0];
		let scanY = fleetCoords[1];
		let scan = scanData.find(item => item.x == scanX && item.y == scanY);
		let cooldownLeft = (userFleets[i].scanEnd - Date.now())/1000;
		let lowerThreshold = userFleets[i].scanCooldown * (userFleets[i].scanCheckWhileCooldownLeft/100);
		if(userFleets[i].scanCooldown - lowerThreshold <= 0) return true; //prevent division by zero

		if(cooldownLeft < lowerThreshold) {
			cLog(3,`${FleetTimeStamp(userFleets[i].label)} SAM scan coord`,scanX,`/`,scanY,` lower cooldown threshold reached, fleet stays here until scan.`);
			return true;
		}

		let currentThreshold = (cooldownLeft - lowerThreshold) / (userFleets[i].scanCooldown - lowerThreshold) * userFleets[i].scanMin2;

		if(scan && scan.c * 100 < currentThreshold) {
			cLog(3,`${FleetTimeStamp(userFleets[i].label)} SAM scan coord`,scanX,`/`,scanY,`turned bad while waiting for the cooldown.`);
			return false;
		}
		cLog(3,`${FleetTimeStamp(userFleets[i].label)} SAM scan coord`,scanX,`/`,scanY,`still valid.`);
		return true;
	}

	async function scanAutoHasEnoughFuel(i, fleetCoords) {
		if(!userFleets[i].scanHomeAtPercent || userFleets[i].scanHomeAtPercent <= 100) return true;
		const starbaseCoords = ConvertCoords(userFleets[i].starbaseCoord);
		if(userFleets[i].scanStartForcedResupply) return false;
		let fuelNeeded = 0;
		if (userFleets[i].moveType == 'warp' || userFleets[i].moveType == 'warp-subwarp-warp') {
			fuelNeeded = calcWarpFuelReq(userFleets[i], fleetCoords, starbaseCoords);
		} else {
			const distFromTargetToStarbase = calculateMovementDistance(fleetCoords, starbaseCoords);
			fuelNeeded = calculateSubwarpFuelBurn(userFleets[i], distFromTargetToStarbase);
		}
		cLog(3,`${FleetTimeStamp(userFleets[i].label)} SAM last fuel amount`,userFleets[i].scanLastFuelAmount,'fuelNeeded',fuelNeeded,'fuelNeededAlt',(fuelNeeded*(userFleets[i].scanHomeAtPercent/100)));
		const hasEnoughFuel = (typeof userFleets[i].scanLastFuelAmount == 'undefined' || userFleets[i].scanLastFuelAmount > fuelNeeded * (userFleets[i].scanHomeAtPercent/100));
		if(!hasEnoughFuel) userFleets[i].scanStartForcedResupply = true;
		return hasEnoughFuel;
	}

	async function handleScanAutoMovement($,o){let _=!1,t=parseInt(userFleets[$].destCoord.split(",")[0].trim()),n=parseInt(userFleets[$].destCoord.split(",")[1].trim());if(!userFleets[$].scanAutoMoveTo){let e=ConvertCoords(userFleets[$].starbaseCoord);CoordsEqual(o,e)?userFleets[$].scanAutoMoveTo=[t,n]:userFleets[$].scanAutoMoveTo=[o[0],o[1]];return}userFleets[$].scanAutoMoveTo=null,_=await readScanMap();let s=globalSettings.scanBlockLength;userFleets[$].scanPatternLength&&(s=userFleets[$].scanPatternLength);let a=globalSettings.scanBlockPattern;userFleets[$].scanPattern&&(a=userFleets[$].scanPattern),s<5&&(s=5);let l=t-s,u=t+s,r=n-s,i=n+s,c=userFleets[$].scanNeighborhoodMinGood,f=parseInt(userFleets[$].scanClusterFactor),h=parseInt(userFleets[$].scanSearchDist);if(_){let d=0,p=[0,0],b=await scanAutoHasEnoughFuel($,o),v=1,g=1;a.includes("2hv")&&(v=2,g=2);let M=0,m=[];if(b){for(let A=o[0]-2;A<=o[0]+2;A++)for(let y=o[1]-2;y<=o[1]+2;y++){if(A==o[0]&&y==o[1]||A<l||A>u||y<r||y>i)continue;let T=_.find($=>$.x==A&&$.y==y);T&&100*T.c>=userFleets[$].scanMin&&M++}if(M>=c){let x=[];for(let S=0-v;S<=0+v;S++)for(let P=0-g;P<=0+g;P++){let B=[o[0]+S,o[1]+P];if(!(a.includes("2hv")&&(Math.abs(S)>=2&&0!=P||Math.abs(P)>=2&&0!=S))&&!(B[0]<l)&&!(B[0]>u)&&!(B[1]<r)&&!(B[1]>i)){if(0!=S||0!=P){let C=_.find($=>$.x==B[0]&&$.y==B[1]);C&&100*C.c>=userFleets[$].scanMin&&m.push([S,P]),C&&C.c>d&&(d=C.c,p=[S,P])}x.push([S,P])}}if(m.length){.5>Math.random()&&(p=m[Math.floor(Math.random()*m.length)]),m=[];let k=[];if(Math.abs(p[0])>=1&&Math.abs(p[1])>=1)k.push([o[0]+p[0],o[1]+p[1]]),k.push([o[0]+p[0],o[1]]),k.push([o[0]+p[0],o[1]+-1*p[1]]),k.push([o[0],o[1]+p[1]]),k.push([o[0]+-1*p[0],o[1]+p[1]]),a.includes("2hv")&&(k.push([o[0],o[1]+2*p[1]]),k.push([o[0]+2*p[0],o[1]]));else for(let w of x)(0!=w[0]||0!=w[1])&&(Math.abs(p[0])>=1&&(0==w[0]||Math.sign(w[0])==Math.sign(p[0]))||Math.abs(p[1])>=1&&(0==w[1]||Math.sign(w[1])==Math.sign(p[1])))&&k.push([o[0]+w[0],o[1]+w[1]]);for(let L of k){if(L[0]<l||L[0]>u||L[1]<r||L[1]>i)continue;let q=_.find($=>$.x==L[0]&&$.y==L[1]);(!q||100*q.c>=userFleets[$].scanMin)&&m.push([L[0],L[1]])}}}}if(M>=c&&m.length>0){let F=m[Math.floor(Math.random()*m.length)];userFleets[$].scanAutoMoveTo=F}else{let D=0,G=0,N=[0,0],R=0;if(b){let j=2.85,z=3;for(;z<2*s-2&&z<=h;){for(let E=-1;E<=1;E+=.2)for(let H=-1;H<=1;H+=.2){if(10!=Math.round(10*Math.abs(E))&&10!=Math.round(10*Math.abs(H)))continue;let I=E,J=H,K=[o[0]+Math.round(I*z),o[1]+Math.round(J*z)];if(K[0]<l||K[0]>u||K[1]<r||K[1]>i)continue;let O=Math.round(j),Q=0,U=0,V=0,W=j*j;for(let X=Math.floor(0-j);X<=Math.ceil(j);X++)for(let Y=Math.floor(0-j);Y<=Math.ceil(j);Y++)if(X*X+Y*Y<=W){if(K[0]+X<l||K[0]+X>u||K[1]+Y<r||K[1]+Y>i)continue;let Z=_.find($=>$.x==K[0]+X&&$.y==K[1]+Y);Z&&100*Z.c>userFleets[$].scanMin&&(Q++,U+=Z.c),V++}Q>=2&&U/Math.pow(Q,f/100)/Math.sqrt(V)>G&&(D=Q,G=U/Math.pow(Q,f/100)/Math.sqrt(V),N=[Math.round(I),Math.round(J)],R=V)}j+=.4,z+=1}}else{let $$=ConvertCoords(userFleets[$].starbaseCoord),$o=[$$[0]-o[0],$$[1]-o[1]];if(0!=$o[0]||0!=$o[1]){if(2>=Math.abs($o[0])&&2>=Math.abs($o[1])){userFleets[$].scanAutoMoveTo=$$,userFleets[$].scanForceResupply=!0,cLog(3,`${FleetTimeStamp(userFleets[$].label)} SAM heading back and near the SB, moving to SB`);return}let $_=[$o[0],$o[1]].map($=>$/Math.max(Math.abs($o[0]),Math.abs($o[1])));N=[Math.round($_[0]),Math.round($_[1])],D=1,a="auto(1,2hv++)",cLog(3,`${FleetTimeStamp(userFleets[$].label)} SAM heading back, direction`,N[0],"/",N[1])}}if(D>0){let $t=N[0],$n=N[1];("auto(1,2hv++)"==a||("auto(1,2hv)"==a||"auto(1,2hv+)"==a)&&(0==N[0]||0==N[1]))&&($t=2*N[0],$n=2*N[1]),p=[o[0]+$t,o[1]+$n];let $e=[],$s=_.find($=>$.x==p[0]&&$.y==p[1]);if($s){m=[],1>=Math.abs($t)&&1>=Math.abs($n)?"auto(1+)"==a&&(0==$t||0==$n)?m.push([o[0]+2*$t,o[1]+2*$n]):0==$t?(m.push([o[0]-1,o[1]+$n]),m.push([o[0]+1,o[1]+$n])):0==$n?(m.push([o[0]+$t,o[1]-1]),m.push([o[0]+$t,o[1]+1])):"auto(1,2hv+)"==a?(m.push([o[0]+2*$t,o[1]+$n]),m.push([o[0]+$t,o[1]+2*$n])):(m.push([o[0],o[1]+$n]),m.push([o[0]+$t,o[1]])):"auto(1,2hv++)"==a&&Math.abs($t)>=2&&Math.abs($n)>=2?(m.push([o[0]+$t,o[1]+$n/2]),m.push([o[0]+$t/2,o[1]+$n])):"auto(1,2hv)"==a?m.push([o[0]+N[0],o[1]+N[1]]):0==$t?(m.push([o[0]-1,o[1]+$n]),m.push([o[0]+1,o[1]+$n])):0==$n&&(m.push([o[0]+$t,o[1]-1]),m.push([o[0]+$t,o[1]+1]));let $a=1;for(let $l of m){let $u=_.find($=>$.x==$l[0]&&$.y==$l[1]);$u&&$u.c>=$s.c+userFleets[$].scanBypassPercent/100&&($e.push([$l[0],$l[1]]),$a>$u.c&&($a=$u.c))}if($e.length>=2)for(let $r of $e){let $i=_.find($=>$.x==$r[0]&&$.y==$r[1]);$i&&$i.c>=$a+userFleets[$].scanBypassPercent/100&&($e=[[$r[0],$r[1]]])}}$e.length?userFleets[$].scanAutoMoveTo=$e[Math.floor(Math.random()*$e.length)]:userFleets[$].scanAutoMoveTo=p}}}if(!userFleets[$].scanAutoMoveTo){let $c=o[0],$f=o[1];$c>t-4&&$c<t+4&&$f>n-4&&$f<n+4?(.5>Math.random()?.5>Math.random()?$c++:$c--:.5>Math.random()?$f++:$f--,cLog(3,`${FleetTimeStamp(userFleets[$].label)} SAM found no best sector and no good direction, moving random to`,$c,"/",$f)):($c<t&&$c++,$c>t&&$c--,$f<n&&$f++,$f>n&&$f--,cLog(3,`${FleetTimeStamp(userFleets[$].label)} SAM found no best sector and no good direction, moving back to start to`,$c,"/",$f)),userFleets[$].scanAutoMoveTo=[$c,$f]}_||(userFleets[$].fontColor="yellow",updateAssistStatus(userFleets[$]))}

	async function operateFleet(i) {
        if (globalErrorTracker.errorCount > 9) toggleAssistant('ERROR');

		//Don't run fleets in an error state
		if (userFleets[i].state.includes('ERROR')) return;

		userFleets[i].lastOp = Date.now();

		if (userFleets[i].stopping || userFleets[i].state.includes('STOPPED')) {
			userFleets[i].stopping = false;
			updateFleetState(userFleets[i], 'STOPPED');
			return;
		}

		const moving =
			userFleets[i].state.includes('Move [') ||
			userFleets[i].state.includes('Warp [') ||
			userFleets[i].state.includes('Subwarp [');
		const waitingForWarpCD = userFleets[i].state.includes('Warp C/D');
		const scanning = userFleets[i].state.includes('Scan') || userFleets[i].state.includes('scan cooldown');
		const mining = userFleets[i].mineEnd && userFleets[i].state.includes('Mine') && (Date.now() < userFleets[i].mineEnd);
		const onTarget = userFleets[i].lastScanCoord == userFleets[i].destCoord;
		const waitingForScan = userFleets[i].scanEnd && (Date.now() <= userFleets[i].scanEnd);
		if(moving) cLog(2, `${FleetTimeStamp(userFleets[i].label)} Operating moving fleet`);
		if(userFleets[i].resupplying || mining) return;
		if(!onTarget && waitingForWarpCD) return;
		let forceNewAutoMoveTarget = false;
		if(scanning && onTarget && waitingForScan) {
			if(userFleets[i].scanAutoMoveTo && userFleets[i].scanCheckWhileCooldownLeft && !await scanTargetStillValid(i,userFleets[i].scanAutoMoveTo))
				forceNewAutoMoveTarget = true;
			else
				return;
		}

		try {
				let fleetSavedData = await GM.getValue(userFleets[i].publicKey.toString(), '{}');
				let fleetParsedData = JSON.parse(fleetSavedData);
				//if(!fleetParsedData.assignment) return;

				userFleets[i].iterCnt++;
				cLog(2, `${FleetTimeStamp(userFleets[i].label)} <getAccountInfo> (${userFleets[i].state})`);
				let fleetAcctInfo = await getAccountInfo(userFleets[i].label, 'full fleet info', userFleets[i].publicKey);
				updateFleetMiscStats(userFleets[i],fleetAcctInfo);
				let [fleetState, extra] = getFleetState(fleetAcctInfo, userFleets[i]);
				cLog(3, `${FleetTimeStamp(userFleets[i].label)} chain fleet state: ${fleetState}`);
				let fleetCoords = fleetState == 'Idle' ? extra : [];
				let fleetMining = fleetState == 'MineAsteroid' ? extra : null;
				userFleets[i].startingCoords = fleetCoords;

				//Correct rare fleet state mismatch bug
				if(moving && fleetState == 'Idle') {
					cLog(1,`${FleetTimeStamp(userFleets[i].label)} Fleet State Mismatch - Updating from ${userFleets[i].state} to ${fleetState}`);
					updateFleetState(userFleets[i], fleetState);
				}

				if ((userFleets[i].iterCnt < 2) && fleetState == 'StarbaseLoadingBay') {
					if(fleetParsedData.assignment == 'Scan' || fleetParsedData.assignment == 'Mine' || fleetParsedData.assignment == 'Transport')
						await execStartupUndock(i, fleetParsedData.assignment);
				}
				else if (fleetState == 'MoveWarp' || fleetState == 'MoveSubwarp') {
					cLog(2, `${FleetTimeStamp(userFleets[i].label)} executing handleMovement`);
					await handleMovement(i, null, null, null);
				}
				else if (fleetParsedData.assignment == 'Scan' && fleetState == 'Idle') {
					updateFleetState(userFleets[i], fleetState);
					let scanBlockPattern = globalSettings.scanBlockPattern;
					if(userFleets[i].scanPattern) scanBlockPattern=userFleets[i].scanPattern;
					if(!scanBlockPattern.includes('auto')) {
						startupScanBlockCheck(i, fleetCoords);
						const curentSBI = userFleets[i].scanBlockIdx;
						await handleScan(i, fleetCoords, userFleets[i].scanBlock[curentSBI]);

						if(userFleets[i].stopping) return;

						//Move instantly if a move is needed as the result of the previous scan
						if(curentSBI !== userFleets[i].scanBlockIdx)	await handleScan(i, fleetCoords, userFleets[i].scanBlock[userFleets[i].scanBlockIdx]);
					} else {
						if(!userFleets[i].scanAutoMoveTo || forceNewAutoMoveTarget) await handleScanAutoMovement(i,fleetCoords);
						let currentAutoMove = [userFleets[i].scanAutoMoveTo[0], userFleets[i].scanAutoMoveTo[1]];
						await handleScan(i, fleetCoords, userFleets[i].scanAutoMoveTo);
						if(!userFleets[i].scanAutoMoveTo) await handleScanAutoMovement(i,fleetCoords);
						if(userFleets[i].stopping) return;
						if(userFleets[i].scanAutoMoveTo && (currentAutoMove[0] != userFleets[i].scanAutoMoveTo[0] || currentAutoMove[1] != userFleets[i].scanAutoMoveTo[1])) await handleScan(i, fleetCoords, userFleets[i].scanAutoMoveTo);
					}
				}
				else if (fleetParsedData.assignment == 'Mine') {
					if(fleetState == 'MineAsteroid' && !userFleets[i].state.includes('Mine')) {
						cLog(1,`${FleetTimeStamp(userFleets[i].label)} Fleet State Mismatch - retrying`);
						for(let retryCount=1; retryCount<=4; retryCount++) {
							await wait(10000);
							fleetAcctInfo = await getAccountInfo(userFleets[i].label, 'full fleet info', userFleets[i].publicKey);
							[fleetState, extra] = getFleetState(fleetAcctInfo, userFleets[i]);
							cLog(1, `${FleetTimeStamp(userFleets[i].label)} chain fleet state (after retry ${retryCount}/4): ${fleetState}`);
							fleetCoords = fleetState == 'Idle' ? extra : [];
							fleetMining = fleetState == 'MineAsteroid' ? extra : null;
							userFleets[i].startingCoords = fleetCoords;
							if(fleetState != 'MineAsteroid') {
								break;
							}
						}
						if(fleetState == 'MineAsteroid' && !userFleets[i].state.includes('Mine')) {
							cLog(1,`${FleetTimeStamp(userFleets[i].label)} Fleet State Mismatch - Updating to Mining again`);
							updateFleetState(userFleets[i], 'Mine [' + TimeToStr(new Date(Date.now())) + ']');
						}
					}
					await handleMining(i, userFleets[i].state, fleetCoords, fleetMining);
				}
				else if (fleetParsedData.assignment == 'Transport') {
					await handleTransport(i, userFleets[i].state, fleetCoords);
				}
		} catch (err) {
			cLog(1,`${FleetTimeStamp(userFleets[i].label)} ERROR`, err);
			logError('Error while operating fleet: ' + (err.message ? err.message : err));
		}
	}

	async function startFleet(i) {
		//Bail if assistant is stopped
		if (!enableAssistant) return;
		if (!sisCarteiraOk()) { sisCarteiraAusente(); setTimeout(() => { startFleet(i); }, 15000); return; }

		let extraTime = 0;
		const fleet = userFleets[i];

		try {
			//cLog(1,`${FleetTimeStamp(userFleets[i].label)} Operating fleet ...`);
			const fleetSavedData = await GM.getValue(fleet.publicKey.toString(), '{}');
			const fleetParsedData = JSON.parse(fleetSavedData);

			//Bail if no assignment
			if(fleetParsedData.assignment) {
				fleet.fontColor = 'aquamarine';
				updateAssistStatus(fleet);

				if(!fleet.initilizedScanPDAs && fleetParsedData.assignment == 'Scan') {
					await createScannerPDAs(fleet);
					fleet.initilizedScanPDAs = true;
				}

				await operateFleet(i);

				fleet.fontColor = 'white';
				updateAssistStatus(fleet);
			}
			//in case we stopped the fleet AND removed the assignment, we still need to check for the 'stopping' state
			else if(userFleets[i].stopping || userFleets[i].state.includes('STOPPED')) {
				userFleets[i].stopping = false;
				updateFleetState(userFleets[i], 'STOPPED');
			}
		}
		catch(error) {
			extraTime = 20000;
			cLog(1,`${FleetTimeStamp(fleet.label)} Uncaught error - waiting 20s longer`, error);

			fleet.fontColor = 'crimson';
			updateAssistStatus(fleet);
		}

		//Add extra wait time if an uncaught error occurred
		setTimeout(() => { startFleet(i); }, 10000 + extraTime);
	}

    async function getStarbasePlayerCargoHolds(starbasePlayer) {
        let starbasePlayerCargoHolds = await cargoProgram.account.cargoPod.all([
            {
                memcmp: {
                    offset: 41,
                    bytes: starbasePlayer.toBase58(),
                },
            },
        ]);

        let starbasePlayerCargoHoldsAndTokens = [];
        for (let cargoHold of starbasePlayerCargoHolds) {
            if (cargoHold.account && cargoHold.account.openTokenAccounts > 0) {
                let cargoHoldTokensRaw = await solanaReadConnection.getParsedTokenAccountsByOwner(cargoHold.publicKey, {programId: tokenProgramPK});
                let cargoHoldTokens = cargoHoldTokensRaw.value.map(item => ({cargoHoldToken: item.pubkey, mint: item.account.data.parsed.info.mint, amount: item.account.data.parsed.info.tokenAmount.uiAmount}));
                starbasePlayerCargoHoldsAndTokens.push({starbasePlayerCargoHold: cargoHold.publicKey, cargoHoldTokens: cargoHoldTokens});
            }
        }
        return starbasePlayerCargoHoldsAndTokens;
    }

    function getStarbasePlayerCargoMaxItem(starbasePlayerCargoHoldsAndTokens, mint) {
        let cargoHold = starbasePlayerCargoHoldsAndTokens.reduce((prev, curr) => {
            let prevCargoHoldToken = prev && prev.cargoHoldTokens.find(item => item.mint === mint);
            let prevAmount = prevCargoHoldToken ? prevCargoHoldToken.amount : -1;
            let currCargoHoldToken = curr.cargoHoldTokens.find(item => item.mint === mint);
            let currAmount = currCargoHoldToken ? currCargoHoldToken.amount : -1;
            return prevAmount > currAmount ? prev : currAmount > -1 ? curr : null;
        });
        return cargoHold;
    }

    function getTargetRecipe(starbasePlayerCargoHoldsAndTokens, userCraft, targetAmount, special) {
        let targetRecipe = null;
        let outputExisting = 0;
        let allRecipes = craftRecipes.concat(upgradeRecipes);
        let craftRecipe = userCraft.name ? userCraft : allRecipes.find(item => item.name === userCraft.item);

        let starbasePlayerIngredientCargoHolds = [];
        for (let input of craftRecipe.input) {
            let craftAmount = input.amount * targetAmount;
            let starbasePlayerCargoHold = getStarbasePlayerCargoMaxItem(starbasePlayerCargoHoldsAndTokens, input.mint.toString());
            if (starbasePlayerCargoHold && starbasePlayerCargoHold.cargoHoldTokens) {
                let cargoHoldToken = starbasePlayerCargoHold.cargoHoldTokens.find(item => item.mint === input.mint.toString());
                cargoHoldToken = cargoHoldToken ? cargoHoldToken : {mint: input.mint.toString(), amount: 0};
                let amountCraftable = cargoHoldToken.amount ? Math.floor(cargoHoldToken.amount / input.amount) : 0;
                craftAmount = craftAmount - cargoHoldToken.amount;
                starbasePlayerIngredientCargoHolds.push({starbasePlayerCargoHold: starbasePlayerCargoHold.starbasePlayerCargoHold, cargoHoldToken: cargoHoldToken, amountCraftable: amountCraftable, craftAmount: craftAmount});
            } else {
                starbasePlayerIngredientCargoHolds.push({starbasePlayerCargoHold: null, cargoHoldToken: {mint: input.mint.toString()}, amountCraftable: 0, craftAmount: craftAmount});
            }
        }

        let limitingIngredient = starbasePlayerIngredientCargoHolds.reduce((prev, curr) => prev && prev.amountCraftable < curr.amountCraftable ? prev : curr);

        console.log('DEBUG limitingIngredient: ', limitingIngredient);
        console.log('DEBUG targetAmount: ', targetAmount);
        if (limitingIngredient.amountCraftable < targetAmount) {
            for (let ingredient of starbasePlayerIngredientCargoHolds) {
                console.log('DEBUG ingredient: ', ingredient);
                if (ingredient.craftAmount > 0) {
                    let filterData = ['SDU'];
                    if(special.includes('f2')) filterData.push('Framework','Framework 3');
                    else if(special.includes('f3')) filterData.push('Framework','Framework 2');
                    else filterData.push('Framework 2','Framework 3');
                    if(special.includes('t2')) filterData.push('Toolkit','Toolkit 3');
                    else if(special.includes('t3')) filterData.push('Toolkit','Toolkit 2');
                    else filterData.push('Toolkit 2','Toolkit 3');
                    //let filteredCraftRecipes = craftRecipes.filter(item => !['Framework 2','Framework 3','Toolkit 2','Toolkit 3','SDU'].includes(item.name));
                    let filteredCraftRecipes = craftRecipes.filter(item => !(filterData.includes(item.name)));
                    let ingredientRecipes = filteredCraftRecipes.filter(item => item.output.mint.toString() === ingredient.cargoHoldToken.mint);
                    for (let ingredientRecipe of ingredientRecipes) {
                        let checkRecipe = getTargetRecipe(starbasePlayerCargoHoldsAndTokens, ingredientRecipe, ingredient.craftAmount, special);
                        if (checkRecipe) targetRecipe = checkRecipe;
                    }
                }
            }
        } else {
            targetRecipe = {craftRecipe: craftRecipe, amountCraftable: limitingIngredient.amountCraftable, craftAmount: targetAmount};
        }

        return targetRecipe;
    }

    async function getStarbaseTime(starbase, activity) {
        const EMPTY_CRAFTING_SPEED_PER_TIER = [0, .2, .275, .35, .425, .5, .5];
	//const EMPTY_CRAFTING_SPEED_PER_TIER = [0, 1, 1, 1, 1, 1, 1];
        let currTime = Date.now() / 1000;
        let gameStateAcct = await sageProgram.account.gameState.fetch(sageGameAcct.account.gameState);
        let sbLevel = gameStateAcct.fleet.upkeep['level' + starbase.account.level];
        let lastGlobalUpdate, lastLocalUpdate, resAmount, resDepletionRate;
        if (activity === 'Craft') {
            lastGlobalUpdate = starbase.account.upkeepFoodGlobalLastUpdate.toNumber();
            lastLocalUpdate = starbase.account.upkeepFoodLastUpdate.toNumber();
            resAmount = starbase.account.upkeepFoodBalance.toNumber();
            resDepletionRate = sbLevel.foodDepletionRate/100;
        } else {
            lastGlobalUpdate = starbase.account.upkeepToolkitGlobalLastUpdate.toNumber();
            lastLocalUpdate = starbase.account.upkeepToolkitLastUpdate.toNumber();
            resAmount = starbase.account.upkeepToolkitBalance.toNumber();
            resDepletionRate = sbLevel.toolkitDepletionRate/100;
        }
        const timeSinceGlobalUpdate = currTime > lastGlobalUpdate ? currTime - lastGlobalUpdate : 0;
        const upkeepTimeRemaining = resDepletionRate > 0 ? resAmount / resDepletionRate : 0;
        const globalUpkeepTimeRemaining = Math.min(timeSinceGlobalUpdate, upkeepTimeRemaining);
        const localUpkeepTimeRemaining = lastLocalUpdate + globalUpkeepTimeRemaining > currTime ? currTime - lastLocalUpdate : globalUpkeepTimeRemaining;
        const resForLocalTimeDiff = localUpkeepTimeRemaining * resDepletionRate;
        const resRemainingLocalTimeDiff = resAmount - resForLocalTimeDiff;
        const resRemainingLocalTimeDiffMin = resRemainingLocalTimeDiff < 0 ? 0 : resRemainingLocalTimeDiff;
        const minUpkeepTimeRemaining = Math.min(localUpkeepTimeRemaining, upkeepTimeRemaining);
        const emptyAdjustment = minUpkeepTimeRemaining == upkeepTimeRemaining ? (timeSinceGlobalUpdate - minUpkeepTimeRemaining) * EMPTY_CRAFTING_SPEED_PER_TIER[starbase.account.level] : 0;
        let starbaseTime = activity === 'Craft' ? lastLocalUpdate + localUpkeepTimeRemaining + emptyAdjustment : lastLocalUpdate + localUpkeepTimeRemaining;
        //let starbaseTime = activity === 'Craft' ? lastLocalUpdate + timeSinceGlobalUpdate * EMPTY_CRAFTING_SPEED_PER_TIER[starbase.account.level] : lastLocalUpdate + localUpkeepTimeRemaining;
        return {starbaseTime: starbaseTime, resRemaining: resRemainingLocalTimeDiffMin};
    }

    async function updateCraft(userCraft) {
        let craftSavedData = await GM.getValue(userCraft.label, '{}');
        let craftParsedData = JSON.parse(craftSavedData);
        craftParsedData.state = userCraft.state;
        craftParsedData.craftingId = userCraft.craftingId;
        craftParsedData.craftingCoords = userCraft.craftingCoords;
        craftParsedData.feeAtlas = userCraft.feeAtlas;
	craftParsedData.errorCount = userCraft.errorCount;
        await GM.setValue(userCraft.label, JSON.stringify(craftParsedData));
    }

    async function craftTimeoutAfterError(userCraft) {
		let craftSavedData = await GM.getValue(userCraft.label, '{}');
        let craftParsedData = JSON.parse(craftSavedData);
		craftParsedData.errorCount = (typeof craftParsedData.errorCount == "undefined" ? 1 : craftParsedData.errorCount + 1);
        await GM.setValue(userCraft.label, JSON.stringify(craftParsedData));
        const waitMinutes = Math.min(20, craftParsedData.errorCount * 2 - 1);
        updateFleetState(userCraft, userCraft.state + " (" + waitMinutes + "m)", true);
        return waitMinutes * 60000;
	}


    async function startCraft(userCraft) {
        if (!enableAssistant) return;
        if (!sisCarteiraOk()) { sisCarteiraAusente(); setTimeout(() => { startCraft(userCraft); }, 15000); return; }
	let localTimeout = 60000;
        try {
            const EMPTY_CRAFTING_SPEED_PER_TIER = [0, .2, .275, .35, .425, .5, .5];
            //const EMPTY_CRAFTING_SPEED_PER_TIER = [0, 1, 1, 1, 1, 1, 1];
            let craftSavedData = await GM.getValue(userCraft.label, '{}');
            let craftParsedData = JSON.parse(craftSavedData);
            userCraft = craftParsedData;
            updateFleetState(userCraft, userCraft.state);

            //if this job isn't active, we exit immediately and check every 10 seconds for an update, this saves at least 2 RPC requests. Also it prevents a error in getRecipe
            if(userCraft.state === 'Idle' && (!userCraft.item || !userCraft.coordinates || !userCraft.amount)) {
		setTimeout(() => { startCraft(userCraft); }, 10000);
		return;
            }

            let targetX;
            let targetY;
            let starbase;
            let craftTime;
            let upgradeTime;
            let starbasePlayer;
            if(userCraft.craftingId) {

	            // temporary, when updating SLYA there is no craftingCoords first, so we need a fallback. Can be removed later.
	            if(!userCraft.craftingCoords) {
			userCraft.craftingCoords = userCraft.coordinates;
	            }
		    targetX = userCraft.craftingCoords.split(',')[0].trim();
		    targetY = userCraft.craftingCoords.split(',')[1].trim();
	            starbase = await getStarbaseFromCoords(targetX, targetY, true);
	            cLog(2, FleetTimeStamp(userCraft.label), 'starbase: ', starbase);
	            craftTime = await getStarbaseTime(starbase, 'Craft');
	            upgradeTime = await getStarbaseTime(starbase, 'Upgrade');
	            cLog(2, FleetTimeStamp(userCraft.label), 'craftTime: ', craftTime);
	            cLog(2, FleetTimeStamp(userCraft.label), 'upgradeTime: ', upgradeTime);

	            starbasePlayer = await getStarbasePlayer(userProfileAcct, starbase.publicKey);
	            starbasePlayer = starbasePlayer ? starbasePlayer.publicKey : await execRegisterStarbasePlayer('Craft', userCraft.craftingCoords);

	            // Get all crafting instances at designated Starbase
	            let craftingInstances = await sageProgram.account.craftingInstance.all([
	                {
	                    memcmp: {
	                        offset: 11,
	                        bytes: starbasePlayer.toBase58(),
	                    },
	                },
	            ]);

	            //let completeBN = new BrowserAnchor.anchor.BN(2);
	            //let completeArr = completeBN.toTwos(64).toArrayLike(BrowserBuffer.Buffer.Buffer, "be", 2);
	            //let complete58 = bs58.encode(completeArr);

	            if (craftingInstances.length < 1) {
	                userCraft.craftingId = 0;
	                updateFleetState(userCraft, 'Idle');
	                await updateCraft(userCraft);
	                //await GM.setValue(userCraft.label, JSON.stringify(userCraft));
	            }

	            // Get all completed crafting processes at the designated Starbase
	            let completedCraftingProcesses = [];
	            let completedUpgradeProcesses = [];
	            let craftingProcessRunning = false;
	            for (let craftingInstance of craftingInstances) {
	                let craftingProcesses = await craftingProgram.account.craftingProcess.all([
	                    {
	                        memcmp: {
	                            offset: 17,
	                            bytes: craftingInstance.publicKey.toBase58(),
	                        },
	                    },
	                ]);

	                for (let craftingProcess of craftingProcesses) {
	                    if (userCraft.craftingId && craftingProcess.account.craftingId.toNumber() == userCraft.craftingId) craftingProcessRunning = true;
	                    if (craftRecipes.some(item => item.publicKey.toString() === craftingProcess.account.recipe.toString())) {
				//fix: if we found ANY other completed process in this starbase, the time for this crafting process wasn't updated, because the first if statement was true
				//therefore we now only select the corresponding completed crafting process
				//this also fixes: we only execute the main code block below when the condition ...
				//if((!craftingProcessRunning) || completedCraftingProcesses.length || completedUpgradeProcesses.length)
				//...is true. But when there was ANY completed craft at this starbase (even a manual one) the block was always unnecessarily executed
				//todo: the later loop through completedCraftingProcesses isn't needed anymore, because we should only have exactly one completed process (this job)
	                        if (craftingProcess.account.endTime.toNumber() < craftTime.starbaseTime && [2,3].includes(craftingProcess.account.status) && userCraft.craftingId && craftingProcess.account.craftingId.toNumber() == userCraft.craftingId) {
	                            completedCraftingProcesses.push({craftingProcess: craftingProcess.publicKey, craftingInstance: craftingInstance.publicKey, recipe: craftingProcess.account.recipe, status: craftingProcess.account.status, inputsChecksum: craftingProcess.account.inputsChecksum, outputsChecksum: craftingProcess.account.outputsChecksum, craftingId: craftingProcess.account.craftingId.toNumber(), quantity: craftingProcess.account.quantity.toNumber() });
	                        } else if (userCraft.craftingId && craftingProcess.account.craftingId.toNumber() == userCraft.craftingId) {
	                            let craftRecipe = craftRecipes.find(item => item.publicKey.toString() === craftingProcess.account.recipe.toString());
	                            let calcEndTime = Math.max(craftingProcess.account.endTime.toNumber() - craftTime.starbaseTime, 0);
	                            let adjustedEndTime = craftTime.resRemaining > 0 ? calcEndTime : (calcEndTime) / EMPTY_CRAFTING_SPEED_PER_TIER[starbase.account.level];
	                            //let adjustedEndTime = (calcEndTime) / EMPTY_CRAFTING_SPEED_PER_TIER[starbase.account.level];
	                            //let craftTimeStr = 'Crafting [' + TimeToStr(new Date(Date.now() + adjustedEndTime * 1000)) + ']';
	                            let craftTimeStr = "\u2692 " + craftRecipe.name + (userCraft.item!=craftRecipe.name?' ('+userCraft.item+')':'') + ' [' + TimeToStr(new Date(Date.now() + adjustedEndTime * 1000)) + ']';
	                            updateFleetState(userCraft, craftTimeStr);
	                            await updateCraft(userCraft);
	                            //update less frequently if we have a long-running crafting task (3 minutes if remaining time >12 minutes, 2 minutes if remaining time >8 minutes), update faster if <60 seconds left
	                            if(adjustedEndTime > 900) localTimeout = 300000;
	                            else if(adjustedEndTime > 180) localTimeout = adjustedEndTime * 1000 / 3;
	                            else if(adjustedEndTime < 60) localTimeout = 30000;
	                        }
	                    } else {
				//same fix here
	                        if (craftingProcess.account.endTime.toNumber() < upgradeTime.starbaseTime && [2,3].includes(craftingProcess.account.status) && userCraft.craftingId && craftingProcess.account.craftingId.toNumber() == userCraft.craftingId) {
	                            completedUpgradeProcesses.push({craftingProcess: craftingProcess.publicKey, craftingInstance: craftingInstance.publicKey, recipe: craftingProcess.account.recipe, status: craftingProcess.account.status, craftingId: craftingProcess.account.craftingId.toNumber()});
	                        } else if (userCraft.craftingId && craftingProcess.account.craftingId.toNumber() == userCraft.craftingId) {
	                            let upgradeTimeDiff = Math.max(craftingProcess.account.endTime.toNumber() - upgradeTime.starbaseTime, 0);
	                            let upgradeTimeStr = upgradeTime.resRemaining > 0 ? 'Upgrading [' + TimeToStr(new Date(Date.now() + upgradeTimeDiff * 1000)) + ']' : 'Paused [' + parseInt(upgradeTimeDiff/60) + 'm remaining]';
	                            updateFleetState(userCraft, upgradeTimeStr);
	                            await updateCraft(userCraft);
	                        }
	                    }
	                }
	            }

	            if (!craftingProcessRunning) {
	                cLog(1,`${FleetTimeStamp(userCraft.label)} Crafting process not found. Setting state to Idle.`);
	                userCraft.craftingId = 0;
	                updateFleetState(userCraft, 'Idle');
	                await updateCraft(userCraft);
	            }

	            // only get current cargo holds if something needs to be completed
	            if(completedCraftingProcesses.length || completedUpgradeProcesses.length)
	            {
	                let starbasePlayerCargoHoldsAndTokens = await getStarbasePlayerCargoHolds(starbasePlayer);

	                for (let craftingProcess of completedCraftingProcesses) {
	                    let craftRecipe = craftRecipes.find(item => item.publicKey.toString() === craftingProcess.recipe.toString());
	                    if (userCraft.craftingId && craftingProcess.craftingId == userCraft.craftingId) {
	                        cLog(1,`${FleetTimeStamp(userCraft.label)} Completing craft at [${targetX}, ${targetY}] for  ${craftRecipe.output.mint.toString()}`);
	                        //updateFleetState(userCraft, 'Craft Completing');
	                        updateFleetState(userCraft, 'Completing: ' + craftRecipe.name + (userCraft.item!=craftRecipe.name?' ('+userCraft.item+')':''));
	                        await execCompleteCrafting(starbase, starbasePlayer, starbasePlayerCargoHoldsAndTokens, craftingProcess, userCraft);
	                        if (!userCraft.state.includes('ERROR')) {
	                            if (userCraft.craftingId && craftingProcess.craftingId == userCraft.craftingId) {
	                                userCraft.craftingId = 0;
					userCraft.errorCount = 0;
	                                updateFleetState(userCraft, 'Idle');
	                                await updateCraft(userCraft);
	                            }
	                        }
				else { localTimeout = await craftTimeoutAfterError(userCraft); }
	                    }
	                }

	                for (let upgradeProcess of completedUpgradeProcesses) {
	                    let craftingRecipe = upgradeRecipes.find(item => item.publicKey.toString() === upgradeProcess.recipe.toString());
	                    if (userCraft.craftingId && upgradeProcess.craftingId == userCraft.craftingId) {
	                        cLog(1,`${FleetTimeStamp(userCraft.label)} Completing upgrade at [${targetX}, ${targetY}] for  ${craftingRecipe.input[0].mint.toString()}`);
	                        updateFleetState(userCraft, 'Upgrade Completing');
	                        await execCompleteUpgrade(starbase, starbasePlayer, starbasePlayerCargoHoldsAndTokens, upgradeProcess, userCraft);
	                        if (!userCraft.state.includes('ERROR')) {
	                            if (userCraft.craftingId && upgradeProcess.craftingId == userCraft.craftingId) {
	                                userCraft.craftingId = 0;
					userCraft.errorCount = 0;
	                                updateFleetState(userCraft, 'Idle');
	                                await updateCraft(userCraft);
	                                //await GM.setValue(userCraft.label, JSON.stringify(userCraft));
	                            }
	                        }
				else { localTimeout = await craftTimeoutAfterError(userCraft); }
	                    }
	                }
		    }
	    } else {
		userCraft.craftingId = 0; //set first, so the coords are not taken from the last crafting job
		updateFleetState(userCraft, 'Idle');
		await updateCraft(userCraft);
	    }

            //we read the current data, as it may have changed by the user in the meantime
            craftSavedData = await GM.getValue(userCraft.label, '{}');
            craftParsedData = JSON.parse(craftSavedData);
            userCraft = craftParsedData;

	    //only continue if the job is in a idle state and if we need something to craft, otherwise we save a cargo hold request
	    if(userCraft.state === 'Idle' && userCraft.item && userCraft.coordinates && userCraft.amount > 0 && userCraft.crew > 0)
	    {

		if(starbase && targetX == userCraft.coordinates.split(',')[0].trim() && targetY == userCraft.coordinates.split(',')[1].trim()) {
			// we use the existing data from the craft completion, obviously the starbase wasn't changed by the user
		} else {
			// fresh job with no previous completion or starbase was changed
			targetX = userCraft.coordinates.split(',')[0].trim();
			targetY = userCraft.coordinates.split(',')[1].trim();
			starbase = await getStarbaseFromCoords(targetX, targetY, true);
			craftTime = await getStarbaseTime(starbase, 'Craft');
			upgradeTime = await getStarbaseTime(starbase, 'Upgrade');
			starbasePlayer = await getStarbasePlayer(userProfileAcct, starbase.publicKey);
			starbasePlayer = starbasePlayer ? starbasePlayer.publicKey : await execRegisterStarbasePlayer('Craft', userCraft.coordinates);
		}

                let starbasePlayerInfo = await sageProgram.account.starbasePlayer.fetch(starbasePlayer);
                let availableCrew = starbasePlayerInfo.totalCrew - starbasePlayerInfo.busyCrew.toNumber();
                let starbasePlayerCargoHoldsAndTokens = await getStarbasePlayerCargoHolds(starbasePlayer);

                let targetRecipe = getTargetRecipe(starbasePlayerCargoHoldsAndTokens, userCraft, Number(userCraft.amount), (userCraft.special ? userCraft.special : ''));

                if (!enableAssistant) return;

                cLog(3, FleetTimeStamp(userCraft.label), 'targetRecipe: ', targetRecipe);
                cLog(3, FleetTimeStamp(userCraft.label), 'starbasePlayerInfo: ', starbasePlayerInfo);
                cLog(3, FleetTimeStamp(userCraft.label), 'availableCrew: ', availableCrew);
                cLog(3, FleetTimeStamp(userCraft.label), 'userCraft: ', userCraft);
                cLog(3, FleetTimeStamp(userCraft.label), 'userCraft.crew: ', userCraft.crew);
                cLog(3, FleetTimeStamp(userCraft.label), 'userCraft.state: ', userCraft.state);

                { const g = sisCrewMissao.get(String(userCraft.coordinates || '').replace(/\s/g, '')); if (g && g.ate > Date.now()) availableCrew -= g.qtd; } // [1.5.5] tripulação da missão Crafting
                if (availableCrew >= userCraft.crew && targetRecipe && targetRecipe.amountCraftable > 0 && userCraft.state === 'Idle') {
                    let craftAmount = Math.min(targetRecipe.craftAmount, targetRecipe.amountCraftable);
                    cLog(1,`${FleetTimeStamp(userCraft.label)} Starting craft at [${targetX}, ${targetY}] for ${craftAmount} ${targetRecipe.craftRecipe.name}`);
                    //updateFleetState(userCraft, 'Craft Starting');
                    let activityType = craftRecipes.some(item => item.name === targetRecipe.craftRecipe.name) ? 'Crafting' : 'Upgrading';

                    //Enough Atlas available for the craft?
                    let enoughAtlas = true;
                    let craftThresholdReached = true;
                    if(activityType == 'Crafting') {
                        const atlasNeeded = Number((craftAmount * targetRecipe.craftRecipe.feeAmount).toFixed(10));
                        const atlasParsedBalance = await solanaReadConnection.getParsedTokenAccountsByOwner(userPublicKey,{ mint: new solanaWeb3.PublicKey('ATLASXmbPQxBUYbxPsV97usA3fPQYEqzQBUHgiFCUsXx')} );
                        const atlasBalance = (atlasParsedBalance.value[0] ? atlasParsedBalance.value[0].account.data.parsed.info.tokenAmount.uiAmount : 0);
                        cLog(3, FleetTimeStamp(userCraft.label), 'atlas needed: ', atlasNeeded, ', atlas available: ', atlasBalance);
                        if(atlasBalance < atlasNeeded) {
                    	    enoughAtlas = false;
                        }
                        if(parseInt(userCraft.belowAmount)) {
				let craftRecipe = craftRecipes.find(item => item.name === userCraft.item);

				let starbasePlayerCargoHold = getStarbasePlayerCargoMaxItem(starbasePlayerCargoHoldsAndTokens, craftRecipe.publicKey.toString());
				let amountAvailable = 0;
				if (starbasePlayerCargoHold && starbasePlayerCargoHold.cargoHoldTokens) {
					let cargoHoldToken = starbasePlayerCargoHold.cargoHoldTokens.find(item => item.mint === craftRecipe.output.mint.toString());
					amountAvailable = (cargoHoldToken && cargoHoldToken.amount) ? cargoHoldToken.amount : 0;
				}
				if(amountAvailable >= parseInt(userCraft.belowAmount)) {
					craftThresholdReached = false;
					cLog(3, FleetTimeStamp(userCraft.label), 'crafting paused: starbase amount available ', amountAvailable, ', craft if stock is below:', parseInt(userCraft.belowAmount));
				}
			}
                    }
                    if(activityType == 'Upgrading' && starbase.account.level >= 5) {
			updateFleetState(userCraft, 'Starbase has max level');
                    } else if(!enoughAtlas) {
                        updateFleetState(userCraft, 'Not enough Atlas: ' + targetRecipe.craftRecipe.name + (userCraft.item!=targetRecipe.craftRecipe.name?' ('+userCraft.item+')':''));
                    } else if(!craftThresholdReached) {
                        updateFleetState(userCraft, 'Stock not less than ' + parseInt(userCraft.belowAmount) + ': ' + userCraft.item);
                    } else {
                        let activityInfo = activityType == 'Crafting' ? "Starting: " + targetRecipe.craftRecipe.name + (userCraft.item!=targetRecipe.craftRecipe.name?' ('+userCraft.item+')':'') : 'Upgrade Starting';
                        updateFleetState(userCraft, activityInfo);
                        let result = await execStartCrafting(starbase, starbasePlayer, starbasePlayerCargoHoldsAndTokens, targetRecipe.craftRecipe, craftAmount, userCraft);
                        if (!userCraft.state.includes('ERROR')) {
                            activityInfo = activityType == 'Crafting' ? "\u2692 " + targetRecipe.craftRecipe.name + (userCraft.item!=targetRecipe.craftRecipe.name?' ('+userCraft.item+')':'') : 'Upgrading';
                            let craftDuration = (targetRecipe.craftRecipe.duration * craftAmount) / userCraft.crew;
                            let calcEndTime = TimeToStr(new Date(Date.now() + craftDuration * 1000));
                            let upgradeTimeStr = upgradeTime.resRemaining > 0 ? calcEndTime : 'Paused';
                            let craftTimeStr = craftTime.resRemaining > 0 ? calcEndTime : TimeToStr(new Date(Date.now() + ((craftDuration * 1000) / EMPTY_CRAFTING_SPEED_PER_TIER[starbase.account.level])));
                            //let craftTimeStr = TimeToStr(new Date(Date.now() + ((craftDuration * 1000) / EMPTY_CRAFTING_SPEED_PER_TIER[starbase.account.level])));
                            let activityTimeStr = activityType == 'Crafting' ? craftTimeStr : upgradeTimeStr;
                            //updateFleetState(userCraft, activityType + ' [' + activityTimeStr + ']');
                            updateFleetState(userCraft, activityInfo + ' [' + activityTimeStr + ']');
                            userCraft.craftingId = result.craftingId;
                            userCraft.craftingCoords = userCraft.coordinates;
                            userCraft.feeAtlas = result.feeAtlas;
                            if (userCraft.sisUnica) { userCraft.item = ''; userCraft.amount = 0; userCraft.sisUnica = false; const g = JSON.parse(await GM.getValue(userCraft.label, '{}')); g.item = ''; g.amount = 0; g.sisUnica = false; await GM.setValue(userCraft.label, JSON.stringify(g)); }
                            userCraft.errorCount = 0;
                            await updateCraft(userCraft);
                            //await GM.setValue(userCraft.label, JSON.stringify(userCraft));
		        }
			else { localTimeout = await craftTimeoutAfterError(userCraft); }
                    }
                } else if (userCraft.state === 'Idle') {
                    //updateFleetState(userCraft, 'Waiting for crew/material');
                    let materialStr = '';
                    if(targetRecipe === null) {
                        materialStr = ': ' + userCraft.item;
                    } else {
                        materialStr = ': ' + targetRecipe.craftRecipe.name + (userCraft.item != targetRecipe.craftRecipe.name ? ' (' + userCraft.item + ')' : '' );
                    }
                    if(availableCrew < userCraft.crew && ((targetRecipe && targetRecipe.amountCraftable <= 0) || (!targetRecipe)) ) {
                        updateFleetState(userCraft, 'Waiting for crew/material' + materialStr);
                    }
                    else if(availableCrew < userCraft.crew) {
                        updateFleetState(userCraft, 'Waiting for crew' + materialStr);
                    }
                    else {
                        updateFleetState(userCraft, 'Waiting for material' + materialStr);
                    }
                    await updateCraft(userCraft);
                }
	    }
        }
        catch(error) {
            cLog(1,`${FleetTimeStamp(userCraft.label)} Uncaught crafting error`, error);
        }
        setTimeout(() => { startCraft(userCraft); }, localTimeout);
    }

	async function startAssistant() {
		for (let i=0, n=userFleets.length; i < n; i++) {
			//Initialize iteration counter
			userFleets[i].iterCnt = 0;

			let fleetSavedData = await GM.getValue(userFleets[i].publicKey.toString(), '{}');
			let fleetParsedData = JSON.parse(fleetSavedData);

			if(fleetParsedData.assignment) updateFleetState(userFleets[i], 'Starting');

			//Stagger fleet starts by 1000ms to avoid overloading the RPC
			setTimeout(() => { startFleet(i);	}, 1500 * (i + 1));
		}

        for (let i=1; i < globalSettings.craftingJobs+1; i++) {
            let craftSavedData = await GM.getValue('craft' + i, '{}');
            let craftParsedData = JSON.parse(craftSavedData);
            // if we just extended the number of crafting jobs, properly initialize the data
            if(!craftParsedData.label && !craftParsedData.state) {
		craftParsedData.label = 'craft' + i;
		craftParsedData.state = 'Idle';
		await GM.setValue('craft'+i, JSON.stringify(craftParsedData));
            }
            //if (craftParsedData.item && craftParsedData.coordinates) startCraft(craftParsedData);

            //Stagger craft starts by 2s to avoid overloading the RPC
            //also we start crafting jobs always, so they are ready to use when we need them (without reloading SLYA)
            //if (craftParsedData.item && craftParsedData.coordinates) {
		    updateFleetState(craftParsedData, craftParsedData.state);
		    setTimeout(() => { startCraft(craftParsedData); }, 2000 * i);
	    //}
        }

		setTimeout(fleetHealthCheck, 5000);
		setTimeout(tokenCheck, 1000);
	}

	async function tokenCheck() {
		if(!enableAssistant) return;

		if (!sisCarteiraOk()) { sisCarteiraAusente(); if (enableAssistant) setTimeout(tokenCheck, 10000); return; }
		if((tokenCheckCounter%10)==0) { // check token balance every 100 seconds
			cLog(1, 'Checking SOL and Atlas balance');
			const solBalance = await solanaReadConnection.getBalance(userPublicKey);
			const atlasBalance = await solanaReadConnection.getParsedTokenAccountsByOwner(userPublicKey,{ mint: new solanaWeb3.PublicKey('ATLASXmbPQxBUYbxPsV97usA3fPQYEqzQBUHgiFCUsXx')} );
			// [1.4.4] o proxy de RPC devolve undefined quando a leitura falha sem ser de conexão: não derruba o ciclo
			if (solBalance === undefined || !atlasBalance || !atlasBalance.value) { tokenCheckCounter = 9; if (enableAssistant) setTimeout(tokenCheck, 10000); return; }
			setTxt(document.getElementById('assist-modal-balance'), 'SOL:'+((solBalance/1000000000).toFixed(3))+' Atlas:'+(atlasBalance.value[0] ? parseInt(atlasBalance.value[0].account.data.parsed.info.tokenAmount.uiAmount) : 0));
			sisSaldo = { sol: solBalance / 1000000000, atlas: atlasBalance.value[0] ? atlasBalance.value[0].account.data.parsed.info.tokenAmount.uiAmount : 0 }; sisSaldoEm = Date.now();
		}
		tokenCheckCounter++;

		if(enableAssistant) setTimeout(tokenCheck, 10000);
	}

	async function fleetHealthCheck() {
		if (!enableAssistant) return;
		if (!sisCarteiraOk()) { sisCarteiraAusente(); setTimeout(fleetHealthCheck, 15000); return; }

		let fleetStallCount = 0;
		for (let i=0, n=userFleets.length; i < n; i++) {
			const fleet = userFleets[i];

			const fleetSavedData = await GM.getValue(fleet.publicKey.toString(), '{}');
			const fleetParsedData = JSON.parse(fleetSavedData);

			//Skip unassigned fleets
			if(!fleetParsedData.assignment) continue; //Ignore unassigned fleets
			if (fleet.state.includes('ERROR')) continue; //Ignore error fleets

			const foo = Math.max(
				fleet.lastOp,
				fleet.scanEnd ? fleet.scanEnd : 0,
				fleet.moveEnd ? fleet.moveEnd : 0,
			);

			if(fleet.lastOp) {
				if(Date.now() - foo > 600000) {
					cLog(3,`${FleetTimeStamp(userFleets[i].label)} Unresponsive`,
						foo ? TimeToStr(new Date(foo)) : 'null',
						fleet.lastOp ? TimeToStr(new Date(fleet.lastOp)) : 'null',
						fleet.scanEnd ? TimeToStr(new Date(fleet.scanEnd)) : 'null',
						fleet.moveEnd ? TimeToStr(new Date(fleet.moveEnd)) : 'null',
					);

					fleet.fontColor = 'tomato';
					updateAssistStatus(fleet);
					fleetStallCount++;
				}
			}
		}

		//Auto-reload when too many fleets fail
		if(globalSettings.reloadPageOnFailedFleets && fleetStallCount && globalSettings.reloadPageOnFailedFleets <= fleetStallCount) {
			cLog(1, `ASSISTANT: ${fleetStallCount} fleets have stalled - reloading ...`);
			location.reload();
			return;
		}

		if(enableAssistant)	setTimeout(fleetHealthCheck, 10000);
	}

    let globalWaitForStopSequence = false;
    async function toggleAssistant(newState=null) {
        let autoSpanRef = document.querySelector('#autoScanBtn > span');
	// Race condition prevented: user presses STOP, waitForSequence is done, enableAssistant is false, but SLYA still executes dock->load/unload->undock sequences, then an error occurs which hits the error threshold and SLYA switches to ERROR state, which would call toogleAssistant. But enableAssistant is false in this moment, which would enable SLYA again. So we need to make sure that an error state always stops SLYA.
	// Also we set enableAssistant to FALSE before the loop, so SLYA stops immediately (which is better/faster when you have a lot of fleets, because otherwise: as long as at least 1 fleet is in a special state, enableAssistant would never be switched to FALSE).
	// To prevent that the user enables SLYA again while SLYA waits for the stop sequence, we make an additional global variable.
	// And we ignore the start/stop button as long as the init sequence isn't done yet.
	// Added Docking and Undocking to the wait sequence, so a fleet finishes the docking->load/unload->undock loop first (and doesn't get interrupted in between).
        if((!initComplete) || ((!newState) && globalWaitForStopSequence)) return;
        if (!enableAssistant && !newState && !sisPodeIniciar()) { sisInicioNegado(); return; }
        if(autoSpanRef.textContent=='Reload') {
		window.location.reload();
		return;
	}
        sisMarcarIntencao(enableAssistant === true || newState ? (newState ? 'erro' : 'parar') : 'iniciar');
        if (enableAssistant === true || newState) {
            globalWaitForStopSequence = true;
            let waitForSequence = true; const sisT0Parada = Date.now();
            setTxt(autoSpanRef, 'Wait...');
            enableAssistant = false;
            while (waitForSequence) {
                let fleetBusy = false;
                for (let i=0, n=userFleets.length; i < n; i++) {
                    //if (['Mine Starting','Mining Stop','Unloading','Loading','Refueling','Craft Completing','Upgrade Completing'].includes(userFleets[i].state)) fleetBusy = true;
                    if(['Mine Starting','Mining Stop','Unloading','Loading','Docking','Undocking','Refueling'].some(v => userFleets[i].state.startsWith(v))) fleetBusy = true;
                }
		for (let i=1; i < globalSettings.craftingJobs+1; i++) {
			let craftSavedData = await GM.getValue('craft' + i, '{}');
			let craftParsedData = JSON.parse(craftSavedData);
			if(['Starting:','Upgrade Starting', 'Completing:', 'Upgrade Completing'].some(v => craftParsedData.state.startsWith(v))) fleetBusy = true;
		}
                if (!fleetBusy) waitForSequence = false;
                else if (Date.now() - sisT0Parada > 180000) { waitForSequence = false; sisParadaLonga(); }
                await wait(5000);
            }
            globalWaitForStopSequence = false;
            setTxt(autoSpanRef, newState ? newState : 'Reload');
        } else {
            enableAssistant = true;
            await startAssistant();
            setTxt(autoSpanRef, 'Stop');
            for (let i=0, n=userFleets.length; i < n; i++) {
                let fleetAcctInfo = await getAccountInfo(userFleets[i].label, 'full fleet info', userFleets[i].publicKey);
                let [fleetState, extra] = getFleetState(fleetAcctInfo, userFleets[i]);
                let fleetCoords = fleetState == 'Idle' && extra ? extra : [];
                userFleets[i].startingCoords = fleetCoords;
                userFleets[i].state = fleetState;
            }
        }
    }




    async function getAllStarbasesForFaction(faction) {
        return new Promise(async resolve => {

			cLog(1, 'Reading faction starbases');
			let starbases = await sageProgram.account.starbase.all([
				{
					memcmp: {
						offset: 201,
						bytes: [faction]
					}
				}
			]);
			cLog(1, starbases.length, 'read');

			cLog(1, 'Reading all planets');
			let planets = await sageProgram.account.planet.all([]);
			cLog(1, planets.length, 'read');

			// first we group all planets of the same sector:
			let planetSectors = [];
			planets.forEach((planet) => {
				let label = (new TextDecoder("utf-8").decode(new Uint8Array(planet.account.name))).replace(/\0/g, '');
				let x=planet.account.sector[0].toNumber();
				let y=planet.account.sector[1].toNumber();
				if(typeof planetSectors[x] == 'undefined') {
					planetSectors[x] = [];
				}
				if(typeof planetSectors[x][y] == 'undefined') {
					planetSectors[x][y] = [];
				}
				planetSectors[x][y].push(planet);
			});

			// now we find out the system names by looking at the planet names
			let validMainTargets = [];
			let validMRZTargets = [];
			starbases.forEach((starbase) => {
				let x=starbase.account.sector[0].toNumber();
				let y=starbase.account.sector[1].toNumber();
				let planets=planetSectors[x][y];
				let name = (new TextDecoder("utf-8").decode(new Uint8Array(planets[0].account.name))).replace(/\0/g, '').split('-');
				let systemName = name[0] + '-' + name[1];
				if(name[0] == 'MRZ') {
					validMRZTargets.push({x, y, name: systemName});
				} else {
					validMainTargets.push({x, y, name: systemName});
				}
				planetData.push({coords: [x,y], lastUpdated: Date.now(), planets: planets});
				starbaseData.push({coords: [x,y], lastUpdated: Date.now(), starbase: starbase});
			});
			validMainTargets.sort((a, b) => { if (a.name < b.name) { return -1; } if (a.name > b.name) { return 1; } return 0; });
			//validMainTargets[0].name = (validMainTargets[0].name.includes('-1') ? validMainTargets[0].name.replace('-1','-CSS') : validMainTargets[0].name);
			validMRZTargets.sort((a, b) => { if (a.name < b.name) { return -1; } if (a.name > b.name) { return 1; } return 0; });
			validTargets = validMainTargets.concat(validMRZTargets);

			console.log('validTargets:',validTargets);

            resolve();
        });
    }


	function initUser() {
		return new Promise(async resolve => {

			if(globalSettings.mySecretKey) {
				// [Sis-Ágora] o erro de JSON.parse traz trechos do texto lido; nunca deixar a mensagem original chegar ao console/log
				try {
					let mySecret = JSON.parse(globalSettings.mySecretKey);
					customKeypair = solanaWeb3.Keypair.fromSecretKey(new Uint8Array(mySecret));
				} catch (e) {
					throw new Error('My secret key inválida: use o formato [12, 34, ...] exportado da carteira');
				}
				cLog(1, "SLYA uses custom key with public address:", customKeypair.publicKey.toString());
				userPublicKey = customKeypair.publicKey;
			} else if (typeof solflare === 'undefined') {
				let walletConn = null; try { walletConn = phantom && phantom.solana ? await phantom.solana.connect() : await solana.connect(); } catch (e) { cLog(1, 'Carteira recusou a conexão:', e && e.message); }
				userPublicKey = (walletConn && walletConn.publicKey) || null;
			} else {
				try { await solflare.connect(); } catch (e) { cLog(1, 'Carteira recusou a conexão:', e && e.message); }
				userPublicKey = solflare.publicKey || null;
			}
			// [1.4.4] a extensão pode responder sem chave (bloqueada / aprovação pendente): tenta de novo antes de desistir
			for (let t = 0; t < 6 && !sisCarteiraOk(); t++) {
				cLog(1, 'Carteira sem chave pública; aguardando a extensão (' + (t + 1) + '/6)');
				await wait(5000);
				try { if (typeof solflare !== 'undefined' && !solflare.isConnected) await solflare.connect(); } catch (e) { }
			}
			if (!sisCarteiraOk()) {
				sisCarteiraAusente();
				throw new Error('Carteira sem chave pública: desbloqueie/conecte a extensão (Solflare/Phantom) ou configure a hot wallet no painel Sis-Ágora e recarregue a página.');
			}

            if (globalSettings.saveProfile && globalSettings.savedProfile && globalSettings.savedProfile.length > 0) {
                cLog(1, 'Skipping User Profile query, using saved profile');
                userProfileAcct = new solanaWeb3.PublicKey(globalSettings.savedProfile[0]);
                userProfileKeyIdx = globalSettings.savedProfile[1];
                pointsProfileKeyIdx = globalSettings.savedProfile[2];
            } else {
                cLog(1, 'Getting User Profiles (this takes a while)');
                let userProfiles = await solanaReadConnection.getProgramAccounts(profileProgramPK);
                let foundProf = [];

                cLog(2, 'initUser: userProfiles[0]', userProfiles[0]);
                for (let userProf of userProfiles) {
                    let userProfData = userProf.account.data.subarray(30);
                    let iter = 0;
                    while (userProfData.length >= 80) {
                        let currProf = userProfData.subarray(0, 80);
                        let profDecoded = profileProgram.coder.types.decode('ProfileKey', currProf);
                        if (profDecoded.key.toString() === userPublicKey.toString()) {
                            let [playerNameAcct] = await solanaReadConnection.getProgramAccounts(
                                profileProgramPK,
                                {
                                    filters: [
                                        {
                                            memcmp: {
                                                offset: 9,
                                                bytes: userProf.pubkey.toString(),
                                            },
                                        },
                                    ],
                                }
                            );
                            let playerName = playerNameAcct ? new TextDecoder().decode(playerNameAcct.account.data.subarray(42)) : '';

                            let permissionType, sagePermission, pointsPermission, pointsStorePermission;
                            switch(profDecoded.scope.toString()) {
                                case sageProgramPK.toString():
                                    permissionType = 'sage';
                                    break;
                                case pointsProgramId.toString():
                                    permissionType = 'points';
                                    break;
                                case pointsStoreProgramId.toString():
                                    permissionType = 'points_store';
                                    break;
                                case profileProgramPK.toString():
                                    permissionType = 'default';
                                    break;
                            }

                            let profAdded = foundProf.findIndex(item => item.profile === userProf.pubkey.toString());
                            profAdded > -1 ? foundProf[profAdded][permissionType] = iter : foundProf.push({profile: userProf.pubkey.toString(), name: playerName, [permissionType]: iter})
                            //foundProf.push({profile: userProf.pubkey.toString(), name: playerName, scope: permissionType, idx: iter})
                        }
                        userProfData = userProfData.subarray(80);
                        //iter > 0 && foundProf.push({profile: userProf, key: profDecoded, idx: iter});
                        iter += 1;
                    }
                }

                //Wait for user to select a profile if more than 1 is available
                let userProfile = foundProf.length > 1 ? await assistProfileToggle(foundProf) : foundProf[0];
                userProfileAcct = new solanaWeb3.PublicKey(userProfile.profile);
                userProfileKeyIdx = userProfile.sage || 0;
                pointsProfileKeyIdx = userProfile.points || 0;
                if (globalSettings.saveProfile) {
                    globalSettings.savedProfile = [userProfileAcct.toString(), userProfileKeyIdx, pointsProfileKeyIdx];

                    await GM.setValue(settingsGmKey, JSON.stringify(globalSettings));
                }
            }

			let profileFactionProgram = new BrowserAnchor.anchor.Program(profileFactionIDL, profileFactionProgramPK, anchorProvider);
			[userProfileFactionAcct] = await profileFactionProgram.account.profileFactionAccount.all([
					{
							memcmp: {
									offset: 9,
									bytes: userProfileAcct.toBase58(),
							},
					},
			]);

            await getAllStarbasesForFaction(userProfileFactionAcct.account.faction);

            let redemptionConfigs = await pointsStoreProgram.account.redemptionConfig.all();
            userRedemptionConfigAcct = redemptionConfigs.find(item => item.account.faction === userProfileFactionAcct.account.faction).publicKey;

            buildXpAccounts(dataRunningXpCategory, userXpAccounts, "userDataRunningXpAccounts")
            buildXpAccounts(councilRankXpCategory, userXpAccounts, "userCouncilRankXpAccounts")
            buildXpAccounts(pilotingXpCategory, userXpAccounts, "userPilotingXpAccounts")
            buildXpAccounts(miningXpCategory, userXpAccounts, "userMiningXpAccounts")
            buildXpAccounts(craftingXpCategory, userXpAccounts, "userCraftingXpAccounts")
            buildXpAccounts(LPCategory, userXpAccounts, "userLPAccounts")

			let userOwnedFleetAccts = await sageProgram.account.fleet.all([
					{
							memcmp: {
									offset: 41,
									bytes: userProfileAcct.toBase58(),
							},
					},
			]);
			cLog(1, 'initUser: userOwnedFleetAccts', userOwnedFleetAccts);

            let userBorrowedFleetAccts = await sageProgram.account.fleet.all([
                {
                    memcmp: {
                        offset: 105,
                        bytes: userProfileAcct.toBase58(),
                    },
                },
            ]);
            cLog(1, 'initUser: userBorrowedFleetAccts', userBorrowedFleetAccts);

            let userFleetAccts = userOwnedFleetAccts.concat(userBorrowedFleetAccts);
            cLog(1, 'initUser: userFleetAccts', userFleetAccts);

			let excludeFleets=[];
			if (globalSettings.excludeFleets && globalSettings.excludeFleets.length > 0) {
				excludeFleets=globalSettings.excludeFleets.trim().replaceAll("\r","").split("\n");
			}
			cLog(1, 'initUser: excludeFleets ',excludeFleets);

			for (let fleet of userFleetAccts) {
				let fleetLabel = (new TextDecoder("utf-8").decode(new Uint8Array(fleet.account.fleetLabel))).replace(/\0/g, '');

				if(excludeFleets.includes(fleetLabel)) continue;

				let fleetSavedData = await GM.getValue(fleet.publicKey.toString(), '{}');
				let fleetParsedData = JSON.parse(fleetSavedData);
				let fleetDest = fleetParsedData && fleetParsedData.dest ? fleetParsedData.dest : '';
				let fleetScanBlock = fleetParsedData && fleetParsedData.scanBlock ? fleetParsedData.scanBlock : [];
				let fleetScanMin = fleetParsedData && fleetParsedData.scanMin ? fleetParsedData.scanMin : 15;
				let fleetScanMin2 = fleetParsedData && fleetParsedData.scanMin2 ? fleetParsedData.scanMin2 : 0;
				let fleetScanMin3 = fleetParsedData && fleetParsedData.scanMin3 ? fleetParsedData.scanMin3 : 0;
				let fleetScanPattern = fleetParsedData && fleetParsedData.scanPattern ? fleetParsedData.scanPattern : '';
				let fleetScanPatternLength = fleetParsedData && fleetParsedData.scanPatternLength ? fleetParsedData.scanPatternLength : 0;
				let fleetSearchDist = fleetParsedData && fleetParsedData.scanSearchDist ? fleetParsedData.scanSearchDist : 15;
				let fleetClusterFactor = fleetParsedData && fleetParsedData.scanClusterFactor ? fleetParsedData.scanClusterFactor : 50;
				let fleetScanNeighborhoodMinGood = fleetParsedData && fleetParsedData.scanNeighborhoodMinGood ? fleetParsedData.scanNeighborhoodMinGood : 3;
				let fleetScanCheckWhileCooldownLeft = fleetParsedData && fleetParsedData.scanCheckWhileCooldownLeft ? fleetParsedData.scanCheckWhileCooldownLeft : 0;
				//let fleetScanCheckWhileCooldownLeftProb = fleetParsedData && fleetParsedData.scanCheckWhileCooldownLeftProb ? fleetParsedData.scanCheckWhileCooldownLeftProb : 0;
				let fleetScanBypassPercent = fleetParsedData && fleetParsedData.scanBypassPercent ? fleetParsedData.scanBypassPercent : 4;
				let fleetScanHomeAtPercent = fleetParsedData && fleetParsedData.scanHomeAtPercent ? fleetParsedData.scanHomeAtPercent : 0;
				//let fleetScanMove = fleetParsedData && fleetParsedData.scanMove == 'false' || false ? false : true;
				let fleetScanMove = fleetParsedData && fleetParsedData.scanMove;
				let fleetMineResource = fleetParsedData && fleetParsedData.mineResource ? fleetParsedData.mineResource : '';
				let fleetStarbase = fleetParsedData && fleetParsedData.starbase ? fleetParsedData.starbase : '';
				let fleetMoveType = fleetParsedData && fleetParsedData.moveType ? fleetParsedData.moveType : 'warp';
				let fleetMoveTarget = fleetParsedData && fleetParsedData.moveTarget ? fleetParsedData.moveTarget : '';

				let fleetScanEnd = fleetParsedData && fleetParsedData.scanEnd ? fleetParsedData.scanEnd : 0;
				//double check for a wrongly set time and correct it if needed:
				if(fleetScanEnd > Date.now() + fleet.account.stats.miscStats.scanCoolDown * 1000) { fleetScanEnd = Date.now() + fleet.account.stats.miscStats.scanCoolDown * 1000; }

				const [fleetRepairKitToken] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
					[
						fleet.account.cargoHold.toBuffer(),
						tokenProgramPK.toBuffer(),
						new solanaWeb3.PublicKey(toolItem.token).toBuffer()
					],
					programPK
				);
				const [fleetSduToken] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
					[
						fleet.account.cargoHold.toBuffer(),
						tokenProgramPK.toBuffer(),
						new solanaWeb3.PublicKey(sduItem.token).toBuffer()
					],
					programPK
				);
				const [fleetFuelToken] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
					[
						fleet.account.fuelTank.toBuffer(),
						tokenProgramPK.toBuffer(),
						new solanaWeb3.PublicKey(fuelItem.token).toBuffer()
					],
					programPK
				);
                const [fleetFoodToken] = await BrowserAnchor.anchor.web3.PublicKey.findProgramAddressSync(
					[
						fleet.account.cargoHold.toBuffer(),
						tokenProgramPK.toBuffer(),
						new solanaWeb3.PublicKey(foodItem.token).toBuffer()
					],
					programPK
				);

				let fleetCurrentCargo = await solanaReadConnection.getParsedTokenAccountsByOwner(fleet.account.cargoHold, {programId: tokenProgramPK});
				let currentFoodCnt = fleetCurrentCargo.value.find(item => item.pubkey.toString() === fleetFoodToken.toString());
				let currentSduCnt = fleetCurrentCargo.value.find(item => item.pubkey.toString() === fleetSduToken.toString());
				let fleetCurrentFuel = await solanaReadConnection.getParsedTokenAccountsByOwner(fleet.account.fuelTank, {programId: tokenProgramPK});
				let currentFuelCnt = fleetCurrentFuel.value.find(item => item.pubkey.toString() === fleetFuelToken.toString());
				let fleetAcctInfo = await getAccountInfo(fleetLabel, 'Full Account Info', fleet.publicKey);
				let [fleetState, extra] = getFleetState(fleetAcctInfo);
				let fleetCoords = fleetState == 'Idle' && extra ? extra : [];
				let fleetScanBlockIdx = 0;
				userFleets.push({
					publicKey: fleet.publicKey,
					label: fleetLabel,
					emprestada: !!(fleet.account.ownerProfile && fleet.account.ownerProfile.toString() !== userProfileAcct.toString()),
					state: fleetState,
					exitWarpSubwarpPending: 0,
					exitSubwarpWillBurnFuel: 0,
					moveTarget: fleetMoveTarget,
					startingCoords: fleetCoords,
					cargoHold: fleet.account.cargoHold,
					fuelTank: fleet.account.fuelTank,
					ammoBank: fleet.account.ammoBank,
					repairKitToken: fleetRepairKitToken,
					sduToken: fleetSduToken,
					fuelToken: fleetFuelToken,
                    			foodToken: fleetFoodToken,
					warpFuelConsumptionRate: fleet.account.stats.movementStats.warpFuelConsumptionRate,
					warpSpeed: fleet.account.stats.movementStats.warpSpeed,
					maxWarpDistance: fleet.account.stats.movementStats.maxWarpDistance,
					subwarpFuelConsumptionRate: fleet.account.stats.movementStats.subwarpFuelConsumptionRate,
					subwarpSpeed: fleet.account.stats.movementStats.subwarpSpeed,
					cargoCapacity: fleet.account.stats.cargoStats.cargoCapacity,
					fuelCapacity: fleet.account.stats.cargoStats.fuelCapacity,
					ammoCapacity: fleet.account.stats.cargoStats.ammoCapacity,
					scanCost: fleet.account.stats.miscStats.scanCost,
                    			sduPerScan: fleet.account.stats.miscStats.sduPerScan,
					scanCooldown: fleet.account.stats.miscStats.scanCoolDown,
					requiredCrew: fleet.account.stats.miscStats.requiredCrew,
					passengerCapacity: fleet.account.stats.miscStats.passengerCapacity,
					crewCount: fleet.account.stats.miscStats.crewCount,
					warpCooldown: fleet.account.stats.movementStats.warpCoolDown,
					miningRate: fleet.account.stats.cargoStats.miningRate,
					foodConsumptionRate: fleet.account.stats.cargoStats.foodConsumptionRate,
					ammoConsumptionRate: fleet.account.stats.cargoStats.ammoConsumptionRate,
					planetExitFuelAmount: fleet.account.stats.movementStats.planetExitFuelAmount,
					destCoord: fleetDest,
					starbaseCoord: fleetStarbase,
					scanBlock: fleetScanBlock,
					scanBlockIdx: fleetScanBlockIdx,
					scanEnd: fleetScanEnd,
					scanSkipCnt: 0,
					scanStrikes: 0,
					scanMin: fleetScanMin,
					scanMin2: fleetScanMin2,
					scanMin3: fleetScanMin3,
					scanPattern: fleetScanPattern,
					scanPatternLength: fleetScanPatternLength,
					scanSearchDist: fleetSearchDist,
					scanClusterFactor: fleetClusterFactor,
					scanNeighborhoodMinGood: fleetScanNeighborhoodMinGood,
					scanCheckWhileCooldownLeft: fleetScanCheckWhileCooldownLeft,
					//scanCheckWhileCooldownLeftProb: fleetScanCheckWhileCooldownLeftProb,
					scanBypassPercent: fleetScanBypassPercent,
					scanHomeAtPercent: fleetScanHomeAtPercent,
					scanMove: fleetScanMove,
					foodCnt: currentFoodCnt ? currentFoodCnt.account.data.parsed.info.tokenAmount.uiAmount : 0,
					sduCnt: currentSduCnt ? currentSduCnt.account.data.parsed.info.tokenAmount.uiAmount : 0,
					fuelCnt: currentFuelCnt ? currentFuelCnt.account.data.parsed.info.tokenAmount.uiAmount : 0,
					moveType: fleetMoveType,
					mineResource: fleetMineResource,
					minePlanet: null,
					fontColor: 'white',
				});
			}

			userFleets.sort(function (a, b) { return a.label.toUpperCase().localeCompare(b.label.toUpperCase()); });

			initComplete = true;
			sisAtualizarUI();
			if(globalSettings.autoStartScript) {
				sisPainelMostrar();
				toggleAssistant();
			}
			sisAoIniciar();
			resolve();
		});
	}

	function makeDraggable(elem) {
			const elemHeader = elem.querySelector('.assist-modal-header');
			let originalX, originalY, elementX, elementY;
			function dragElem(e)
			{
					elementY = elementY + e.clientY - originalY;
					elementX = elementX + e.clientX - originalX;
					originalY = e.clientY;
					originalX = e.clientX;
					elem.style.top = elementY + "px";
					elem.style.left = elementX + "px";
					elem.style.right = 'auto';
					elem.style.bottom = 'auto';
			}
			elemHeader.addEventListener('mousedown', function(e) {
					if (elem.classList.contains('sis-tela-cheia') || (e.target && e.target.closest && e.target.closest('button, input, select, label, a, .dropdown'))) return;
					e.preventDefault();
					originalX = e.clientX;
					originalY = e.clientY;
					elementX = elem.offsetLeft;
					elementY = elem.offsetTop;

					window.addEventListener('mousemove', dragElem, false);

					window.addEventListener('mouseup', () => {
							window.removeEventListener('mousemove', dragElem, false);
					}, false);

			}, false);
	}

	let observer = new MutationObserver(waitForLabs);
	function waitForLabs(mutations, observer){
		let elemTrigger = observer ? '#root > div:first-of-type > div:first-of-type > div > header > h1' : 'body';
		if(document.querySelectorAll(elemTrigger).length > 0 && !document.getElementById("assistContainer")) {
			document.getElementById("assistContainerIso") && document.getElementById("assistContainerIso").remove();
			observer && observer.disconnect();
			let assistCSS = document.createElement('style');
			const statusPanelOpacity = globalSettings.statusPanelOpacity / 100;
			let assistCSSString = `.assist-modal {display: none; position: fixed; z-index: 2; padding-top: 100px; left: 0; top: 0; width: 100%; height: 100%; overflow: auto; background-color: rgba(0,0,0,0.4); text-align:center; } .assist-modal-content {position: relative; display: inline-block; text-align:left; background-color: rgb(41, 41, 48); margin: auto; padding: 0; border: 1px solid #888; width: 785px; min-width: 450px; max-width: 95%; height: auto; min-height: 50px; max-height: 95%; overflow-y: auto; box-shadow: 0 4px 8px 0 rgba(0,0,0,0.2),0 6px 20px 0 rgba(0,0,0,0.19); -webkit-animation-name: animatetop; -webkit-animation-duration: 0.4s; animation-name: animatetop; animation-duration: 0.4s;} .assist-modal-save { font-size:100%; font-weight:bold; vertical-align:top; margin-left:0.5em; } #assist-modal-error {color: red; margin-left: 5px; margin-right: 5px; font-size: 16px; display:block; } .assist-modal-header-right {color: rgb(255, 190, 77); margin-left: auto !important; font-size: 20px;} .assist-btn {background-color: rgb(41, 41, 48); color: rgb(255, 190, 77); margin-left: 2px; margin-right: 2px;} .assist-btn:hover {background-color: rgba(255, 190, 77, 0.2);} .assist-modal-close { font-size:130%; line-height:80%; vertical-align:middle; } .assist-modal-close:hover, .assist-modal-close:focus {font-weight: bold; text-decoration: none; cursor: pointer;} .assist-modal-btn {color: rgb(255, 190, 77); padding: 5px 5px; margin-right: 5px; text-decoration: none; background-color: rgb(41, 41, 48); border: none; cursor: pointer;} .assist-modal-save:hover { background-color: rgba(255, 190, 77, 0.2); } .assist-modal-header {display: flex; position:sticky; z-index:1000; top:0; left:0; align-items: center; padding: 2px 16px; background-color: #544735; border-bottom: 2px solid rgb(255, 190, 77); color: rgb(255, 190, 77);} .assist-modal-body {padding: 2px 16px; font-size: 12px;} .assist-modal-body > table, .assist-modal-body table.main table {width: 100%;border-collapse: collapse;} .assist-modal-body th, .assist-modal-body td {padding:0 7px 0 0; line-height:130%;} #assistStatus {background-color: rgba(0,0,0,${statusPanelOpacity}); opacity: ${statusPanelOpacity}; backdrop-filter: blur(10px); position: absolute; top: 82px; left: 10px; z-index: 1;} #assistStarbaseStatus {background-color: rgba(0,0,0,${statusPanelOpacity}); opacity: ${statusPanelOpacity}; backdrop-filter: blur(10px); position: absolute; top: 80px; right: 20px; z-index: 1;} #assistCheck {background-color: rgba(0,0,0,0.75); backdrop-filter: blur(10px); position: absolute; margin: auto; left: 0; right: 0; top: 100px; width: 650px; min-width: 450px; max-width: 75%; z-index: 1;} .dropdown { position: absolute; display: none; margin-top: 25px; margin-left: 152px; background-color: rgb(41, 41, 48); min-width: 120px; box-shadow: 0 8px 16px 0 rgba(0, 0, 0, 0.2); z-index: 2; } .dropdown.show { display: block; } .assist-btn-alt { color: rgb(255, 190, 77); padding: 12px 16px; text-decoration: none; display: block; background-color: rgb(41, 41, 48); border: none; cursor: pointer; } .assist-btn-alt:hover { background-color: rgba(255, 190, 77, 0.2); } #checkresults { padding: 5px; margin-top: 20px; border: 1px solid grey; border-radius: 8px;} .dropdown button {width: 100%; text-align: left;} #assistModal table {border-collapse: collapse;} .assist-scan-row, .assist-scan2-row, .assist-mine-row, .assist-transport-row {background-color: rgba(255, 190, 77, 0.1); border-left: 1px solid white; border-right: 1px solid white; border-bottom: 1px solid white} .show-top-border {background-color: rgba(255, 190, 77, 0.1); border-left: 1px solid white; border-right: 1px solid white; border-top: 1px solid white;} #fleetTable { margin-top: 8px } #assistModal .assist-modal-content { width:auto } .transport-to-target select, .transport-to-starbase select { max-width: 11.5em; } #assistModal .assist-modal-body option { background-color:white } #assistModal .assist-modal-body > table { width: auto } #fleetTable tbody:nth-child(1) td { position:sticky; top:62px; background-color: #292930; padding: 5px 0 2px 0; } `;
			assistCSSString += ` #assistStats {background-color: rgba(0,0,0,${statusPanelOpacity}); opacity: ${statusPanelOpacity}; backdrop-filter: blur(10px); position: absolute; top: 80px; right: 20px; z-index: 1; } #assistStats table { border-collapse: collapse; border-spacing:1px; } #assistStats td, #assistStats th { padding:0 7px 0 0; }`; // statsadd
			assistCSSString += ` .tooltip { position: relative; display: inline-block; border-bottom: 1px dotted white; line-height: 105%; margin:0 2px 0 2px; } .tooltip .tooltiptext { visibility: hidden; font-weight:normal; border:1px solid #45382d; width: max-content; max-width:250px; box-shadow: 5px 5px 5px rgba(0,0,0,0.4); line-height:130%; background-color: #1f1914; color: #eee; text-align: left; border-radius: 6px; padding: 8px; position: absolute; z-index: 1; } .tooltip:hover .tooltiptext { visibility: visible; } `;
			assistCSSString += ` .sis-corpo h3 { margin: 14px 0 6px; font-size: 15px; } .sis-corpo small { display: block; opacity: .75; margin: 2px 0 6px; } .sis-linha { margin: 6px 0; } .sis-linha > label:first-child { display: inline-block; min-width: 120px; } .sis-chk { margin-left: 6px; } .sis-grade { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); gap: 8px; margin: 6px 0; } .sis-grade label { display: grid; gap: 4px; font-size: 13px; } .sis-botoes { margin: 10px 0; display: flex; gap: 6px; flex-wrap: wrap; } .sis-fraco { opacity: .75; font-size: 13px; margin: 4px 0; } .sis-erro { color: tomato; margin: 6px 0; } .sis-lista { max-height: 360px; overflow: auto; margin: 6px 0; padding-left: 18px; font-size: 13px; line-height: 1.5; } #sisModal input[type=text], #sisModal input[type=password], #sisModal input[type=number] { background: #1e1e24; color: #fff; border: 1px solid #555; padding: 3px 6px; }`;
			assistCSSString += sisCssTema(statusPanelOpacity); assistCSS.textContent = assistCSSString;

			// [Sis-Ágora] painel local do Sis-Bot (conexão, chave, tetos) e janela de aceite de configuração
			let sisModal = document.createElement('div');
			sisModal.classList.add('assist-modal');
			sisModal.id = 'sisModal';
			sisModal.style.display = 'none';
			let sisModalContent = document.createElement('div');
			sisModalContent.classList.add('assist-modal-content');
			setHtmlEstatico(sisModalContent, '<div class="assist-modal-header"><span>Sis-Bot · Sis-Ágora</span><div class="assist-modal-header-right"><button class="assist-modal-btn assist-modal-save" id="sisSalvar">Salvar</button> <span class="assist-modal-close">&#x2715;</span></div></div>'
				+ '<div class="assist-modal-body sis-corpo">'
				+ '<h3>Conexão com o dashboard</h3><div id="sisConexao"></div><div id="sisCfgInfo" class="sis-fraco"></div><div id="sisFila" class="sis-fraco"></div>'
				+ '<div class="sis-linha"><label>Token do perfil</label><input id="sisToken" type="password" size="44" autocomplete="off" spellcheck="false"> <label class="sis-chk"><input id="sisTokenClear" type="checkbox"> apagar token</label></div>'
				+ '<small>Gere o token em Dashboard › Sis-Bot › Conexão e alertas. Ele é de um perfil só: o script precisa estar nesse perfil.</small>'
				+ '<div class="sis-botoes"><button class="assist-modal-btn" id="sisSyncBtn">Sincronizar agora</button> <button class="assist-modal-btn" id="sisVerPendente" style="display:none">Ver configuração pendente</button></div>'
				+ '<h3>Este navegador no perfil</h3><div id="sisInstInfo"></div>'
				+ '<div class="sis-linha"><label>Nome deste navegador</label><input id="sisInstNome" size="30" maxlength="40" autocomplete="off" spellcheck="false"></div>'
				+ '<div class="sis-linha"><label class="sis-chk"><input id="sisEspelhar" type="checkbox"> Aplicar sozinho as configurações já aceitas no navegador primário (espelho)</label></div>'
				+ '<small>Com vários navegadores no mesmo perfil, só o primário executa (ordens, assistente, transações). Os outros ficam de reserva: medem os RPCs e o ping, espelham a configuração e assumem se o primário cair ou ficar bem pior. Quem escolhe é o dashboard (Sis-Bot › Conexão e alertas › Navegadores). Vários navegadores precisam de liberação de um Administrador.</small>'
				+ '<h3>Hot wallet</h3><div class="sis-linha"><label>Chave secreta</label><input id="mySecretKey" type="password" size="44" autocomplete="off" spellcheck="false"> <label class="sis-chk"><input id="mySecretKeyClear" type="checkbox"> apagar chave salva</label></div>'
				+ '<small>Formato [12, 34, ...]. Use uma carteira só para o Sis-Bot, adicionada ao seu perfil como chave do SAGE, com pouco SOL e ATLAS. A chave nunca sai deste navegador: não é exibida, exportada, mandada para o dashboard nem escrita no console. Sem chave, o Sis-Bot usa Solflare/Phantom.</small>'
				+ '<h3>RPCs da Solana</h3><div id="sisRpcLista" class="sis-fraco"></div>'
				+ '<div class="sis-linha"><label>Seus RPCs (um por linha, https://…)</label><textarea id="sisRpcs" rows="3" cols="52" autocomplete="off" spellcheck="false" placeholder="https://mainnet.helius-rpc.com/?api-key=…"></textarea></div>'
				+ '<div class="sis-linha"><label class="sis-chk"><input id="sisAceitarRpcs" type="checkbox"> Usar os RPCs definidos pelo dashboard (Administração › RPCs)</label></div>'
				+ '<small>Ordem: os seus RPCs (acima) → os do dashboard → a lista padrão do script (só o RPC oficial do Star Atlas), que só vale quando o dashboard não mandou lista. Se um RPC falhar (429, cota esgotada, fora do ar), o Sis-Bot passa sozinho para o próximo. As URLs nunca vão para o console nem para o status enviado ao dashboard. Os seus RPCs valem depois de recarregar a página; os do dashboard entram sozinhos na próxima sincronização.</small>'
				+ '<h3>Tetos locais</h3><small>Valem acima de qualquer configuração recebida do dashboard. O dashboard não lê nem altera estes valores.</small>'
				+ '<div class="sis-grade"><label>Taxa de prioridade máxima (lamports)<input id="sisTetoTaxa" type="number" min="0" max="50000"></label>'
				+ '<label>Multiplicador máximo no crafting (%)<input id="sisTetoMult" type="number" min="1" max="500"></label>'
				+ '<label>Transações por hora (0 = sem limite)<input id="sisTetoTx" type="number" min="0" max="100000"></label></div>'
				+ '<div class="sis-linha"><label class="sis-chk"><input id="sisComandos" type="checkbox"> Aceitar comandos do dashboard (iniciar, parar, reiniciar frota, desacoplar, recarregar)</label></div>'
				+ '<div class="sis-linha"><label class="sis-chk"><input id="sisModoLocal" type="checkbox"> Modo local: ignorar configurações e comandos do dashboard (continua só enviando o status)</label></div>'
				+ '<div class="sis-linha"><label class="sis-chk"><input id="sisRetomarAuto" type="checkbox"> Voltar a rodar sozinho depois de recarregar a página ou atualizar o script (se estava rodando)</label></div>'
				+ '<h3>Este navegador</h3><div class="sis-grade"><label>Log no console (0–9)<input id="debugLogLevel" type="number" min="0" max="9"></label>'
				+ '<label>Opacidade do fundo do painel de status (%, mín. 55)<input id="statusPanelOpacity" type="number" min="1" max="100"></label>'
				+ '<label>Frotas por coluna (só no painel antigo; o novo é uma frota por linha)<input id="fleetsPerColumn" type="number" min="0" max="100"></label></div>'
				+ '<div class="sis-botoes"><button class="assist-modal-btn" id="sisTrocarPerfil">Escolher outro perfil</button> <button class="assist-modal-btn" id="disableLoadOtherFleets">Disable "Load other fleets"</button></div>'
				+ '</div>');
			sisModal.append(sisModalContent);

			let sisAceiteModal = document.createElement('div');
			sisAceiteModal.classList.add('assist-modal');
			sisAceiteModal.id = 'sisAceiteModal';
			sisAceiteModal.style.display = 'none';
			sisAceiteModal.style.zIndex = 4;
			let sisAceiteContent = document.createElement('div');
			sisAceiteContent.classList.add('assist-modal-content');
			setHtmlEstatico(sisAceiteContent, '<div class="assist-modal-header"><span id="sisAceiteTitulo">Configuração do dashboard</span><div class="assist-modal-header-right"><span class="assist-modal-close">&#x2715;</span></div></div>'
				+ '<div class="assist-modal-body sis-corpo"><div id="sisAceiteInfo"></div><div id="sisAceiteErro" class="sis-erro"></div>'
				+ '<h3>O que muda neste navegador</h3><ul id="sisAceiteLista" class="sis-lista"></ul><div id="sisAceiteMsg" class="sis-fraco"></div>'
				+ '<div class="sis-botoes"><button class="assist-modal-btn assist-modal-save" id="sisAceitar">Aceitar e aplicar</button> <button class="assist-modal-btn" id="sisRecusar">Recusar</button> <button class="assist-modal-btn" id="sisDepois">Decidir depois</button></div></div>');
			sisAceiteModal.append(sisAceiteContent);

			let errorModal = document.createElement('div');
			errorModal.classList.add('assist-modal');
			errorModal.id = 'errorModal';
			errorModal.style.display = 'none';
			errorModal.style.zIndex = 3;
			let errorModalContent = document.createElement('div');
			errorModalContent.classList.add('assist-modal-content');
			setHtmlEstatico(errorModalContent, '<div class="assist-modal-header"><span>Error Log</span><div class="assist-modal-header-right"><button id="reloadLogBtn" class="assist-modal-btn">Reload</button><button id="clearBtn" class="assist-modal-btn">Clear</button> <span class="assist-modal-close">&#x2715;</span></div></div><div class="assist-modal-body"><div>Snapshot of the error log (to see an updated state, close/open this overlay or use the reload button). Logged errors: ix errors, network errors, unhandled exceptions</div><textarea id="errorText" rows="12" cols="80" max-width="100%"></textarea></div><br>');
			errorModal.append(errorModalContent);

			let profileModal = document.createElement('div');
			profileModal.classList.add('assist-modal');
			profileModal.id = 'profileModal';
			profileModal.style.display = 'none';
			profileModal.style.zIndex = 3;
			let profileModalContent = document.createElement('div');
			profileModalContent.classList.add('assist-modal-content');
			setHtmlEstatico(profileModalContent, '<div class="assist-modal-header"><span>Profile Selection</span><div class="assist-modal-header-right"><span class="assist-modal-close">&#x2715;</span></div></div><div class="assist-modal-body"><span id="assist-modal-error"></span><div></div><span>Select a profile to connect to Lab Assistant.</span><div></div><div id="profileDiv" max-width="100%"></div></div>');
			profileModal.append(profileModalContent);

			let assistStatus = document.createElement('div');
			assistStatus.id = 'assistStatus';
			assistStatus.style.display = 'none';
			let assistStatusContent = document.createElement('div');
			assistStatusContent.classList.add('assist-status-content');
			setHtmlEstatico(assistStatusContent, '<div class="assist-modal-header" style="cursor: move;"><span class="sis-st-titulo">Sis-Bot <small>· Status das frotas</small></span><span class="sis-st-ctl"></span><small id="assist-modal-balance" class="sis-chip"></small><small id="assist-modal-fee" class="sis-chip">Fee:'+currentFee+'</small><small id="sisRpcChip" class="sis-chip" title="RPCs">RPC</small><div class="assist-modal-header-right"><button type="button" class="sis-hbtn" id="sisTelaCheia" title="Ocupar a página inteira">&#x26F6;</button><span class="assist-modal-close" title="Recolher o painel">&#x25B4;</span></div></div>'
				+ '<div class="assist-modal-body sis-st-corpo"><div id="sisStatusResumo"></div><div id="sisStatusAbas" class="sis-abas"></div>'
				+ '<div class="sis-st-filtro"><input id="sisStatusFiltro" placeholder="Filtrar frota…" autocomplete="off">'
				+ '<div class="sis-seg" id="sisModo" title="Modo de exibição (fica gravado neste navegador)"><button type="button" data-m="1" title="Uma linha por frota, só o básico">Linha</button><button type="button" data-m="2" title="Estado, local e medidores">Padrão</button><button type="button" data-m="3" title="Cartões com todos os detalhes">Completo</button></div>'
				+ '<div class="sis-seg" id="sisFonte"><button type="button" data-f="-" title="Diminuir a fonte">A−</button><span id="sisFonteVal" title="Tamanho da fonte (clique para voltar a 100%)">100%</span><button type="button" data-f="+" title="Aumentar a fonte">A+</button></div>'
				+ '<span id="sisStatusInfo"></span></div><div id="sisStatusCab" class="sis-st-cabecalho"></div><div id="sisStatusFrotas"></div>'
				+ '<div id="sisStatusPag" class="sis-pag"><button type="button" data-p="ini" title="Primeira página">«</button><button type="button" data-p="ant" title="Página anterior">‹</button><span id="sisPagTxt"></span><button type="button" data-p="prox" title="Próxima página">›</button><button type="button" data-p="fim" title="Última página">»</button><label>por página <select id="sisPorPag"><option>10</option><option>20</option><option>50</option><option>100</option></select></label></div></div>');
			assistStatus.append(assistStatusContent);

			//statsadd start
			let assistStats = document.createElement('div');
			assistStats.id = 'assistStats';
			assistStats.style.display = 'none';
			let assistStatsContent = document.createElement('div');
			assistStatsContent.classList.add('assist-status-content');
			setHtmlEstatico(assistStatsContent, '<div class="assist-modal-header" style="cursor: move;">Sis-Bot <small>· Estatísticas</small>&nbsp;<small id="assist-modal-time" style="font-size:75%;"></small>&nbsp;&nbsp;<a href="javascript:;" id="assist-stats-reset" style="color:inherit;font-size:70%;">zerar</a>&nbsp;&nbsp;<div class="assist-modal-header-right"><span class="assist-modal-close">&#x2715;</span></div></div><div id="assistStatsContent" class="assist-modal-body"></div>');
			assistStats.append(assistStatsContent);
			//statsadd end

			let assistStarbaseStatus = document.createElement('div');
			assistStarbaseStatus.id = 'assistStarbaseStatus';
			assistStarbaseStatus.style.display = 'none';
			let assistStarbaseStatusContent = document.createElement('div');
			assistStarbaseStatusContent.classList.add('assist-status-content');
			setHtmlEstatico(assistStarbaseStatusContent, '<div class="assist-modal-header" style="cursor: move;">Starbase Status<div class="assist-modal-header-right"><span class="assist-modal-close">&#x2715;</span></div></div><div class="assist-modal-body"><table style="border-spacing: 10px 0;"><tr><td>Starbase</td><td>Coords</td><td>Food</td><td>Tools</td></tr></table></div>');
			assistStarbaseStatus.append(assistStarbaseStatusContent);

			let autoContainer = document.createElement('div');
			autoContainer.style.display = 'flex';
			autoContainer.style.flexDirection = 'row';
			let autoButton = document.createElement('button');
			autoButton.id = 'autoScanBtn';
			autoButton.classList.add('assist-btn');
			autoButton.style.marginRight = '5px';
			//autoButton.style.position = 'absolute';
			//autoButton.style.left = '50%';
			//autoButton.style.transform = 'translate(-50%, 0)';
			autoButton.addEventListener('click', function(e) {toggleAssistant();});
			let autoBtnSpan = document.createElement('span');
			autoBtnSpan.innerText = initComplete == true ? enableAssistant === true ? 'Stop' : 'Start' : 'Wait...';
			autoBtnSpan.style.fontSize = '14px';
			autoButton.appendChild(autoBtnSpan);

			let dropdown = document.createElement('div');
			dropdown.classList.add('dropdown');
			let dropdownBtn = document.createElement('button');
			dropdownBtn.style.marginRight = '5px';
			dropdownBtn.classList.add('assist-btn');
			dropdownBtn.innerText = 'Tools';
			dropdownBtn.addEventListener('click', function () {
					dropdown.classList.toggle('show');
			})
			dropdown.addEventListener('click', function() {
					dropdown.classList.remove('show');
			});

			let sisButton = document.createElement('button');
			sisButton.id = 'sisBtn';
			sisButton.classList.add('assist-btn');
			sisButton.style.marginRight = '5px';
			sisButton.addEventListener('click', function(e) { if (sisPendente) sisAbrirAceite(); else sisAbrirPainel(); });
			let sisBtnSpan = document.createElement('span');
			sisBtnSpan.innerText = initComplete ? 'Sis-Ágora' : 'Wait...';
			sisBtnSpan.style.fontSize = '14px';
			sisButton.appendChild(sisBtnSpan);

			let sisPainelButton = document.createElement('button');
			sisPainelButton.classList.add('assist-btn','assist-btn-alt');
			sisPainelButton.addEventListener('click', function(e) { sisAbrirPainel(); });
			let sisPainelSpan = document.createElement('span');
			sisPainelSpan.innerText = 'Sis-Ágora: conexão e tetos';
			sisPainelSpan.style.fontSize = '14px';
			sisPainelButton.appendChild(sisPainelSpan);

			let undockAllButton = document.createElement('button');
			undockAllButton.classList.add('assist-btn','assist-btn-alt');
			undockAllButton.addEventListener('click', function(e) { if (enableAssistant) { alert('Pare o bot antes de desacoplar todas as frotas.'); return; } handleUndockAll(); });
			let undockAllSpan = document.createElement('span');
			undockAllSpan.innerText = 'Undock all';
			undockAllSpan.style.fontSize = '14px';
			undockAllButton.appendChild(undockAllSpan);

			let assistStatusButton = document.createElement('button');
			assistStatusButton.id = 'assistStatusBtn';
			assistStatusButton.classList.add('assist-btn','assist-btn-alt');
			assistStatusButton.addEventListener('click', function(e) {assistStatusToggle();});
			let assistStatusSpan = document.createElement('span');
			assistStatusSpan.innerText = 'Status';
			assistStatusSpan.style.fontSize = '14px';
			assistStatusButton.appendChild(assistStatusSpan);

			//statsadd
			let assistStatsButton = document.createElement('button');
			assistStatsButton.id = 'assistStatsBtn';
			assistStatsButton.classList.add('assist-btn','assist-btn-alt');
			assistStatsButton.addEventListener('click', function(e) {assistStatToggle('#assistStats');});
			let assistStatsSpan = document.createElement('span');
			assistStatsSpan.innerText = 'Statistics';
			assistStatsSpan.style.fontSize = '14px';
			assistStatsButton.appendChild(assistStatsSpan);
			//statsadd

			let assistErrorButton = document.createElement('button');
			assistErrorButton.id = 'assistErrorBtn';
			assistErrorButton.classList.add('assist-btn','assist-btn-alt');
			assistErrorButton.addEventListener('click', function(e) {assistErrorToggle();});
			let assistErrorSpan = document.createElement('span');
			assistErrorSpan.innerText = 'Error log';
			assistErrorSpan.style.fontSize = '14px';
			assistErrorButton.appendChild(assistErrorSpan);

			let assistStarbaseStatusButton = document.createElement('button');
			assistStarbaseStatusButton.id = 'assistStarbaseStatusBtn';
			assistStarbaseStatusButton.classList.add('assist-btn','assist-btn-alt');
			assistStarbaseStatusButton.addEventListener('click', function(e) {assistStarbaseStatusToggle();});
			let assistStarbaseStatusSpan = document.createElement('span');
			assistStarbaseStatusSpan.innerText = 'Starbase Status';
			assistStarbaseStatusSpan.style.fontSize = '14px';
			assistStarbaseStatusButton.appendChild(assistStarbaseStatusSpan);


			dropdown.appendChild(assistStatusButton);
			dropdown.appendChild(assistStarbaseStatusButton);
			dropdown.appendChild(assistStatsButton); //statsadd
			//dropdown.appendChild(assistConfigButton);
			dropdown.appendChild(sisPainelButton);
			dropdown.appendChild(undockAllButton);
			dropdown.appendChild(assistErrorButton);

			let targetElem = document.querySelector('body');
			// [Sis-Ágora] 1.1.9: Start/Stop, Sis-Ágora e Tools ficam no cabeçalho do painel de Status; o container só guarda as janelas
			if (observer) {
					autoContainer.id = 'assistContainer';
					targetElem = document.querySelector('#root > div:first-of-type > div:first-of-type > div > header > h1');
					targetElem.style.fontSize = '18px';
					targetElem.append(assistCSS);
					targetElem.append(autoContainer);
			} else {
					autoContainer.id = 'assistContainerIso';
					targetElem.prepend(autoContainer);
					targetElem.prepend(assistCSS);
			}
			assistStatusContent.querySelector('.sis-st-ctl').append(autoButton, sisButton, dropdownBtn, dropdown);
			// these were originally attached to targetElem
			autoContainer.append(sisModal);
			autoContainer.append(sisAceiteModal);
			autoContainer.append(assistStatus);
			autoContainer.append(assistStarbaseStatus);
			autoContainer.append(assistStats); //statsadd
			autoContainer.append(errorModal);
			autoContainer.append(profileModal);
			//autoContainer.append(addAcctModal);
			document.querySelector('#sisModal .assist-modal-close').addEventListener('click', function(e) { sisAbrirPainel(); });
			document.querySelector('#sisSalvar').addEventListener('click', function(e) { sisSalvarPainel(); });
			document.querySelector('#sisSyncBtn').addEventListener('click', function(e) { sisSyncAgora(); });
			document.querySelector('#sisVerPendente').addEventListener('click', function(e) { sisAbrirAceite(); });
			document.querySelector('#sisTrocarPerfil').addEventListener('click', async function(e) {
				const g = JSON.parse(await GM.getValue(settingsGmKey, '{}')); g.savedProfile = []; await GM.setValue(settingsGmKey, JSON.stringify(g)); location.reload();
			});
			document.querySelector('#sisAceiteModal .assist-modal-close').addEventListener('click', function(e) { sisFecharAceite(); });
			document.querySelector('#sisDepois').addEventListener('click', function(e) { sisFecharAceite(); });
			document.querySelector('#sisAceitar').addEventListener('click', function(e) { sisAplicar(); });
			document.querySelector('#sisRecusar').addEventListener('click', function(e) { sisRecusar(); });
			let assistStatusClose = document.querySelector('#assistStatus .assist-modal-close');
			assistStatusClose.addEventListener('click', function(e) {assistStatusToggle();});
			let assistStarbaseStatusClose = document.querySelector('#assistStarbaseStatus .assist-modal-close');
			assistStarbaseStatusClose.addEventListener('click', function(e) {assistStarbaseStatusToggle();});
			let assistStatsClose = document.querySelector('#assistStats .assist-modal-close'); //statsadd
			assistStatsClose.addEventListener('click', function(e) {assistStatToggle('#assistStats');}); //statsadd
			let assistStatsReset = document.querySelector('#assistStats #assist-stats-reset'); //statsadd
			assistStatsReset.addEventListener('click', function(e) { transactionStats={ "start": (Math.round(Date.now() / 1000)), "groups":{} }; solanaReadCount = 0; solanaWriteCount = 0; document.querySelector('#assistStatsContent').replaceChildren(); }); //statsadd
			//let addAcctBtn = document.querySelector('#addAcctBtn');
			//addAcctBtn.addEventListener('click', function(e) {addKeyToProfile(document.querySelector('#addAcctDiv').value);});
			//let removeAcctBtn = document.querySelector('#removeAcctBtn');
			//removeAcctBtn.addEventListener('click', function(e) {removeKeyFromProfile();});
			let assistErrorClose = document.querySelector('#errorModal .assist-modal-close');
			assistErrorClose.addEventListener('click', function(e) {assistErrorToggle();});
			let assistErrorClearBtn = document.querySelector('#clearBtn');
			assistErrorClearBtn.addEventListener('click', function(e) {clearErrors();});
			let assistErrorReloadBtn = document.querySelector('#reloadLogBtn');
			assistErrorReloadBtn.addEventListener('click', function(e) {reloadErrors();});
			let disableLoadOtherFleetsButton = document.querySelector('#disableLoadOtherFleets');
			disableLoadOtherFleetsButton.addEventListener('click', function(e) {
				const sageConfig = localStorage.getItem('sage-labs-settings');
				let sageConfigParsed = JSON.parse(sageConfig);
				sageConfigParsed.loadOtherFleets = false;
				localStorage.setItem('sage-labs-settings', JSON.stringify(sageConfigParsed));
				setTxt(disableLoadOtherFleetsButton, 'Settings updated!');
				setTimeout(() => { setTxt(disableLoadOtherFleetsButton, 'Disable "Load other fleets"'); }, 2000);
			});

			let profileModalClose = document.querySelector('#profileModal .assist-modal-close');
			profileModalClose.addEventListener('click', function(e) {assistProfileToggle(null);});
			//let addAcctClose = document.querySelector('#addAcctModal .assist-modal-close');
			//addAcctClose.addEventListener('click', function(e) {assistAddAcctToggle();});

			makeDraggable(assistStatus);
			sisPainelIniciar();
			makeDraggable(assistStarbaseStatus);
			makeDraggable(assistStats); //statsadd
		}
	}
	observer.observe(document, {childList: true, subtree: true});
	waitForLabs(null, null);

	await initUser();
	let autoSpanRef = document.querySelector('#autoScanBtn > span');
	autoSpanRef ? setTxt(autoSpanRef, 'Start') : null;
	cLog(0,'init complete');
	cLog(0,'Fleets:', userFleets);
})();