# M1 genérico — prova de conceito

**Status:** prova de conceito funcional. Nenhum arquivo central da Fábrica foi tocado (`app.py`, `criar_animado.py`, `engine_*`, adaptadores, `catalogo.json`, `roteiro.py`, `nucleo.py`): `diff -rq` contra o `estudio/` original mostra só arquivos **novos**, e os estilos Telejornal/Whiteboard continuam renderizando numa cópia limpa com os arquivos sobrepostos.

## 1. Resultado em poucas linhas

- O M1 agora monta cenas **a partir de JSON**: cenário, elementos, linha do tempo, câmera, HUD e legenda. Os recursos visuais (personagens, objetos, cenários) vêm de **arquivos de dados** (JSON ou SVG) ou de **definições inline no próprio roteiro**.
- O motor tem **zero código específico** de tigre, homem, TV, sofá ou vaso (verifiquei com `grep`: só aparecem em comentários de exemplo). Um recurso novo é um arquivo novo, não uma função nova.
- Prova com os três elementos pedidos: **tigre** (22 partes articuladas, caminhada, rugido), **TV** (arquivo **SVG** real + JSON de animação: ligar/desligar) e **homem sentado num sofá que se levanta** (pés plantados). Mais um vaso criado só por JSON inline, para provar o caminho "recurso inexistente".
- Vídeos de 22 s em **9:16 e 16:9** (3 cenas). Render médio de **0,39–0,42 s/quadro** (mais rápido que as cenas à mão do protótipo anterior).
- 12 testes automatizados passam (parser SVG, SVGs maliciosos, JSON inválido, biblioteca/aprovação, tolerância a erros).

## 2. O que já existia e o que foi aproveitado

| Aproveitado sem mudança | Descartado como caminho | 
|---|---|
| `camadas.py` (Sprite, Mundo, parallax, bloom, vinheta, luz aditiva) | `objetos.py`, `cenarios.py`, `cenas.py`: cada objeto/cena era uma função Python específica — é exatamente o que **não** escala |
| `pintura.py` (Lona com supersampling, gradientes, brilho) | |
| `comum/` (tween, HUD, legendas), pipeline de quadro da `CenaBase`, contrato `Estilo` | |

A **menor evolução** foi acrescentar uma camada de **recursos + rig + cena genérica** por cima, sem reescrever o motor.

## 3. Arquitetura adicionada (≈ 2 050 linhas)

```text
estudio/estudio/cenas2d/
├── recursos/
│   ├── svgpath.py      parser seguro de caminhos SVG (M L H V C S Q T A Z, relativos/absolutos) + transformações
│   ├── svg.py          SVG -> recurso (<g id> vira parte; gradientes; sidecar .json completa hierarquia/animações)
│   ├── validar.py      valida/normaliza recursos (tipos, limites, ciclos, cores, caminhos); erro => RecursoInvalido
│   └── biblioteca.py   biblioteca em disco: registrar, deduplicar por hash, versionar, aprovar/rejeitar, prévia, galeria HTML
├── rig.py              partes + pivôs + hierarquia, animações por keyframes (Catmull-Rom), mistura entre clipes,
│                       desenho vetorial com recorte, gradientes, brilhos, âncoras, slots, luz por parte, cenários por camadas
└── cena_generica.py    modelo "generica": interpreta o JSON da cena
estudio/m1_recursos.py  CLI da biblioteca (registrar, listar, aprovar, rejeitar, galeria, previa)
estudio/cenas2d_proto.py  CLI de render (ganhou --biblioteca, --registrar, --aprovacao, --exigir-aprovados, --galeria)
```

**Como a generalidade funciona**
1. **Recurso = dados.** Partes em hierarquia, cada uma com pivô, ordem de profundidade e formas (path SVG, polígono, retângulo, elipse, linha, brilho), cores sólidas ou gradientes, **recorte pelo corpo** (listras do tigre ficam dentro do corpo), **variantes** de forma (olho aberto/fechado, boca neutra/sorriso/surpresa) e **animações** (faixas de keyframes por parte: rotação, translação, escala, opacidade, variante).
2. **Rig genérico.** Matriz da parte = matriz do pai · translação(pivô) · rotação · escala · translação(−pivô). Poses de vários clipes são **misturadas** (transição parado→caminhar sem salto). Cada quadro re-rasteriza só a área ocupada.
3. **Sincronia de passos.** O ciclo de caminhada pode ser ligado à **distância percorrida** (`"sincronizar": true`), então o tigre não "patina" mesmo acelerando e freando.
4. **Interação por âncoras.** O homem é "ancorado" no ponto `chao` do sofá (`"ancorar"`): mover o sofá move o homem. Luzes nascem das partes (a TV acende o ambiente conforme a opacidade do brilho da tela).
5. **Slots (reuso aninhado).** A TV declara um slot `tela`; a cena 3 coloca o **mesmo tigre** andando dentro dela, recortado pela tela.
6. **Capacidades e fallback.** O motor lista as animações de cada recurso (`capacidades()`), aceita aliases (`correr`→`caminhar`, `sentar`→`sentado`…) e, se o pedido não existir, usa `parado` ou a pose base **com aviso** — o vídeo não quebra.
7. **Composição por formato.** Posições, escalas e enquadramento do cenário são declarados **por formato** (`{"9x16": …, "16x9": …}`); legendas respeitam as áreas seguras.

### Tecnologias avaliadas
| Opção | Decisão |
|---|---|
| **Pillow + parser próprio de SVG/paths** | **Adotado.** Sem dependências novas, controle total, ~25 ms para reamostrar camadas |
| SVG para formas/curvas/grupos | **Adotado** (subset: path/rect/circle/ellipse/line/polygon/polyline, transform, gradiente linear, viewBox) |
| JSON para rig, animação, cena | **Adotado** (SVG não tem articulação nem animação utilizável por nós) |
| cairosvg / resvg / Skia | Não adotados: exigem DLLs/binários no Windows; o subset acima cobre o necessário. Podem entrar **depois** como *conversor offline* de SVG complexos |
| Lottie / Spine / DragonBones | Não adotados: formatos de runtime próprios; o JSON atual já cobre rig 2D e fica sob nosso controle |
| Blender / modelos de imagem locais | Fora do escopo (como pedido) |

## 4. Prova de conceito: o que foi avaliado

Os recursos foram **autorados por mim** (simulando o que uma IA/ilustrador entregaria) com scripts offline em `exemplos/recursos_poc/_autoria/` (cálculo de keyframes e cinemática inversa **só para gerar os números**). Esses scripts não fazem parte do motor. **Não testei SVGs gerados por uma IA externa de verdade** — isso é o próximo teste necessário.

| Critério | Avaliação honesta |
|---|---|
| **Qualidade dos desenhos** | **Boa para estilo vetorial plano com luz.** Tigre reconhecível (listras recortadas, cauda segmentada, orelha, olho, bigodes), TV e cenários bonitos. **Fracos:** mãos e rosto do homem simples; patas do tigre "bolas"; sem contorno/textura de ilustrador |
| **Separação das partes** | **Boa.** Tigre: 22 partes (4 pernas de 3 segmentos, cauda de 3, cabeça, mandíbula, orelhas, olho). Homem: 18 partes (braços/pernas de 3 segmentos, torso, cabeça, olho, boca, controle). Variantes para olho/boca |
| **Naturalidade dos movimentos** | **Média-boa.** Caminhada de marcha lateral legível, balanço de corpo/cauda/cabeça, piscar. Sentar→levantar com **pés plantados** funciona. **Fraco:** rigidez nas pernas, sem rotação de cabeça nem caminhada do homem; interação com a TV é "apontar o controle", não toque físico |
| **Composição e profundidade** | **Boa:** 6–9 camadas de cenário com parallax, sombra de contato, luz da TV/abajur/pôr do sol, câmera que segue o tigre. **Fracos no 9:16:** as cenas 2 e 3 deixam ~40 % superior quase vazio (parede escura) e, na cena 1, a cauda do tigre é cortada pela borda esquerda |
| **Reuso** | **Comprovado:** tigre usado na cena 1 e dentro da TV (cena 3); sofá/homem/TV/cenário reusados nas cenas 2 e 3; vaso inline registrado 1× e reconhecido como *reutilizado* na cena 3 |
| **9:16 e 16:9** | **Recomposto de verdade** (posições, escalas, enquadramento, TV mais longe no vertical). Ainda exige ajuste manual de coordenadas por formato |

## 5. Desempenho (30 fps, 22 s = 660 quadros; sandbox de 1 núcleo)

| Formato | Total | Média/quadro | Cena 1 | Cena 2 | Cena 3 | Pico de memória |
|---|---|---|---|---|---|---|
| 9:16 | 271 s | 0,393 s | 0,431 | 0,327 | 0,431 | 420 MB |
| 16:9 | 290 s | 0,422 s | 0,423 | 0,354 | 0,498 | 395 MB |

Projeção (média incl. codificação 0,41 / 0,44 s/quadro): **60 s ≈ 12–13 min · 90 s ≈ 19–20 min · 300 s ≈ 61–66 min** a 30 fps por formato; a 24 fps, ×0,8. Paralelizar por faixa de quadros (não feito) é o maior ganho disponível.

## 6. Segurança e robustez (testado)

- **Nenhum código é executado**: só se lê dado. `eval/exec/import` não existem no caminho; campos desconhecidos (`__import__`, `exec`) são **ignorados com aviso**.
- **SVG:** rejeitados `DOCTYPE/ENTITY` (ataque de expansão), `<script>`, `<foreignObject>`, `<image>`, `href` externo e atributos `on*`; `<use>/<clipPath>/<mask>/<filter>/<text>/<style>` são ignorados **com aviso**. Limites de tamanho (2 MB), nós, comandos e pontos.
- **JSON:** tipos, intervalos finitos, ids, ciclos de hierarquia, limites de partes/formas/keyframes; caminho SVG mal formado é rejeitado na validação.
- **Na cena:** recurso inexistente/inválido → **substituto visível** (caixa com X) + aviso; animação inexistente → alias/`parado`/pose base + aviso; elemento com erro é ignorado sem derrubar o vídeo. Verificado no teste `test_erros_nao_derrubam_o_render` (tigre pedindo `voar`, ref `dragao` inexistente, SVG com `<script>`, recurso novo inline).
- **Não coberto:** isolamento por processo/limite de tempo e memória para recursos hostis (hoje só há limites de tamanho/quantidade). Recomendo rodar a validação em subprocesso quando os recursos vierem de fora.

## 7. Biblioteca e aprovação

- `registrar` valida e guarda; **o mesmo desenho com outro id não duplica** (hash do conteúdo, ignorando id/nome/tags); mudou o conteúdo ⇒ **nova versão**.
- Política **configurável**: `--aprovacao auto` (produção em massa: entra já aprovado) ou `manual` (entra **pendente**); `--exigir-aprovados` bloqueia o uso de pendentes/rejeitados.
- Cada render grava `<saida>.recursos.json` (novo/reutilizado/versão/status por recurso + avisos).
- **Revisão:** `galerias/*.html` mostra, por recurso, prévia (pose base + 3 instantes de cada animação, ou camadas do cenário), selo **NOVO/REUTILIZADO** e **APROVADO/PENDENTE**, com os comandos de aprovar/rejeitar. Também `m1_recursos.py previa/galeria/aprovar/rejeitar`.
- **Interface da Fábrica (proposta, não implementada — não toquei em `app.py`/`ui`):** `GET /api/projetos/{id}/recursos` (lê o JSON acima), `GET /api/recursos/{id}/previa.png`, `POST /api/recursos/{id}/aprovar|rejeitar|nova-versao`, e `aprovacao: auto|manual` no `canal.json`. Reaproveita o que já existe: prévia, status e histórico.

## 8. Limitações e riscos (o que ainda falta)

1. **Falta validar com SVGs reais de IA/bibliotecas** (gradiente radial, `clipPath`, `mask`, filtros, texto, `<use>` não são suportados). O importador avisa, mas o desenho pode sair incompleto.
2. **Animações vêm prontas no recurso.** Não há *retargeting* nem cinemática inversa no motor (o IK só foi usado para gerar keyframes). SVG "puro" sem sidecar vira um objeto **estático** (nesse caso a IA precisa fornecer o JSON de articulação, ou implementamos auto-rig simples).
3. **A IA que dirige a cena precisa conhecer o vocabulário** de animações de cada recurso (`capacidades()` já expõe isso para ser enviado ao prompt).
4. **Layout automático ausente:** coordenadas por formato ainda são escritas na cena (ou pela IA). Falta um resolvedor ("chão", "perto de X", "à esquerda do sofá").
5. **Interação física simples:** âncoras e linha do tempo, sem colisão/contato (o homem não toca a TV de fato).
6. **Visual:** vazio no topo e corte de cauda no 9:16 (acima), HUD com texto pequeno em alguns casos, transição entre cenas só por escurecimento, sem áudio/alinhamento de narração.
7. **Desempenho:** ~19 min para 90 s por formato; precisa de paralelização antes de produzir em escala.
8. **Memória:** cenário em camadas é pré-renderizado por cena; vídeos longos precisam de cache LRU.
9. **Fonte Inter (OFL):** copiar o texto oficial da licença para `assets/fontes/` antes de distribuir.

## 9. Próximos passos sugeridos (ordem)

1. Testar com **5–10 SVGs reais** de IA/bibliotecas livres e ampliar o importador (gradiente radial, `clipPath`, `<use>` interno).
2. **Auto-rig mínimo** para SVG sem JSON (agrupar por `<g>`, pivôs, animações genéricas: balançar, pulsar, entrar/sair) — dá movimento a qualquer objeto.
3. **Paralelizar o render** e cachear sprites/luzes.
4. **Resolvedor de layout** (posições semânticas → coordenadas por formato).
5. Integração com a interface (endpoints acima) e com o fluxo da Fábrica (`criar_animado.py`/adaptador) **somente após sua autorização**.

## 10. Como executar

```bash
cd estudio
python m1_recursos.py registrar exemplos/recursos_poc --biblioteca biblioteca_m1
python cenas2d_proto.py exemplos/cenas2d_generico_poc.json saida_9x16.mp4 --formato 9x16 --biblioteca biblioteca_m1
python cenas2d_proto.py exemplos/cenas2d_generico_poc.json saida_16x9.mp4 --formato 16x9 --biblioteca biblioteca_m1
python cenas2d_proto.py ... --cena 2          # só a cena 2 (render em segmentos; juntar com ffmpeg -f concat)
python cenas2d_proto.py ... --aprovacao manual --galeria revisao.html     # recursos novos entram PENDENTES
python m1_recursos.py aprovar tigre | rejeitar tigre "motivo" | listar --status pendente
python -m unittest testes.teste_recursos -v
```

---

## Apêndice — formato do recurso (`m1-recurso/1`)

```jsonc
{ "id": "tigre", "tipo": "personagem|objeto|cenario", "nome": "…", "tags": [], "versao": 1,
  "caixa": [x0, y0, x1, y1],            // limites no espaço do desenho (mesma régua para todos: ~3 unidades = 1 cm)
  "base": [x, y],                       // ponto de apoio (pés / chão) que é posicionado na cena
  "frente": "direita|esquerda",         // para espelhar automaticamente
  "passo": 330,                         // unidades por ciclo de caminhada (sincroniza passos com o deslocamento)
  "sombra": {"rx":330,"ry":20,"x":480,"y":612,"a":0.55},
  "ancoras": {"chao": [395,684], "mao": [240,418,"mao_n"]},       // [x,y] ou [x,y,parte]
  "paleta": {"pelo": "#ee8f33"},        // usada como "$pelo" nas cores
  "aliases": {"correr": "caminhar"},
  "partes": [ { "id", "pai", "pivo":[x,y], "z", "oculta",
                "formas": [ {"t":"path","d":"M…","fill":"#hex | $nome | [r,g,b,a] | {\"g\":[[0,\"#a\"],[1,\"#b\"]],\"dir\":\"v|h\"}",
                             "stroke":"#hex","sw":2,"op":1,"clip":"base","m":[a,b,c,d,e,f]},
                            {"t":"poly","pts":[[x,y]…]}, {"t":"rect","x","y","w","h","r"}, {"t":"ellipse","cx","cy","rx","ry"},
                            {"t":"line","pts":[…]}, {"t":"glow","cx","cy","rx","ry","cor","a","exp"} ],
                "variantes": {"aberto": [formas], "fechado": [formas]}, "var_padrao": "aberto" } ],
  "pose_base": {"parte": {"rot":0,"tx":0,"ty":0,"sx":1,"sy":1,"op":1,"var":"x"}},
  "animacoes": { "caminhar": {"dur":1.0,"loop":true,
                   "faixas": {"parte": {"rot": [[0,-20],[0.5,20,"linear"],[1,-20]], "var": [[0,"aberto"],[0.9,"fechado"]]}}} },
  "luzes_por_parte": [{"parte":"tv_brilho","cor":"#8fd0ff","raio":520,"int":0.42}],
  "slots": {"tela": {"parte":"tv_conteudo","caixa":[x,y,w,h]}},
  "camadas": [ {"id","k":0.1,"desfoque":0,"formas":[…]} ]          // só para tipo "cenario" (k = profundidade/parallax)
}
```
**SVG:** `<g id="x">` filho direto da raiz vira a parte `x`; formas soltas viram a parte `base`. Um `nome.anim.json` ao lado pode trazer `partes` (pai/pivô/z), `partes_novas`, `animacoes`, `ancoras`, `slots`, `luzes_por_parte`, `pose_base`.

**Cena (`"modelo": "generica"`):** `cenario {ref|def|arquivo, enq{fmt:{esc,foco}}, ambiente}`, `elementos [{id, ref|def|arquivo, escala, pos{fmt}, mover[{t,para,ease}], olhar, linha_do_tempo[{t,anim,vel,mix,sincronizar}], ancorar{em,ancora,propria}, slots{nome:{elemento}}, k, z}]`, `camera {seguir, zoom[], pan[]}`, `luzes[]`, `hud {chip, etiquetas, pilulas}`, `legenda[]`, `destaques[]`.
