# Handoff — Assistência Social (Social + Habitação)

> Complementa `01-handoff-geral-e-cidade.md`. Gerado em 2026-10-15.
> **[verificado]** = li no código/manual. **[inferido]** = deduzido; confirmar com usuário-chave ou com a base.
> Limitação importante: a **árvore de menus e os dados de referência vivem no banco** (não no repo). Os caminhos de menu abaixo vêm dos manuais (`manuais/social`, `manuais/habitacao`) e dos nomes de arquivo.

---

## 0. Resumo em 10 linhas

- "Assistência Social" no e-Cidade = **dois módulos independentes** que compartilham só infraestrutura: **Social** (`soc*`) e **Habitação** (`hab*`). No scaffold Laravel já foram previstos como `app/Domain/AssistenciaSocial/{Social,Habitacao}` (vazios hoje).
- **Social** = cadastro e acompanhamento de **famílias e cidadãos** vulneráveis, **alimentado principalmente por arquivos do CadÚnico federal** (importados), com visitas de CRAS/CREAS, benefícios (SIBEC/Bolsa Família, BPC), cursos/oficinas e muitos relatórios.
- **Habitação** = **inscrição em programas habitacionais** (e outros benefícios): cadastra programas, pessoas interessadas (via CGM) com ficha socioeconômica e composição familiar, seleciona, inscreve e conduz cada inscrição por um **workflow de pareceres por departamento** (apoiado em processo do Protocolo).
- Pessoa-chave: Social usa **cidadão** (`ov02`, tabela da Ouvidoria); Habitação usa **CGM** (`z01`). **Não há ligação nativa entre os dois módulos** (confirmar se há integração por CPF/NIS fora do código).
- Ambos usam o motor genérico de **Avaliação** (perguntas/respostas) como "ficha".

---

## 1. Mapa de arquivos

### 1.1 Social (`soc*`) — 87 telas na raiz + `model/social/` + `manuais/social/`

| Grupo | Arquivos | Função |
|---|---|---|
| **soc1 – Cadastros** | `soc1_cidadao001/002` (+`forms/db_frmcidadao.php`), `soc1_localatendimento001`, `soc1_visitatipo001-003`, `soc1_cursosocial001`, `soc1_matriculacursosocial001` | Cidadão, Local de atendimento (CRAS/CREAS), Tipo de visita, Curso/Oficina social, matrícula em curso |
| **soc2 – Relatórios** | `soc2_*001/002` (par formulário/saída) | ~30 relatórios (ver §3.6) |
| **soc3 – Consultas** | `soc3_consultacidadao*`, `soc3_consultafamilia*`, `soc3_consulta*.RPC.php`, `soc3_consultaretornocidadao001`, `soc3_consultatelefonecidadao001` | Consulta de cidadão e de família (abas: dados, cadastro único, avaliação, benefícios, cursos, composição, CRAS/CREAS, visitas) |
| **soc4 – Procedimentos** | `soc4_importacaobasemunicipio001`, `…beneficiosibec001`, `soc4_importaaveriguacoes001`, `…bcp001`, `…revisaocadastral001`, `soc4_processaavaliacoes001`, `soc4_rotinadevisita001-003`, `soc4_rotinaatualizacao001`, `soc4_alteralocalatendimento001`, `soc4_encerraatendimento001`, `soc4_controlefrequencia001`, `soc4_*.RPC.php` | Importações, processamento, visitas, vínculo de local, frequência |
| **Modelo** | `model/social/Cidadao`, `Familia`, `FamiliaRepository`, `FamiliaVisita`, `LocalAtendimentoSocial`, `LocalAtendimentoFamilia`, `CursoSocial`, `TipoFamiliar`, `VisitaTipo`, `FormaRetorno`… e `model/social/cadastrounico/*` | Regras de negócio |
| **DAOs** | `classes/db_cidadao*`, `db_cadastrounico*`, `db_cursosocial*`, `db_localatendimento*`, `db_visitatipo`, `db_tipofamiliar`… | Tabelas |

> Atenção: `model/social/` também contém ~dezenas de `db_*_classe.php` de outros módulos (abatimento, acordo…) — são cópias soltas, ignorar.

### 1.2 Habitação (`hab*`) — 52 telas + `model/habitacao/` + `manuais/habitacao/`

| Grupo | Arquivos | Função |
|---|---|---|
| **hab1 – Cadastros** | `hab1_habitprograma001-006` (+`004`/`forms/db_frmhabitprograma.php`), `hab1_tipogrupoprograma*`, `hab1_tipogrupoprogramaformaavaliacao001`, `hab1_tipogrupoprogramaprocdoc001`, `hab1_formaavaliacao*`, `hab1_formaavaliacaousuario001`, `hab1_avaliacao001-006`, `hab1_avaliacaogrupopergunta001`, `hab1_avaliacaopergunta001`, `hab1_avaliacaoperguntaopcao001`, `hab1_workflow001-006`, `hab1_workflowativ001`+`.RPC`, `hab1_habitprogramalistacompra001`, `hab1_habitprogramalote001`, `hab1_cadastrocanditado001`, `hab1_cadastrosocioeconomico001/002/004`, `hab1_composicaofamiliar001`, `hab1_cadastroavaliacao001`+`.RPC`, `hab1_habitparametro001` | Programas, grupos, avaliações, workflow, candidatos |
| **hab2 – Consultas/Relatórios** | `hab2_inscricoes001/002` | Relatório de inscrições |
| **hab4 – Procedimentos** | `hab4_selecaoinscricao001`+`.RPC`, `hab4_contemplacao001`+`.RPC`, `hab4_inscricaodadesistencia001`, `hab4_inscricaocanditado.RPC`, `hab4_situacaocpf001`+`.RPC`, `hab4_programalistacompra.RPC` | Seleção/inscrição, lançamento de atividades (contemplação), desistência, situação do CPF |
| **Modelo** | `model/habitacao/{CandidatoHabitacao,InteresseHabitacao,InteresseProgramaHabitacao,InscricaoHabitacao,CidadaoRepository}` | Regras |
| **Outros** | `hab1_carga001.php` | ⚠️ Executa SQL livre vindo do POST (`db101_cargadados`) — ferramenta de carga; restringir acesso |

---

## 2. Modelo de dados (resumo, tirado de `classes/db_*_classe.php`)

### 2.1 Social

```
cidadao (ov02)  ──< cidadaotelefone (ov07) / cidadaoemail (ov08) / cidadaotiporetorno (ov04)
   │  ╰── cidadaofiliacao (ov29) (pai/mãe)      ╰── cidadaocgm (ov03) → CGM (opcional)
   │
   ├── cidadaocadastrounico (as02): NIS, apelido, codigounicocidadao, dataatualizacao
   │        ├── cadastrounicosituacao (as12) → tiposituacaocadastrounico: 1 Averiguação, 2 BPC, 3 Revisão Cadastral
   │        └── cidadaobeneficio (as08): programa, mês/ano competência, situação, NIS, tipo, datas, motivo…
   ├── cidadaoavaliacao (as01) → avaliacaogruporesposta   (Avaliação 3000004 – Cidadão)
   ├── cursosocialcidadao (as22) → cursosocial (as19) [+ aula, dia da semana, ausência]
   └── cidadaocomposicaofamiliar (as03: cidadao + tipofamiliar) ──> cidadaofamilia (as04)

cidadaofamilia (as04): dataentrevista, rendafamiliar (per capita), dataatualizacao, aparelhoeletricocontinuo
   ├── cidadaofamiliacadastrounico (as15): código familiar CadÚnico   → define "família é do CadÚnico"
   ├── cidadaofamiliaavaliacao (as06)  (Avaliação 3000003 – Família)
   ├── cidadaofamiliavisita (as05): data, hora, profissional (CGM), visitatipo (as13), observação
   │       ├── …visitacontato (as10)  ├── …visitaencaminhamento (as14 → CGM do local encaminhado)
   └── localatendimentofamilia (as23): local, datavinculo, fimatendimento, ativo, usuário
            └── localatendimentosocial (as16): tipo 1=CRAS 2=CREAS, descrição, departamento (db_depart)

cadastrounicobasemunicipal (as09): cópia crua (linha do TXT) por tipo/chave de registro — insumo do "Processar Avaliação"
```
[verificado] Tabelas de apoio: `tipofamiliar` (z14, também usada pelo CGM), `visitatipo` (flag "exige encaminhamento").

### 2.2 Habitação

```
programa:  habittipogrupoprograma (ht02) ──< habitgrupoprograma (ht03) ──< habitprograma (ht01)
              │ (documentos exigidos: ht09→protocolo procdoc)         │ ht01_workflow → workflow (db112)
              │ (formas de avaliação: ht06 → habitformaavaliacao ht07 + usuários ht08)   → workflowtipoproc (db116) → tipo de processo do Protocolo
              └ habitprogramaconcedente (ht19→CGM), habitprogramalote (ht05→lote cadastral), habitprogramalistacompra (ht17)+item (ht18→material/unidade)

candidato: habitcandidato (ht10 → CGM)
   ├── habitfichasocioeconomica (ht12) → avaliacaogruporesposta  (respostas do questionário)
   ├── cgmcomposicaofamiliar (z15) → composição familiar do CGM  (+ tipo familiar z14)
   ├── cgmsituacaocpf (z17): 1 Regular, 2 Irregular, 3 Suspenso
   └── habitcandidatointeresse (ht20: candidato × grupo, ativo)
           └── habitcandidatointeresseprograma (ht13: interesse × programa, ht13_codproc = nº do processo no Protocolo)
                   └── habitinscricao (ht15: usuário, data/hora, prioridade, lembrete)
                           └── habitinscricaocancelamento (ht22: tipo 1=cancelamento 2=desistência, motivo)

workflow: workflow (db112) ──< workflowativ (db114, ordem, depto via andamento padrão) ──< workflowativexec (db113: usuário, data, obs, concluído)
          (+ atributos dinâmicos por atividade: workflowativdb_cadattdinamico / workflowativexecucaoatributovalor)
          (+ liga a execução à transferência do processo: proctransferworkflowativexec)
habitparametro (ht16, por exercício): avaliação padrão, receita padrão, nº parcelas, dia de pagamento, meses de carência
```

---

## 3. SOCIAL — processo de uso, passo a passo

### 3.1 Visão do ciclo (o "porquê")

O Social **não é preenchido família a família manualmente**: a base nasce de **arquivos do governo federal** e o operador **atualiza, acompanha e emite documentos/relatórios** em cima dela. O trabalho manual é: visitas, vínculo com CRAS/CREAS, cursos, correção de cadastro de cidadão e emissão de declarações.

```
Federal (Caixa/MDS/SIBEC)           e-Cidade (Procedimentos)                       Saída
────────────────────────            ───────────────────────────────                ─────────────
Base CadÚnico (TXT, layout 188) ──►  Importação da Base Municipal                    Consultas
Situação de Benefícios (TXT, 189)─►  Importação de Benefícios (por competência)      Relatórios
Averiguações (TXT, layout 208) ───►  Importação de Averiguações   ┐                  Declarações
Arquivo BPC (TXT)  ───────────────►  Importação do Arquivo BPC    ├ situações do     (Tarifa Social,
Revisão Cadastral (TXT, 210) ─────►  Importação da Revisão Cad.   ┘ cidadão          INSS)
                                  ►  Processar Avaliação Sócio-Econômica  (alimenta relatórios cadastrais)
Equipe CRAS/CREAS ────────────────►  Rotina de Visita / Atualização da Família / Local de Atendimento
```

### 3.2 Pré-requisitos (uma vez / quando mudar)
1. **Local de Atendimento** (Cadastro → `soc1_localatendimento001`): cada CRAS/CREAS é um **departamento** (`db_depart`) com tipo 1=CRAS ou 2=CREAS e descrição. A tela puxa o departamento do usuário logado (provavelmente é preciso estar no departamento do CRAS/CREAS) [inferido].
2. **Tipo de Visita** (`soc1_visitatipo001-003`): descrição + "exige encaminhamento" (se marcado, a visita obriga informar o local de encaminhamento). [verificado]
3. **Cursos/Oficinas** (opcional) (`soc1_cursosocial001`): nome, tipo de curso, período, horas/dia, ministrante e responsável (CGMs), dias da semana e agenda de aulas; **matrícula** de cidadãos em `soc1_matriculacursosocial001`; frequência em `soc4_controlefrequencia001`.
4. **Layouts de importação** (códigos 188, 189, 208, 210 em `db_layouttxt`) e **programas sociais** devem existir no banco — sem eles a importação falha. [inferido]

### 3.3 Rotinas de importação (Procedimentos → Cadastro Único)

| Rotina (tela) | Entrada | O que faz | Observações |
|---|---|---|---|
| **Importação da Base Municipal** (`soc4_importacaobasemunicipio001`, `ImportacaoCadastroUnico`) | TXT da base municipal solicitada à Caixa (layout 188) | Lê linha a linha por tipo de registro: `00` header (datas de posição/extração), `01` família (cria/atualiza `Familia` por código familiar; endereço; data entrevista; renda per capita = `vlr_renda_media/100`), `04` pessoa (cria/atualiza `CadastroUnico`+cidadão; NIS; parentesco→`tipofamiliar`), `05` documentos, `09` telefones, `06/07/08/11/12` apenas guardados. | **Apaga e recria** `cadastrounicobasemunicipal` (restart sequence) a cada importação. Pessoa sem NIS recebe NIS sintético `S<família><ordem>`. Cidadão entra com situação 2 e ativo. Só `.txt`. |
| **Importação de Benefícios** (`…beneficiosibec001`, `ImportacaoBeneficiosCidadao`) | Relatório SIBEC "Situação de Benefícios (Automático)" (layout 189) | Descobre a **competência mês/ano** do arquivo, avisa se já existe e **remove os benefícios da competência** antes de regravar; grava em blocos de 100 numa transação; ignora linhas sem NIS. | Fonte: SIBEC → Relatório → Sintético → Acompanhamento Operacional → Situação de Benefícios (Automático). |
| **Importação de Averiguações** (layout 208) | TXT de averiguação | Marca o cidadão (por NIS) com situação `1 – Averiguação`. Aparece nas consultas de cidadão e família. | Risco de cancelamento do benefício. |
| **Importação do Arquivo BPC** | TXT BPC | Situação `2 – BPC`. | Alimenta relatório de idosos/deficientes com BPC. |
| **Importação da Revisão Cadastral** (layout 210) | TXT revisão | Situação `3 – Revisão cadastral`. | |
| **Processar Avaliação Sócio-Econômica** (`soc4_processaavaliacoes001`) | (usa `cadastrounicobasemunicipal`) | Conta cidadãos/famílias **sem avaliação atualizada** e processa: monta a **Avaliação** (3000004 cidadão / 3000003 família) a partir das linhas cruas da base. | **Obrigatório após cada importação de base** para os relatórios cadastrais (guarda de criança, deficiência, zona, trabalho formal, EJA, mulher chefe de família, raça/cor). |

Ordem recomendada: **Base → Processar Avaliação → Benefícios (por competência) → Averiguação / BPC / Revisão**. [inferido do manual + dependências]

### 3.4 Rotinas do dia a dia (Procedimentos)

- **Rotina de Visita** (`soc4_rotinadevisita001`): escolher família → data/hora, **profissional que fez a visita** (CGM), **tipo de visita**, observação (relatório de acompanhamento) e, se o tipo exigir, **local de encaminhamento** (CGM) → grava `cidadaofamiliavisita` (+ encaminhamento/contato). Telas `002/003` = alterar/excluir.
- **Rotina de Atualização da Família** (`soc4_rotinaatualizacao001`): registra a **data de atualização do formulário manuscrito do CadÚnico** (`as02_dataatualizacao`) para a família selecionada. Alimenta o filtro do relatório "Cadastro Único".
- **Local de atendimento da família**: `soc4_alteralocalatendimento001` troca a família de CRAS/CREAS — o sistema **encerra o vínculo ativo anterior** (fim = data do novo vínculo) e abre outro; `soc4_encerraatendimento001` só encerra (exige data de fim ≥ vínculo + motivo). Histórico em `localatendimentofamilia`. Regra: **uma família tem no máximo um vínculo ativo**. [verificado em `LocalAtendimentoFamilia::salvar`]
- **Cidadão** (`soc1_cidadao001/002`): cadastro/alteração manual (nome, CPF, RG, endereço, data nasc., sexo, telefones, e-mails, filiação, formas de retorno, avaliação). Serve para quem não veio do CadÚnico. O cadastro de cidadão é o **mesmo da Ouvidoria**.

### 3.5 Consultas (Consulta → Cidadão / Família)
- **Cidadão**: dados pessoais; se tem registro no CadÚnico; avaliações socioeconômicas; benefícios (federais/estaduais/municipais); cursos/oficinas; sinalizadores "em averiguação", "recebe BPC", "em revisão cadastral".
- **Família**: localização, composição familiar, benefícios, avaliação, vínculo CRAS/CREAS, visitas.

### 3.6 Relatórios e documentos (Relatórios)
- **Cadastro Único**: lista A–Z de famílias filtrando por data de atualização do CadÚnico e/ou data de visita.
- **Documentos** (vão ao **Agata**, `dbagata`): **Declaração Tarifa Social** (concessionária de energia; válida se renda per capita ≤ ½ salário mínimo; senão "Família sem direito ao benefício!") e **Declaração INSS** (renda mensal até 2 salários mínimos, solicitante com NIS e sem renda; senão mesma mensagem). Regra em `Familia::validaRendaMensalAte` (salários mínimos vêm de parâmetro; renda ≤ 0 conta como válida).
- **Benefícios**: por família / famílias por benefício — exige selecionar **mês/ano de competência**.
- **Cadastrais**: famílias por bairro; parentes que detêm guarda da criança; pessoas com deficiência; famílias por zona; famílias com trabalho formal; pessoas EJA; mulheres chefe de família; famílias por raça/cor; idosos com BPC; atendidas pelo CRAS/EAS; idosos/deficientes com BPC; crianças < 18 com benefício ativo. Os cinco primeiros dependem do **Processar Avaliação**.

### 3.7 Pontos de atenção do Social
- **Quase tudo depende da qualidade do arquivo federal.** Ao reimportar, famílias já existentes (mesmo código familiar) são atualizadas pelo arquivo (endereço, entrevista, renda), podendo sobrescrever ajustes locais — validar em homologação.
- Benefícios são **por competência**; reimportar substitui a competência inteira.
- Famílias são identificadas pelo **código familiar do CadÚnico** (`as15`) → `Familia::isCadastroUnico()`. Família sem esse código = cadastro manual.
- BPC/Quilombola/Indígena são lidos de **perguntas do formulário F1.01 já processadas** (respostas por código fixo, ex.: `3000469`, `3000470`) — mudança de layout federal quebra os relatórios.
- Sem testes automatizados para este módulo [verificado: `tests/` não referencia social/habitação].

---

## 4. HABITAÇÃO — processo de uso, passo a passo

### 4.1 Visão do ciclo

```
 PREPARAR                       CADASTRAR                SELECIONAR/INSCREVER         TRAMITAR                FECHAR
 ────────                       ─────────                ────────────────────         ────────                ──────
 Parâmetros                     Candidato (CGM)          Seleção                      Lançamento de           Última atividade
 Tipo de Grupo de Programa      + Ficha Sócio-Econ.      (checa CPF Regular,          Atividades              concluída → interesse
 Grupo de Programa              + Composição familiar    cria interesse+processo)     (parecer por depto,     arquivado
 Forma de Avaliação/usuários    + Interesse (grupo/prog) → Inscrição (nº, prioridade) Concluído S/N)           ─ ou ─
 Avaliação (questionário)                                                                                      Desistência/Cancelamento
 Workflow (atividades/deptos)
 Programa (+lista de compras, lotes, concedente)
```

### 4.2 Preparação (menu Cadastro / Procedimentos) — ordem importa

1. **Parâmetros da Habitação** (`hab1_habitparametro001`, Procedimentos): por exercício — avaliação padrão, receita padrão, nº máximo de parcelas, dia padrão de pagamento, meses de carência. Os programas herdam esses valores.
2. **Cadastro de Avaliação** (`hab1_avaliacao001-006`): é o **questionário socioeconômico**. Abas: *Avaliação* (tipo, descrição, ativo) → *Grupos* de perguntas → *Perguntas* (tipo de resposta: objetiva, dissertativa, múltipla escolha; ordem; obrigatória; ativa) → *Respostas* (opções, "aceita texto").
3. **Forma de Avaliação** (`hab1_formaavaliacao*`): define quem avalia — dados (período) + **usuários responsáveis** (`ht08`).
4. **Tipo de Grupo de Programa** (`hab1_tipogrupoprograma*`): abas *Grupo*, **Documentos** (tipos de documento do **Protocolo** que o candidato deve apresentar; marcar obrigatório) e **Avaliações obrigatórias** (formas de avaliação exigidas). Depois o **Grupo de Programa** (`ht03`) é criado dentro do tipo.
5. **Workflow** (`hab1_workflow001-006` + `hab1_workflowativ001`): *Workflow* (descrição + descrição do **tipo de processo do Protocolo** que ele cria — grava `workflow`+`workflowtipoproc`) e *Atividades* (descrição + **departamento** onde o processo deve ficar; **ordenar** define o fluxo; "Lançar atributos" cria campos extras a preencher na execução). ⚠️ **Depois que o workflow é ligado a um programa e há inscritos, a ordem das atividades não pode mais ser alterada.**
6. **Programa** (`hab1_habitprograma001`, abas *Dados do Programa* e *Lista de Compras*): grupo, descrição, concedente (CGM), **controle de participação** (sem aviso/sem bloqueio · com aviso/sem bloqueio · com aviso e bloqueio), **controle de quantidade** (sem limite · máxima determinada + nº de beneficiados), validade, descrição do contrato, lei, exige assinatura do concedente, exige CPF, parcelas/dia/receita de pagamento, **código do workflow**. Lista de compras: itens (material pcmater + unidade + quantidade) com data limite e forma de avaliação. Programa só pode ser **excluído sem candidatos**.

### 4.3 Cadastrar o candidato (Cadastro → Cadastro Sócio-Econômico)

`hab1_cadastrosocioeconomico001` abre 3 abas:
1. **Grupo/Programa** (`…004`): escolher/criar o **CGM** (pessoa física) do interessado — `hab1_cadastrocanditado001` embute o cadastro geral (`prot1_cadgeralmunic001/002`) —; selecionar grupo(s) e, opcionalmente, programa(s) de interesse; informar **situação do CPF** (Regular / Irregular / Suspenso).
2. **Composição Familiar** (`hab1_composicaofamiliar001`): cada familiar com tipo de parentesco, nome/CGM, profissão, escolaridade, nascimento, sexo, renda (`cgmcomposicaofamiliar`).
3. **Cadastro Sócio-Econômico** (`hab1_cadastroavaliacao001`): escolher o **grupo de perguntas**, responder, *Salvar Perguntas* e *Salvar Cadastro*.

Ao salvar (`CandidatoHabitacao::salvar`): cria/atualiza `habitcandidato` e **apaga e recria** a ligação com a ficha (`habitfichasocioeconomica`) — a ficha sempre aponta para o último grupo de respostas. `addInteresseGrupo` só cria interesse se não houver um ativo para o mesmo grupo.

### 4.4 Seleção e inscrição (Procedimentos → Seleção) — `hab4_selecaoinscricao001`

1. Escolher **Grupo** e (opcional) **Programa**. "Nenhum" = só quem tem interesse apenas no grupo.
2. **Pesquisar interessados** → lista com *Ficha* (abre a ficha sócio-econômica).
3. Escolher o **Programa p/ Inscrição**, marcar candidatos e **Gerar Inscrição**.

Regras na geração (`GerarInscricao`, tudo numa transação — **um erro cancela o lote inteiro**):
- Exige **situação do CPF = Regular (1)**. Irregular, suspenso ou não informado → erro nominal e nada é gravado.
- Se o interesse era só de grupo → cria interesse em programa + **cria o processo no Protocolo** (tipo vindo do workflow do programa; requerente = CGM; aborta com "Tipo de processo não cadastrado" se o workflow não tiver tipo). Se já era do programa, só gera a inscrição.
- `InscricaoHabitacao::salvar` grava a inscrição (usuário, data, hora, prioridade, lembrete "Inscrição gerada por seleção") e **já faz a primeira transferência e recebimento** do processo para o 1º departamento do andamento padrão.

> O **controle de participação** e o **controle de quantidade** do programa estão cadastrados mas não vi a verificação deles no caminho de `GerarInscricao` — confirmar onde são aplicados (provável: no formulário de interesse `hab4_inscricaocanditado.RPC`). [inferido — conferir]

### 4.5 Tramitação (Procedimentos → Lançamento de Atividades) — `hab4_contemplacao001`

Cada usuário vê **só os processos que estão no seu departamento** (filtro por departamento logado + período de envio). Clicando na atividade:
- Informa **Obs (parecer)** e **Concluído = SIM/NÃO** (+ atributos dinâmicos, se houver) → grava `workflowativexec`.
- **SIM**: se existe próximo departamento no andamento padrão → **transfere e recebe** o processo para ele e registra o vínculo `proctransferworkflowativexec`. Se **era a última atividade** → o interesse do candidato é **cancelado/arquivado** ("Arquivamento conforme: <parecer>") — é assim que o fluxo "termina" (não existe um status "contemplado" separado; a contemplação é o fim do workflow). [verificado em `hab4_contemplacao.RPC.php`]
- **NÃO**: cancela o interesse no programa e **devolve o candidato ao estado "interesse só no grupo"** (cria novo interesse de grupo) — pode ser selecionado de novo.

### 4.6 Desistência / cancelamento
- **Inscrição em Programas → Desistência** (`hab4_inscricaodadesistencia001`): lança desistência (tipo 2) com motivo em `habitinscricaocancelamento`; cancela o interesse no programa, arquiva o processo e, por padrão, recria o interesse de grupo (`$lGeraInteresse = true`).
- Cancelamento (tipo 1) é o caminho usado por processos internos.

### 4.7 Consultas e relatórios
- **Consulta → Processo** (Protocolo): acompanha o andamento por nº de processo ou nome do requerente.
- **Relatório de Inscrições** (`hab2_inscricoes001/002`): filtra por programa, candidato e inscrição.
- **Situação CPF** (`hab4_situacaocpf001`): regulariza o CPF depois (necessário para voltar a ser inscrito).

### 4.8 Pontos de atenção da Habitação
- **Dependência do Protocolo**: o motor de fila é o processo do Protocolo. Departamentos do workflow precisam existir e ter pessoas com permissão de receber/lançar; tipo de processo e andamento padrão (`andpadrao`) precisam estar cadastrados, senão a inscrição falha ou o processo trava.
- Mudar workflow de programa **com inscritos** é arriscado (ordem travada).
- Ficha socioeconômica é **substituída** a cada "Salvar Cadastro" (não versiona respostas antigas).
- `ht13` aparece como prefixo em duas tabelas (`habitsituacaoinscricao` e `habitcandidatointeresseprograma`) — cuidado ao ler SQL; na prática, `ht13_*` nas telas = interesse no programa.
- Tabelas `habitsituacaoinscricao*`, `habitprogramalote` e lista de compras existem, mas **não vi fluxo operacional completo** que as use além do cadastro (provável parte da evolução incompleta do módulo — confirmar com negócio).
- Módulo opera por **CGM**; não há importação nem reaproveitamento do cidadão do Social.

---

## 5. Relação entre as duas frentes

| Aspecto | Social | Habitação |
|---|---|---|
| Pessoa | `cidadao` (ov02), pode ligar a CGM (`cidadaocgm`) | `CGM` (z01) |
| Família | `cidadaofamilia` + composição por cidadão | `cgmcomposicaofamiliar` por CGM |
| Origem dos dados | Arquivos federais importados + cadastro manual | Cadastro manual pelo atendimento |
| Ficha | Avaliações 3000003/3000004 (geradas da base) | Avaliação criada em `hab1_avaliacao` (livre) |
| Motor | Rotinas + relatórios | Workflow sobre Protocolo |
| Quem usa | Equipe CRAS/CREAS, gestão social | Setor de habitação/engenharia/assistente social |
| Integração entre si | **Nenhuma encontrada no código** | idem |

Oportunidade/lacuna natural: cruzar candidato (CGM→CPF) com cidadão/NIS (Social) para pré-qualificar renda e CadÚnico. Hoje isso seria manual.

---

## 6. Para quem for evoluir/manter

1. **Onde mexer**: tela `soc*/hab*` → `forms/db_frm…` → `.RPC.php` → `model/social|habitacao`. Colunas em `classes/db_<tabela>_classe.php` (bloco `$campos`).
2. **Constantes mágicas**: layouts 188/189/208/210; avaliações 3000003/3000004; situações de CadÚnico 1/2/3; tipo de local 1=CRAS 2=CREAS; situação CPF 1/2/3; tipo de parentesco "14" como padrão e mapeamento `cod_parentesco_rf_pessoa → tipofamiliar` em `ImportacaoCadastroUnico::adicionarCidadao` (de-para CadÚnico↔e-Cidade). Documentar/centralizar antes de refatorar.
3. **Encoding latin1** e CRLF nos arquivos legados — use `iconv` e preserve.
4. **Laravel**: `routes/web/assistencia-social/*.php` e `app/Domain/AssistenciaSocial/*` são pontos de entrada previstos; nada implementado. Se for migrar, começar pelos **relatórios/consultas de leitura** do Social (menor risco) e manter importações/workflow no legado até haver testes.
5. **Testes**: inexistentes para estes módulos. Prioridade: `ImportacaoCadastroUnico` (parsing por tipo de registro), `Familia::validaRendaMensalAte`, `LocalAtendimentoFamilia::salvar`, e `GerarInscricao`.
6. **Segurança**: revisar `hab1_carga001.php` (SQL livre), concatenação de SQL nos RPCs (`addslashes`), e a leitura de arquivos por caminho enviado pelo cliente (`$oParam->arquivo` nas importações do Social).

## 7. Perguntas em aberto (confirmar com usuário-chave / base)

1. Qual a árvore de menu real dos módulos Social e Habitação (para alinhar com os caminhos dos manuais)?
2. Controle de participação / quantidade de beneficiados é aplicado em qual ponto?
3. Existe processo definido para "contemplação" além de concluir a última atividade (ex.: geração de parcelas/receita usando `ht01_receitapadraopagamento`)? Não achei código que gere cobrança.
4. O município usa de fato `habitprogramalote` e `lista de compras`?
5. Há integração CadÚnico↔Habitação feita fora do sistema (planilhas)?
6. Programas sociais (`as08_programasocial`) e `tiposituacaocadastrounico`: quais registros existem na base de produção?
