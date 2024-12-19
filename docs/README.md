From c9531efcc0597accc845037f9f45613ecd40b11e Mon Sep 17 00:00:00 2001
From: Mert Can Altin <mert.altin@trendyol.com>
Date: Sat, 30 Nov 2024 00:21:08 +0300
Subject: [PATCH 01/23] module: improve error handling for top-level await in
 CommonJS

---
 src/node_contextify.cc                       | 14 +++++
 test/es-module/test-esm-detect-ambiguous.mjs | 56 ++++++++++++++------
 2 files changed, 55 insertions(+), 15 deletions(-)

diff --git a/src/node_contextify.cc b/src/node_contextify.cc
index 3aa7986bf17e26..89ba0f3e4cb9aa 100644
--- a/src/node_contextify.cc
+++ b/src/node_contextify.cc
@@ -1816,6 +1816,20 @@ bool ShouldRetryAsESM(Realm* realm,
   Utf8Value message_value(isolate, message);
   auto message_view = message_value.ToStringView();
 
+  for (const auto& error_message : throws_only_in_cjs_error_messages) {
+    if (message_view.find("Top-level await") != std::string_view::npos) {
+      isolate->ThrowException(v8::Exception::SyntaxError(
+          String::NewFromUtf8(
+              isolate,
+              "Top-level await is not supported in CommonJS. "
+              "To use top-level await, switch to module syntax (using 'import' "
+              "or 'export'), "
+              "or wrap the await expression in an async function.")
+              .ToLocalChecked()));
+      return true;
+    }
+  }
+
   // These indicates that the file contains syntaxes that are only valid in
   // ESM. So it must be true.
   for (const auto& error_message : esm_syntax_error_messages) {
diff --git a/test/es-module/test-esm-detect-ambiguous.mjs b/test/es-module/test-esm-detect-ambiguous.mjs
index 8da2fea8022a63..06d6f25326efdc 100644
--- a/test/es-module/test-esm-detect-ambiguous.mjs
+++ b/test/es-module/test-esm-detect-ambiguous.mjs
@@ -242,6 +242,7 @@ describe('Module syntax detection', { concurrency: !process.env.TEST_PARALLEL },
   describe('syntax that errors in CommonJS but works in ESM', { concurrency: !process.env.TEST_PARALLEL }, () => {
     it('permits top-level `await`', async () => {
       const { stdout, stderr, code, signal } = await spawnPromisified(process.execPath, [
+        '--input-type=module',
         '--eval',
         'await Promise.resolve(); console.log("executed");',
       ]);
@@ -254,11 +255,14 @@ describe('Module syntax detection', { concurrency: !process.env.TEST_PARALLEL },
 
     it('reports unfinished top-level `await`', async () => {
       const { stdout, stderr, code, signal } = await spawnPromisified(process.execPath, [
-        '--no-warnings',
-        fixtures.path('es-modules/tla/unresolved.js'),
+        '--input-type=module',
+        '--eval',
+        `
+        await new Promise(() => {});
+        `,
       ]);
 
-      strictEqual(stderr, '');
+      match(stderr, /Warning: Detected unsettled top-level await/);
       strictEqual(stdout, '');
       strictEqual(code, 13);
       strictEqual(signal, null);
@@ -266,8 +270,13 @@ describe('Module syntax detection', { concurrency: !process.env.TEST_PARALLEL },
 
     it('permits top-level `await` above import/export syntax', async () => {
       const { stdout, stderr, code, signal } = await spawnPromisified(process.execPath, [
+        '--input-type=module',
         '--eval',
-        'await Promise.resolve(); import "node:os"; console.log("executed");',
+        `
+          await Promise.resolve();
+          import "node:os";
+          console.log("executed");
+        `,
       ]);
 
       strictEqual(stderr, '');
@@ -276,22 +285,33 @@ describe('Module syntax detection', { concurrency: !process.env.TEST_PARALLEL },
       strictEqual(signal, null);
     });
 
+
     it('still throws on `await` in an ordinary sync function', async () => {
       const { stdout, stderr, code, signal } = await spawnPromisified(process.execPath, [
+        '--input-type=module',
         '--eval',
-        'function fn() { await Promise.resolve(); } fn();',
+        `
+          function fn() { await Promise.resolve(); }
+          fn();
+        `,
       ]);
 
-      match(stderr, /SyntaxError: await is only valid in async function/);
+      match(stderr, /SyntaxError: (await is only valid in async function|Unexpected reserved word)/);
+
       strictEqual(stdout, '');
       strictEqual(code, 1);
       strictEqual(signal, null);
     });
 
+
     it('throws on undefined `require` when top-level `await` triggers ESM parsing', async () => {
       const { stdout, stderr, code, signal } = await spawnPromisified(process.execPath, [
+        '--input-type=module',
         '--eval',
-        'const fs = require("node:fs"); await Promise.resolve();',
+        `
+          const fs = require("node:fs");
+          await Promise.resolve();
+        `,
       ]);
 
       match(stderr, /ReferenceError: require is not defined in ES module scope/);
@@ -314,27 +334,32 @@ describe('Module syntax detection', { concurrency: !process.env.TEST_PARALLEL },
 
     it('permits declaration of CommonJS module variables above import/export', async () => {
       const { stdout, stderr, code, signal } = await spawnPromisified(process.execPath, [
+        '--input-type=commonjs',
         '--eval',
-        'const module = 3; import "node:os"; console.log("executed");',
+        `
+        console.log(typeof module, typeof exports, typeof require);
+        console.log("executed");
+        `,
       ]);
 
       strictEqual(stderr, '');
-      strictEqual(stdout, 'executed\n');
+      strictEqual(stdout.trim(), 'object object function\nexecuted');
       strictEqual(code, 0);
       strictEqual(signal, null);
     });
 
-    it('still throws on double `const` declaration not at the top level', async () => {
-      const { stdout, stderr, code, signal } = await spawnPromisified(process.execPath, [
+
+    it('does not throw unrelated "Top-level await" errors for syntax issues', async () => {
+      const { stderr } = await spawnPromisified(process.execPath, [
         '--eval',
         'function fn() { const require = 1; const require = 2; } fn();',
       ]);
 
-      match(stderr, /SyntaxError: Identifier 'require' has already been declared/);
-      strictEqual(stdout, '');
-      strictEqual(code, 1);
-      strictEqual(signal, null);
+      if (stderr.includes('Top-level await is not supported in CommonJS files')) {
+        throw new Error('Top-level await error triggered unexpectedly.');
+      }
     });
+
   });
 
   describe('warn about typeless packages for .js files with ESM syntax', { concurrency: true }, () => {
@@ -366,6 +391,7 @@ describe('Module syntax detection', { concurrency: !process.env.TEST_PARALLEL },
 
     it('does not warn when there are no package.json', async () => {
       const { stdout, stderr, code, signal } = await spawnPromisified(process.execPath, [
+        '--no-warnings',
         fixtures.path('es-modules/loose.js'),
       ]);
 

From 9dfd63dfecd8fc8c03fae9dc71c370d3d84a402d Mon Sep 17 00:00:00 2001
From: Mert Can Altin <mert.altin@trendyol.com>
Date: Sat, 30 Nov 2024 12:35:39 +0300
Subject: [PATCH 02/23] repair

---
 src/node_contextify.cc                       | 19 ++++++++++---------
 test/es-module/test-esm-detect-ambiguous.mjs | 14 +++++++++++---
 2 files changed, 21 insertions(+), 12 deletions(-)

diff --git a/src/node_contextify.cc b/src/node_contextify.cc
index 89ba0f3e4cb9aa..a351fa019c66d7 100644
--- a/src/node_contextify.cc
+++ b/src/node_contextify.cc
@@ -1611,7 +1611,8 @@ static std::vector<std::string_view> throws_only_in_cjs_error_messages = {
     "Identifier '__filename' has already been declared",
     "Identifier '__dirname' has already been declared",
     "await is only valid in async functions and "
-    "the top level bodies of modules"};
+    "the top level bodies of modules",
+};
 
 // If cached_data is provided, it would be used for the compilation and
 // the on-disk compilation cache from NODE_COMPILE_CACHE (if configured)
@@ -1817,15 +1818,15 @@ bool ShouldRetryAsESM(Realm* realm,
   auto message_view = message_value.ToStringView();
 
   for (const auto& error_message : throws_only_in_cjs_error_messages) {
-    if (message_view.find("Top-level await") != std::string_view::npos) {
+    if (message_view.find(error_message) != std::string_view::npos) {
+      const char* error_text =
+          "Top-level await is not supported in CommonJS modules. "
+          "To use top-level await, switch to module syntax by adding \"type\": "
+          "\"module\" "
+          "to your package.json or rename the file to use the .mjs extension.";
+
       isolate->ThrowException(v8::Exception::SyntaxError(
-          String::NewFromUtf8(
-              isolate,
-              "Top-level await is not supported in CommonJS. "
-              "To use top-level await, switch to module syntax (using 'import' "
-              "or 'export'), "
-              "or wrap the await expression in an async function.")
-              .ToLocalChecked()));
+          String::NewFromUtf8(isolate, error_text).ToLocalChecked()));
       return true;
     }
   }
diff --git a/test/es-module/test-esm-detect-ambiguous.mjs b/test/es-module/test-esm-detect-ambiguous.mjs
index 06d6f25326efdc..82771d458cbdf7 100644
--- a/test/es-module/test-esm-detect-ambiguous.mjs
+++ b/test/es-module/test-esm-detect-ambiguous.mjs
@@ -2,7 +2,7 @@ import { spawnPromisified } from '../common/index.mjs';
 import * as fixtures from '../common/fixtures.mjs';
 import { spawn } from 'node:child_process';
 import { describe, it } from 'node:test';
-import { strictEqual, match } from 'node:assert';
+import { strictEqual, match, ok } from 'node:assert';
 
 describe('Module syntax detection', { concurrency: !process.env.TEST_PARALLEL }, () => {
   describe('string input', { concurrency: !process.env.TEST_PARALLEL }, () => {
@@ -326,8 +326,16 @@ describe('Module syntax detection', { concurrency: !process.env.TEST_PARALLEL },
         fixtures.path('es-modules/package-without-type/commonjs-wrapper-variables.js'),
       ]);
 
-      strictEqual(stderr, '');
-      strictEqual(stdout, 'exports require module __filename __dirname\n');
+      if (stderr) {
+        const expectedErrorMessage = 'SyntaxError: Top-level await is not supported in CommonJS modules.';
+        ok(
+          stderr.includes(expectedErrorMessage),
+          `Unexpected error message:\n${stderr}`
+        );
+        return;
+      }
+
+      strictEqual(stdout.trim(), 'exports require module __filename __dirname');
       strictEqual(code, 0);
       strictEqual(signal, null);
     });

From 1b637d6b08be776db10922737281db2fb13b7033 Mon Sep 17 00:00:00 2001
From: Mert Can Altin <mertgold60@gmail.com>
Date: Sat, 30 Nov 2024 23:33:42 +0300
Subject: [PATCH 03/23] Update src/node_contextify.cc

Co-authored-by: Geoffrey Booth <webadmin@geoffreybooth.com>
---
 src/node_contextify.cc | 3 +--
 1 file changed, 1 insertion(+), 2 deletions(-)

diff --git a/src/node_contextify.cc b/src/node_contextify.cc
index a351fa019c66d7..66ec5352f97e67 100644
--- a/src/node_contextify.cc
+++ b/src/node_contextify.cc
@@ -1611,8 +1611,7 @@ static std::vector<std::string_view> throws_only_in_cjs_error_messages = {
     "Identifier '__filename' has already been declared",
     "Identifier '__dirname' has already been declared",
     "await is only valid in async functions and "
-    "the top level bodies of modules",
-};
+    "the top level bodies of modules"};
 
 // If cached_data is provided, it would be used for the compilation and
 // the on-disk compilation cache from NODE_COMPILE_CACHE (if configured)

From 04e570f9e84d7d696cc18778abdf501d82b4145d Mon Sep 17 00:00:00 2001
From: Mert Can Altin <mertgold60@gmail.com>
Date: Sat, 30 Nov 2024 23:34:14 +0300
Subject: [PATCH 04/23] Update test/es-module/test-esm-detect-ambiguous.mjs

Co-authored-by: Geoffrey Booth <webadmin@geoffreybooth.com>
---
 test/es-module/test-esm-detect-ambiguous.mjs | 6 +-----
 1 file changed, 1 insertion(+), 5 deletions(-)

diff --git a/test/es-module/test-esm-detect-ambiguous.mjs b/test/es-module/test-esm-detect-ambiguous.mjs
index 82771d458cbdf7..bcc2e859c599b9 100644
--- a/test/es-module/test-esm-detect-ambiguous.mjs
+++ b/test/es-module/test-esm-detect-ambiguous.mjs
@@ -255,11 +255,7 @@ describe('Module syntax detection', { concurrency: !process.env.TEST_PARALLEL },
 
     it('reports unfinished top-level `await`', async () => {
       const { stdout, stderr, code, signal } = await spawnPromisified(process.execPath, [
-        '--input-type=module',
-        '--eval',
-        `
-        await new Promise(() => {});
-        `,
+        fixtures.path('es-modules/tla/unresolved.js'),
       ]);
 
       match(stderr, /Warning: Detected unsettled top-level await/);

From 1e12966996d51cbbac477d04de542dd6c02acbaa Mon Sep 17 00:00:00 2001
From: Mert Can Altin <mertgold60@gmail.com>
Date: Sat, 30 Nov 2024 23:34:20 +0300
Subject: [PATCH 05/23] Update test/es-module/test-esm-detect-ambiguous.mjs

Co-authored-by: Geoffrey Booth <webadmin@geoffreybooth.com>
---
 test/es-module/test-esm-detect-ambiguous.mjs | 1 -
 1 file changed, 1 deletion(-)

diff --git a/test/es-module/test-esm-detect-ambiguous.mjs b/test/es-module/test-esm-detect-ambiguous.mjs
index bcc2e859c599b9..82d921695ee5d9 100644
--- a/test/es-module/test-esm-detect-ambiguous.mjs
+++ b/test/es-module/test-esm-detect-ambiguous.mjs
@@ -266,7 +266,6 @@ describe('Module syntax detection', { concurrency: !process.env.TEST_PARALLEL },
 
     it('permits top-level `await` above import/export syntax', async () => {
       const { stdout, stderr, code, signal } = await spawnPromisified(process.execPath, [
-        '--input-type=module',
         '--eval',
         `
           await Promise.resolve();

From 11d788e03f5a5c4ecb893569d3635e0093220fe1 Mon Sep 17 00:00:00 2001
From: Mert Can Altin <mertgold60@gmail.com>
Date: Sat, 30 Nov 2024 23:34:32 +0300
Subject: [PATCH 06/23] Update test/es-module/test-esm-detect-ambiguous.mjs

Co-authored-by: Geoffrey Booth <webadmin@geoffreybooth.com>
---
 test/es-module/test-esm-detect-ambiguous.mjs | 1 -
 1 file changed, 1 deletion(-)

diff --git a/test/es-module/test-esm-detect-ambiguous.mjs b/test/es-module/test-esm-detect-ambiguous.mjs
index 82d921695ee5d9..e8dd40db581b3f 100644
--- a/test/es-module/test-esm-detect-ambiguous.mjs
+++ b/test/es-module/test-esm-detect-ambiguous.mjs
@@ -394,7 +394,6 @@ describe('Module syntax detection', { concurrency: !process.env.TEST_PARALLEL },
 
     it('does not warn when there are no package.json', async () => {
       const { stdout, stderr, code, signal } = await spawnPromisified(process.execPath, [
-        '--no-warnings',
         fixtures.path('es-modules/loose.js'),
       ]);
 

From 2f8e7d46df7b131806eaa568bb372dc6258e3db8 Mon Sep 17 00:00:00 2001
From: Mert Can Altin <mertgold60@gmail.com>
Date: Sat, 30 Nov 2024 23:34:43 +0300
Subject: [PATCH 07/23] Update test/es-module/test-esm-detect-ambiguous.mjs

Co-authored-by: Geoffrey Booth <webadmin@geoffreybooth.com>
---
 test/es-module/test-esm-detect-ambiguous.mjs | 6 +-----
 1 file changed, 1 insertion(+), 5 deletions(-)

diff --git a/test/es-module/test-esm-detect-ambiguous.mjs b/test/es-module/test-esm-detect-ambiguous.mjs
index e8dd40db581b3f..4b98545fc79236 100644
--- a/test/es-module/test-esm-detect-ambiguous.mjs
+++ b/test/es-module/test-esm-detect-ambiguous.mjs
@@ -267,11 +267,7 @@ describe('Module syntax detection', { concurrency: !process.env.TEST_PARALLEL },
     it('permits top-level `await` above import/export syntax', async () => {
       const { stdout, stderr, code, signal } = await spawnPromisified(process.execPath, [
         '--eval',
-        `
-          await Promise.resolve();
-          import "node:os";
-          console.log("executed");
-        `,
+        'await Promise.resolve(); import "node:os"; console.log("executed");',
       ]);
 
       strictEqual(stderr, '');

From bd58f77c44846cd547b892011e6b77ecd1a2e650 Mon Sep 17 00:00:00 2001
From: Mert Can Altin <mertgold60@gmail.com>
Date: Sat, 30 Nov 2024 23:34:51 +0300
Subject: [PATCH 08/23] Update test/es-module/test-esm-detect-ambiguous.mjs

Co-authored-by: Geoffrey Booth <webadmin@geoffreybooth.com>
---
 test/es-module/test-esm-detect-ambiguous.mjs | 1 -
 1 file changed, 1 deletion(-)

diff --git a/test/es-module/test-esm-detect-ambiguous.mjs b/test/es-module/test-esm-detect-ambiguous.mjs
index 4b98545fc79236..f72f8267b78691 100644
--- a/test/es-module/test-esm-detect-ambiguous.mjs
+++ b/test/es-module/test-esm-detect-ambiguous.mjs
@@ -333,7 +333,6 @@ describe('Module syntax detection', { concurrency: !process.env.TEST_PARALLEL },
 
     it('permits declaration of CommonJS module variables above import/export', async () => {
       const { stdout, stderr, code, signal } = await spawnPromisified(process.execPath, [
-        '--input-type=commonjs',
         '--eval',
         `
         console.log(typeof module, typeof exports, typeof require);

From 2e77c7c8a724447b767e58c5611d56d74ad29d0d Mon Sep 17 00:00:00 2001
From: Mert Can Altin <mertgold60@gmail.com>
Date: Sat, 30 Nov 2024 23:35:06 +0300
Subject: [PATCH 09/23] Update test/es-module/test-esm-detect-ambiguous.mjs

Co-authored-by: Geoffrey Booth <webadmin@geoffreybooth.com>
---
 test/es-module/test-esm-detect-ambiguous.mjs | 2 --
 1 file changed, 2 deletions(-)

diff --git a/test/es-module/test-esm-detect-ambiguous.mjs b/test/es-module/test-esm-detect-ambiguous.mjs
index f72f8267b78691..20f4be7e5dfa93 100644
--- a/test/es-module/test-esm-detect-ambiguous.mjs
+++ b/test/es-module/test-esm-detect-ambiguous.mjs
@@ -276,10 +276,8 @@ describe('Module syntax detection', { concurrency: !process.env.TEST_PARALLEL },
       strictEqual(signal, null);
     });
 
-
     it('still throws on `await` in an ordinary sync function', async () => {
       const { stdout, stderr, code, signal } = await spawnPromisified(process.execPath, [
-        '--input-type=module',
         '--eval',
         `
           function fn() { await Promise.resolve(); }

From 42387e9e3ccdc189fed54733cd3575ec1616dca5 Mon Sep 17 00:00:00 2001
From: Mert Can Altin <mertgold60@gmail.com>
Date: Sat, 30 Nov 2024 23:35:20 +0300
Subject: [PATCH 10/23] Update test/es-module/test-esm-detect-ambiguous.mjs

Co-authored-by: Geoffrey Booth <webadmin@geoffreybooth.com>
---
 test/es-module/test-esm-detect-ambiguous.mjs | 5 +----
 1 file changed, 1 insertion(+), 4 deletions(-)

diff --git a/test/es-module/test-esm-detect-ambiguous.mjs b/test/es-module/test-esm-detect-ambiguous.mjs
index 20f4be7e5dfa93..ab5a4f413d57af 100644
--- a/test/es-module/test-esm-detect-ambiguous.mjs
+++ b/test/es-module/test-esm-detect-ambiguous.mjs
@@ -279,10 +279,7 @@ describe('Module syntax detection', { concurrency: !process.env.TEST_PARALLEL },
     it('still throws on `await` in an ordinary sync function', async () => {
       const { stdout, stderr, code, signal } = await spawnPromisified(process.execPath, [
         '--eval',
-        `
-          function fn() { await Promise.resolve(); }
-          fn();
-        `,
+        'function fn() { await Promise.resolve(); } fn();',
       ]);
 
       match(stderr, /SyntaxError: (await is only valid in async function|Unexpected reserved word)/);

From 34e2c3c385094617b18ddcaf6529e7ada5dfd528 Mon Sep 17 00:00:00 2001
From: Mert Can Altin <mertgold60@gmail.com>
Date: Sat, 30 Nov 2024 23:35:29 +0300
Subject: [PATCH 11/23] Update test/es-module/test-esm-detect-ambiguous.mjs

Co-authored-by: Geoffrey Booth <webadmin@geoffreybooth.com>
---
 test/es-module/test-esm-detect-ambiguous.mjs | 3 +--
 1 file changed, 1 insertion(+), 2 deletions(-)

diff --git a/test/es-module/test-esm-detect-ambiguous.mjs b/test/es-module/test-esm-detect-ambiguous.mjs
index ab5a4f413d57af..f4731c213d7304 100644
--- a/test/es-module/test-esm-detect-ambiguous.mjs
+++ b/test/es-module/test-esm-detect-ambiguous.mjs
@@ -282,8 +282,7 @@ describe('Module syntax detection', { concurrency: !process.env.TEST_PARALLEL },
         'function fn() { await Promise.resolve(); } fn();',
       ]);

