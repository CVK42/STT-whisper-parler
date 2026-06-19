# Feature : Double mode de collage (Ctrl+Win / Ctrl+Alt)

> **Statut :** Implémentée, puis **revertée** (le 2026-06-15) car elle ne fonctionnait pas comme attendu.  
> **Commit de la feature :** `8470371` (présent dans le reflog git local tant que `git gc` n'est pas passé)  
> **Commit stable actuel :** `64928f8`

Ce document explique comment restaurer ou re-implémenter la feature.

---

## Ce que fait la feature

| Raccourci | Au repos | En cours d'enregistrement |
|---|---|---|
| `Ctrl gauche + Win gauche` | **Démarre** l'enregistrement | **Stoppe** + colle le texte + appuie sur **Entrée** |
| `Ctrl gauche + Alt gauche` | **Rien** (stop-only) | **Stoppe** + colle le texte **sans Entrée** |

- Post-processing IA (OpenRouter / Gemini Flash Lite) toujours actif sur les deux raccourcis.
- AltGr (Alt droit) n'interfère pas car on utilise `alt_left` (gauche uniquement).
- `keyboard_implementation` doit être `handy_keys` (supporte les raccourcis modificateurs-seuls).

---

## Option 1 — Restaurer via cherry-pick (le plus rapide)

```powershell
# Vérifier que le commit de la feature est encore accessible
git log --oneline --all | Select-String "8470371"

# Si oui, cherry-pick
git cherry-pick 8470371

# Rebuild
$env:LIBCLANG_PATH = "C:\Program Files\LLVM\bin"
$env:LIB = "C:\Program Files (x86)\Windows Kits\10\Lib\10.0.26100.0\um\x64;C:\Program Files (x86)\Windows Kits\10\Lib\10.0.26100.0\ucrt\x64"
$env:PATH = "C:\Users\ChristianKourajian\.bun\bin;C:\Users\ChristianKourajian\.cargo\bin;C:\Program Files\LLVM\bin;C:\Program Files\CMake\bin;" + $env:PATH
bun run tauri:dev:parlerdev
```

Si le cherry-pick échoue (commit GC'd), utiliser l'Option 2 ci-dessous.

---

## Option 2 — Re-implémenter manuellement

### Fichiers à modifier (dans l'ordre)

#### 1. `src-tauri/src/settings.rs`

**Objectif :** Ajouter le binding `transcribe_no_submit` dans les défauts, changer le raccourci par défaut de `transcribe` vers `ctrl_left+super_left`.

Dans la fonction `get_default_settings()` (vers ligne 713-793), trouver le bloc `bindings` et faire :

```rust
// Changer le binding par défaut de "transcribe" (Windows seulement)
// Remplacer "ctrl+space" par "ctrl_left+super_left"

// Ajouter après le binding "transcribe_with_post_process" :
bindings.insert("transcribe_no_submit".to_string(), ShortcutBinding {
    id: "transcribe_no_submit".to_string(),
    name: "Transcribe (no Enter)".to_string(),
    description: "Stops the current recording and pastes the text without pressing Enter.".to_string(),
    default_binding: "ctrl_left+alt_left".to_string(),
    current_binding: "ctrl_left+alt_left".to_string(),
});
```

Le backfill automatique dans `load_or_create_app_settings()` injectera ce nouveau binding dans les settings persistés sans migration manuelle.

---

#### 2. `src-tauri/src/transcription_coordinator.rs`

**Objectif :** Permettre à `ctrl+alt` de stopper un enregistrement démarré par `ctrl+win`, sans que `ctrl+alt` puisse démarrer.

Trouver `is_transcribe_binding` (vers ligne 46-48) :

```rust
fn is_transcribe_binding(id: &str) -> bool {
    id == "transcribe" || id == "transcribe_with_post_process"
}
```

Modifier en :

```rust
fn is_transcribe_binding(id: &str) -> bool {
    id == "transcribe" || id == "transcribe_with_post_process" || id == "transcribe_no_submit"
}

fn can_start_recording(id: &str) -> bool {
    // transcribe_no_submit est stop-only — ne peut pas démarrer
    id == "transcribe" || id == "transcribe_with_post_process"
}
```

Dans la logique toggle (vers ligne 100-115), branche `is_pressed`, modifier le match sur `Stage::Idle` pour utiliser `can_start_recording` :

```rust
Stage::Idle => {
    if can_start_recording(&binding_id) {
        start(&app, &mut stage, &binding_id, &hotkey_string);
    } else {
        debug!("Ignore '{binding_id}' at idle (stop-only binding)");
    }
}
// Pour le stop cross-binding : n'importe quel binding transcribe peut stopper
Stage::Recording { binding_id: ref bid, .. }
    if is_transcribe_binding(bid) && is_transcribe_binding(&binding_id) => {
    // Le binding de STOP (binding_id) décide du comportement (Entrée ou non)
    stop(&app, &mut stage, &binding_id, &hotkey_string);
}
```

---

#### 3. `src-tauri/src/actions.rs`

**Objectif :** Ajouter un champ `submit: bool` à `TranscribeAction` et câbler les deux bindings.

Trouver `TranscribeAction` (vers ligne 50-52) :

```rust
// Avant
pub struct TranscribeAction {
    pub post_process: bool,
}

// Après
pub struct TranscribeAction {
    pub post_process: bool,
    pub submit: bool,
}
```

Dans `ACTION_MAP` (vers ligne 841-862), mettre à jour les entrées :

```rust
"transcribe" => TranscribeAction { post_process: true, submit: true },
"transcribe_no_submit" => TranscribeAction { post_process: true, submit: false },
"transcribe_with_post_process" => TranscribeAction { post_process: true, submit: true },
```

Dans la fonction de stop de `TranscribeAction` (vers ligne 620, après `let post_process = self.post_process;`), ajouter :

```rust
let submit = self.submit;
```

Au moment du paste (vers ligne 754), remplacer :

```rust
// Avant
utils::paste(final_text, ah_clone.clone())
// Après
utils::paste_with_submit(final_text, ah_clone.clone(), Some(submit))
```

---

#### 4. `src-tauri/src/clipboard.rs`

**Objectif :** Permettre un override de `auto_submit` par raccourci.

Trouver `pub fn paste(...)` (vers ligne 591) et renommer/modifier :

```rust
// Nouvelle signature avec override optionnel
pub fn paste_with_submit(
    text: String,
    app_handle: AppHandle,
    submit_override: Option<bool>,
) -> Result<(), String> {
    // ...
    let settings = /* récupérer les settings comme avant */;
    let auto_submit = submit_override.unwrap_or(settings.auto_submit);
    // reste identique à la logique existante de paste()
}

// Wrapper de compat pour les autres appelants
pub fn paste(text: String, app_handle: AppHandle) -> Result<(), String> {
    paste_with_submit(text, app_handle, None)
}
```

Dans `utils/mod.rs` (ou l'endroit où `utils::paste` est réexporté), ajouter `paste_with_submit`.

---

#### 5. `src/components/settings/general/GeneralSettings.tsx`

**Objectif :** Afficher le nouveau binding dans l'UI.

Après la ligne `<ShortcutInput shortcutId="transcribe" grouped={true} />`, ajouter :

```tsx
<ShortcutInput shortcutId="transcribe_no_submit" grouped={true} />
```

---

#### 6. `src/components/settings/advanced/AdvancedSettings.tsx`

**Objectif :** Rendre le sélecteur `keyboard_implementation` toujours visible (pas caché derrière `experimental_enabled`), pour que l'utilisateur puisse confirmer/changer `handy_keys` sans activer le mode expérimental.

Déplacer `<KeyboardImplementationSelector />` du bloc `{experimentalEnabled && (...)}` vers le groupe "app" (toujours visible) :

```tsx
<SettingsGroup title={t("settings.advanced.groups.app")}>
  <StartHidden ... />
  <AutostartToggle ... />
  {/* ... autres settings ... */}
  <KeyboardImplementationSelector descriptionMode="tooltip" grouped={true} />
</SettingsGroup>
```

---

#### 7. `src/i18n/locales/fr/translation.json` et `en/translation.json`

Ajouter les traductions pour le nouveau binding (la clé exacte dépend du pattern existant — chercher `"transcribe_with_post_process"` pour voir la structure attendue).

---

#### 8. `settings_store.json` (app fermée)

**Fichier :** `C:\Users\ChristianKourajian\AppData\Roaming\com.melvynx.parler.dev\settings_store.json`

Vérifier / corriger manuellement avec l'app **fermée** :

```json
"keyboard_implementation": "handy_keys",
"transcribe": {
  "current_binding": "ctrl_left+super_left",
  ...
},
"transcribe_no_submit": {
  "current_binding": "ctrl_left+alt_left",
  "default_binding": "ctrl_left+alt_left",
  ...
}
```

---

## Pourquoi la feature a été revertée

La feature a été construite et compilée avec succès (build OK, pas de crash). L'utilisateur a signalé "ça ne marche pas" sans préciser le symptôme exact. Les causes possibles à investiguer si on re-tente :

1. **`keyboard_implementation` basculé en `tauri`** — le sélecteur était caché, l'app peut retomber sur `tauri` si `handy_keys` échoue à l'init (voir `shortcut/mod.rs` lignes 43-54). Corriger : s'assurer que `handy_keys` est actif via les logs ou le sélecteur dégaté.
2. **`settings_store.json` écrasé par l'app** — si le fichier a été édité pendant que l'app tournait, l'app l'a réécrit au close, perdant les changements. Solution : éditer avec l'app fermée.
3. **Conflit de binding** — si `ctrl_left+super_left` était déjà assigné à `copy_latest_history` ou autre, handy_keys peut l'ignorer silencieusement. Vérifier dans les logs (`log_level: "debug"`).
4. **Cross-binding stop non déclenché** — la logique de stop cross-binding est conditionnée par `is_transcribe_binding(bid) && is_transcribe_binding(&binding_id)`. Si le `bid` stocké lors du start ne correspond pas exactement à la string attendue, la branche ne s'exécute pas.

---

## Build (rappel des env vars Windows)

À lancer au début de chaque session PowerShell avant `bun run tauri:dev:parlerdev` :

```powershell
$env:LIBCLANG_PATH = "C:\Program Files\LLVM\bin"
$env:LIB = "C:\Program Files (x86)\Windows Kits\10\Lib\10.0.26100.0\um\x64;C:\Program Files (x86)\Windows Kits\10\Lib\10.0.26100.0\ucrt\x64"
$env:PATH = "C:\Users\ChristianKourajian\.bun\bin;C:\Users\ChristianKourajian\.cargo\bin;C:\Program Files\LLVM\bin;C:\Program Files\CMake\bin;" + $env:PATH
bun run tauri:dev:parlerdev
```

---

## Tests end-to-end après restauration

1. **Ouvrir Notepad.**
2. **Test Ctrl+Win start → Ctrl+Win stop (avec Entrée)** : parler, stopper avec `ctrl+win` → texte post-traité collé + curseur à la ligne suivante.
3. **Test Ctrl+Win start → Ctrl+Alt stop (sans Entrée)** : parler, stopper avec `ctrl+alt` → texte post-traité collé, curseur reste sur la même ligne.
4. **Test stop-only au repos** : au repos, presser `ctrl+alt` seul → **rien ne démarre**.
5. **Test AltGr** : taper `@` ou `€` (Alt droit) pendant l'usage normal de Windows → aucun déclenchement parasite.
6. **Logs debug** : ouvrir `%APPDATA%\com.melvynx.parler.dev\logs\` pour vérifier qu'aucun fallback vers `tauri` n'a eu lieu.
