# Travel Cost Premium

App PWA, offline-first, para registrar e controlar custos de viagens e passeios.

## Correções desta versão

**v2.2**

4. **Reconexão da pasta em 1 toque, de qualquer tela.** Antes era preciso ir
   até Configurações › Armazenamento para reconectar. Agora a própria pilha
   de status no topo (ao lado da lupa) já reconecta com um único toque quando
   mostra "Toque aqui para reconectar a pasta" — reusa o mesmo handle salvo,
   sem abrir o seletor de novo. Importante: por segurança, **nenhum
   navegador** permite reconceder permissão de gravação de arquivos sem um
   toque explícito do usuário — não existe "automático de verdade" aqui, mas
   agora é só 1 toque em vez de navegar até as configurações.
5. **Memórias agora têm ordenação**, igual às Despesas: por data da captura,
   data de adição, nome ou tipo, crescente/decrescente, com a preferência
   salva.
6. Versão exibida em Configurações › Aplicativo atualizada para **2.2**.

**v2.1 (anterior)**

1. **Pasta duplicada ao vincular pasta local.** Antes, o app sempre criava uma
   subpasta chamada "Travel Cost" dentro da pasta escolhida em "Vincular
   pasta local" — então, se você já tinha escolhido a própria pasta "Travel
   Cost" (o mais comum), o resultado era `Travel Cost/Travel Cost/<viagem>/...`.
   Agora a pasta que você escolhe no seletor **é** a raiz: as viagens são
   criadas diretamente dentro dela, sem repetir o nome. *Isso não corrige
   sozinho pastas já duplicadas de antes* — se você já tem uma
   `Travel Cost/Travel Cost/`, mova o conteúdo de dentro para a pasta de fora
   e apague a duplicada uma vez, manualmente.
2. **"Pasta local não vinculada" depois de fechar e reabrir.** Isso acontecia
   porque a permissão de gravação de uma pasta (concedida pelo
   `showDirectoryPicker`) é resetada pelo navegador em algumas situações
   (reiniciar o navegador, por exemplo) — é um comportamento de segurança do
   próprio navegador, não um bug de perda de dados: a vinculação continua
   salva, só a permissão precisa ser reconfirmada. Antes disso, o app não
   diferenciava "nunca vinculei" de "vinculei mas perdi a permissão nesta
   sessão", e por isso mostrava a mensagem confusa. Agora, quando isso
   acontece, aparece **"Pasta vinculada — toque em Config. para reconectar"**
   e, em Configurações › Armazenamento, um botão **"Reconectar pasta"** que
   reusa a mesma pasta (sem abrir o seletor de novo e sem risco de duplicar).
3. **Câmera só tirava 1 foto por vez.** É uma limitação do próprio
   navegador/sistema operacional quando se usa `<input capture>` — não tem
   como pedir múltiplas fotos numa única chamada de câmera nativa. A solução
   foi um **"Modo rajada"**: uma caixa de seleção ao lado do botão Foto (em
   Memórias e também ao anexar comprovante de despesa) que reabre a câmera
   automaticamente logo depois de cada foto, até você desmarcar a opção —
   na prática dá para tirar 4, 5 ou quantas fotos seguidas precisar, com um
   toque rápido entre cada uma.

## Como executar

Não precisa de build. É HTML/CSS/JS puro.

- **Local rápido:** abra `index.html` direto no navegador. Funciona, mas o Service
  Worker (offline real + instalação PWA) só é registrado em `http://localhost`
  ou `https://` — não em `file://`. Para testar isso, rode um servidor local:
  ```
  cd travelcost
  python3 -m http.server 8080
  ```
  e acesse `http://localhost:8080`.
- **Hospedagem:** qualquer hospedagem estática com HTTPS (GitHub Pages, Netlify,
  Vercel, Cloudflare Pages, um bucket S3 com CloudFront, etc.). PWA exige HTTPS
  (exceto localhost).

## Arquivos

```
index.html            → app inteiro (HTML + CSS + JS)
manifest.webmanifest   → metadados de instalação (PWA)
sw.js                  → service worker (cache do app shell, offline)
icon.svg                → ícone do app
README.md
```

## O que já está implementado e funcional

- Design system completo (tokens de cor claro/escuro, tipografia, espaçamento)
  seguindo exatamente a paleta e a fonte (Montserrat, com troca de fonte)
  especificadas.
- Armazenamento local em **IndexedDB** (não localStorage) — viagens, despesas,
  centros de custo, formas de pagamento, anexos (Blobs) e configurações.
- Fluxo completo: criar viagem → editar → finalizar → reabrir → arquivar.
- Despesas: criar, editar, duplicar, excluir (com confirmação e exclusão
  lógica via `deletedAt`), anexar comprovante (foto pela câmera ou arquivo),
  ordenar (data do movimento/criação/alfabética/valor), pesquisar, filtrar
  (centro de custo, forma de pagamento, sem comprovante).
- Aviso quando a data da despesa está fora do período da viagem, sem bloquear.
- Centros de custo e formas de pagamento totalmente gerenciáveis pelo usuário,
  com os defaults sugeridos no prompt. Cartões nunca armazenam número
  completo, CVV ou senha — apenas apelido, banco, bandeira e últimos 4
  dígitos.
- Dashboard com totais, viagens recentes e pendências.
- Relatório por viagem (resumo por centro de custo + despesas detalhadas) e
  geração de **PDF via impressão do navegador** (`window.print()` com CSS
  `@media print` dedicado) — ver limitação abaixo.
- Backup manual: exportação de todos os dados em JSON.
- Tema claro/escuro (com opção "seguir sistema"), seletor de fonte (9 fontes
  do Google Fonts), tamanho de fonte ajustável, tudo persistido.
- Navegação responsiva: sidebar no desktop, barra inferior no mobile; grid de
  cards adaptável (1/2/3 colunas).
- Onboarding curto na primeira abertura, com opção de pular.
- Indicador de sincronização honesto sobre o estado real (local vs. offline
  vs. nuvem ainda não configurada) — nunca finge estar sincronizado.
- Estados vazios em todas as telas relevantes.
- Foco visível, labels em todos os campos, `prefers-reduced-motion`
  respeitado, tamanhos de toque adequados.

## Armazenamento em nuvem: sem OAuth, por decisão do usuário

Esta versão **abandona o fluxo OAuth** e usa uma abordagem mais simples, como
pedido: configuração de pasta feita em Configurações › Armazenamento, com
dois mecanismos complementares:

1. **Link da pasta compartilhada (Google Drive/OneDrive).** O usuário cola o
   link (com permissão de edição) e ele fica salvo **somente neste
   dispositivo**, servindo como referência/atalho ("Abrir pasta"). É
   importante saber: **um link por si só não permite que o navegador grave
   arquivos nele.** O Google Drive e o OneDrive não expõem uma API de upload
   anônimo via link — mesmo um link "qualquer pessoa com o link pode editar"
   exige autenticação para qualquer chamada de API de escrita. Isso não é uma
   limitação deste app; é assim que as duas plataformas funcionam. Por isso o
   app é transparente: enquanto só o link estiver configurado, os arquivos
   ficam marcados como "aguardando sincronização".

2. **Pasta local vinculada (File System Access API) — a parte que realmente
   grava arquivos.** Em navegadores desktop compatíveis (Chrome/Edge), o
   usuário pode vincular uma pasta local de verdade — por exemplo, a pasta
   que o **Google Drive para Desktop** ou o **cliente do OneDrive** já
   sincronizam automaticamente com a nuvem. A partir daí, o Travel Cost grava
   comprovantes, notas fiscais e memórias diretamente nessa pasta local
   (usando `showDirectoryPicker`/`FileSystemDirectoryHandle`, sem pedir login
   nenhum), e quem sobe os arquivos para a nuvem é o próprio sincronizador do
   Google/Microsoft já instalado no computador. Isso é 100% real — não é mock.
   Em Safari/iOS e na maioria dos navegadores mobile essa API não existe
   ainda; nesses casos o app permanece local-first normalmente.

**Por dispositivo, por design — exatamente como pedido:** o handle da pasta
local não pode ser serializado, então cada dispositivo precisa vinculá-la de
novo. O **backup em JSON** carrega o link de referência e todas as
configurações (tema, fonte, centros de custo, formas de pagamento etc.), mas
nunca a permissão de gravação — isso o navegador nunca guarda automaticamente
por segurança, em nenhum app.

### Estrutura de pastas criada automaticamente

Assim que uma pasta local é vinculada, **toda vez que uma viagem é criada**
(e também de forma preguiçosa, ao salvar a primeira despesa/memória se a
pasta for vinculada depois), o app cria a estrutura completa:

A pasta que você vincula em Configurações › Armazenamento **é** a raiz — não
é criada nenhuma subpasta com o nome "Travel Cost" dentro dela:

```
<pasta que você vinculou>/
└── <Apelido da viagem>/
    ├── Despesas/
    │   ├── Notas Fiscais/
    │   ├── Comprovantes/
    │   └── Documentos/
    ├── Memórias/
    │   ├── Fotos/
    │   └── Vídeos/
    ├── Relatorios/
    └── backup/
```

Comprovantes de despesas vão para `Despesas/Comprovantes`, fotos e vídeos do
módulo Memórias vão para `Memórias/Fotos` e `Memórias/Vídeos`, com nomes de
arquivo organizados automaticamente (`AAAA-MM-DD_HH-MM-SS_<apelido>_xxxx.ext`).

## Módulo Memórias

Implementado nesta entrega:

- Aba **Memórias** dentro da viagem, com contagem de fotos/vídeos.
- Captura direta: **Tirar foto** (`capture="environment"`), **Gravar vídeo**,
  ou **Galeria/arquivo** (sem `capture`, permite escolher da galeria).
- Grid de miniaturas (fotos e vídeos, com indicador visual de vídeo);
  visualizador em tela cheia com foto ampliada ou player de vídeo nativo.
- Descrição opcional editável por memória.
- Exclusão com confirmação (com aviso claro quando o arquivo já foi gravado
  na pasta vinculada — a remoção lá é manual, o app nunca apaga na nuvem
  silenciosamente).
- Indicador de "aguardando sincronização" em cada miniatura quando a pasta
  não está vinculada.
- Grava direto na pasta local vinculada, na subpasta correta
  (`Memórias/Fotos` ou `Memórias/Vídeos`), quando disponível.
- Card de resumo de memórias na aba Resumo da viagem.

Ainda não implementado (próxima fase, se fizer sentido): geolocalização
opcional por memória, geração de álbum em PDF, agrupamento por dia,
associação de uma memória como comprovante de despesa, e detecção de
duplicidade ao importar a mesma mídia duas vezes.

## Limitação assumida conscientemente: geração de PDF

Gerar PDF client-side "de verdade" normalmente usa uma biblioteca (ex.:
jsPDF) carregada via CDN. Isso criaria uma dependência de rede — contrariando
o requisito de que o app não fique inutilizável offline se a CDN cair. A
solução adotada foi um **relatório em HTML com CSS de impressão dedicado**
(`@media print`), e o botão "Gerar PDF" chama `window.print()`, onde o
usuário escolhe "Salvar como PDF" no diálogo do navegador. Isso funciona
100% offline, sem qualquer dependência externa, em todos os navegadores
modernos (Android, iOS, desktop). Se preferir uma biblioteca de PDF
embutida localmente (sem CDN) para gerar o arquivo programaticamente, é
possível trocar essa etapa depois — vale a pena decidir isso já pensando em
como os anexos (fotos/PDFs de comprovante) devem ou não entrar no relatório.

## Passo a passo: publicar no GitHub Pages

Boa notícia: `manifest.webmanifest` e `sw.js` já usam caminhos relativos
(`./`), então funcionam sem alteração tanto na raiz de um domínio quanto num
subcaminho como `https://usuario.github.io/repositorio/` — não precisa mexer
em `start_url`, `scope` nem nos caminhos dos ícones.

1. **Crie o repositório** em github.com → "New repository". Pode ser público
   ou privado (GitHub Pages funciona nos dois, mas no plano gratuito só
   publica direto de repositório público, ou privado se você tiver GitHub
   Pro/Team/Enterprise).
2. **Suba estes 5 arquivos na raiz do repositório** (não dentro de nenhuma
   subpasta): `index.html`, `manifest.webmanifest`, `sw.js`, `icon.svg`,
   `README.md`. Pode arrastar e soltar direto pela interface do GitHub
   ("Add file" → "Upload files") ou usar git:
   ```
   git init
   git add index.html manifest.webmanifest sw.js icon.svg README.md
   git commit -m "Travel Cost Premium"
   git branch -M main
   git remote add origin https://github.com/SEU-USUARIO/SEU-REPOSITORIO.git
   git push -u origin main
   ```
3. **Ative o GitHub Pages:** no repositório, vá em **Settings → Pages**. Em
   "Build and deployment", em "Source", escolha **"Deploy from a branch"**.
   Em "Branch", escolha **main** e a pasta **/ (root)**. Clique em **Save**.
4. **Aguarde 1–2 minutos.** O GitHub mostra o link no topo da própria página
   de Settings → Pages, algo como:
   `https://SEU-USUARIO.github.io/SEU-REPOSITORIO/`
5. **Abra esse link no celular/computador.** Confirme que carrega, que dá
   para instalar como app (ícone de instalar na barra do navegador, ou
   "Adicionar à tela inicial" no menu do Chrome/Safari mobile), e que
   funciona offline depois de abrir uma vez (ative o modo avião e recarregue
   — deve continuar funcionando).
6. **Sempre que atualizar o código:** suba os arquivos novos (novo commit) e
   espere alguns segundos. Como o Service Worker faz cache do app, pode ser
   necessário fechar e reabrir o app (ou aguardar a próxima abertura) para a
   versão nova aparecer — o `sw.js` já está com `CACHE_NAME` incrementado
   nesta entrega (`travel-cost-v2`) para forçar a atualização em quem já
   tinha a versão antiga instalada. Da próxima vez que você mudar o
   `index.html`, incremente esse nome de novo (`v3`, `v4`...) para garantir
   que o cache antigo seja descartado.
7. **Vincular a pasta local funciona normalmente no GitHub Pages**, já que é
   servido por HTTPS (exigência da File System Access API e do Service
   Worker). Só não funciona em navegadores mobile (Android/iOS), que ainda
   não suportam essa API — nesses casos o app permanece local-first, como já
   documentado acima.

## Testado

- Sintaxe do JavaScript validada.
- Fluxo manual: criar viagem → adicionar despesa com comprovante (foto) →
  fechar e reabrir o navegador (dados persistem via IndexedDB) → editar →
  duplicar → excluir → ordenar/filtrar/pesquisar → finalizar → reabrir →
  gerar relatório/imprimir → adicionar memória (foto/vídeo) → excluir
  memória → exportar backup JSON → restaurar backup JSON → alternar
  tema/fonte → redimensionar para 320px/768px/1440px.
- **Ainda não testado em dispositivo real** (iOS Safari, Android Chrome, nem
  a vinculação de pasta local no Windows/macOS com Drive/OneDrive Desktop de
  verdade instalado) — recomendo validar isso especialmente antes de usar em
  produção. `capture="environment"` e `showDirectoryPicker` têm suporte
  variável entre navegadores/SOs.

## Próximas fases sugeridas (seguindo a ordem do prompt original)

1. Álbum de memórias em PDF, geolocalização opcional, associação de memória
   como comprovante de despesa.
2. Exportação CSV e pacote ZIP completo (dados + arquivos juntos).
3. Gráficos (gastos por dia, por centro de custo, por forma de pagamento).
4. Auditoria/histórico de alterações por despesa.
5. Atalhos de teclado no desktop (N, V, /).
6. Se um dia quiser voltar a cogitar OAuth de verdade (Google Drive/OneDrive
   API), a decisão consciente aqui foi evitar isso — mas a estrutura de
   pastas e nomes de arquivo já ficaram compatíveis com esse cenário, caso
   mude de ideia no futuro.
