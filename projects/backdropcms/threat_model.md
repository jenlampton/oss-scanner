# Threat model

The scanner reads this file before it starts. It covers Backdrop CMS core. Severity follows the
([Drupal security risk levels](https://www.drupal.org/drupal-security-team/security-risk-levels-defined) and the
[Backdrop CMS security advisory policy](https://backdropcms.org/security-advisory-policy)).

## What this project does and where untrusted input enters

Backdrop CMS is a PHP content management system and a fork of the Drupal project. Sites built on it are public
web applications, so treat all HTTP input as untrusted: paths, query strings, form submissions (including Form
API `#ajax` callbacks and multi-step form state), uploaded files, cookies and headers, and content that one user
stores and other users later see (node bodies, titles, field values, comments, menu links, URL aliases, contact
form messages, and so on).

### Where the code is

| Path | What it is |
| --- | --- |
| `/backdrop` | The Backdrop CMS core git checkout (the `1.x` branch, where core development happens). Patches for core belong here. This is the document root of a working site, built the way [docs.backdropcms.org documents](https://docs.backdropcms.org/documentation/installation-instructions), on the current Backdrop 1.x development build, served by Apache with `.htaccess` enabled, on MariaDB. |
| `/backdrop/files` | The directory that will hold user-uploaded files. Must be web-writable. |
| `/backdrop/files/config` | Configuration files needed for running Backdrop CMS.|

Contributed modules and themes are out of scope.

### Trust model

- **Anonymous visitors** are untrusted.
- **Authenticated users** must stay within the permissions their roles grant, plus entity access, node
  grants, field access, text-format access and route access checks.
- **Permissions marked `restrict access: true`** in a module's `*.module` (for example "administer site
  configuration", "administer users", "administer permissions", "administer account settings", "administer
  filters", "administer software updates") and uid 1 are trusted. Per the Backdrop CMS security advisory policy,
  a vulnerability that needs one of these permissions is **not** a security issue. Do not report it unless it lets the attacker do something that permission does not already grant. Check the `.module` file before treating a permission as restricted. Many "administer …" permissions are **not** restricted, for example  "administer blocks", "administer menu", "administer taxonomy", "administer comments" and "administer url aliases" (the `editor` role has this one). Issues reachable with those are in scope and scored as A:Admin.
- **Text formats are a trust boundary.** A user allowed to use an unfiltered format (for example "Raw HTML")
  can add arbitrary markup by design, which is not a finding. XSS through a *filtered* format (`filtered_html`) or through `filter_xss()` / `filter_xss_admin()` *is* a finding.
- `settings.php`, the environment, and the file system are trusted operator input.

## Components that matter most / least

Highest priority:

1. Access bypass: route requirements (`_permission`, `_access`, `_entity_access`, `_csrf_token`), entity and
   field access handlers, node grants, query access tags (`node_access`, entity query), and Views access and
   argument handling.
2. XSS: render arrays (`#markup` and `#children`) with unfiltered input, `:url` placeholders outside an `href`,
   placeholders inside unquoted or JavaScript attributes, user input that has already been rendered and is passed as `@`), JavaScript `Backdrop.behaviors` that build HTML, `backdropSettings`, and attribute handling. Plain `@` and `%` placeholders escape their values.
3. SQL injection through the Database API (`condition()` with user-controlled operators or field names,
   `query()` with concatenation, `orderBy()`, `addExpression()`) and Views and Search API query building.
4. CSRF: state-changing GET routes without `_csrf_token`, and Form API token validation.
5. Remote code execution or object injection: `unserialize()` on user-controlled data, file upload
   validation (extensions, `.htaccess`, double extensions), the stream wrappers, the image toolkit, and
   private file access.
6. Open redirects (the `destination` parameter), session handling, the flood control and
   password reset / login flows, and cache poisoning (missing cache contexts that leak per-user content to
   other users).

Lower priority or out of scope:

- Test code and test-only modules (`tests/`, `*_test` modules, `core/modules/system/tests`) and anything under
  `core/scripts`.

## How to exercise it

- Run `backdrop-start` first. It starts MariaDB and Apache (the container has no init system) and is safe to run
  again. The site is then at `http://127.0.0.1/`. `backdrop-stop` stops both.
- The site is pre-installed with the standard profile and a private file system at `/files/private`:
  - `admin` / `admin`: uid 1, fully trusted
  - `editor` / `editor`: the `editor` role, useful for testing access bypass and XSS
  - anonymous: no login
- `bee` is on the `PATH` and runs as `www-data`, the Apache user, so files it writes stay writable by the site.
  `bee uli --name=editor` prints a working login link. Use `bee php-eval` to set up content, `bee pm-enable`
  to enable more core modules, and `bee role:perm:add` to give a role exactly the permissions a scenario needs.
- The database is `mysql://backdrop:backdrop@127.0.0.1/backdrop`. `bee sql-cli` opens a client.
- PHP errors go to Apache's error log on stderr, not into responses, as on a production site.
- Run functional tests using the shell script `./core/scripts/run-tests.sh`. Functional tests all run offline.

## How you rate severity

Use the Drupal security risk calculator. Give the risk string with every report, for example
`AC:None/A:None/CI:Some/II:Some/E:Theoretical/TD:Default` (4+4+3+3+1+2 = 17, Critical). Each factor adds points:

- **Access complexity (AC)**: None (the user visits a page) 4, Basic (the user must follow a specific
  path) 2, Complex (a multi-step, unintuitive process with many dependencies) 1
- **Authentication (A)**, the least privileged role that can exploit it: None (anonymous) 4,
  User (basic, commonly assigned permissions) 2, Admin (broad permissions whose `restrict access` is false) 1.
  A permission with `restrict access: true` is trusted, so an issue that needs one is not scored (see above).
- **Confidentiality impact (CI)**: All non-public data 5, Some 3, None 0
- **Integrity impact (II)**: All data can be modified or deleted 5, Some 3, None 0
- **Exploit (E)**: Exploit (public exploit code exists) 4, Proof (public documentation of how to exploit it) 2,
  Theoretical 1. A newly found issue with a private reproducer is Theoretical.
- **Target distribution (TD)**: All module configurations 3, Default or common configurations 2,
  Uncommon configurations only 1

Total score: 0–4 Not critical, 5–9 Less critical, 10–14 Moderately critical, 15–19 Critical,
20–25 Highly critical.

Rules of thumb:

- Stored XSS any authenticated user can trigger is usually Moderately critical. Reflected XSS
  against anonymous visitors in a default configuration is usually Critical.
- Anonymous SQL injection or remote code execution in a default configuration is Highly critical.
- An access bypass that exposes unpublished content or private files is usually Moderately critical. An
  access bypass that grants a role or administrative access is Critical or higher.
- An issue that needs a `restrict access: true` permission is not a vulnerability. Leave it out.
- A denial of service that needs only a single anonymous request and little bandwidth is in scope, but rarely
  above Less critical. Volumetric DoS is out of scope.

## How reports and patches should look

- Name the core version, the risk string and score, the affected file and function, the least
  privileged role that can trigger it, and a self-contained reproducer: bee commands plus `curl`, or a
  functional test.
- Write patches as `git diff` against the source. Follow Backdrop CMS coding standards, use core APIs
  (`check_plain`, `filter_xss`, `filter_xss_admin`, and placeholders in queries) rather than ad hoc escaping,
  and keep cacheability metadata correct.

## Anything to leave alone

- Missing security headers, `X-Powered-By`, version disclosure `core/install.php`, username enumeration through
  the login or password reset forms, and the self-XSS of a user editing their own account. Backdrop CMS treats
  all of these as known, public hardening issues.
- Findings that depend on turning on `display_errors`, on other `settings.php` changes, or on development modules
  (`devel`, verbose `dblog` output). The private file system configured in this image counts as a normal
  configuration, not a change.
