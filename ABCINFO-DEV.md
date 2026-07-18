# Abcinfo-Remote — Documentation de développement (EXHAUSTIVE)

> Fork RustDesk brandé **Abcinfo-Remote** pour Eric Miermon Informatique (abcinfo.ch).
> Client d'assistance à distance, réception seule, pré-configuré sur un relais RustDesk auto-hébergé en Suisse.
> **Ce document est LA référence.** Repo : `github.com/nanofjah/rustdesk` (public, branche `master`).
> Dernière mise à jour : 2026-07-18 — **projet en production, les 2 plateformes signées/notarisées et en ligne.**

---

## 0. Résumé ultra-court
- On **fork RustDesk** (AGPLv3) et on le **rebrand** (design, français, réception seule, serveur+clé gravés) via **GitHub Actions**.
- Le build sort **non signé** → on le **signe localement sur le Mac d'Eric** : Windows via **Certum (jsign)**, Mac via **Developer ID Apple + notarytool**.
- On **publie** les 2 binaires sur **`https://remote.abcinfo.ch`** (VPS + Caddy).
- Client Windows : `https://remote.abcinfo.ch/pc` — Client Mac : `https://remote.abcinfo.ch/mac`.

---

## 1. Infrastructure & valeurs clés

| Élément | Valeur |
|---|---|
| **Relais RustDesk (serveur)** | `remote.abcinfo.ch` |
| **Clé publique du relais (RS_PUB_KEY)** | `e7DV9f24+bwd6FsgWoukIdHxJsqa+xf+xzVx9JaTC08=` |
| **VPS** | `179.237.84.38`, user `ubuntu`, SSH clé `~/.ssh/id_ed25519` (alias possible). Ubuntu, Docker + Caddy. |
| **Repo fork (build brandé)** | `github.com/nanofjah/rustdesk` — branche `master` — **PUBLIC** (obligatoire AGPL) |
| **Clone local du fork** | `/tmp/rustdesk-fork` — **éphémère**, source de vérité = GitHub. Toujours `git pull` avant de bosser. |
| **Repo portail** | `github.com/nanofjah/remote-abcinfo` (branche `main`) — local `~/Projets/remote-abcinfo/` |
| **Mémoire Claude** | `~/.claude/projects/-Users-eric/memory` → symlink NAS `/Users/eric/bigNas/nestorproject/SharedAgentMemory` → repo `github.com/nanofjah/claude-memory` (branche `master`). ⚠️ voir §9. |

Le relais lui-même (hbbs+hbbr OSS + Caddy) vit sur le VPS dans `/opt/rustdesk/` et `/opt/caddy/`. Il est **en prod et testé** (connexion chiffrée E2E, jamais via serveurs publics RustDesk).

---

## 2. Le fork & le workflow CI

### 2.1 Workflow
- Fichier : `.github/workflows/flutter-build.yml` (workflow officiel RustDesk, **trimmé**).
- Jobs actifs : `generate-bridge`, `build-RustDeskTempTopMostWindow`, **`build-for-windows-flutter`** (x86_64-pc-windows-msvc), **`build-for-macOS`** (matrice `aarch64-apple-darwin` **+** `x86_64-apple-darwin`). Les autres = `if: false`.
- Étape « Publish Release » / « Publish DMG package » = **`if: false`** (sinon 403 sur un fork).
- Déclenchement : `gh workflow run flutter-build.yml --repo nanofjah/rustdesk --ref master` (a `workflow_dispatch`).
- Durée : ~40-50 min. Artefacts utiles :
  - Windows : **`abcinfo-installers-windows-x86_64`** → contient l'**exe portable single-file** (`SignOutput/rustdesk-<ver>-x86_64.exe`, se lance sans installer) + un `.msi`.
  - macOS : **`rustdesk-unsigned-macos-aarch64`** et **`rustdesk-unsigned-macos-x86_64`** → chacun un `.dmg` non signé contenant `Abcinfo-Remote.app`.

### 2.2 Injection serveur/clé/nom au build (config.rs)
Le serveur, la clé, le nom d'app et le mode réception-seule sont **injectés au build** (pas commités dans le submodule) par des étapes du workflow qui patchent `libs/hbb_common/src/config.rs` :
- `RENDEZVOUS_SERVERS = &["remote.abcinfo.ch"]`
- `RS_PUB_KEY = "e7DV9f24…C08="`
- `APP_NAME` (RwLock) → `Abcinfo-Remote`
- `is_incoming_only()` forcé à `true` (réception seule → panneau « contrôler » masqué)
- **Windows** : étape en `sed` (Git Bash = GNU sed). **macOS** : étape séparée en **perl** (runners macOS = BSD sed, `sed -i` incompatible). Les deux jobs ont leur propre étape d'injection après le checkout.

---

## 3. Les modifications de branding (fichiers)

### 3.1 Nom de l'application « Abcinfo-Remote »
- **Windows** : `flutter/windows/CMakeLists.txt` → `set(BINARY_NAME "Abcinfo-Remote")` (⚠️ doit égaler APP_NAME sinon install/service/tray/raccourci cassés). Workflow self-extract `-e …Abcinfo-Remote.exe` ; MSI `preprocess.py --app-name "Abcinfo-Remote"`.
- **macOS** : `flutter/macos/Runner/Configs/AppInfo.xcconfig` → `PRODUCT_NAME = Abcinfo-Remote` (c'est l'xcconfig qui pilote, pas le pbxproj `$(TARGET_NAME)`). `build.py` L420 patché (`cp service …Abcinfo-Remote.app/…`). Refs `RustDesk.app`→`Abcinfo-Remote.app` dans le workflow (create-dmg/codesign).
- **Bundle id macOS** : `com.carriez.rustdesk` → **`com.abcinfo.remote`** (les 3 `PRODUCT_BUNDLE_IDENTIFIER` du **pbxproj** + xcconfig). Motif : les Réglages Système (TCC : Enregistrement écran/Accessibilité/Saisie) identifient l'app par bundle id → affichaient « RustDesk ». Maintenant « Abcinfo-Remote » + cohabite avec le RustDesk standard d'Eric. **SAFE** : `src/platform/macos.rs` `correct_app_name()` substitue dynamiquement `com.carriez.rustdesk`→bundle id réel et `RustDesk`→app_name dans les plists launchd du daemon non-surveillé.

### 3.2 Design de l'écran (tout dans `flutter/lib/desktop/pages/desktop_home_page.dart` + `common.dart`)
- Thème **sombre forcé** (`getThemeModePreference`→dark dans common.dart), accent **ambre `0xFFFFB300`**.
- Fond **matrice** (widget `_AbcMatrix`) ; police **Share Tech Mono** (TTF dans `flutter/assets/` + `pubspec.yaml`).
- Titre **`>_ abcinfo`** (orange) + curseur clignotant + tagline « Service en informatique » (`_AbcTitle`).
- Pied de page 2 lignes **cliquables** (www.abcinfo.ch / tél / mailto), orange (`_AbcFooter`). Pas de « Genève ».
- Champs **ID** et **mot de passe** : cadre **orange carré** (Border.all `0xFFFFB300`, sans borderRadius) **uniquement autour de la valeur** ; `filled: false` + `fillColor: transparent` → le fond (matrice) transparaît. Valeurs en orange.
- Pavé install **sans cadre** ; bouton **« Installer »** = `FixedWidthButton` orange, `radius:0` (coins carrés).
- Lien **« Définir un mot de passe permanent »** (§4) sous le mot de passe.

### 3.3 Dialogue « Définir le mot de passe » (`desktop_setting_page.dart` `setPasswordDialog`)
- Titre orange, `fontSize 15`, Flexible+maxLines 2 (ne déborde pas).
- Règles de robustesse en **2 lignes fixes centrées avec tirets** : `Chiffre - Majuscule` / `Minuscule - Longueur >8` (Column de 2 Row, labels custom colorés gris `0xFF94A3B8`→orange `0xFFFFB300` via `rules[i].validate(pass)`).
- Boutons **Annuler/OK** = `OutlinedButton` orange compacts (padding réduit, `tapTargetSize.shrinkWrap`), centrés, dans un `FittedBox` (tiennent sur 1 ligne).
- **Focus des champs en orange** : `focusedBorder` UnderlineInputBorder + `floatingLabelStyle` + `cursorColor` `0xFFFFB300` sur p0/p1 (au lieu du bleu par défaut).

### 3.4 Langue : français forcé par défaut
- `src/flutter_ffi.rs` fn `translate` → force locale `"fr"` (UI Flutter).
- `src/lang.rs` fn `translate` → force `"fr"` (textes Rust : tray, notifications).
- Un choix de langue enregistré (option `lang`) reste prioritaire via `resolve_lang`.

### 3.5 Copyright / About / AGPL (`desktop_setting_page.dart` + métadonnées)
RustDesk est **AGPL-3.0** → on **préserve l'attribution** et on ajoute la nôtre. « Purslane Tech Pte. Ltd. » remplacé par **`© <année> Eric Miermon Informatique · basé sur RustDesk (© Purslane Tech, AGPLv3)`** dans :
- Fenêtre About Flutter : titre « À propos d'Abcinfo-Remote » + copyright + lien « Site web »→abcinfo.ch + **lien « Code source (AGPLv3) »** → `github.com/nanofjah/rustdesk`.
- macOS `AppInfo.xcconfig` `PRODUCT_COPYRIGHT`.
- Windows `flutter/windows/runner/Runner.rc` (CompanyName/FileDescription/LegalCopyright/ProductName).
- `src/main.rs` author.

**Conformité AGPL** : repo `nanofjah/rustdesk` **PUBLIC** (aucun secret : clé injectée = PUBLIQUE ; secrets signature = pas dans le repo) + lien source dans l'app. C'est obligatoire car on distribue le binaire à des clients.

### 3.6 Fonctionnel : accès non surveillé
- `desktop_home_page.dart` `initState` : au lancement, `mainSetOption(verification-method=use-both-passwords)` + `approve-mode=password` + `startService()`. → mot de passe usage-unique affiché + permanent possible + module serveur actif.
- **PAS de mot de passe gravé** (décision : secret partagé = dangereux). L'utilisateur pose un **mot de passe permanent à l'installation**, via le bouton dédié → `setPasswordDialog()` → `mainSetPermanentPasswordWithResult` (**IPC vers le service déjà élevé → pas d'UAC, pas de page Sécurité verrouillée**). Requiert le client **installé** (service actif).
- macOS non-surveillé : cartes affichées 1 à 1 (Écran→Accessibilité→Saisie via `buildHelpCards`), puis carte « Install » (daemon `mainIsInstalledDaemon`) qui exige `mainIsInstalled()` (app dans **/Applications**). Le daemon/agent launchd ont `RunAtLoad=true`+`KeepAlive` → démarrent au boot, accès même à l'écran de login.

### 3.7 Icônes
- **Icône exe/app/Dock** (couleur, navy + `>_` ambre) :
  - Windows : `flutter/windows/runner/resources/app_icon.ico`, `res/icon.ico`, `res/tray-icon.ico`.
  - macOS : `flutter/macos/Runner/AppIcon.icns` (régénérée via `iconutil` depuis `res/mac-icon.png` 1024px).
  - Fenêtre/Dock Flutter (toutes plateformes) : **`flutter/assets/icon.png`** lu par `loadIcon()`.
- **Icône TRAY macOS** (barre de menus) : DOIT être un **template** (transparent, `>_` monochrome). `res/mac-tray-dark-x2.png` = SVG sans fond, `>_` noir, rendu 60×60 (`rsvg-convert`). `src/tray.rs` : sur macOS utilise **l'image template embarquée** ; sur Windows/Linux utilise `load_icon_from_asset()` (icône couleur). Voir §9 (bug historique).

---

## 4. Signature Windows (Certum, sur le Mac)

### Certificat
- **Certum Code Signing in the Cloud** (acheté via revendeur **SSLmentor**, ~177 $/1 an). Order CA ID `aaefd48c-a45f-48dd-bd02-13ae5814d185`. CN `Eric Miermon Informatique` (OV, nom d'entreprise). Exp 2026-07-16 → **2027-07-16**.
- **Compte SimplySign = `eric@abcinfo.ch`** (système CA Certum = source de vérité). ⚠️ **PAS** `contact@abcinfo.ch` (= uniquement le login revendeur SSLmentor ; Certum ne le connaît pas).

### Outils (déjà installés sur le Mac)
- **SimplySign Desktop** (`/Applications/SimplySign Desktop.app`) : app **barre de menus** (LSUIElement=true, pas de Dock/fenêtre). Fournit la lib PKCS#11 : `/usr/local/lib/libSimplySignPKCS.dylib`. Install : `brew fetch --cask simplysign` puis lancer le `.pkg` **à la main** (sudo interactif).
- **SimplySign mobile** (téléphone / ou app iOS-sur-Mac) : génère les **TOTP** et valide.
- **jsign** (`brew install jsign`).

### Procédure (à chaque version)
1. Ouvrir **SimplySign Desktop**, cliquer son icône dans la **barre de menus** (peut être cachée par l'encoche du MacBook), **se connecter** : `eric@abcinfo.ch` + **TOTP** du mobile. Session **~2 h**, signatures illimitées.
2. Vérifier le token (sans PIN) : `pkcs11-tool --module /usr/local/lib/libSimplySignPKCS.dylib -L` → doit montrer `Slot 0 … token label: Code Signing … CERTUM`. « No slots » = pas connecté. Si le token apparaît mais `jsign` sort `CKR_FUNCTION_FAILED` → **session expirée** : `pkill -f "SimplySign Desktop"` puis `open -a "SimplySign Desktop"` et se reconnecter.
3. Config `pkcs11.cfg` :
   ```
   name = SimplySign
   library = /usr/local/lib/libSimplySignPKCS.dylib
   slotListIndex = 0
   ```
4. Signer (⚠️ **aucun PIN** — `--storepass ""` : le token est `pin min/max 0/0`, c'est la session Desktop qui authentifie ; ne jamais tenter de PIN au hasard = risque blocage carte) :
   ```
   jsign --keystore pkcs11.cfg --storetype PKCS11 --storepass "" \
     --alias 339161E30BBBE0ABB83D2474E0B208B4 \
     --tsaurl http://time.certum.pl  <fichier.exe>
   ```
   - **alias = LABEL du certificat** = `339161E30BBBE0ABB83D2474E0B208B4` (retrouvable via `pkcs11-tool … -O --type cert`).
5. Vérifier : `osslsigncode verify <fichier.exe>` → Subject `CN=Eric Miermon Informatique…`, Issuer `Certum Code Signing 2021 CA`, timestamp présent.

---

## 5. Signature + notarisation macOS (Apple, sur le Mac)

### Compte / cert
- **Apple Developer Individual** (99 $/an, Apple ID `eric@abcinfo.ch`). Organization refusé (raison individuelle ≠ personne morale). D-U-N-S `483681933` gardé pour plus tard.
- **Certificat** : `Developer ID Application: eric miermon (7GMWH9L5RL)` — dans le trousseau login. **Team ID = `7GMWH9L5RL`**.
- **notarytool** : profil trousseau **`abcinfo-notary`** créé par `xcrun notarytool store-credentials "abcinfo-notary" --apple-id "eric@abcinfo.ch" --team-id "7GMWH9L5RL"` (+ mot de passe spécifique app d'appleid.apple.com). ⚠️ **ce profil DISPARAÎT régulièrement du trousseau entre sessions** (cause floue) → le recréer avec la même commande. **TODO durable : clé API App Store Connect (.p8), plus stable.**

### Le CI ne produit PAS l'universel — on le fait en LOCAL
Le workflow build 2 arch séparées. On les **fusionne au `lipo`** sur le Mac, puis signe + notarise. Xcode + notarytool présents.

### Procédure complète (script, dossier de travail ex. `/tmp/mac-final`)
```bash
CERT="Developer ID Application: eric miermon (7GMWH9L5RL)"
ENT=/tmp/rustdesk-fork/flutter/macos/Runner/Release.entitlements
PROFILE=abcinfo-notary
RID=<run id du build>

# 1) Télécharger les 2 arch, extraire l'app de chaque dmg (hdiutil attach)
gh run download $RID --repo nanofjah/rustdesk -n rustdesk-unsigned-macos-aarch64 -D dl-arm
gh run download $RID --repo nanofjah/rustdesk -n rustdesk-unsigned-macos-x86_64 -D dl-x64
#   → app-aarch64.app  et  app-x86_64.app

# 2) Fusion universelle : base = x86_64 (superset, contient les libswift), lipo chaque Mach-O commun avec l'arm64.
#    (voir script complet dans l'historique ; résultat : main = "x86_64 arm64")

# 3) Signature inside-out (dylibs → frameworks → service → app), hardened runtime + timestamp + entitlements :
find "$UNI/Contents/Frameworks" -type f -exec sh -c 'file "$1"|grep -q Mach-O && codesign --force --timestamp --options runtime -s "$0" "$1"' "$CERT" {} \;
find "$UNI/Contents/Frameworks" -depth -type d -name "*.framework" -exec codesign --force --timestamp --options runtime -s "$CERT" {} \;
codesign --force --timestamp --options runtime --entitlements "$ENT" -s "$CERT" "$UNI/Contents/MacOS/service"
codesign --force --timestamp --options runtime --entitlements "$ENT" -s "$CERT" "$UNI"
codesign --verify --deep --strict "$UNI"
#   ⚠️ ~40 binaires horodatés (appel réseau chacun) = LENT → faire en tâche de fond. 1er accès clé : "Toujours autoriser".

# 4) Notariser l'APP + staple :
ditto -c -k --keepParent "$UNI" app.zip
xcrun notarytool submit app.zip --keychain-profile "$PROFILE" --wait   # → status: Accepted
xcrun stapler staple "$UNI"

# 5) Créer le dmg (avec l'app stapled) :
mkdir stage && cp -R "$UNI" stage/ && ln -s /Applications stage/Applications
hdiutil create -volname "Abcinfo-Remote" -srcfolder stage -ov -format UDZO Abcinfo-Remote-universal.dmg

# 6) Notariser le DMG + staple :
xcrun notarytool submit Abcinfo-Remote-universal.dmg --keychain-profile "$PROFILE" --wait
xcrun stapler staple Abcinfo-Remote-universal.dmg

# 7) Vérifier : spctl -a -vvv "$UNI"  → "accepted, source=Notarized Developer ID"
```

---

## 6. Déploiement sur le portail (VPS)

- Portail statique servi par **Caddy** : `remote.abcinfo.ch`. Repo `~/Projets/remote-abcinfo/` → `nanofjah/remote-abcinfo` (branche `main`).
- Sur le VPS : `/opt/caddy/Caddyfile`, `/opt/caddy/site/index.html`, binaires dans `/opt/caddy/site/downloads/`. Conteneur Docker = `caddy` (Caddyfile monté `/etc/caddy/Caddyfile`, site `/srv/site`, volume `caddy_data` = certs ACME persistants).
- Routes Caddy :
  - `/pc` → `Abcinfo-Remote.exe` (Content-Disposition + rewrite `/downloads/Abcinfo-Remote.exe`). **Plus d'astuce config-dans-le-nom** (serveur+clé compilés dans l'app).
  - `/mac` → `Abcinfo-Remote.dmg` (rewrite `/downloads/Abcinfo-Remote-mac.dmg`).
  - Android **retiré** (clients quasi tous iOS ; iOS non contrôlable de toute façon).
- Déploiement d'un binaire :
  ```bash
  scp -q <fichier> ubuntu@179.237.84.38:/tmp/<nom>
  ssh ubuntu@179.237.84.38 'sudo mv /tmp/<nom> /opt/caddy/site/downloads/<nom> && sudo chmod 644 /opt/caddy/site/downloads/<nom>'
  ```
- Après un changement de **Caddyfile** : `sudo docker exec caddy caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile` puis **`cd /opt/caddy && sudo docker compose up -d --force-recreate`** (⚠️ `caddy reload` seul ne suffit PAS toujours — utiliser force-recreate).
- Changer index.html/downloads seuls = pas de reload nécessaire (file_server lit à chaud).
- Vérif live : `curl -sI https://remote.abcinfo.ch/pc` et `/mac` (HTTP 200 + content-length attendu).

---

## 7. Le contrôleur d'Eric (pour dépanner ses clients)
- Eric utilise le **RustDesk officiel standard** (pas brandé) comme contrôleur.
- Config **une fois** : Réglages → Réseau → ID/Relay Server : **ID Server** `remote.abcinfo.ch`, **Key** `e7DV9f24…C08=`. Sans ça il tape sur les serveurs publics.
- Favoris = **locaux** par machine, **pas de limite** codée (liste dans la config). Carnet synchronisé = RustDesk **Server Pro** (payant) — non retenu.

---

## 8. Décisions figées / écartées
- **OSS pas Pro**, **Caddy pas Nginx**, **build brandé from-source** (pas l'astuce exe-renommé/pkg de l'ancienne approche — obsolète).
- **Android abandonné** (clients iOS ; + politique Google stricte sur remote/Accessibilité). Play Store écarté.
- **iOS non supporté** par RustDesk en réception (Apple l'interdit).
- **Azure Trusted Signing abandonné** : la **Suisse n'est pas éligible** → Certum retenu.
- Mot de passe **non gravé** (posé à l'install).

---

## 9. PIÈGES CONNUS (⚠️ à relire avant de toucher au projet)

1. **`.gitignore` `*png` / `*jpg`** : bloque **silencieusement** les nouveaux assets (`flutter/assets/icon.png`, `flutter/assets/hero-bg.jpg`) lors d'un `git add -A` → l'asset n'est jamais commité → absent des builds → l'app retombe sur l'asset RustDesk. **Des exceptions ont été ajoutées** (`!flutter/assets/icon.png`, `!flutter/assets/hero-bg.jpg`). **RÈGLE : après création de tout asset, vérifier `git ls-files <fichier>`.** (M'a coûté 2 rebuilds : hero-bg.jpg puis icon.png.)
2. **Icône tray macOS = template** : `with_icon_as_template(true)` transforme tout pixel opaque en blanc → une icône pleine (fond navy) devient un **bloc blanc**. Il FAUT une image transparente monochrome (`>_`). Windows/Linux gardent l'icône couleur. Corrigé dans `src/tray.rs` (branche `#[cfg(target_os="macos")]`).
3. **Profil notarytool `abcinfo-notary` disparaît** du trousseau entre sessions (récurrent). Le recréer (`store-credentials`). **TODO : passer à une clé API App Store Connect (.p8).**
4. **Session SimplySign expire ~2 h** : token visible mais `jsign` → `CKR_FUNCTION_FAILED`. → redémarrer SimplySign Desktop + reconnexion.
5. **SmartScreen « fichier peu téléchargé »** : normal pour un binaire/cert neufs (réputation), **s'estompe**. Aucun certificat n'y échappe depuis que MS a supprimé l'avantage EV (2026). Accélérer : soumettre sur `microsoft.com/en-us/wdsi/filesubmission` (rôle « software developer »).
6. **macOS « sélecteur de fenêtre » à l'install** = permission **Enregistrement d'écran** (ScreenCaptureKit) = **normal Apple**, pas un bug.
7. **`caddy reload` insuffisant** parfois → `docker compose up -d --force-recreate`.
8. **BSD sed sur runners macOS** : l'injection config.rs du job macOS doit être en **perl**, pas `sed -i`.
9. **Rebuild à chaque version RustDesk upstream** : les patchs (config.rs, tray.rs, design) peuvent casser si l'upstream bouge — à revérifier.
10. **Mémoire NAS peut se vider** : le dossier `SharedAgentMemory` (symlink NAS) s'est retrouvé vide une fois → **restaurer depuis GitHub** (`git clone github.com/nanofjah/claude-memory` dedans). GitHub = le vrai filet.

---

## 10. TODO / améliorations possibles (optionnel)
- **Clé API App Store Connect (.p8)** pour la notarisation (fin du profil qui s'évapore).
- **Signature Windows automatisée en CI** (Certum : jsign + TOTP généré, ou conteneur p11-kit `hpvb/certum-container`) — actuellement signature **locale** par version.
- **Mac : signature/notarisation en CI** (secrets MACOS_P12_BASE64/PASSWORD/CODESIGN_IDENTITY/NOTARIZE_JSON déjà câblés dans le workflow, désactivés).
- **Windows ARM** (rare) et **binaire universel Mac généré en CI** (job lipo).
- **Lien caché « contrôleur »** pré-configuré pour Eric (RustDesk officiel + config-in-filename) — non fait.
- Améliorer la **réputation SmartScreen** (soumission MS).
