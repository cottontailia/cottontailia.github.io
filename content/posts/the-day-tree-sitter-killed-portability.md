+++
title = "The Day Tree-sitter Killed Portability"
date = 2026-02-13
description = "Same Emacs, same config. Worked on macOS, died on Windows with ABI mismatch. How I built treesit-env to solve Tree-sitter's portability nightmare."

[extra]
keywords = ["emacs", "tree-sitter", "treesit", "ABI mismatch", "windows", "msys2", "mingw", "portability"]
author = "cottontailia"
title_suffix = "Emacs treesit-env.el"
+++

Same Emacs 29.2. Same config file.  
It worked perfectly on macOS.  
I copied it to Windows. Tree-sitter died.

```
treesit-load-language-error: Cannot load language rust
ABI version mismatch (library: 15, runtime: 14)
```

I checked.  
macOS build: `(treesit-library-abi-version)` → **15**  
Windows build: `(treesit-library-abi-version)` → **14**

That's when I realized.  
The 10-year assumption of "portable init.el" had just collapsed.

---

## Act 1: The Migration Nightmare

### The First Wall: "Grammar not found"

I had spent weeks perfecting my Emacs setup on macOS. Same version. Same packages. Same config file. I copied it to my Windows machine.

I opened a Rust file.

```
Cannot find tree-sitter grammar for rust
```

"Just not installed yet."  
I ran `M-x treesit-install-language-grammar`.  
Entered the repository URL.

```
treesit-install-language-grammar: Cannot find a C compiler
```

### The MSYS2 Swamp

I Googled. Every article gave the same answer.

**"Install MSYS2"**

- Download: **3.5GB**
- Install time: 30 minutes
- PATH configuration
- Restart Emacs

I tried again.

```
Error: cc1.exe not found
```

Another search.

"You need `mingw-w64-x86_64-gcc`"

Installed via pacman.  
Another restart.

Finally, compilation succeeded.

### The Gateway to ABI Hell

I opened a Rust file.

```
treesit-load-language-error: Cannot load language rust
ABI version mismatch (library: 15, runtime: 14)
```

"What is this?"

I checked.  
Windows build of Emacs 29.2 supports **ABI 14** at most.  
macOS build of the same Emacs 29.2 supports **ABI 15**.

**Same version number. Different capabilities.**

In that moment, I realized the "portable config file" had been a lie.

---

## Act 2: Four Circles of Hell

### Hell 1: Platform-Dependent ABI Fragmentation

Same Emacs 29.2, but:
- macOS: ABI 15 supported
- Windows: ABI 14 only
- Linux (distribution-dependent): 13, 14, 15 mixed

Grammar settings in init.el won't work on another machine.  
Because **Emacs capabilities vary by build conditions**.

### Hell 2: Crashes Even When ABI Matches

"Fine, I'll just use ABI 14 grammars."

I installed the latest rust grammar on macOS.  
ABI check: **14**  
Emacs support: **15**

"Perfect."

I opened a file.

**Emacs crashed. Segfault. No error message.**

I searched.  
Buried in a GitHub issue:

> "ABI 14 has internal breaking changes between tree-sitter 0.20.x and 0.21.x"

**Same version number. Different internals.**

Which commit actually works with my Emacs?  
Trial and error is the only way.

### Hell 3: The Windows "3.5GB Tax"

To get a few kilobytes of syntax highlighting:
- MSYS2: 3.5GB
- mingw-w64-toolchain: hundreds of MB
- PATH conflicts
- Build error debugging

**Polluting the OS just to get a .so file.**

### Hell 4: Configuration Fragmentation

I want to add Rust.

```elisp
;; 1. Define repository
(add-to-list 'treesit-language-source-alist
             '(rust "https://github.com/tree-sitter/tree-sitter-rust"))

;; 2. Install (manual execution)
;; M-x treesit-install-language-grammar

;; 3. Bind major mode
(add-to-list 'major-mode-remap-alist '(rust-mode . rust-ts-mode))

;; 4. Bind file extension
(add-to-list 'auto-mode-alist '("\\.rs\\'" . rust-ts-mode))
```

I want to add TypeScript.  
Repeat the same process.

Python, YAML, Markdown...

Settings are now scattered across 4 places in init.el.  
When I want to remove a language, I don't know where to delete.

**This is maintenance hell.**

---

## Act 3: The Obsession Begins

I spent 6 hours fighting MSYS2.  
I manually traversed Git history to find compatible commits.  
On the third night, I decided.

**"Never again."**

I'll hijack Emacs's internal functions if I have to.  
I'll brute-force scan Git repositories if I have to.

This isn't a "convenient tool."  
**This is infrastructure born from obsession.**

---

## The Solution: treesit-env.el

I built four weapons.

### Weapon 1: Brute-Force Git History Search

"I don't know which commit works."  
**Then try them all.**

```elisp
(while (and (> current-abi abi-max) (< attempt limit))
  ;; Deepen history by 10 commits at a time
  (call-process "git" nil nil nil "fetch" "--deepen" "10")
  
  (dolist (commit new-commits)
    (with-temp-buffer
      ;; Read file without checkout
      (call-process "git" nil t nil "show" 
                    (format "%s:src/parser.c" commit))
      
      ;; Parse LANGUAGE_VERSION with regex
      (goto-char (point-min))
      (when (re-search-forward "LANGUAGE_VERSION[ \t]+\\([0-9]+\\)" nil t)
        (setq current-abi (string-to-number (match-string 1))))
      
      ;; Checkout immediately when compatible
      (when (<= current-abi abi-max)
        (call-process "git" nil nil nil "checkout" commit)
        (throw 'found commit)))))
```

**How it works:**
1. Dig into history with `git fetch --deepen`
2. Read files directly with `git show COMMIT:src/parser.c` (no checkout needed)
3. Extract `LANGUAGE_VERSION` via regex
4. Stop the moment ABI is within limits

It's slow. Network-heavy.  
**But it never fails.**

### Weapon 2: Compiler Hijacking

"MSYS2 is too heavy."  
**Use Zig. A single 200MB binary.**

"But Emacs expects gcc."  
**Then lie to it.**

```elisp
(cl-letf* (((symbol-function 'executable-find)
            (lambda (command)
              ;; Looking for "gcc"? → Return Zig instead
              (let ((base (file-name-sans-extension 
                          (file-name-nondirectory command))))
                (cond ((member base '("cc" "gcc" "clang"))
                       (car treesit-env-compiler-cc))
                      ((member base '("c++" "g++"))
                       (car treesit-env-compiler-c++))
                      (t (funcall old-exec-find command))))))
           
           ((symbol-function 'call-process)
            (lambda (program &rest args)
              ;; Missing .o files during linking? → Auto-inject them
              (when (and (member "-shared" args)
                         (not (cl-some (lambda (a) 
                                        (string-match-p "\\.o$" a)) 
                                      args)))
                (setq args (append args 
                                  (directory-files "." nil "\\.o$"))))
              (apply old-call-process program args))))
```

**What's happening:**

Emacs believes "gcc exists."  
I decided to return Zig.

Emacs expects ".o files as arguments."  
I decided to scan the directory and inject them.

**This isn't a "hack." It's obsession.**

### Weapon 3: Declarative Configuration

"Settings are scattered."  
**Centralize them.**

```elisp
(treesit-env rust
  :vc "tree-sitter/tree-sitter-rust"
  :mode "\\.rs\\'"
  :revision auto)  ;; Auto-search Git history
```

That's it.

- Repository definition
- Installation
- Mode binding
- ABI compatibility search

**Everything is automatic.**

### Weapon 4: Recipe Dump

"Someone should maintain the canonical recipes."  
**No. Let the community share them.**

```elisp
M-x treesit-env-dump-recipes
```

This outputs your working configuration in recipe format.  
Paste it in a Gist. Post it in a forum.

**No centralized maintainer needed.**

---

## Usage

### Basic Setup

```elisp
(use-package treesit-env
  :vc (:url "https://github.com/cottontailia/treesit-env")
  :custom
  (treesit-env-default-revision-auto t)  ;; Auto ABI search
  (treesit-env-abi-max 14)               ;; Force ABI 14 or lower
  
  :config
  ;; Use Zig on Windows
  (when (eq system-type 'windows-nt)
    (setq treesit-env-compiler-cc '("zig" "cc" "-O3"))
    (setq treesit-env-compiler-c++ '("zig" "c++" "-O3")))
  
  ;; Activate languages
  (treesit-env rust python typescript))
```

### Pattern Collection

```elisp
;; Pattern 1: Official repository (short form)
(treesit-env rust)  ;; → tree-sitter/tree-sitter-rust

;; Pattern 2: tree-sitter-grammars org
(treesit-env markdown :vc grammars)

;; Pattern 3: Custom repository
(treesit-env python :vc "user/custom-python-grammar")

;; Pattern 4: Auto ABI search
(treesit-env go :revision auto)

;; Pattern 5: Fixed revision (fastest)
(treesit-env rust :revision "v0.20.4")

;; Pattern 6: Auto dependency resolution
(treesit-env tsx :deps typescript)

;; Pattern 7: Monorepo
(treesit-env typescript 
  :vc "tree-sitter/tree-sitter-typescript"
  :src-path "typescript/src")

;; Pattern 8: Multiple extensions
(treesit-env javascript 
  :mode "\\.js\\'" "\\.mjs\\'" "\\.cjs\\'")
```

---

### Recipe Definition and Sharing

Complete examples: see `treesit-env-recipe-placeholder.el` in the repository.  
**Warning**: It's a "placeholder"—minimal recipes, no maintenance promise.

```elisp
;; Define recipes with treesit-env-recipes macro
(defconst my-recipes
  (treesit-env-recipes
   (rust :vc "tree-sitter/tree-sitter-rust" :revision "v0.20.4")
   (python :vc grammars :mode "\\.py\\'")
   (typescript :deps tsx :src-path "typescript/src")))

;; Import with treesit-env-source
(treesit-env-source my-recipes)

;; M-x treesit-env-dump-recipes outputs in this format too
```

Copy someone's success. Share yours.  
**Decentralized ecosystem.**

---

## Under the Hood: Technical Deep Dive

### Why Read parser.c Directly?

Tree-sitter maintainers don't care about Emacs.  
They race forward toward the latest library (ABI 15).

Version info in GitHub READMEs is unreliable.  
Tags can't be trusted.

**The only truth is the source code.**

At the top of `src/parser.c`:

```c
#define LANGUAGE_VERSION 14
```

Read this.  
Only this tells you "Is this commit really ABI 14?"

### The Madness of Zig Hijacking

Emacs's build system is stubborn.

It expects specific compiler names (`gcc`, `g++`).  
It expects specific argument orders.  
It expects specific file naming conventions.

Zig doesn't satisfy these.

So when Emacs "calls gcc," I **run Zig instead** behind the scenes.

```elisp
;; Emacs's perspective
(call-process "gcc" nil t nil "-shared" "-o" "rust.so" "parser.c")

;; Actual execution
(call-process "zig" nil t nil "cc" "-O3" "-shared" "-o" "rust.so" "parser.c" "parser.o" "scanner.o")
```

Those `.o` files at the end? I injected them.  
Because Zig's linker requires explicit file lists.

**Emacs knows nothing. It just sees "build succeeded."**

### Recursive Dependency Resolution

tsx depends on typescript.  
C depends on cpp (sometimes).

Users don't know this. They don't need to.

```elisp
(defvar treesit-env--installing-stack nil)

(defun treesit-env--execute-install (recipe)
  (let ((lang (plist-get recipe :lang))
        (deps (plist-get recipe :deps)))
    
    ;; Detect circular dependencies
    (when (memq lang treesit-env--installing-stack)
      (error "Circular dependency detected: %s" lang))
    
    ;; Install dependencies first
    (push lang treesit-env--installing-stack)
    (dolist (dep deps)
      (unless (treesit-language-available-p dep)
        (treesit-env--execute-install (get-recipe dep))))
    (pop treesit-env--installing-stack)
    
    ;; Install the language itself
    (treesit-env--compile recipe)))
```

Simple. Reliable.

**Users never think about dependencies.**

---

## Philosophy: Why CC0?

This tool was born from personal rage.

- 6 hours fighting MSYS2
- Manual Git history traversal
- Config files that won't work across 3 machines

This isn't "innovative algorithm."  
It's **plumbing work** to make unstable Tree-sitter infrastructure usable.

That's why I chose CC0 (Public Domain).

I don't seek stars or contributors.  
I just wanted an environment that works **the same way on any machine**.

### Why I Won't Maintain Recipes

I don't want to be a "URL maintainer."  
I want to write code.

That's why I created `treesit-env-dump-recipes`.

You can export your working config.  
Paste it in a Gist. Post it on Reddit. Copy it to someone's blog.

**Let the community self-organize.**

Centralized databases are fragile.  
Distributed knowledge is resilient.

---

## Conclusion

If you've ever:

- Been lost after seeing "ABI mismatch"
- Regretted installing MSYS2
- Despaired when your config broke on another machine
- Manually traversed Git history to find working commits

**This tool is for you.**

This isn't a perfect solution.  
Someday, when Emacs and Tree-sitter mature, tools like this won't be needed.

But today, you're suffering.

So I'm releasing this infrastructure born from obsession.

---

**GitHub**: <https://github.com/cottontailia/treesit-env>  
**License**: CC0 (Public Domain)
