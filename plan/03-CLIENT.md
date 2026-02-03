# Client Implementation Plan - Updater CLI (Ultra-Detailed for Freshers)

## 📖 Table of Contents
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Step-by-Step Implementation](#implementation)
4. [Testing](#testing)
5. [Integration](#integration)
6. [Troubleshooting](#troubleshooting)

---

## 🎯 Overview

### What are we building?
A custom version of Shorebird's updater tool that connects to YOUR backend instead of Shorebird's cloud.

### Why this approach?
- ✅ Official tool from Shorebird (battle-tested)
- ✅ Only 1 line of code to change
- ✅ No need to understand Rust
- ✅ Works exactly like original

### What will you have after this?
A binary file called `shorebird_updater` that:
- Checks YOUR backend for updates
- Downloads patches from YOUR R2 storage
- Installs patches in Flutter apps
- Works on macOS, Linux, Windows

### Time required: 10-15 minutes

---

## 📋 Prerequisites

### ✅ Checklist Before Starting

**1. Backend must be deployed**
- [ ] Cloudflare Workers deployed
- [ ] Have production URL (e.g., `https://shorebird-backend.abc123.workers.dev`)
- [ ] Backend tested with curl

**Why?** You need the URL to put in config.rs

**2. Tools installed**
- [ ] Git installed (`git --version`)
- [ ] Rust installed (`rustc --version`) - We'll install if needed
- [ ] Terminal/Command line open

**Why?** Git to clone, Rust to build

**3. Know your backend URL**
- Write it down: `_________________________________`
- Example: `https://shorebird-backend.abc123.workers.dev`

**Why?** You'll need to type this exactly

---

## 🚀 STEP 1: Clone Repository

### 📍 Purpose
Download the official updater source code from Shorebird's GitHub.

### 🎯 What you'll do
Copy the code to your computer so you can modify it.

### 📝 Exact Actions

**Action 1.1: Open Terminal**
- **macOS:** Press `Cmd + Space`, type "Terminal", press Enter
- **Windows:** Press `Win + R`, type "cmd", press Enter
- **Linux:** Press `Ctrl + Alt + T`

**Expected:** A black/white window with text cursor

**Common mistake:** Opening wrong app (like Text Editor)
**How to avoid:** Look for window with `$` or `>` symbol

---

**Action 1.2: Navigate to project folder**

```bash
cd /Users/neun/Desktop/codepush
```

**What this does:** Changes directory to your project folder

**Expected output:**
```
/Users/neun/Desktop/codepush $
```

**Common mistakes:**
- ❌ Typing wrong path
- ❌ Folder doesn't exist

**How to fix:**
```bash
# Check current location
pwd

# Create folder if needed
mkdir -p /Users/neun/Desktop/codepush
cd /Users/neun/Desktop/codepush
```

---

**Action 1.3: Clone repository**

```bash
git clone https://github.com/shorebirdtech/updater.git
```

**What this does:** Downloads all code from GitHub to your computer

**Expected output:**
```
Cloning into 'updater'...
remote: Enumerating objects: 1234, done.
remote: Counting objects: 100% (1234/1234), done.
remote: Compressing objects: 100% (567/567), done.
remote: Total 1234 (delta 890), reused 1234 (delta 890)
Receiving objects: 100% (1234/1234), 2.34 MiB | 5.67 MiB/s, done.
Resolving deltas: 100% (890/890), done.
```

**Time:** 10-30 seconds (depends on internet speed)

**Common mistakes:**
- ❌ "git: command not found" → Install Git first
- ❌ "Permission denied" → Check internet connection
- ❌ Folder already exists → Delete old folder first: `rm -rf updater`

**How to verify:**
```bash
ls -la
# Should see "updater" folder
```

---

**Action 1.4: Enter updater folder**

```bash
cd updater
```

**What this does:** Moves into the downloaded folder

**Expected output:**
```
/Users/neun/Desktop/codepush/updater $
```

**How to verify:**
```bash
ls
# Should see: Cargo.toml, library/, README.md, etc.
```

### ✅ Checkpoint 1
- [ ] Terminal open
- [ ] In `/Users/neun/Desktop/codepush/updater` folder
- [ ] Can see files with `ls` command

**If stuck:** Re-read Action 1.1 to 1.4

---

## 🚀 STEP 2: Modify Configuration

### 📍 Purpose
Change the base URL from Shorebird's API to YOUR backend API.

### 🎯 What you'll do
Edit ONE line in ONE file to point to your Cloudflare Workers.

### 📝 Exact Actions

**Action 2.1: Open config file**

**Option A: Using VS Code (Recommended)**
```bash
code library/src/config.rs
```

**Option B: Using nano (Terminal editor)**
```bash
nano library/src/config.rs
```

**Option C: Using any text editor**
- Open Finder/Explorer
- Navigate to `updater/library/src/`
- Double-click `config.rs`

**Expected:** File opens showing Rust code

**Common mistakes:**
- ❌ Opening wrong file
- ❌ File not found

**How to verify:** File should contain text like:
```rust
pub const BASE_URL: &str = "https://api.shorebird.dev";
```

---

**Action 2.2: Find the BASE_URL line**

**In VS Code:**
- Press `Cmd + F` (Mac) or `Ctrl + F` (Windows/Linux)
- Type: `BASE_URL`
- Press Enter

**In nano:**
- Press `Ctrl + W`
- Type: `BASE_URL`
- Press Enter

**Expected:** Cursor jumps to this line:
```rust
pub const BASE_URL: &str = "https://api.shorebird.dev";
```

**Line number:** Usually around line 10-20

**Common mistakes:**
- ❌ Can't find the line → Make sure you opened correct file
- ❌ Multiple BASE_URL → Use the one with `pub const`

---

**Action 2.3: Change the URL**

**BEFORE:**
```rust
pub const BASE_URL: &str = "https://api.shorebird.dev";
```

**AFTER:**
```rust
pub const BASE_URL: &str = "https://shorebird-backend.YOUR-SUBDOMAIN.workers.dev";
```

**⚠️ IMPORTANT:**
- Replace `YOUR-SUBDOMAIN` with your actual subdomain
- Keep the quotes `"`
- Keep the semicolon `;`
- Don't add trailing slash `/`

**Example (correct):**
```rust
pub const BASE_URL: &str = "https://shorebird-backend.abc123.workers.dev";
```

**Examples (wrong):**
```rust
// ❌ Missing quotes
pub const BASE_URL: &str = https://shorebird-backend.abc123.workers.dev;

// ❌ Trailing slash
pub const BASE_URL: &str = "https://shorebird-backend.abc123.workers.dev/";

// ❌ Missing semicolon
pub const BASE_URL: &str = "https://shorebird-backend.abc123.workers.dev"

// ❌ Wrong URL
pub const BASE_URL: &str = "https://api.shorebird.dev";
```

**How to verify:** Read the line 3 times to make sure it's correct

---

**Action 2.4: Save the file**

**In VS Code:**
- Press `Cmd + S` (Mac) or `Ctrl + S` (Windows/Linux)
- Close the file

**In nano:**
- Press `Ctrl + O` (save)
- Press Enter (confirm)
- Press `Ctrl + X` (exit)

**Expected:** File saved, no error messages

**How to verify:**
```bash
# View the file to confirm change
cat library/src/config.rs | grep BASE_URL
```

**Expected output:**
```rust
pub const BASE_URL: &str = "https://shorebird-backend.abc123.workers.dev";
```

### ✅ Checkpoint 2
- [ ] File `config.rs` opened
- [ ] Found `BASE_URL` line
- [ ] Changed to YOUR backend URL
- [ ] Saved file
- [ ] Verified with `cat` command

**If stuck:** Re-read Action 2.1 to 2.4

---

## 🚀 STEP 3: Install Rust (If Needed)

### 📍 Purpose
Install Rust programming language to build the updater tool.

### 🎯 What you'll do
Check if Rust is installed. If not, install it.

### 📝 Exact Actions

**Action 3.1: Check if Rust is installed**

```bash
rustc --version
```

**Expected output (if installed):**
```
rustc 1.75.0 (82e1608df 2023-12-21)
```

**Expected output (if NOT installed):**
```
rustc: command not found
```

**If installed:** Skip to Step 4
**If NOT installed:** Continue to Action 3.2

---

**Action 3.2: Install Rust**

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

**What this does:** Downloads and runs Rust installer

**Expected output:**
```
info: downloading installer

Welcome to Rust!

This will download and install the official compiler for the Rust
programming language, and its package manager, Cargo.

...

1) Proceed with installation (default)
2) Customize installation
3) Cancel installation
>
```

**Action:** Type `1` and press Enter

**Time:** 2-5 minutes (depends on internet speed)

**Expected output (at end):**
```
Rust is installed now. Great!

To get started you may need to restart your current shell.
This would reload your PATH environment variable to include
Cargo's bin directory ($HOME/.cargo/bin).

To configure your current shell, run:
source "$HOME/.cargo/env"
```

**Common mistakes:**
- ❌ "curl: command not found" → Install curl first
- ❌ Installation hangs → Check internet connection
- ❌ Permission denied → Don't use `sudo`

---

**Action 3.3: Activate Rust**

```bash
source $HOME/.cargo/env
```

**What this does:** Makes Rust available in current terminal

**Expected output:** (none - silent success)

**How to verify:**
```bash
rustc --version
```

**Expected:**
```
rustc 1.75.0 (82e1608df 2023-12-21)
```

### ✅ Checkpoint 3
- [ ] Rust installed
- [ ] `rustc --version` shows version number
- [ ] `cargo --version` shows version number

**If stuck:** Close terminal, open new one, try `rustc --version` again

---

## 🚀 STEP 4: Build the Updater

### 📍 Purpose
Compile the Rust source code into an executable binary file.

### 🎯 What you'll do
Run one command that builds the updater tool.

### 📝 Exact Actions

**Action 4.1: Make sure you're in updater folder**

```bash
pwd
```

**Expected output:**
```
/Users/neun/Desktop/codepush/updater
```

**If wrong location:**
```bash
cd /Users/neun/Desktop/codepush/updater
```

---

**Action 4.2: Build in release mode**

```bash
cargo build --release
```

**What this does:** 
- Compiles Rust code
- Creates optimized binary
- Puts it in `target/release/` folder

**Expected output:**
```
   Compiling proc-macro2 v1.0.70
   Compiling unicode-ident v1.0.12
   Compiling libc v0.2.151
   ...
   Compiling updater v0.1.0 (/Users/neun/Desktop/codepush/updater)
    Finished release [optimized] target(s) in 2m 34s
```

**Time:** 2-5 minutes (first time is slower)

**Progress indicators:**
- `Compiling` = Building dependencies
- `Finished` = Done!

**Common mistakes:**
- ❌ "error: could not compile" → Check config.rs syntax
- ❌ Build hangs → Wait, it's normal for first build
- ❌ "cargo: command not found" → Rust not installed properly

**If error in config.rs:**
```bash
# View the error
# Usually shows line number

# Fix the file
nano library/src/config.rs

# Build again
cargo build --release
```

---

**Action 4.3: Verify binary was created**

```bash
ls -lh target/release/updater
```

**Expected output:**
```
-rwxr-xr-x  1 neun  staff   3.2M Jan  1 12:00 target/release/updater
```

**What to check:**
- File exists ✅
- Size is ~3-5 MB ✅
- Has execute permission (`x`) ✅

**Common mistakes:**
- ❌ File not found → Build failed, check errors
- ❌ File is 0 bytes → Build incomplete

---

**Action 4.4: Test the binary**

```bash
./target/release/updater --version
```

**Expected output:**
```
updater 0.1.0
```

**Common mistakes:**
- ❌ "Permission denied" → Run: `chmod +x target/release/updater`
- ❌ "No such file" → Build didn't complete

### ✅ Checkpoint 4
- [ ] Build completed without errors
- [ ] Binary file exists at `target/release/updater`
- [ ] Binary runs and shows version

**If stuck:** Delete `target/` folder and rebuild:
```bash
rm -rf target
cargo build --release
```

---

## 🚀 STEP 5: Rename and Install

### 📍 Purpose
Give the binary a custom name and make it accessible from anywhere.

### 🎯 What you'll do
Copy the binary with a new name and install it globally.

### 📝 Exact Actions

**Action 5.1: Create renamed copy**

```bash
cp target/release/updater target/release/shorebird_updater
```

**What this does:** Creates a copy with new name

**Why rename?** 
- Distinguish from original Shorebird updater
- Avoid conflicts
- Clear it's YOUR custom version

**Expected output:** (none - silent success)

**How to verify:**
```bash
ls -lh target/release/ | grep updater
```

**Expected:**
```
-rwxr-xr-x  1 neun  staff   3.2M Jan  1 12:00 updater
-rwxr-xr-x  1 neun  staff   3.2M Jan  1 12:00 shorebird_updater
```

**Common mistakes:**
- ❌ Typo in name → Check spelling
- ❌ File not created → Check source file exists

---

**Action 5.2: Test renamed binary**

```bash
./target/release/shorebird_updater --version
```

**Expected output:**
```
updater 0.1.0
```

**Note:** Version still shows "updater" - that's OK!

---

**Action 5.3: Install globally (Optional but recommended)**

```bash
sudo cp target/release/shorebird_updater /usr/local/bin/
sudo chmod +x /usr/local/bin/shorebird_updater
```

**What this does:**
- Copies binary to system folder
- Makes it executable
- Allows running from anywhere

**Expected output:**
```
Password: [type your password]
```

**Note:** Password won't show while typing - that's normal!

**Common mistakes:**
- ❌ "Permission denied" → Need `sudo`
- ❌ Wrong password → Try again
- ❌ `/usr/local/bin` doesn't exist → Create it: `sudo mkdir -p /usr/local/bin`

---

**Action 5.4: Verify global installation**

```bash
# Test from any location
cd ~
shorebird_updater --version
```

**Expected output:**
```
updater 0.1.0
```

**If this works:** ✅ Installed globally!
**If "command not found":** Binary not in PATH, use full path instead

### ✅ Checkpoint 5
- [ ] Binary renamed to `shorebird_updater`
- [ ] Renamed binary works
- [ ] (Optional) Installed globally
- [ ] Can run `shorebird_updater --version` from anywhere

---

## 🧪 STEP 6: Test with Your Backend

### 📍 Purpose
Verify the updater connects to YOUR backend and can check for updates.

### 🎯 What you'll do
Run a test command to check if backend communication works.

### 📝 Exact Actions

**Action 6.1: Prepare test parameters**

You need:
- **app_id:** Your app's bundle ID (e.g., `com.yourapp.name`)
- **version:** App version (e.g., `1.0.0`)
- **platform:** `android` or `ios`
- **patch:** Current patch number (use `0` for testing)

**Write them down:**
- app_id: `_________________________________`
- version: `_________________________________`
- platform: `_________________________________`

---

**Action 6.2: Run check command**

```bash
shorebird_updater check \
  --app-id com.yourapp.name \
  --version 1.0.0 \
  --platform android \
  --patch 0
```

**Replace:**
- `com.yourapp.name` → Your actual app ID
- `1.0.0` → Your actual version
- `android` → Your platform

**What this does:** Asks YOUR backend if updates are available

**Expected output (no patches yet):**
```
No update available
```

**Expected output (patch exists):**
```
Update available: patch 1
Download URL: https://pub-xxxxx.r2.dev/com.yourapp.name/android/1.bin
SHA256: abc123def456...
Size: 1234567 bytes
```

**Common mistakes:**
- ❌ "Connection refused" → Backend not running
- ❌ "404 Not Found" → Wrong base_url in config.rs
- ❌ "Invalid app_id" → Check app_id matches backend

---

**Action 6.3: Verify backend connection**

```bash
# Test backend directly with curl
curl "https://YOUR-BACKEND-URL/api/check?app_id=com.yourapp.name&version=1.0.0&platform=android&patch=0"
```

**Expected output:**
```json
{"has_update":false}
```

**Or if patch exists:**
```json
{
  "has_update":true,
  "patch":{
    "patch_number":1,
    "download_url":"https://...",
    "sha256":"...",
    "size_bytes":1234567
  }
}
```

**If this works but updater doesn't:**
- Problem is in updater configuration
- Double-check base_url in config.rs
- Rebuild: `cargo build --release`

### ✅ Checkpoint 6
- [ ] Check command runs without errors
- [ ] Backend responds (even if no update)
- [ ] curl test confirms backend works
- [ ] Ready for real usage

---

## 📱 STEP 7: Integration with Flutter (Overview)

### 📍 Purpose
Understand how to use the updater in your Flutter app.

### 🎯 Options

**Option 1: Call from Dart (Simplest)**

```dart
import 'dart:io';

Future<bool> checkForUpdate() async {
  final result = await Process.run(
    'shorebird_updater',
    [
      'check',
      '--app-id', 'com.yourapp.name',
      '--version', '1.0.0',
      '--platform', Platform.isAndroid ? 'android' : 'ios',
      '--patch', '0',
    ],
  );
  
  return result.exitCode == 0 && result.stdout.contains('Update available');
}
```

**Option 2: Bundle with app**
- Copy binary to app assets
- Extract on first run
- Call from Dart

**Option 3: Create Flutter package**
- Wrap updater in Dart package
- Publish to pub.dev
- Reuse across projects

**Detailed integration guide:** See separate document

---

## 🚨 Troubleshooting Guide

### Problem: "git: command not found"

**Cause:** Git not installed

**Solution:**
```bash
# macOS
xcode-select --install

# Ubuntu/Debian
sudo apt-get install git

# Windows
# Download from https://git-scm.com/
```

---

### Problem: "cargo: command not found"

**Cause:** Rust not installed or not in PATH

**Solution:**
```bash
# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Activate in current terminal
source $HOME/.cargo/env

# Verify
cargo --version
```

---

### Problem: Build fails with "error: could not compile"

**Cause:** Syntax error in config.rs

**Solution:**
1. Read error message carefully
2. Note line number
3. Open config.rs
4. Check that line
5. Common issues:
   - Missing quote `"`
   - Missing semicolon `;`
   - Typo in URL

**Example error:**
```
error: expected `;`, found `pub`
 --> library/src/config.rs:10:60
```

**Fix:** Add semicolon at end of line 10

---

### Problem: "No update available" but patch exists

**Cause:** Updater not connecting to YOUR backend

**Solution:**
1. Check base_url in config.rs
2. Rebuild: `cargo build --release`
3. Test backend with curl
4. Check app_id matches
5. Check version matches

---

### Problem: Binary won't run ("Permission denied")

**Cause:** File not executable

**Solution:**
```bash
chmod +x target/release/shorebird_updater
./target/release/shorebird_updater --version
```

---

## ✅ Final Checklist

### Build Phase
- [ ] Repository cloned
- [ ] config.rs modified with YOUR base_url
- [ ] Rust installed
- [ ] Build completed successfully
- [ ] Binary created at `target/release/updater`
- [ ] Binary renamed to `shorebird_updater`
- [ ] Binary runs and shows version

### Testing Phase
- [ ] Check command works
- [ ] Connects to YOUR backend
- [ ] Backend responds correctly
- [ ] curl test confirms backend works

### Installation Phase
- [ ] Binary installed globally (optional)
- [ ] Can run from any directory
- [ ] Ready to integrate with Flutter

---

## 📊 Summary

### What You Built
A custom updater tool that:
- ✅ Checks YOUR Cloudflare Workers backend
- ✅ Downloads patches from YOUR R2 storage
- ✅ Works exactly like official Shorebird updater
- ✅ Only required changing 1 line of code

### Time Spent
- Clone: 1 min
- Modify: 2 min
- Install Rust: 5 min (if needed)
- Build: 3 min
- Test: 2 min
- **Total: ~10-15 minutes**

### Next Steps
1. Integrate with Flutter app
2. Test full update flow
3. Deploy to production
4. Monitor updates

---

## 🎓 Learning Points for Freshers

### What You Learned
1. **Git:** How to clone repositories
2. **Rust:** Basic understanding of build process
3. **Configuration:** How to modify source code
4. **Binary compilation:** How code becomes executable
5. **System integration:** Installing tools globally
6. **Testing:** Verifying functionality

### Skills Gained
- ✅ Command line proficiency
- ✅ Source code modification
- ✅ Build tool usage (cargo)
- ✅ Debugging and troubleshooting
- ✅ API testing with curl

---

## 🎉 Congratulations!

You've successfully:
- ✅ Built a custom updater tool
- ✅ Connected it to YOUR backend
- ✅ Tested it works
- ✅ Ready for production use

**You're now ready to integrate this into your Flutter app!**

---

## 📚 Additional Resources

- **Rust Book:** https://doc.rust-lang.org/book/
- **Cargo Guide:** https://doc.rust-lang.org/cargo/
- **Shorebird Docs:** https://docs.shorebird.dev/
- **Flutter Integration:** See `04-FLUTTER-INTEGRATION.md`

**Need help?** Review this document step-by-step. Every action is explained in detail!
