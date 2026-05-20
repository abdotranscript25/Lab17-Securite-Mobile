# 🔐 Lab 17 – Sécurité mobile : OWASP Uncrackable Level 3

## 🎯 Objectif du laboratoire

Ce laboratoire est un **exercice de reverse engineering** sur une application Android volontairement sécurisée. L'objectif est de **contourner toutes les protections** pour trouver le mot de passe secret.

| Protection | Technique utilisée | Contournement |
|------------|-------------------|---------------|
| **Anti-root** | Détection de l'environnement rooté | Patch du smali |
| **Anti-tampering** | Vérification du CRC des fichiers | Suppression du message d'erreur |
| **Anti-debug** | `ptrace()` dans le code natif | Patch librairie native (RET) |
| **Anti-Frida** | Scan de `/proc/self/maps` | Patch librairie native |
| **Vérification d'intégrité** | CRC sur `.so` et DEX | Patch du `verifyLibs()` |
| **Obfuscation native** | LCG, malloc, listes chaînées | Extraction clé + XOR Python |

---

## 🧠 Concepts clés abordés

### 1. Outils utilisés

| Outil | Rôle |
|-------|------|
| **Jadx-GUI** | Décompiler l'APK pour lire le code Java |
| **apktool** | Décompiler/recompiler l'APK, modifier le smali, extraire les librairies |
| **Ghidra** | Analyser et patcher la librairie native (`.so`) |
| **uber-apk-signer** | Signer l'APK patchée pour pouvoir l'installer |
| **adb** | Installer et déboguer sur l'émulateur |
| **Python** | Déchiffrer la clé encodée (XOR) |

### 2. Structure de l'APK cible

```
UnCrackable-Level3.apk
├── classes.dex              → Code Java (converti en smali)
├── lib/
│   ├── arm64-v8a/
│   │   └── libfoo.so        → Code natif (C/C++)
│   └── x86_64/
│       └── libfoo.so
├── resources.arsc
└── AndroidManifest.xml
```

### 3. Qu'est-ce que le smali ?

Le **smali** est la représentation lisible du bytecode Java (dex). C'est ce qu'on modifie avec apktool pour patcher le comportement de l'application.

```smali
.class public Lsg/vantagepoint/uncrackable3/MainActivity;
.super Landroid/app/Activity;

.method private showDialog(Ljava/lang/String;)V
    .locals 1
    const-string v0, "Rooting detected"
    invoke-virtual {p0, v0}, ...
.end method
```

### 4. Qu'est-ce que Ghidra ?

Ghidra est un décompilateur open-source développé par la NSA. Il transforme le code binaire (`.so`) en pseudo-code C lisible.

```c
// Pseudo-code Ghidra - fonction _INIT_0
void sub_73D0(void) {
    // Vérifie /proc/self/maps pour "frida", "exposed"
    // Vérifie ptrace() pour anti-debug
    // Si problème → goodbye()
}
```

---

## 🏗️ Architecture de l'application cible

```
┌─────────────────────────────────────────────────────────────────────┐
│                    UnCrackable-Level3.apk                           │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                   Java (smali)                                │  │
│  │                                                               │  │
│  │  MainActivity.java                                            │  │
│  │  • verifyLibs()    → vérifie CRC des .so et du DEX           │  │
│  │  • onCreate()      → root + tampering → showDialog() / quit  │  │
│  │  • check.check_code() → appel natif pour validation          │  │
│  └──────────────────────────────┬────────────────────────────────┘  │
│                                 │ JNI                               │
│                                 ▼                                   │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                  Native (libfoo.so)                           │  │
│  │                                                               │  │
│  │  • .init_array    → exécuté au chargement de la lib          │  │
│  │  • Scan /proc/self/maps → cherche "frida", "exposed"         │  │
│  │  • ptrace()       → anti-debug                               │  │
│  │  • check_code()   → validation du mot de passe               │  │
│  │  • Obfuscation : LCG, malloc, listes chaînées                │  │
│  │  • Clé encodée en little-endian (XOR)                        │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🔄 Pipeline complet du reverse engineering

```
ÉTAPE 1 — Analyse statique avec Jadx
  └── Ouvre l'APK → lit le code Java → comprend les protections
      • verifyLibs()  → vérifie CRC des fichiers
      • onCreate()    → root + tampering detection
      • check_code()  → validation native (JNI)
        ↓
ÉTAPE 2 — Décompilation avec apktool
  └── apktool d UnCrackable-Level3.apk -o uncrackable3
      → smali/ (code modifiable) + lib/ (librairies natives)
        ↓
ÉTAPE 3 — Patch du smali (couche Java)
  └── Modifier MainActivity.smali :
      Remplacer l'appel à showDialog() par return-void
      → L'app ne quitte plus même si root/tampering détecté ✅
        ↓
ÉTAPE 4 — Patch de la librairie native avec Ghidra
  └── Ouvrir libfoo.so dans Ghidra
      • Fonction _INIT_0 (init_array) → première instruction = RET
      → Anti-debug et anti-Frida désactivés ✅
      • Exporter la librairie patchée
        ↓
ÉTAPE 5 — Patch final de CodeCheck (validation)
  └── Modifier CodeCheck.smali :
      Remplacer check_code par return true
      → N'importe quel mot de passe est accepté ✅
        ↓
ÉTAPE 6 — Reconstruction + Signature
  └── apktool b uncrackable3 -o UnCrackable-Level3-patched.apk
      uber-apk-signer -a UnCrackable-Level3-patched.apk
        ↓
ÉTAPE 7 — Installation et test
  └── adb uninstall owasp.mstg.uncrackable3
      adb install UnCrackable-Level3-patched-aligned-debugSigned.apk
      → L'app s'ouvre sans message d'erreur ✅
```

---

## 🔑 Commandes essentielles

| Commande | Rôle |
|----------|------|
| `apktool d app.apk -o dossier` | Décompiler l'APK |
| `apktool b dossier -o app-patched.apk` | Reconstruire l'APK |
| `uber-apk-signer -a app-patched.apk` | Signer l'APK |
| `adb install -r app-patched.apk` | Installer l'APK |
| `adb uninstall package.name` | Désinstaller l'application |
| `adb shell getprop ro.product.cpu.abi` | Vérifier l'architecture |
| `jadx-gui` | Analyser le code Java |
| `ghidraRun` | Analyser le code natif |

---

## 📝 Modifications apportées au smali

### 1. Suppression de l'appel à `showDialog` — `MainActivity.smali`

**Avant :**
```smali
invoke-virtual {p0, v0}, Lsg/vantagepoint/uncrackable3/MainActivity;->showDialog(Ljava/lang/String;)V
```

**Après :**
```smali
return-void
```

### 2. Patch de `showDialog` pour qu'elle ne fasse rien

```smali
.method private showDialog(Ljava/lang/String;)V
    .locals 0
    return-void
.end method
```

### 3. Patch de `check_code` pour retourner toujours vrai

```smali
.method public check_code(Ljava/lang/String;)Z
    .locals 1
    const/4 v0, 0x1
    return v0
.end method
```

---

## 🧬 Patch de la librairie native avec Ghidra

### Fonction `_INIT_0` — neutralisation anti-debug / anti-Frida

| État | Instruction | Code hex |
|------|-------------|----------|
| **Avant** | `PUSH EBP` | `0x55` |
| **Après** | `RET` | `0xC3` |

La fonction retourne immédiatement, neutralisant toutes les vérifications anti-debug et anti-Frida.

---

## 🔑 Le mot de passe secret

### Clé encodée (24 octets en little-endian)

```
1d 08 11 13 0f 17 49 15 0d 00 03 19 5a 1d 13 15 08 0e 5a 00 17 08 13 14
```

### Clé XOR : `pizzapizzapizzapizzapizzapizza`

### 🐍 Script Python de déchiffrement

```python
encoded = bytes.fromhex("1d0811130f1749150d0003195a1d1315080e5a0017081314")
xor_key = b"pizzapizzapizzapizzapizzapizza"
secret  = bytes(a ^ b for a, b in zip(encoded, xor_key))
print("Mot de passe :", secret.decode())
```

### ✅ Résultat

```
Mot de passe : making owasp great again
```

---

## 📸 Captures d'écran

### Jadx

<img width="1600" height="829" alt="2" src="https://github.com/user-attachments/assets/b1342f59-3e10-45a5-a255-34bb595e5fef" />
<img width="1379" height="815" alt="2_1" src="https://github.com/user-attachments/assets/ffa7a518-31bd-4c35-beee-bd64788cbc7f" />


### Apktool et modification de MainActivity.smali


<img width="775" height="336" alt="3" src="https://github.com/user-attachments/assets/e40b3f00-6179-4e16-8a14-3f0553263376" />
<img width="890" height="246" alt="3_1" src="https://github.com/user-attachments/assets/bf8745b0-7efa-48ad-951f-57d1c2a773ab" />


### Ghidra 

<img width="1570" height="686" alt="4" src="https://github.com/user-attachments/assets/e64564ad-cb32-4609-a472-126d150a877c" />
<img width="876" height="697" alt="4_1" src="https://github.com/user-attachments/assets/b2cc4825-9b58-486f-b643-31f0b71b65ad" />
<img width="1103" height="279" alt="fct_antiDebug_patche" src="https://github.com/user-attachments/assets/0f612d02-a982-4c36-88fd-303e25385a36" />

### Signature APK et réinstaller l'apk dans l'émulateur



<img width="613" height="235" alt="5" src="https://github.com/user-attachments/assets/de439492-98ec-4480-9abf-4baf7f94a321" />
<img width="765" height="680" alt="5_1" src="https://github.com/user-attachments/assets/0eb3195b-f44e-42ed-8910-addc25336b08" />
<img width="771" height="254" alt="5_2" src="https://github.com/user-attachments/assets/287ac752-40b2-43d5-89be-d3d276445934" />

### Test avant patcher l'apk 

<img width="391" height="828" alt="TestAvantPatch" src="https://github.com/user-attachments/assets/002b0408-aeb7-45e6-8264-ae11e272b486" />


### Test après patcher l'apk

<img width="360" height="749" alt="TestApresPatch" src="https://github.com/user-attachments/assets/29a1bb62-a883-4928-86db-7ca612ab0ce9" />

---

---

## 📚 RÉCAPITULATIF – CE QUE J'AI APPRIS

---

### ✅ Synthèse du laboratoire

Ce laboratoire m'a permis de maîtriser les techniques de **reverse engineering Android** en combinant analyse statique (Jadx, apktool), analyse native (Ghidra) et cryptanalyse (XOR). J'ai contourné 6 niveaux de protection distincts sur une application conçue par l'OWASP pour résister à ces attaques.

---

### 📝 Les 9 points essentiels à retenir

| # | Point clé |
|---|-----------|
| 1 | **APK = ZIP** → on peut l'extraire pour accéder au smali et aux `.so` |
| 2 | **apktool** décompile le bytecode dex en smali modifiable |
| 3 | Le **smali** est le langage assembleur du bytecode Android |
| 4 | **Ghidra** décompile les `.so` natifs en pseudo-code C |
| 5 | Patcher `_INIT_0` avec `RET` désactive TOUTES les vérifications du `.init_array` |
| 6 | L'**anti-debug** via `ptrace()` est neutralisé dès que la fonction ne s'exécute plus |
| 7 | L'**anti-Frida** scanne `/proc/self/maps` — neutralisé de la même façon |
| 8 | Une APK patchée doit être **resignée** avant installation (`uber-apk-signer`) |
| 9 | Une clé obfusquée en **XOR** peut être déchiffrée avec un simple script Python |

---

### 📊 Comparaison des niveaux de protection

| Protection | Couche | Outil pour contourner | Difficulté |
|------------|--------|-----------------------|------------|
| Anti-root | Java (smali) | apktool | ⭐⭐ |
| Anti-tampering | Java (smali) | apktool | ⭐⭐ |
| Anti-debug (`ptrace`) | Natif (`.so`) | Ghidra | ⭐⭐⭐⭐ |
| Anti-Frida | Natif (`.so`) | Ghidra | ⭐⭐⭐⭐ |
| Obfuscation XOR | Natif (`.so`) | Python | ⭐⭐⭐ |
| Signature APK | Package | uber-apk-signer | ⭐ |

---

### 💡 Bonnes pratiques retenues

- [x] Toujours analyser le code Java avec Jadx **avant** de modifier le smali
- [x] Vérifier l'architecture (`arm64-v8a` vs `x86_64`) avant de patcher le bon `.so`
- [x] Tester l'APK après **chaque** patch pour isoler les problèmes
- [x] Toujours désinstaller l'APK originale avant d'installer la patchée
- [x] Utiliser `adb logcat` pour déboguer les crashs après installation

---

### 🎯 Compétences acquises

| Compétence | Niveau |
|------------|--------|
| Décompiler une APK avec apktool | ✅ Maîtrisé |
| Lire et modifier du smali | ✅ Maîtrisé |
| Analyser une librairie native avec Ghidra | ✅ Maîtrisé |
| Patcher une fonction native (RET) | ✅ Maîtrisé |
| Contourner l'anti-debug (`ptrace`) | ✅ Maîtrisé |
| Contourner l'anti-Frida | ✅ Maîtrisé |
| Signer une APK patchée | ✅ Maîtrisé |
| Déchiffrer une clé XOR avec Python | ✅ Maîtrisé |
| Comprendre l'obfuscation (LCG, listes chaînées) | ✅ Maîtrisé |

---

### ✅ Vérification finale

- [x] Analyse statique avec Jadx réussie
- [x] Décompilation avec apktool réussie
- [x] Patch smali (suppression root/tampering) réussi
- [x] Patch `libfoo.so` (anti-debug → RET) réussi
- [x] Patch CodeCheck (return true) réussi
- [x] Reconstruction APK réussie
- [x] Signature APK réussie
- [x] Installation sur émulateur réussie
- [x] Application accepte le mot de passe `making owasp great again` ✅

---

### 👨‍💻 Auteur

| Élément | Information |
|---------|-------------|
| **Nom** | El Hachimi Abdelhamid |
| **GitHub** | [abdotranscript25](https://github.com/abdotranscript25) |
| **Lab** | Sécurité Mobile - Lab 17 |

---

### 📅 Version

| Élément | Information |
|---------|-------------|
| **Date** | Mai 2026 |
| **Version** | 1.0 |
| **Statut** | ✅ Finalisé |
