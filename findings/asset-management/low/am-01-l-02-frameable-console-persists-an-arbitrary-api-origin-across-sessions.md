# Source-code-based

Project: Anonymous asset-management vault protocol (AM-01)  
Category: Asset management, vaults and integrations  
Severity: Low  
Engagement: Independent hunting + responsible disclosure  
Chain: Stellar / Soroban; Web2 findings concern the associated console and API  
Tech stack: Rust 2021, Soroban SDK 22.0.3, WASM, token/strategy vaults, Blend integration; JavaScript/React, Axios, bearer-token API and XDR signing workflows  
Evidence: Source/configuration analysis; no finding-specific executable exploit test  
Remediation: not demonstrated  
Contribution: Research and disclosure case study; exclusive discovery credit is not asserted  
Proof availability: Excerpts and source references; the original harness and dependencies are not distributed.

# Persistent API-origin override forwards bearer credentials

## Description

Console user changes Settings → any nonempty api_base_url persists across logout/account switching → login, refresh, transaction-builder and signed-XDR submission flows trust that origin → credentials and transaction data can be routed outside the intended backend. An attacker-induced setting change remains unproved.

Frameability, persistence and dangerous dataflow confirmed. Reliable unprivileged writer, credential theft and deceptively signed transaction not demonstrated. Source labels this Low/hardening and incomplete.

### Mechanism step by step

#### Step 1 — Settings persists an arbitrary nonempty origin

Changing this value requires user action or a separate attacker-controlled writer. That writer was not proved.

Source: `work/console-passive-review/evidence/bundle.js:480`, exact substring of the minified line, character offsets 1578905–1579177 (zero-based).

```javascript
function Gye(){let{network:e,setNetwork:t}=g_(),[n,r]=(0,C.useState)(()=>localStorage.getItem(`api_base_url`)||`[URL redacted]`);function i(){n.trim()&&(localStorage.setItem(`api_base_url`,n.trim()),Bw.success(`API URL saved`,{description:n.trim()}))}function a(){
```

#### Step 2 — Authenticated requests follow the historical origin

The request interceptor applies the selected baseURL and attaches bearer authorization.

Source: `work/console-passive-review/evidence/bundle.js:18`, exact substring of the minified line, character offsets 551201–551678 (zero-based).

```javascript
function a_(){return localStorage.getItem(i_.API_BASE_URL)||`[URL redacted]`}var o_=r_.create();function s_(e,t){if(r_.isAxiosError(e)){let t=e.response?.data?.message;if(typeof t==`string`&&t.trim()!==``)return t;if(Array.isArray(t)&&t.length>0)return t.join(`. `)}return e instanceof Error&&e.message?e.message:t}o_.interceptors.request.use(e=>{e.baseURL=a_();let t=localStorage.getItem(i_.ACCESS_TOKEN);return t&&(e.headers.Authorization=`Bearer ${t}`),e});var c_=!1
```

#### Step 3 — Logout leaves the origin intact

The logout cleanup removes credential and user keys, but not the API-origin key. Frameability plus this dataflow is established; reliable clickjacking or malicious-wallet-signature inducement is not.

Source: `work/console-passive-review/evidence/bundle.js:18`, exact substring of the minified line, character offsets 552708–552849 (zero-based).

```javascript
function d_(){localStorage.removeItem(i_.ACCESS_TOKEN),localStorage.removeItem(i_.REFRESH_TOKEN),localStorage.removeItem(i_.USER)}function ute
```

## Proof of concept

The shipped client code establishes persistent API-origin selection, bearer forwarding and logout behavior. Frameability was also observed. No complete attacker-to-victim test establishes a reliable external writer that changes another user’s origin without informed consent. The missing writer prevents an account-takeover or asset-drain claim.

**Proof limits:** Source/dataflow evidence only; victim origin modification and deceptive signature induction are unproved.

## Relevant test excerpts

No executable test is included for this case. The source excerpts above support the stated behavior and its prerequisites.
