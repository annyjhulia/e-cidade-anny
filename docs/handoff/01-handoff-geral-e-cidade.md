# Handoff geral — e-Cidade

> Gerado em 2026-10-15 a partir da leitura do repositório (branch `master`, último commit `f17b9737a` "Atualizacao para o php 7.4").
> Convenção: **[verificado]** = li no código/manual; **[inferido]** = deduzido por nomes/padrões, confirmar com quem conhece o sistema.

---

## 1. O que é

e-Cidade é um GRP municipal (DBSeller / Software Público Brasileiro) — ERP de prefeitura cobrindo finanças, patrimônio, tributação, RH, educação e saúde. Usado por servidores e, em parte, pelo cidadão (portal / prefeitura online).

- Repositório espelho no GitHub (DBSeller/e-cidade); oficial no Portal do Software Público. [verificado, README]
- Banco: **PostgreSQL** (`ext-pgsql`). Backend: **PHP** — README ainda diz 5.6; `composer.json` já exige `>=7.4` e o último commit migrou para 7.4. Docker em `docker/` (apache + fpm53/fpm56). [verificado]
- Há **duas arquiteturas convivendo**:
  1. **Legado (99% do sistema):** páginas PHP soltas na raiz, HTML+JS (Prototype.js) gerado no servidor, RPCs AJAX, classes `db_*_classe.php`. Aqui mora o negócio real.
  2. **Novo (Laravel 8, `app/`, `routes/`, `src/`, `api/`):** esqueleto em construção. `app/Domain/AssistenciaSocial/{Habitacao,Social}` e `routes/web/assistencia-social/{habitacao,social}.php` **existem mas estão vazios** (`.gitkeep` / `<?php`). [verificado]

Escala: ~16,5 mil arquivos versionados, 12 commits no histórico (o histórico real da DBSeller não veio junto — não dá para usar `git blame` como documentação).

---

## 2. Como o código está organizado

### 2.1 Raiz: o prefixo do arquivo é o módulo

Quase tudo está na raiz, com o padrão `<prefixo><n>_<assunto><seq>.php`:

| Sufixo/Padrão | Significado [verificado pelo padrão] |
|---|---|
| `xxx1_…` | **Cadastros** (menu Cadastros) |
| `xxx2_…` | **Relatórios / Consultas impressas** |
| `xxx3_…` | **Consultas** em tela |
| `xxx4_…` | **Procedimentos** (rotinas, processamentos, importações) |
| `…001.php` / `002` / `003` / `004` | Incluir / Alterar / Excluir (e variações) da mesma tela |
| `….RPC.php` | Endpoint AJAX (recebe `json` por POST, devolve JSON) |
| `func_….php` | Janela de **pesquisa/lookup** (popup "ancora" de código) |
| `db_*` em `libs/`, `dbforms/` | Infraestrutura (sessão, menu, formulários, rótulos) |

Prefixos por volume (nº de arquivos `.php` na raiz) e módulo provável:

| Prefixo | Qtd | Módulo |
|---|---|---|
| func | 3154 | lookups de pesquisa (transversal) |
| con | 1564 | Contabilidade **e** Configuração/DB (misturados) [inferido] |
| edu | 1255 | Educação (escola/secretaria) |
| pes | 1165 | Pessoal / Folha de pagamento |
| fis | 792 | Fiscalização / Fiscal |
| cai | 747 | Caixa / Tesouraria |
| orc | 633 | Orçamento / PPA / LDO |
| sau | 629 | Saúde (UBS, agendamento) |
| cad | 541 | Cadastro imobiliário (IPTU) |
| emp | 473 | Empenho |
| iss | 401 | ISSQN |
| com | 373 | Compras |
| pro / prot | 339+49 | Protocolo / Processos |
| rec | 283 | Recursos humanos (admissão, assentamento) |
| mat | 283 | Material / Almoxarifado |
| pat | 261 | Patrimônio |
| agu | 221 | Água |
| ate | 209 | Atendimento (suporte DBSeller) |
| lab / far / vac / tfd | 199/193/80/84 | Laboratório / Farmácia / Vacinas / TFD (saúde) |
| mer | 187 | Merenda escolar |
| vei | 172 | Veículos |
| arr | 166 | Arrecadação |
| lic | 164 | Licitações |
| jur / div | 148/148 | Jurídico / Dívida ativa e diversos |
| cem / itb / bib / ouv | 114/113/98/88 | Cemitério / ITBI / Biblioteca / Ouvidoria |
| **soc** | **87** | **Social** (ver handoff 02) |
| **hab** | **52** | **Habitação** (ver handoff 02) |
| eso | 85 | eSocial |
| aco / ac | 65/18 | Acordos / Contratos |
| tra, pre, pla, cus, tri, amb, fum… | — | Trânsito, Prefeitura Online, Planejamento, Custos, Tributário, Meio Ambiente, Fumicultura… |

### 2.2 Pastas

| Pasta | Conteúdo |
|---|---|
| `classes/` (4375) | `db_<tabela>_classe.php` — um DAO por tabela (`cl_<tabela>`), gerados do dicionário. Cabeçalho `//MODULO: x` e o bloco `$campos` documentam colunas. **Melhor "schema" disponível no repo.** |
| `model/` (1790) | Regras de negócio orientadas a objeto, por módulo (`model/social`, `model/habitacao`, `model/protocolo`…) — `Nome.model.php`. |
| `forms/` (2479) | `db_frm<assunto>.php` — corpo HTML dos formulários, incluído pelas telas `…001.php`. |
| `libs/` | núcleo: `db_conecta`, `db_sessoes`, `db_stdlib`, `db_utils`, `db_app.utils` (autoload por pacote), `db_menu`/`db_submenus`. |
| `dbforms/` | helpers de UI: `db_input`, `db_inputdata`, `db_ancora`, `db_funcoes`. |
| `std/` | `DBDate`, `DAOBasica`, `Database`… |
| `scripts/` | JS (Prototype.js, widgets `DBAbas`, `datagrid`, `DBLancador`…). |
| `dbagata/` | motor de relatórios **Agata** (templates; telas chamam `$oApiAgata->setParameter(...)`). |
| `dd/` | dicionário de dados. |
| `db/` | **migrations Phinx antigas** + `DML` + `vX.Y.Z/up|down` (releases de banco 2.3.15→2.3.58). `database/migrations` = migrations Laravel recentes (2022+). |
| `manuais/` (6429) | **Manuais HTML por módulo** — a documentação funcional oficial. Ex.: `manuais/social`, `manuais/habitacao`. |
| `release_notes/` | notas de versão. |
| `mensagens/` | JSON de mensagens/textos (i18n por módulo). |
| `webservices/`, `api/`, `integracao_externa/` | SOAP/REST e integrações (webiss, gissonline, infisc, portal transparência, iptu2agua, mensageria…). |
| `src/` | código PSR-4 novo (ESocial, Sped, Patrimonial, Financeiro, Educação, Saúde, Tributário, RH). |
| `app/`, `routes/`, `bootstrap/`, `artisan` | Laravel 8 (Passport, Pusher). Domínios espelham módulos; maioria vazia. |
| `tests/` | PHPUnit (Feature/Unit) + Cypress. Cobertura mínima. |
| `docker/`, `ci/`, `bin/` | ambiente, CI, utilitários. |

Atenção ao lixo versionado na raiz (`*.php.orig`, `*.phpant`, `*.php.bk2403`, `*.php_ok`, `desktop-package.tar.gz`): são cópias antigas; **não editar**, o arquivo vivo é o `.php` sem sufixo.

---

## 3. Como uma tela funciona (ciclo de vida)

1. **Login** (`index.php`/`login.php`) escolhe instituição; sessão guarda `DB_id_usuario`, `DB_instit`, `DB_anousu`, `DB_datausu`, `DB_modulo`, `DB_id_depart`…
2. **Escolha do módulo** (`modulos.php`): grava `DB_modulo`/`DB_nome_modulo`, ano de exercício e registra em `db_usumod`/`db_usuariosonline`. O módulo define a árvore de menu e a "área" (`atendcadareamod`).
3. **Menu** montado do banco: `db_menu` / `db_itensmenu` (`libs/db_submenus.php`), filtrado por permissões por usuário/módulo/instituição/ano (`db_permissao`). **A árvore de menu não está no repositório** — vive na base. Os caminhos de menu só aparecem nos manuais.
4. **Tela** `xxx1_assunto001.php`: faz `require` das libs + `classes/db_*_classe.php`, carrega rótulos (`$oRotulo->label("campo")` → `$Lcampo`, `$Icampo`), inclui `forms/db_frm….php`, termina com `db_menu(...)`.
5. **Ações**: ou POST na própria página (inclusão/alteração/exclusão via DAO `cl_x->incluir/alterar/excluir`), ou AJAX para `….RPC.php` com `{exec: "acao", ...}`; resposta JSON `{status, message}` (status 2 = erro; mensagens vão em `urlencode`).
6. **Transações** explícitas: `db_inicio_transacao()` / `db_fim_transacao($lErro)`.
7. **Pesquisas**: campos com "âncora" abrem `func_*.php` em iframe-janela (`js_OpenJanelaIframe`).

Convenções de nome de coluna: `<sigla da tabela><n>_<campo>` (ex.: `as04_rendafamiliar`, `ht01_descricao`). A sigla identifica a tabela em qualquer SQL.

---

## 4. Blocos funcionais (macro) e processos de uso

### 4.1 Núcleo/transversal (usado por todos)
- **Configuração (DB):** instituições, departamentos (`db_depart`), usuários, permissões, menus, parâmetros, auditoria (`account`), layouts de arquivo (`db_layouttxt` — usado por importações), documentos/parágrafos/modelos.
- **CGM** (`prot1_cadcgm…`): **cadastro geral de pessoas físicas/jurídicas.** É a chave de pessoa em quase todos os módulos (tributário, compras, habitação…). Subtabelas: situação do CPF, composição familiar do CGM, etc.
- **Protocolo:** processos, tipos de processo, andamentos padrão, transferência/recebimento entre departamentos. Vários módulos (Habitação, Ouvidoria) criam processos aqui como "motor de fila de trabalho".
- **Avaliação / Questionários dinâmicos** (`avaliacao*`, `db_cadattdinamico`, `workflow*`): motor genérico de formulário com perguntas/respostas e de workflow por atividades. Reaproveitado por eSocial, Social, Habitação etc. (códigos de avaliação fixos, p.ex. `3000003` Família, `3000004` Cidadão).

### 4.2 Financeiro/contábil
Orçamento (PPA/LDO/LOA) → Empenho → Liquidação → Pagamento (Caixa/Tesouraria) → Contabilidade (lançamentos, balancetes, SICOM/SIAI/prestações). Fluxo típico: `orc` define dotações → `emp` empenha → `mat/com` geram ordem de compra → `cai` paga/arrecada → `con` contabiliza e emite relatórios legais. Inclui Custos, Planejamento, Acordos/Contratos (`aco`, `ac`).

### 4.3 Patrimonial
Compras (solicitação → orçamento → processo de compra), Licitações, Material/Almoxarifado, Patrimônio (bens, depreciação, incorporação), Veículos, Protocolo, Acordos.

### 4.4 Tributário
Cadastros (imobiliário `cad`, ISSQN `iss`, ITBI `itb`, água `agu`, cemitério `cem`, contribuição de melhoria) → Arrecadação (`arr`: débitos, parcelamento, recibos) → Caixa (baixa bancária) → Dívida Ativa (`div`, inicial/CDA, Jurídico `jur`) → Fiscal (`fis`: notificações, auto de infração, alvará).

### 4.5 Recursos Humanos
Cadastro do servidor (`rec`/`pes`), assentamentos, afastamentos, folha de pagamento (`pes`: cálculo, rubricas, empenhos da folha), eSocial (`eso`, `src/RecursosHumanos/ESocial`), recadastramento de aposentados.

### 4.6 Educação
Secretaria/Escola (`edu`): alunos, matrículas, turmas, diário, censo escolar, transporte escolar (`tra`), merenda (`mer`), biblioteca (`bib`). O aluno pode ter vínculo com o **cidadão** (tabela `alunocidadao`) — ponto de contato com o Social.

### 4.7 Saúde
UBS/ambulatorial (`sau`), agendamento, farmácia (`far`), laboratório (`lab`, integrações Luckmann/Infinity), vacinas (`vac`), TFD (`tfd`), família/microárea (ESF). Atenção: tabela `familia` (sd33) é da **Saúde** — não confundir com `cidadaofamilia` (as04) do **Social**.

### 4.8 Assistência social (foco do handoff 02)
Duas frentes: **Social** (`soc*`: CadÚnico, famílias, visitas, CRAS/CREAS, benefícios) e **Habitação** (`hab*`: programas, candidatos, ficha socioeconômica, inscrição, workflow de avaliação). Ver `02-handoff-assistencia-social.md`.

### 4.9 Cidadão / Ouvidoria / Portal
Ouvidoria (`ouv`) mantém a tabela **cidadão** (`ov02`), reusada pelo Social. Prefeitura Online (`pre`, `prefeitura_on_line`) e Portal da Transparência (`integracao_externa/portal_transparencia`, `src/PortalTransparencia`).

---

## 5. Padrão macro de uso (o que o operador faz, em qualquer módulo)

1. **Parametrizar** o módulo (menu Procedimentos/Parâmetros) — exercício, receitas, contas, textos.
2. **Cadastrar a base** (menu Cadastros): tipos, programas, locais, pessoas (CGM/cidadão).
3. **Operar** (menu Procedimentos): rotinas diárias, importações em lote, processamentos de fechamento.
4. **Consultar** (menu Consultas) e **emitir relatórios** (menu Relatórios, PDF via Agata/FPDF).
5. Trocar de **ano de exercício** no cabeçalho do módulo quando aplicável — muitos parâmetros são por `anousu`.

Todas as ações respeitam permissão por item de menu. Se "não aparece a opção", é permissão ou menu não cadastrado no banco, não código.

---

## 6. Como rodar / mexer

- Ambiente: `docker-compose up -d` na raiz (portas 8053/8056). Config sensível fora do git (`libs/db_conn.php`, `libs/db_acessa.php` — há `.dist`). `config/*.php` idem.
- Banco: **não há dump** no repo. Existem só migrations/DML incrementais (`db/`), então não dá subir uma base do zero só com este repo — é preciso uma base DBSeller existente.
- Para entender uma tela: abrir `xxx1_assunto001.php` → seguir o `forms/db_frm…` → seguir o `.RPC.php` → seguir `model/<modulo>/…` → conferir colunas em `classes/db_<tabela>_classe.php`.
- Para achar o caminho de menu de uma tela: procurar no `manuais/<modulo>/index.php` (texto) — a árvore real está no banco (`db_itensmenu`).
- Edição: encoding dos `.php` é **ISO-8859-1** (latin1). Use `iconv -f latin1 -t utf8` para ler e preserve o encoding ao salvar (há commit "Ajuste de css e encoding" no histórico).

---

## 7. Riscos e pegadinhas gerais

- **Legado pesado**: `register_globals`-style (`$HTTP_POST_VARS`, `session_register`), SQL concatenado, `addslashes`. Qualquer endpoint novo deve usar parâmetros/escape rigoroso.
- Há ferramentas administrativas que executam SQL arbitrário (ex.: `hab1_carga001.php` roda `db_query($_POST['db101_cargadados'])`) — dependem só de permissão de menu. [verificado]
- Duas "famílias": Social (`cidadaofamilia`) vs Saúde (`familia`) vs CGM (`cgmfamilia`). Três conceitos, três tabelas.
- Dois "cidadãos": `cidadao` (ouvidoria/social, `ov02`) e **CGM** (`z01`). Habitação usa CGM; Social usa cidadão (opcionalmente ligado a CGM via `cidadaocgm`).
- Menus e dados de referência (tipos, avaliações) estão no banco — mudar código sem a base correspondente quebra silenciosamente.
- Migração em curso para Laravel: não existe ainda ponte formal legado↔Laravel para Assistência Social.
