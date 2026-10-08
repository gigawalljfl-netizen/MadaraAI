# Madara — assistente de voz com a Gemini Live API + controle do celular

App Android (Java) que conversa por voz com o Gemini — persona "Madara",
inspirada no estilo frio e sarcástico de uma IA de ficção, sem citar falas de
filme nenhuma. Usa a **Gemini Live API** via WebSocket puro (sem SDK), controla
boa parte do celular por comando de voz, lê e enxerga a tela, tem memória
persistente por conta (Firebase) e login restrito a contas que você mesmo cria.

## ⚠️ Leia isto antes de tudo: o que o Android NUNCA permite

Um pedido importante foi "conseguir desbloquear a tela" com administrador de
dispositivo. Isso **não é possível para nenhum app**, em nenhuma versão do
Android, com nenhuma permissão — nem root de terceiros resolve isso de forma
confiável em aparelhos modernos. Se fosse possível, a senha do celular não
protegeria nada. Por isso:

- O app **bloqueia** a tela por comando (administrador de dispositivo ou
  acessibilidade) — isso sim é permitido e está implementado.
- O app **nunca** desbloqueia. O Madara sabe disso e vai explicar isso se
  você pedir, em vez de fingir que tentou.

Outros dois limites reais do Android, também documentados no código:

- **Hotspot**: nenhum app comum consegue ligar/desligar o ponto de acesso
  Wi-Fi sozinho (só apps de sistema). O comando `abrir_hotspot` abre a tela
  certa para você tocar no botão.
- **Bluetooth**: ligar mostra uma caixa de confirmação do sistema (não dá pra
  pular); desligar funciona direto em alguns aparelhos e em outros abre a
  tela de Bluetooth para você tocar.

## 1. Como a Gemini Live API funciona (resumo prático)

A Live API abre uma **conexão WebSocket** que fica aberta, trocando mensagens
JSON em tempo real (nada de pergunta-resposta única).

1. **Conectar**: `wss://generativelanguage.googleapis.com/ws/google.ai.generativelanguage.v1beta.GenerativeService.BidiGenerateContent?key=SUA_CHAVE`
2. **Setup** (primeira mensagem, obrigatória):
   ```json
   {
     "setup": {
       "model": "models/gemini-3.8-live",
       "generationConfig": {
         "responseModalities": ["AUDIO"],
         "speechConfig": { "voiceConfig": { "prebuiltVoiceConfig": { "voiceName": "Orus" } }, "languageCode": "pt-BR" }
       },
       "systemInstruction": { "parts": [{ "text": "..." }] },
       "tools": [{ "functionDeclarations": [...] }]
     }
   }
   ```
   (`responseModalities` e `speechConfig` ficam **dentro** de `generationConfig`
   — um erro comum é colocar isso direto em `setup`, e a API rejeita.)
3. Servidor responde `setupComplete` → dá pra mandar áudio.
4. **Áudio do microfone**: PCM 16 bits/16kHz/mono, em pedaços, base64, dentro
   de `realtimeInput.audio`. O servidor detecta sozinho início/fim da fala.
5. **Imagem da tela** (novo): mesma ideia, mas em `realtimeInput.video`, com
   `mimeType: "image/jpeg"` — é assim que o Madara "enxerga" um print de
   verdade, em vez de só ler uma lista de textos.
6. **Resposta em voz**: chega em pedaços dentro de
   `serverContent.modelTurn.parts[].inlineData.data` (PCM 24kHz, base64).
7. **Comandos do celular**: o servidor manda `toolCall` com uma ou mais
   funções pedidas; o app executa e devolve com `toolResponse`.

## 2. Vozes disponíveis

A Live API tem 30 vozes prontas. Algumas com timbre mais "sério/grave" (bom
pra a persona do Madara): `Orus`, `Charon`, `Iapetus`, `Alnilam`, `Fenrir`.
Troque no campo "Voz" da tela, ou peça por voz — o Madara tem a função
`mudar_voz` e reconecta sozinho com a voz nova quando decide que combina mais
com o momento (a API não permite trocar de voz no meio de uma sessão sem
reconectar — é assim que existe hoje).

## 2.1 Avatar (Orbe) e não se ouvir a si mesmo

- **Avatar**: o Madara tem um avatar visual — um orbe que pulsa e muda de cor
  conforme o estado (cinza calmo parado, azul pulsando rápido ouvindo,
  vermelho pulsando bem rápido enquanto fala). Está em `OrbeView.java`, no
  lugar onde antes tinha só um botão de microfone.
- **Não responder à própria voz**: o app ativa cancelamento de eco, supressão
  de ruído e controle automático de ganho do próprio Android (quando o
  aparelho suporta — a maioria suporta) para reduzir a chance do microfone
  captar a voz do Madara saindo do alto-falante. Não é 100% garantido em
  volume muito alto ou aparelhos mais simples — é uma redução real de
  problema, não uma solução perfeita, porque depende do hardware de cada
  celular.

## 2.2 Extended Thinking (raciocínio estendido)

Se você usar o modelo `gemini-3.8-live-extended-thinking` (em vez do
`gemini-3.8-live` normal), o Madara ganha um passo de raciocínio antes de
responder — melhor para perguntas mais complexas ou várias ações em
sequência, ao custo de mais latência.

Para usar:
1. Na tela de Configurações, troque **Modelo Live** para
   `gemini-3.8-live-extended-thinking`.
2. Preencha **Nível de pensamento** com `low`, `medium` ou `high`.

**Importante**: esse campo só pode ser preenchido com esse modelo específico
— com qualquer outro modelo (incluindo o `gemini-3.8-live` normal), ele
**precisa** ficar vazio, porque a API rejeita a conexão com erro 400 se
receber esse parâmetro para um modelo que não é o de Extended Thinking. O
app já cuida disso sozinho (só manda o parâmetro se detectar
"extended-thinking" no nome do modelo), mas vale saber o motivo se resolver
mexer no código.

Enquanto o Madara está nesse raciocínio mais longo (antes de responder ou
entre uma chamada de função e outra), o orbe fica **roxo** — um estado visual
a mais, além do azul (ouvindo) e vermelho (falando).

## 2.3 "Ouvir" som do celular (ambiente e arquivos)

Duas capacidades diferentes, para dois casos de uso:

- **`ouvir_ambiente`**: desliga temporariamente o cancelamento de eco (que
  normalmente filtra justamente o som saindo do alto-falante, pra evitar o
  Madara se ouvir falando). Útil pra comentar uma música ou vídeo tocando
  alto perto do celular. Se esquecer de desativar, ele volta sozinho depois
  de 20 segundos — é uma rede de segurança, já que ficar sem cancelamento de
  eco por muito tempo faz ele voltar a se confundir com a própria voz.
- **`ouvir_arquivo_de_audio`**: decodifica de verdade um arquivo de áudio
  salvo (ex: uma mensagem de voz do WhatsApp) e manda pro modelo "ouvir",
  pedaço por pedaço, no ritmo real da gravação — depois ele pode resumir,
  transcrever ou comentar o conteúdo. Suporta os formatos que o Android
  souber decodificar nativamente (opus, aac/m4a, mp3...).
  **Localizar mensagens de voz do WhatsApp**: a pasta varia com a versão —
  tente pedir `procurar_arquivo` por ".opus" primeiro, ou procure manualmente
  em `WhatsApp/Media/WhatsApp Voice Notes/` ou
  `Android/media/com.whatsapp/WhatsApp/Media/WhatsApp Voice Notes/`.

## 2.4 Tela inicial e atualização obrigatória

Ao abrir o app, uma tela de splash (`SplashActivity`) mostra "MADARA" por um
instante — isso evita a impressão de "tela preta travada" enquanto o app
confere, em segundo plano, se existe uma versão mais nova publicada como
**GitHub Release** no repositório. Se existir, a tela mostra um aviso com
botão de download e **não deixa passar** (não tem botão de pular) até você
baixar a versão nova e instalar por cima manualmente — continua sendo uma
instalação manual, como qualquer app fora da Play Store, só a checagem e o
aviso são automáticos. Veja a seção 10 sobre como publicar uma atualização
pra isso funcionar.

## 3. Estrutura do projeto

```
geminivoz/
├── app/src/main/java/com/meuapp/geminivoz/
│   ├── LoginActivity.java           # tela de entrada (Firebase Auth, sem cadastro público)
│   ├── MainActivity.java            # tela principal: só o orbe (toca = conectar/falar)
│   ├── SettingsActivity.java        # chave de API, modelo, voz, permissões especiais, log
│   ├── MadaraService.java           # Foreground Service: conexão viva, reconexão automática
│   ├── MadaraPersonalidade.java     # texto da personalidade + instrução de sistema
│   ├── GeminiLiveClient.java        # protocolo da Live API
│   ├── AudioRecorderStreamer.java   # microfone → PCM 16kHz
│   ├── AudioPlayer.java             # toca a voz que volta
│   ├── CommandExecutor.java         # executa os comandos do celular
│   ├── ComandoAccessibilityService.java  # lê tela, toca botões, tira print
│   ├── ComandoDeviceAdminReceiver.java   # permite bloquear a tela por comando
│   └── MemoryManager.java           # memória de longo prazo (Firestore)
├── app/src/main/res/xml/
│   ├── accessibility_service_config.xml
│   └── device_admin.xml
└── .github/workflows/android-debug-build.yml
```

## 4. Todos os comandos disponíveis

| Função | O que faz | Precisa de algo? |
|---|---|---|
| `abrir_app` | Abre um app instalado pelo nome | — |
| `ligar_para` | Abre o discador com o número pronto | — |
| `enviar_sms` | Abre o app de SMS com a mensagem pronta | — |
| `abrir_configuracoes` | Abre telas de config (wifi, bluetooth, som, bateria) | — |
| `ajustar_volume` | Aumenta/diminui/define o volume | — |
| `controlar_midia` | Play/pause/próxima/anterior em qualquer app | — |
| `lanterna` | Liga/desliga a lanterna | — |
| `pesquisar_na_web` | Abre o navegador já pesquisando | — |
| `navegar_celular` | Voltar, home, recentes, notificações, bloquear tela | Acessibilidade* |
| `ler_tela` | Descreve o texto visível na tela | Acessibilidade* |
| `ver_tela` | Manda um print de verdade pro modelo enxergar (melhor pra jogos/telas complexas) | Acessibilidade* |
| `tocar_em` | Toca num botão/item pelo texto | Acessibilidade* |
| `tocar_coordenada` | Toca num pixel exato (x,y) — funciona em jogos e apps "Lite" onde tocar_em falha | Acessibilidade* |
| `deslizar` | Arrasta o dedo entre dois pontos — swipe, mover peças em jogos, rolar telas difíceis | Acessibilidade* |
| `digitar_texto` | Digita num campo de texto selecionado | Acessibilidade* |
| `rolar_tela` | Rola a tela pra cima/baixo/esquerda/direita | Acessibilidade* |
| `capturar_tela` | Salva um print na galeria | Acessibilidade* |
| `ajustar_brilho` | Define o brilho da tela | Permissão "Modificar configurações"** |
| `ativar_rotacao_automatica` | Ativa/desativa a rotação automática | Permissão "Modificar configurações"** |
| `alternar_bluetooth` | Liga/desliga o Bluetooth | Confirmação do sistema |
| `abrir_hotspot` | Abre a tela do ponto de acesso Wi-Fi | Você toca no botão |
| `solicitar_acesso_a_arquivos` | Pede acesso a todos os arquivos | Você confirma |
| `tornar_se_administrador` | Vira administrador (só para poder bloquear a tela) | Você confirma |
| `lembrar` | Guarda um fato na memória de longo prazo | Estar logado |
| `mudar_voz` | Reconecta com outra voz | — |
| `listar_arquivos` | Lista arquivos/pastas | Acesso a arquivos |
| `escrever_arquivo` | Cria ou edita um arquivo de texto | Acesso a arquivos |
| `ler_arquivo` | Lê o conteúdo de um arquivo de texto | Acesso a arquivos |
| `renomear_arquivo` | Renomeia um arquivo/pasta | Acesso a arquivos |
| `apagar_arquivo` | Apaga um arquivo/pasta (irreversível) | Acesso a arquivos |
| `procurar_arquivo` | Procura arquivos pelo nome | Acesso a arquivos |
| `espaco_armazenamento` | Mostra espaço usado/livre e pastas mais pesadas | Acesso a arquivos |
| `copiar_arquivo` | Copia um arquivo | Acesso a arquivos |
| `mover_arquivo` | Move/recorta um arquivo | Acesso a arquivos |
| `compactar_arquivo` | Compacta um arquivo/pasta num .zip | Acesso a arquivos |
| `descompactar_arquivo` | Descompacta um .zip | Acesso a arquivos |
| `visualizar_arquivo` | Abre um arquivo direto (imagem, áudio, texto, vídeo) no app padrão do celular | Acesso a arquivos |
| `compartilhar_arquivo` | Abre o menu de compartilhar do Android para um arquivo | Acesso a arquivos |
| `criar_evento` | Cria compromisso no calendário | Permissão de calendário |
| `ler_agenda` | Lê compromissos dos próximos dias | Permissão de calendário |
| `criar_alarme` | Define um alarme (sem abrir o app) | — |
| `criar_timer` | Inicia um cronômetro regressivo | — |
| `ouvir_ambiente` | Liga/desliga o cancelamento de eco, pra ouvir som tocando alto no celular | — |
| `ouvir_arquivo_de_audio` | Decodifica e "ouve" um arquivo de áudio salvo (ex: voz do WhatsApp) | Acesso a arquivos |
| `ler_notificacoes` | Lê as notificações recentes | Acesso a notificações* |
| `o_que_esta_tocando` | Descobre título/artista do que está tocando em qualquer app | Acesso a notificações* |
| `ler_sms` | Lê os últimos SMS recebidos (só leitura) | Permissão de SMS |
| `tirar_foto` | Tira uma foto (câmera traseira ou frontal) e te mostra | Permissão de câmera |

\* **Acesso a notificações**: ative em Configurações > Acesso a notificações >
Madara (tem botão direto na tela do app, separado da acessibilidade).

\* **Acessibilidade**: ative em Configurações > Acessibilidade > Madara (tem
botão direto na tela do app). \*\* **"Modificar configurações do sistema"**:
o app abre a tela na primeira vez que precisar; ative lá.

### Por que `tocar_em` falha em alguns apps (TikTok Lite, Facebook Lite, jogos)

`tocar_em` procura um elemento com aquele texto na árvore de acessibilidade —
funciona muito bem em apps "normais" (com botões de verdade). Mas apps que
desenham a própria interface na tela (comum em versões "Lite", jogos de
tabuleiro, WebViews) não expõem elemento nenhum ali — são só pixels
desenhados, sem nome para o Android encontrar.

Para esses casos existe `tocar_coordenada` e `deslizar`: o Madara chama
`ver_tela` primeiro (recebe a imagem de verdade), identifica visualmente onde
precisa tocar, e manda a coordenada de pixel exata — do jeito que um dedo
tocaria, sem precisar de nome de elemento nenhum. Isso é mais lento (cada
tentativa é print → decisão → toque) mas funciona em qualquer app, incluindo
jogos.

### Orbe flutuante (mini avatar sobre outros apps)

Ative em **"Ativar mini orbe flutuante"** na tela do app (abre a permissão
"Aparecer sobre outros apps" do Android, é uma permissão especial). Uma vez
ativado, sempre que o Madara estiver falando — mesmo com você usando outro
app — um pequeno orbe pulsante aparece na tela. Pode arrastar ele para
qualquer canto; a posição fica salva enquanto o serviço estiver rodando.

Quando falta uma permissão, o app não trava nem finge que funcionou — ele
avisa o Madara do jeito certo (que vai te contar, no seu estilo) **e** já
abre a tela de configuração certa para você resolver com um toque.

## 5. Memória e contas (Firebase)

Cada usuário loga com e-mail/senha (Firebase Authentication) e tem sua própria
memória guardada no Firestore. Quando o Madara usa a função `lembrar`, o fato
é salvo na conta; na próxima vez que você conectar, esses fatos entram
automaticamente na instrução de sistema, então ele "lembra" de verdade.

**Importante entender o que isso significa na prática**: a memória é uma
lista de fatos curtos que o próprio Madara decide guardar durante a conversa
— não é uma gravação da conversa inteira. Se ele não chamar `lembrar` sobre
algo, aquilo não fica guardado, mesmo que tenha sido dito na sessão. A
instrução de sistema já pede pra ele usar essa função quando algo parecer
importante, mas isso depende do julgamento dele a cada momento — se quiser
garantir que algo específico fique guardado, a forma mais confiável é pedir
diretamente ("guarda isso: ...").

**De propósito, não tem botão de "criar conta" no app.** Isso evita ter que
montar um sistema de limite de contas só para uso pessoal nesta fase. Em vez
disso, você (o administrador) cria as contas manualmente:

### Configurando o Firebase (obrigatório antes do próximo build)

1. Acesse https://console.firebase.google.com e crie um projeto novo.
2. Adicione um app Android com o pacote `com.meuapp.geminivoz`.
3. Baixe o arquivo `google-services.json` gerado e coloque em `app/google-services.json`
   (na mesma pasta do `app/build.gradle`).
4. No console, ative **Authentication > Sign-in method > E-mail/senha**.
5. Em **Authentication > Users**, clique em "Add user" e crie a sua conta
   (e-mail + senha) e, se quiser, até mais 9 contas — esse é o limite
   confortável para essa fase de testes sem precisar de infraestrutura extra
   (um limite automático de "só 10 contas" exigiria uma Cloud Function paga,
   fora do escopo por enquanto).
6. Em **Firestore Database**, clique em "Create database".
7. **Importante — isso é bem provavelmente a causa de "a memória não
   funciona"**: se você escolher o modo "produção" (travado por padrão), o
   Firestore recusa TODA leitura e escrita até você configurar regras de
   segurança — e esse app até agora tentava guardar memória "em silêncio",
   sem avisar que a tentativa estava sendo recusada (isso já foi corrigido
   no código: agora o log mostra o erro de verdade). Para corrigir, vá em
   **Firestore Database > Regras** e cole isto, substituindo o que já
   estiver lá:
   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /usuarios/{userId} {
         allow read, write: if request.auth != null && request.auth.uid == userId;
       }
     }
   }
   ```
   Isso permite que cada usuário logado leia/escreva **só o próprio**
   documento de memória — nada de um usuário ver a memória de outro. Clique
   em "Publicar" depois de colar.
8. Faça commit do `google-services.json` junto com o resto do projeto.

**Sem esse arquivo, o build vai falhar** (o plugin do Firebase exige ele para
existir). Se você quiser testar sem Firebase por enquanto, é só remover as
linhas do Firebase em `build.gradle` (raiz e `app/`) e do `AndroidManifest`
referentes a login — mas aí perde a memória entre sessões.

## 5.1 Robustez: segundo plano e reconexão automática

A conversa agora roda dentro de um **Foreground Service** (`MadaraService`), não
mais direto na tela. Isso muda duas coisas na prática:

- **Apagar a tela ou trocar de app não derruba a conversa.** O Android mostra
  uma notificação fixa ("Madara — Conectado") enquanto isso, porque é
  obrigatório por lei do sistema avisar quando um app grava áudio em segundo
  plano — não dá pra tirar essa notificação, é assim para qualquer app.
- **Se a internet cair no meio de uma conversa**, o serviço detecta e tenta
  reconectar sozinho, esperando um pouco mais a cada tentativa (2s, 4s, 8s,
  16s, até um teto de 30s), até a conexão voltar — sem precisar tocar em nada.
  Isso só acontece se a queda foi inesperada; se você tocou em "Desconectar"
  de propósito, ele não tenta voltar sozinho.

Limite real que continua existindo: se o **sistema matar o processo inteiro**
por falta de memória (raro, mas acontece em celulares com pouca RAM com muitos
apps abertos), a conversa se perde de verdade e é preciso tocar em Conectar de
novo — isso está fora do que qualquer app consegue controlar.

## 5.2 Limites deliberados (não são bugs)

Alguns pedidos foram implementados com um limite proposital, por segurança ou
por respeito a outras pessoas:

- **Navegar no WhatsApp/Facebook**: o Madara já consegue fazer isso — ele vê a
  tela (`ver_tela`), toca (`tocar_em`), digita (`digitar_texto`) e rola
  (`rolar_tela`) em qualquer app, então "cria um grupo no WhatsApp com Fulano"
  já funciona, passo a passo, com você acompanhando. O que ele **não** faz é
  decidir sozinho responder mensagens ou interagir com outras pessoas sem você
  ter pedido aquela ação específica — as pessoas do outro lado não sabem que
  estão falando com uma IA, e automação desse tipo também viola os termos de
  uso do WhatsApp/Facebook (risco real de banimento da conta).
- **Ligações e SMS continuam pedindo confirmação manual** (abre o discador/app
  de mensagens prontos, você toca em enviar/ligar). Isso é escolha, não
  limitação técnica — evita ligação/mensagem indesejada por engano. Se quiser
  tirar essa trava depois, é possível, mas não veio ativado por padrão.
- **Administrador de dispositivo**: só a política de bloquear a tela foi
  ativada. Políticas mais fortes (apagar dados do aparelho, resetar senha)
  existem no Android mas não foram incluídas — um bug ou mal-entendido
  executando isso seria irreversível.

## 5.3 Correções: erros 400 e 429 do painel da API

Duas causas prováveis, já corrigidas neste commit:

- **400 (Bad Request)**: um screenshot em resolução nativa (comum passar de
  1440×3000 pixels num celular atual) vira um JPEG grande o bastante para a
  API rejeitar o pedido. Agora `ver_tela` redimensiona a imagem para no
  máximo 1024px no lado maior antes de mandar — resolução de sobra para o
  modelo ler texto e identificar botões, bem mais leve para enviar.
  `tocar_coordenada` e `deslizar` continuam funcionando certinho: o app
  guarda o fator de redução usado e converte a coordenada que o modelo manda
  de volta para o pixel real da tela automaticamente. `capturar_tela`
  (o print que salva na galeria) continua em resolução total, já que esse
  nunca é enviado pela rede.
- **429 (Too Many Requests)**: se o orbe fosse tocado duas vezes rápido, ou
  uma reconexão automática disparasse ao mesmo tempo que uma manual, o app
  podia abrir duas conexões ao mesmo tempo com a mesma chave — cada uma
  consome cota, e isso pode estourar limite de requisições por minuto. Agora
  existe uma trava: enquanto uma conexão está em andamento, uma segunda
  tentativa é simplesmente ignorada até a primeira terminar (com sucesso ou
  erro).

Se o painel continuar mostrando erros depois dessas correções, o próximo
passo é olhar a mensagem de erro exata (geralmente dá pra ver clicando no
gráfico ou nos logs do Google AI Studio) — o texto do erro diz exatamente
qual parte do pedido a API rejeitou.

## 5.4 Estabilidade: o serviço de acessibilidade "parando de funcionar"

Duas causas reais foram corrigidas nesta rodada:

1. **Chave de assinatura instável entre builds**: sem configurar isso, cada
   build (seu Android Studio, o GitHub Actions) gerava sua própria chave de
   debug aleatória. Isso faz o Android tratar cada novo APK como um "app
   diferente" — só instala substituindo o anterior se você desinstalar
   primeiro, o que reseta acessibilidade, notificações, orbe flutuante e
   administrador do dispositivo toda vez. Agora existe uma `debug.keystore`
   fixa, versionada no repositório (`app/build.gradle` aponta pra ela
   explicitamente), então todo build futuro — local ou no CI — usa a mesma
   assinatura, e atualizar o app pura e simplesmente **atualiza** (mantém as
   permissões especiais), em vez de exigir reinstalação.
   **Só desta vez** você ainda vai precisar desinstalar o app antes de
   instalar essa versão (porque ela já nasce com a chave nova, diferente da
   que você tinha instalada) — depois disso o problema para de acontecer.
2. **Crash silencioso derrubando o processo inteiro**: em jogos e apps que
   atualizam a tela rápido (a Damas, TikTok Lite, Facebook Lite), os objetos
   que a acessibilidade usa pra "ver" a tela podem ficar inválidos no meio da
   leitura — e uma notificação mal-formada também podia derrubar tudo. Sem
   proteção, uma exceção nesses casos matava o processo inteiro, e como o
   serviço de acessibilidade roda no mesmo processo, o Android o desativava e
   mostrava "este serviço está a funcionar incorretamente". Agora todo esse
   código está protegido (try/catch) e não deixa mais isso acontecer, além de
   um tratador de exceções global como rede de segurança extra
   (`MadaraApplication.java`).

## 5.4.1 Diagnóstico: arquivo de crash e fix do Android 14

- **Todo crash agora fica registrado** em `Download/Madara/crash_log.txt` no
  armazenamento do celular, com a mensagem de erro completa e em qual
  "thread" aconteceu. Se o app travar de novo, abra esse arquivo (ou peça pro
  próprio Madara ler com `ler_arquivo`, se ele ainda estiver funcionando) e
  me manda o conteúdo — isso me dá o erro exato, em vez de eu ter que
  adivinhar pela descrição do que você viu na tela.
- **Crash ao conectar no Android 14+**: a causa era específica dessa versão —
  o Android passou a exigir que a permissão de microfone já esteja concedida
  **antes** de iniciar o serviço em primeiro plano do tipo "microfone", e o
  app só pedia essa permissão depois de já estar conectado. Corrigido: agora
  o microfone é pedido antes de tentar conectar, em vez de depois.

## 5.5 "Contar até 100" e outras tarefas longas não param mais sozinhas

Duas correções juntas resolvem isso:

- **`realtimeInputConfig.activityHandling: "NO_INTERRUPTION"`** foi ativado no
  setup da API — sem isso, qualquer ruído/eco que o servidor confundisse com
  "o usuário começou a falar" cortava a resposta do Madara no meio. Com essa
  opção, ele sempre termina o que está falando; se você quiser interromper de
  verdade, ele processa isso assim que a fala atual acabar (não instantâneo,
  mas confiável).
- A instrução de sistema agora deixa explícito que tarefas mecânicas e
  literais (contar, listar, repetir) devem ser completadas do início ao fim,
  mesmo que isso contrarie o estilo normal de respostas curtas do Madara — e
  que silêncio do usuário durante isso é esperado, não um pedido pra parar.

## 6. Rodando no Android Studio

1. Abra a pasta `geminivoz/`. Aceite a criação do Gradle Wrapper se pedir.
2. Gradle vai baixar OkHttp, AppCompat, Material e as libs do Firebase.
3. Rode num celular real com Android 8.0+ (screenshot/`ver_tela` e
   `capturar_tela` exigem Android 11+; o resto funciona desde o 8.0).
4. Faça login com uma conta criada no Firebase.
5. A tela principal só tem o orbe — toque no ícone de engrenagem (canto
   superior direito) para ir em Configurações, colar sua chave de API do
   Gemini (tem um botão que leva direto pro aistudio.google.com se o campo
   estiver vazio), escolher modelo/voz/nível de pensamento nos menus prontos
   (sem precisar digitar nada) e tocar
   em **Conectar**. Depois disso, basta voltar e tocar no orbe: se estiver
   desconectado, ele conecta; se já estiver conectado, liga/desliga o
   microfone.
6. Ative o **controle avançado** (botão na tela) se quiser `ver_tela`,
   `tocar_em`, `ler_tela`, `capturar_tela` e navegação do sistema.

## 7. Compilando pelo GitHub Actions

`.github/workflows/android-debug-build.yml` roda a cada push em `main`:
baixa o código, instala JDK 17 + Gradle 8.7, roda `gradle assembleDebug` e
publica o APK como artefato. Sem `google-services.json` commitado, este passo
falha — veja a seção 5.

## 8. Comandos para subir no GitHub

```bash
git add .
git commit -m "Fase 3: Madara com controle total do celular, memória e login"
git push origin main
```
(troque `origin`/`main` pelo remote e branch que você já está usando)

## 10. Publicando uma atualização (pro verificador funcionar)

Commits e pushes normais **não** disparam o aviso de atualização — só um
**GitHub Release de verdade** dispara. Você disse que vai assinar e publicar
isso pessoalmente pelo Termux quando achar que está pronto, então o fluxo é:

1. Gere o APK assinado com a sua própria chave (via `gradle assembleRelease`
   ou o comando que você preferir no Termux).
2. No GitHub, vá em **Releases > Draft a new release**.
3. Em "Tag", coloque a versão no mesmo formato usado em `versionName`
   (ex: `0.0.0.0.2`) — o verificador compara exatamente esse texto.
4. Anexe o arquivo `.apk` gerado como asset do Release.
5. Publique. Na próxima vez que alguém abrir o app, a tela de splash vai
   detectar a versão nova e pedir a atualização.

**Importante**: o `versionName` em `app/build.gradle` (hoje `0.0.0.0.1`)
precisa ser atualizado por você a cada Release de verdade que publicar — ele
não sobe sozinho a cada commit, só funciona como você descreveu (reflete
mudança quando você decidir publicar).

## 10.1 Política de Privacidade e Termos de Uso

Documento completo em `app/src/main/res/raw/politica_privacidade.txt`,
acessível dentro do app na tela de Login e em Configurações. Pontos centrais:

- A chave de API de cada usuário fica **só no aparelho dele**, nunca é vista
  ou guardada pelos Desenvolvedores.
- Só duas coisas saem do aparelho em direção ao servidor dos Desenvolvedores
  (Firebase): as memórias que o Madara guarda, e os dados da conta (e-mail de
  login). Nada de áudio, imagem, SMS ou arquivo é armazenado lá.
- A voz, imagens e texto da conversa vão direto do aparelho pro Google
  (dono do modelo Gemini que faz o "Madara" pensar) — isso é regido pela
  política de privacidade do próprio Google, não pela dos Desenvolvedores.
- Nenhum dado é vendido ou compartilhado com terceiros, exceto se exigido por
  lei/ordem judicial.

Caso o texto final mude, edite só o arquivo `.txt` — o app carrega ele
dinamicamente, não precisa mexer em código.

## 11. Limitações conhecidas / próximos passos

- `ligar_para` e `enviar_sms` pedem confirmação manual do usuário — é
  proposital, ligar/mandar SMS sem confirmação exige permissões perigosas
  (`CALL_PHONE`, `SEND_SMS`) que não valem o risco nesta fase.
- Limite de 10 contas é manual (você controla quem existe no console do
  Firebase); um limite automático via Cloud Function fica para depois.
- "Jogar" jogos complexos depende de quão bem o modelo interpreta os prints
  enviados por `ver_tela` — funciona melhor em jogos simples de toque do que
  em jogos que precisam de reflexo rápido, já que cada rodada de "tirar
  print → mandar → esperar decisão → tocar" tem uma latência perceptível.
- `digitar_texto` no Termux especificamente não tem garantia: o terminal do
  Termux não é uma caixa de texto padrão do Android, então pode não responder
  a `ACTION_SET_TEXT` como um app comum responderia. Vale testar — se não
  funcionar, o log mostra o erro exato e dá pra investigar uma alternativa.
- Brilho e rotação automática dependem da permissão "Modificar configurações
  do sistema", que é por app e manual (Android não deixa conceder isso
  silenciosamente, por segurança).
