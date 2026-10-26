# BluetoothManager — revisão geral do projeto

## 1. Código morto → `DeadCode/`
- Os 45 arquivos que estavam como *Nenhum / não compilar* (`<None>`) foram movidos para `DeadCode/`, mantendo a estrutura de pastas original.
- Cópias (`* - Copia*.cs`), backups (`.old`) e ~30 classes marcadas como `<Compile>` mas **nunca referenciadas** (nem em C# nem em XAML) também foram para `DeadCode/`. Total: ~164 arquivos.
- Detecção feita por script, em ondas (um tipo só some se ninguém mais o usa), incluindo métodos de extensão, comentários ignorados e XAML varrido.
- `Class/Extensions/EventBinding/EventBindingExtension.cs` **permanece**: o XAML usa `{Event:Binding}`, que resolve para `BindingExtension` (sufixo omitido), então uma busca por nome não a enxerga.
- `DeadCode/` não faz parte do build (o csproj usa includes explícitos). Pode apagar quando quiser.
- Artefatos temporários da raiz (`BluetoothManager_*_wpftmp.csproj`, `Saída-Depuração.txt`) → `DeadCode/_Artefatos/`.
- Entradas órfãs no csproj (`CommandBase`, `WinndowCommand`, `PressEffect`) removidas.

## 2. Organização de pastas / namespaces (namespace = pasta)
| Arquivo | Novo local | Novo namespace |
|---|---|---|
| `BindableBase.cs` (raiz) | `Class/ViewModel/` | `BluetoothManager.Class.ViewModel` |
| `AppSettings.cs` (raiz) | `Class/DataModel/` | `BluetoothManager.Class.DataModel` |
| `Class/ConnectionStatus.cs` | `Class/ViewModel/ConnectionStatusInformation.cs` | `...Class.ViewModel` |
| `RadioModel` | `Class/ViewModel/` | `...Class.ViewModel` |
| `BluetoothDeviceStore` | `Class/Services/` | `...Class.Services` |
| `ListViewExpander`, `NotifyType` | `Class/UI/Helpers/` | `...Class.UI.Helpers` |
| `DeviceContextMenuViewModel` | `Class/UI/ViewModels/` | `...Class.UI.ViewModels` |
| Leitores/gerenciador de bateria + `HfpHandsFreeSession` | `Class/Services/Battery/` | `...Services.Battery` |
| `AudioPlaybackReceiverService` | `Class/Services/Audio/` | `...Services.Audio` |
| `BluetoothBackgroundTask` | (mesma pasta) | `...Class.BackgroundTask` (estava em `BluetoothManager`) |
| `DevInfoHelper` | (mesma pasta) | `...Class.Helpers` (estava no namespace global) |
| `Helper`, `CustomPopupPlacementHelper` | `Class/UI/Helpers/` | `...Class.UI.Helpers` (estavam em `UI.Controls`) |
| `SndSSO`, `SettingsPageHelper` | (mesma pasta) | namespace corrigido |
- `BatteryUpdateEventArgs.cs` renomeado para `BatteryUpdatedEventArgs.cs` (nome do arquivo = nome do tipo).
- `using` adicionados automaticamente em todo arquivo que referencia tipos movidos; referências parcialmente qualificadas (`Services.X`) corrigidas.
- Verificação por script: nenhum `using` faltando, nenhum arquivo do csproj inexistente, nenhum `.cs` fora do csproj.

## 3. Vazamentos de memória / recursos
- **Objetos WinRT nativos agora são descartados (`Dispose`)**: `BluetoothDevice`, `BluetoothLEDevice`, `GattSession`, `RfcommDeviceService` em `GattSessionHelper`, `BluetoothAudioSupportHelper` (novo `DisposeServices`) e `ClassOfDeviceKindReader` (em `finally`).
- **Assinaturas de evento em singletons** (`RadioModel.Current`, `BluetoothBatteryManager`, `AudioPlaybackReceiverService`) agora são soltas: `ConnectButton` (Loaded/Unloaded), `btnSwitch` (Loaded/Unloaded, antes assinava no construtor e nunca soltava) e `BluetoothDeviceViewModel.Dispose` (`UnregisterDevice`, remove `CallStateChanged`/`ReceivingChanged`).
- `BluetoothBatteryManager.CloseDeviceSessions` fecha sessões GATT/HFP do dispositivo removido.
- `ConnectionStatusInformation.ReleaseHeldConnections()` libera conexões mantidas pelo timer de status.
- Restantes (`MainWindow`, `App`, `FlyoutViewModel`, `BackgroundTask`) assinam eventos com vida igual à do app: intencional, não é vazamento.

## 4. Falhas corrigidas / lógica duplicada
- Status do celular preso em "conectado" ao perder a conexão (gatilho `staleConnected` + reavaliação ao cair o AEP).
- BG não reconectava dispositivos sem VM na UI (*headless*): `GetMonitoredDevices`, `EnsureHeadlessDevice`, `RemoveHeadlessDevice`, `_attachingIds`.
- Reconexão KS reportava sucesso falso: agora exige cobertura confirmada (`consecutiveComplete >= 1`).
- Sonda de bateria pulava o leitor BLE para IDs só-BLE (blocos `try` separados).
- `PairAsync` retorna cedo se já conectado; mensagens de resultado por motivo (`conresult`): Bluetooth do PC desligado / dispositivo desligado ou fora de alcance / falha genérica.
- Dead code interno removido (`additionalProperties` em `DeviceInformationHelper`, etc.).

## 5. Funcionalidades entregues nesta rodada (anteriores à revisão)
- Notificação do AutoConnect ao trocar o dispositivo de áudio padrão.
- AG: ocioso → "Conectado"; chamada → "Chamadas, conectado"; recebendo áudio → "Recebendo áudio, conectado".
- Gate no BG: verifica rádio do PC e presença do alvo antes de reconectar (mensagem única por transição).
- Textos "Conectando..." / "Desconectando..."; tooltip da bandeja atualiza em tempo de execução.
- `PairAsync`/`ConnectButton`: diagnóstico **só após a falha** (sem gate antes da tentativa, como pedido).

## 6. Segunda rodada (pós-revisão)

### 6.1 `AsyncOperationExtensions` própria (pacote `System.Runtime.WindowsRuntime` removido)
- A classe arquivada só cobria `IAsyncOperation<T>`. A nova (`Class/Extensions/AsyncOperationExtensions.cs`) cobre os 4 tipos WinRT (`IAsyncAction`, `IAsyncOperation<T>`, `IAsyncActionWithProgress<P>`, `IAsyncOperationWithProgress<T,P>`) com `GetAwaiter` **e** `AsTask` (com `CancellationToken`).
- Melhorias: continuações fora da thread de callback do WinRT (`RunContinuationsAsynchronously`), erro sem `ErrorCode` tratado, exceção ao ler o resultado propagada para a Task, `Close()` da operação ao terminar, namespace = pasta (antes era global, origem da ambiguidade intermitente com o pacote).
- `Class/Extensions/RandomAccessStreamExtensions.cs` (novo): `ReadAllBytesAsync()` substitui `AsStreamForRead()`, que também vinha do pacote. `DeviceGlyphProvider` agora usa ele e passou a **descartar o `DeviceThumbnail`** (antes vazava o stream nativo).
- Os 3 `.AsTask()` existentes (leitores RFCOMM/HFP) passam a usar a classe nova.
- `using BluetoothManager.Class.Extensions;` adicionado em 28 arquivos que fazem `await`.
- Removidos: pacote do `packages.config`, `Import` do `.targets` e o alvo `EnsureNuGetPackageBuildImports` do csproj. O csproj não tinha `<Reference>` explícita ao pacote, só esse `Import`.
- **Se o build reclamar** de `await`/`AsTask`/`AsStreamForRead` em algum ponto que eu não previ, o caminho de volta é restaurar o pacote (as 3 linhas acima) e **excluir** `AsyncOperationExtensions.cs` do csproj (os dois juntos causam ambiguidade).
- Limitação WinRT: `await`/`AsTask` duas vezes na MESMA operação lança `InvalidOperationException` (`Completed` só aceita uma atribuição).

### 6.2 Status "Música / Voz / Voz e música, conectado"
- Perfis: **A2DP** = música estéreo; **HSP** (Headset) e **HFP** (Hands-Free) são dois perfis de **voz** independentes (HFP é o sucessor do HSP, com controle de chamadas). HFP **não** é a união de A2DP + HSP. Logo `hasVoice = HFP || HSP` está correto; "Voz e música" só deve aparecer com os dois grupos realmente ativos.
- Causa real do "sempre Voz e música": o intérprete somava a consulta **RFCOMM** (`GetRfcommServicesForIdAsync`), que devolve o serviço sempre que o registro SDP existe, isto é, quando o fone *anuncia* HFP/HSP, mesmo desconectado. Todo fone anuncia HFP, então `hasVoice` era sempre verdadeiro. O fallback de A2DP ("existe algum endpoint ativo pro MAC") ainda marcava A2DP por qualquer endpoint, inclusive o Hands-Free.
- Correção (`AudioProfileInterpreter`): música/voz agora vêm **só da topologia** (endpoint vivo: nó A2DP Sink / `BTHHFENUM`). RFCOMM ficou apenas para o papel AG (celular) e só é consultado quando não há mídia (menos consultas nativas a cada 1,5 s).
- Correção (`BluetoothTopologyWalker.DetectMediaProfiles`): o atalho "endpoint padrão" só encerra se **os dois** perfis foram vistos. O endpoint padrão costuma ser só o A2DP; o Hands-Free é outro endpoint, e um resultado parcial esconderia a voz.
- Custo: fones só com A2DP passam a fazer a varredura completa de endpoints a cada ciclo (antes o atalho encerrava cedo). Se notar lentidão, o próximo passo é cachear o resultado por alguns segundos.
- Não alterado: o *sweep* de desconexão em `BluetoothConnectionService` (~linha 484) ainda mistura RFCOMM; ali o efeito é tentar desconectar um perfil que já saiu, não o rótulo do status.

### 6.3 Watcher de não pareados: descarte e vazamentos (`BluetoothDeviceCollectionViewModel`)
O fluxo base já estava correto: `DetachWatcher` solta os handlers, `Prune`/`Watcher_Removed` passam por `RemoveDevice` e a VM é `Dispose`d (assinaturas, timer, bateria). O que **ainda vazava** ao entrar e sair não pareados:
- **`BluetoothDeviceStore`** guarda a própria VM por Id (`PairAsync` registra depois de `PairAsync`, inclusive quando falha) e só era limpo ao desemparelhar: a VM descartada ficava presa para sempre. Agora `RemoveDevice` faz `Unregister`.
- **Fake-disconnect**: `ClearFakeDisconnect` re-gravava `false` no cache estático e o `IsFakeDisconnected` fazia `GetOrAdd` para qualquer Id → uma entrada por dispositivo (BLE não pareado muda de endereço a cada aparição) + um `Settings.Save()` em disco por remoção. Agora o cache guarda só Ids ativos, remove ao limpar, e só grava em disco quando algo mudou. O dicionário do `AppSettings` passou a ser protegido por `lock` (lido pela thread de status, escrito pela UI).
- **Estado estático por Id**: novo `BluetoothConnectionService.ForgetDevice(id)` (fake-disconnect, guarda do KS, throttle do CfgMgr via `BluetoothTopologyWalker.ForgetTransient`) chamado em `RemoveDevice`. Os caminhos persistidos do KS **não** são apagados.
- **Eventos atrasados**: um `Added` já despachado antes do `-=` ainda chegava e recriava um não pareado depois do `Prune` (preso até a próxima desativação). Watchers parados entram numa `ConditionalWeakTable` (fraca, não retém os watchers) e `Added/Updated/Removed` os ignoram.
- **`CollectionClear`** reimplementava só parte da remoção; agora usa `RemoveDevice` (um ponto único).
- **`Prune`/`CollectionClear` não descartam** dispositivo em pareamento (`IsPairing`) ou conectando perfis: perder o foco no meio do pareamento descartava a VM que o `PairAsync` ainda usava. Se falhar, sai na próxima desativação da janela.
- `BluetoothBatteryManager`: leitura em andamento que termina após `UnregisterDevice` não recria mais o nível da bateria.
- Não pareados nunca têm bateria lida (`IsPaired` barra) nem entram em `deviceCache`.

**Observação (não alterado, depende de hardware):** quando um dispositivo muda de estado de pareamento, ele sai de um watcher e entra no outro com o mesmo Id. Se o `Added` do watcher novo chegar antes do `Removed` do antigo, o `Added` é ignorado (Id já indexado) e o `Removed` depois descarta a VM: o dispositivo some da lista até a próxima abertura da janela. Correção sugerida: no `Removed`, reconsultar `DeviceInformation.CreateFromIdAsync(id)` e só remover se não existir mais ou se o estado de pareamento ainda corresponder ao watcher que removeu. Confirme no hardware (parear/desemparelhar com a janela aberta) antes de aplicar.
- `BluetoothDeviceStore` hoje só tem escrita (`GetDeviceById`/`GetAllDevices` não são chamados em lugar nenhum): é candidato a remoção.

### 6.4 "Ghost" depois de desemparelhar
- **Origem do ghost:** o app **não grava dispositivos no registro**. Quem mantém o registro é o Windows: ao desemparelhar, o cache de dispositivos conhecidos (`HKLM\SYSTEM\CurrentControlSet\Services\BTHPORT\Parameters\Devices`) continua com a entrada, e o watcher de não pareados a lista como "não pareado" mesmo **fora de alcance**. O app só reproduzia o que o Windows reportava.
- **Correção principal:** o watcher passa a pedir `System.Devices.Aep.IsPresent`. Não pareado com `IsPresent=false` (e sem conexão) **não vira VM nem aparece na lista**: fica estacionado como `DeviceInformation` leve em `_absentUnpaired` e entra se ficar presente; se um não pareado listado sair de alcance, ele sai da lista. Sem a propriedade, assume presente (nada some por engano). `_absentUnpaired` é limpo quando os watchers param.
- **Atualização após remover:** novo `RefreshUnpairedDevices()` (descarta os não pareados atuais e reinicia só o watcher de descoberta), chamado 1,5 s depois de desemparelhar.
- **`UnpairAsync` removia da lista SEMPRE**, até com `AccessDenied`/`Failed`: o dispositivo seguia pareado no Windows mas sumia da lista e a VM era descartada. Agora só remove se o resultado foi `Unpaired`/`AlreadyUnpaired`.
- **Persistência do app (nenhuma no registro):** `AutoConnectDevices` e `FakeDisconnectedDevices` em `user.config`; caminhos do KS em `%LocalAppData%\BluetoothManager\ks-paths.txt`. Ao desemparelhar de vez, `ForgetDevicePermanently` remove as três.
- **Bug encontrado na revisão dos caches:** cada `BluetoothDeviceViewModel` criava o próprio `AppSettings`, cada um com cópia privada dos mapas e regravando o dicionário inteiro: uma instância sobrescrevia ou ressuscitava o que outra gravou (entrada fantasma em `user.config`). Os mapas agora são estáticos, compartilhados e protegidos por `lock`.
- Caches revisados: `BluetoothDeviceStore`, fake-disconnect, guarda do KS, throttle do CfgMgr, `kindCache`/`inFlight`, `ActiveSockets`/`ActiveGattSessions` (só dispositivos conectados), bateria. Só os três persistidos acima sobrevivem a um desemparelhamento.

### 6.5 Ciclo de vida dos controles / memória
A janela principal é criada uma vez e só é exibida/ocultada (Loaded/Unloaded não disparam no Hide); o que gira são as linhas (`DeviceView` por `BluetoothDeviceViewModel`). Retenções encontradas e corrigidas:
- `Brush.cs` / `ThemeBindingInfo.cs`: registro estático forte de bindings de tema (prendia cada controle e `ThemeChanged`). Agora `ConditionalWeakTable`, assinatura idempotente e auto-desinscrição quando o alvo some.
- `DeviceContextMenuViewModel`: lambdas em `RadioModel.Current.PropertyChanged` viraram handler nomeado, removido em `Dispose()`; `BluetoothDeviceViewModel.Dispose` agora chama `Popup?.Dispose()`.
- `ListViewItem._pressedOriginItem`: estático forte → `WeakReference`.
- `DeviceView`: handlers registrados em `Loaded` e removidos em `Unloaded` (simétrico; antes uma linha recarregada perdia o popup).
- `BattIcon`: novo `Detach()` (chamado no `Dispose` do VM) e o laço de espera do `BluetoothBatteryManager` aborta quando o ícone é desligado.
- `FlyoutViewModel`: ao entrar em `Hidden`, `Dialog.IsVisible=false` limpa `Focused`/`Source`.
- `ConnectButton` revisado: assina `RadioModel`/VM em `Loaded` e solta em `Unloaded` (ok). `PairingPanel`: só assina o próprio `KeyDown` (ok); os `TaskCompletionSource` pendentes só vivem durante um pareamento.
- XAML: `ConnectButton.xaml` passou a usar `xmlns:helpers="...Class.UI.Helpers"` para `CustomPopupPlacementHelper` (antes `ctl:`, namespace antigo). Tooltip da caixa "Conexão automática" reescrito para o modelo atual (reconexão automática; Desconectar via KS).
- `HfpHandsFreeSession.ParseCindValueAt`: lambda `v` → `item` e `out int v` → `out int value` (conflito de escopo CS0136).
Nada disto foi compilado nem testado em hardware.

### 6.6 Leitura de bateria de celular: quando começa
Gatilhos automáticos (todos passam por `ReadBatteryForDeviceCoreAsync`): (1) `BattIcon.RegisterWhenReady` ao criar/religar o ícone ou quando `IsPaired` vira true; (2) `BluetoothBackgroundTask.HandleAepConnection` ao detectar conexão; (3) timer de 5 min. Todos exigem o dispositivo pareado **e link ativo** (`ConnectionStatus == Connected`); sem link a leitura é abortada para não "chamar" aparelhos ausentes. Por isso um celular só era lido após conectar do lado dele.
Novo: `RegisterDeviceAsync(device, openLinkIfNeeded: true)` ignora o portão do link. `PairAsync.TryConnectAsync` chama isso (fire-and-forget, `StartManualBatteryRead`) quando o dispositivo pareado não está conectado, deixando os leitores (GATT/RFCOMM/HF→AG) abrirem a conexão. Se já houver leitura em andamento para o mesmo Id, a manual é ignorada. Não compilado nem testado em hardware.

### 6.7 Rastreador de sinal e leitura de bateria por proximidade
- `Class/Services/Signal/DeviceSignalTracker.cs` (novo): guarda o último RSSI por dispositivo, classifica em Near/Neutral/Far (limiares -70/-90 dBm, validade 60 s) e dispara `EnteredRange` quando o dispositivo passa a estar perto. Só conhece números. O RSSI e os limiares saíram do `BluetoothBackgroundTask`, que agora apenas o consulta (`Signals.IsNear`, `Classify`); o comportamento de reconexão não muda.
- `Class/Services/Battery/BatteryProximityReader.cs` (novo): política da leitura automática. Ao receber `EnteredRange` de um dispositivo pareado, **desconectado** e com rádio ligado, chama `RegisterDeviceAsync(..., openLinkIfNeeded: true)`, com intervalo mínimo de 2 min por dispositivo. Criado/descartado em `App.xaml.cs`.
- Limitação: o RSSI só vem do watcher do BG, que fica parado enquanto a janela (watcher principal) está aberta; com a janela aberta valem os gatilhos do `BattIcon`. Não compilado nem testado em hardware.

### 6.8 Status da leitura manual de bateria (`PairAsync.TryConnectAsync`)
- Novo `BluetoothDeviceViewModel.IsReadingBattery` → `DisplayStatus` mostra "Lendo bateria..." (depois de "Conectando..."), sem colapsar o botão conectar/desconectar (isso continua só com `IsConnectingProfiles`). Não usa `BusyStatus`, que é do `ConnectButton`.
- A leitura manual agora é aguardada (antes era fire-and-forget) e só roda quando a conexão de perfis NÃO deu certo.
- Notificação: dispositivo de áudio (`SupportsAutoConnect`) mantém a mensagem de conexão; dispositivo sem perfil de áudio (celular) deixa de mostrar "Falha ao conectar o dispositivo!" e passa a mostrar o resultado da bateria (`BluetoothResultHelper.batresult`: "Bateria: X%" / "Não foi possível ler a bateria do dispositivo."). Não compilado nem testado em hardware.

### 6.9 Watcher próprio (`ManagedDeviceWatcher`)
Validado em cópia experimental e incorporado ao projeto principal.
- `Class/Services/Observer/ManagedDeviceWatcher.cs` (novo): dono do ciclo de vida de um `DeviceWatcher`. (1) Eventos de watcher parado/reciclado são descartados por geração (`IsCurrent`), inclusive depois de enfileirados na UI; (2) dispositivo sem evento por 45 s é conferido por `IsPresent` (`CreateFromIdAsync`) e, se ausente, dispara `Expired` — cobre quem some sem `Removed`; (3) `Recycle()` recria o watcher a cada 10 min para soltar o cache interno do WinRT (o novo reanuncia o que está presente).
- `BluetoothDeviceCollectionViewModel`: os dois watchers (pareados e descoberta) passam a ser `ManagedDeviceWatcher`; só o de descoberta expira/recicla. `_retiredWatchers`, `CreateAndStartWatcher` e `DetachWatcher` antigos saíram. `OnUnpairedExpired` remove da lista e do estacionamento (`_absentUnpaired`); dispositivo com pareamento em andamento é preservado (`Touch`). Id expirado que volta a ficar presente é readotado (`AdoptExpiredDeviceAsync`), pois o WinRT não reenvia `Added`.
- Diagnóstico: a cada varredura (15 s) o Debug mostra `[WATCHER-DIAG] rastreados/expirados/lista/indice/estacionados/headless/memoria`. Para validar: deixe a janela aberta com dispositivos aparecendo e sumindo e veja se esses números param de subir.
- Não mudou: watcher do BG (`BluetoothBackgroundTask`) continua `DeviceWatcher` cru. Não compilado nem testado em hardware.

### 6.10 [CÓPIA] Achados do log de depuração (2026-10-23)
- **Laço de leitura de bateria a cada ~1,5 s:** `ConnectionStatusInformation` (timer de 1,5 s) chama `OnConnectionStatusUpdated` para dispositivo conectado → `BluetoothDeviceViewModel` reatribuía `IsPaired` e o setter notificava SEMPRE → `BattIcon.Device_PropertyChanged(IsPaired)` → `RegisterDeviceAsync` → leitura completa (PNP walk + RFCOMM/SDP sem cache ~1 s + objetos nativos). Corrigido: `IsPaired`, `CanPair`, `IsConnected` e `IsBleConnectable` só notificam quando o valor muda.
- **3-4 `ArgumentException` por tick:** `BluetoothDevice.FromIdAsync` com Id BLE-only. Agora `BluetoothIdHelper.IsBleOnlyId` pula a chamada (status e sonda do `BluetoothBatteryManager`).
- **Leitura de celular com sessão HFP viva:** `ReadBatteryForDeviceCoreAsync` devolve o `LastLevel` da sessão antes de abrir RFCOMM/SDP de novo.
- **Memória:** o log mostrou heap gerenciado de 4–5 MB, bem abaixo dos 28–37 MB do Gerenciador de Tarefas ⇒ o crescimento é nativo/pendente de finalizador (watchers e `BluetoothDevice` WinRT parados só liberam no finalizador, e o GC quase não roda com heap pequeno). `MemoryDiagnostics` registra `[MEM] gerenciada/privada/workingset` ao ativar/ocultar a janela e a cada varredura, e `CollectAfterHide` força `GC.Collect` + finalizadores 2 s depois de ocultar (desligável: `MemoryDiagnostics.CollectWhenWindowHides`). Comparar os `[MEM]` antes/depois da coleta diz se a hipótese vale.
- Não compilado nem testado em hardware.

### 6.11 Sincronização de status com os watchers próprios e interpretador

- **BG no `ManagedDeviceWatcher`**: o watcher do `BluetoothBackgroundTask` deixou de ser um `DeviceWatcher` cru; os handlers descartam eventos de geração antiga (`IsCurrent`).
- **`Added` de dispositivo já conhecido** (reinício/reciclagem reenvia tudo) passa a chamar `vm.Update(info)` — em `EnsureDevice` (principal) e `EnsureHeadlessDevice` (BG). Antes era ignorado e o status ficava defasado.
- **`BluetoothDeviceViewModel.Update(DeviceInformationUpdate)`** agora chama `ConnectionStatusInformation.OnDeviceUpdated` quando `IsPaired`/`IsConnected`/`CanPair` mudam. O `ConnectionStatusInformation` só lê o modelo e nunca é avisado por ele; os guards de mudança nos setters removeram o "empurrão" repetido que escondia essa lacuna.
- **`AudioProfileInterpreter`**: música da topologia (um perfil por navegação); voz = topologia OU RFCOMM (HFP/HSP); papel AG pela mesma consulta RFCOMM. RFCOMM é pulado durante `IsRecentlyKsDisconnected`. Ressalva: RFCOMM indica registro SDP, não perfil conectado — pode mostrar "Voz e música" quando só há A2DP ativo.
- `MemoryDiagnostics.CollectWhenWindowHides = false` (o GC forçado não reduziu a memória privada nos logs).
- **Exceções residuais**: `BluetoothAudioSupportHelper` (3 pontos) e `GattSessionHelper` agora pulam `FromIdAsync` conforme o tipo de id (`BluetoothIdHelper.IsBleOnlyId`). O Gatt só consulta ids BLE — se algum dual-mode com id clássico dependia dele para "conectado", reverter esse guard.
- **Timer de status**: `ConnectionStatusInformation.WindowActive` (ligado em `Initialize`/`StopWatcher`) muda o intervalo de 1,5 s (janela ativa) para 6 s (oculta).
- **`Watcher_Removed` consciente da origem**: o watcher de não pareados ignora Removed de VM já pareada (mesmo id BLE ao parear) e o de pareados ignora Removed de VM não pareada. Antes o dispositivo recém-pareado sumia até reabrir a janela.
- **Tooltip da bandeja**: `GetConnectedAudioProfilesAsync` inclui `LIVE_MUSIC`/`LIVE_VOICE` da topologia, então desconectar A2DP/voz dispara a reavaliação do `MediaStatus`. RFCOMM só complementa a voz quando a topologia já viu endpoint vivo (senão "Voz" ficava preso). Intervalo do timer oculto: 3 s.
- Limite conhecido: desconectar SÓ a voz mantendo a música ainda pode mostrar "Voz e música" (registro SDP).
- **Trim com a janela oculta**: `MemoryDiagnostics.CollectAfterHide` agenda (20 s, cancelado ao reativar) GC compactando o LOH + `EmptyWorkingSet`. Reduz o working set mostrado no Gerenciador de Tarefas; a memória privada comprometida só cai se houver o que liberar. Páginas necessárias voltam por page fault ao reabrir. Desligar: `TrimWhenHiddenIdle = false`.

## 7. Riscos e pontos de atenção
- Não compilado; a troca do pacote por `AsyncOperationExtensions` é a mudança de maior risco de build (ver 6.1); mudanças de namespace podem exigir ajuste pontual de `using` se algum XAML/arquivo local seu referenciar o namespace antigo.
- `System.Devices.Aep.IsPresent` consultado sob demanda (fail-open); semântica nos seus aparelhos ainda não validada.
- `BluetoothBatteryGenericReader` usa heurística; vale monitorar leituras erradas.
- `App.xaml.cs` tem `using static ...VisualStyleElement.ToolTip` sem uso real (resíduo; inofensivo).
- Duas tabelas de nomes de perfil (`BluetoothAudioSupportHelper.AudioProfileNames` e `BluetoothServiceStateController`) têm propósitos distintos e foram mantidas; unificar é possível, mas exige teste.
- `SndVolSSO.cs` e `Mdl2Icon.cs` não são usados; mantidos por serem pequenos (candidatos a `DeadCode/`).
