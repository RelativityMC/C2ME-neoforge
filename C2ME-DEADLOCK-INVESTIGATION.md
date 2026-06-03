# C2ME — Deadlock investigation: synchronous `getChunkBlocking` on the server thread hangs forever

> **But de ce document**
> Te donner *tout* le contexte nécessaire pour attaquer, dans le repo C2ME forké, la correction d'un deadlock dur du **thread principal serveur** ("Server thread") qui se produit dès qu'un appel **synchrone** `getChunk` / `getChunkBlocking` est fait sur le thread principal pour un **chunk non encore chargé**.
>
> Ce fichier est écrit pour être lu *avant* d'ouvrir le code : il contient les preuves (3 stack traces complètes capturées en prod), l'analyse du mécanisme, des hypothèses de cause racine classées, et une liste de points d'entrée à inspecter dans la base de code C2ME. À la fin il y a un plan d'investigation suggéré.
>
> ⚠️ Les numéros de ligne `net.minecraft.*` proviennent du **runtime NeoForge déobfusqué (Mojmap)** de Minecraft 1.21.1. Dans le repo C2ME (Fabric/Yarn) les noms peuvent différer — voir le tableau de correspondance Mojmap↔Yarn plus bas.

---

## 1. TL;DR

- **Symptôme** : le serveur se fige (thread principal parké à vie), MSPT explose, les joueurs restent sur "Encrypting…" puis time out, ou tombent dans un monde vide puis time out.
- **Mécanisme commun** : un appel `Level.getChunk(...)` synchrone (avec `create=true`, statut `FULL`) est fait **sur le Server thread** pour un chunk **non chargé**. Le vanilla descend dans `ServerChunkCache.getChunkBlocking` → `MainThreadExecutor.managedBlock(BooleanSupplier)` qui *boucle* : `while(!chunkFuture.isDone()) { if(!runAllTasks()) LockSupport.parkNanos("waiting for tasks", 100_000L); }`. **La future du chunk n'est jamais complétée** et aucune tâche n'arrive jamais sur le `mainThreadProcessor` → boucle infinie (livelock/deadlock).
- **C2ME est dans la pile** : l'appel passe par les wrappers mixinextras de C2ME `c2me_base$instrumentGetChunk` (@WrapMethod) et `c2me_base$instrumentAwaitChunk` (@WrapOperation). Le deadlock arrive **à l'intérieur** du `managedBlock` enveloppé par `instrumentAwaitChunk`.
- **Condition nécessaire** : le chunk visé est **non chargé** (sinon `getChunkBlocking` retourne immédiatement, pas d'attente). Donc ça ne se déclenche que pour des chunks distants/non générés/non encore promus au statut demandé.
- **3 déclencheurs indépendants observés en prod** (même signature exacte) : (1) reinstatement de force-chunks Mekanism au boot, (2) FTB Chunks qui force-load les chunks claimés au login du joueur, (3) une procédure MCreator du mod *butcher* qui fait `getBlockState` dans un chunk non chargé à chaque tick d'entité.
- **Reproductible sur 0.3.0+alpha.0.91 ET 0.3.0+alpha.0.93** (dernière alpha, ~mai 2026). Le bug est donc présent dans la branche courante.
- **Hypothèse principale** : le `managedBlock` du `MainThreadExecutor` (instrumenté par C2ME) pompe la mauvaise file. La complétion de la future de chunk, dans le **chunk system réécrit** de C2ME, n'est pas pilotée par le `mainThreadProcessor` vanilla que `runAllTasks()` draine — donc rien ne fait avancer la future tant que le thread principal est coincé dans `managedBlock`. Cyclique : le seul thread qui pourrait piloter le scheduler C2ME (le main thread) est bloqué à l'attendre.

---

## 2. Environnement de reproduction (prod)

| Élément | Version |
|---|---|
| Minecraft | 1.21.1 |
| Loader | NeoForge 21.1.214 |
| Java | OpenJDK 21.0.11 (image `itzg/minecraft-server:java21`) |
| **C2ME** | `c2me-neoforge-mc1.21.1-0.3.0+alpha.0.91` **et** `…0.3.0+alpha.0.93` (les deux reproduisent) |
| Modpack | "Coven Cobblemon" 1.3.1 (CurseForge), très gros pack (centaines de mods) |
| Mémoire | `-Xmx32G`, `MaxDirectMemorySize=2G`, flags MeowIce |
| CPU container | 8 vCPU (cpuset), `globalExecutorParallelism = default` (≈6) |

> **Note** : C2ME avait été **ajouté manuellement** par-dessus le modpack (il n'en fait pas partie). Il a été retiré pour rétablir le service. Le fork est destiné à corriger le bug puis potentiellement le réintégrer.

### Mods impliqués dans les 3 déclencheurs
| Mod | Version | Rôle dans le bug |
|---|---|---|
| Mekanism | 10.7.15 | Bloc *chunk loader* (Anchor) → reinstatement de force-chunks au boot (cas 1) |
| FTB Chunks | 2101.1.9 | Force-load des chunks claimés au login (cas 2) |
| `butcher` (MCreator) | 3.5.6 | `SortinghadrecipeProcedure.onEntityTick` → `getBlockState` chunk non chargé (cas 3) |
| Async (axalotl `com.axalotl.async`) | 0.2.0+alpha | Parallélise le tick d'entités ; présent dans le cas 3 mais **pas** dans les cas 1/2 → aggravateur potentiel, pas la cause racine |

### Config C2ME au moment des reproductions
**Tout est sur `"default"`** (`config/c2me.toml`, `version = 3`). Aucune option n'a été modifiée. Points notables (valeurs par défaut) :
- `[chunkSystem] asyncSerialization = true`, `useLegacyScheduling = true`, `delayFullChunkEvents = true`, `allowPOIUnloading = true`, `syncPlayerTickets = true`
- `[ioSystem] replaceImpl = true`
- `[noTickViewDistance] enabled = true`
- `[generalOptimizations] midTickChunkTasksInterval = 100000` (ns)
- `globalExecutorParallelism = default`

---

## 3. La signature de deadlock commune (à connaître par cœur)

Extrait commun aux **3** thread dumps (haut de pile identique au bit près) :

```
"Server thread" #... prio=8 ... runnable  [0x...]
   java.lang.Thread.State: TIMED_WAITING (parking)
        at jdk.internal.misc.Unsafe.park(java.base@21.0.11/Native Method)
        - parking to wait for  <0x...> (a java.lang.String)
        at java.util.concurrent.locks.LockSupport.parkNanos(java.base@21.0.11/Unknown Source)
        at net.minecraft.util.thread.BlockableEventLoop.waitForTasks(minecraft@1.21.1/BlockableEventLoop.java:143)
        at net.minecraft.util.thread.BlockableEventLoop.managedBlock(minecraft@1.21.1/BlockableEventLoop.java:133)
        at net.minecraft.server.level.ServerChunkCache$MainThreadExecutor.managedBlock(minecraft@1.21.1/ServerChunkCache.java:533)
        at net.minecraft.server.level.ServerChunkCache.mixinextras$bridge$managedBlock$102(minecraft@1.21.1/ServerChunkCache.java)
        at net.minecraft.server.level.ServerChunkCache$$Lambda/0x...call(...)
        at net.minecraft.server.level.ServerChunkCache.wrapOperation$z??000$c2me_base$instrumentAwaitChunk(minecraft@1.21.1/ServerChunkCache.java:5132)
        at net.minecraft.server.level.ServerChunkCache.getChunkBlocking(minecraft@1.21.1/ServerChunkCache.java:1750)
        at net.minecraft.server.level.ServerChunkCache.getChunk$mixinextras$wrapped$107(minecraft@1.21.1/ServerChunkCache.java:1671)
        at net.minecraft.server.level.ServerChunkCache.mixinextras$bridge$getChunk$mixinextras$wrapped$107$108(...)
        at net.minecraft.server.level.ServerChunkCache$$Lambda/0x...call(...)
        at net.minecraft.server.level.ServerChunkCache.wrapMethod$z??000$c2me_base$instrumentGetChunk(minecraft@1.21.1/ServerChunkCache.java:5146)
        at net.minecraft.server.level.ServerChunkCache.getChunk(minecraft@1.21.1/ServerChunkCache.java)
        at net.minecraft.world.level.Level.getChunk(minecraft@1.21.1/Level.java:202)
        at net.minecraft.world.level.Level.getChunk(minecraft@1.21.1/Level.java:5335)
        <<< ici diverge selon le déclencheur (voir §4) >>>
```

### Lecture détaillée de cette signature
1. **`LockSupport.parkNanos("waiting for tasks", 100_000L)`** — le blocker `(a java.lang.String)` est littéralement la chaîne `"waiting for tasks"` passée par `BlockableEventLoop.waitForTasks`. C'est donc le park *vanilla* de 100 µs, en boucle. Le thread se réveille toutes les 100 µs, ré-évalue le `BooleanSupplier`, le trouve toujours faux, re-park. **Livelock**, pas un park sur moniteur.
2. **`BlockableEventLoop.managedBlock(BooleanSupplier until)`** (ligne 133) — boucle vanilla : `while (!until.getAsBoolean()) { if (!this.runAllTasks()) this.waitForTasks(); }`. Le `until` est `() -> chunkFuture.isDone()` (ou équivalent). Donc **la future du chunk ne se complète jamais** et `runAllTasks()` ne trouve **jamais** de tâche à exécuter (sinon il ne parkerait pas).
3. **`ServerChunkCache$MainThreadExecutor.managedBlock`** (ligne 533) — l'override MC qui, en plus de drainer les tâches, appelle normalement `ServerChunkCache.this.pollTask()` / `runDistanceManagerUpdates()`. C'est *ce* `managedBlock` qui est enveloppé par C2ME.
4. **`wrapOperation$…$c2me_base$instrumentAwaitChunk`** (ServerChunkCache.java:5132) — un `@WrapOperation` **mixinextras** de C2ME (module/refmap `c2me_base`) qui enveloppe l'appel `managedBlock(...)` à l'intérieur de `getChunkBlocking`. Méthode handler C2ME : **`instrumentAwaitChunk`**.
5. **`wrapMethod$…$c2me_base$instrumentGetChunk`** (ServerChunkCache.java:5146) — un `@WrapMethod` **mixinextras** de C2ME qui enveloppe entièrement `ServerChunkCache.getChunk(...)`. Méthode handler C2ME : **`instrumentGetChunk`**.
6. `getChunkBlocking` (1750) est appelé par `getChunk` (1671) avec `create = true`, à un statut `FULL` (chemin `Level.getChunk(int,int)` → `getChunk(x, z, ChunkStatus.FULL, true)`).

**Point clé** : le deadlock n'est PAS dans du code C2ME custom au sommet de pile — il est dans le `managedBlock` **vanilla** *à l'intérieur* du wrapper C2ME `instrumentAwaitChunk`. Donc soit `instrumentAwaitChunk`/`instrumentGetChunk` laisse passer un appel qui, dans le chunk system réécrit, ne peut jamais aboutir sur le main thread ; soit la future attendue dépend d'un "tick"/drain de scheduler C2ME que `managedBlock` ne fait pas.

---

## 4. Les 3 cas reproduits (stack traces complètes, bas de pile)

Toutes capturées via `kill -3 <pid_java>` (SIGQUIT → thread dump dans stdout) sur le serveur figé, **deux dumps espacés** à chaque fois pour confirmer l'absence de progression (le compteur `cpu=` du Server thread n'augmentait quasiment pas : ~+1 à +1,4 s de CPU sur 15–30 s d'horloge → thread parké, pas occupé).

### Cas 1 — Boot : reinstatement des force-chunks Mekanism
Se produit pendant `prepareLevels`, **avant** le `Done` du serveur → le serveur n'atteint jamais l'état "démarré".

```
... (signature commune §3) ...
        at net.minecraft.world.level.Level.getChunk(Level.java:5325)
        at net.minecraft.world.level.Level.getChunkAt(Level.java:5320)
        at net.minecraft.world.level.Level.getBlockEntity(Level.java:777)
        at mekanism.common.tile.component.TileComponentChunkLoader$ChunkValidationCallback.validateTickets(mekanism@10.7.15/TileComponentChunkLoader.java:302)
        at mekanism.common.tile.component.TileComponentChunkLoader$ChunkValidationCallback.validateTickets(TileComponentChunkLoader.java:291)
        at net.neoforged.neoforge.common.world.chunk.ForcedChunkManager.lambda$reinstatePersistentChunks$6(neoforge@21.1.214/ForcedChunkManager.java:145)
        at java.lang.Iterable.forEach(...)
        at net.neoforged.neoforge.common.world.chunk.ForcedChunkManager.reinstatePersistentChunks(ForcedChunkManager.java:139)
        at net.minecraft.server.MinecraftServer.prepareLevels(MinecraftServer.java:519)
        at net.minecraft.server.MinecraftServer.loadLevel(MinecraftServer.java:339)
        at net.minecraft.server.dedicated.DedicatedServer.initServer(DedicatedServer.java:193)
        at net.minecraft.server.MinecraftServer.runServer(MinecraftServer.java:670)
```
- Déclencheur : NeoForge réinstaure les force-chunks persistés (`<dim>/data/chunks.dat`, entrée `ModForced` / `Controller: mekanism:chunk_loader`). Mekanism valide chaque ticket via `getBlockEntity` → `getChunkAt` (chunk non chargé pendant `prepareLevels`) → `getChunkBlocking`.
- **Contournement appliqué en prod** : supprimer `world/data/chunks.dat`.

### Cas 2 — Login : FTB Chunks force-load les chunks claimés
Se produit après `Done`, pendant le handler de connexion du joueur. Le joueur "joined the game" puis time out (~60 s) sans réception de chunks.

```
... (signature commune §3) ...
        at net.neoforged.neoforge.common.world.chunk.ForcedChunkManager.forceChunk(neoforge@21.1.214/ForcedChunkManager.java:96)
        at net.neoforged.neoforge.common.world.chunk.TicketController.forceChunk(neoforge@21.1.214/TicketController.java:68)
        at dev.ftb.mods.ftbchunks.neoforge.FTBChunksExpectedImpl.addChunkToForceLoaded(ftbchunks@2101.1.9/FTBChunksExpectedImpl.java:13)
        at dev.ftb.mods.ftbchunks.FTBChunksExpected.addChunkToForceLoaded(FTBChunksExpected.java)
        at dev.ftb.mods.ftbchunks.data.ChunkTeamDataImpl.lambda$updateChunkTickets$6(ftbchunks@2101.1.9/ChunkTeamDataImpl.java:473)
        at java.util.ArrayList.forEach(...)
        at dev.ftb.mods.ftbchunks.data.ChunkTeamDataImpl.updateChunkTickets(ChunkTeamDataImpl.java:469)
        at dev.ftb.mods.ftbchunks.FTBChunks.loggedIn(ftbchunks@2101.1.9/FTBChunks.java:229)
        at dev.ftb.mods.ftbchunks.FTBChunks$$Lambda/0x...accept(...)
```
- Déclencheur : au login, FTB Chunks ré-applique les tickets de force-load des chunks claimés du joueur via `ForcedChunkManager.forceChunk` (synchrone) → `getChunk` d'un chunk non chargé.
- **Contournement appliqué en prod** : retirer les clés `force_loaded:` dans `world/ftbchunks/<team-uuid>.snbt` (claims conservés, force-load désactivé). A réglé *ce* trigger, mais le cas 3 est apparu juste après → preuve que ce n'est pas spécifique à FTB Chunks.

### Cas 3 — Tick : procédure MCreator du mod `butcher` lit un chunk non chargé
Se produit pendant le tick serveur normal (`tickServer` → `tickChildren`), sous le ticking d'entités parallélisé par le mod Async. **C'est le cas le plus révélateur** car il prouve que *n'importe quel* accès `getBlockState`/`getChunk` synchrone d'un chunk non chargé sur le main thread suffit — aucun "force-load" en jeu.

```
... (signature commune §3) ...
        at net.minecraft.world.level.Level.getBlockState(Level.java:5827)
        at net.mcreator.butcher.procedures.SortinghadrecipeProcedure.execute(butcher@3.5.6/SortinghadrecipeProcedure.java:41)
        at net.mcreator.butcher.procedures.SortinghadrecipeProcedure.onEntityTick(SortinghadrecipeProcedure.java:33)
        at ...LambdaForm... (NeoForge event dispatch)
        at net.neoforged.bus.EventBus.post(EventBus.java:360 / 328)
        at net.neoforged.neoforge.event.EventHooks.fireEntityTickPre(neoforge@21.1.214/EventHooks.java:953)
        at net.minecraft.server.level.ServerLevel.tickNonPassenger(ServerLevel.java:773)
        at com.axalotl.async.common.ParallelProcessor.tickSynchronously(async@0.2.0+alpha-1.21.1/ParallelProcessor.java:162)
        at com.axalotl.async.common.ParallelProcessor.callEntityTickBatch(ParallelProcessor.java:114)
        at net.minecraft.server.level.ServerLevel.redirect$elc000$async$overwriteEntityTicking(ServerLevel.java:17577)
        at net.minecraft.server.level.ServerLevel.tick(ServerLevel.java:400)
        at net.minecraft.server.MinecraftServer.tickChildren(MinecraftServer.java:1037)
        at net.minecraft.server.dedicated.DedicatedServer.tickChildren(DedicatedServer.java:317)
        at net.minecraft.server.MinecraftServer.tickServer(MinecraftServer.java:917)
        at net.minecraft.server.MinecraftServer.runServer(MinecraftServer.java:707)
        at net.minecraft.server.MinecraftServer.lambda$spin$2(MinecraftServer.java:267)
        at java.lang.Thread.run(...)
```
- Déclencheur : une entité tick → la procédure `butcher` fait `level.getBlockState(pos)` sur une position dont le chunk n'est pas chargé → `getChunk(create=true, FULL)` → `getChunkBlocking`.
- Note : `tickSynchronously` indique qu'Async est retombé en exécution synchrone sur le Server thread pour cette entité — donc **l'appel fautif tourne bien sur le main thread**. Async n'est pas la cause (absent des cas 1/2) mais peut augmenter la fréquence d'accès "limites" à des chunks non chargés.

---

## 5. Pourquoi ça deadlock — analyse du mécanisme

### 5.1 Le contrat vanilla de `getChunkBlocking`
Dans le MC vanilla, `ServerChunkCache.getChunkBlocking` (appelé par `getChunk(..., create=true)` hors du main-thread-fast-path) fait, en simplifié :
```java
CompletableFuture<ChunkResult<ChunkAccess>> future = getChunkFutureMainThread(x, z, status, true);
this.mainThreadProcessor.managedBlock(future::isDone); // pompe les tâches jusqu'à complétion
return future.join().orElse(...);
```
Le `MainThreadExecutor.managedBlock` draine `mainThreadProcessor` **et** appelle `ServerChunkCache.this.pollTask()` (qui fait avancer le `ChunkMap`/distance manager). Les workers vanilla complètent la future en postant les étapes finales sur le `mainThreadProcessor`, que `managedBlock` exécute → la future se complète → on sort. **Le main thread se "self-drive".**

### 5.2 Ce que C2ME change
C2ME remplace le système de chunks par un scheduler asynchrone (le "chunk system rewrite", cf. `instrumentGetChunk` / `instrumentAwaitChunk`). Le chargement/génération est piloté par des executors C2ME + un scheduler. Les `@WrapMethod`/`@WrapOperation` `instrumentGetChunk`/`instrumentAwaitChunk` enveloppent respectivement `getChunk` et l'appel `managedBlock` interne.

### 5.3 L'hypothèse du livelock
Le dump montre que, **dans le wrapper `instrumentAwaitChunk`**, on entre quand même dans le `managedBlock` vanilla, qui boucle `runAllTasks() / waitForTasks()` sans jamais voir `future.isDone()` devenir vrai **ni** de tâche à exécuter. Conséquences logiques possibles :

- **(H1) Mauvaise file pompée.** La future de chunk de C2ME se complète via un **autre** mécanisme que le `mainThreadProcessor` vanilla (p.ex. une file de scheduler C2ME, ou un callback sur un thread worker qui ne re-poste pas sur `mainThreadProcessor`). `managedBlock` draine `mainThreadProcessor` → vide → park → la future ne progresse jamais sur le main thread. **C'est l'hypothèse la plus cohérente avec un park "waiting for tasks" indéfini.**

- **(H2) Étape de pipeline qui exige le main thread, mais n'est jamais soumise.** Promouvoir un chunk au statut `FULL` peut nécessiter de charger/générer les chunks voisins (rayon). Si l'une de ces étapes doit s'exécuter sur le main thread (p.ex. `delayFullChunkEvents` repousse les events "full chunk" au main thread) mais que `managedBlock` ne pompe pas la file où ces étapes atterrissent, on a un cycle : étape main-thread en attente du main thread, lui-même bloqué dans `managedBlock`.

- **(H3) Re-entrance / contexte invalide.** L'appel `getChunkBlocking` arrive dans un contexte où le scheduler C2ME n'est pas censé être pompé de façon ré-entrante (pendant `prepareLevels` (cas 1), pendant le handler de login (cas 2), au milieu d'un tick d'entité (cas 3)). Si `instrumentGetChunk` suppose que ces appels passent par le fast-path "chunk déjà chargé" et ne gère pas le chemin bloquant ré-entrant, la future est peut-être soumise mais jamais ticée.

- **(H4) Soumission perdue à cause d'un statut/flag.** `getChunkBlocking` peut, sous C2ME, retourner une future déjà "scheduled" mais dont l'avancement dépend d'un `tickScheduler`/`pollTask` que le `managedBlock` instrumenté n'invoque plus (si C2ME a remplacé `pollTask`/`runDistanceManagerUpdates` par un no-op et déplacé le travail ailleurs).

Le fait que les **3 cas** convergent vers le même park "waiting for tasks" pointe fortement vers **H1/H2** : le pont entre "future de chunk C2ME" et "drain du main thread dans `managedBlock`" est rompu pour le chemin **synchrone bloquant** depuis le main thread sur un chunk non chargé.

### 5.4 Condition nécessaire confirmée
Si le chunk est **déjà chargé** au statut demandé, `getChunkBlocking` retourne immédiatement (pas de `managedBlock`) → pas de deadlock. C'est pourquoi le bug ne frappe que des chunks **non chargés / distants / non promus**. À garder pour la reproduction et pour cibler le correctif (le fast-path "déjà chargé" marche ; c'est le **slow-path bloquant** qui casse).

---

## 6. Où regarder dans le code C2ME

Le préfixe mixinextras dans la pile (`…$c2me_base$instrumentGetChunk` / `…$c2me_base$instrumentAwaitChunk`) indique : **module/refmap `c2me-base`**, handlers `instrumentGetChunk` et `instrumentAwaitChunk`. Points de départ concrets :

1. **Grep direct** (le plus rapide) :
   ```
   grep -rn "instrumentAwaitChunk\|instrumentGetChunk" --include=*.java
   grep -rn "getChunkBlocking\|managedBlock"            --include=*.java
   grep -rn "@WrapMethod\|@WrapOperation"               --include=*.java | grep -i chunk
   ```
   → trouver le mixin du `c2me-base` qui cible `ServerChunkCache` (Yarn : **`ServerChunkManager`**) avec :
   - un `@WrapMethod` sur `getChunk(...)` → `instrumentGetChunk`
   - un `@WrapOperation` sur l'appel `managedBlock(...)` à l'intérieur de `getChunkBlocking` → `instrumentAwaitChunk`

2. **Le chunk system réécrit** (probable module séparé) — chercher les packages du type :
   ```
   com.ishland.c2me.rewrites.chunksystem.*
   com.ishland.c2me.opts.chunkio.* / threading.chunkio.*
   com.ishland.c2me.opts.scheduling.*
   ```
   Classes plausibles : `ChunkState`, `NewChunkStatus`, `ChunkLoadingManager`/mixin de `ChunkMap` (Yarn `ThreadedAnvilChunkStorage`), le scheduler "theFlash", la file/executor `MainThreadExecutor`.

3. **Le pont main-thread** — chercher comment C2ME pilote la complétion des futures de chunk et **qui** poste sur le `mainThreadProcessor` (Yarn : `ServerChunkManager.MainThreadExecutor`) :
   ```
   grep -rn "mainThreadProcessor\|MainThreadExecutor\|runAllTasks\|pollTask\|runDistanceManagerUpdates\|runJobsUntil" --include=*.java
   ```
   Vérifier si `instrumentAwaitChunk` (ou le scheduler) est censé, pendant l'attente, **pomper aussi** la file du scheduler C2ME — et si ce drain est manquant/contourné quand on entre via le slow-path bloquant.

4. **`@WrapMethod instrumentGetChunk`** — vérifier le fast-path vs slow-path : est-ce que pour un chunk non chargé il **soumet** correctement une demande au scheduler avant d'entrer dans `managedBlock`, ou est-ce qu'il délègue à `original.call()` (vanilla) qui suppose le pilotage vanilla disparu ?

### Tableau de correspondance Mojmap (dumps) ↔ Yarn (repo Fabric)
| Mojmap (NeoForge runtime) | Yarn (probable dans le repo) |
|---|---|
| `net.minecraft.server.level.ServerChunkCache` | `net.minecraft.server.world.ServerChunkManager` |
| `ServerChunkCache$MainThreadExecutor` | `ServerChunkManager$MainThreadExecutor` |
| `ServerChunkCache.getChunkBlocking` | `ServerChunkManager.getChunk(...)` chemin bloquant (`getChunkSync`/join) |
| `net.minecraft.util.thread.BlockableEventLoop` | `net.minecraft.util.thread.ThreadExecutor` |
| `BlockableEventLoop.managedBlock` / `waitForTasks` | `ThreadExecutor.runTasks(BooleanSupplier)` / `waitForTasks` |
| `net.minecraft.world.level.Level.getChunk` / `getBlockState` | `net.minecraft.world.World.getChunk` / `getBlockState` |
| `net.minecraft.server.level.ServerLevel` | `net.minecraft.server.world.ServerWorld` |
| `ChunkMap` | `ThreadedAnvilChunkStorage` |
| `ChunkStatus.FULL` | `ChunkStatus.FULL` |

> Le repo C2ME pour 1.21.1/NeoForge peut être une branche du monorepo `RelativityMC/C2ME-fabric` avec un sous-module/portage NeoForge, ou un repo "reforged" dédié. Les lignes `wrapMethod$zgg000$…` (0.93) vs `…$zgf000$…` (0.91) ne sont que des suffixes mixinextras générés — sans importance.

---

## 7. Reproduction en environnement de dev (minimal, sans le modpack)

Le but : reproduire **sans** Mekanism/FTB/butcher, pour itérer vite. La recette = forcer un `getChunk` synchrone d'un **chunk non chargé** sur le **main thread**.

1. Workspace : MC 1.21.1, NeoForge 21.1.214 (ou Fabric équivalent), C2ME buildé depuis le fork, config par défaut.
2. Choisir **l'un** de ces déclencheurs minimaux (du plus simple au plus proche prod) :
   - **(A) Commande de debug** : enregistrer une commande qui, sur le Server thread, appelle
     ```java
     ServerLevel level = ctx.getSource().getLevel();
     int cx = 10_000, cz = 10_000;           // loin du spawn, chunk non chargé/non généré
     level.getChunk(cx, cz);                  // = getChunk(cx, cz, FULL, true) → getChunkBlocking
     ```
     Exécuter en jeu → si le serveur fige avec la signature §3, c'est reproduit.
   - **(B) Force-chunk** : appeler `ForcedChunkManager.forceChunk(level, ...)` (NeoForge) sur un chunk non chargé pendant un tick → reproduit les cas 1/2.
   - **(C) getBlockState distant en tick** : event `EntityTickEvent.Pre` qui fait `level.getBlockState(new BlockPos(farX, y, farZ))` sur un chunk non chargé → reproduit le cas 3.
3. Capturer le thread dump : `jstack <pid>` (workspace dev a le JDK) ou `kill -3 <pid>`. Confirmer la signature §3.
4. **Bisection par config** pour localiser le sous-système fautif (chacune sur un run propre) :
   - `[chunkSystem] useLegacyScheduling = false` puis `= true`
   - `[chunkSystem] delayFullChunkEvents = false`
   - `[ioSystem] replaceImpl = false`
   - `[noTickViewDistance] enabled = false`
   - `[generalOptimizations] midTickChunkTasksInterval = -1` (désactive le mid-tick)
   Noter quelle combinaison fait disparaître le deadlock → pointe le module responsable.

> ⚠️ Attention : si le mod **Async (axalotl)** ou **C2ME** désactive/remplace `pollTask`/`runDistanceManagerUpdates`, le dépôt de reproduction doit être **C2ME seul** d'abord (cas 1/2 prouvent qu'Async n'est pas requis).

---

## 8. Méthodologie de diagnostic utilisée (réutilisable)

- L'image serveur est un **JRE** (pas de `jstack`/`jcmd`). Méthode : `kill -3 <pid_java>` (SIGQUIT) → la JVM dump tous les threads sur **stdout** (donc dans `docker logs`), **sans tuer** le process.
- Identifier le pid : `pgrep -f neoforge` (le pid 1 est le wrapper `mc-server-runner`, pas la JVM).
- **Confirmer un deadlock vs lenteur** : prendre **2 dumps** espacés de 10–30 s et comparer le champ `cpu=…ms` du `"Server thread"`. Ici : ~+1 à +1,4 s de CPU sur 15–30 s d'horloge ⇒ thread **parké** (livelock), pas en train de calculer.
- Repérer aussi l'état des workers C2ME (`"c2me-worker-N"`) : tous `WAITING (parking)` ⇒ rien en vol côté workers ⇒ la future n'est même pas en cours de traitement (cohérent avec H1/H4 : demande jamais réellement schedulée/avancée).

---

## 9. Faits saillants / contraintes pour le fix

- Le correctif doit faire en sorte que, sur le **chemin bloquant synchrone main-thread** (`getChunkBlocking` pour un chunk non chargé), **la future soit effectivement pilotée jusqu'à complétion pendant `managedBlock`** — soit en pompant la(les) bonne(s) file(s) du scheduler C2ME dans `instrumentAwaitChunk`, soit en garantissant que la complétion re-poste sur le `mainThreadProcessor` vanilla que `managedBlock` draine.
- Ne pas casser le fast-path "chunk déjà chargé" (qui marche).
- Doit fonctionner dans les 3 contextes : `prepareLevels` (avant `Done`), handler de login, milieu de tick d'entité. Ces 3 contextes tournent sur le **Server thread** mais à des phases différentes du cycle de vie — vérifier que le scheduler C2ME est "drivable" depuis chacun.
- Cibler aussi le ticket NeoForge : `ForcedChunkManager.forceChunk` / `reinstatePersistentChunks` appellent `getChunk`/`getBlockEntity` synchrones — chemin légitime côté plateforme, donc **c'est à C2ME de le supporter**, pas aux mods de l'éviter.
- Régression à surveiller : éviter de réintroduire un drain ré-entrant qui violerait les invariants de thread-safety du chunk system (sinon crash "Accessing LegacyRandomSource from multiple threads", cf. `[fixes] enforceSafeWorldRandomAccess`).

---

## 10. Plan d'attaque suggéré (à affiner une fois dans le repo)

1. **Localiser** `instrumentGetChunk` et `instrumentAwaitChunk` (grep §6.1) et lire intégralement le mixin `c2me-base` sur `ServerChunkManager` + le `MainThreadExecutor`.
2. **Cartographier** le chemin de complétion d'une future de chunk dans le chunk system réécrit : qui la complète, sur quel thread, via quelle file ? Identifier ce que `managedBlock`/`runAllTasks` draine réellement vs ce qui ferait avancer la future.
3. **Confirmer l'hypothèse** (probablement H1/H2) avec un point de repro minimal (§7-A) + logs/breakpoints : vérifier que la future est bien soumise au scheduler, et que pendant `managedBlock` rien ne tick ce scheduler.
4. **Bisection config** (§7.4) pour corréler avec un sous-module (chunk system rewrite vs IO vs notickvd vs mid-tick).
5. **Implémenter** le pilotage manquant : dans `instrumentAwaitChunk`, pendant l'attente, pomper aussi la file du scheduler C2ME (ou forcer la complétion à re-poster sur le `mainThreadProcessor`). Comparer avec la façon dont C2ME gère déjà l'attente bloquante *ailleurs* (il doit bien exister un chemin qui marche, p.ex. au worldgen/pregen) et réutiliser ce mécanisme.
6. **Tester** les 3 reproductions minimales (§7 A/B/C) → plus de park "waiting for tasks", chunk retourné, MSPT normal.
7. **Non-régression** : run avec `enforceSafeWorldRandomAccess = true` (défaut) pour s'assurer qu'on n'a pas introduit d'accès off-thread.

---

## Annexe A — extraits de logs de prod utiles

- `Done` jamais atteint au cas 1 (boot bloqué) ; au cas 2/3 `Done (5.x s)!` atteint puis fige.
- Juste avant le fige côté tick : `Can't keep up! Is the server overloaded? Running ~5000ms or ~100 ticks behind` (cohérent avec le main thread qui part en `managedBlock` infini).
- Workers C2ME observés `"c2me-worker-0".."5"` + `"c2me-sched"` + `"C2ME Storage #NN"`, tous parkés pendant le fige.
- Avertissement mixin (non lié au deadlock, mais présent) : conflit `@ModifyConstant` entre `packetfixer` et `connectivity` sur `loginTimeout` — sans impact sur ce bug.

## Annexe B — versions exactes des frames clés (pour matcher les lignes)
- `BlockableEventLoop.java:143` (`waitForTasks`), `:133` (`managedBlock`)
- `ServerChunkCache.java:533` (`MainThreadExecutor.managedBlock`), `:1750` (`getChunkBlocking`), `:1671` (`getChunk`)
- Wrappers C2ME : `ServerChunkCache.java:5132` (`instrumentAwaitChunk`), `:5146` (`instrumentGetChunk`) — *lignes synthétiques injectées par mixin, pas dans le source vanilla*.
- `Level.java:202` / `:5335` / `:5325` / `:5320` (`getChunk`/`getChunkAt`) ; `:5827` (`getBlockState`) ; `:777` (`getBlockEntity`).
