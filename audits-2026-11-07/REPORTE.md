# Auditoría ofensiva — Astar Network (código in-scope) — 2026-11-07

**Fecha:** 2026-11-07 (America/Caracas)
**Rama de trabajo:** `arena/2bf61c50-astar` (sesión Arena; sin commits de upstream modificados)
**Objetivo auditado:** runtime de Astar (parachain Polkadot, paraId 2006, EVM chain_id 592), pallets propios, precompiles EVM, XCM config, treasury vendored, collator.
**Conclusión ejecutiva: NO CRÍTICO LIVE encontrado** — y, más importante: **la verificación live NO pudo ejecutarse** (RPC/explorers/crates.io bloqueados por la política de egress del sandbox). Por tanto esta conclusión es **solo sobre el código fuente** `@ 02b78d7c`, no sobre el estado desplegado. Ver sección 8 (limitaciones) — léanla antes de usar este documento.

---

## 1. Alcance y fuente de verdad

- Repo principal del workspace (este checkout): `nightbox33-cloud/Astar` @ `02b78d7c4a177515e7e3602f09faec4baf340f35` (`master`, 2026-11-05, "Ci/compat para id session keys (#1709)").
- **Verificación anti-manipulación:** se hizo fetch de `AstarNetwork/Astar` (upstream oficial) `master` en el sandbox y se comparó el árbol:
  - `HEAD^{tree} == FETCH_HEAD^{tree} == eea8de53e3bad3467cab29070f951ba44657f76e`
  - `git fetch origin <sha>` devolvió el **mismo commit** `02b78d7...` → el fork `nightbox33-cloud/Astar` **no tiene drift** respecto al upstream master. El código auditado es el código público real.
- `git rev-list --count HEAD` = 1 (snapshot squashed por el fork), por eso el diff contra upstream es por árbol.
- PR #1709 (el commit HEAD) toca **solo CI** (`.github/workflows/runtime-compatibility.yml`, `third-party/zombienet/*`) — verificado vía `gh pr view 1709 --repo AstarNetwork/Astar`. Sin cambios de runtime.
- Dependencias pineadas en `Cargo.lock` (fuente de verdad fetched en el sandbox):
  | Dep | Ref | Rev pineado |
  |---|---|---|
  | `github.com/paritytech/polkadot-sdk` | branch `stable2603` | `ed30dfeb9818b48373a7aa69f6c6195677f9bad3` |
  | `github.com/AstarNetwork/frontier` | branch `stable2603` | `279bb1c8f184cbc8964c50c9b5c563d5ab3e2477` |
  | `github.com/rust-ethereum/evm` | branch `v0.x` | `a656db9050c65170b050360c3fa66c0fd8bf226a` |
- Sin `.gitmodules` en este commit (`vendor/` va in-tree: treasury, evm-tracing, rpc, etc.). Sin Git LFS detectado.
- Runtime versions (fuente): `spec_version = 2600` en `runtime/{astar,shiden,shibuya}`, `transaction_version = 3`, `authoring_version = 1`. Astar EVM `ChainId = 0x250 (592)`, SS58 = 5, paraId = 2006.

## 2. REQUISITO PREVIO OBLIGATORIO — estado

**Org auditada:** `AstarNetwork` (GitHub). No se encontraron orgs de fundación en otros hosts accesibles (GitLab/Bitbucket no verificados — no hay acceso; no se asume existencia).

### 2.1 Repos clonados
- **155/155** repos públicos visibles para el token clonados (`--depth 1`) en `/home/user/foundation-mirror/` (2.9 GB).
- **0** repos privados visibles para el token (`gh repo list AstarNetwork --limit 300` → 155 repos, 0 `isPrivate`). Repos privados existentes pero no accesibles/visibles: **bloqueados — no verificables, se desconoce su existencia**. Por tanto **no se declara cobertura completa** en el sentido de repos de la fundación.
- Log con hash exacto por repo: `audits-2026-11-07/clone-log.tsv` (155 líneas `OK <repo> <sha>`). Ejemplos relevantes:
  - `AstarNetwork/Astar` (workspace) = upstream master (ver §1)
  - `AstarNetwork/Audits` @ `25060988…` (reportes; los PDFs dApp-staking v3 y collective-proxy leídos en esta auditoría)
  - `AstarNetwork/substrate` @ `2026-11-01` push, `AstarNetwork/polkadot-sdk` @ `2026-08-12` push, `AstarNetwork/frontier` (polkadot-v0.9.43, push 2026-10-30) — forks de referencia, **no** usados por `Cargo.toml` (el build usa `paritytech/polkadot-sdk` y `AstarNetwork/frontier` pineados, ver §1)
  - `AstarNetwork/dapp-staking-reimbursement-script` @ `master` — script de reembolso histórico (ver §6.2)
- Commits registrados por repo en `clone-log.tsv`. No se auditaron en profundidad los ~150 repos de frontend/docs/forks sin valor para fondos on-chain; están clonados e inventariados, no revisados línea a línea (ver §8).

### 2.2 Anclaje a estado live — BLOQUEADO
- `https://rpc.astar.network` (y cualquier RPC/explorer): **TLS cortado por egress del sandbox** (`OpenSSL SSL_ERROR_SYSCALL` en ClientHello; reproducido 2 veces con `-v`).
- `https://index.crates.io` y `https://static.crates.io`: idem bloqueados → **build de Rust imposible** (además: 2 CPU / 3 GB RAM).
- Bloque ancla de mainnet: **NO DISPONIBLE**. No se pudo verificar: runtime desplegado == `spec_version 2600` de este árbol; params live (mínimo locked, rewards, caps); roles; balances; TVL; treasury; stake locked; bytecode de precompiles (los precompiles van en el wasm del runtime, no son cuentas EVM desplegables).
- Genesis (`bin/collator/res/astar.json`) sí está en el repo y se usó como referencia estática (§5), pero **no es el estado live**.

### 2.3 Mapa dirección/artefacto → repo/commit (estático)
| Dirección / artefacto | Tipo | Repo / commit | Verificación live |
|---|---|---|---|
| Runtime Astar `spec_version 2600` | wasm on-chain | `AstarNetwork/Astar` @ `02b78d7…` | **NO VERIFICADO** |
| Precompiles EVM: `0x1..0x9` (ECRecover, Sha256, Ripemd160, Identity, Modexp, Bn128*, Blake2F), `0x400..0x402` (1024 Sha3FIPS256, 1025 Dispatch, 1026 ECRecoverPublicKey), sr25519 `0x5000+a` = 20481..20487 | precompile set | idem + `AstarNetwork/frontier` @ `279bb1c8…` | genesis revert-code en `astar.json` (12 cuentas `0x000…0001..0x5000`); vivo NO verificado |
| Precompile XC-20 `0xFFFFFFFF…` (por asset id ≥ 2^64 / ≥ 0x1000007C0) | precompile set | idem | ids gestionados en docs (§6.4) |
| XCM config (`xcm_config.rs`), ReserveAssetFilter, SafeCallFilter | runtime | idem | NO VERIFICADO |
| `pallet-treasury` vendored (`vendor/treasury`, v37.0.0, "polkadot v1.15.0") | pallet | in-tree | revisión manual §6.5 |
| Treasury / CommunityTreasury (`PalletId py/treas`, `py/comtr`) | cuentas módulo | runtime | balance live NO MEDIBLE |
| Collator pot (collator-selection `PotId`) | cuenta módulo | pallet | NO MEDIBLE |
| paraId 2006, chain_id 592 | params | genesis/runtime | NO VERIFICADO |

## 3. FALLBACK — estado

**Activado** (no se encontró crítico en el código de fundación; sí se hizo el fallback de protocolos in-chain). Alcance limitado por §2.2 (sin RPC no se puede confirmar qué está desplegado ni verificar bytecode live). Protocolos in-chain enumerados (fuentes: docs en mirror + DefiLlama vía web search):

| Protocolo | Categoría | Repo clonado (`--depth 1`) | TVL Astar (aprox., DefiLlama abr-2025) |
|---|---|---|---|
| Starlay Finance (`starlay-finance/starlay-protocol`, fork de Aave v2; oracle DIA `0x35490A8AC7cD0Df5C4d7Ab4243A6B517133BcDB1` en Astar + FallbackOracle `0x35E6D71…`) | Lending | `@ 949fd67d76fdd5b2d55a38432449220ce8e38d94` | **≈ $131k** |
| Starlay Incentives Controller | Incentivos | `@ d137630b78186f586d410ad9e76675acfd2495d7` | — |
| StellaSwap (ex-Arthswap; DEX algebra) | DEX | `stellaswap/core @ 70e9b4be…`, `v2-core @ 4dd59067…`, `stable-amm @ 333e1d0b…` | muy bajo (histórico $16M→) |
| Zenlink DEX (`zenlinkpro/*`) | DEX pallet | NO clonado (fuera de Astar EVM de facto; pallet Wasm) | — |

**Resultado del fallback:**
- Starlay: revisión acotada de `markets/starlay/*` y layout (`LendingPool`, oracle DIA + fallback). Fork de Aave v2 auditado (CertiK 2022, en repo, `audit/Certik-starlay-04-14-2022.pdf`). Antecedente público: exploit 2024 (~$2.1M reportado) atribuido a error operativo del equipo, no a bug de contrato sin parchear. **No se identificó en esta pasada un crítico permissionless nuevo**, pero la revisión fue acotada (sin compilar, sin tests, sin fork live, sin verificación de bytecode/mercados activos). TVL tocable es de orden **$10²–10³**, no de terceros en riesgo relevante.
- StellaSwap `core`/`v2-core`/`stable-amm`: forks de estilo Uniswap-v2/Algebra; solo skim de estructura. Sin conclusión de criticidad. **Sin cobertura real de bytecode live.**
- **No se pudo** verificar que los repos coincidan con lo desplegado (sin RPC), ni correr `eth_call`/`simulateTransaction`, ni forkear mainnet. La parte "reconstruir desde bytecode" del fallback **no es ejecutable** sin acceso a RPC.
- Conclusión fallback: **no crítico encontrado; cobertura parcial y NO declarada completa** de DeFi in-chain. Los montos en juego (Starlay ≈$131k) son de orden muy inferior a cualquier tesoro/fondo de fundación teórico.

## 4. Superficie revisada (código de fundación) con veredicto

| Componente | Qué se revisó | Veredicto |
|---|---|---|
| `pallets/dapp-staking` (v3, 2825+1948 LOC) | extrinsics lock/unlock/claim/stake/unstake/move_stake/epoch handler/`compute_tier_rewards`/`DAppTierRewards::try_claim`/ledger freeze (`total_locked_amount`) | sin crítico; ver §6.1–6.3 |
| `pallets/inflation` | `StakingRewardHandler`, recalculation, mint (`Balanced`), pools | sin crítico; mint estrictamente acotado por pools/decaimiento |
| `pallets/collator-selection` (+commit HEAD #1709 = CI puro) | bonds, kick/slash, note_author, new_session | sin crítico; diseño intencional (§6.6) |
| `pallets/xc-asset-config` | todos los extrinsics son `ManagerOrigin = Root` | sin hallazgos permissionless |
| `pallets/collective-proxy` | call filter, collective origin | SR Labs 2024-11: "no security issues"; revisión propia coherente |
| `pallets/dynamic-evm-base-fee` | dinámica de fee | sin hallazgos |
| `vendor/treasury` (v37.0.0) | propose/approve/spend/burn | sin hallazgos; approve/reject exigen origin de gobernanza (§6.5) |
| `precompiles/xcm` | `assets_withdraw`, `reserve_transfer`, `transfer*`, `remote_transact`, `send_xcm` (deshabilitado), `redirect_relay_to_asset_hub`, `ensure_dot_transfer_policy`, `resolve_transfer_type` | sin crítico; ver §6.4 |
| `precompiles/assets-erc20` | transfer/approve/transferFrom/allowance | estándar Frontier; saturación max en approve; sin mint |
| `precompiles/dapp-staking` | dispatch a pallet dapp-staking con origen del caller | sin crítico |
| `precompiles/dispatch-lockdrop` / `precompiles/dispatch` (1025) vía runtime `WhitelistedCalls`/`WhitelistedLockdropCalls` | whitelist: DappStaking, Assets::transfer, democracy/treasury/preimage, batch | sin crítico; orígenes de gobernanza siguen exigidos (§6.7) |
| `primitives::evm` (`EvmRevertCodeHandler`), `EVMFungibleAdapterWrapper` | creación/destroy XC-20 ↔ revert code EVM | sin hallazgos |
| `primitives::xcm` (`ReserveAssetFilter`, `FixedRateOfForeignAsset`, split location) | lógica de reserva/fees XCM | sin hallazgos; docs-ids XC20 disjuntos de creación permissionless (§6.4) |
| `runtime/astar` xcm_config, BaseFilter, SafeCallFilter, EVM config | barrier, transact filter, ChainId, gas | sin hallazgos; `WithOriginFilter` aplica SafeCallFilter incluso a Root (verificado en polkadot-sdk pineado) |
| `tests/` (suites xcm-simulator/integration/dapp-staking) | presencia, no ejecutables | no ejecutados (§8) |

## 5. Inventario medido (solo estático; live BLOQUEADO)

- **Emisión genesis** (`astar.json`, 11 cuentas): **7,000,000,100 ASTR**. Top genesis: 2.1e9, 1.05e9, 7e8 ×3, 3.5e8 ×5 (cuentas de fundación/ecosistema por estructura de genesis, no por actividad live).
- **Collator candidacy bond:** 3.2e24 planck = **3.2M ASTR**; `desiredCandidates = 200`; invulnerables 2 (`5HmwQvvke…`, `5Ekik1AYt…`).
- **EVM genesis accounts:** 12 cuentas revert-code (0x1..0x9, 0x400..0x402, 0x5000=20480).
- **Gestionados XC20 (docs):** DOT id = `340282366920938463463374607431768211455` (u128::MAX); USDT id = `4294969280`; ACA/aUSD/LDOT/GLMR/INTR/PHA/BNC/vDOT/CLV/vsDOT/PINK id = `18446744073709551616..633`; vs creación permissionless Astar `id < 4294967295` (u32::MAX−1). **Disjuntos → no hay squatting de ids gestionados.** Precompiles XC20 en `0xFFFFFFFF…<id>`.
- **XC reserves:** `ReserveAssetFilter` solo acepta sibling paraId propio o Asset Hub (1000) para DOT/KSM-like `(1, [])`.
- **Bloqueado:** issuance live, treasury balances, dApp-staking locked/staked, TVL, precios, finality, cantidad de collators. Sin RPC no se puede medir ni refutar nada live.
- Antecedente medido (script oficial `dapp-staking-reimbursement-script/reimburse.csv`): sobre-pago de recompensas dApp v3 ≈ **103,816.73 ASTR** reembolsado por la fundación (12 beneficiarios). Era de la ventana bloques 6,216,237 → 6,285,334.

## 6. Análisis de los candidatos a crítico (los 4 criterios) y descartes

### 6.1 SR Labs 2024-01 “critical” (double-spend vía unlock/relock) — MITIGADO, verificado en este árbol
- El freeze del pallet se aplica sobre `ledger.total_locked_amount()` (activo + unlocking) en `update_ledger` (`pallets/dapp-staking/src/lib.rs`), no sobre `active_locked_amount`. Con esto, la PoC de "transferir free-balance y relockear" ya no crea stake fantasma.
- Criterios 1–4: falla criterio 1 hoy (mitigación presente). **No es crítico live.**

### 6.2 Sobre-pago de recompensas dApp v3 (incidente histórico, ≈103,817 ASTR) — MITIGADO, verificado
- `compute_tier_rewards` normaliza por `max(observed_total_weight, expected_full_weight)` ⇒ suma de claims de un tier ≤ `tier_allocation` (prueba: Σ claims = alloc · observed/normalization ≤ alloc). `try_claim` toma `base + rank × rank_reward`.
- `reimburse.csv` (12 entradas, suma ≈ 1.038e23 planck = **103,816.73 ASTR**) documenta el impacto histórico y su reembolso.
- Criterio 1: mitigación presente. **No es crítico live.** (Es el precedente más cercano a un "critical live" y está cerrado.)

### 6.3 Rounding/math en `Perbill::from_rational` y pools (staker/bonus/dapp)
- Todos los payouts están acotados por el pool del era/periodo (`staker_reward_pool`, `bonus_reward_pool`, `DAppTierRewards`). Rounding siempre contra el beneficiario (a favor del sistema). No hay camino permissionless de mint fuera de pools. **No crítico.**

### 6.4 XCM / XC-20 / `remote_transact` (precompile 20484)
- `send_xcm` está deshabilitado (`SEND_XCM_UNSUPPORTED`). `remote_transact` construye `WithdrawAsset(fee) → BuyExecution(Unlimited) → Transact(SovereignAccount, caller)` y deriva el origin como `AccountId32{network: UniversalLocation.global_consensus(), id: caller}` — el origen es la cuenta soberana **del caller**, no de terceros; el fee sale del propio caller; la llamada va a parachain sibling (relay prohibido).
- `WithOriginFilter<SafeCallFilter>` (verificado en polkadot-sdk `xcm-executor/src/traits/conversion.rs` en el rev pineado) añade filtro de calls incluso a orígenes Root: solo System/Balances/Vesting/DappStaking/Assets/Session/Proxy/Multisig seleccionados; sin EVM/Ethereum/PolkadotXcm send/execute. Relay no puede ejecutar calls arbitrarios vía Transact.
- `ReserveAssetFilter`: solo reserva sibling propia o Asset Hub para DOT-like; `redirect_relay_to_asset_hub` reescribe destino `parent` con activo local → AssetHub.
- Falta de verificación live (qué XC assets están registrados en `XcAssetConfig` hoy) = **no verificable** — riesgo residual de gobernanza, no un bug de permisos.
- **No crítico** (falla criterios 1 y 3 sin ancla live; además el flujo que existe no saca fondos de terceros).

### 6.5 `vendor/treasury` vendored (v37.0.0, legacy)
- `propose_spend` (bond), `approve_proposal`/`reject_proposal` exigen `ApproveOrigin`/`RejectOrigin` (gobernanza). `spend_funds` reparte según approvals; burn de superávit salvo `missed_any`. Sin hooks de fundación hardcodeados; `OnSlash`/`SpendFunds` inyectados por config (Treasury → CommunityTreasury para instancias secundarias). Sin hallazgo permissionless. El hecho de que sea legacy/viejo es riesgo de mantenimiento, no un critical explotable. **No crítico.**

### 6.6 `collator-selection` `note_author` y slash de bonds
- `note_author` paga la mitad del pot al autor (diseño); `kick_stale_candidates` expulsa y slashea candidatos inactivos en `new_session` (automático, tiempo-disparado, no requiere rol). El slashed bond va a `SlashDestination` (configurable por UpdateOrigin) o se retiene. El PnL de un atacante pasivo es negativo por construcción (bond en riesgo). No mueve fondos de terceros sin que medien bonds propios en juego. **No crítico** (falla criterios 2–4: no hay delta > 0 extraíble sin riesgo propio).

### 6.7 Precompiles de dispatch EOA-only (1025 `Dispatch`, 20487 `DispatchLockdrop`)
- Whitelist `WhitelistedCalls`: `Utility::{batch,batch_all}` recursivo, `DappStaking(*)`, `Assets::transfer`, `Democracy(*)`, `Treasury(*)`, `CommunityTreasury(*)`, `Preimage(*)`. Solo EOA (checks `()` — no callable por contratos/precompiles).
- Los calls de gobernanza (`Democracy`, `Treasury`, `Preimage`) exigen orígenes internos (proporciones/Root) que un EOA no satisface; el resto (`DappStaking`, `Assets::transfer`) son signed-origin normales. Nada en la whitelist permite mover fondos ajenos. **No crítico.**

### 6.8 `BaseFilter` permissionless `Assets::create` con `id < u32::MAX`
- Permite crear assets arbitrarios (spam/scam), by design. Ids XC20 gestionados son ≥ 4294969280 > u32::MAX → no squatting de precompiles XC20 existentes. **No crítico** (criterio 4: sin delta de terceros).

### 6.9 Otros revisados sin hallazgos relevantes
- `assets-erc20` (cancel_approval → approve_transfer; saturación a max en approve grande — semántica ERC-20 estándar); `sr25519`/`substrate-ecdsa` verify (views que devuelven bool; seguridad de firma es del contrato consumidor); `EvmRevertCodeHandler` (collision → falla creación; destroy → borra code); `EVMFungibleAdapterWrapper` (tips → CollatorRewardPot, fees → DealWithFees con 80/20 burn/collator); `SafeMode`/`TxPause` (orígenes Root); runtime APIs; `construct_runtime` (sin Sudo en producción — `astar-sudo` es feature de dev; indexes correctos).

### Tabla de criterios (resumen)
| Candidato | C1 bug vivo | C2 permissionless | C3 call-graph live | C4 PoC Δ>0 terceros | ¿Crítico? |
|---|---|---|---|---|---|
| 6.1 relock/unlock double-spend | ❌ mitigado | — | — | — | NO |
| 6.2 tier-reward overpay | ❌ mitigado | — | — | — | NO |
| 6.4 remote_transact XCM | ✔ código vivo | ✔ | ⚠ no anclado live | ❌ (fee+origen propios) | NO |
| 6.6 kick/slash collators | ✔ | ✔ | ✔ | ❌ (bond propio en riesgo) | NO |
| 6.7 dispatch whitelist | ✔ | ✔ | ✔ | ❌ (orígenes internos mandan) | NO |
| 6.8 assets create | ✔ | ✔ | ✔ | ❌ | NO |

## 7. Respuesta al objetivo ("atacante sin rol con más valor del que puso, de fondos de terceros")

**No encontrado.** Ningún camino permissionless revisado en el código `@ 02b78d7c` (== upstream master) mueve valor de terceros hacia el atacante con Δ > 0 tras el unwind. Los dos históricos de sobre-pago están parcheados y verificados en el árbol. Todo lo "caro" (mint, spend de treasury, cambio de params inflation/tiers/xc-assets/safemode/upgrades) está detrás de orígenes de gobernanza/Root, fuera del criterio "permissionless".

**Qué NO se puede afirmar:** que el *runtime desplegado* sea este árbol; que no existan bugs en los deps pineados (polkadot-sdk/frontier — solo revisión puntual, no auditoría); que los DeFi in-chain no tengan uno (ver §3, cobertura parcial); que no haya drift entre `spec_version 2600` del repo y el wasm on-chain.

## 8. Limitaciones y huecos de cobertura (leer antes de confiar en este informe)

1. **RPC/explorers/APIs externas bloqueadas** por egress (TLS reset documentado §2.2): sin bloque ancla, sin balances/params/bytecode live, sin fork PoC, sin verificación de runtime desplegado.
2. **Sin build** (crates.io y crates index bloqueados; 2 CPU/3 GB): no se compiló ni se corrió la suite de tests (`pallets/*/src/tests.rs`, xcm-simulator). Los veredictos son de lectura estática.
3. **Repos privados de la fundación:** invisibles para el token; no verificados. Posible código relevante (ops, infra, keys tooling) fuera de alcance.
4. **Forks de referencia clonados pero no auditados** (`AstarNetwork/substrate|polkadot-sdk|frontier|polkadot|cumulus|astar-frame` ≈ 1 GB, 2023–2026): no se hizo review de su diff vs upstream — el build usa `paritytech/polkadot-sdk` y `AstarNetwork/frontier` pineados (§1), que sí se clonaron (polkadot-sdk fetch `--depth 1` del SHA exacto, frontier y evm ídem) pero solo se inspeccionaron puntualmente (xcm-executor, frontier evm/precompile dispatch).
5. **~150 repos de la fundación** (UI, docs, forks legacy, pallets experimentales): clonados e inventariados en `clone-log.tsv`, **no revisados**; se asume irrelevantes para fondos on-chain (ningún hallazgo deriva de ellos).
6. **DeFi in-chain:** cobertura parcial (§3); TVL inmediato ≈ $131k en Starlay. Sin verificación de bytecode ni de mercados activos; sin fork tests.
7. **Shiden/Shibuya:** mismo árbol, no auditados por separado; spec_version 2600 en los tres.
8. Reporte generado 2026-11-07; estado live puede cambiar (SafeMode, upgrades) sin cambiar este documento.

## 9. Recomendaciones (defensivas, no explotables)

- Verificar on-chain con un nodo propio (no este sandbox): `spec_version` del runtime live == 2600; hash del wasm vs build de este commit; balances de treasury/community-treasury/collator-pot; `ActiveProtocolState` (mantenimiento); `SafeMode`/`TxPause`; IDs XC20 registrados en `XcAssetConfig` vs docs (§5); `Candidates`/`Invulnerables`.
- Monitoreo continuo: emisión efectiva vs `InflationConfiguration`; suma de `EraRewards` reclamados vs pools; `DAppTiers` suma por era ≤ Σ `reward_portion × dapp_reward_pool`; `Ledger` freeze consistency (`try_state` del pallet ya lo chequea — considerarlo en try-runtime CI).
- Reivindicar el requisito previo con acceso de solo-lectura a RPC y a repos privados de la fundación antes de declarar cobertura completa; correr `try-runtime` + tests en CI con el toolchain pineado (`rust-toolchain.toml`, canal 1.93.0).
- Mantener el enfoque de reembolso (§6.2) y su transparencia; publicar el postmortem del incidente si existe.
- Starlay/ecosistema: exigir a la fundación el reporte de los audits vigentes (CertiK incompleto para incentivos) antes de considerar TVL protegible.

## 10. Evidencia, comandos y artefactos

Todos bajo `/home/user/Astar` (repo) y `/home/user/foundation-mirror` (mirror) y `/home/user/inchain-deps` (fallback):

```bash
# Upstream vs snapshot (anti-tamper)
git fetch --depth 1 https://github.com/AstarNetwork/Astar.git master
git rev-parse HEAD^{tree} FETCH_HEAD^{tree}   # idénticos: eea8de53…
# Requisito previo (155/155 OK)
/home/user/foundation-mirror/clone_all.sh     # escribe clone-log.tsv (copiado a audits-2026-11-07/)
# Dependencias pineadas
git init deps/frontier && git -C deps/frontier fetch --depth 1 origin 279bb1c8f184cbc8964c50c9b5c563d5ab3e2477
git init deps/polkadot-sdk && git -C deps/polkadot-sdk fetch --depth 1 origin ed30dfeb9818b48373a7aa69f6c6195677f9bad3
git init deps/evm && git -C deps/evm fetch --depth 1 origin a656db9050c65170b050360c3fa66c0fd8bf226a
# Fallback (DeFi in-chain)
git clone --depth 1 https://github.com/starlay-finance/starlay-protocol        # 949fd67d
git clone --depth 1 https://github.com/starlay-finance/starlay-incentives-controller # d137630b
git clone --depth 1 https://github.com/stellaswap/core                          # 70e9b4be
git clone --depth 1 https://github.com/stellaswap/v2-core                       # 4dd59067
git clone --depth 1 https://github.com/stellaswap/stable-amm                    # 333e1d0b
# Prueba de bloqueo egress
curl -sv https://rpc.astar.network/ -d '{"id":1,"jsonrpc":"2.0","method":"system_health","params":[]}'  # SSL_ERROR_SYSCALL
curl -sI https://static.crates.io/config.json                                                      # SSL_ERROR_SYSCALL
# Lectura de reportes de auditoría oficiales
pip install --break-system-packages pypdf   # extrajo texto de los PDFs en foundation-mirror/Audits
# Inventario genesis
python3 - <<'EOF'
import json; d=json.load(open('bin/collator/res/astar.json'))
b=d['genesis']['runtime']['balances']['balances']
print(len(b), sum(int(x[1]) for x in b)/1e18)  # 11 cuentas, 7000000100 ASTR
EOF
```

- `audits-2026-11-07/clone-log.tsv` — 155 líneas `OK <repo> <sha>`.
- `foundation-mirror/Audits/reports/` — PDFs leídos (dApp Staking v3 SR Labs 2024-01; Collective proxy SR Labs 2024-11).
- `foundation-mirror/dapp-staking-reimbursement-script/reimburse.csv` — suma 1.038e23 planck = 103,816.73 ASTR.

## 11. Veredicto

**No crítico live** en el código auditado (`@ 02b78d7c`, == upstream master AstarNetwork/Astar, runtime spec_version 2600). **Sin inventario live medible** (RPC/explorers bloqueados): emisión genesis 7.0000001e9 ASTR; TVL dApp-staking/treasury/DeFi live **no cuantificable en este sandbox**. **Cobertura no completa**: repos privados invisibles; fallback in-chain parcial; runtime desplegado no verificado; sin compilación/tests/fork PoC. Antes de operación real (whitelist, despliegue, mesa de fondos), reelaborar el requisito previo con acceso RPC de lectura y verificar runtime/params on-chain.

---

## 12. Addendum 2026-11-07 (segunda pasada, post-entrega)

Pregunta del dueño: existe "un crítico" adicional según chats previos; verificarlo, incluso por RPC con Python.

**Prueba de alcanzabilidad RPC (Python `urllib`, 2 lotes, ~23 endpoints):** `rpc.astar.network`, `astr.api.onfinality.io`, `astr-rpc.publicnode.com`, `astr.drpc.org`, `1rpc.io/astr`, `astr.llamarpc.com`, `astr.stacks.euro`, `astar-mainnet.rpcfast.com`, `astar.unitedbloc.com`, `astar.awesometools.su`, `rpc.astravalon.tech`, `astar-rpc.blockdaemon.com`, `astermainnet.rpc.subquery.network`, `astr.meowrpc.com`, `unitedbloc.com`, `rpc.astar.org`, `unique.network` endpoints, etc. **Resultado: 100% fallo** — TLS `SSL_ERROR_SYSCALL` / `Connection reset` en todos los que resuelven; `gaierror` / `Temporary failure in name resolution` en el resto. El egress allowlist es el mismo que con `curl` (`github.com`, `codeload.github.com`, `api.github.com`, `registry.npmjs.org`, `pypi.org`, `files.pythonhosted.org`). **Ningún estado live fue obtenido.** No se intentó contornear el filtro de red.

**Áreas nuevas revisadas en la segunda pasada (sin hallazgos críticos):**
- XCM: `xcm_config.rs` — barrier `AllowTopLevelPaidExecutionFrom<Everything>` (pago exigido vía `Trader`), teleports deshabilitados (`TeleportFilter = Nothing`), reserva transfer solo instrucciones, `SafeCallFilter` en `CallDispatcher`, `FixedRateOfForeignAsset` vía `XcAssetConfig` (sin mapping → error XCM, nunca default), `AssetLocationIdConverter` → `JustTry` (ubicación no registrada → fallo), `AccountId32Aliases`/`Account32Hash`/`ParentIsPreset`/`SiblingParachainConvertsVia` sin origen privilegiado; `XcmExecuteFilter = Nothing`.
- Inflación `on_initialize`/`on_finalize`: `DoRecalculation` único, `force_*` todos `ensure_root`; `payout_reward` acuña vía `Balanced::deposit` + drop de `Debt` (aumento de emisión contabilizado); pools por era/período/bloque acotados por `max_inflation_rate * issuance` (`is_valid`: partes suman 1); `staker_and_dapp_reward_pools` divide por `ideal_staking_rate` (Perquintill `/` Perquintill, rate validado root-only); reclamos (staker/dApp/bonus) eliminan la entrada reclamada → sin doble cobro; total reclamado ≤ pool (stake/…).
- Tiers: `RankedTier` acotado (`MAX_RANK=10`, `tier ≤ 0xf`); `compute_tier_rewards` normaliza por `max(observed, expected)` → suma distribuida ≤ alloc; `find_rank` con `.min(MAX_RANK)` y `try_into`→`unwrap_or_default`.
- Bonus: `StakerInfo::remove` tras `claim_bonus_reward`; pool acotado por `Perbill(eligible/total_vp_stake)` del snapshot `PeriodEnd`.
- Claims staker: pool por era no decrementado pero proporcional `amount/staked` → suma ≤ pool (stake nunca reclamado sin estar en `staked` snapshot); contador `EraRewardSpan` (larga) → expiración = pérdida del reward, no robo.
- Contabilidad dual `CurrentEraInfo` (current/next bucket) con entradas `era`-anotadas: unstake sustrae de cada bucket una sola vez (validado con trazas `stake→migrate→unstake` y `CompactStakingInfo`).
- Migración `dapp-staking` V11→V12 (`VersionedMigration`, una sola vez, fallo → `maintenance = true`), sin acuñar/mover fondos.
- Precompiles `assets-erc20` `mint/burn`: despachan `pallet_assets` con origen del caller; el permiso lo hace cumplir el pallet (owner/admin del asset); sin escalada. `dapp-staking` precompile: `claim_*` con origen del caller y destino correcto (beneficiario o cuenta), `claim_dapp_reward` callable por cualquiera pero paga al `reward_beneficiary` (comportamiento intencional documentado). `dispatch-lockdrop`: origen = `blake2_256(pubkey_comprimida)`, debe poseer pubkey ECDSA cuya dirección EVM == caller → cuenta derivada propia del caller.
- Gobernanza `primitives/governance.rs`: solo `EnsureProportionAtLeast` stock + `EitherOfDiverse` — sin aritmética de fracciones propia.
- Genesis: 7.0000001e9 ASTR en 11 cuentas (dato histórico, no live).

**Conclusión:** no se localiza ningún crítico nuevo en el código fuente revisado en esta segunda pasada; la afirmación de "crítico pendiente" no pudo ser verificada. Los únicos caminos que no están descartados por el código sino por el entorno son los que requieren estado on-chain (spec_version 2600 real, wasm vs. fuente, balances/treasury/dapp-staking actuales, SafeMode activo, registros XC). Para verificarlo se necesita: (a) un endpoint RPC de lectura **permitido explícitamente** para este sandbox o un **nodo archivo local** apuntando a Astar, o (b) el área/índice/extrinsic sospechoso o los hallazgos de los chats previos para enfocarlo, o (c) aceptar el veredicto estático.
