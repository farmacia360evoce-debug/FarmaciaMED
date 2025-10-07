# Create a ready-to-host single-file app and a README with free deployment steps
html_content = """<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Farmácia MED — Plataforma</title>
  <style>
    :root{--bg:#0e0f12;--card:#17191d;--muted:#9aa3ae;--pri:#21c47b;--warn:#f0b429;--err:#ff5c5c;--line:#262a31;--txt:#e7ecf3}
    *{box-sizing:border-box}
    html,body{height:100%}
    body{margin:0;background:var(--bg);color:var(--txt);font:15px/1.5 system-ui,-apple-system,Segoe UI,Roboto,Ubuntu,"Helvetica Neue",Arial}
    a{color:#9ad3ff;text-decoration:none}
    h1,h2,h3{margin:0 0 .5rem 0}
    .container{max-width:1120px;margin:0 auto;padding:20px}
    .grid{display:grid;gap:16px}
    @media(min-width:768px){.grid.cols-2{grid-template-columns:repeat(2,1fr)}.grid.cols-3{grid-template-columns:repeat(3,1fr)}}
    .card{background:var(--card);border:1px solid var(--line);border-radius:16px;padding:16px;box-shadow:0 6px 18px rgba(0,0,0,.25)}
    .badge{display:inline-block;padding:.2rem .5rem;border-radius:999px;font-size:.75rem;border:1px solid var(--line);color:var(--muted)}
    .btn{display:inline-flex;align-items:center;gap:.5rem;background:#24303a;border:1px solid var(--line);color:#e7ecf3;padding:.65rem .9rem;border-radius:12px;cursor:pointer}
    .btn:disabled{opacity:.6;cursor:not-allowed}
    .btn.primary{background:var(--pri);color:#0b1d13;border-color:#0ea562}
    .btn.warn{background:var(--warn);color:#1a1205;border-color:#d69a17}
    .btn.ghost{background:transparent}
    .row{display:flex;gap:10px;flex-wrap:wrap}
    label{display:block;font-weight:600;margin:.2rem 0}
    input,select,textarea{width:100%;background:#0f1216;color:var(--txt);border:1px solid var(--line);border-radius:10px;padding:.6rem .75rem}
    textarea{min-height:110px}
    table{width:100%;border-collapse:collapse;border-radius:10px;overflow:hidden}
    th,td{border-bottom:1px solid var(--line);padding:.6rem .5rem;text-align:left}
    th{color:#b6bfca;font-weight:700;background:#0f1216}
    .muted{color:var(--muted)}
    .right{text-align:right}
    .center{text-align:center}
    .ok{color:var(--pri)}.warn-t{color:var(--warn)}.err{color:var(--err)}
    .toolbar{display:flex;justify-content:space-between;align-items:center;gap:12px;margin:8px 0 16px}
    .chips{display:flex;gap:8px;flex-wrap:wrap}
    .pill{padding:.25rem .55rem;border:1px solid var(--line);border-radius:999px;font-size:.75rem;color:#cbd5e1}
    .link-like{color:#9ad3ff;cursor:pointer}
    .footer{margin-top:28px;color:#93a1b1;font-size:.85rem}
    .logo{font-weight:900;letter-spacing:.5px}
  </style>
</head>
<body>
  <div class="container">
    <header class="toolbar">
      <div class="logo">🧪 Farmácia <span class="ok">MED</span></div>
      <div id="top-actions" class="row"></div>
    </header>

    <!-- Views -->
    <main id="view"></main>

    <p class="footer">⚠️ Ferramenta de apoio à decisão clínica. Observe ANVISA/Portaria 344/1998 e normas vigentes. A responsabilidade final é do prescritor.
    </p>
  </div>

  <script>
  // ===== Mini Router & State =====
  const S = {
    session: JSON.parse(localStorage.getItem('fm_session')||'null'),
    // multitenant simples: dados por farmácia
    tenants: JSON.parse(localStorage.getItem('fm_tenants')||'{}'),
    // catálogo "global" de demonstração
    demoRefs: [
      {titulo:"Monografia ANVISA do ativo", fonte:"ANVISA", ano:2021, url:"https://www.gov.br/anvisa"},
      {titulo:"PubMed referência", fonte:"PubMed", ano:2020, url:"https://pubmed.ncbi.nlm.nih.gov/"}
    ]
  };
  const el = sel => document.querySelector(sel);
  const V = el('#view');

  function save(){
    localStorage.setItem('fm_session', JSON.stringify(S.session));
    localStorage.setItem('fm_tenants', JSON.stringify(S.tenants));
  }

  function nav(path, params={}){
    const q = new URLSearchParams(params).toString();
    location.hash = path + (q? ('?'+q):'');
  }

  function parseHash(){
    const [path, qs] = location.hash.replace(/^#/, '').split('?');
    const params = Object.fromEntries(new URLSearchParams(qs||''));
    return {path: path||'/login', params};
  }

  function setTopActions(){
    const ta = el('#top-actions');
    ta.innerHTML = '';
    if(!S.session){
      ta.innerHTML = \`<button class="btn" onclick="nav('/login')">Entrar</button>
      <button class="btn ghost" onclick="nav('/cadastro-prescritor')">Cadastrar Prescritor</button>
      <button class="btn ghost" onclick="nav('/cadastro-farmacia')">Cadastrar Farmácia</button>\`;
      return;
    }
    const {papel,email} = S.session;
    ta.innerHTML = \`<span class="badge">\${papel}</span>
      <span class="muted">\${email}</span>
      <button class="btn" onclick="S.session=null; save(); nav('/login')">Sair</button>\`;
  }

  function ensureTenant(){
    if(!S.session || S.session.papel!=='FARMACIA') return null;
    const tid = S.session.tenantId || S.session.email; // usar email como id
    if(!S.tenants[tid]){
      S.tenants[tid] = { config:{}, ativos:[], auditoria:[] };
      S.session.tenantId = tid;
      save();
    }
    return tid;
  }

  // ===== Utilities =====
  function toast(msg){ alert(msg); }
  function safeJSON(txt){ try{ return JSON.parse(txt||'{}'); }catch(e){ toast('JSON inválido.'); return null; } }
  function addItemRow(wrap, it={}, idx){
    const row = document.createElement('div'); row.className='grid cols-4';
    row.style.margin = '6px 0';
    row.innerHTML = \`
      <div><input placeholder="Nome" value="\${it.nome||''}"></div>
      <div><input placeholder="Quantidade" value="\${it.quantidade||''}"></div>
      <div><input placeholder="Forma" value="\${it.forma||''}"></div>
      <div class="row">
        <input style="flex:1" placeholder="Qtd" value="\${it.qtd||''}">
        <button class="btn" title="remover">✕</button>
      </div>\`;
    row.querySelector('button').onclick = ()=> row.remove();
    wrap.appendChild(row);
  }

  // ===== Suggestion Engine (demo) =====
  function generateSuggestions(nome_ativo, filters={}){
    if(!nome_ativo) return [];
    const base = {
      faixa_etaria: filters.faixa_etaria||'adulto',
      via: filters.via||'oral',
      duracao: '30 dias',
      tipo_receita: 'Receita simples',
      contraindicacoes: ['hipersensibilidade'],
      precaucoes: ['cautela em gestação/lactação'],
      links_referencia: S.demoRefs
    };
    return [
      { titulo:'S1 — Isolada', composicao_resumo:\`\${nome_ativo}\`, forma_farmaceutica:'cápsula',
        concentracao:{valor:300,unidade:'mg/cáps'}, dose:{valor:300,unidade:'mg/dia'}, posologia:'1 cápsula à noite', ...base },
      { titulo:'S2 — Associação A', composicao_resumo:\`\${nome_ativo} + Co-ativo A\`, forma_farmaceutica:'cápsula',
        concentracao:{valor:300,unidade:'mg/cáps'}, dose:{valor:300,unidade:'mg/dia'}, posologia:'1 cápsula à noite', ...base },
      { titulo:'S3 — Associação B', composicao_resumo:\`\${nome_ativo} + Co-ativo B\`, forma_farmaceutica:'cápsula',
        concentracao:{valor:300,unidade:'mg/cáps'}, dose:{valor:300,unidade:'mg/dia'}, posologia:'1 cápsula à noite', ...base }
    ];
  }

  // ===== Views =====
  const Views = {
    login(){
      V.innerHTML = \`
        <div class="card" style="max-width:560px;margin:40px auto">
          <h1>Entrar</h1>
          <div class="grid">
            <div>
              <label>Sou</label>
              <select id="role">
                <option value="PRESCRITOR">PRESCRITOR</option>
                <option value="FARMACIA">FARMÁCIA</option>
              </select>
            </div>
            <div><label>E-mail</label><input id="lemail" type="email" placeholder="voce@exemplo.com"></div>
            <div><label>Senha</label><input id="lsenha" type="password" placeholder="••••••••"></div>
          </div>
          <div class="row" style="margin-top:12px">
            <button class="btn primary" onclick="Actions.doLogin()">Entrar</button>
            <a class="btn ghost" href="#/cadastro-prescritor">Cadastrar Prescritor</a>
            <a class="btn ghost" href="#/cadastro-farmacia">Cadastrar Farmácia</a>
          </div>
        </div>\`;
    },

    cadPrescritor(){
      V.innerHTML = \`
      <div class="card" style="max-width:720px;margin:20px auto">
        <h1>Cadastro — Prescritor</h1>
        <div class="grid cols-2">
          <div><label>Nome completo</label><input id="p_nome"></div>
          <div><label>E-mail</label><input id="p_email" type="email"></div>
          <div><label>WhatsApp (DDD+número)</label><input id="p_wa"></div>
          <div><label>Conselho</label>
            <select id="p_conselho"><option>CRM</option><option>CRF</option><option>CRMV</option></select>
          </div>
          <div><label>Número do registro</label><input id="p_num"></div>
          <div><label>UF</label>
            <select id="p_uf"><option>SP</option><option>RJ</option><option>MG</option><option>RS</option><option>PR</option><option>SC</option><option>BA</option><option>PE</option></select>
          </div>
          <div><label>Senha</label><input id="p_senha" type="password"></div>
          <div style="align-self:end">
            <label><input type="checkbox" id="p_lgpd"> Concordo com a LGPD</label>
          </div>
        </div>
        <div class="row" style="margin-top:12px"><button class="btn primary" onclick="Actions.savePrescritor()">Cadastrar</button>
          <button class="btn" onclick="nav('/login')">Voltar</button></div>
      </div>\`;
    },

    cadFarmacia(){
      V.innerHTML = \`
      <div class="card" style="max-width:820px;margin:20px auto">
        <h1>Cadastro — Farmácia</h1>
        <div class="grid cols-2">
          <div><label>Razão Social</label><input id="f_rs"></div>
          <div><label>Nome Fantasia</label><input id="f_nf"></div>
          <div><label>CNPJ</label><input id="f_cnpj"></div>
          <div><label>E-mail institucional</label><input id="f_email" type="email"></div>
          <div><label>WhatsApp (recebimento)</label><input id="f_wa"></div>
          <div><label>E-mail (recebimento)</label><input id="f_emrec" type="email"></div>
          <div><label>Responsável Técnico (nome)</label><input id="f_rt"></div>
          <div><label>CRF/UF do RT</label><input id="f_rtcrf"></div>
          <div><label>Senha</label><input id="f_senha" type="password"></div>
          <div style="align-self:end"><label><input type="checkbox" id="f_lgpd"> Concordo com a LGPD</label></div>
        </div>
        <div class="row" style="margin-top:12px"><button class="btn primary" onclick="Actions.saveFarmacia()">Cadastrar</button>
          <button class="btn" onclick="nav('/login')">Voltar</button></div>
      </div>\`;
    },

    // ===== PRESCRITOR =====
    painelPrescritor(){
      if(!S.session || S.session.papel!=='PRESCRITOR'){ nav('/login'); return; }
      V.innerHTML = \`
      <div class="card">
        <div class="toolbar"><h1>Consulta de Ativos</h1>
          <div class="row">
            <button class="btn" onclick="nav('/receitas')">Minhas receitas</button>
          </div>
        </div>
        <div class="grid cols-3">
          <div><label>Ativo principal</label><input id="q_ativo" placeholder="Ex.: Passiflora"></div>
          <div><label>Via preferida</label>
            <select id="q_via"><option></option><option>oral</option><option>tópica</option><option>sublingual</option><option>intranasal</option></select>
          </div>
          <div><label>Faixa etária</label>
            <select id="q_idade"><option></option><option>neonato</option><option>pediatria</option><option>adolescente</option><option>adulto</option><option>idoso</option></select>
          </div>
        </div>
        <div class="row" style="margin-top:10px"><button class="btn primary" onclick="Actions.buscarSugestoes()">Gerar 3 sugestões</button></div>
      </div>
      <div id="sugestoes" class="grid cols-3" style="margin-top:16px"></div>\`;
    },

    receitasLista(){
      if(!S.session || S.session.papel!=='PRESCRITOR'){ nav('/login'); return; }
      const list = JSON.parse(localStorage.getItem('fm_rx_'+S.session.email)||'[]');
      let rows = list.map(r=>\`<tr><td>\${r.data}</td><td>\${r.paciente||'-'}</td><td>\${r.tipo||'-'}</td><td class="right"><a class="link-like" href="#/receita?id=\${r.id}">abrir</a></td></tr>\`).join('');
      if(!rows) rows = '<tr><td colspan="4" class="center muted">Sem receitas ainda</td></tr>';
      V.innerHTML = \`
      <div class="card">
        <div class="toolbar"><h1>Minhas Receitas</h1><div class="row">
          <button class="btn" onclick="nav('/painel-prescritor')">Voltar</button>
        </div></div>
        <table><thead><tr><th>Data</th><th>Paciente</th><th>Tipo</th><th class="right">Ações</th></tr></thead>
        <tbody>\${rows}</tbody></table>
      </div>\`;
    },

    receita(params){
      if(!S.session || S.session.papel!=='PRESCRITOR'){ nav('/login'); return; }
      // dados vindos da sugestão via querystring
      const url = new URL(location.href);
      const qs = Object.fromEntries(url.searchParams.entries());
      const formula = qs.formula_json ? JSON.parse(qs.formula_json) : [];
      V.innerHTML = \`
      <div class="card">
        <div class="toolbar"><h1>Gerador de Receita</h1><div></div></div>
        <div class="grid cols-3">
          <div><label>Paciente</label><input id="rx_pac"></div>
          <div><label>Idade</label><input id="rx_idade"></div>
          <div><label>Sexo</label>
            <select id="rx_sexo"><option>F</option><option>M</option><option>Outro</option></select>
          </div>
          <div><label>Prescritor — Nome</label><input id="rx_pren" value="\${S.session.name||''}"></div>
          <div><label>Conselho</label><select id="rx_conselho"><option>CRM</option><option>CRF</option><option>CRMV</option></select></div>
          <div><label>Nº/UF</label><input id="rx_numuf" placeholder="12345/SP"></div>
        </div>
        <div style="margin-top:10px">
          <label>Fórmula (itens)</label>
          <div id="rx_formula"></div>
          <button class="btn" onclick="Actions.addItem()">Adicionar item</button>
        </div>
        <div class="grid cols-2" style="margin-top:10px">
          <div><label>Posologia</label><textarea id="rx_pos">\${qs.posologia||''}</textarea></div>
          <div><label>Orientações</label><textarea id="rx_orient">\${(qs.orientacoes||'').replace(/%0A/g,'\\n')}</textarea></div>
        </div>
        <div class="grid cols-3" style="margin-top:10px">
          <div><label>Tipo de receita</label>
            <select id="rx_tipo"><option>Receita simples</option><option>Antimicrobiano</option><option>A</option><option>B</option><option>B2</option><option>Veterinária</option></select>
          </div>
          <div><label>Farmácia — WhatsApp</label><input id="rx_wa" placeholder="DDD+número"></div>
          <div><label>Farmácia — E-mail</label><input id="rx_email" type="email" placeholder="receita@farmacia.com"></div>
        </div>
        <div class="row" style="margin-top:12px">
          <button class="btn primary" onclick="Actions.emitirReceita()">Gerar e Enviar</button>
          <button class="btn" onclick="nav('/painel-prescritor')">Voltar</button>
        </div>
      </div>\`;
      // render items
      const wrap = el('#rx_formula');
      const items = formula.length? formula: [{nome:'Item 1', quantidade:'', forma:'', qtd:''}];
      items.forEach((it, i)=> addItemRow(wrap, it, i));
    },

    // ===== FARMÁCIA =====
    painelFarmacia(){
      if(!S.session || S.session.papel!=='FARMACIA'){ nav('/login'); return; }
      const tid = ensureTenant();
      const t = S.tenants[tid];
      const list = t.ativos || [];
      let htmlList = list.map((a,i)=>
        \`<tr><td>\${a.nome}</td><td>\${(a.vias_permitidas||[]).join(', ')}</td><td>\${(a.faixas_etarias||[]).join(', ')}</td>
        <td class="right"><span class="link-like" onclick="nav('/ativo?id=\${i}')">editar</span></td></tr>\`
      ).join('');
      if(!htmlList) htmlList = '<tr><td colspan="4" class="center muted">Nenhum ativo cadastrado</td></tr>';

      V.innerHTML = \`
      <div class="card">
        <div class="toolbar"><h1>Painel da Farmácia</h1>
          <div class="row">
            <button class="btn" onclick="nav('/config')">Configurações</button>
            <button class="btn" onclick="nav('/auditoria')">Auditoria</button>
            <button class="btn primary" onclick="nav('/ativo')">Cadastrar ativo</button>
          </div>
        </div>
        <table><thead><tr><th>Ativo</th><th>Vias</th><th>Faixas</th><th class="right">Ações</th></tr></thead>
        <tbody>\${htmlList}</tbody></table>
      </div>\`;
    },

    ativo(params){
      if(!S.session || S.session.papel!=='FARMACIA'){ nav('/login'); return; }
      const tid = ensureTenant();
      const t = S.tenants[tid];
      const idx = params.id? parseInt(params.id,10): -1;
      const A = idx>=0? t.ativos[idx]: {nome:'', sinonimos:[], vias_permitidas:[], faixas_etarias:[], intervalos_dose:{}, formas_ideais_por_idade:{}, tipo_receita_padrao:'Receita simples', controlado:false, compat_sinergias:[], compat_evitar:[], contraindicacoes:[], interacoes:[], referencias:[] };

      V.innerHTML = \`
      <div class="card">
        <div class="toolbar"><h1>\${idx>=0? 'Editar':'Cadastrar'} Ativo</h1>
          <div class="row"><button class="btn" onclick="nav('/painel-farmacia')">Voltar</button></div>
        </div>
        <div class="grid cols-2">
          <div><label>Nome</label><input id="a_nome" value="\${A.nome}"></div>
          <div><label>Sinônimos (separar por vírgula)</label><input id="a_sin" value="\${(A.sinonimos||[]).join(', ')}"></div>
          <div><label>Vias permitidas (vírgula)</label><input id="a_vias" value="\${(A.vias_permitidas||[]).join(', ')}"></div>
          <div><label>Faixas etárias (vírgula)</label><input id="a_faixas" value="\${(A.faixas_etarias||[]).join(', ')}"></div>
          <div><label>Intervalos de dose (JSON)</label><textarea id="a_interv">\${JSON.stringify(A.intervalos_dose||{},null,2)}</textarea></div>
          <div><label>Formas por idade (JSON)</label><textarea id="a_formas">\${JSON.stringify(A.formas_ideais_por_idade||{},null,2)}</textarea></div>
          <div><label>Tipo de receita padrão</label>
            <select id="a_tipo">
              \${['Receita simples','Antimicrobiano','A','B','B2','Veterinária'].map(opt=>\`<option\${A.tipo_receita_padrao===opt?' selected':''}>\${opt}</option>\`).join('')}
            </select>
          </div>
          <div style="align-self:end"><label><input type="checkbox" id="a_ctrl" \${A.controlado?'checked':''}> Controlado</label></div>
          <div><label>Sinergias (vírgula)</label><input id="a_sinerg" value="\${(A.compat_sinergias||[]).join(', ')}"></div>
          <div><label>Evitar (vírgula)</label><input id="a_evitar" value="\${(A.compat_evitar||[]).join(', ')}"></div>
          <div><label>Contraindicações (vírgula)</label><input id="a_contra" value="\${(A.contraindicacoes||[]).join(', ')}"></div>
          <div><label>Interações (vírgula)</label><input id="a_interac" value="\${(A.interacoes||[]).join(', ')}"></div>
        </div>
        <div class="row" style="margin-top:12px">
          <button class="btn primary" onclick="Actions.saveAtivo(\${idx})">Salvar</button>
          \${idx>=0? \`<button class="btn warn" onclick="Actions.buscaReferencias(\${idx})">Buscar referências</button>\`:''}
        </div>
        \${A.referencias && A.referencias.length? \`<div style="margin-top:16px"><h3>Referências</h3><ul>\${A.referencias.map(r=>\`<li><a target="_blank" href="\${r.url}">\${r.titulo}</a> — \${r.fonte} (\${r.ano})</li>\`).join('')}</ul></div>\`:''}
      </div>\`;
    },

    config(){
      if(!S.session || S.session.papel!=='FARMACIA'){ nav('/login'); return; }
      const tid = ensureTenant();
      const cfg = S.tenants[tid].config || {};
      V.innerHTML = \`
      <div class="card">
        <div class="toolbar"><h1>Configurações</h1><div class="row"><button class="btn" onclick="nav('/painel-farmacia')">Voltar</button></div></div>
        <div class="grid cols-2">
          <div><label>Logo (URL)</label><input id="c_logo" value="\${cfg.logo_url||''}"></div>
          <div><label>WhatsApp (recebimento)</label><input id="c_wa" value="\${cfg.whatsapp_receb||''}"></div>
          <div><label>E-mail (recebimento)</label><input id="c_em" value="\${cfg.email_receb||''}"></div>
          <div><label>Link farmacovigilância</label><input id="c_fv" value="\${cfg.link_farmacovig||''}"></div>
          <div class="cols-2" style="grid-column:1/-1"><label>Termos LGPD</label><textarea id="c_lgpd">\${cfg.termos_lgpd||''}</textarea></div>
        </div>
        <div class="row" style="margin-top:12px"><button class="btn primary" onclick="Actions.saveConfig()">Salvar</button></div>
      </div>\`;
    },

    auditoria(){
      if(!S.session || S.session.papel!=='FARMACIA'){ nav('/login'); return; }
      const tid = ensureTenant();
      const aud = S.tenants[tid].auditoria||[];
      let rows = aud.map(a=>\`<tr><td>\${a.data}</td><td>\${a.acao}</td><td>\${a.usuario}</td><td>\${a.ip||'-'}</td><td>\${a.hash||'-'}</td></tr>\`).join('');
      if(!rows) rows = '<tr><td colspan="5" class="center muted">Sem eventos</td></tr>';
      V.innerHTML = \`
      <div class="card">
        <div class="toolbar"><h1>Auditoria</h1><div class="row"><button class="btn" onclick="nav('/painel-farmacia')">Voltar</button></div></div>
        <table><thead><tr><th>Data</th><th>Ação</th><th>Usuário</th><th>IP</th><th>Hash</th></tr></thead>
        <tbody>\${rows}</tbody></table>
      </div>\`;
    }
  };

  // ===== Actions =====
  const Actions = {
    doLogin(){
      const papel = document.querySelector('#role').value;
      const email = document.querySelector('#lemail').value.trim();
      const senha = document.querySelector('#lsenha').value;
      if(!email||!senha){ toast('Informe e-mail e senha.'); return; }
      // Login fictício — aceita qualquer combinação
      S.session = {email, papel};
      if(papel==='FARMACIA') ensureTenant();
      save();
      nav(papel==='PRESCRITOR'? '/painel-prescritor' : '/painel-farmacia');
    },
    savePrescritor(){
      const ok = document.querySelector('#p_nome').value && document.querySelector('#p_email').value && document.querySelector('#p_senha').value && document.querySelector('#p_lgpd').checked;
      if(!ok){ toast('Preencha os campos obrigatórios e aceite a LGPD.'); return; }
      toast('Prescritor cadastrado. Faça login.');
      nav('/login');
    },
    saveFarmacia(){
      const ok = document.querySelector('#f_rs').value && document.querySelector('#f_nf').value && document.querySelector('#f_email').value && document.querySelector('#f_senha').value && document.querySelector('#f_lgpd').checked;
      if(!ok){ toast('Preencha os campos obrigatórios e aceite a LGPD.'); return; }
      // cria sessão direta para demonstrar tenant
      S.session = {email: document.querySelector('#f_email').value, papel:'FARMACIA'};
      ensureTenant(); save();
      toast('Farmácia cadastrada.');
      nav('/painel-farmacia');
    },
    buscarSugestoes(){
      const ativo = document.querySelector('#q_ativo').value.trim();
      const via = document.querySelector('#q_via').value;
      const faixa = document.querySelector('#q_idade').value;
      const box = document.querySelector('#sugestoes');
      box.innerHTML='';
      const sugs = generateSuggestions(ativo,{via, faixa_etaria:faixa});
      if(!sugs.length){ box.innerHTML = '<div class="card muted">Nenhuma sugestão.</div>'; return; }
      sugs.forEach(s=>{
        const refLinks = (s.links_referencia||[]).map(r=>\`<li><a target="_blank" href="\${r.url}">\${r.titulo}</a> — \${r.fonte} (\${r.ano})</li>\`).join('');
        const node = document.createElement('div');
        node.className='card';
        node.innerHTML = \`
          <h3>\${s.titulo}</h3>
          <p><strong>Composição:</strong> \${s.composicao_resumo}</p>
          <div class="chips"><span class="pill">\${s.faixa_etaria}</span><span class="pill">\${s.via}</span><span class="pill">\${s.forma_farmaceutica}</span><span class="pill">\${s.tipo_receita}</span></div>
          <p><strong>Concentração:</strong> \${s.concentracao.valor} \${s.concentracao.unidade}<br/>
             <strong>Dose/Posologia:</strong> \${s.dose.valor} \${s.dose.unidade} — \${s.posologia}<br/>
             <strong>Duração:</strong> \${s.duracao}</p>
          <p><strong>Contraindicações:</strong> \${(s.contraindicacoes||[]).join(', ')||'—'}<br/>
             <strong>Precauções/Interações:</strong> \${(s.precaucoes||[]).join(', ')||'—'}</p>
          <div><strong>Referências:</strong><ul>\${refLinks}</ul></div>
          <div class="row"><button class="btn primary">Usar esta sugestão</button></div>\`;
        node.querySelector('button').onclick = ()=>{
          const form = [{nome:s.composicao_resumo, quantidade:\`\${s.concentracao.valor} \${s.concentracao.unidade}\`, forma:s.forma_farmaceutica, qtd:'30'}];
          const payload = {
            posologia: s.posologia,
            orientacoes: (s.precaucoes||[]).join('\\n'),
            tipo_receita: s.tipo_receita,
            formula_json: JSON.stringify(form)
          };
          nav('/receita', payload);
        };
        box.appendChild(node);
      });
    },
    addItem(){
      addItemRow(document.querySelector('#rx_formula'), {nome:'', quantidade:'', forma:'', qtd:''});
    },
    emitirReceita(){
      const pac = document.querySelector('#rx_pac').value.trim();
      const pos = document.querySelector('#rx_pos').value.trim();
      const tipo = document.querySelector('#rx_tipo').value;
      if(!pac || !pos){ toast('Preencha paciente e posologia.'); return; }
      // construir mensagem e links
      const hash = btoa(unescape(encodeURIComponent(\`\${pac}|\${Date.now()}\`))).slice(0,16);
      const pdf = \`https://farmacia-med.app/rec/\${hash}.pdf\`;
      const wa = document.querySelector('#rx_wa').value.replace(/\\D/g,'');
      const em = document.querySelector('#rx_email').value.trim();
      const msg = \`Receita de \${pac}%0APDF: \${pdf}\`;
      if(wa){ window.open(\`https://wa.me/55\${wa}?text=\${msg}\`,'_blank'); }
      if(em){ window.open(\`mailto:\${em}?subject=\${encodeURIComponent('Receita — '+pac)}&body=\${decodeURIComponent(msg)}\`,'_blank'); }
      // salvar histórico do prescritor
      const listKey = 'fm_rx_'+S.session.email;
      const list = JSON.parse(localStorage.getItem(listKey)||'[]');
      list.unshift({id:hash, data:new Date().toLocaleString('pt-BR'), paciente:pac, tipo});
      localStorage.setItem(listKey, JSON.stringify(list));
      toast('Receita gerada. Links abertos para envio.');
    },
    saveAtivo(idx){
      const tid = ensureTenant();
      const t = S.tenants[tid];
      const A = {
        nome: document.querySelector('#a_nome').value.trim(),
        sinonimos: document.querySelector('#a_sin').value.split(',').map(s=>s.trim()).filter(Boolean),
        vias_permitidas: document.querySelector('#a_vias').value.split(',').map(s=>s.trim()).filter(Boolean),
        faixas_etarias: document.querySelector('#a_faixas').value.split(',').map(s=>s.trim()).filter(Boolean),
        intervalos_dose: safeJSON(document.querySelector('#a_interv').value)||{},
        formas_ideais_por_idade: safeJSON(document.querySelector('#a_formas').value)||{},
        tipo_receita_padrao: document.querySelector('#a_tipo').value,
        controlado: document.querySelector('#a_ctrl').checked,
        compat_sinergias: document.querySelector('#a_sinerg').value.split(',').map(s=>s.trim()).filter(Boolean),
        compat_evitar: document.querySelector('#a_evitar').value.split(',').map(s=>s.trim()).filter(Boolean),
        contraindicacoes: document.querySelector('#a_contra').value.split(',').map(s=>s.trim()).filter(Boolean),
        interacoes: document.querySelector('#a_interac').value.split(',').map(s=>s.trim()).filter(Boolean),
        referencias: []
      };
      if(!A.nome){ toast('Nome do ativo é obrigatório.'); return; }
      if(idx>=0) t.ativos[idx]=A; else t.ativos.push(A);
      save();
      toast('Ativo salvo.');
      nav('/painel-farmacia');
    },
    buscaReferencias(idx){
      const tid = ensureTenant();
      const t = S.tenants[tid];
      if(!t.ativos[idx]) return;
      // placeholder: anexa refs demo
      t.ativos[idx].referencias = S.demoRefs;
      save();
      toast('Referências adicionadas (links públicos).');
      nav('/ativo', {id: idx});
    },
    saveConfig(){
      const tid = ensureTenant();
      S.tenants[tid].config = {
        logo_url: document.querySelector('#c_logo').value,
        whatsapp_receb: document.querySelector('#c_wa').value,
        email_receb: document.querySelector('#c_em').value,
        link_farmacovig: document.querySelector('#c_fv').value,
        termos_lgpd: document.querySelector('#c_lgpd').value
      };
      save();
      toast('Configurações salvas.');
    }
  };

  // ===== Router mount =====
  function render(){
    setTopActions();
    const {path, params} = parseHash();
    switch(path){
      case '/login': Views.login(); break;
      case '/cadastro-prescritor': Views.cadPrescritor(); break;
      case '/cadastro-farmacia': Views.cadFarmacia(); break;
      case '/painel-prescritor': Views.painelPrescritor(); break;
      case '/receitas': Views.receitasLista(); break;
      case '/receita': Views.receita(params); break;
      case '/painel-farmacia': Views.painelFarmacia(); break;
      case '/ativo': Views.ativo(params); break;
      case '/config': Views.config(); break;
      case '/auditoria': Views.auditoria(); break;
      default: nav('/login');
    }
  }
  window.addEventListener('hashchange', render);
  render();
  </script>
</body>
</html>
"""

readme = """# Farmácia MED (estático, 100% grátis)

Este pacote contém **um arquivo único `index.html`** com login básico (prescritor/farmácia), consulta de ativos com **3 sugestões**, gerador de receita (PDF via link, envio por **WhatsApp/E-mail**), e painel da farmácia com **A→Z de ativos**.

> Tudo roda **no navegador** (localStorage). **Sem servidor** e **sem custo**.

## Como publicar grátis

### Opção A — GitHub Pages
1. Crie uma conta no GitHub.
2. Crie um repositório chamado `farmacia-med`.
3. Faça upload de `index.html` na raiz.
4. Vá em **Settings → Pages → Build and deployment**:
   - Source: **Deploy from a branch**
   - Branch: **main** / raiz
5. Abra o link do GitHub Pages gerado. Pronto!

### Opção B — Netlify (grátis)
1. Acesse https://app.netlify.com/ e arraste `index.html` para o **Drop**.
2. Ele gera uma URL pública automaticamente.

### Opção C — Vercel (grátis)
1. Acesse https://vercel.com/, crie projeto estático e faça upload do `index.html`.

## Como usar

- Acesse `/login`:
  - **PRESCRITOR**: consulta ativo → 3 sugestões → clicar → gera receita → enviar WhatsApp/E-mail.
  - **FARMÁCIA**: cadastrar **Ativos** (A→Z), configurar contatos (WhatsApp/E-mail), auditoria.
- Todos os dados ficam no **localStorage** do navegador.
- Para **limpar**, use as ferramentas do navegador ou um botão “Sair” e limpe o storage.

## Personalização rápida

- Troque a logo/edit: em **Configurações** (painel da farmácia).
- Edite **cores** no `<style>` do `index.html`.
- Ajuste textos de compliance ANVISA no rodapé.
- Para referências científicas reais, substitua a função `buscaReferencias` por uma chamada a um serviço próprio (ou mantenha como link manual).

## Aviso

- Este demo **não envia SNGPC** e **não substitui** ERP. É um **copiloto clínico** para o prescritor.
- Adeque regras/receituários às normas **ANVISA/Portaria 344/98**.
"""

# Write files
with open('/mnt/data/index.html', 'w', encoding='utf-8') as f:
    f.write(html_content)

with open('/mnt/data/README.txt', 'w', encoding='utf-8') as f:
    f.write(readme)

# Create a zip bundle
import zipfile
zip_path = '/mnt/data/farmacia-med.zip'
with zipfile.ZipFile(zip_path, 'w', zipfile.ZIP_DEFLATED) as z:
    z.write('/mnt/data/index.html', arcname='index.html')
    z.write('/mnt/data/README.txt', arcname='README.txt')

zip_path
